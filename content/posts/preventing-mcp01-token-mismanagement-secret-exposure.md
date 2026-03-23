---
title: "Preventing Token Mismanagement and Secret Exposure in MCP Servers"
date: 2026-03-23
draft: false
tags: ["MCP", "AI", ".NET", "OWASP", "Security", "Key Vault", "Azure App Configuration", "OpenTelemetry"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3a1rfzlad2fxamsyqtdh.png
    alt: "Preventing OWASP MCP01 Token Mismanagement and Secret Exposure in a .NET MCP server with Key Vault, telemetry redaction, and runtime injection."
    caption: "Implementing OWASP MCP01 mitigations against Token Mismanagement and Secret Exposure in a .NET 10 MCP server built with the Model Context Protocol SDK."
---

When building an MCP server for my side project (Biotrackr), one of the first things I had to think about was how to manage secrets. The MCP server needs an APIM subscription key to call downstream health data APIs, and it also exposes an API key for clients connecting to it. That's two secrets that need to be stored, distributed, and protected and if either one leaks, the blast radius could extend across the entire platform.

Tokens and credentials are the primary authentication mechanism between models, tools, and servers in MCP systems. If they're mishandled, embedded in config files, persisted in model context memory, or logged without redaction, the MCP server becomes an unintentional secret repository. Long-lived sessions and stateful agents make this worse, because tokens can be inadvertently stored, indexed, or retrieved through prompts, system recalls, or log inspection.

In this article, we'll cover **Token Mismanagement and Secret Exposure (MCP01:2025)** and how Biotrackr implements prevention and mitigation strategies to keep secrets out of places they shouldn't be.

## What is Token Mismanagement and Secret Exposure?

Token Mismanagement and Secret Exposure occurs when developers mishandle secrets by embedding them in configuration files, environment variables, prompt templates, or allowing them to persist within model context memory. MCP enables long-lived sessions, stateful agents, and context persistence, which means tokens can be inadvertently stored, indexed, or retrieved through user prompts, system recalls, or log inspection.

This creates **contextual secret leakage** where the model or protocol layer becomes an unintentional secret repository.

The impact can be severe:

- Complete environment compromise through API or infrastructure access
- Unauthorized code modifications or repository tampering
- Lateral movement across integrated services (CI/CD, cloud storage, issue trackers)
- Data exfiltration from vector databases or file stores
- High-impact permissions granted without direct human intervention

## How does this affect my MCP server?

In my MCP server, there are two secrets that need careful management: an APIM subscription key for calling downstream health data APIs, and an API key that clients must present to access the MCP server itself. If the APIM subscription key leaked through logs, traces, or model context, an attacker could directly call the health data APIs bypassing the MCP server entirely. If the MCP server's own API key is compromised, any client could impersonate an authorized agent.

The secrets flow through multiple infrastructure layers: Key Vault for storage, App Configuration for distribution, managed identity for authentication, and OpenTelemetry for observability. Each layer is a potential exposure point that needs to be secured.

The [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) specification defines 5 prevention and mitigation control categories for MCP01, so let's walk through each control and see how Biotrackr implements them.

## Secure Vault Storage

*"Store secrets in secure vaults."*

The MCP server's API key is stored as a secret in Azure Key Vault with RBAC-based access control. The Key Vault module configures RBAC authorization (no access policies), soft delete for recovery, and diagnostic logging to Log Analytics for audit trails:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'standard'
      family: 'A'
    }
    tenantId: tenant().tenantId
    enableRbacAuthorization: true
    enabledForTemplateDeployment: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
  }
}
```

With `enableRbacAuthorization: true`, access to secrets is governed entirely by Azure RBAC role assignments. This means you can use Azure's identity platform to control exactly which principals can read, write, or manage secrets, and every access attempt is auditable through the diagnostic settings piped to Log Analytics.

The MCP server's API key is then stored as a Key Vault secret during infrastructure deployment:

```bicep
resource mcpServerApiKeySecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  name: 'mcpserverapikey'
  parent: keyVault
  properties: {
    value: mcpApiKeyValue
  }
}
```

The secret never appears in source code, application configuration files, or environment variables. It's generated deterministically during deployment using `uniqueString()` functions and only exists in Key Vault at rest.

## Runtime Secret Injection

*"Use environment variable injection only at runtime, never at build time."*

The MCP server resolves secrets through a managed identity → App Configuration → Key Vault chain that only executes at application startup. The Container App is deployed with just two non-sensitive environment variables: the App Configuration endpoint and the managed identity client ID:

```bicep
envVariables: [
  {
    name: 'azureappconfigendpoint'
    value: appConfig.properties.endpoint
  }
  {
    name: 'managedidentityclientid'
    value: uai.properties.clientId
  }
]
```

At startup, the application uses these to authenticate to App Configuration with managed identity, which in turn resolves Key Vault references:

```csharp
var credential = new ManagedIdentityCredential(managedIdentityClientId);
builder.Configuration.AddAzureAppConfiguration(config =>
{
    config.Connect(new Uri(azureAppConfigEndpoint), credential)
    .Select(keyFilter: KeyFilter.Any, LabelFilter.Null)
    .ConfigureKeyVault(kv =>
    {
        kv.SetCredential(credential);
    });
});
```

The `ManagedIdentityCredential` authenticates to App Configuration using the user-assigned identity, and `ConfigureKeyVault` tells the SDK to automatically resolve any Key Vault references it encounters using the same credential. No secrets are baked into the container image, written to `appsettings.json`, or available during the build phase: they're only materialized in memory at runtime.

## Short-Lived, Scoped Tokens

*"Issue short-lived, scoped tokens aligned with least privilege principles."*

Short-lived tokens reduce the window of exposure if a credential is compromised. Azure Key Vault supports automated rotation policies that can regenerate secrets on a schedule and notify downstream systems via Event Grid.

Here's how an automated rotation policy can be configured for a Key Vault secret:

```bicep
resource mcpServerApiKeySecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  name: 'mcpserverapikey'
  parent: keyVault
  properties: {
    value: mcpApiKeyValue
    attributes: {
      exp: dateTimeToEpoch(dateTimeAdd(utcNow(), 'P90D'))
    }
  }
}
```

Setting an `exp` (expiration) attribute on the secret means it's automatically flagged as expired after 90 days. Combined with a Key Vault rotation policy and an Event Grid subscription, the rotation workflow becomes:

1. Key Vault raises a `SecretNearExpiry` event
2. Event Grid triggers an Azure Function or Logic App
3. The function generates a new key, updates the Key Vault secret, and rotates the APIM subscription
4. The MCP server picks up the new value on its next restart or configuration refresh

This keeps token lifetimes bounded rather than indefinite.

## Session Token Renewal

*"Require token renewal for every new MCP session."*

In session-based MCP transports, tokens can persist across sessions meaning a compromised token from one session could be replayed in another. Biotrackr's MCP server uses stateless transport, which inherently avoids this problem:

```csharp
builder.Services
    .AddMcpServer()
    .WithHttpTransport(o => o.Stateless = true)
    .WithToolsFromAssembly();
```

With `Stateless = true`, each HTTP request is fully independent. There are no persistent sessions, no session IDs, and no stored tokens between requests. Every request must present valid credentials, and nothing carries over from previous interactions.

For MCP servers that need session-based transport, token renewal can be enforced by binding short-lived tokens to the `Mcp-Session-Id` header and requiring re-authentication when sessions expire. The stateless approach eliminates this concern entirely.

## Per-Agent Token Binding

*"Bind tokens to the specific agent, tool, or session context."*

Per-agent token binding prevents a token issued to one client from being used by another. This is particularly important in multi-tenant MCP environments where multiple agents share the same server.

Azure API Management can enforce this through JWT validation policies that bind tokens to specific client identities. Here's how the Biotrackr MCP server's APIM policy supports JWT-based authentication with audience and issuer validation:

```xml
<policies>
  <inbound>
    <base />
    <choose>
      <when condition="@(context.Request.Headers.GetValueOrDefault(
        &quot;Authorization&quot;,&quot;&quot;).StartsWith(&quot;Bearer &quot;))">
        <validate-jwt header-name="Authorization"
          failed-validation-httpcode="401"
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
        <check-header name="Ocp-Apim-Subscription-Key"
          failed-check-httpcode="401"
          failed-check-error-message="Unauthorized: Missing or invalid subscription key" />
      </otherwise>
    </choose>
  </inbound>
</policies>
```

This policy validates JWT tokens against the configured OpenID Connect discovery endpoint, checking both the `audience` (which service the token was issued for) and the `issuer` (which identity provider issued it). When combined with Entra ID app registrations, each agent or client application gets its own client ID and must authenticate individually. A token issued to Agent A cannot be used by Agent B because their audiences differ.

The `choose` block also allows a graceful fallback to subscription key authentication for clients that don't support JWT, maintaining backward compatibility while enabling stronger per-agent binding for clients that do.

## Context Memory Isolation

*"Prevent sensitive data persistence in model memory or context windows."*

In stateful MCP servers, conversation context and tool results can persist across turns meaning API keys, tokens, or sensitive tool responses could be stored in the model's context window and retrieved through crafted follow-up prompts.

Biotrackr eliminates this entirely with stateless transport:

```csharp
builder.Services
    .AddMcpServer()
    .WithHttpTransport(o => o.Stateless = true)
    .WithToolsFromAssembly();
```

With `Stateless = true`, no conversation history, tool results, or session context is maintained between requests. Each request is a clean slate. There's no memory of previous interactions, which means there's no stored context for an attacker to extract secrets from.

This also means that even if a tool response inadvertently includes sensitive data in one request, that data is discarded entirely before the next request arrives.

## Telemetry Redaction

*"Redact or mask secrets before writing to logs or telemetry. Redact or sanitize inputs and outputs before logging."*

Telemetry is essential for debugging and monitoring, but without proper redaction, it can become an unintentional secret leaker. OpenTelemetry's HTTP client instrumentation captures request headers by default, which would include the `Ocp-Apim-Subscription-Key` header used to authenticate with downstream APIs.

Biotrackr explicitly redacts this header in its OpenTelemetry configuration:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b.SetResourceBuilder(resourceBuilder)
        .AddSource("Biotrackr.Mcp.Server")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation(o =>
        {
            o.FilterHttpRequestMessage = _ => true;
            o.EnrichWithHttpRequestMessage = (activity, request) =>
            {
                // Ensure subscription key header is never captured in traces
                activity.SetTag("http.request.header.ocp_apim_subscription_key", "[REDACTED]");
            };
        })
        .AddAzureMonitorTraceExporter(options =>
        {
            options.ConnectionString = appInsightsConnectionString;
        }));
```

The `EnrichWithHttpRequestMessage` callback fires for every outbound HTTP request. By explicitly setting the `ocp_apim_subscription_key` tag to `[REDACTED]`, the actual subscription key value is overwritten before the trace is exported to Application Insights. This means that even someone with full read access to Application Insights traces cannot extract the subscription key from telemetry data.

The API key used *for* the MCP server (presented by clients in the `X-Api-Key` header) is similarly protected. The `ApiKeyAuthMiddleware` validates it using `CryptographicOperations.FixedTimeEquals` (preventing timing attacks) and only logs the request path on failure, never the key value:

```csharp
if (!context.Request.Headers.TryGetValue(ApiKeyHeaderName, out var providedKey) ||
    !CryptographicOperations.FixedTimeEquals(
        Encoding.UTF8.GetBytes(providedKey.ToString()),
        Encoding.UTF8.GetBytes(_expectedApiKey)))
{
    _logger.LogWarning("Rejected request to {Path} — missing or invalid API key",
        context.Request.Path);
    context.Response.StatusCode = StatusCodes.Status401Unauthorized;
    await context.Response.WriteAsJsonAsync(new { error = "Unauthorized" });
    return;
}
```

The log message includes `{Path}` but deliberately excludes the provided key value. A failed authentication attempt reveals the target path but never the credential itself.

## Ephemeral Credential Contexts

*"Use ephemeral contexts for operations involving credentials."*

Ephemeral contexts ensure that credentials exist in memory only for the duration of a single operation and are not persisted to disk, cached, or shared across requests.

Biotrackr achieves this through two mechanisms. First, the stateless transport ensures no request context persists:

```csharp
builder.Services
    .AddMcpServer()
    .WithHttpTransport(o => o.Stateless = true)
    .WithToolsFromAssembly();
```

Second, the APIM subscription key is injected per-request through a delegating handler rather than being stored in a shared mutable state:

```csharp
public class ApiKeyDelegatingHandler : DelegatingHandler
{
    private const string SubscriptionKeyHeader = "Ocp-Apim-Subscription-Key";
    private readonly string? _subscriptionKey;

    public ApiKeyDelegatingHandler(IOptions<BiotrackrApiSettings> settings)
    {
        _subscriptionKey = settings.Value.SubscriptionKey;
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

The `ApiKeyDelegatingHandler` is registered as a transient service, meaning each HTTP request gets a fresh handler instance. The subscription key is read from the options pattern (which itself was resolved from Key Vault at startup) and attached to the outbound request header only for that specific HTTP call. The `HttpRequestMessage` is ephemeral, once the response is received and the handler completes, the request (and its headers) are eligible for garbage collection.

## Token Rotation and Invalidation

*"Rotate and invalidate all tokens immediately upon suspected exposure."*

Fast rotation and invalidation limits the damage window when a secret is compromised. Azure Key Vault supports rotation policies that work with Event Grid to automate the entire lifecycle.

Here's how an automated rotation flow can be configured:

```bicep
resource mcpServerApiKeySecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  name: 'mcpserverapikey'
  parent: keyVault
  properties: {
    value: mcpApiKeyValue
    attributes: {
      exp: dateTimeToEpoch(dateTimeAdd(utcNow(), 'P90D'))
    }
  }
}

resource rotationPolicy 'Microsoft.KeyVault/vaults/secrets/rotationPolicy@2023-07-01' = {
  parent: mcpServerApiKeySecret
  properties: {
    lifetimeActions: [
      {
        trigger: {
          timeBeforeExpiry: 'P30D'
        }
        action: {
          type: 'Notify'
        }
      }
    ]
    attributes: {
      expiryTime: 'P90D'
    }
  }
}
```

This configuration sets a 90-day expiry on the secret and triggers a notification 30 days before expiration. The notification is raised as a Key Vault event, which can be consumed by Event Grid to trigger an Azure Function that generates a new key and updates both Key Vault and the downstream APIM subscription.

For immediate invalidation during an incident, APIM subscription keys can be regenerated through the Azure CLI:

```bash
az apim subscription regenerate-key \
  --resource-group <rg> \
  --service-name <apim> \
  --subscription-id <sub-id> \
  --key-type primary
```

This instantly invalidates the old key and issues a new one, cutting off any attacker who may have obtained the previous credential.

## Secrets Managers for Runtime Injection

*"Use HSMs or Secrets Managers for runtime injection."*

Azure Key Vault serves as the centralized secrets manager, and secrets are distributed to the MCP server through Key Vault references in Azure App Configuration, not through environment variables or config files.

The API key is stored in App Configuration as a Key Vault reference rather than a plaintext value:

```bicep
resource mcpServerApiKeySetting 'Microsoft.AppConfiguration/configurationStores/keyValues@2025-02-01-preview' = {
  name: 'mcpserverapikey'
  parent: appConfig
  properties: {
    value: '{"uri":"${keyVault.properties.vaultUri}secrets/mcpserverapikey"}'
    contentType: 'application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8'
  }
}
```

The `contentType` of `application/vnd.microsoft.appconfig.keyvaultref+json` tells the App Configuration SDK that this isn't a plaintext value. It's a pointer to a Key Vault secret. When the application reads this setting, the SDK automatically detects the content type, extracts the Key Vault URI, and resolves the actual secret value using the credential configured in `ConfigureKeyVault`.

The managed identity that performs this resolution is granted the `Key Vault Secrets Officer` role scoped specifically to the Key Vault instance:

```bicep
var keyVaultSecretsOfficerRoleDefinitionId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'b86a8fe4-44ce-4948-aee5-eccb2c155cd7'
)

resource keyVaultSecretsOfficerRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(subscription().id, resourceGroup().id, keyVault.id)
  properties: {
    principalId: uai.properties.principalId
    roleDefinitionId: keyVaultSecretsOfficerRoleDefinitionId
    principalType: 'ServicePrincipal'
  }
}
```

This role assignment means only the MCP server's managed identity can read secrets from this Key Vault. No other service, developer, or pipeline has access unless explicitly granted. The identity itself is a user-assigned managed identity, meaning it has a stable identity across deployments and its permissions are defined in infrastructure code, not configured manually.

## Wrapping up

Token Mismanagement and Secret Exposure (MCP01:2025) is about ensuring secrets never end up where they shouldn't be, in source code, configuration files, logs, traces, or model context.

The controls are layered: Key Vault for storage → Key Vault references for distribution → managed identity for authentication → telemetry redaction for observability → stateless transport for context isolation. Even if one layer is compromised, the others limit the blast radius.

The key takeaway here is to **treat your MCP server's secrets with the same rigor as any production API**. The fact that it's an AI component doesn't exempt it from fundamental credential hygiene.

In the next post in this series, I'll cover **MCP02:2025 — Privilege Escalation via Scope Creep**, exploring how Biotrackr ensures the MCP server's permissions don't grow beyond what's necessary.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
