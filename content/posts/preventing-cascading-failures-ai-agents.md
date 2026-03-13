---
title: "Preventing Cascading Failures in AI Agents"
date: 2026-03-12
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "APIM", "Resilience", "Circuit Breaker", "OpenTelemetry"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1tyjxphuz2rmud27gh8a.png
    alt: "Preventing OWASP ASI08 Cascading Failures in a .NET AI agent with resilience handlers, structured error responses, caching, and distributed tracing."
    caption: "Implementing OWASP ASI08 mitigations against Cascading Failures in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

Your AI agent depends on a chain of services. In my side project ([Biotrackr](https://github.com/willvelida/biotrackr)), the chain looks like this: Claude API for reasoning, APIM for routing, downstream APIs for health data, and Cosmos DB for chat history. When one link in that chain fails, things can get ugly fast.

Imagine this: Claude API returns a 429 (rate limited). The agent retries the same request. Each retry consumes more tokens. More 429s. The conversation times out. The user sees an error and submits again, doubling the load. A single rate limit hit has cascaded into a degraded experience, wasted tokens, and a frustrated user (in my case, just me screaming at my own agent 😅 For you however, that would be your customers!).

Cascading Failures (ASI08) is about building resilience into the agent so that one fault doesn't propagate into a system-wide failure. The OWASP specification defines this as *"failures that propagate across agent systems, where an initial malfunction in one agent or component triggers a chain of subsequent failures."*

ASI08 builds on the resilience dimensions that traditional distributed systems engineering has long addressed, but applies them to a context where failures are uniquely expensive. Every retry burns LLM tokens, and every confused error-handling loop amplifies cost. The OWASP specification defines 10 prevention and mitigation guidelines. Let's walk through each one and see how Biotrackr implements (or could implement) them.

## What are Cascading Failures in Agent Systems?

In traditional web applications, a failing dependency usually means one feature degrades. In agent systems, failures compound in ways that are uniquely expensive and destructive.

Here are a few agent-specific cascade scenarios:

1. **Tool failure → retry loop → token consumption** — a failing tool causes the agent to retry, and each retry consumes LLM tokens. The agent is trying to be helpful by retrying, but it's burning through your API budget while achieving nothing.
2. **LLM outage → UI hang → user retries → amplification** — Claude API goes down, the UI shows a loading spinner, the user submits again, and now you've got double the load on an already struggling system.
3. **Data inconsistency → hallucination → bad analysis** — an API returns partial data (maybe a timeout cuts the response short), and the agent fills in gaps with hallucinated data. The user gets confidently wrong analysis.
4. **Rate limit → backpressure → timeout** — APIM rate limits trigger exponential retries that eventually timeout, wasting compute and tokens at every step.

The key difference between cascading failures in traditional systems and agent systems is **token cost**. In a traditional web app, retries are cheap, maybe a few milliseconds of compute. In an agent system, every retry attempt involves sending the full conversation context back to the LLM. A retry loop that sends 10 requests to Claude doesn't just waste HTTP round-trips, it consumes 10x the tokens.

## Why does this matter for Biotrackr?

Why should I care about cascading failures in my little side project?

The chat agent has 3 external dependencies: Claude API, APIM/Biotrackr APIs, and Cosmos DB. Each tool call involves a chain: agent → APIM → health data API → Cosmos DB → back to the agent → sent to Claude. A failure at any point in this chain can cascade through the entire conversation.

And each link in the chain costs money. Claude API tokens for reasoning, APIM calls for routing, Cosmos DB RUs for data access. A cascade of retries across all three services amplifies costs multiplicatively, not linearly.

Even for a side project, I don't want to wake up to a surprise bill because the Activity API had a bad day and the agent decided to retry 50 times. Have a think about the agents you've deployed in your organization. How many external dependencies does your agent have? How expensive are retries in your system?

With all this in mind, let's walk through each prevention and mitigation strategy we can implement to prevent cascading failures, with some examples of how I've implemented them in my agent.

## Zero-Trust Fault Tolerance

*"Design system with fault tolerance that assumes availability failure of LLM, agentic function components and external sources."*

The foundation of cascade prevention is assuming everything will fail. The LLM will go down. The APIs will return errors. Cosmos DB will throttle you. If you design for the happy path, the first failure takes down the whole system.

Biotrackr implements zero-trust fault tolerance at multiple layers: resilience handlers on HTTP calls, structured error responses from tools, in-memory caching to survive downstream outages, and graceful degradation when the LLM itself is unavailable.

The highest-value single line of code for fault tolerance is `AddStandardResilienceHandler()`. `Microsoft.Extensions.Http.Resilience` provides production-grade resilience out of the box, and this one call adds five layers of protection:

```csharp
// Program.cs — HttpClient with resilience handler
builder.Services.AddHttpClient("BiotrackrApi", (sp, client) =>
{
    var settings = sp.GetRequiredService<IOptions<Settings>>().Value;
    client.BaseAddress = new Uri(settings.ApiBaseUrl
        ?? throw new InvalidOperationException("Biotrackr:ApiBaseUrl is not configured."));
})
.AddHttpMessageHandler<ApiKeyDelegatingHandler>()
.AddStandardResilienceHandler();  // ← This single line adds 5 resilience layers
```

That one `.AddStandardResilienceHandler()` call adds:

1. **Rate limiter** — limits concurrent outbound requests, preventing the agent from overwhelming APIM with concurrent tool calls
2. **Total request timeout** (default 30s) — if a tool call takes more than 30 seconds end-to-end, it's abandoned. The agent gets a timeout error instead of waiting indefinitely
3. **Retry** (default 3 retries) — transient 5xx errors are retried with exponential backoff and jitter. The agent doesn't need to handle retries itself
4. **Circuit breaker** (default: opens after 10% failure rate in a 30s window) — if APIM is consistently failing, the circuit opens and tool calls fail immediately. No more wasted tokens on requests that are going to fail anyway
5. **Attempt timeout** (default 10s per attempt) — each individual retry attempt has its own timeout, preventing slow responses from consuming the full request budget

When a tool fails, the error message sent back to the agent matters critically. If the tool throws an exception, the Agent Framework may surface internal details to the LLM, wasting tokens on a response the user can't use, and potentially leaking infrastructure details. Instead, every tool in Biotrackr catches errors and returns structured JSON:

```csharp
// ActivityTools.cs — structured error response
public async Task<string> GetActivityByDate(string date)
{
    if (!DateOnly.TryParse(date, out _))
        return """{"error": "Invalid date format. Use YYYY-MM-DD."}""";

    var client = httpClientFactory.CreateClient("BiotrackrApi");
    var response = await client.GetAsync($"/activity/{date}");

    if (!response.IsSuccessStatusCode)
        return $"{{\"error\": \"Activity data not found for {date}.\"}}";

    return await response.Content.ReadAsStringAsync();
}
```

By returning a clean JSON error, we give the agent a clear signal: this data isn't available right now. The agent can communicate that to the user and move on. No confused retry loops, no stack traces leaking infrastructure details to the LLM.

Caching isn't just there for performance optimisation, it's also a fault tolerance mechanism. If an API is intermittently failing, cached results from successful calls are still available. Every tool uses `IMemoryCache` with adaptive TTLs:

```csharp
// ActivityTools.cs — caching prevents cascading failures
var cacheKey = $"activity:{date}";
if (cache.TryGetValue(cacheKey, out string? cached))
    return cached!;  // ← API is down, but we have cached data — no cascade

var result = await response.Content.ReadAsStringAsync();

var ttl = DateOnly.Parse(date) == DateOnly.FromDateTime(DateTime.UtcNow)
    ? TimeSpan.FromMinutes(5)    // Today's data — may still be syncing
    : TimeSpan.FromHours(1);     // Historical — stable
cache.Set(cacheKey, result, ttl);
```

The Claude API is the agent's brain. If it's down, the agent can't function. For LLM unavailability, the key principle is: don't let the user's experience degrade worse than "chat is unavailable." The Chat API's `/healthz/liveness` endpoint could be extended to check Claude API reachability. If the health check fails, the UI can show a degraded state banner instead of letting users submit messages that will inevitably fail, preventing the amplification cascade where frustrated users resubmit and double the load.

Some key points here:

- **Five-layer resilience** — `AddStandardResilienceHandler()` adds rate limiting, timeouts, retries with backoff, circuit breaking, and per-attempt timeouts in one line
- **Structured errors** — tools return `{"error": "..."}` instead of throwing exceptions, preventing the agent from entering confused retry states
- **Cache as fallback** — `IMemoryCache` with adaptive TTLs means the agent can answer questions about recently-fetched data even when the API is down
- **Graceful LLM degradation** — a clean "unavailable" message is better than a spinner that never resolves or a cryptic error

What's missing is a readiness health check for Claude API availability. The current `/healthz/liveness` endpoint doesn't verify that the LLM is reachable. Adding a readiness probe that pings Claude API would let the UI proactively disable chat when the LLM is down, preventing the user-retry amplification cascade entirely.

## Isolation and Trust Boundaries

*"Sandbox agents, least privilege, network segmentation, scoped APIs, and mutual auth to contain failure propagation."*

Isolation ensures that when a failure does occur, it stays contained. A failing tool shouldn't be able to take down the conversation store. A compromised API key shouldn't grant access to the entire infrastructure.

Biotrackr enforces isolation at multiple levels: network boundaries via APIM, least-privilege identity via Entra Agent ID, container-level resource limits, and TLS enforcement on all communication channels.

APIM acts as a network boundary and trust gateway between the agent and downstream APIs. The agent never calls downstream APIs directly, as all traffic flows through APIM, which enforces authentication and rate limiting:

```csharp
// ApiKeyDelegatingHandler.cs — APIM as trust boundary
protected override async Task<HttpResponseMessage> SendAsync(
    HttpRequestMessage request, CancellationToken cancellationToken)
{
    if (!string.IsNullOrWhiteSpace(_subscriptionKey))
    {
        request.Headers.TryAddWithoutValidation(SubscriptionKeyHeader, _subscriptionKey);
    }
    return await base.SendAsync(request, cancellationToken);
}
```

The agent authenticates to Cosmos DB via Entra Agent ID with least-privilege access (Cosmos DB Data Contributor on a single account, not Contributor at the resource group level):

```csharp
// AgentIdentityCosmosClientFactory.cs — agent identity scoped to Cosmos DB Data Contributor
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;

return new CosmosClient(_settings.CosmosEndpoint, _credential, new CosmosClientOptions
{
    SerializerOptions = new CosmosSerializationOptions
    {
        PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
    }
});
```

Container-level resource limits provide a last-resort ceiling, and TLS is enforced on all external communication:

```bicep
// infra/apps/chat-api/main.bicep — resource constraints and TLS
resources: {
  cpu: json('0.25')    // 0.25 vCPU — limits compute abuse
  memory: '0.5Gi'      // 512MB — prevents memory exhaustion
}

ingress: {
  external: true
  targetPort: 8080
  transport: 'http'
  allowInsecure: false  // TLS required — no plaintext HTTP allowed
}
```

Some key points here:

- **APIM as boundary** — the agent never directly contacts downstream APIs. APIM provides authentication, rate limiting, and network segmentation between the agent and backend services
- **Least-privilege identity** — the agent identity has Cosmos DB Data Contributor (role `00000000-0000-0000-0000-000000000002`) on a single account — it cannot access Key Vault, Storage, or other resources
- **Container sandbox** — 0.25 vCPU and 512MB memory per replica. Even if the agent enters a retry loop, resource consumption is bounded
- **TLS everywhere** — `allowInsecure: false` on Container App ingress, APIM endpoints enforce HTTPS, Cosmos DB connections are TLS-only
- **Federated Identity Credential** — the agent authenticates via FIC (no client secrets in production), and tokens are automatically rotated by the platform.

## JIT, One-Time Tool Access with Runtime Checks

*"Issue short-lived, task-scoped credentials for each agent run and validate every high-impact tool invocation against a policy-as-code rule before executing it. This ensures a compromised or drifting agent cannot trigger chain reactions across other agents or systems."*

This guideline is about ensuring that tool access is ephemeral and validated. An agent should only have the credentials it needs for the current task, and high-impact operations should be checked against a policy before execution.

Biotrackr partially implements this through its credential architecture, but does not yet have per-invocation credential issuance or policy-as-code validation on tool calls.

The agent identity uses Entra Agent ID with Federated Identity Credentials. Tokens are short-lived (typically 1-hour lifetime) and automatically rotated by the platform:

```csharp
// AgentIdentityCosmosClientFactory.cs — short-lived, platform-managed tokens
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
// Tokens are issued by Entra ID with a finite lifetime
// The SDK handles refresh automatically — no manual credential management
```

The APIM subscription key is scoped to a single APIM instance and loaded from Azure App Configuration (backed by Key Vault), not hardcoded:

```csharp
// Settings.cs — credentials loaded from App Configuration at startup
public string ApiSubscriptionKey { get; set; }  // Resolved from Key Vault reference in App Config
public string AnthropicApiKey { get; set; }       // Resolved from Key Vault reference in App Config
```

Tool inputs are validated before execution. Date formats are checked, date ranges are capped at 365 days, and page sizes are bounded to prevent resource exhaustion:

```csharp
// ActivityTools.cs — input validation as a runtime check
if (!DateOnly.TryParse(date, out _))
    return """{"error": "Invalid date format. Use YYYY-MM-DD."}""";

// Date range tools enforce a maximum span
if ((endDate.ToDateTime(TimeOnly.MinValue) - startDate.ToDateTime(TimeOnly.MinValue)).Days > 365)
    return """{"error": "Date range cannot exceed 365 days."}""";
```

Some key points here:

- **Short-lived tokens** — Entra Agent ID tokens have a finite lifetime and are automatically refreshed by the SDK, limiting the window of exposure if a token is compromised
- **Key Vault-backed secrets** — APIM subscription keys and Anthropic API keys are stored in Key Vault and accessed via App Configuration references, not environment variables or config files
- **Input validation** — every tool validates its inputs before making downstream calls, acting as a basic runtime policy check

## Independent Policy Enforcement

*"Separate planning and execution via an external policy engine to prevent corrupt planning from triggering harmful actions."*

This guideline addresses a fundamental risk in agent systems: if the LLM handles both deciding what to do and doing it, a single hallucination or injection can cascade into harmful actions. Separating planning from execution ensures that even if the LLM's reasoning is corrupted, an independent layer validates actions before they execute.

Biotrackr implements partial separation through its architecture. The system prompt is immutably loaded from an external source, and APIM acts as an external enforcement layer. However, there is no explicit policy engine separating the agent's planning from tool execution.

The system prompt is loaded from Azure App Configuration at startup. The agent cannot modify its own instructions at runtime, and a corrupted conversation cannot change the rules:

```csharp
// Program.cs — system prompt loaded from App Configuration, immutable at runtime
var systemPrompt = builder.Configuration.GetValue<string>("Biotrackr:ChatSystemPrompt")!;

AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,  // Read-only — agent cannot modify this
    tools: [ /* fixed tool set — agent cannot add or remove tools */ ]);
```

APIM acts as an external enforcement layer that the agent cannot bypass. Even if the agent's reasoning is corrupted and it tries to make 1,000 API calls, APIM enforces rate limits and subscription quotas independently:

```xml
<!-- APIM policy — enforcement independent of agent behavior -->
<rate-limit-by-key calls="100" renewal-period="60"
    counter-key="@(context.Subscription.Id)" />
```

The tool set is fixed at startup — the agent cannot dynamically register new tools or remove safety checks:

```csharp
// Program.cs — fixed tool registration, agent cannot modify
tools:
[
    AIFunctionFactory.Create(activityTools.GetActivityByDate),
    AIFunctionFactory.Create(activityTools.GetActivityByDateRange),
    AIFunctionFactory.Create(activityTools.GetActivityRecords),
    // ... all 12 tools registered at startup, immutable
]);
```

Some key points here:

- **Immutable system prompt** — loaded from Azure App Configuration at startup, not modifiable by the agent at runtime
- **Fixed tool set** — the agent cannot dynamically add, remove, or modify tools. The tool definitions are compiled into the application
- **APIM as external policy** — rate limits, subscription quotas, and authentication are enforced by APIM independently of the agent's reasoning
- **No tool self-registration** — a corrupted planning step cannot cause the agent to register a "delete all data" tool.

For a single-user side project, the immutable configuration and APIM enforcement are sufficient. For a multi-agent production system where agents can invoke other agents, an independent policy engine becomes critical to prevent one agent's corrupt planning from triggering cascading harmful actions across the system.

## Output Validation and Human Gates

*"Checkpoints, governance agents, or human review for high risk before agent outputs are propagated downstream."*

In agent systems, outputs aren't just text. They can trigger actions, influence downstream systems, or inform real-world decisions. Before an agent's output is propagated (to the user, to another agent, or to a downstream system), high-risk outputs should be validated or reviewed.

Biotrackr implements basic output guardrails through the system prompt and structured error responses, but does not currently have automated output validation or human-in-the-loop gates.

The system prompt includes a safety boundary that instructs the agent to disclaim medical authority:

```csharp
// System prompt includes output guardrails
"You are not a medical professional — remind users to consult a healthcare provider for medical advice."
```

Structured error responses from tools prevent the agent from propagating infrastructure details downstream. When a tool fails, the user sees a clean error instead of a stack trace:

```csharp
// ActivityTools.cs — errors are sanitised before reaching the agent/user
if (!response.IsSuccessStatusCode)
    return $"{{\"error\": \"Activity data not found for {date}.\"}}";
// No stack traces, no internal URLs, no connection strings leak to the LLM or user
```

The `ConversationPersistenceMiddleware` provides a checkpoint where output validation could be inserted. It already intercepts all agent responses before they're persisted:

```csharp
// ConversationPersistenceMiddleware.cs — checkpoint for output validation
var responseText = new System.Text.StringBuilder();

await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
{
    foreach (var content in update.Contents)
    {
        if (content is TextContent textContent)
        {
            responseText.Append(textContent.Text);
        }
    }
    yield return update;
}

// After streaming completes: validate before persisting
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);
```

Some key points here:

- **System prompt guardrails** — the agent is instructed to disclaim medical authority, creating a soft output gate
- **Sanitised errors** — tool failures return clean JSON errors, preventing infrastructure details from leaking to the user
- **Persistence checkpoint** — the middleware intercepts all responses before persistence, providing a natural insertion point for validation

What's missing is automated output validation. The middleware could scan the assistant's response before persistence for content that violates safety constraints. For example, detecting if the agent provided a specific medical diagnosis despite the system prompt guardrail:

```csharp
// Recommended: output validation before persistence
private bool ContainsRiskyHealthAdvice(string content)
{
    var patterns = new[]
    {
        @"\b(diagnos|prescri|you\s+have|you\s+should\s+take)\b",
        @"\b(stop\s+taking|increase\s+your\s+dose|skip\s+your\s+medication)\b"
    };
    return patterns.Any(p => Regex.IsMatch(content, p, RegexOptions.IgnoreCase));
}

// In middleware, after streaming completes:
if (ContainsRiskyHealthAdvice(assistantContent))
{
    logger.LogWarning("Agent response in session {SessionId} may contain risky health advice", sessionId);
    // Option 1: Append a disclaimer automatically
    // Option 2: Flag for human review before the next session message is processed
}
```

For a production health-data agent, you could introduce a governance agent. A second, simpler LLM call that reviews the primary agent's output for safety compliance before it's sent to the user. This adds latency and cost, but for high-risk domains (health, finance, legal), the validation cost is trivial compared to the liability of propagating bad advice. Human-in-the-loop gates (e.g., requiring approval for conversations that exceed a certain message count or contain flagged content) provide the strongest guarantee but only scale for genuinely high-risk actions.

## Rate Limiting and Monitoring

*"Detect fast-spreading commands and throttle or pause on anomalies."*

Rate limiting and monitoring are the detection and containment layer. Even with all other controls in place, you need the ability to detect when something unusual is happening and throttle or pause before a cascade spreads.

Biotrackr implements rate limiting at the APIM boundary and comprehensive monitoring via OpenTelemetry, with Cosmos DB diagnostic logging providing infrastructure-level anomaly detection.

APIM subscription quotas act as a budget ceiling that's completely independent of the agent code. Even if every resilience layer in the application fails, APIM will still enforce rate limits:

```xml
<!-- APIM policy — rate limiting independent of agent behavior -->
<rate-limit-by-key calls="100" renewal-period="60"
    counter-key="@(context.Subscription.Id)" />
```

OpenTelemetry captures the full request chain for anomaly detection:

```csharp
// Program.cs — OpenTelemetry setup
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()     // Incoming requests (AG-UI SSE endpoint)
        .AddHttpClientInstrumentation()      // Outgoing requests (APIM tool calls)
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

This captures the full request chain: user message → streaming middleware → agent → tool call → APIM → downstream API → Cosmos DB. When a cascade happens, the trace shows exactly where the chain broke.

Container App health probes provide automated failure detection and recovery:

```bicep
// infra/apps/chat-api/main.bicep — liveness probes detect cascading failures
healthProbes: [
  {
    type: 'Liveness'
    httpGet: {
      port: 8080
      path: '/healthz/liveness'
    }
    initialDelaySeconds: 15
    periodSeconds: 30
    failureThreshold: 3
    timeoutSeconds: 1
  }
]
```

Some key points here:

- **APIM rate limiting** — enforced externally, independent of agent code. The agent can't bypass this. If the subscription hits its rate limit, all tool calls get 429s, the circuit breaker opens, and the agent degrades gracefully
- **Distributed tracing** — OpenTelemetry traces span the full request chain, making cascade propagation visible
- **Health probes** — Container App liveness probes detect unresponsive containers and restart them automatically after 3 consecutive failures
- **Defence in depth** — APIM handles external rate limiting, `AddStandardResilienceHandler()` handles transient failures, and health probes handle container-level failures

What's missing is application-level rate limiting and anomaly alerting. The current setup detects anomalies in hindsight (via traces and logs) but doesn't automatically throttle or pause when anomalies are detected in real-time. Azure Monitor alerts could trigger on suspicious patterns:

```kql
// Recommended: KQL alert for cascade indicators
// Alert when tool call error rate exceeds 50% in a 5-minute window
AppRequests
| where TimeGenerated > ago(5m)
| where Url contains "/activity" or Url contains "/sleep" or Url contains "/weight" or Url contains "/food"
| summarize TotalCalls = count(), FailedCalls = countif(ResultCode >= 400) by bin(TimeGenerated, 1m)
| where FailedCalls * 1.0 / TotalCalls > 0.5
| project TimeGenerated, TotalCalls, FailedCalls, ErrorRate = round(FailedCalls * 100.0 / TotalCalls, 1)
```

An application-level rate limiter could detect and throttle fast-spreading tool calls within a single session, like a session that triggers 50 tool calls in a minute, which is not normal conversational behavior.

## Blast-Radius Guardrails

*"Implement blast-radius guardrails such as quotas, progress caps, circuit breakers between planner and executor."*

Blast-radius guardrails limit the damage when a cascade does occur. The goal isn't just to prevent cascades, it's to ensure that when things go wrong, the impact is bounded and predictable.

Biotrackr implements blast-radius guardrails through circuit breakers, container resource limits, input validation caps, and caching. Token budget and tool call counters are architecturally supported but not yet implemented.

The circuit breaker in `AddStandardResilienceHandler()` is the primary blast-radius control. When APIM is consistently failing, the circuit opens and tool calls fail immediately:

Let's walk through a concrete cascade scenario to see the circuit breaker in action.

**Trigger**: The Activity API's underlying Cosmos DB returns 429 (too many requests).

**Without controls**:
1. Tool call 1: 429 → agent sees error → retries with same parameters
2. Tool call 2: 429 → agent sees error → tries different parameters
3. Tool call 3–10: more 429s → Claude receives error messages → tries to analyze anyway with partial data
4. Token consumption: 50 tool calls, full conversation context sent to Claude each time
5. Result: ~$2 in tokens, 30-second timeout, user sees an error
6. User retries → the whole cycle starts again

**With controls**:
1. Tool call 1: 429 → `AddStandardResilienceHandler` retries with exponential backoff
2. Tool call 2: 429 → second retry (backoff increased)
3. Tool call 3: 429 → third retry (backoff increased further)
4. Circuit breaker opens → all subsequent tool calls to APIM fail immediately
5. Tool returns: `{"error": "Activity data temporarily unavailable."}`
6. Agent relays to user: *"I'm having trouble fetching your activity data right now. Please try again in a few minutes."*
7. Total cost: 3 API calls + 1 Claude exchange → ~$0.01

The difference is orders of magnitude both in cost and in user experience.

Container resource limits provide a hard ceiling on compute consumption:

```bicep
// infra/apps/chat-api/main.bicep — resource constraints
resources: {
  cpu: json('0.25')    // 0.25 vCPU — limits compute abuse
  memory: '0.5Gi'      // 512MB — prevents memory exhaustion
}
```

Input validation caps prevent the agent from requesting unbounded data ranges:

```csharp
// ActivityTools.cs — progress cap on data range queries
if ((endDate.ToDateTime(TimeOnly.MinValue) - startDate.ToDateTime(TimeOnly.MinValue)).Days > 365)
    return """{"error": "Date range cannot exceed 365 days."}""";

// PaginationRequest.cs — page size cap prevents unbounded queries
public int PageSize { get; set; } = 20;  // Max: 100, enforced via validation
```

The cache also serves as a **redundancy eliminator**. If the agent is tricked (via prompt injection) into calling the same tool 10 times with the same parameters, only the first call hits the API. The rest are served from cache. This limits both the cost impact and the load on downstream services during an attack.

Some key points here:

- **Circuit breaker** — after enough failures in a window, tool calls fail fast. The agent gets an immediate error instead of burning through retries
- **Container resource cap** — 0.25 vCPU and 512MB per replica. Even a runaway agent can't exhaust the host
- **Input validation caps** — date ranges capped at 365 days, page sizes capped at 100, date formats validated before API calls
- **Cache as deduplication** — repeated identical tool calls serve from cache, limiting cascading load on downstream APIs

What's missing is a per-session token budget circuit breaker and a per-session tool call counter. These would provide explicit quotas per conversation:

```csharp
// Recommended: per-session tool call budget in ConversationPersistenceMiddleware
if (toolCalls.Count > MaxToolCallsPerSession)
{
    await repository.SaveMessageAsync(sessionId, "assistant",
        "I've reached the maximum number of data queries for this conversation. " +
        "Please start a new conversation to continue.");
    yield break;
}

// Recommended: per-session token budget
if (cumulativeTokens > MaxTokensPerSession)
{
    // Yield a final message: "This conversation has reached its analysis limit."
    yield break;
}
```

For a side project, the combination of circuit breakers and caching keeps costs manageable. But if you're running a multi-tenant agent, per-session quotas become essential. One user's prompt injection shouldn't eat into everyone else's quota.

## Behavioral and Governance Drift Detection

*"Track decisions vs baselines and alignment; flag gradual degradation."*

Cascading failures don't always start with a bang. Sometimes they start with a gradual drift. The agent starts making slightly different tool call patterns, response quality degrades incrementally, or error rates creep up slowly enough that no single event triggers an alert. Drift detection is about establishing baselines and flagging when behavior diverges.

Biotrackr captures the data needed for drift detection through its middleware audit trail and structured logging, but does not currently implement baseline comparison or drift alerts.

The `ConversationPersistenceMiddleware` creates a per-session audit trail of every tool call, providing the raw data for behavioral analysis:

```csharp
// ConversationPersistenceMiddleware.cs — tool call audit trail
var toolCalls = new List<string>();

await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
{
    foreach (var content in update.Contents)
    {
        if (content is FunctionCallContent functionCall)
        {
            toolCalls.Add(functionCall.Name);
        }
    }
    yield return update;
}

// Persisted to Cosmos DB with the assistant response
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);

logger.LogInformation("Persisted assistant response for session {SessionId} ({ToolCount} tool calls)",
    sessionId, toolCalls.Count);
```

When you see a conversation with 15 tool calls (normal is 1–3 per turn), that's an early indicator of drift or a cascade. Structured logging with session context enables querying for unusual patterns:

```csharp
// ChatHistoryRepository.cs — structured logging with session context
_logger.LogInformation("Saving {Role} message to conversation {SessionId}", role, sessionId);
_logger.LogInformation("Saved message to conversation {SessionId}, total messages: {Count}",
    sessionId, conversation.Messages.Count);
```

Some key points here:

- **Tool call counts per turn** — persisted in Cosmos DB and logged, providing the raw data for baseline comparison
- **Message counts per session** — logged on every save, allowing detection of sessions that grow abnormally large
- **Structured logging** — session IDs, role, and tool counts in structured format enable Log Analytics queries across all sessions

What's missing is baseline definition and drift alerting. The data is captured, but nobody is watching it. Establishing baselines (e.g., "average tool calls per turn is 2.1, standard deviation is 0.8") and alerting when behavior deviates would catch gradual degradation before it cascades:

```kql
// Recommended: KQL query for behavioral drift detection
// Detect sessions where tool call patterns deviate from baseline
AppLogs
| where Message contains "tool calls"
| parse Message with * "(" ToolCount:int " tool calls)"
| summarize AvgToolCalls = avg(ToolCount), MaxToolCalls = max(ToolCount),
    P95ToolCalls = percentile(ToolCount, 95) by bin(TimeGenerated, 1h)
| where P95ToolCalls > 5  // Baseline: 95th percentile should be ≤ 5 tool calls
| project TimeGenerated, AvgToolCalls, MaxToolCalls, P95ToolCalls
```

For governance drift, you'd also want to track whether the agent's responses are consistently following system prompt constraints over time. A periodic evaluation that sends test prompts to the agent and validates the responses against expected behavior (e.g., "does the agent still include the healthcare provider disclaimer?") would catch drift in alignment. This becomes critical when models are updated. A new Claude version might subtly change how the agent interprets tool results.

## Digital Twin Replay and Policy Gating

*"Re-run the last week's recorded agent actions in an isolated clone of the production environment to test whether the same sequence would trigger cascading failures. Gate any policy expansion on these replay tests passing predefined blast-radius caps before deployment."*

Digital twin replay is the most advanced control. A recording agent actions in production and replaying them in an isolated environment to validate that policy changes or infrastructure updates don't introduce new cascade risks.

Biotrackr does not implement digital twin replay, but the architecture captures enough data to support it, and CI/CD pipelines already enforce a lighter form of pre-deployment validation. (This would cost money though, and I'm too cheap to implement this just for the sake of a side project!)

The conversation history in Cosmos DB contains a full record of every user message, assistant response, and tool call sequence. This is the raw material for replay:

```csharp
// ChatConversationDocument.cs — full conversation record suitable for replay
public class ChatConversationDocument
{
    [JsonPropertyName("sessionId")]
    public string SessionId { get; set; } = string.Empty;

    [JsonPropertyName("lastUpdated")]
    public DateTime LastUpdated { get; set; } = DateTime.UtcNow;

    [JsonPropertyName("messages")]
    public List<ChatMessage> Messages { get; set; } = [];
    // Each message includes: role, content, timestamp, toolCalls list
}
```

CI/CD already enforces infrastructure validation before deployment which can act as a lighter form of policy gating:

```yaml
# deploy-chat-api.yml — pre-deployment validation pipeline
lint-bicep:
    name: Lint Bicep Template  # Static analysis of IaC

validate-bicep:
    name: Validate Bicep Template  # ARM template validation

what-if-bicep:
    name: What-If Bicep Template  # Preview infrastructure changes before apply
```

Some key points here:

- **Conversation records as replay data** — every session's message history, tool calls, and timestamps are persisted in Cosmos DB, providing the input data for replay testing
- **CI/CD validation** — Bicep linting, ARM template validation, and what-if previews gate infrastructure changes before deployment
- **Version control** — system prompt, tool definitions, and IaC are all version-controlled in Git with PR review

What's missing is the full replay infrastructure. A production-ready implementation would:

1. **Export recent agent sessions** — query Cosmos DB for conversations from the last week, including tool call sequences
2. **Spin up an isolated clone** — deploy a staging Container App with the proposed changes (new model version, updated system prompt, modified policies)
3. **Replay conversations** — feed the recorded user messages into the staging agent and capture the new responses and tool call patterns
4. **Compare blast-radius metrics** — compare tool call counts, error rates, token usage, and response quality between production and staging runs
5. **Gate deployment** — only promotion to production if replay metrics fall within predefined caps (e.g., "tool calls per turn must not increase by more than 20%")

```csharp
// Conceptual: replay test for blast-radius validation
[Fact]
public async Task ReplayLastWeek_ShouldNotExceedBlastRadiusCaps()
{
    // Arrange: load last week's conversations from Cosmos DB
    var conversations = await repository.GetConversationsSince(DateTime.UtcNow.AddDays(-7));

    foreach (var conversation in conversations)
    {
        // Act: replay user messages through the agent with proposed changes
        var replayResult = await replayEngine.ReplayConversation(conversation, newAgent);

        // Assert: blast-radius caps
        Assert.True(replayResult.ToolCallsPerTurn <= MaxToolCallsPerTurn * 1.2);
        Assert.True(replayResult.TotalTokens <= MaxTokensPerSession);
        Assert.True(replayResult.ErrorRate <= 0.05);
    }
}
```

This is an advanced control that makes the most sense for production agents handling high-value workflows. For a side project, the CI/CD validation pipeline and version-controlled configuration provide a reasonable lightweight approximation.

## Logging and Non-Repudiation

*"Record all inter-agent messages, policy decisions, and execution outcomes in tamper-evident, time-stamped logs bound to cryptographic agent identities. Maintain lineage metadata for every propagated action to support forensic traceability, rollback validation, and accountability during cascades."*

When a cascade occurs, you need to reconstruct exactly what happened, in what order, and who (or what) caused it. Logging and non-repudiation ensure that every decision, tool call, and outcome is recorded with enough metadata to support forensic analysis.

Biotrackr implements comprehensive logging through multiple layers: application-level structured logging, conversation persistence with tool call metadata, OpenTelemetry distributed tracing, and Cosmos DB diagnostic logging. The agent identity provides a cryptographic binding to all operations.

Structured application logging captures session-scoped operations with traceability:

```csharp
// ChatHistoryRepository.cs — structured logging with session context
_logger.LogInformation("Saving {Role} message to conversation {SessionId}", role, sessionId);
_logger.LogInformation("Saved message to conversation {SessionId}, total messages: {Count}",
    sessionId, conversation.Messages.Count);
```

Every assistant response is persisted with a full tool call audit trail:

```csharp
// ConversationPersistenceMiddleware.cs — tool call lineage
var toolCalls = new List<string>();

await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
{
    foreach (var content in update.Contents)
    {
        if (content is FunctionCallContent functionCall)
        {
            toolCalls.Add(functionCall.Name);
        }
    }
    yield return update;
}

// Persisted: which tools were called, when, in which session
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);

logger.LogInformation("Persisted assistant response for session {SessionId} ({ToolCount} tool calls)",
    sessionId, toolCalls.Count);
```

Every message has a timestamp and role attribution, providing a timeline for forensic reconstruction:

```csharp
// ChatMessage.cs — provenance metadata on every message
public class ChatMessage
{
    [JsonPropertyName("role")]
    public string Role { get; set; } = string.Empty;  // "user" or "assistant"

    [JsonPropertyName("timestamp")]
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    [JsonPropertyName("toolCalls")]
    public List<string>? ToolCalls { get; set; }  // Tool names invoked in this turn
}
```

OpenTelemetry provides distributed tracing across the full request chain:

```csharp
// Program.cs — OpenTelemetry for distributed tracing
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()     // Incoming SSE requests
        .AddHttpClientInstrumentation()      // Outgoing APIM tool calls
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

Cosmos DB diagnostic logging captures all data plane operations for infrastructure-level audit:

```bicep
// serverless-cosmos-db.bicep — data plane logging
logs: [
  { category: 'DataPlaneRequests', enabled: true }     // All read/write operations
  { category: 'QueryRuntimeStatistics', enabled: true } // Query performance
  { category: 'ControlPlaneRequests', enabled: true }   // Management operations
]
```

The agent identity provides a cryptographic binding. All Cosmos DB operations are authenticated via Entra Agent ID with Federated Identity Credentials, meaning every operation in the audit log is bound to a verifiable identity:

```csharp
// AgentIdentityCosmosClientFactory.cs — cryptographic identity binding
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
// All Cosmos DB operations are authenticated under this identity
// Entra ID audit logs record which identity performed each operation
```

Some key points here:

- **Message-level provenance** — every message has role, timestamp, and tool call list. A forensic investigator can reconstruct the exact sequence of events in a conversation
- **Distributed tracing** — OpenTelemetry traces span the full request chain (user → agent → tool → APIM → API → Cosmos DB), enabling correlation of failures across services
- **Infrastructure audit logs** — Cosmos DB data plane requests are logged to Log Analytics, providing an independent record of all database operations
- **Cryptographic identity** — Entra Agent ID with FIC provides a verifiable, non-repudiable identity for all agent operations. The platform rotates credentials automatically

What's missing is tamper-evident logging. Currently, application logs are written to standard output and collected by the Container App platform. An attacker with access to the logging infrastructure could theoretically modify or delete logs. For true non-repudiation, logs should be written to an immutable storage backend:

```bicep
// Recommended: immutable blob storage for tamper-evident audit logs
resource auditStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: auditStorageName
  properties: {
    immutableStorageWithVersioning: {
      enabled: true  // Write-once, read-many — logs cannot be modified or deleted
    }
  }
}
```

There's also no lineage metadata for propagated actions. When the agent calls `GetActivityByDate`, the tool result influences the assistant's response, which is then persisted and potentially loaded in a future session. Currently, there's no explicit link between "this response was influenced by tool call X which returned data from API Y." A lineage graph that tracks `user_message → tool_call → api_response → assistant_response → persistence` would support root cause analysis during cascades.

For a production multi-agent system, you'd also want signed log entries (each log entry cryptographically signed by the agent's identity) and append-only log storage to ensure that post-incident analysis is reliable and legally defensible.

## Wrapping up

Cascading Failures (ASI08) is about making your agent resilient to the inevitable: dependencies will fail. The question is whether a single failure becomes a $0.01 graceful degradation or a $2000.00 cascading meltdown (or more!).

The controls are layered: zero-trust fault tolerance (`AddStandardResilienceHandler()` + structured errors + caching) → isolation boundaries (APIM + least-privilege identity + container sandbox) → blast-radius guardrails (circuit breakers + input caps + resource limits) → monitoring and detection (OpenTelemetry + health probes + structured logging) → forensic traceability (conversation audit trail + distributed traces + diagnostic logs). Even if one layer fails, the others contain the blast radius.

That's the key takeaway here. **Build resilience at every layer of the agent's dependency chain, because a single-point-of-failure in an agent system doesn't just affect one feature. It amplifies across every tool call, every retry, and every token.**

`AddStandardResilienceHandler()` is the single highest-value line of code for your agent's resilience. If you take nothing else away from this post, add that one line to your HttpClient registration. Structured error responses are the second most impactful. They prevent the agent from entering confused retry states that amplify costs.

In the next post in this series, I'll cover **ASI09 — Human-Agent Trust Exploitation**, which is about what happens when users over-trust the agent's outputs and make decisions based on AI-generated analysis without verification. Many of the controls we've discussed here (structured error responses, output validation, observability) help users understand when the agent's outputs might be unreliable.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
