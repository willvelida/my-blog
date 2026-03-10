---
title: "Preventing Tool Misuse in AI Agents"
date: 2026-03-11
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "APIM", "Managed Identity"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j00h7d9jpmrquhve9pd7.png
    alt: "Preventing OWASP ASI02 Tool Misuse in a .NET AI agent with date range limits, page size caps, read-only tools, egress controls, and managed identity."
    caption: "Implementing OWASP ASI02 mitigations against Tool Misuse and Exploitation in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

In my side project (Biotrackr), I have a chat agent that I use to query my data using natural language. This agent has 12 tools that call APIs to retrieve data that provides context to a LLM. I'm using Claude as my LLM provider, so Claude will decide which tool to call, and with what parameters.

Let's pretend that we are bad actors trying to disrupt my agent. Say we decide to prompt inject the agent, and get it to perform an expensive query to retrieve 100 years of data (I'm not that old thankfully!) in an attempt to return a massive payload, consume thousands of Claude API tokens, and hammer my APIM gateway.

This attack is called **Tool misuse** (ASI02) and it's about constraining what an autonomous agent can do with its tools. Essentially, we want to apply the principle of least privilege to function calling.

In this blog post, we'll take a deep dive into Tool Misuse and Exploitation, and cover what controls we can implement to prevent and mitigate it using examples from my agent.

## Tool Misuse and Exploitation

In AI Agents, there's a risk that agents can misuse legitimate tools due to prompt injection, misalignment, or unsafe delegation or ambiguous instructions. This can lead to problems like data exfiltration, tool output manipulation or workflow hijacking. 

Further risks can arise from how the agent chooses and applies tools; agent memory, dynamic tool selection, and delegation can all contribute to misuse via chaining, privilege escalation, and unintended actions.

Agents can act within their authorized boundaries, but apply their tools in an unsafe or unintended manner, such as deleting data, over-invoking APIs etc.

A simple example of this could be a customer service bot that has access to financial APIs via its tools. An attacker could get the agent to invoke its access to these tools to issue refunds, without any human intervention.

## Implementing controls for Biotrackr

Why does this matter for my little side project?

The obvious one that comes to mind is financial. This is just a small side project for me to work on my software engineering skills while keeping me informed of my health. Ideally, I don't want to break the bank just running the thing.

APIM, Claude APIs, Azure infrastructure. These all cost money to run, even for a small project like mine. If an attacker was able to perform some clever prompt injection attacks that retrieve large amounts of data, this will start to add up in both Claude API tokens, and Azure spending!

I currently have 1 agent deployed with access to numerous tools across numerous data domains. The attack surface is quite large for an autonomous agent. Each tool call costs Claude API tokens, as well as APIM and Cosmos DB costs.

And this is just a small side project. Have a think about the agents you've deployed in your organization, or the agents you use day-to-day. How many tools does it use? How many data domains does it have access to? How large would the surface area be if a malicious attacker was able to misuse the tools available to the agent?

With all this in mind, let's walkthrough each prevention and mitigation strategy we can implement to prevent tool misuse, with some examples at how I've implemented them in my agent.

## Least Agency and Least Privilege for Tools

OWASP defines this control as:

*"Define per-tool least-privilege profiles (scopes, maximum rate, and egress allowlists) and restrict agentic tool functionality and each tool's permissions and data scope to those profiles — e.g., read-only queries for databases, no send/delete rights for email summarizers, and minimal CRUD operations when exposing APIs."*

One simple way to do this is to **not give the agent access to destructive tools** (such as DELETE, PUT, POST operations). In Biotrackr, all tools are HTTP GET operations. The agent only has the ability to read the data, not modify it.

This is enforced by the tool method signature and the API design, not just stated in the system prompt. Chat history can be deleted, but this isn't handled by the agent tools. So if the agent was hijacked, the worst it can do is read data. It won't be able to delete conversations, modify my data, or alter the application's configuration.

Another method we can implement is to set **hard limits on how much data the tools can interact with**. In Biotrackr, I have a couple of APIs that can retrieve data over a specified date range. Without any limits, the agent could be tricked into fetching a large amount of data within a single call. Here's an example of setting a hard limit of a date range that doesn't exceed 365 days:

```csharp
[Description("Get activity data for a date range. Maximum 365 days. Date format: YYYY-MM-DD.")]
public async Task<string> GetActivityByDateRange(
    [Description("The start date, in YYYY-MM-DD format")] string startDate,
    [Description("The end date, in YYYY-MM-DD format")] string endDate)
{
    if (!DateOnly.TryParse(startDate, out var start) || !DateOnly.TryParse(endDate, out var end))
        return """{"error": "Invalid date format. Use YYYY-MM-DD."}""";

    if ((end.ToDateTime(TimeOnly.MinValue) - start.ToDateTime(TimeOnly.MinValue)).Days > 365)
        return """{"error": "Date range cannot exceed 365 days."}""";

    // ... proceed with validated range
}
```

Our limit is documented in the `[Description]` attribute, so Claude will see "Maximum 365 days" and is less likely to request a range over that. **The validation is also enforced in the code**, so even if Claude ignores or is tricked to ignore the description, the tool itself will reject it. We also return the error message to the user that the date range cannot exceed 365 days.

This can also be applied to page size caps on paginated tools. Paginated records accept a `pageSize` parameter. Without a cap, the agent could request `pageSize=100000000000`, pulling in a massive dataset! 

```csharp
[Description("Get paginated activity records. Returns the most recent records by default.")]
public async Task<string> GetActivityRecords(
    [Description("Page number (default: 1)")] int pageNumber = 1,
    [Description("Page size (default: 10, max: 50)")] int pageSize = 10)
{
    pageSize = Math.Min(pageSize, 50);  // Hard cap at 50
    // ... fetch with capped page size
}
```

We're performing some silent capping, which doesn't throw an error and allows the agent to retrieve the data, but just within a bounded context. 50 records is still a lot of data which we can use for meaningful analysis, but at the same time preventing bulk data extraction.

## Action-Level Authentication and Approval

*"Require explicit authentication for each tool invocation and human confirmation for high-impact or destructive actions (delete, transfer, publish). Display a pre-execution plan or dry-run diff before final approval."*

In Biotrackr, every tool call results in an HTTP request to APIM. Each request carries an `Ocp-Apim-Subscription-Key` header via a delegating handler. This means each tool invocation is individually authenticated at the API gateway, not just the initial user session.

```csharp
public class ApiKeyDelegatingHandler : DelegatingHandler
{
    private const string SubscriptionKeyHeader = "Ocp-Apim-Subscription-Key";
    private readonly string? _subscriptionKey;

    public ApiKeyDelegatingHandler(IOptions<Settings> settings)
    {
        _subscriptionKey = settings.Value.ApiSubscriptionKey;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        if (!string.IsNullOrWhiteSpace(_subscriptionKey))
        {
            request.Headers.TryAddWithoutValidation(SubscriptionKeyHeader, _subscriptionKey);
        }

        return await base.SendAsync(request, cancellationToken);
    }
}
```

Every tool call passes through APIM authentication, meaning that the LLM cannot bypass this. Using APIM, we can enforce per-subscription rate limits, quota policies, and request validation that's independent of the agent's behavior.

If the subscription key is revoked, all tool calls fail immediately, which can be used as a kill switch for the agent's external access.

Now in my example, we're just making GET requests. Your agents might have more destructive tools available that can modify data. This isn't necessarily bad, but I'd recommend adding human-in-the-loop mechanisms to ensure that bad actors can't manipulate these tools.

## Execution Sandboxes and Egress Controls

*"Run tool or code execution in isolated sandboxes. Enforce outbound allowlists and deny all non-approved network destinations."*

When I think about sandboxing, I think about agent code execution (running untrusted Python, shell commands, that kind of thing). My agent doesn't do any of that. All it does is make HTTP calls to my APIM gateway. So full-blown container sandboxing (gVisor, Firecracker) would be overkill here. What I can do instead is constrain *where* those HTTP calls can go.

Biotrackr constrains egress at the HttpClient level: all tools share a single named `HttpClient` with a fixed `BaseAddress`. Tools cannot make arbitrary outbound HTTP calls, they can only reach the configured APIM endpoint.

```csharp
builder.Services.AddHttpClient("BiotrackrApi", (sp, client) =>
{
    var settings = sp.GetRequiredService<IOptions<Settings>>().Value;
    client.BaseAddress = new Uri(settings.ApiBaseUrl);  // Single allowed destination
})
.AddHttpMessageHandler<ApiKeyDelegatingHandler>()
.AddStandardResilienceHandler();
```

Here, the `BaseAddress` is set once at startup. Tools only append the relative paths, and cannot change the destination. No tool can create its own `HttpClient`, all HTTP access goes through the factory.

This is a lightweight implementation of an allowlist, where one allowed outbound destination is the APIM gateway, and everything else is unreachable.

The tools themselves are not capable of executing arbitrary code, as they are static method implementations that are registered at startup. We could even combine this with networking controls for the compute, like applying outbound traffic rules for our Chat API.

## Policy Enforcement Middleware ("Intent Gate")

*"Treat LLM or planner outputs as untrusted. A pre-execution Policy Enforcement Point (PEP/PDP) validates intent and arguments, enforces schemas and rate limits, issues short-lived credentials, and revokes or audits on drift."*

This is something that I haven't formally implemented. Instead, what I've done is use input validation inside each tool method. So every tool will validate data formats via `DateOnly.TryParse()`, enforces range limits, and caps page sizes before making any API call. The LLM's outputs are treated as untrusted at the tool level.

```csharp
// Every tool validates inputs before executing — LLM output is never trusted
if (!DateOnly.TryParse(date, out _))
    return """{"error": "Invalid date format. Use YYYY-MM-DD."}""";
```

To address this guideline further, you can implement centralized middleware that intercepts **all** tool calls before execution, which would be more robust than just implementing validation in each tool.

This middleware could validate arguments against a JSON schema before the tool runs, enforce per-session rate limits per conversation (so only 20 tools could be called in an entire conversation), log full tool arguments (not just the name of the tool that was called) for auditing, and classify the intent to ensure that tools that don't match the current conversation context are rejected.

For a personal health project with 12 read-only tools, the distributed validation approach is pragmatic enough. But if your agent has write access to databases or can trigger external workflows, a centralized policy gate becomes much more important. You don't want to rely on every tool developer remembering to add validation.

## Adaptive Tool Budgeting

*"Apply usage ceilings (cost, rate, or token budgets) with automatic revocation or throttling when exceeded."*

Again, this isn't something that I've implemented explicitly, but there are a couple of things I have done to potentially limit the impact of volume abuse.

### In-Memory Caching (Redundancy Elimination)

If the agent is tricked into calling the same tool 10 times with the same parameters, only the first call hits the API. The rest are served from cache.

```csharp
var cacheKey = $"activity:{date}";
if (cache.TryGetValue(cacheKey, out string? cached))
    return cached!;

// ... fetch from API ...

var ttl = DateOnly.Parse(date) == DateOnly.FromDateTime(DateTime.UtcNow)
    ? TimeSpan.FromMinutes(5)    // Today's data — may still be syncing
    : TimeSpan.FromHours(1);     // Historical — stable
cache.Set(cacheKey, result, ttl);
```

Some key points here:

- Cache key strategy: `{domain}:{date}` for by-date, `{domain}:{startDate}:{endDate}` for ranges, `{domain}-records:{page}:{size}` for pagination
- Zero infrastructure cost — `IMemoryCache` is in-process, per-Container App instance
- Adaptive TTL: today's data (5 min), historical (1 hour), ranges (30 min), paginated records (15 min)

### Circuit Breaker + Resilience Handlers (Failure Containment)

If tools start failing (APIM rate limit, downstream API outage), the agent might keep retrying. Each retry costs Claude API tokens. The standard resilience handler prevents cascading failures:

```csharp
.AddStandardResilienceHandler();  // Retry + circuit breaker + timeout
```

- 3 retries with exponential backoff, circuit breaker after 5 failures, 30-second timeout
- When the circuit is open, tool calls fail fast — the agent gets an error immediately instead of waiting
- APIM subscription quotas act as an external budget ceiling independent of the agent code

**What's missing:**

A per-session tool call counter that returns an error after a threshold (e.g., 20 tool calls per conversation). This would limit the blast radius of a prompt injection that tries to exhaust the API budget by calling different tools with different parameters (bypassing the cache).

For my use case, the caching and circuit breaker combination keeps costs manageable. But if you're running a multi-tenant agent where many users share the same infrastructure, per-session budgets become essential. One user's prompt injection shouldn't eat into everyone else's quota.

## Just-in-Time and Ephemeral Access

*"Grant temporary credentials or API tokens that expire immediately after use. Bind keys to specific user sessions to prevent lateral abuse."*

Biotrackr uses Azure Managed Identity for Cosmos DB access via the `MicrosoftIdentityTokenCredential` through Microsoft Entra Agent ID. There are no stored credentials, no connection strings. Tokens are acquired and rotated automatically by the Azure identity platform.

```csharp
public class AgentIdentityCosmosClientFactory : ICosmosClientFactory
{
    public CosmosClient Create()
    {
        _credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
        _credential.Options.RequestAppToken = true;

        return new CosmosClient(_settings.CosmosEndpoint, _credential, new CosmosClientOptions
        {
            SerializerOptions = new CosmosSerializationOptions
            {
                PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
            }
        });
    }
}
```

**Key points:**
- No Cosmos DB connection string or API key in configuration. Managed Identity only
- Agent Identity scoping via `.WithAgentIdentity()` ensures the Cosmos client operates with constrained RBAC permissions, not full account access
- `RequestAppToken = true` enables the autonomous agent pattern. The agent gets its own identity, separate from the user.

For a single-user side project, Managed Identity covers the credential risk well. If you're building for multiple users, you'd want per-session tokens that expire when the conversation ends. Even if a session is compromised, the blast radius is bounded to that one conversation.

## Semantic and Identity Validation ("Semantic Firewalls")

*"Enforce fully qualified tool names and version pins to avoid tool alias collisions or typosquatted tools; validate the intended semantics of tool calls rather than relying on syntax alone. Fail closed on ambiguous resolution."*

In Biotrackr, I've avoided this entirely as all of my tools are registered by direct method reference in `Program.cs` at startup. So there's no dynamic tool discovery, no plugin loading, no tool name resolution. The tool set is fixed at compile time.

```csharp
AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,
    tools:
    [
        AIFunctionFactory.Create(activityTools.GetActivityByDate),
        AIFunctionFactory.Create(activityTools.GetActivityByDateRange),
        AIFunctionFactory.Create(activityTools.GetActivityRecords),
        AIFunctionFactory.Create(sleepTools.GetSleepByDate),
        AIFunctionFactory.Create(sleepTools.GetSleepByDateRange),
        AIFunctionFactory.Create(sleepTools.GetSleepRecords),
        AIFunctionFactory.Create(weightTools.GetWeightByDate),
        AIFunctionFactory.Create(weightTools.GetWeightByDateRange),
        AIFunctionFactory.Create(weightTools.GetWeightRecords),
        AIFunctionFactory.Create(foodTools.GetFoodByDate),
        AIFunctionFactory.Create(foodTools.GetFoodByDateRange),
        AIFunctionFactory.Create(foodTools.GetFoodRecords),
    ]);
```

Some key points here:

- No string-based tool names that could be spoofed or collided. Tools are registered via `AIFunctionFactory.Create()` with direct method references.
- No dynamic tool loading from external sources.
- The tool set is auditable in a single location (`Program.cs`)
- Adding a new tool requires a code change, build, and deployment, not a runtime configuration change.
- If a future version adopted MCP or a plugin system, this would need revisiting with version pinning and signature verification.

## Logging, Monitoring, and Drift Detection

*"Maintain immutable logs of all tool invocations and parameter changes. Continuously monitor for anomalous execution rates, unusual tool-chaining patterns (e.g., DB read followed by external transfer), and policy violations."*

Biotrackr logs tool invocations through the `ConversationPersistenceMiddleware`, which intercepts the agent's streaming response and captures which tools were called:

```csharp
// In ConversationPersistenceMiddleware
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

// Persist to Cosmos DB with tool call names
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);

logger.LogInformation("Persisted assistant response for session {SessionId} ({ToolCount} tool calls)",
    sessionId, toolCalls.Count);
```

OpenTelemetry tracing (`AddAspNetCoreInstrumentation()` + `AddHttpClientInstrumentation()`) provides distributed traces correlating user messages → tool calls → API calls → Cosmos reads. HttpClient logging via the resilience handler captures request URL, status code, and duration. APIM analytics provide request volume, latency distribution, and error rates at the gateway level.

To strengthen this further, we could log the tool call *arguments*. Full argument logging would enable detection of suspicious parameter patterns (e.g. repeated max-range date queries). The trick here is to balance that concern against privacy concerns. This is a health application after all, and the parameters could contain sensitive information.

## Wrapping up

Tool Misuse (ASI02) is about constraining autonomy, giving the agent enough freedom to be useful, but enforcing hard limits to prevent abuse. The OWASP specification defines 8 prevention and mitigation guidelines, and in Biotrackr, we implement 5 fully and 3 partially.

The controls are layered: read-only tools → input validation → egress constraints → caching → circuit breakers → per-request auth → logging. Even if one layer fails, the others limit the damage. That's the key takeaway here. **Treat your agent's tool calls like you'd treat external API requests: authenticate, validate, constrain, and log every one.**

There are gaps I haven't addressed yet. A centralized policy enforcement middleware (Guideline 4), per-session tool call budgets (Guideline 5), and full argument logging (Guideline 8) are all on the backlog. If you're building agents with write access to databases, external APIs, or email systems, those controls become critical rather than nice-to-have.

In the next post in this series, I'll cover **ASI03 — Identity and Privilege Abuse**, which is what happens when an agent's identity is exploited to access resources beyond its intended scope. Many of the controls we've discussed here (Managed Identity, APIM authentication, static tool registration) are the first line of defence against that too.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️