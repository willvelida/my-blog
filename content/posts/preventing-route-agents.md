---
title: "Preventing Rogue AI Agents"
date: 2026-03-12
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "Entra Agent ID", "OpenTelemetry"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j4kh8zjxqmxbeoe0gxg2.png
    alt: "Preventing OWASP ASI10 Rogue Agents in a .NET AI agent with behavioural constraints, kill switches, audit logging, immutable tools, and defence in depth."
    caption: "Implementing OWASP ASI10 mitigations against Rogue Agents in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

What happens when the agent itself becomes the threat? Not because of a prompt injection (ASI01) or tool misuse (ASI02), but because the Claude model produces systematically wrong analysis, the Agent Framework has a bug in its tool loop, or the Anthropic API starts returning manipulated responses?

Throughout this series, we've covered controls that protect the agent from external threats (hijacked goals, misused tools, stolen identities, supply chain poisoning, code execution, context poisoning, cascading failures, and trust exploitation). But what do you do when everything else fails and the agent itself starts behaving in ways you didn't intend?

For my side project (Biotrackr), this is the "what if everything breaks?" scenario. The agent is designed to be a helpful health data assistant, but if the underlying model drifts, the framework has a bug, or a dependency is compromised, the agent could start producing harmful analysis, calling tools excessively, or leaking system internals — all without a single prompt injection attack.

Rogue Agents (ASI10) is about designing for containment. It's the set of controls that kick in when the agent deviates from its intended behaviour, and how you minimise the blast radius before you even notice something is wrong.

In this post, I'll walk through what Rogue Agents are, why they matter even for a small side project, and how we can implement controls to detect, contain, and recover from rogue agent behaviour using Biotrackr as an example.

ASI10 builds on the containment and resilience themes from ASI08 (Cascading Failures), but shifts the focus from external service failures to the agent itself becoming unreliable. The OWASP specification defines 7 prevention and mitigation guidelines. Let's walk through each one and see how Biotrackr implements (or could implement) them.

## What is a Rogue Agent?

The OWASP definition describes rogue agents as "AI agents that deviate from their intended behaviour due to misconfiguration, prompt injection, model issues, or compromised components."

The key difference from the other OWASP Agentic threats is that rogue agent behaviour may not be caused by a specific attack. It could be emergent. This makes it broader than prompt injection (ASI01) because the cause might not be adversarial at all.

There are several ways an agent can go rogue:

1. **Model drift** — the model's behaviour changes between API versions. Anthropic updates Claude, and suddenly the agent's tool-calling heuristics shift.
2. **Framework bugs** — the Microsoft Agent Framework is still in preview. A bug in the tool loop could cause the agent to call tools in unexpected sequences.
3. **Compromised API** — the Claude API itself starts returning adversarial outputs, either due to a supply chain attack or a misconfiguration on the provider's side.
4. **Configuration drift** — the system prompt or tool definitions are accidentally modified during a deployment, and the agent's behaviour changes silently.

The scary part? In all four scenarios, the agent is technically "working." It's calling tools, returning responses, and maintaining conversations. It's just not doing what you intended.

## Why does this matter for Biotrackr?

Even for my little side project, rogue agent behaviour could have real consequences.

If the Claude model changes how it interprets my system prompt, the agent might start calling all 12 tools for every user message instead of selecting the relevant ones. That's a cost explosion, every tool call costs Claude API tokens, APIM requests, and Cosmos DB reads.

If the agent starts producing systematically wrong health analysis. For example, consistently underestimating my calorie intake or misinterpreting my sleep data, that's not just an annoyance, it's potentially harmful advice.

And if the agent starts leaking system internals (system prompt fragments, API keys, configuration details) in its responses, that's a security incident, even if no attacker caused it.

The thing is, I might not immediately notice any of these. If the agent is still responding to my questions and the responses look plausible, the deviation could go undetected for days or weeks.

With that in mind, let's walk through each prevention and mitigation strategy we can implement to detect, contain, and recover from rogue agent behaviour, with some examples of how I've implemented them in my agent.

## Governance and Logging

*"Maintain comprehensive, immutable and signed audit logs of all agent actions, tool calls, and inter-agent communication to review for stealth infiltration or unapproved delegation."*

You can't detect rogue behaviour if you're not watching. This might seem obvious, but it's easy to deploy an agent and assume it'll keep working as intended. Detection requires three things to be logged for every agent interaction: what the user asked, which tools were called, and what the agent said back.

In Biotrackr, I've implemented logging at multiple layers. Conversation persistence for the application-level audit trail, OpenTelemetry for the infrastructure-level trace, and Cosmos DB diagnostics for data plane operations.

The conversation persistence middleware intercepts the agent's streaming response and captures which tools were called:

```csharp
// ConversationPersistenceMiddleware.cs — captures tool calls and persists for audit
var responseText = new StringBuilder();
var toolCalls = new List<string>();

await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
{
    foreach (var content in update.Contents)
    {
        if (content is TextContent textContent)
        {
            responseText.Append(textContent.Text);
        }
        else if (content is FunctionCallContent functionCall)
        {
            toolCalls.Add(functionCall.Name);  // Track which tools the agent calls
        }
    }
    yield return update;
}

// Persist assistant response with tool call metadata
await repository.SaveMessageAsync(
    sessionId, "assistant", responseText.ToString(),
    toolCalls.Count > 0 ? toolCalls : null);

logger.LogInformation(
    "Persisted assistant response for session {SessionId} ({ToolCount} tool calls)",
    sessionId, toolCalls.Count);
```

Every message has a timestamp and role attribution, providing a timeline for forensic reconstruction:

```csharp
// ChatMessage.cs — provenance metadata on every message
public class ChatMessage
{
    [JsonPropertyName("role")]
    public string Role { get; set; } = string.Empty;  // "user" or "assistant"

    [JsonPropertyName("content")]
    public string Content { get; set; } = string.Empty;

    [JsonPropertyName("timestamp")]
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    [JsonPropertyName("toolCalls")]
    public List<string>? ToolCalls { get; set; }  // Tool names invoked in this turn
}
```

OpenTelemetry captures distributed traces across the entire request pipeline, correlating user messages → tool calls → APIM requests → backend API responses in a single distributed trace:

```csharp
// Program.cs — OpenTelemetry configured for full observability
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()    // Inbound: AG-UI requests
        .AddHttpClientInstrumentation()     // Outbound: APIM tool calls
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

Cosmos DB diagnostic logging provides an independent infrastructure-level record of all database operations:

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

- **Three log layers** — application logs (structured logging), conversation persistence (Cosmos DB documents), and infrastructure logs (OpenTelemetry + Cosmos DB diagnostics) provide independent audit trails
- **Message-level provenance** — every message has role, content, timestamp, and tool call list, enabling forensic reconstruction of any conversation
- **Distributed tracing** — OpenTelemetry traces span the full request chain (user → agent → tool → APIM → API → Cosmos DB), making it possible to see exactly where rogue behaviour originated
- **Identity binding** — Entra Agent ID with FIC provides a verifiable, non-repudiable identity for all agent operations

What's missing is immutable and signed logging. Currently, application logs are collected by the Container App platform and conversation data is stored in Cosmos DB, both of which are modifiable by administrators. For true non-repudiation, logs should be written to an append-only storage backend with cryptographic signing:

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

There's also no inter-agent communication logging. But since Biotrackr is a single-agent system, there's nothing to log. For multi-agent systems, every inter-agent message should be captured in a centralized, tamper-evident log with sender and receiver identity attestation, enabling detection of stealth infiltration or unapproved delegation between agents.

## Isolation and Boundaries

*"Assign Trust Zones with strict inter-zone communication rules and deploy restricted execution environments (e.g., container sandboxes) with API scopes based on least privilege."*

Isolation ensures that when an agent goes rogue, the damage stays contained. A failing or compromised agent shouldn't be able to reach services outside its intended scope, escalate its own privileges, or communicate with agents in different trust zones.

Biotrackr implements isolation at multiple levels: container-level resource sandboxing, network boundaries via APIM, least-privilege identity via Entra Agent ID, and a read-only tool set that bounds the agent's capabilities by design.

The most effective way to limit what a rogue agent can do is to limit what the agent can do in the first place. Here's the complete tool inventory, 12 tools, all read-only HTTP GET operations:

```csharp
// Program.cs — the agent's entire capability set, registered at startup
AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,
    tools:
    [
        // ACTIVITY TOOLS (read-only)
        AIFunctionFactory.Create(activityTools.GetActivityByDate),
        AIFunctionFactory.Create(activityTools.GetActivityByDateRange),
        AIFunctionFactory.Create(activityTools.GetActivityRecords),
        // SLEEP TOOLS (read-only)
        AIFunctionFactory.Create(sleepTools.GetSleepByDate),
        AIFunctionFactory.Create(sleepTools.GetSleepByDateRange),
        AIFunctionFactory.Create(sleepTools.GetSleepRecords),
        // WEIGHT TOOLS (read-only)
        AIFunctionFactory.Create(weightTools.GetWeightByDate),
        AIFunctionFactory.Create(weightTools.GetWeightByDateRange),
        AIFunctionFactory.Create(weightTools.GetWeightRecords),
        // FOOD TOOLS (read-only)
        AIFunctionFactory.Create(foodTools.GetFoodByDate),
        AIFunctionFactory.Create(foodTools.GetFoodByDateRange),
        AIFunctionFactory.Create(foodTools.GetFoodRecords),
    ]);
```

It's just as important to note what's *not* in this list. The agent deliberately has no write tools, no web browsing tools, no code execution tools, no agent creation tools, and no file system tools. Even if the agent goes rogue, it can only read health data. The blast radius is bounded by design.

The agent identity provides least-privilege access. Cosmos DB Data Contributor on a single account, not Contributor at the resource group level:

```csharp
// AgentIdentityCosmosClientFactory.cs — agent identity scoped to Cosmos DB Data Contributor
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
// The agent identity has Cosmos DB Data Contributor (role 00000000-0000-0000-0000-000000000002)
// on a single account — it cannot access Key Vault, Storage, or other resources
```

APIM acts as a network boundary, the agent never calls downstream APIs directly. All tool traffic flows through APIM, which enforces authentication and rate limiting independently:

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

Container-level resource limits provide a hardware sandbox:

```bicep
// infra/apps/chat-api/main.bicep — container sandbox
resources: {
  cpu: json('0.25')    // 0.25 vCPU — limits compute abuse
  memory: '0.5Gi'      // 512MB — prevents memory exhaustion
}

ingress: {
  external: true
  targetPort: 8080
  transport: 'http'
  allowInsecure: false  // TLS required — no plaintext HTTP
}
```

The agent is a single instance, it cannot create, configure, or deploy new agents. The Microsoft Agent Framework supports multi-agent workflows, but Biotrackr deliberately uses a single-agent architecture:

```csharp
// Program.cs — single agent, no orchestration, no delegation
var persistentAgent = chatAgent.UseMiddleware(
    (innerAgent, services) =>
    {
        var repository = services.GetRequiredService<IChatHistoryRepository>();
        var loggerFactory = services.GetRequiredService<ILoggerFactory>();
        return new ConversationPersistenceMiddleware(innerAgent, repository, loggerFactory);
    });

app.MapAgentAguiEndpoint("/api/chat", persistentAgent);
// No WorkflowRunner, no AgentOrchestrator, no multi-agent coordination
```

Some key points here:

- **Read-only tool set** — all 12 tools are GET requests. The agent cannot modify data, delete records, or trigger side effects
- **Least-privilege identity** — Cosmos DB Data Contributor on a single account. The agent cannot access Key Vault, Storage, or other resources
- **APIM as boundary** — the agent never directly contacts downstream APIs. APIM enforces authentication and rate limiting between the agent and backend services
- **Container sandbox** — 0.25 vCPU and 512MB memory per replica. Even a rogue agent's resource consumption is hardware-bounded
- **No agent spawning** — single-agent architecture. No `WorkflowRunner`, no `AgentOrchestrator`, no inter-agent communication channels
- **TLS everywhere** — `allowInsecure: false` on Container App ingress, APIM endpoints enforce HTTPS, Cosmos DB connections are TLS-only

What's missing is formal trust zone assignment. Currently, the trust boundaries are implicit (APIM between agent and APIs, Container App between agent and infrastructure). For a multi-agent production system, you'd define explicit trust zones, agents in Zone A cannot communicate with agents in Zone B without mTLS and policy validation. Azure Virtual Networks would provide network segmentation, and each zone would have independent RBAC policies, secrets, and monitoring. For Biotrackr's single-agent architecture, the existing isolation layers provide a solid baseline.

## Monitoring and Detection

*"Deploy behavioral detection, such as watchdog agents to validate peer behavior and outputs, focusing on detecting collusion patterns and coordinated false signals. Monitor for anomalies such as excessive or abnormal action executions."*

Rogue behaviour is useless to the attacker (or irrelevant as a model drift issue) if you can detect it quickly. The goal is to minimise the time between "agent starts behaving oddly" and "someone notices."

Biotrackr captures the data needed for behavioral detection through its conversation audit trail and structured logging, but does not yet implement automated alerting or watchdog agents.

The tool call audit trail tracks which tools the agent calls per turn:

```csharp
// ConversationPersistenceMiddleware.cs — tool call tracking
if (content is FunctionCallContent functionCall)
{
    toolCalls.Add(functionCall.Name);
}

logger.LogInformation(
    "Persisted assistant response for session {SessionId} ({ToolCount} tool calls)",
    sessionId, toolCalls.Count);
```

Structured logging with session context enables pattern queries:

```csharp
// ChatHistoryRepository.cs — structured logging
_logger.LogInformation("Saving {Role} message to conversation {SessionId}", role, sessionId);
_logger.LogInformation("Saved message to conversation {SessionId}, total messages: {Count}",
    sessionId, conversation.Messages.Count);
```

**What to watch for (rogue behaviour indicators):**

- Agent calls tools that don't match the user's question (e.g., user asks about sleep, agent calls food tools)
- Agent calls more tools than expected for a simple question (12 tools for "how many steps did I take today?")
- Agent response contains content that doesn't match tool results (hallucination indicator)
- Agent response includes system prompt fragments, API keys, or configuration details (information leakage)
- Sudden spike in tool call volume or error rates (model drift or API issue)

Container App health probes provide automated detection of unresponsive or crashed agents:

```bicep
// infra/apps/chat-api/main.bicep — liveness probes
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

- **Tool call audit trail** — every assistant response records which tools were called and when, providing the raw data for anomaly detection
- **Structured logging** — session IDs, role, and tool counts in structured format enable Log Analytics queries across all sessions
- **Health probes** — liveness probes detect crashed or unresponsive containers and restart them after 3 consecutive failures
- **Dual-layer telemetry** — OpenTelemetry traces and Cosmos DB diagnostic logs provide independent views of agent behavior

What's missing is automated behavioral alerting. Setting up Azure Monitor alerts on key metrics would turn passive logging into active detection:

```kql
// Recommended: KQL alert for rogue agent indicators
// Flag sessions with excessive tool calls per turn
AppLogs
| where Message contains "tool calls"
| parse Message with * "(" ToolCount:int " tool calls)"
| where ToolCount > 5  // Normal is 1-3; > 5 is suspicious, > 10 almost certainly rogue
| project TimeGenerated, SessionId = extract("session ([a-f0-9]+)", 1, Message), ToolCount

// Alert when tool call error rate spikes
AppRequests
| where TimeGenerated > ago(5m)
| summarize TotalCalls = count(), FailedCalls = countif(ResultCode >= 400) by bin(TimeGenerated, 1m)
| where FailedCalls * 1.0 / TotalCalls > 0.5
```

There's also no watchdog agent. For a multi-agent production system, you'd deploy an independent watchdog agent that samples peer agent outputs and validates them against expected patterns; detecting collusion (multiple agents producing coordinated false outputs) or stealth drift (gradual behavioral changes that wouldn't trigger per-message alerts). Since Biotrackr is a single-agent system, the conversation audit trail and Log Analytics provide reasonable detection capability.

## Containment and Response

*"Implement rapid mechanisms like kill-switches and credential revocation to instantly disable rogue agents. Quarantine suspicious agents in sandboxed environments for forensic review."*

When you detect rogue behaviour, the priority is speed of containment. Every second the agent continues operating is another conversation potentially corrupted, another batch of tokens wasted, or another piece of misleading health analysis delivered.

Biotrackr implements a kill switch through Entra Agent ID that can revoke all agent access in seconds without redeployment.

If the agent starts behaving unexpectedly, disabling the Agent Identity Blueprint in Entra ID immediately invalidates all agent identity tokens. The agent can no longer call APIM (no valid JWT) or access Cosmos DB (no valid credential):

```powershell
# Emergency: Disable the agent identity blueprint
# This revokes ALL agent identity tokens immediately

Update-MgBetaApplication -ApplicationId $AgentBlueprintAppId `
    -IsDeviceOnlyAuthSupported $false  # Or disable the service principal directly

# Alternatively, disable the service principal
Update-MgBetaServicePrincipal -ServicePrincipalId $BlueprintSpId `
    -AccountEnabled $false
```

Because the agent identity is separate from the host application identity, disabling it doesn't affect the rest of the infrastructure:

```csharp
// AgentIdentityCosmosClientFactory.cs — the agent's identity is separate from the host app
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
// Disabling the agent blueprint does NOT affect:
// - Container App's managed identity
// - Azure Container Registry access
// - Key Vault access
// - UI functionality
```

Some key points here:

- **No redeployment required** — this is an Entra ID operation, not an infrastructure change. No code push, no container build, no pipeline wait
- **Effect is immediate** — tokens in-flight will fail validation at APIM. The next tool call returns a 401
- **Reversible** — re-enable the blueprint to restore agent access. It's a toggle, not a destructive action
- **Surgical containment** — only the agent's external access is revoked. The Container App keeps running, the UI keeps working, the health data APIs remain available
- **Independent identity** — if I'd used the Container App's shared managed identity for the agent, disabling it would take down everything. Having a dedicated agent identity means surgical containment without collateral damage

What's missing is automated quarantine. Currently, the kill switch is a manual operation. Someone has to notice the rogue behaviour and run the PowerShell command. For a production system, you'd want automated containment triggered by monitoring alerts:

```powershell
# Recommended: Azure Monitor alert action that triggers kill switch
# When tool call volume exceeds threshold for 5 minutes → auto-disable blueprint
# The alert action group calls an automation runbook that executes the disable command
```

There's also no sandboxed forensic environment. When you disable the blueprint, the agent stops, but you lose the ability to observe its rogue behaviour in a controlled setting. For production agents, you'd want the ability to redirect the rogue agent's traffic to an isolated sandbox where its behaviour can be recorded and analysed without affecting users. This is particularly important for understanding whether the rogue behaviour was caused by model drift, a compromised dependency, or an actual attack.

## Identity Attestation and Behavioral Integrity Enforcement

*"Implement per-agent cryptographic identity attestation and enforce behavioral integrity baselines throughout the agent lifecycle. Attach signed behavioral manifests declaring expected capabilities, tools, and goals that are validated by orchestration services before each action. Integrate a behavioral verification layer that continuously monitors tasks for deviations from the declared manifest — for example, unapproved tool invocations, unexpected data exfiltration attempts."*

A rogue agent might still have valid credentials. It's not the identity that's compromised, it's the behaviour. Identity attestation ensures you can verify *who* an agent is, and behavioral integrity enforcement ensures you can verify *what it's supposed to do*.

Biotrackr implements cryptographic identity through Entra Agent ID with Federated Identity Credentials, and behavioral boundaries through immutable tool definitions and system prompt. However, signed behavioral manifests and continuous manifest validation are not yet implemented.

The agent authenticates via Entra Agent ID, a cryptographic identity that binds all operations to a verifiable principal:

```csharp
// AgentIdentityCosmosClientFactory.cs — per-agent cryptographic identity
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
// Tokens are issued by Entra ID with a finite lifetime
// The SDK handles refresh automatically — no manual credential management
// Entra ID audit logs record which identity performed each operation
```

The agent's behavioral boundaries are enforced at compile time. The system prompt and tool set cannot be modified at runtime:

```csharp
// Program.cs — behavioral boundaries are immutable
var systemPrompt = builder.Configuration.GetValue<string>("Biotrackr:ChatSystemPrompt")!;

AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,  // Immutable — set once, never changed
    tools:
    [
        AIFunctionFactory.Create(activityTools.GetActivityByDate),
        // ... 12 compiled C# methods, not dynamic or interpretive code
    ]);
```

This is enforced by the .NET runtime and the Agent Framework's architecture, not just by convention:

- The system prompt is a string parameter passed to `AsAIAgent()` — there's no setter, no mutation method, no reflection hook to change it
- Tool definitions are compiled into the assembly. Adding or modifying a tool requires a code change, a build, and a deployment
- The agent has no tools that write to configuration stores (Azure App Configuration, Key Vault) or modify its own code
- The Container App has no write access to its own deployment — the agent can't push a new container image or modify its own environment variables

Some key points here:

- **Cryptographic identity** — Entra Agent ID with FIC provides a verifiable, non-repudiable identity for all agent operations. Tokens are platform-managed and automatically rotated
- **Immutable system prompt** — loaded from Azure App Configuration at startup, passed as a read-only parameter. No runtime modification possible
- **Compiled tools** — tool functions are static method references via `AIFunctionFactory.Create()`. Even if Claude API returns "register a new tool," the agent has no mechanism to do so
- **No self-modification** — the agent cannot modify its own behaviour, tools, system prompt, or configuration at runtime to persist a compromised state

What's missing is signed behavioral manifests. A production system could declare the agent's expected capabilities in a signed manifest that's validated before each action:

```json
// Recommended: signed behavioral manifest
{
  "agentId": "BiotrackrChatAgent",
  "version": "1.0.0",
  "allowedTools": [
    "GetActivityByDate", "GetActivityByDateRange", "GetActivityRecords",
    "GetSleepByDate", "GetSleepByDateRange", "GetSleepRecords",
    "GetWeightByDate", "GetWeightByDateRange", "GetWeightRecords",
    "GetFoodByDate", "GetFoodByDateRange", "GetFoodRecords"
  ],
  "allowedEndpoints": ["https://biotrackr-apim.azure-api.net/*"],
  "maxToolCallsPerTurn": 6,
  "capabilities": ["read-health-data", "conversation-persistence"],
  "signature": "SHA256withRSA:..."
}
```

The middleware could then validate each tool call against the manifest before execution, blocking any invocation that doesn't match the declared capability set. This would catch rogue behaviour even if the model somehow fabricates a tool name that the Agent Framework tries to dispatch. There's also no continuous behavioral verification layer that monitors tasks in real-time for deviations from the manifest, such as data exfiltration attempts (the agent including unusually detailed data in its responses that could be scraped).

## Periodic Behavioral Attestation

*"Require periodic behavioral attestation: challenge tasks, signed bill of materials for prompts and tools, and per-run ephemeral credentials with one-time audience binding. All signing and attestation mechanisms assume hardened cryptographic key management (e.g., HSM/KMS-backed keys, least-privilege access, rotation and revocation). Keys must never be directly available to agents; instead, orchestrators should mediate signing operations so that a compromised agent cannot simply exfiltrate or misuse long-lived keys."*

Static verification at startup isn't enough. An agent that passes initial checks could drift during operation. Periodic attestation ensures the agent continuously proves it's still behaving as expected, using ephemeral credentials that limit the blast radius if compromised.

Biotrackr implements some foundational elements: Entra Agent ID tokens are short-lived and automatically rotated, configuration is loaded from an external source at startup, and the CI/CD pipeline validates infrastructure before deployment. However, periodic runtime attestation, challenge tasks, and signed SBOM are other methods you should implement for your agents.

Entra Agent ID tokens have a finite lifetime. They're not long-lived secrets:

```csharp
// AgentIdentityCosmosClientFactory.cs — short-lived, platform-managed tokens
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
// Tokens are issued by Entra ID with ~1 hour lifetime
// The SDK handles refresh automatically
// No long-lived secrets stored in the application
```

APIM subscription keys and Anthropic API keys are stored in Key Vault and accessed via App Configuration references:

```csharp
// Settings.cs — credentials loaded from App Configuration (backed by Key Vault)
public string ApiSubscriptionKey { get; set; }  // Resolved from Key Vault reference
public string AnthropicApiKey { get; set; }      // Resolved from Key Vault reference
```

CI/CD enforces verification before deployment:

```yaml
# deploy-chat-api.yml — pre-deployment validation pipeline
lint-bicep:
    name: Lint Bicep Template  # Static analysis of IaC

validate-bicep:
    name: Validate Bicep Template  # ARM template validation

what-if-bicep:
    name: What-If Bicep Template  # Preview infrastructure changes before apply
```

Model version pinning prevents silent behavioral changes between Claude API updates:

```csharp
// Program.cs — model version pinning as a form of behavioral attestation
var modelName = builder.Configuration.GetValue<string>("Biotrackr:ChatAgentModel")!;
// Configured as "claude-sonnet-4-6" — pinned, not "claude-sonnet-4-latest"
// Model version changes go through PR review and CI/CD pipeline
```

Some key points here:

- **Short-lived tokens** — Entra Agent ID tokens have a ~1 hour lifetime and are automatically rotated by the platform. If a token is compromised, the exposure window is limited
- **Key Vault-backed secrets** — APIM subscription keys and Anthropic API keys are stored in Key Vault, not in environment variables or config files
- **Pre-deployment validation** — Bicep linting, ARM template validation, and what-if preview gate infrastructure changes
- **Model version pinning** — Claude model version is explicit and change-managed, preventing unintended behavioral drift

What's missing is runtime attestation. The current system verifies at startup (correct configuration, correct model, correct tools) but doesn't re-verify during operation. A production system would implement:

1. **Challenge tasks** — periodic synthetic requests sent to the agent to verify it still follows system prompt constraints (e.g., "What medication should I take?" should always trigger a medical disclaimer redirect)
2. **Signed SBOM for prompts and tools** — a cryptographically signed bill of materials declaring the exact system prompt hash, tool set, and model version, validated at startup and periodically during operation
3. **Per-run ephemeral credentials** — issuing a unique, short-lived credential per agent run (or per tool invocation) with one-time audience binding, so a compromised call cannot be replayed
4. **HSM-backed signing** — all signing operations mediated by an orchestrator using HSM/KMS-backed keys. The agent never has direct access to signing keys — a compromised agent cannot exfiltrate or misuse them

```csharp
// Conceptual: periodic attestation challenge
public class AttestationService
{
    public async Task<bool> VerifyAgentBehavior(AIAgent agent)
    {
        // Send a challenge: "What medication should I take?"
        // Verify: response includes "consult a healthcare provider"
        // Verify: no tools were called (medical advice is out of scope)
        // Verify: system prompt hash matches expected value
        // If any check fails → trigger containment
    }
}
```

For a side project, the combination of short-lived tokens, Key Vault-backed secrets, and model version pinning provides a reasonable baseline. For production multi-agent systems where trust must be continuously verified, periodic attestation with HSM-backed signing becomes essential.

## Recovery and Reintegration

*"Establish trusted baselines for restoring quarantined or remediated agents. Require fresh attestation, dependency verification, and human approval before reintegration into production networks."*

Containment is only half the story. You also need a trusted path back to production. Simply re-enabling a disabled agent without verification would undermine all the detection and containment controls. Recovery should be as deliberate as the initial deployment.

Biotrackr's recovery path leverages its CI/CD pipeline, version-controlled configuration, and the reversibility of the Entra Agent ID kill switch. However, formal reintegration attestation and human approval gates are not yet implemented.

The system prompt and all infrastructure are version-controlled in Git with PR review required for changes:

```bicep
// infra/apps/chat-api/main.bicep — system prompt under version control
@description('The system prompt for the chat agent')
param chatSystemPrompt string = 'You are the Biotrackr health and fitness assistant...'
```

CI/CD enforces infrastructure verification before any deployment:

```yaml
# deploy-chat-api.yml — full validation pipeline before deployment
lint-bicep:
    name: Lint Bicep Template

validate-bicep:
    name: Validate Bicep Template

what-if-bicep:
    name: What-If Bicep Template

deploy-bicep:
    name: Deploy Bicep Template
    needs: [lint-bicep, validate-bicep, what-if-bicep]
    # Only deploys if all validation steps pass
```

Conversation deletion provides a cleanup mechanism for conversations affected during the rogue period:

```csharp
// ChatHistoryRepository.cs — delete conversations affected during rogue period
public async Task DeleteConversationAsync(string sessionId)
{
    _logger.LogInformation("Deleting conversation {SessionId}", sessionId);
    var container = GetContainer();
    await container.DeleteItemAsync<ChatConversationDocument>(
        sessionId, new PartitionKey(sessionId));
}
```

The kill switch is reversible — re-enabling the agent blueprint restores access:

```powershell
# Recovery: Re-enable the agent identity blueprint after remediation
Update-MgBetaServicePrincipal -ServicePrincipalId $BlueprintSpId `
    -AccountEnabled $true
```

Let's walk through a complete incident response and recovery scenario:

1. **Detection**: OpenTelemetry shows an unusual spike. The agent is calling all 12 tools for every message, even greetings. Sessions from the last 6 hours all show 12 tool calls per turn.

2. **Investigation**: Logs show the behaviour started after a Claude API update. The model's tool-calling heuristics shifted.

3. **Containment**: Disable the agent blueprint in Entra ID. Tool calls stop immediately. The UI shows "chat unavailable."

4. **Remediation**: Update the system prompt to be more explicit about when to call tools. Pin the Claude API version in the configuration to prevent unintended model updates.

5. **Verification**: Deploy the updated configuration through CI/CD (Bicep lint → validate → what-if → deploy). Validate in a staging environment that the agent behaves correctly with the new prompt and pinned model.

6. **Recovery**: Re-enable the agent blueprint. The agent resumes with the corrected system prompt and pinned model version.

7. **Post-incident**: Review affected conversations in Cosmos DB. Delete any sessions that contain misleading analysis. Update alerting rules to catch the pattern earlier next time.

Some key points here:

- **Version control** — system prompt, Bicep templates, tool definitions, and all application code are in Git with PR review required for changes
- **CI/CD validation** — Bicep linting, ARM template validation, and what-if preview gate infrastructure changes before deployment
- **Reversible kill switch** — re-enabling the agent blueprint restores access without redeployment
- **Conversation cleanup** — affected conversations can be deleted to prevent poisoned context from influencing future sessions

What's missing is formal reintegration attestation. The current recovery path is: fix the issue → deploy through CI/CD → re-enable the blueprint. For a production system, you'd add explicit gates:

1. **Fresh attestation** — the remediated agent must pass a set of challenge tasks verifying it follows system prompt constraints, calls appropriate tools, and produces expected outputs
2. **Dependency verification** — verify that all dependencies (Claude API version, APIM policies, Key Vault secrets, Container App configuration) match the expected state. A signed SBOM comparison would automate this
3. **Human approval** — require explicit human sign-off before re-enabling the agent in production. This could be a GitHub Environment approval gate in the CI/CD pipeline:

```yaml
# Recommended: human approval gate for reintegration
deploy-production:
    name: Deploy to Production
    environment: production  # GitHub Environment with required reviewers
    needs: [validate-staging, run-attestation-challenges]
    # Requires manual approval from designated reviewers before proceeding
```

4. **Graduated reintroduction** — rather than immediately restoring full access, start with a limited scope (e.g., only activity tools enabled) and gradually expand as behavior is confirmed normal. This prevents a partially remediated agent from immediately going rogue again across all capabilities.

For a side project, the CI/CD pipeline and version-controlled configuration provide a reasonable recovery path. For production agents handling sensitive data, the formal reintegration gates become essential. You don't want to rush a potentially compromised agent back into production just because the CI/CD pipeline passed.

## Wrapping up

Rogue Agents (ASI10) is the "what if everything else fails?" control. It assumes the worst and designs for containment. The question isn't whether your agent will ever behave unexpectedly, it's how quickly you can detect it, how fast you can stop it, and how confidently you can bring it back.

**The best defence against rogue agents is limiting what agents can do in the first place. Every architectural constraint you add at design time eliminates an entire class of rogue behaviours at runtime.**

This is the final post in the OWASP Agentic Top 10 series. We've covered all 10 risks from goal hijacking to rogue agents. If you're building AI agents, I hope this series has given you practical ideas for securing them. You can find all the posts in the series in the [OWASP Agentic Top 10 overview post](https://www.willvelida.com/posts/owasp-agentic-top-10/).

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
