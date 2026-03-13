---
title: "Preventing Identity and Privilege Abuse in AI Agents"
date: 2026-03-11
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "APIM", "Managed Identity", "Entra Agent ID"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wgas8z75g1swg7fwnvx3.png
    alt: "Preventing OWASP ASI03 Identity and Privilege Abuse in a .NET AI agent with Entra Agent ID, RBAC, federated credentials, and per-action authorization."
    caption: "Implementing OWASP ASI03 mitigations against Identity and Privilege Abuse in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

One of the challenges I faced developing an agent for my side project ([Biotrackr](https://github.com/willvelida/biotrackr)) was how do I manage identity. Some AI Agents share the same service principals or managed identity with the application, which is used to authenticate API calls, access databases etc.

This is an issue, because if the application has contributor access to a database, so does the agent. If the agent gets compromised, then the blast radius extends to the entire application's permission scope.

I've written a [couple of articles on Microsoft Entra Agent ID](https://www.willvelida.com/tags/entra-agent-id/), and how it solves this issue by giving AI Agents their own identity in Microsoft Entra. This is great, because this identity is separate from the host application and it gives the agent its own dedicated permissions, audit trails, and a kill switch.

Biotrackr uses Agent ID to ensure that the chat agent has read-only access to health data, and nothing more. 

In this article, we'll cover **Agent Identity and Privilege Abuse** and how we can implement prevention and mitigation strategies to prevent escalation of agent privileges to perform actions beyond its intended scope, using Biotrackr as an example.

## What is Identity and Privilege Abuse?

Identity and Privilege Abuse exploits dynamic trust and delegation in Agents to escalate access and bypass controls by manipulating delegation chains, role inheritance, control flows, and agent context. This includes cached credentials or conversation history.

When it comes to agents, identity refers to both the defined persona of the agent and to any authentication material that represents it. Agent-to-Agent trust or inherited credentials can be exploited to escalate access, hijack privileges, or execute unauthorized actions.

Without a distinct identity for the agent, it can operate in an attribution gap, making enforcing policies like least-privilege impossible.

## Implementing controls for Biotrackr

Why does this matter for my little side project?

The chat agent is designed to retrieve data from Cosmos DB for chat history, and APIM for health data. These are two distinct resource planes.

If I used the shared managed identity for the agent, the surface area expands to everything that the Container App holds. This includes the Azure Container Registry, Key Vault, Application Insights, Log Analytics, Azure App Configuration. This is a significant blast radius for the agent to access.

If the agent were to be compromised via prompt injection, this could have a destructive impact on resources *well beyond* the scope of the agent.

The agent also uses tools to complete tasks and analysis. Each tool call hits real infrastructure and incurs real costs. Implementing an identity for the agent is crucial for when tool-level controls are bypassed.

With this in mind, let's take a look at the prevention and mitigation strategies we can implement to prevent identity and privilege abuse for our agents, using Biotrackr as an example.

### Entra Agent ID concepts

Before walking through the guidelines, three Entra Agent ID constructs are referenced throughout:

1. **Agent Identity Blueprint** — a template that defines the agent's shared configuration (description, OAuth2 scopes, credentials, owners). Think of it as the "class" from which agent instances are created.

2. **Agent Identity** — a single-tenant service principal with an `agent` subtype, created from a blueprint. This is the actual identity that acquires tokens and calls APIs. Think of it as an "instance" of the blueprint.

3. **Federated Identity Credential (FIC)** — links the blueprint to a user-assigned managed identity. Instead of client secrets, the managed identity's assertion is used as the credential. Automatic rotation, no secrets to manage.

## Enforce Task-Scoped, Time-Bound Permissions

*"Issue short-lived, narrowly scoped tokens per task and cap rights with permission boundaries — using per-agent identities and short-lived credentials (e.g., mTLS certificates or scoped tokens) — to limit blast radius, block delegated-abuse and maintenance-window attacks, and mitigate un-scoped inheritance, orphaned privileges, and reflection-loop elevation."*

Biotrackr implements this control through two layered constraints:

### Per-Agent Identity via Entra Agent ID

The agent has its own dedicated identity, which is separate from the host Container App's managed identity. The `AgentIdentityCosmosClientFactory` acquires tokens scoped specifically to the agent:

```csharp
public class AgentIdentityCosmosClientFactory : ICosmosClientFactory
{
    private readonly MicrosoftIdentityTokenCredential _credential;
    private readonly Settings _settings;

    public AgentIdentityCosmosClientFactory(
        MicrosoftIdentityTokenCredential credential,
        IOptions<Settings> options)
    {
        _credential = credential;
        _settings = options.Value;
    }

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

The `WithAgentIdentity()` tells the credential to acquire tokens as the agent identity (not the host app). `RequestAppToken = true` requests an app-only token (autonomous agent flow, no user delegation).

The resulting token carries agent-specific claims: `xms_act_fct: 11`, `xms_sub_fct: 11` and the agent's RBAC is scoped to Cosmos DB Data Contributor on a single account, meaning that it cannot access Key Vault, Storage, or other resources.

### Federated Identity Credential (No Secrets in Production)

Instead of a long-lived client secret, the agent authenticates via a Federated Identity Credential (FIC) linked to the Container App's user-assigned managed identity:

```powershell
# Links the UAI to the blueprint — no client secrets needed at runtime
$federatedCredential = @{
    Name      = "biotrackr-uai"
    Issuer    = "https://login.microsoftonline.com/$TenantId/v2.0"
    Subject   = $ManagedIdentityPrincipalId  # UAI's principal ID
    Audiences = @("api://AzureADTokenExchange")
}

New-MgBetaApplicationFederatedIdentityCredential `
    -ApplicationId $AgentBlueprintAppId `
    -BodyParameter $federatedCredential
```

FIC uses the managed identity's assertion as the credential, and Azure rotates the underlying managed identity tokens automatically (typically every 24 hours).

We could strengthen this further by using per-task scoping (e.g. a token valid only for fetching activity data). Agent Identity tokens are scoped to the Cosmos DB account, rather than individual operations.

## Isolate Agent Identities and Contexts

*"Run per-session sandboxes with separated permissions and memory, wiping state between tasks to prevent Memory-Based Escalation and reduce Cross-Repository Data Exfiltration."*

Biotrackr separates the agent identity from the host application identity, providing that identity-level isolation between the agent and the UI.

```csharp
// Program.cs — agent identity is registered separately from the host identity
builder.Services.AddMicrosoftIdentityAzureTokenCredential();
builder.Services.AddAgentIdentities();
builder.Services.AddScoped<ICosmosClientFactory, AgentIdentityCosmosClientFactory>();
```

If we need to disable the agent identity, only the agent's access will be revoked. The UI will continue to function since it uses its own identity for non-agent operations.

Each conversation session is isolated in Cosmos DB with its own partition key:

```csharp
// ConversationPersistenceMiddleware — session isolation
var sessionId = session?.GetHashCode().ToString("x8") ?? Guid.NewGuid().ToString();

// Save the user message to Cosmos under the session's partition
await repository.SaveMessageAsync(sessionId, "user", userContent);
```

Each conversation session gets a unique partition key, so one conversation cannot read or modify another's data. The conversation history is scoped to the session, meaning that the agent can only see messages from the current conversation, not cross-session.

There are a couple of things missing in Biotrackr that's worth pointing out here. For tool response caching, I'm using `IMemoryCache` to share caching across all sessions. A cached response from one user's session could be served to another.

Since this is my side project, I've designed it to be single-user. For multi-user systems however, this is something you'd need to address via per-session or per-user cache keys.

There's also no per-session sandboxing or permissions. The agent's RBAC scope is the same regardless of which session initiated the call.

## Mandate Per-Action Authorization

*"Re-verify each privileged step with a centralized policy engine that checks external data, stopping Cross-Agent Trust Exploitation and Reflection Loop Elevation."*

Every tool call in Biotrackr results in an HTTP request to APIM, and each request is individually authenticated:

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

APIM then validates the subscription key (or JWT token) on every request via an inbound policy:

```xml
<inbound>
  <base />
  <choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault(&quot;Authorization&quot;,&quot;&quot;)
                       .StartsWith(&quot;Bearer &quot;))">
      <validate-jwt header-name="Authorization" failed-validation-httpcode="401"
                     failed-validation-error-message="Unauthorized: Invalid or missing JWT token">
        <openid-config url="{{openid-config-url}}" />
        <audiences>
          <audience>{{jwt-audience}}</audience>
        </audiences>
        <issuers>
          <issuer>{{jwt-issuer}}</issuer>
        </issuers>
      </validate-jwt>
    </when>
    <otherwise>
      <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401"
                     failed-check-error-message="Unauthorized: Missing or invalid subscription key" />
    </otherwise>
  </choose>
</inbound>
```

Having these in place means that every tool call passes through APIM authentication, preventing the LLM from bypassing it. If the key is revoked, or the JWT is invalid, then the tool call will fail immediately.

APIM can enforce additional policies, such as rate limits, request quotas, IP restrictions etc. that can act as guardrails that are independent from the agent code.

We can further strengthen this using a centralized policy engine that checks intent context. APIM validates the identity, but it doesn't verify if the tool call is consistent with the original question from the user.

For multi-agent systems, cross-agent trust exploitation would need each agent to re-verify the calling agent's permissions. This isn't an issue for me yet!

## Apply Human-in-the-Loop for Privilege Escalation

*"Require human approval for high-privilege or irreversible actions to provide a safety net that would stop Memory-Based Escalation, Cross-Agent Trust Exploitation, and Maintenance Window attacks."*

This isn't a big issue for me primarily because the Biotrackr agent has no destructive capabilities. All tools that the agent has are read-only HTTP GET operations. The Chat API has a delete endpoint for conversations, but this isn't exposed as an agent tool.

If future tools introduce write operations (which is something I'm thinking about implementing), human approval steps should be added before agents can execute these types of actions.

The AG-UI protocol that I've implemented into the Chat API supports streaming events. A `confirmation_requested` event type could pause the stream and wait for user approval on high-privilege actions.

Again, something for the backlog should the time come 😄

## Define Intent

*"Bind OAuth tokens to a signed intent that includes subject, audience, purpose, and session. Reject any token use where the bound intent doesn't match the current request."*

This is partially implemented in my chat agent. The agent identity token includes subject and audience claims by default:

- **Subject**: The agent identity service principal (`appId` set via `WithAgentIdentity()`).
- **Audience**: Cosmos DB resource URI (for database access) or APIM audience (for API access via `{{jwt-audience}}` in the APIM policy).

```csharp
// The token is implicitly scoped to subject (agent identity) and audience (Cosmos DB)
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
```

There are a couple of things missing that could be implemented here:

- **Purpose binding** — the token does not encode what the agent intends to do (e.g., "fetch activity data for March 2026"). Any valid token can call any Cosmos DB operation within the Data Contributor role
- **Session binding** — the token is not tied to a specific conversation session. The same token could theoretically be used across sessions
- **Signed intent validation** — APIM validates the token's subject and audience but does not check a `purpose` or `session_id` claim
- To fully implement this guideline, the token request could include custom claims (via Entra claims transformation) that encode the session ID and intended operation. APIM could then validate these claims match the request path and query parameters
- A lighter-weight approach: include a `X-Session-Id` header in API calls and log it alongside the JWT claims for correlation, without enforcing it as a hard gate

## Evaluate Agentic Identity Management Platforms

*"Major platforms integrate agents into their identity and access management systems, treating them as managed non-human identities with scoped credentials, audit trails, and lifecycle controls. Examples include Microsoft Entra, AWS Bedrock Agents, Salesforce Agentforce, Workday's Agentic System of Record (ASOR) model, and similar emerging patterns in Google Vertex AI."*

I think I'm doing a pretty good job of it using Microsoft Entra Agent ID! 😉🤖

### Blueprint Creation (Pre-Provision Script)

```powershell
# Create Agent Identity Blueprint via Microsoft Graph beta API
$body = @{
    "@odata.type"          = "Microsoft.Graph.AgentIdentityBlueprint"
    "displayName"          = "biotrackr-chat-agent"
    "sponsors@odata.bind"  = @("https://graph.microsoft.com/v1.0/users/$($user.id)")
    "owners@odata.bind"    = @("https://graph.microsoft.com/v1.0/users/$($user.id)")
} | ConvertTo-Json -Depth 5

$response = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/beta/applications/graph.agentIdentityBlueprint" `
    -Body $body -ContentType "application/json"
```

### Agent Identity Provisioning (Post-Provision Script)

```powershell
# Acquire blueprint token via client_credentials
$tokenResponse = Invoke-RestMethod -Method POST `
    -Uri "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token" `
    -ContentType "application/x-www-form-urlencoded" `
    -Body @{
        client_id     = $AgentBlueprintAppId
        scope         = "https://graph.microsoft.com/.default"
        client_secret = $AgentBlueprintClientSecret
        grant_type    = "client_credentials"
    }

# Create Agent Identity using the blueprint's own token
$agentBody = @{
    "@odata.type"              = "#Microsoft.Graph.AgentIdentity"
    "displayName"              = "biotrackr-chat-agent"
    "agentIdentityBlueprintId" = $AgentBlueprintAppId
    "sponsors@odata.bind"      = @("https://graph.microsoft.com/v1.0/users/$SponsorUserId")
} | ConvertTo-Json -Depth 5

$agentResponse = Invoke-RestMethod -Method POST `
    -Uri "https://graph.microsoft.com/beta/serviceprincipals/Microsoft.Graph.AgentIdentity" `
    -Headers @{
        "Authorization" = "Bearer $($tokenResponse.access_token)"
        "OData-Version" = "4.0"
    } `
    -Body $agentBody -ContentType "application/json"
```

### Application Registration

```csharp
// Program.cs — register agent identity services
builder.Services.AddMicrosoftIdentityAzureTokenCredential();
builder.Services.AddAgentIdentities();
builder.Services.AddScoped<ICosmosClientFactory, AgentIdentityCosmosClientFactory>();
```

Some key things to note here:

- Blueprint → Agent Identity is a 1:many relationship. One blueprint can govern multiple agent instances across environments
- `access_agent` OAuth2 scope is configured on the blueprint, and controls what delegated permissions the agent can request
- Sponsors and owners are assigned, providing accountability and governance
- The blueprint's temporary client secret is used only for one-time provisioning, FIC handles runtime authentication
- One thing to note is that Entra Agent ID is in preview. The API surface may change, but the identity model (blueprint → agent → FIC) is production-grade.

## Bind Permissions to Subject, Resource, Purpose, and Duration

*"Bind permissions to subject, resource, purpose, and duration. Require re-authentication on context switch. Prevent privilege inheritance across agents unless the original intent is re-validated. Include automated revocation on idle or anomaly."*

Again this is something I'm only implementing partially. The agent identity's RBAC is bound to a specific subject and resource:

```powershell
# Cosmos DB RBAC — bound to subject (agent SP) and resource (specific Cosmos account)
$cosmosScope = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName" +
    "/providers/Microsoft.DocumentDB/databaseAccounts/$CosmosDbAccountName"

az cosmosdb sql role assignment create `
    --account-name $CosmosDbAccountName `
    --resource-group $ResourceGroupName `
    --role-definition-id "00000000-0000-0000-0000-000000000002" `
    --principal-id $agentSpObjectId `
    --scope $cosmosScope
```

The subject is the Agent identity service principal, not the host app or the managed identity. The resource is the single Cosmos DB account, which it needs for read/write operations for chat history.

We can revoke the agent identity by disabling the blueprint. All agent identity tokens are immediately invalid, and can no longer authenticate to APIM or Cosmos DB. The UI will continue to function normally, since it uses its own managed identity.

We can strengthen this control further by implementing the following:

- **Purpose binding** — the role assignment doesn't encode what the agent should do with Cosmos DB access (read chat history vs. write chat history vs. delete data)
- **Duration binding** — the RBAC assignment is permanent until manually removed. A time-bound role assignment (using Entra PIM for non-human identities, when available) would satisfy this fully
- **Re-authentication on context switch** — the agent uses the same token across all operations within a session. Switching from "analyze activity data" to "delete conversation" doesn't trigger re-authentication. Since the agent has no delete tools, this is low-risk, but a multi-tool agent with mixed read/write operations should re-authenticate on privilege escalation
- **Automated revocation on idle** — no mechanism to detect agent inactivity and revoke tokens. A future Azure Automation runbook could disable the blueprint after N hours of no tool calls, re-enabling it on the next user message

## Detect Delegated and Transitive Permissions

*"Monitor when an agent gains new permissions indirectly through delegation chains. Flag cases where a low-privilege agent inherits or is handed higher-privilege scopes during multi-agent workflows."*

Biotrackr has only one agent so far. There are no delegation chains or multi-agent workflows. The chat agent is the only agent, and it cannot delegate to other agents or grant permissions to sub-agents.

```csharp
// Program.cs — single agent, no delegation
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

The tool set is static, and they are registered at compile time via `AIFunctionFactory.Create()` with direct method references. No tool dynamically requests new permissions or creates new identity contexts. The agent cannot call other agents, invoke other services beyond APIM, or escalate its own RBAC.

If I were to introduce more agents into the system, each agent would need its own agent identity with independent RBAC. For communication between agents, we'd need a mechanism to monitor tool outputs between agents and ensure that if an agent used that output to request a higher-privilege scope, we flag it.

## Detect Abnormal Cross-Agent Privilege Elevation and Device-Code Style Phishing

*"Detect abnormal cross-agent privilege elevation and device-code style phishing flows by monitoring when agents request new scopes or reuse tokens outside their original, signed intent."*


The agent's token acquisition is constrained by the `AgentIdentityCosmosClientFactory` — it always requests the same scope (Cosmos DB) with the same identity:

```csharp
// Token acquisition is fixed — always the same identity, same scope
_credential.Options.WithAgentIdentity(_settings.AgentIdentityId);
_credential.Options.RequestAppToken = true;
```

The agent cannot request new scopes at runtime — it doesn't have access to the `TokenCredential` outside of the factory. Device-code flow is not applicable as the agent uses `client_credentials` (via FIC), not interactive authentication. Token reuse outside the original intent is mitigated by the APIM subscription key being separate from the Cosmos DB token — even if one is compromised, the other is unaffected.

Biotrackr has three observability layers that could detect anomalous behavior:

1. **Entra sign-in logs** — tokens acquired with `xms_act_fct: 11` are classified as AI agent activity. Unusual patterns (new scopes, new resources, off-hours acquisition) can be detected via Log Analytics
2. **OpenTelemetry tracing** — distributed traces correlate user messages → tool calls → API calls → Cosmos reads. An unexpected trace pattern (e.g., Cosmos write operations the agent shouldn't make) would be visible

```csharp
// OpenTelemetry tracing captures the full request chain
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

3. **Conversation persistence middleware** — tool call names are logged to Cosmos DB, providing a per-session audit trail

```csharp
// Tool call names are captured per-session
if (content is FunctionCallContent functionCall)
{
    toolCalls.Add(functionCall.Name);
}

// Persisted to Cosmos with tool call audit
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);
```

We can strengthen controls for this threat further by adding automated alerting on anomalous token requests, logging the tool call arguments, logging when agents request a new scope, and configuring Azure Monitor alerts to capture scope drift changes for agents.

## Wrapping up

Identity and Privilege Abuse (ASI03) is about ensuring your agent has its own identity with the minimum permissions it needs, and nothing more.

The controls are layered: dedicated agent identity → federated credentials → scoped RBAC → per-request APIM authentication → session isolation → observability. Even if one layer is compromised, the others constrain the blast radius. **Give your agent its own identity from day one.**

There are gaps I haven't addressed yet. Purpose and session-bound tokens (Guideline 5), time-bound RBAC assignments (Guideline 7), and automated anomaly alerting (Guideline 9) are all on the backlog. If you're building multi-agent systems where agents delegate to each other or share resources, those controls become critical rather than nice-to-have.

One thing worth noting is that Microsoft Entra Agent ID is still in preview. The API surface may evolve, but the core identity model (blueprint, agent identity, federated credential) is solid and production-grade. If you're building agents on Azure, I'd recommend adopting it now rather than retrofitting shared identities later.

In the next post in this series, I'll cover **ASI04 — Supply Chain Vulnerabilities**, which explores the risks of depending on external models, tools, and packages in your agent's supply chain. Many of the controls we've discussed here (static tool registration, no dynamic plugin loading) are the first line of defence against that too.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️