---
title: "Creating Entra Agent ID Blueprints and Identities with PowerShell and .NET"
date: 2026-02-28
draft: false
tags: ["Agents", "Security", "AI", "Identity", "Entra ID", "CSharp", "Azure Container Apps"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y92he48d4sqvcxrravvu.png
    alt: "Learn how to create agent identity blueprints using Microsoft Graph PowerShell and provision agent identities through an ASP.NET Web API running on Azure Container Apps, with managed identity authentication and federated credentials."
    caption: "Learn how to create agent identity blueprints using Microsoft Graph PowerShell and provision agent identities through an ASP.NET Web API running on Azure Container Apps, with managed identity authentication and federated credentials."
---

In Microsoft Entra Agent ID, we use agent identity blueprints to create agent identities and request tokens using those agent identities. These agent identities represent AI Agents within your tenant, and is usually provisioned when we create a new AI Agent.

In this post, we'll go through an end-to-end example of creating an agent blueprint using Microsoft Graph PowerShell, then we'll create an Agent Identity using a ASP.NET Web API that we'll deploy to Azure Container Apps.

For the complete E2E sample, please check it out on my [GitHub](https://github.com/willvelida/agent-identity-samples/tree/main/entra-agent-id/create-agent-blueprint-and-identities).

*Note. At the time of writing, Microsoft Entra Agent ID is currently in preview. Please check the [docs](https://learn.microsoft.com/en-us/entra/agent-id/identity-platform/what-is-agent-id-platform) for the latest information.*

## Creating an agent identity blueprint using Microsoft Graph PowerShell

Agent identity blueprints provide us with the template and management structure for creating and managing multiple agent identities. This serves as the parent of an agent identity. All agent identities in a Microsoft Entra ID are created from an agent identity blueprint.

Organizations can deploy many instances of an AI agent which pursues different goals and require different levels of access. However, these many instances will share certain characteristics, and blueprints record these common characteristics so that all agent identities created using the blueprint have a consistent configuration.

Before we get started, you'll need the following prerequisites:

- [Privileged Role Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#privileged-role-administrator) role is required to grant Microsoft Graph Application permissions, and [Cloud Application Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#cloud-application-administrator) or [Application Adminisitrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#application-administrator) is required to grant Microsoft Graph delegated permissions.
- [Agent ID Developer](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#agent-id-developer) or [Agent ID Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#agent-id-administrator) roles are required to create agent identity blueprints.
- Since we're using PowerShell, you'll need version 7.
- While this is all in Preview, you'll need the beta version for both Microsoft Graph and PowerShell.

You can assign the required roles to your account in Microsoft Entra ID in the Azure Portal.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pqa1w0gn7r344lqra6tl.png)

## Preparing your environment: authorizing a client to create agent identity blueprints

Before you can create an agent identity blueprint, you need to authorize a client with the right Microsoft Graph permissions. The full blueprint lifecycle requires four delegated permissions:

| Permission | What it allows |
|---|---|
| `AgentIdentityBlueprint.Create` | Create the blueprint application object |
| `AgentIdentityBlueprint.AddRemoveCreds.All` | Add credentials (federated identity credentials, secrets, or certificates) to the blueprint |
| `AgentIdentityBlueprint.ReadWrite.All` | Update the blueprint (e.g. configure identifier URIs and OAuth scopes) |
| `AgentIdentityBlueprintPrincipal.Create` | Create the service principal for the blueprint |

We also need `User.Read` so the script can resolve the current signed-in user and assign them as the blueprint's owner and sponsor.

### Connecting to Microsoft Graph

In the sample's `setup.ps1`, we connect once upfront with all the required scopes:

```powershell
$allScopes = @(
    "AgentIdentityBlueprint.Create",
    "AgentIdentityBlueprint.AddRemoveCreds.All",
    "AgentIdentityBlueprint.ReadWrite.All",
    "AgentIdentityBlueprintPrincipal.Create",
    "User.Read"
)
Connect-MgGraph -Scopes $allScopes -TenantId $TenantId -UseDeviceCode -NoWelcome
```

There are a few deliberate choices here:

**Single connection with all scopes.** You could connect separately for each step with only the scopes that step needs that would follow the principle of least privilege per operation. But because `setup.ps1` runs all steps end-to-end in a single execution (triggered automatically as part of `azd up`), splitting the connection would mean multiple consent prompts interrupting the deployment. By requesting all scopes in a single call, the user consents once and the script runs to completion.

**Device code flow (`-UseDeviceCode`).** The default authentication method for `Connect-MgGraph` is interactive browser-based auth. We use device code flow instead because it works in environments where a browser isn't available on the same machine (containers, SSH sessions, CI runners) and behaves consistently across Windows, macOS, and Linux without worrying about browser integration.

**Suppressing the welcome banner (`-NoWelcome`).** A small detail, but when `setup.ps1` is called by a pre-provision hook, the Microsoft Graph welcome banner would clutter the `azd up` output. `-NoWelcome` keeps the output clean and focused on the script's own status messages.

### Installing the beta modules

Since Microsoft Entra Agent ID is currently in preview, blueprint operations require the beta version of the Microsoft Graph PowerShell modules. Before connecting, the sample ensures they're installed:

```powershell
if (-not (Get-Module -ListAvailable -Name Microsoft.Graph.Beta.Applications)) {
    Install-Module Microsoft.Graph.Beta.Applications -Scope CurrentUser -Force
}
if (-not (Get-Module -ListAvailable -Name Microsoft.Graph.Applications)) {
    Install-Module Microsoft.Graph.Applications -Scope CurrentUser -Force
}
```

We install both the beta and stable modules because certain cmdlets (like `Update-MgBetaApplication` for scope configuration) require the beta module, while others (like resolving the current user) use the stable API. The `Get-Module -ListAvailable` check makes this idempotent. On subsequent runs, the install is skipped entirely.

## Creating the agent identity blueprint

Every agent identity blueprint must have a **sponsor** (the human user or group accountable for the agent) and should have an **owner** who can make changes to the blueprint. These administrative relationships are a core part of the Entra Agent ID model; they ensure every agent is traceable back to a responsible human.

Creating a blueprint is a two-part process: resolve the current user to serve as sponsor and owner, then POST to the Microsoft Graph beta endpoint to create the blueprint application object.

### Resolving the current user

```powershell
$user = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/me" -OutputType PSObject

Write-Host "Current user: $($user.DisplayName) ($($user.Id))"
Write-Host "Sponsor user: $($user.DisplayName) ($($user.Id))"
```

You might expect to use `Get-MgContext` to get the signed-in user, but its `Account` property can be empty when authenticating via Windows Web Account Manager (WAM). Calling the `/me` endpoint directly via `Invoke-MgGraphRequest` is more reliable — it always returns the authenticated user's profile regardless of which authentication method was used.

The user's object ID (`$user.Id`) is what we need for the `sponsors@odata.bind` and `owners@odata.bind` properties. In this sample, we assign the same user as both sponsor and owner, but in a production scenario you might assign a group or a different user for each role.

### Building and sending the request

```powershell
$body = @{
    "@odata.type" = "Microsoft.Graph.AgentIdentityBlueprint"
    "displayName" = $AgentBlueprintPrincipalName
    "sponsors@odata.bind" = @("https://graph.microsoft.com/v1.0/users/$($user.Id)")
    "owners@odata.bind" = @("https://graph.microsoft.com/v1.0/users/$($user.Id)")
} | ConvertTo-Json -Depth 5

$response = Invoke-MgGraphRequest `
    -Method POST `
    -Uri "https://graph.microsoft.com/beta/applications/graph.agentIdentityBlueprint" `
    -Body $body `
    -ContentType "application/json"
```

A few things to note about this request:

**The `@odata.type` discriminator.** The endpoint `/beta/applications/graph.agentIdentityBlueprint` is an OData type cast. It tells Microsoft Graph you're creating an `AgentIdentityBlueprint`, not a regular application registration. The `@odata.type` property in the body reinforces this. Without it, Graph would treat this as a standard app registration and the blueprint-specific properties wouldn't be processed.

**Binding sponsors and owners by reference.** The `sponsors@odata.bind` and `owners@odata.bind` properties use OData reference binding syntax. Rather than embedding user objects, you provide URIs that point to existing user (or group) resources. This is the same pattern used elsewhere in Microsoft Graph when establishing relationships between entities.

**Using `Invoke-MgGraphRequest` instead of a typed cmdlet.** At the time of writing, the Microsoft Graph PowerShell SDK doesn't have a dedicated `New-MgBetaAgentIdentityBlueprint` cmdlet. The `Invoke-MgGraphRequest` wrapper gives us direct access to the beta REST API while still benefiting from the SDK's authentication and token management.

**The `-Depth 5` on `ConvertTo-Json`.** PowerShell's default serialization depth is 2, which would truncate the nested arrays in `sponsors@odata.bind` and `owners@odata.bind`. Specifying `-Depth 5` ensures the full request body is serialized correctly.

### What comes back

The response includes the newly created blueprint's `appId`, which we capture for every subsequent step:

```powershell
$applicationId = $response.appId
if (-not $applicationId) {
    Write-Error "Failed to create agent identity blueprint — no appId returned from Graph API."
    exit 1
}
Write-Host "Blueprint appId: $applicationId"
```

This `appId` is used to configure the identifier URI and scope, create the service principal, add federated credentials, and configure the .NET API, so the script fails fast if it's missing. At the end of `setup.ps1`, we persist it to the `azd` environment with `azd env set AGENT_BLUEPRINT_APP_ID $applicationId`, making it available to the Bicep templates during provisioning and the post-provision hook that configures the federated identity credential.

## Configuring the identifier URI and OAuth2 scope

If your agent is designed to receive incoming requests from users or other agents, it needs two things: an **identifier URI** that uniquely identifies the blueprint as a resource, and an **OAuth2 permission scope** that callers can request in their access tokens. Without these, there's no way for a client application to request a token that's scoped to your agent.

### Setting the identifier URI

```powershell
$IdentifierUri = "api://$applicationId"
```

The identifier URI follows the standard `api://{appId}` convention used across Microsoft identity platform. This becomes the audience claim (`aud`) in tokens issued for this blueprint. When a client later requests a token with the scope `api://{appId}/access_agent`, Entra ID uses this URI to resolve which application the token is for.

### Defining the scope

```powershell
$ScopeId = [guid]::NewGuid()

$scope = @{
    adminConsentDescription = "Allow the application to access the agent on behalf of the signed-in user."
    adminConsentDisplayName = "Access agent"
    id                      = $ScopeId
    isEnabled               = $true
    type                    = "User"
    value                   = "access_agent"
}
```

The scope definition is an `oauth2PermissionScopes` entry with a few important properties:

- **`id`** — A unique GUID for this scope. We generate it at script runtime with `[guid]::NewGuid()`. This ID is referenced later when the test client app registration declares which permissions it needs.
- **`type: "User"`** — This is a delegated permission scope, meaning it requires a signed-in user. The agent identity API acts on behalf of the user who calls it (for example, using their object ID as the agent identity's sponsor).
- **`value: "access_agent"`** — The string that clients include in their token requests. A client would request the scope `api://{appId}/access_agent` to get a delegated token for this blueprint.
- **`isEnabled: true`** — The scope is active and can be consented to. You'd set this to `false` if you needed to deprecate a scope without immediately breaking existing consent grants.

### Applying the configuration

```powershell
Update-MgBetaApplication -ApplicationId $applicationId `
    -IdentifierUris @($IdentifierUri) `
    -Api @{ oauth2PermissionScopes = @($scope) }

Write-Host "Configured identifier URI: $IdentifierUri"
Write-Host "Created scope 'access_agent' with ID: $ScopeId"
```

This is one place where the PowerShell SDK's typed cmdlets are more convenient than raw REST calls. The equivalent Graph API call would be a `PATCH` to `/beta/applications/{id}` with a JSON body containing `identifierUris` and a nested `api.oauth2PermissionScopes` array, plus the `OData-Version: 4.0` header. `Update-MgBetaApplication` handles the serialization and headers for us, and sets both the identifier URI and the scope in a single call.

This requires the `AgentIdentityBlueprint.ReadWrite.All` permission we consented to earlier during the `Connect-MgGraph` call.

## Creating the agent blueprint principal

At this point we have a blueprint application object with an identifier URI and scope, but it can't actually be used in the tenant yet. In the Microsoft identity platform, an application object is a global definition that describes *what* an application is. To make it usable within a specific tenant, you need a **service principal**, which is the local instance of that application in the tenant.

For agent identity blueprints, the service principal is a special type called an **agent identity blueprint principal**. It's created through a dedicated endpoint rather than the standard service principal creation path, because it carries agent-specific behaviour — it's the entity that enables agent identities to be created from the blueprint within your tenant.

### Creating the principal

```powershell
$spBody = @{
    appId = $applicationId
}

$spResponse = Invoke-MgGraphRequest `
    -Method POST `
    -Uri "https://graph.microsoft.com/beta/serviceprincipals/graph.agentIdentityBlueprintPrincipal" `
    -Headers @{ "OData-Version" = "4.0" } `
    -Body ($spBody | ConvertTo-Json)

Write-Host "Agent blueprint principal created for appId: $applicationId"
$spResponse
```

The request body is minimal, just the `appId` from the blueprint we created earlier. The endpoint `/beta/serviceprincipals/graph.agentIdentityBlueprintPrincipal` is another OData type cast, similar to how we used `/beta/applications/graph.agentIdentityBlueprint` when creating the blueprint itself. It tells Microsoft Graph to create an `AgentIdentityBlueprintPrincipal` rather than a standard service principal.

**The `OData-Version: 4.0` header** is required here. This endpoint uses OData v4 conventions for type casting, and without this header the request will fail. We pass it explicitly via the `-Headers` parameter on `Invoke-MgGraphRequest`.

This step requires the `AgentIdentityBlueprintPrincipal.Create` permission.

## Creating agent identities with Microsoft.Identity.Web

Now that our agent identity blueprint has been created, we can now create agent identities that can be used to represent AI Agents in our tenant. We'd usually do this when creating a new Agent.

For this scenario, we'll use our agent identity blueprint that we've just created, as well as ASP.NET minimal web API running in Azure Container Apps that will host the agent identity creation logic.

### Obtaining an access token using agent identity blueprint

This is straightforward using `Microsoft.Identity.Web`. We can request an access token from Microsoft Entra by installing this package to our web API:

```bash
dotnet add package Microsoft.Identity.Web
```

This package includes an interface that automatically requests an access token and attaches it to outbound HTTP requests.

he configuration in `appsettings.json` tells Microsoft.Identity.Web how to authenticate incoming requests *and* how to acquire tokens for outbound Graph API calls:

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<tenant-id>",
    "ClientId": "<agent-blueprint-app-id>",
    "Scopes": "access_agent",
    "ClientCredentials": [
      {
        "SourceType": "SignedAssertionFromManagedIdentity",
        "ManagedIdentityClientId": "<managed-identity-client-id>"
      }
    ]
  },
  "DownstreamApis": {
    "agent-identity": {
      "BaseUrl": "https://graph.microsoft.com",
      "RelativePath": "/beta/serviceprincipals/Microsoft.Graph.AgentIdentity",
      "Scopes": ["00000003-0000-0000-c000-000000000000/.default"],
      "RequestAppToken": true
    }
  }
}
```

There are two important differences from the example in the getting started guide.

**Managed identity instead of a client secret.** The getting started guide uses `"SourceType": "ClientSecret"` with a secret string stored in configuration. That's fine for local development, but the guide itself warns against using client secrets in production. Our sample uses `"SourceType": "SignedAssertionFromManagedIdentity"` instead. This tells `Microsoft.Identity.Web` to use the Container App's managed identity token as a signed assertion to acquire an app-only token for the blueprint, the federated identity credential pattern we configured in `postprovision.ps1`. No secrets are stored anywhere.

**The `DownstreamApis` section.** This is a `Microsoft.Identity.Web` feature that preconfigures named API endpoints. The `"agent-identity"` entry defines the base URL, the relative path to the agent identity creation endpoint (`/beta/serviceprincipals/Microsoft.Graph.AgentIdentity`), the scope to request (Microsoft Graph's `.default` scope for app-only tokens), and `"RequestAppToken": true` to use client credentials flow rather than on-behalf-of.

### Wiring up the services

In `Program.cs`, the service registration is concise:

```csharp
builder.Services.AddMicrosoftIdentityWebApiAuthentication(builder.Configuration)
    .EnableTokenAcquisitionToCallDownstreamApi();
builder.Services.AddDownstreamApis(builder.Configuration.GetSection("DownstreamApis"));
builder.Services.AddInMemoryTokenCaches();
```

`AddMicrosoftIdentityWebApiAuthentication` sets up JWT bearer authentication using the blueprint's app registration. Incoming requests must carry a valid `access_agent` token. `EnableTokenAcquisitionToCallDownstreamApi` enables the outbound token acquisition pipeline. `AddDownstreamApis` registers the named API configuration from `appsettings.json`, and `AddInMemoryTokenCaches` caches the acquired app-only tokens so the API doesn't request a new token on every Graph call.

### Creating the agent identity

The `/create-agent-identity` endpoint is where the actual agent identity creation happens:

```csharp
app.MapPost("/create-agent-identity", async (HttpContext httpContext,
    [FromBody] CreateAgentIdentityRequest? request) =>
{
    var config = httpContext.RequestServices.GetRequiredService<IConfiguration>();
    var blueprintId = config["AgentIdentity:BlueprintId"]
        ?? throw new InvalidOperationException("AgentIdentity:BlueprintId is not configured.");

    var sponsorUserId = httpContext.User
        .FindFirstValue("http://schemas.microsoft.com/identity/claims/objectidentifier")
        ?? httpContext.User.FindFirstValue(ClaimTypes.NameIdentifier)
        ?? throw new InvalidOperationException(
            "Could not determine the caller's object ID from the token.");

    var displayName = request?.DisplayName
        ?? httpContext.User.FindFirstValue("name")
        ?? httpContext.User.Identity?.Name
        ?? "My agent identity";

    IDownstreamApi downstreamApi = httpContext.RequestServices
        .GetRequiredService<IDownstreamApi>();

    var jsonResult = await downstreamApi.PostForAppAsync<AgentIdentity, AgentIdentity>(
        "agent-identity",
        new AgentIdentity {
            DisplayName = displayName,
            AgentIdentityBlueprintId = blueprintId,
            SponsorsOdataBind = new[] {
                $"https://graph.microsoft.com/v1.0/users/{sponsorUserId}"
            }
        });

    return Results.Ok(new { agentIdentityId = jsonResult?.Id });
}).RequireAuthorization();
```

An endpoint that uses `IDownstreamApi.PostForAppAsync` to POST an `AgentIdentity` object to Microsoft Graph. But there are several differences in how the sample handles it:

**The sponsor is derived from the caller's token.** The getting started guide hard-codes the sponsor as a user ID in the request body. In the sample, we extract the object ID from the `oid` claim in the caller's delegated `access_agent` token. This means the person who calls the API automatically becomes the sponsor of the agent identity they create, enforcing the accountability model that Entra Agent ID is built around.

**The display name falls back gracefully.** The caller can provide a display name in the request body, but if they don't, the API uses the `name` claim from their token. If that's also missing, it defaults to `"My agent identity"`. The getting started guide uses a static string.

**The blueprint ID comes from configuration, not code.** Instead of hard-coding the blueprint app ID, it's read from `IConfiguration` via `AgentIdentity:BlueprintId`. This value is injected as an environment variable by the Bicep templates during provisioning, making the API configuration-driven.

**The endpoint is a `POST`, not a `GET`.** The getting started guide maps the creation endpoint as a `GET`, which is convenient for browser testing but doesn't follow REST conventions for resource creation. The sample uses `MapPost`.

### The AgentIdentity model

The `AgentIdentity` class serializes to the JSON body that Microsoft Graph expects:

```csharp
public class AgentIdentity
{
    [JsonPropertyName("@odata.type")]
    public string ODataType { get; set; } = "#Microsoft.Graph.AgentIdentity";

    [JsonPropertyName("displayName")]
    public string? DisplayName { get; set; }

    [JsonPropertyName("agentIdentityBlueprintId")]
    public string? AgentIdentityBlueprintId { get; set; }

    [JsonPropertyName("id")]
    public string? Id { get; set; }

    [JsonPropertyName("sponsors@odata.bind")]
    public string[]? SponsorsOdataBind { get; set; }

    [JsonPropertyName("owners@odata.bind")]
    public string[]? OwnersOdataBind { get; set; }
}
```

The `@odata.type` property is set to `#Microsoft.Graph.AgentIdentity`. This is the discriminator that tells Graph this is an agent identity, not a regular service principal. The `agentIdentityBlueprintId` links the identity back to the blueprint we created earlier. The `sponsors@odata.bind` property uses the same OData reference binding syntax we saw in `setup.ps1` when creating the blueprint itself.

Properties use `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` so that null fields like `Id` and `OwnersOdataBind` are omitted from the outbound request body but can be populated when deserializing the Graph response.

## Deleting agent identities

When an agent is deallocated or destroyed, the associated agent identity should be cleaned up. Leaving orphaned agent identities in the tenant is a security concern as they still have credentials and permissions, even if nothing is using them.

The sample exposes a `DELETE /agent-identity/{id}` endpoint that removes an agent identity by its service principal ID:

```csharp
app.MapDelete("/agent-identity/{id}", async (HttpContext httpContext, string id) =>
{
    try
    {
        IDownstreamApi downstreamApi = httpContext.RequestServices
            .GetRequiredService<IDownstreamApi>();

        await downstreamApi.DeleteForAppAsync<string, string>(
            "agent-identity",
            null!,
            options =>
            {
                options.RelativePath = $"/beta/serviceprincipals/{id}";
            });

        return Results.Ok(new { deleted = id });
    }
    catch (Exception ex)
    {
        return Results.Problem(ex.Message);
    }
}).RequireAuthorization();
```

The pattern mirrors the creation endpoint, which uses `IDownstreamApi` with the same `"agent-identity"` named configuration to handle token acquisition. However, there are a few differences worth noting.

**The relative path is overridden.** The `DownstreamApis` configuration for `"agent-identity"` points to `/beta/serviceprincipals/Microsoft.Graph.AgentIdentity`, which is the OData type cast endpoint used for creating agent identities. For deletion, we don't need the type cast. We're targeting a specific service principal by its ID. The `options.RelativePath` override replaces the configured path entirely with `/beta/serviceprincipals/{id}`.

**`DeleteForAppAsync` uses an app-only token.** Like creation, the deletion call uses client credentials flow (`RequestAppToken: true` from the configuration). The API authenticates the incoming caller with a delegated token, but the outbound Graph call uses the blueprint's app-only token acquired via the managed identity federated credential.

**The endpoint uses `MapDelete`.** The getting started guide maps its delete endpoint as a `GET`, passing the ID as a query parameter. The sample uses `MapDelete` with the ID as a route parameter (`/agent-identity/{id}`), following REST conventions where `DELETE` on a resource URI removes that resource.

## Running the sample

If you want to try this yourself, the entire sample deploys with three commands. You'll need an Azure subscription, an Entra ID tenant where you have permissions to create app registrations, and the [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) installed. You can also run this sample in the provided DevContainer!

### Clone and initialize

```bash
git clone https://github.com/Azure-Samples/agent-identity-samples.git
cd agent-identity-samples/entra-agent-id/create-agent-blueprint-and-identities

azd auth login
azd init
```

When `azd init` prompts you, select your Azure subscription and a region.

### Deploy with `azd up`

```bash
azd up
```

This single command runs the full lifecycle. During **pre-provision**, the script prompts for your Entra ID tenant ID and a display name for the blueprint, then triggers a device code flow to authenticate with Microsoft Graph.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o4jlvzpupgiolugrfg42.png)

Once you consent, the script creates the blueprint, configures the scope, and creates the service principal. The terminal output shows each step completing.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x9tpkd0s5474f145cakm.png)

Next, `azd` provisions the Azure infrastructure: Container Registry, Container App, managed identity, and Log Analytics. After provisioning, the **post-provision** hook triggers another device code flow to configure the federated identity credential and register the test client.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3n6epx2afzjyyty3qsbg.png)

Finally, the API is built, pushed to the Container Registry, and deployed to the Container App.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u8j5kctdqe1qjksuixh0.png)

### Test the API

With everything deployed, run the test client:

```bash
pwsh ./test-client.ps1
```

The script reads the configuration from the `azd` environment, acquires a token via device code flow (one more browser authentication), and calls the API to create an agent identity. It then asks if you want to delete it.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ygckvh6xecbd8nwxtife.png)

For a full detailed how-to for running this sample, please check the [README.md](https://github.com/willvelida/agent-identity-samples/blob/main/entra-agent-id/create-agent-blueprint-and-identities/README.md) file

## Wrapping up

In this post, we walked through the end-to-end process of creating an agent identity blueprint using Microsoft Graph PowerShell and then creating agent identities through an ASP.NET Web API running on Azure Container Apps. While Microsoft Entra Agent ID is still in preview, having a practical understanding of how blueprints, principals, and agent identities fit together will put you in a strong position as this platform matures. 

I'll be continuing to explore Entra Agent ID in future posts, including how we can call Azure services from agents using Entra Agent ID using the .NET SDK.

If you have any questions about the content here, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️

