---
title: "Preventing Insecure Inter-Agent Communication in AI Agents"
date: 2026-03-12
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework", "APIM", "Managed Identity", "Entra Agent ID", "A2A", "MCP"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p3wpof34c8y2b5lox81v.png
    alt: "Preventing OWASP ASI07 Insecure Inter-Agent Communication in a .NET AI agent with mutual authentication, signed messages, anti-replay, typed contracts, and protocol pinning."
    caption: "Implementing OWASP ASI07 mitigations against Insecure Inter-Agent Communication in a .NET 10 AI agent built with the Microsoft Agent Framework."
---

[Biotrackr](https://github.com/willvelida/biotrackr) is a single-agent system. One agent, twelve tools, one identity. That is an architectural choice that eliminates an entire vulnerability class **Insecure Inter-Agent Communication (ASI07)**. But what happens when the system grows?

Imagine Biotrackr evolves into a multi-agent platform: a **Data Retrieval Agent** that fetches health records, a **Health Advisor Agent** that provides wellness recommendations based on trends, and an **Orchestrator Agent** that coordinates them. Suddenly, agents are talking to each other, passing data, delegating tasks, sharing context. Every message between them is a potential attack surface.

Even though ASI07 doesn't apply to Biotrackr today, understanding these risks early prevents insecure patterns from being baked into the architecture when multi-agent requirements arrive. The mitigations (mutual authentication, signed messages, schema validation) benefit any distributed system, not just multi-agent AI.

In this article, we'll cover **Insecure Inter-Agent Communication** and how we could implement prevention and mitigation strategies if Biotrackr were a multi-agent system. We'll ground each control in hypothetical but concrete .NET code that builds on Biotrackr's existing architecture.

## What is Insecure Inter-Agent Communication?

Insecure Inter-Agent Communication occurs when exchanges between agents lack proper authentication, integrity, or semantic validation, allowing interception, spoofing, or manipulation of agent messages and intents. Multi-agent systems depend on continuous communication between autonomous agents that coordinate via APIs, message buses, and shared memory, significantly expanding the attack surface.

The threat spans multiple layers:

1. **Transport layer** — unencrypted channels enabling message interception and injection
2. **Routing layer** — misdirected discovery traffic creating fake agent relationships
3. **Semantic layer** — modified natural-language instructions altering agent goals mid-conversation
4. **Side-channel layer** — timing and behavioral cues leaking agent decision patterns

There are several ways this can be exploited:

- **MITM injection** — an attacker intercepts unencrypted messages between agents and injects hidden instructions that alter agent goals and decision logic
- **Message tampering** — modified or injected messages blur task boundaries between agents, leading to data leakage or goal confusion during coordination
- **Replay attacks** — replayed delegation or trust messages trick agents into granting access or honoring stale instructions
- **Protocol downgrade** — attackers coerce agents into weaker communication modes, making malicious commands appear as valid exchanges
- **Discovery spoofing** — misdirected discovery traffic forges relationships with malicious agents or unauthorized coordinators

This is different from ASI03 (Identity & Privilege Abuse), which focuses on credential and permissions misuse, and ASI06 (Memory & Context Poisoning), which targets stored knowledge corruption. ASI07 focuses on compromising **real-time messages** between agents, leading to misinformation, privilege confusion, or coordinated manipulation across distributed agentic systems.

## Why does this matter (even for a single-agent system)?

Why think about this for my little side project?

Most agent systems start as single-agent and evolve into multi-agent as requirements grow. Today, Biotrackr has one agent that fetches health data and provides analysis. But the moment I want to add a Health Advisor Agent that interprets trends, or a Goal Tracking Agent that monitors fitness goals, or a Notification Agent that sends alerts when metrics deviate; I've introduced inter-agent communication, and an entire new class of vulnerabilities with it.

Understanding inter-agent communication risks now means I can design for them when the time comes, rather than retrofitting it later. The mitigations we'll walk through (mutual authentication, signed messages, typed contracts) are good distributed systems practices regardless of whether agents are involved.

ASI07 builds on the identity controls we implemented in ASI03 (Entra Agent ID), the tool-level constraints from ASI02, and the supply chain guarantees from ASI04. While those controls limit what an agent can do, who it can be, and whether it's running trusted code, ASI07 asks: are the messages between agents actually what they claim to be?

The OWASP specification defines 9 prevention and mitigation guidelines. Let's walk through each one and see how a multi-agent Biotrackr could implement them.

## The Hypothetical: Multi-Agent Biotrackr

To ground each guideline in concrete code, we'll work with a hypothetical three-agent Biotrackr architecture:

| Agent | Role | Tools | Identity |
|-------|------|-------|----------|
| **Orchestrator** | Routes user questions to specialist agents, combines responses | `RouteToDataAgent()`, `RouteToAdvisorAgent()` | `biotrackr-orchestrator-agent` (Entra Agent ID) |
| **Data Retrieval** | Fetches health records from APIM (current Biotrackr agent) | 12 existing tools (activity, sleep, weight, food) | `biotrackr-data-agent` (Entra Agent ID) |
| **Health Advisor** | Analyzes trends, provides wellness recommendations | `AnalyzeTrends()`, `GenerateRecommendation()` | `biotrackr-advisor-agent` (Entra Agent ID) |

**Communication flow:**
1. User asks: *"How has my sleep quality changed this month, and what should I do about it?"*
2. Orchestrator routes the data retrieval part to the Data Retrieval Agent
3. Data Retrieval Agent fetches sleep records via APIM, returns structured data
4. Orchestrator passes the data to the Health Advisor Agent
5. Health Advisor Agent analyzes trends and returns a recommendation
6. Orchestrator combines both responses and streams the answer to the user via AG-UI

Every arrow in that flow is an attack surface. Let's walk through the 9 prevention guidelines.

## Secure Agent Channels

*"Use end-to-end encryption with per-agent credentials and mutual authentication. Enforce PKI certificate pinning, forward secrecy, and regular protocol reviews to prevent interception or spoofing."*

In the current single-agent architecture, the Chat API calls APIM over HTTPS with a subscription key. Transport encryption exists but mutual authentication does not. In a multi-agent system, each agent would need to authenticate to the others.

### Per-Agent mTLS via Azure Container Apps

Each agent would run as a separate Container App with its own managed identity. Inter-agent calls would use mTLS with certificate pinning:

```csharp
// Each agent runs as a separate Container App with its own managed identity
// Inter-agent calls use mTLS with certificate pinning
builder.Services.AddHttpClient("DataRetrievalAgent", (sp, client) =>
{
    client.BaseAddress = new Uri("https://biotrackr-data-agent.internal.azurecontainerapps.io");
})
.ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
{
    ClientCertificateOptions = ClientCertificateOption.Automatic,
    ServerCertificateCustomValidationCallback = (message, cert, chain, errors) =>
    {
        // Pin to the Data Retrieval Agent's specific certificate thumbprint
        return cert?.GetCertHashString() == expectedDataAgentThumbprint;
    }
});
```

### Per-Agent Entra Agent ID credentials

Each agent would use its own Agent Identity (from ASI03) for inter-agent calls. The Orchestrator would acquire a token scoped to the Data Retrieval Agent's audience:

```csharp
// Orchestrator acquires a token to call the Data Retrieval Agent
var credential = new MicrosoftIdentityTokenCredential();
credential.Options.WithAgentIdentity(orchestratorAgentIdentityId);
credential.Options.RequestAppToken = true;

var token = await credential.GetTokenAsync(
    new TokenRequestContext(new[] { "api://biotrackr-data-agent/.default" }),
    cancellationToken);

// Include token in inter-agent request
httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token.Token);
```

Some key points here:

- mTLS ensures both sides verify identity — the Data Retrieval Agent rejects connections from unknown agents
- Certificate pinning prevents a compromised CA from issuing rogue certificates
- Azure Container Apps internal networking keeps inter-agent traffic off the public internet
- Each agent has its own Entra Agent ID — if the Orchestrator is compromised, it cannot impersonate the Data Retrieval Agent's identity

This builds directly on Biotrackr's existing identity architecture. The `AgentIdentityCosmosClientFactory` already shows how to acquire tokens as a specific agent identity, extending this to inter-agent authentication is the same pattern with a different audience.

What's missing is forward secrecy and automated certificate rotation. The mTLS setup pins to a static thumbprint, in production, you'd want ephemeral Diffie-Hellman key exchange to ensure that even if a long-term key is compromised, past session traffic cannot be decrypted. Certificate rotation should be automated via Azure Key Vault with a grace period where both old and new certificates are accepted during rollover. There's also no regular protocol review cadence. A scheduled audit (quarterly, for example) of cipher suites, TLS versions, and certificate expiry would catch configuration drift before it becomes exploitable.

## Message Integrity and Semantic Protection

*"Digitally sign messages, hash both payload and context, and validate for hidden or modified natural-language instructions. Apply natural-language-aware sanitization and intent-diffing to detect goal, parameter tampering, hidden or modified natural-language instructions."*

Beyond transport encryption, each message between agents would be digitally signed to prevent tampering. If an attacker somehow gets inside the network (or an agent is compromised), they still can't modify messages without breaking the signature.

### Signed Inter-Agent Messages

```csharp
public class SignedAgentMessage
{
    public string SenderId { get; set; }        // Agent identity ID
    public string RecipientId { get; set; }      // Target agent identity ID
    public string Payload { get; set; }          // JSON-serialized request/response
    public string PayloadHash { get; set; }      // SHA-256 hash of Payload
    public string ContextHash { get; set; }      // Hash of conversation context at time of sending
    public string Signature { get; set; }        // RSA signature over PayloadHash + ContextHash
    public DateTimeOffset Timestamp { get; set; }
    public string Nonce { get; set; }            // Anti-replay (see Guideline 3)
}

public class AgentMessageValidator
{
    public bool ValidateMessage(SignedAgentMessage message, RSA senderPublicKey)
    {
        // Verify payload hasn't been tampered with
        var computedHash = SHA256.HashData(Encoding.UTF8.GetBytes(message.Payload));
        if (Convert.ToBase64String(computedHash) != message.PayloadHash)
            return false;

        // Verify signature was produced by the claimed sender
        var dataToVerify = Encoding.UTF8.GetBytes(message.PayloadHash + message.ContextHash);
        return senderPublicKey.VerifyData(
            dataToVerify,
            Convert.FromBase64String(message.Signature),
            HashAlgorithmName.SHA256,
            RSASignaturePadding.Pkcs1);
    }
}
```

### Semantic Validation (Intent-Diffing)

Beyond cryptographic integrity, we'd also validate that the semantic intent of a message hasn't been altered. This is relevant because an agent might be compromised and produce cryptographically valid but semantically wrong responses:

```csharp
public class SemanticIntentValidator
{
    // Validates that the Data Agent's response matches the original query intent
    public bool ValidateResponseIntent(
        string agentResponse,
        string expectedDataType)  // "sleep", "activity", etc.
    {
        // Structural check: response should contain expected data type
        using var doc = JsonDocument.Parse(agentResponse);
        var root = doc.RootElement;

        // Reject responses that contain unexpected data types
        // (e.g., the data agent returning weight data when sleep was requested)
        if (!root.TryGetProperty(expectedDataType, out _))
            return false;

        // Reject responses with suspiciously large payloads (data exfiltration attempt)
        if (agentResponse.Length > MaxExpectedResponseSize)
            return false;

        return true;
    }
}
```

Some key points here:

- Cryptographic signing ensures a message from the Data Retrieval Agent was actually produced by it — not injected by an attacker
- Context hashing ties the message to the conversation state — replaying a message in a different context fails validation
- Semantic validation catches cases where the message is cryptographically valid but semantically wrong (e.g., a data agent returning manipulated health records that pass signature checks because the agent itself was compromised)
- This is the same trust boundary approach we already use in Biotrackr — tool results are treated as untrusted input. In a multi-agent system, agent responses get the same treatment

What's missing is NLP-aware sanitization for detecting hidden instructions embedded in natural-language payloads. The semantic validator checks structural properties (expected data type, payload size) but doesn't scan for prompt injection patterns within agent messages e.g., a compromised Data Agent embedding "ignore previous instructions" within a JSON field value. For production multi-agent systems, you'd want a dedicated sanitization layer that scans agent message payloads for known injection patterns before they reach the receiving agent's LLM context. Automated intent-diffing, comparing the original request intent against the response intent using embedding similarity, would provide an additional detection layer beyond structural validation.

## Agent-Aware Anti-Replay

*"Protect all exchanges with nonces, session identifiers, and timestamps tied to task windows. Maintain short-term message fingerprints or state hashes to detect cross-context replays."*

Consider this attack: an attacker captures a legitimate response from the Data Retrieval Agent ("sleep quality score: 85/100 for March 2026") and replays it a month later when the actual data has changed. The Health Advisor Agent would provide recommendations based on stale data with potentially harmful advice.

```csharp
public class AntiReplayMiddleware
{
    private readonly IDistributedCache _messageFingerprints;
    private readonly TimeSpan _taskWindow = TimeSpan.FromMinutes(5);

    public async Task<bool> ValidateAndRecordAsync(SignedAgentMessage message)
    {
        // 1. Check timestamp is within the task window
        if (DateTimeOffset.UtcNow - message.Timestamp > _taskWindow)
            return false; // Message too old — possible replay

        // 2. Check nonce hasn't been seen before
        var nonceKey = $"nonce:{message.SenderId}:{message.Nonce}";
        var existing = await _messageFingerprints.GetStringAsync(nonceKey);
        if (existing is not null)
            return false; // Nonce already used — replay detected

        // 3. Record nonce with expiry matching 2x the task window
        await _messageFingerprints.SetStringAsync(
            nonceKey,
            message.Timestamp.ToString("O"),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _taskWindow * 2
            });

        // 4. Compute and store message fingerprint for cross-context detection
        var fingerprint = ComputeFingerprint(message);
        var fingerprintKey = $"fingerprint:{fingerprint}";
        await _messageFingerprints.SetStringAsync(
            fingerprintKey,
            message.SenderId,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _taskWindow * 2
            });

        return true;
    }

    private static string ComputeFingerprint(SignedAgentMessage message)
    {
        var data = $"{message.SenderId}:{message.PayloadHash}:{message.ContextHash}";
        return Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes(data)));
    }
}
```

Some key points here:

- **Nonces** ensure each message is unique — replaying the exact same message fails the nonce check
- **Timestamps** tied to task windows (5 minutes) reject messages older than the current task scope — stale data cannot be injected
- **Message fingerprints** detect cross-context replays where the same payload is sent to a different agent or session
- **Redis/distributed cache** ensures replay detection works across Container App replicas — if the Orchestrator has multiple instances, all replicas share the same nonce store

This pattern is familiar if you've worked with API idempotency keys or CSRF tokens. The difference in a multi-agent system is that the "client" is another agent, not a browser.

What's missing is distributed cache high-availability. If the Redis/distributed cache goes down, the anti-replay middleware has no state to check against, opening a window for replay attacks during the outage. A production system would need cache replication across availability zones, fallback to local in-memory nonce tracking (accepting the risk of per-instance-only detection), and alerting when the distributed cache is unavailable. Cross-region replay detection is also absent. If the agents span multiple Azure regions, the nonce store needs to be globally consistent or region-aware to prevent cross-region replay of captured messages.

## Protocol and Capability Security

*"Disable weak or legacy communication modes. Require agent-specific trust negotiation and bind protocol authentication to agent identity. Enforce version and capability policies at gateways or middleware."*

In the current architecture, APIM enforces protocol policies for external API access. In a multi-agent system, the same principle extends to inter-agent communication. Each agent should only accept connections that use approved protocols and come from agents with matching capabilities.

```csharp
public class ProtocolEnforcementMiddleware
{
    private static readonly HashSet<string> AllowedProtocolVersions = new()
    {
        "biotrackr-agent-protocol/1.0",
        "biotrackr-agent-protocol/1.1"
    };

    public async Task<bool> ValidateProtocol(HttpRequest request)
    {
        // Reject requests without protocol version header
        if (!request.Headers.TryGetValue("X-Agent-Protocol-Version", out var version))
            return false;

        // Reject unknown or legacy protocol versions
        if (!AllowedProtocolVersions.Contains(version.ToString()))
            return false;

        // Reject non-TLS connections (defence in depth — infra should enforce this too)
        if (!request.IsHttps)
            return false;

        // Validate the caller's agent identity matches a known agent
        var agentId = request.Headers["X-Agent-Identity-Id"].ToString();
        if (!await IsRegisteredAgent(agentId))
            return false;

        return true;
    }
}
```

Before an agent delegates a task, it should verify the target agent's capabilities match the request:

```csharp
public class AgentCapabilityRegistry
{
    private readonly Dictionary<string, AgentCapabilities> _registry = new()
    {
        ["biotrackr-data-agent"] = new AgentCapabilities
        {
            SupportedOperations = ["GetActivityByDate", "GetSleepByDate", /* ... */],
            MaxPayloadSize = 1_048_576, // 1MB
            ProtocolVersions = ["biotrackr-agent-protocol/1.0", "biotrackr-agent-protocol/1.1"],
            AllowedDataTypes = ["activity", "sleep", "weight", "food"]
        },
        ["biotrackr-advisor-agent"] = new AgentCapabilities
        {
            SupportedOperations = ["AnalyzeTrends", "GenerateRecommendation"],
            MaxPayloadSize = 524_288, // 512KB
            ProtocolVersions = ["biotrackr-agent-protocol/1.1"],
            AllowedDataTypes = ["analysis", "recommendation"]
        }
    };

    public bool CanHandle(string agentId, string operation)
    {
        return _registry.TryGetValue(agentId, out var caps)
            && caps.SupportedOperations.Contains(operation);
    }
}
```

Some key points here:

- Legacy protocol versions are explicitly rejected — no downgrade path. If an attacker forces the Orchestrator into a "legacy compatibility mode" that uses unencrypted HTTP, the middleware rejects it
- Agent identity is bound to protocol authentication — a valid TLS connection from an unknown agent ID is still rejected
- Capability negotiation prevents the Orchestrator from sending unsupported operations to an agent (e.g., sending `DeleteRecord` to the Data Retrieval Agent, which only supports read operations)
- This extends the same pattern Biotrackr already uses. APIM validates subscription keys and JWTs on every request. In a multi-agent system, APIM (or equivalent middleware) validates the inter-agent protocol too

What's missing is dynamic capability updates and an automated protocol review cadence. The capability registry is hardcoded. Adding a new operation to the Data Retrieval Agent requires redeploying the Orchestrator. A production system would load the capability registry from a central configuration store (like Azure App Configuration) with change notifications, so that capability changes propagate without redeployment. Regular automated protocol reviews (scanning for deprecated cipher suites, expired capability entries, or unused protocol versions) would catch configuration drift before it becomes a vulnerability.

## Limit Metadata-Based Inference

*"Reduce the attack surface for traffic analysis by using fixed-size or padded messages where feasible, smoothing communication rates, and avoiding deterministic communication schedules. These lightweight measures make it harder for attackers to infer agent roles or decision cycles from metadata alone, without requiring heavy protocol redesign."*

Even without reading message contents, an attacker observing inter-agent traffic patterns could infer useful information. This is particularly relevant in a health data application. Traffic patterns could reveal what kind of health data a user is querying and when:

- **Which agent is being consulted** — message size varies by domain (sleep data payloads differ from food data)
- **User behavior patterns** — the user checks sleep data every morning, weight data on Mondays
- **Decision cycles** — the Orchestrator always calls Data Agent before Advisor Agent, and the timing reveals the workflow

```csharp
public class TrafficNormalizationHandler : DelegatingHandler
{
    private const int PaddedMessageSize = 8192; // 8KB fixed-size messages

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        // Add jitter to avoid deterministic timing
        var jitter = Random.Shared.Next(50, 150); // 50-150ms random delay
        await Task.Delay(TimeSpan.FromMilliseconds(jitter), cancellationToken);

        // Pad request payload to fixed size to prevent size-based inference
        if (request.Content is not null)
        {
            var content = await request.Content.ReadAsStringAsync(cancellationToken);
            var padded = PadToFixedSize(content, PaddedMessageSize);
            request.Content = new StringContent(padded, Encoding.UTF8, "application/json");
        }

        return await base.SendAsync(request, cancellationToken);
    }

    private static string PadToFixedSize(string content, int targetSize)
    {
        if (content.Length >= targetSize)
            return content; // Truncation handled separately

        // Add padding field to JSON (stripped by receiver)
        var paddingLength = targetSize - content.Length - 15; // Account for JSON key
        if (paddingLength <= 0) return content;

        var padding = new string(' ', paddingLength);
        return content.TrimEnd('}') + $",\"_pad\":\"{padding}\"}}";
    }
}
```

I'll be honest, this guideline is the lightest of the bunch. For a side project like Biotrackr, the latency impact of padding and jitter probably isn't worth the complexity. But for production multi-agent systems handling sensitive data (healthcare, finance, legal), traffic analysis resistance is a real concern.

Some key points here:

- Fixed-size messages prevent size-based inference — an observer cannot tell if the agent is fetching a single day's data or a month's worth
- Communication rate smoothing with random jitter prevents timing analysis — the Orchestrator doesn't reveal its decision pattern
- Azure Container Apps internal networking already limits external visibility, but defence in depth applies
- For large payloads (a month of food records), chunking into fixed-size blocks is more practical than single-message padding

What's missing is response padding (the code only pads requests), variable-rate scheduling for background tasks, and decoy traffic generation. The jitter range (50-150ms) is narrow enough that statistical analysis over many requests could still reveal timing patterns. A production system might use a wider jitter distribution or inject decoy inter-agent messages that carry no real data but normalise the traffic pattern. For chunked large payloads, you'd also want consistent chunk counts to prevent observers from inferring data volume from the number of network round-trips.

## Protocol Pinning and Version Enforcement

*"Define and enforce allowed protocol versions (e.g., MCP, A2A, gRPC). Reject downgrade attempts or unrecognized schemas and validate that both peers advertise matching capability and version fingerprints."*

If multi-agent Biotrackr used Google's A2A (Agent-to-Agent) protocol or MCP for inter-agent communication, version pinning would prevent downgrade attacks. This is the gateway-level enforcement counterpart to the middleware-level checks in Guideline 4.

```xml
<!-- APIM policy for inter-agent protocol enforcement -->
<inbound>
    <base />
    <!-- Reject inter-agent calls with unsupported protocol versions -->
    <choose>
        <when condition="@(!new[] { &quot;biotrackr-agent-protocol/1.0&quot;, &quot;biotrackr-agent-protocol/1.1&quot; }
                           .Contains(context.Request.Headers.GetValueOrDefault(
                               &quot;X-Agent-Protocol-Version&quot;, &quot;&quot;)))">
            <return-response>
                <set-status code="426" reason="Upgrade Required" />
                <set-body>{"error": "Unsupported agent protocol version"}</set-body>
            </return-response>
        </when>
    </choose>
    <!-- Reject requests missing agent version fingerprint -->
    <check-header name="X-Agent-Version-Fingerprint" failed-check-httpcode="400"
                   failed-check-error-message="Missing agent version fingerprint" />
</inbound>
```

And the corresponding application-level enforcement:

```csharp
public class ProtocolPinningOptions
{
    // Explicitly pinned protocol versions — no wildcards, no "latest"
    public Dictionary<string, string[]> AllowedVersions { get; set; } = new()
    {
        ["a2a"] = ["2026.1", "2026.2"],       // Only these A2A versions
        ["mcp"] = ["2025-03-26"],              // Only this MCP spec version
        ["grpc"] = ["1.70.0", "1.71.0"],       // Only these gRPC versions
    };

    // Reject agents that don't advertise a version fingerprint
    public bool RequireVersionFingerprint { get; set; } = true;

    // Reject schema down-conversion attempts
    public bool RejectSchemaDowngrade { get; set; } = true;
}
```

Some key points here:

- Protocol versions are pinned to a known-good set — not "latest" or ">=1.0"
- APIM acts as the gateway enforcer — even if the agent code has a bug that accepts legacy protocols, the gateway blocks it
- Version fingerprints include the agent's protocol implementation hash — ensuring both peers run compatible code
- Downgrade attempts return HTTP 426 (Upgrade Required), not a silent fallback

This is similar to how we pin NuGet package versions in ASI04 (Supply Chain Vulnerabilities). The principle is the same: if you don't control which version is in use, an attacker might force you onto a version with known vulnerabilities.

Biotrackr already uses APIM as a gateway with policy enforcement (JWT validation, subscription keys from ASI03). Extending this to inter-agent protocol validation is a natural evolution of the existing architecture.

What's missing is automated version compatibility testing and a formal deprecation workflow. When a protocol version needs to be retired, there's no process for gracefully deprecating it. Agents running the old version would be immediately rejected. A production system would need a deprecation window where both old and new versions are accepted, with monitoring to ensure all agents have upgraded before the old version is removed. Automated compatibility tests in the CI/CD pipeline would verify that agents build and pass integration tests against the current pinned versions before deployment.

## Discovery and Routing Protection

*"Authenticate all discovery and coordination messages using cryptographic identity. Secure directories with access controls and verified reputations, validate identity and intent end-to-end, and monitor for anomalous routing flows."*

In a multi-agent system, agents need to discover each other. Without protection, an attacker could register a malicious agent that intercepts traffic or masquerades as a legitimate one.

Imagine this scenario: an attacker registers a fake "Data Retrieval Agent" in the discovery service. The Orchestrator routes a user's sleep data query to the fake agent, which returns manipulated data (showing "sleep quality: excellent" when it's actually poor). The Health Advisor Agent then provides bad recommendations based on the false data. The user never knows.

```csharp
public class SecureAgentRegistry
{
    private readonly CosmosClient _cosmosClient; // Registry stored in dedicated Cosmos container
    private readonly ILogger<SecureAgentRegistry> _logger;

    public async Task<AgentRegistration?> DiscoverAgent(
        string agentRole,
        string requestingAgentId,
        string requestingAgentSignature)
    {
        // 1. Verify the requesting agent's identity
        if (!await VerifyAgentIdentity(requestingAgentId, requestingAgentSignature))
        {
            _logger.LogWarning(
                "Discovery request from unverified agent: {AgentId}", requestingAgentId);
            return null;
        }

        // 2. Look up agent by role in the secure registry
        var query = new QueryDefinition(
            "SELECT * FROM c WHERE c.role = @role AND c.status = 'active'")
            .WithParameter("@role", agentRole);

        var container = _cosmosClient.GetContainer("biotrackr", "agent-registry");
        var iterator = container.GetItemQueryIterator<AgentRegistration>(query);
        var results = await iterator.ReadNextAsync();

        var agent = results.FirstOrDefault();
        if (agent is null) return null;

        // 3. Verify the discovered agent's registration is still valid
        if (agent.RegistrationExpiry < DateTimeOffset.UtcNow)
        {
            _logger.LogWarning(
                "Discovered agent {AgentId} has expired registration", agent.AgentId);
            return null;
        }

        // 4. Verify certificate chain
        if (!await VerifyCertificateChain(agent.CertificateThumbprint))
        {
            _logger.LogWarning(
                "Discovered agent {AgentId} has invalid certificate", agent.AgentId);
            return null;
        }

        return agent;
    }
}

public class AgentRegistration
{
    public string AgentId { get; set; }                // Entra Agent Identity ID
    public string Role { get; set; }                   // "data-retrieval", "health-advisor"
    public string Endpoint { get; set; }               // Internal Container App URL
    public string CertificateThumbprint { get; set; }  // mTLS certificate
    public string[] SupportedOperations { get; set; }  // Capability list
    public DateTimeOffset RegistrationExpiry { get; set; }
    public string RegistrationSignature { get; set; }  // Signed by infrastructure admin
}
```

Some key points here:

- The agent registry is stored in a dedicated Cosmos container with its own RBAC — only the infrastructure admin can write to it
- Discovery requests require cryptographic identity verification — an unregistered agent cannot query the registry
- Agent registrations have expiry dates — stale entries are automatically rejected
- Registration signatures are produced by the infrastructure admin's key, not the agent itself — an agent cannot self-register
- Cosmos DB parameterised queries prevent injection — the same pattern Biotrackr already uses for chat history queries

The fake agent scenario fails here because the attacker's agent registration would lack a valid infrastructure admin signature and have no matching Entra Agent ID.

What's missing is anomalous routing flow monitoring and reputation scoring. The registry validates identity and certificates but doesn't track behavioural patterns e.g., an agent that suddenly starts querying for agents outside its normal interaction pattern. A production system would want routing flow monitoring that detects anomalies like an agent discovering agents it's never communicated with before, or discovery requests at unusual times. Reputation scoring based on historical behaviour (uptime, response quality, discovery patterns) would provide a soft trust signal beyond binary identity verification.

## Attested Registry and Agent Verification

*"Use registries or marketplaces that provide digital attestation of agent identity, provenance, and descriptor integrity. Require signed agent cards and continuous verification before accepting discovery or coordination messages. Leverage the PKI trusted root certificate registries to enable robust agent verification and attestation of critical attributes."*

Building on Guideline 7, each agent would publish a signed "agent car. I machine-readable descriptor of its identity, capabilities, and provenance. Think of it like a digital passport for the agent.

```csharp
public class AgentCard
{
    public string AgentId { get; set; }                  // Entra Agent Identity ID
    public string BlueprintId { get; set; }              // Entra Agent Blueprint ID
    public string DisplayName { get; set; }              // "Biotrackr Data Retrieval Agent"
    public string Version { get; set; }                  // "1.2.0"
    public string[] SupportedProtocols { get; set; }     // ["biotrackr-agent-protocol/1.1"]
    public string[] SupportedOperations { get; set; }    // ["GetActivityByDate", ...]
    public string ContainerImageDigest { get; set; }     // sha256:abc123... — provenance
    public string BuildPipelineUrl { get; set; }         // GitHub Actions run URL
    public DateTimeOffset IssuedAt { get; set; }
    public DateTimeOffset ExpiresAt { get; set; }
    public string Signature { get; set; }                // Signed by CI/CD pipeline identity
}

public class AgentCardVerifier
{
    private readonly RSA _cicdPublicKey; // CI/CD pipeline's signing key

    public bool Verify(AgentCard card)
    {
        // Reject expired cards
        if (DateTimeOffset.UtcNow > card.ExpiresAt)
            return false;

        // Verify the card was signed by the CI/CD pipeline (not self-signed)
        var cardData = JsonSerializer.Serialize(card with { Signature = "" });
        var dataBytes = Encoding.UTF8.GetBytes(cardData);
        var signatureBytes = Convert.FromBase64String(card.Signature);

        return _cicdPublicKey.VerifyData(
            dataBytes, signatureBytes,
            HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
    }
}
```

The key difference between agent cards and the discovery registry is **continuous verification**. Discovery happens once when agents first connect. Agent cards are verified periodically to catch agents whose cards expire or whose certificates are revoked mid-session:

```csharp
// Orchestrator verifies agent cards periodically, not just at discovery time
public class ContinuousAgentVerification : BackgroundService
{
    private readonly AgentCardVerifier _verifier;
    private readonly List<AgentRegistration> _knownAgents;
    private readonly ILogger<ContinuousAgentVerification> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            foreach (var agent in _knownAgents)
            {
                var card = await FetchAgentCard(agent.Endpoint);
                if (!_verifier.Verify(card))
                {
                    _logger.LogCritical(
                        "Agent {AgentId} failed continuous verification — revoking trust",
                        agent.AgentId);
                    await RevokeAgentTrust(agent.AgentId);
                }
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

Some key points here:

- Agent cards include the container image digest — the Orchestrator can verify that the agent is running the expected code, not a tampered image. This ties directly into ASI04's supply chain controls
- Cards are signed by the CI/CD pipeline identity, not the agent itself — a compromised agent cannot forge its own attestation
- Continuous verification (every 5 minutes) catches agents whose cards expire or whose certificates are revoked mid-session
- The `BlueprintId` ties back to Entra Agent ID — the card's identity claims can be cross-referenced with Entra sign-in logs for audit
- Biotrackr's GitHub Actions CI/CD pipeline already builds and deploys container images — adding a signing step to produce agent cards is an incremental addition

What's missing is PKI trusted root certificate registry integration for hierarchical trust. The agent card verification uses a single CI/CD signing key. If that key is compromised, all agent cards become untrustworthy. Integrating with PKI trusted root certificate registries would provide a chain of trust where agent attestation traces back to a trusted root authority, rather than a single key. Cross-organisation attestation (for multi-tenant or federated agent systems) is also absent. The current model assumes all agents are operated by the same organisation. Hardware-backed attestation (TPM or Azure Confidential Computing) for the signing keys would provide the highest assurance level for critical attributes like agent identity and provenance.

## Typed Contracts and Schema Validation

*"Use versioned, typed message schemas with explicit per-message audiences. Reject messages that fail validation or attempt schema down-conversion without declared compatibility. Typed contracts help with structure, but semantic divergence across agents remains an inherent challenge; mitigations therefore focus on integrity, provenance, and controlled communication patterns rather than attempting full semantic alignment."*

This is probably the most impactful guideline for day-to-day development. Instead of passing free-form strings between agents (which is what happens when agents communicate via natural language), each inter-agent message uses a versioned, typed contract.

Why does this matter? Consider the "semantics split-brain" attack: a single instruction is parsed into divergent intents by different agents, producing conflicting but seemingly legitimate actions. With free-form text, the Data Retrieval Agent and Health Advisor Agent might interpret the same message differently. With typed contracts, both agents parse the message into the same strongly-typed structure.

```csharp
// Shared contract library: Biotrackr.AgentContracts

[JsonDerivedType(typeof(DataRetrievalRequest), "dataRetrieval")]
[JsonDerivedType(typeof(TrendAnalysisRequest), "trendAnalysis")]
public abstract record AgentMessage
{
    public required string SchemaVersion { get; init; }   // "1.0", "1.1"
    public required string SenderId { get; init; }        // Entra Agent Identity ID
    public required string RecipientId { get; init; }     // Explicit audience
    public required string CorrelationId { get; init; }   // Trace correlation
    public required DateTimeOffset Timestamp { get; init; }
}

public record DataRetrievalRequest : AgentMessage
{
    public required string DataType { get; init; }    // "sleep", "activity", "weight", "food"
    public required DateOnly StartDate { get; init; }
    public required DateOnly EndDate { get; init; }
    public int PageSize { get; init; } = 10;  // Capped at 50 per ASI02 constraints
}

public record DataRetrievalResponse : AgentMessage
{
    public required string DataType { get; init; }
    public required JsonElement Data { get; init; }   // Typed health data
    public required int RecordCount { get; init; }
    public required bool IsComplete { get; init; }    // Pagination indicator
}

public record TrendAnalysisRequest : AgentMessage
{
    public required string DataType { get; init; }
    public required DataRetrievalResponse SourceData { get; init; } // Provenance chain
    public required string AnalysisType { get; init; } // "weekly", "monthly", "trend"
}
```

The schema validation middleware would enforce these contracts at every inter-agent boundary:

```csharp
public class SchemaValidationMiddleware
{
    private static readonly HashSet<string> SupportedSchemaVersions = ["1.0", "1.1"];

    public async Task<bool> ValidateAsync(AgentMessage message, string expectedRecipientId)
    {
        // Reject unknown schema versions
        if (!SupportedSchemaVersions.Contains(message.SchemaVersion))
        {
            _logger.LogWarning(
                "Rejected message with unsupported schema version {Version} from {Sender}",
                message.SchemaVersion, message.SenderId);
            return false;
        }

        // Reject messages not addressed to this agent (explicit audience)
        if (message.RecipientId != expectedRecipientId)
        {
            _logger.LogWarning(
                "Rejected message addressed to {Recipient}, but this agent is {Expected}",
                message.RecipientId, expectedRecipientId);
            return false;
        }

        // Validate typed contract constraints
        if (message is DataRetrievalRequest dataRequest)
        {
            // Enforce ASI02-style constraints on inter-agent requests too
            if (dataRequest.PageSize > 50)
                return false;

            if ((dataRequest.EndDate.ToDateTime(TimeOnly.MinValue) -
                 dataRequest.StartDate.ToDateTime(TimeOnly.MinValue)).Days > 365)
                return false;
        }

        return true;
    }
}
```

Some key points here:

- **Typed contracts** prevent the "semantics split-brain" attack — both agents parse the message into the same strongly-typed structure instead of interpreting free-form text differently
- **Explicit audiences** (`RecipientId`) ensure a message for the Data Retrieval Agent cannot be routed to the Health Advisor Agent
- **Schema versioning** allows backward-compatible evolution without breaking existing agents
- **ASI02 constraints carry forward** — tool-level guardrails (page size caps, date range limits) are enforced on inter-agent messages too, preventing one agent from bypassing another agent's controls
- **Provenance chains** (`SourceData` in `TrendAnalysisRequest`) let the Health Advisor Agent verify the data came from the Data Retrieval Agent, not from an injected source

This is familiar pattern if you've worked with API versioning in ASP.NET. The difference is that in a multi-agent system, both the client and the server are agents, and neither should be fully trusted.

What's missing is semantic divergence detection beyond structural validation. The typed contracts ensure structural consistency, but two agents might still interpret the same data differently if their LLM reasoning diverges. For example, the Data Retrieval Agent might return a weight trend as "increasing" while the Health Advisor Agent interprets the same numbers as "stable within normal variance." Mitigations for semantic divergence focus on integrity (signature validation), provenance (tracking data lineage), and controlled communication patterns (explicit schema contracts) rather than attempting full semantic alignment which is an inherent limitation of LLM-based systems. Formal contract testing frameworks (similar to Pact for microservices) would catch schema mismatches before deployment, and embedding-based similarity checks on request/response pairs would provide an automated signal for semantic drift.

## Putting It All Together

Let's walk through the hypothetical communication flow from earlier, showing all 9 controls in action:

**User asks:** *"How has my sleep quality changed this month, and what should I do about it?"*

**What the multi-agent system does (with controls):**

1. The Orchestrator authenticates to the Data Retrieval Agent using its Entra Agent ID token scoped to `api://biotrackr-data-agent/.default`, over an mTLS connection with certificate pinning — **secure agent channels**
2. The Orchestrator sends a typed `DataRetrievalRequest` with `DataType = "sleep"`, `SchemaVersion = "1.1"`, and `RecipientId = "biotrackr-data-agent"` — **typed contracts and schema validation**
3. The request is digitally signed with the Orchestrator's private key, and the payload hash is computed over both the request body and the current conversation context hash — **message integrity and semantic protection**
4. The request includes a unique nonce and timestamp. The Data Retrieval Agent checks the nonce against Redis and verifies the timestamp is within the 5-minute task window — **agent-aware anti-replay**
5. APIM validates the `X-Agent-Protocol-Version: biotrackr-agent-protocol/1.1` header and rejects any downgrade attempts — **protocol pinning and version enforcement**
6. The Data Retrieval Agent verifies the Orchestrator's identity against the `AgentCapabilityRegistry` to confirm it supports the `RouteToDataAgent` operation — **protocol and capability security**
7. The request is padded to 8KB and sent with random jitter (50-150ms) to prevent traffic analysis — **limit metadata-based inference**
8. The Orchestrator discovered the Data Retrieval Agent through the `SecureAgentRegistry`, which verified the agent's certificate chain, registration signature, and expiry — **discovery and routing protection**
9. The `ContinuousAgentVerification` background service verified the Data Retrieval Agent's signed agent card within the last 5 minutes, confirming the container image digest matches the CI/CD-signed attestation — **attested registry and agent verification**
10. The Data Retrieval Agent returns a typed `DataRetrievalResponse` with sleep records. The Orchestrator validates the signature, checks the semantic intent (response contains sleep data, not weight data), and forwards it to the Health Advisor Agent — repeating steps 1-9 for the second hop
11. The Health Advisor Agent receives a `TrendAnalysisRequest` that includes the `SourceData` from the Data Retrieval Agent — providing a provenance chain back to the original API call
12. The Orchestrator combines both responses and streams the answer to the user via AG-UI

Every arrow in the communication flow is authenticated, signed, replay-protected, schema-validated, and protocol-pinned. If any single control fails, the others limit the blast radius.

## Wrapping up

Insecure Inter-Agent Communication (ASI07) is the vulnerability class that emerges when you move from single-agent to multi-agent architectures. The OWASP specification defines 9 prevention and mitigation guidelines: secure channels, signed messages, anti-replay, protocol enforcement, traffic normalization, protocol pinning, discovery protection, attested registries, and typed contracts.

The controls are layered: mTLS → per-agent identity tokens → signed messages → anti-replay nonces → protocol pinning → capability negotiation → authenticated discovery → CI/CD-signed agent cards → typed schemas with explicit audiences. Even if one layer is bypassed, the others limit the blast radius. **Inter-agent communication is the largest new attack surface in multi-agent systems. Every message between agents can be intercepted, spoofed, replayed, or semantically manipulated, and each layer of defence addresses a different attack vector.**

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
