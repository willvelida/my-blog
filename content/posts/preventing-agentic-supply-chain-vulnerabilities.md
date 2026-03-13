---
title: "Preventing Agentic Supply Chain Vulnerabilities"
date: 2026-03-11
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "APIM", "Managed Identity", "Entra Agent ID"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zhdfy22lsmzomcq9o3ak.png
    alt: "Preventing OWASP ASI04 Agentic Supply Chain Vulnerabilities in a .NET AI agent with SBOMs, dependency pinning, kill switches, and zero-trust architecture."
    caption: "Implementing OWASP ASI04 mitigations against Agentic Supply Chain Vulnerabilities in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

Your AI Agent's security is only as strong as its weakest dependency. Whatever packages you are using within your agents, you're trusting that those packages that have been published haven't been tampered with and that they don't contain vulnerabilities. The same applies for every transitive dependency in your graph.

In [Biotrackr](https://github.com/willvelida/biotrackr), I'm using a couple of packages that are still in preview, so there may be flaky APIs that could affect my agent's security and reliability. Agentic Supply Chain Vulnerabilities are amplified in agents because AI frameworks are in preview (at time of writing). The technology is evolving rapidly, and these frameworks have deep dependency trees that are harder to audit.

In this article, we'll cover **Agentic Supply Chain Vulnerabilities**, and how we can implement prevention and mitigation strategies to prevent vulnerabilities from affecting our supply chain, using Biotrackr as an example.

## What are Agentic Supply Chain Vulnerabilities?

Agentic Supply Chain Vulnerabilities arise when agents, tools, and related artefacts they work with are provided by third parties and may be malicious, compromised, or tampered with in transit.

These could be NuGet packages that the agents use, or other artefacts like models and model weights, tools, plug-ins, datasets, other agents, MCP servers, A2A, agentic registries and related artifacts.

All of these dependencies may introduce unsafe code, hidden instructions, or deceptive behaviors into the agent's execution chain.

Unlike traditional AI or software supply chains, agents often compose capabilities at runtime, which increases the attack surface. This can create a live supply chain that can cascade vulnerabilities across agents.

## How does this affect my agent?

In my agent, there's a few dependencies on preview AI frameworks. Each is a supply chain node that could be compromised. These preview packages tend to have smaller install bases and less community scrutiny than stable releases.

For example, a poisoned version of `Microsoft.Agents.AI.Anthropic` could exfiltrate conversation history, health data responses, or worse, the agent's Entra identity tokens! The blast radius would increase beyond the agent to every downstream service it authenticates to.

The system prompt itself is retrieved from Azure App Configuration at runtime. If this was compromised or tampered with, the agent's behavior changes silently without any code deployment.

My agent's tool calls hit APIs that return real health data through APIM. A supply chain attack on the tool definitions or the HTTP client pipeline could redirect API calls, inject malicious parameters, or even exfiltrate responses.

Let's not forget that agents run on infrastructure, and are deployed via CI/CD processes. All of these are supply chain artifacts that need the same governance as the application code.

ASI04 builds on trust boundaries established by ASI02 (tool-level controls) and ASI03 (identity and privilege). While those controls limit what the agent can do and who it can be, ASI04 asks: is the agent actually running the code you think it's running? The OWASP specification defines 9 prevention and mitigation controls, so let's walk through each one and see how Biotrackr implements (or could implement) them.

## Provenance, SBOMs, and AIBOMs

*"Sign and attest manifests, prompts, and tool definitions; require and operationalize SBOMs, AIBOMs with periodic attestations; maintain inventory of AI components; use curated registries and block untrusted sources."*

Imagine a scenario where a compromised version of `Microsoft.Agents.AI.Anthropic` ships with a subtle change: it silently logs every tool call result to an external endpoint before returning it to the agent. Your unit tests still pass, your deployment succeeds, and the agent behaves normally. You'd never know unless you had a formal inventory of exactly what's in your build and a way to verify it hasn't changed.

That's the problem SBOMs and AIBOMs solve. All manifests involved in your agents (prompts, packages, tool definitions) should be treated as supply chain artifacts, not just configuration. For both your *Software Bill of Materials* (SBOM) and *AI Bill of Materials* (AIBOM), enumerate every software dependency. For AIBOMs, this includes model versions, prompt templates, tool registrations, embedding models, and guardrail configurations. 

Maintain an inventory of AI components, require periodic attestation (not just when you build, do it on a schedule and whenever components change), and use trusted registries.

All NuGet packages come from verified, signed publishers on nuget.org. The project uses no third-party or community package sources:

```xml
<!-- Biotrackr.Chat.Api.csproj — all packages from verified publishers -->
<PackageReference Include="Microsoft.Agents.AI" Version="1.0.0-rc3" />              <!-- Microsoft (verified) -->
<PackageReference Include="Microsoft.Agents.AI.Anthropic" Version="1.0.0-rc3" />     <!-- Microsoft (verified) -->
<PackageReference Include="Microsoft.Azure.Cosmos" Version="3.57.1" />               <!-- Microsoft (verified) -->
<PackageReference Include="Azure.Identity" Version="1.18.0" />                       <!-- Microsoft (verified) -->
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.11.2" /> <!-- OpenTelemetry Authors (verified) -->
```

NuGet package signing provides cryptographic verification of publisher identity. Each package is signed by its publisher and countersigned by nuget.org. The CI/CD pipeline authenticates to Azure via OIDC federated identity (no long-lived secrets), and container images are pushed to a private Azure Container Registry:

```yaml
# deploy-chat-api.yml — OIDC authentication, no stored secrets
permissions:
  id-token: write  # Enables OIDC federation

steps:
  - name: Azure login
    uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Dependency Gatekeeping

*"Allowlist and pin; scan for typosquats (PyPI, npm, LangChain, LlamaIndex); verify provenance before install or activation; auto-reject unsigned or unverified."*

Dependency gatekeeping means only approved, verified packages can enter the dependency graph, and every version is pinned to prevent supply chain drift. Biotrackr implements this through exact version pinning and automated dependency scanning.

Every package in the `.csproj` is pinned to an exact version. No wildcards, no floating versions:

```xml
<!-- Biotrackr.Chat.Api.csproj — exact version pinning -->
<PackageReference Include="Microsoft.Agents.AI" Version="1.0.0-rc3" />
<PackageReference Include="Microsoft.Agents.AI.Anthropic" Version="1.0.0-rc3" />
<PackageReference Include="Microsoft.Agents.AI.Hosting" Version="1.0.0-preview.260304.1" />
<PackageReference Include="Microsoft.Agents.AI.Hosting.AGUI.AspNetCore" Version="1.0.0-preview.260304.1" />
<PackageReference Include="Microsoft.Azure.Cosmos" Version="3.57.1" />
<PackageReference Include="Microsoft.Identity.Web.AgentIdentities" Version="4.5.0" />
<PackageReference Include="Azure.Identity" Version="1.18.0" />
```

Dependabot scans three package ecosystems weekly and creates PRs for vulnerable or outdated packages:

```yaml
# .github/dependabot.yml — automated dependency scanning
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

Treat every package, even ones that are still in preview, with the same discipline as you would with stable packages. Using wildcards for package versions has the potential for vulnerable or compromised builds being pulled into the agent's code.

If you're hosting your code on GitHub, you can use Dependabot PRs to trigger the full CI pipeline to ensure that any updated packages don't introduce breaking changes into your agent code. Dependabot can scan other parts of your supply chain such as GitHub Action versions, and base Docker images.

Regarding `NuGet.config` files, you can add this with `<trustedSigners>` to enforce that packages can only be pulled from trusted publishers:

```xml
<!-- Recommended NuGet.config addition -->
<configuration>
  <trustedSigners>
    <repository name="nuget.org" serviceIndex="https://api.nuget.org/v3/index.json">
      <certificate fingerprint="..." hashAlgorithm="SHA256" allowUntrustedRoot="false" />
    </repository>
  </trustedSigners>
</configuration>
```

The goal here is to make it harder for an unsigned or untrusted package to slip into your dependency graph unnoticed. Combined with exact version pinning and Dependabot scanning, this creates multiple checkpoints that a malicious package would need to pass through before reaching your agent's runtime.

## The Amplified Risk of Preview Packages

All of the above applies equally to preview packages, but preview packages carry additional supply chain risk that's worth calling out explicitly. Four of Biotrackr's core dependencies are pre-release:

```xml
<!-- These four packages are all pre-release -->
<PackageReference Include="Microsoft.Agents.AI" Version="1.0.0-rc3" />
<PackageReference Include="Microsoft.Agents.AI.Anthropic" Version="1.0.0-rc3" />
<PackageReference Include="Microsoft.Agents.AI.Hosting" Version="1.0.0-preview.260304.1" />
<PackageReference Include="Microsoft.Agents.AI.Hosting.AGUI.AspNetCore" Version="1.0.0-preview.260304.1" />
```

Preview packages have smaller install bases, which means fewer developers are exercising the code paths and fewer eyes are catching bugs or vulnerabilities. The Agent Framework's `1.0.0-rc3` API surface might change significantly before GA, and breaking changes between preview versions can introduce subtle security regressions that aren't flagged by a CVE.

Notice that we're also tracking two different preview tracks here: `-rc3` and `-preview.260304.1`. That means monitoring two independent release cadences for breaking changes, and each update needs careful review because a new preview could change security-relevant behaviour without any advisory.

The mitigations are the same ones we've already covered (exact version pinning, Dependabot scanning, full CI pipeline on every update), but the discipline matters more. A floating version like `*-preview*` in a `.csproj` could silently pull a compromised or broken build into your agent. Treat preview packages with the same rigour as stable ones, and document exactly which preview version you depend on and why.

## Containment and Reproducible Builds

*"Run sensitive agents in sandboxed containers with strict network or syscall limits; require reproducible builds."*

Agents should run in sandboxed containers with minimal privileges and deterministic builds. Biotrackr implements this through a multi-stage Docker build with non-root execution and lock files for reproducible NuGet restores.

The Chat.Api Dockerfile uses a multi-stage build to minimise the final image's attack surface, where build tools, the SDK, and source code are excluded from the production image:

```dockerfile
# Dockerfile — multi-stage build with non-root execution
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID                    # Non-root execution
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Biotrackr.Chat.Api/Biotrackr.Chat.Api.csproj", "Biotrackr.Chat.Api/"]
RUN dotnet restore "./Biotrackr.Chat.Api/Biotrackr.Chat.Api.csproj"
COPY . .
RUN dotnet build "./Biotrackr.Chat.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
RUN dotnet publish "./Biotrackr.Chat.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Biotrackr.Chat.Api.dll"]
```

For reproducible NuGet builds, `packages.lock.json` records the exact version of every direct and transitive dependency at restore time. In CI, `dotnet restore --locked-mode` fails the build if the lock file doesn't match, preventing silent dependency drift:

```xml
<!-- Recommended .csproj addition for lock file enforcement -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

Some key points about this:

- **Official Microsoft base images** from `mcr.microsoft.com` are used. These are trusted images that are regularly patched.
- **Non-root execution** (`USER $APP_UID`) — the agent process cannot escalate to root or write to system directories.
- **Multi-stage build** — the final image contains only the published .NET runtime and application DLLs, not the SDK or build tooling
- **Dependabot scans Docker base images weekly** — vulnerable base images are flagged for update.
- Lock files ensure the same `dotnet restore` produces identical dependency graphs across CI and local builds.

## Secure Prompts and Memory

*"Put prompts, orchestration scripts, and memory schemas under version control with peer review; scan for anomalies."*

System prompts and memory schemas are code. A tampered prompt can completely alter agent behaviour without changing a single line of application code. Biotrackr treats prompts as infrastructure-as-code artifacts, deploying them through the same CI/CD pipeline as the application itself.

The system prompt is defined as a Bicep parameter and is deployed to Azure App Configuration through the CI/CD pipeline:

```bicep
// infra/apps/chat-api/main.bicep — system prompt as IaC
@description('The system prompt for the chat agent')
param chatSystemPrompt string

// Deployed to App Configuration — not a loose environment variable
resource chatSystemPromptSetting 'Microsoft.AppConfiguration/configurationStores/keyValues@2025-02-01-preview' = {
  name: 'Biotrackr:ChatSystemPrompt'
  parent: appConfig
  properties: {
    value: chatSystemPrompt
  }
}
```

At runtime, the prompt is loaded from App Configuration via managed identity and passed to the agent:

```csharp
// Program.cs — prompt loaded from App Configuration, not hardcoded
var systemPrompt = builder.Configuration.GetValue<string>("Biotrackr:ChatSystemPrompt")!;

AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,
    tools: [ /* 12 registered tools */ ]);
```

Chat history (memory) is persisted to Cosmos DB via `ConversationPersistenceMiddleware`. The schema is defined in code, and tool call names are recorded per-session:

```csharp
// ConversationPersistenceMiddleware — memory schema under version control
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

await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);
```

Prompts are not stored as loose files or environment variables that could be silently modified, and the conversation persistence schema (session ID, role, content, tool calls) is defined in code and reviewed.

## Inter-Agent Security

*"Enforce mutual auth and attestation via PKI and mTLS; no open registration; sign and verify all inter-agent messages."*

In multi-service architectures, every service-to-service call is a supply chain boundary. A compromised downstream API could inject malicious tool results into the agent's reasoning. Biotrackr enforces authentication at every inter-service boundary through layered identity and API gateway controls.

The Chat.Api authenticates to Cosmos DB using Entra Agent ID, a dedicated agent identity separate from the host application:

```csharp
// AgentIdentityCosmosClientFactory — agent authenticates with its own identity
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
```

Downstream API calls (Activity, Sleep, Weight, Food) go through APIM with per-request authentication. The `ApiKeyDelegatingHandler` injects the subscription key on every outbound HTTP call:

```csharp
// ApiKeyDelegatingHandler — per-request auth for downstream APIs
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

APIM validates the identity on every request via a JWT validation policy:

```xml
<!-- policy-jwt-auth.xml — APIM validates JWT or subscription key on every request -->
<validate-jwt header-name="Authorization" failed-validation-httpcode="401">
  <openid-config url="{{openid-config-url}}" />
  <audiences><audience>{{jwt-audience}}</audience></audiences>
  <issuers><issuer>{{jwt-issuer}}</issuer></issuers>
</validate-jwt>
```

The CI/CD pipeline itself uses OIDC federated identity. No long-lived secrets are stored in GitHub:

```yaml
# deploy-chat-api.yml — workload identity federation, no stored secrets
- name: Azure login
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Every service-to-service call requires authentication (JWT Token or a Subscription Key). The Chat.Api cannot call downstream APIs without passing through APIM's authentication gate. The agent identity is scoped with least-privilege RBAC, meaning that it can only access Cosmos DB, not other Azure infrastructure. The build pipeline itself uses OIDC, not stored secrets, which reduces the supply chain risk in the CI/CD layer.

Currently, the Chat.API is the only agent within the Biotrackr system. If a second agent was introduced, each agent would need its own Blueprint, with an independent RBAC and mutual attestation.

## Continuous Validation and Monitoring

*"Re-check signatures, hashes, and SBOMs (incl. AIBOMs) at runtime; monitor behavior, privilege use, lineage, and inter-module telemetry for anomalies."*

Build-time checks are necessary but not sufficient. A supply chain compromise can happen after deployment. Runtime validation means continuously verifying that what's running matches what was deployed, and monitoring for behavioural anomalies that signal tampering. 

Biotrackr implements partial runtime monitoring through OpenTelemetry and Dependabot.

OpenTelemetry provides distributed tracing and metrics across every inbound request and outbound API call:

```csharp
// Program.cs — full observability pipeline
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()    // Traces inbound requests
        .AddHttpClientInstrumentation()    // Traces outbound calls (Claude, APIM)
        .AddOtlpExporter())               // Exports to Azure Monitor
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

The `ConversationPersistenceMiddleware` records every tool call the agent invokes, creating an audit trail of agent behaviour per session:

```csharp
// ConversationPersistenceMiddleware — tool call audit trail
var toolCalls = new List<string>();

await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
{
    foreach (var content in update.Contents)
    {
        if (content is FunctionCallContent functionCall)
        {
            toolCalls.Add(functionCall.Name);  // Records which tools the agent called
        }
    }
    yield return update;
}

// Persists tool call list alongside the response in Cosmos DB
await repository.SaveMessageAsync(sessionId, "assistant", assistantContent,
    toolCalls.Count > 0 ? toolCalls : null);
```

Dependabot provides continuous dependency scanning across three ecosystems:

```yaml
# .github/dependabot.yml — weekly scanning of all dependency types
updates:
  - package-ecosystem: "github-actions"   # CI pipeline actions
    schedule: { interval: "weekly" }
  - package-ecosystem: "nuget"            # .NET packages
    schedule: { interval: "weekly" }
  - package-ecosystem: "docker"           # Base image vulnerabilities
    schedule: { interval: "weekly" }
```

Some key points here:

- **Distributed tracing** captures the full request lifecycle — from user request through APIM to downstream APIs and Claude
- **Tool call logging** creates an audit trail that could detect unexpected tool invocations (a key indicator of supply chain compromise)
- **Dependabot** scans weekly — for preview packages this is especially important, as preview versions may have known issues fixed in newer previews
- GitHub Security Advisories provide early warnings for zero-day vulnerabilities in dependencies

## Pinning Beyond Packages

*"Pin prompts, tools, and configs by content hash and commit ID. Require staged rollout with differential tests and auto-rollback on hash drift or behavioral change."*

Here's a question worth asking: if someone changed the system prompt in Azure App Configuration directly (bypassing your CI/CD pipeline), would you know? The agent would pick up the new prompt on its next restart and behave differently, but no code change would show up in your commit history.

Traditional pinning stops at package versions. Agentic systems need to pin everything that affects behaviour (prompts, tool definitions, configurations, and model parameters) by content hash, not just by name or version.

NuGet packages are pinned to exact versions (see Control 2), and `packages.lock.json` pins the full transitive dependency graph. The system prompt is pinned in Bicep source code and deployed through CI/CD (see Control 4):

```bicep
// infra/apps/chat-api/main.bicep — prompt pinned in IaC
param chatSystemPrompt string
```

Docker base images are pinned to specific major versions:

```dockerfile
# Dockerfile — base image pinned to .NET 10
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
```

Tools are statically registered at startup. The set of 12 tools is defined in code, not loaded dynamically:

```csharp
// Program.cs — static tool registration, tools defined at compile time
AIAgent chatAgent = anthropicClient.AsAIAgent(
    model: modelName,
    name: "BiotrackrChatAgent",
    instructions: systemPrompt,
    tools:
    [
        AIFunctionFactory.Create(activityServices.GetActivityRecordsByDate),
        AIFunctionFactory.Create(activityServices.GetActivityRecordsByDateRange),
        AIFunctionFactory.Create(activityServices.GetAllActivityRecords),
        // ... 9 more tools
    ]);
```

Key things to point out here:

- Package versions are pinned exactly. No floating versions or wildcards.
- The system prompt is version-controlled and deployed through CI/CD. All changes require a PR.
- Tools are statically defined in compiled code. They cannot be modified at runtime without a redeployment.
- Docker base images are Dependabot-monitored and updated through PRs, not silently.

## Supply Chain Kill Switch

*"Implement emergency revocation mechanisms that can instantly disable specific tools, prompts, or agent connections across all deployments when a compromise is detected, preventing further cascading damage."*

When a supply chain compromise is detected, you need to stop the bleeding immediately, not wait for a CI/CD pipeline to redeploy. Kill switches should be pre-built and tested, not improvised during an incident. Biotrackr has multiple layered kill switches already built into its architecture.

The Bicep infrastructure includes a feature flag that controls the authentication policy. Switching it forces a redeployment with a different auth mode:

```bicep
// infra/apps/chat-api/main.bicep — auth policy toggle
@description('Enable JWT validation for managed identity authentication')
param enableManagedIdentityAuth bool = true

// This controls which APIM policy is deployed:
var chatApiPolicy = enableManagedIdentityAuth
  ? loadTextContent('policy-jwt-auth.xml')
  : loadTextContent('policy-subscription-key.xml')
```

A redeployment with `enableManagedIdentityAuth = false` switches the auth policy, effectively killing the agent's JWT-based access to all downstream APIs.

Beyond the Bicep toggle, Biotrackr has two immediate kill switches that require no redeployment:

1. **APIM subscription key revocation** — instantly revoke the Chat.Api's subscription key in the Azure Portal → the agent can no longer call downstream APIs (Activity, Sleep, Weight, Food). Takes effect within minutes.

2. **Agent identity disablement** — disable the agent's Entra ID managed identity → the agent can no longer authenticate to Cosmos DB or Azure App Configuration. The agent immediately loses access to conversation history and its system prompt.

## Zero-Trust Security Model

*"Design system with security fault tolerance that assumes failure or exploitation of LLM or agentic function components."*

Zero-trust means assuming every component in the chain. The LLM, downstream APIs, the prompt store, even the agent framework itself, could be compromised. The architecture should ensure that a single compromised component cannot cascade into full system compromise. 

In Biotrackr, the `ConversationPersistenceMiddleware` wraps the inner agent, processing streaming responses before they reach the user. Tool call results flow through structured JSON, not raw text:

```csharp
// ConversationPersistenceMiddleware — processes responses before they reach the user
public async IAsyncEnumerable<AgentUpdate> RunStreamingAsync(
    IReadOnlyList<ChatMessage> messages,
    AgentSession session,
    RunOptions? options = null,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    var innerAgent = _agentFactory();
    var toolCalls = new List<string>();
    var contentBuilder = new StringBuilder();

    await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
    {
        // Process each update — tool calls are captured, content is accumulated
        foreach (var content in update.Contents)
        {
            if (content is FunctionCallContent functionCall)
                toolCalls.Add(functionCall.Name);
            if (content is Microsoft.Agents.AI.Abstractions.TextContent textContent)
                contentBuilder.Append(textContent.Text);
        }
        yield return update;  // Stream to user only after processing
    }
}
```

HTTP resilience patterns protect against downstream service failures. Circuit breakers prevent cascading failures:

```csharp
// Program.cs — standard resilience handler with circuit breakers
builder.Services.AddHttpClient("ActivityApiClient", client =>
{
    client.BaseAddress = new Uri(activitySettings.BaseUrl);
})
.AddHttpMessageHandler<ApiKeyDelegatingHandler>()
.AddStandardResilienceHandler();  // Retry, circuit breaker, timeout
```

Even internal APIs are accessed through APIM with authentication. There is no "trusted internal network" bypass:

```
Chat.Api → APIM (JWT/subscription key validation) → Activity API
                                                   → Sleep API
                                                   → Weight API
                                                   → Food API
```

## Wrapping up

Agentic Supply Chain Vulnerabilities (ASI04) go far beyond "pin your NuGet packages." The OWASP specification defines 9 controls, and in Biotrackr we implement 5 fully and 4 partially.

The controls are layered: verified publishers → exact version pinning → Dependabot scanning → multi-stage Docker builds → prompts as IaC → APIM authentication gates → OIDC federated identity → OpenTelemetry tracing → kill switches. Even if one layer is compromised, the others limit the blast radius. **Treat every component in your agent's stack as a supply chain artifact: packages, prompts, tool definitions, base images, CI/CD actions, and infrastructure templates.**

There are gaps I haven't addressed yet. SBOM and AIBOM generation (Control 1), runtime hash validation of prompts and tool registrations (Controls 6 and 7), and per-tool kill switches via feature flags (Control 8) are all things that I'll introduce in the future (and you should defintiely implement where appropriate). If you're using dynamic tool loading, MCP servers, or multi-agent architectures, those controls become critical rather than nice-to-have. You're trusting code that wasn't compiled into your application.

What makes ASI04 unique compared to the other controls we've covered is that it assumes the code you're running might not be what you think it is. ASI02 constrains what the agent can do with its tools. ASI03 constrains who the agent can be. ASI04 asks: are those tools and identities actually the ones you deployed, or has something been tampered with along the way?

In the next post in this series, I'll cover **ASI05 and ASI06 — Unexpected Code Execution and Memory Poisoning**, which explore what happens when agents execute untrusted code or when their conversation history is weaponised against them.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
