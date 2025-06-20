---
title: "Building Remote MCP Servers with .NET and Azure Container Apps"
date: 2025-06-20
draft: false
tags: ["Dotnet", "Model Context Protocol", "LLMs", "GenAI", "AI", "Azure Container Apps"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h82bssymqql8p4lp3qtf.png
    alt: "Building Remote MCP Servers with .NET and Azure Container Apps"
    caption: "Learn how to build remote MCP servers using .NET with SSE, and then deploy them to Azure Container Apps"
---

A couple of months ago, I wrote a blog post on how you can create Model Context Protocol (MCP) servers using C#. Using a basic API, I was able to create a MCP server that allowed me to call Australian Football League (AFL) data and supply that as context to LLMs so I can ask it question about AFL results, teams, stats etc. using that API.

That blog post talked about how we can use MCP servers that run locally on our machines using `stdio` transport. In this article, I'll talk about how we can use *Server-Sent Events* (SSE) transport to build remote MCP servers that we can host on Azure Container Apps.

## What are MCP servers again?

Model Context Protocol, or MCP for short, is an open protocol that standardizes how applications can provide context to LLMs. MCP provides a standardized way to connect AI models to different data sources and tools to help us build agents and complex workflows on top of LLMs.

When we use LLMs, they'll need to integrate with a variety of different tools and data sources. MCP provides a list of pre-built integrations that we can integrate with LLMs. It also gives us the flexibility to change between different LLMs and different vendors (so from GPT to Deepseek for example).

## How do remote MCP servers work?

Currently, MCP defines two standard transport mechanisms for client-server communication:

1. `stdio`, which is communication over standard in and standard out.
2. `Streamable HTTP`

For remote servers, Streamable HTTP transport allows the server to operate as an independent process that can handle multiple client connections.

Streamable HTTP uses HTTP POST and GET requests, along with *Server-Sent Events* (SSE) to stream multiple server messages.

For this to work, our MCP servers must provide an HTTP endpoint that supports both POST and GET methods. Thankfully, the C# SDK provides a package called `ModelContextProtocol.AspNetCore` that we can use to scaffold this capability.

In this article, I'm just going to go through the basics of deploying my MCP server to Azure Container Apps. I'll go into more detail in a future post on how we can implement security to ensure that our MCP servers don't get abused!

To read more about how the Streamable HTTP transport mechanism works in MCP, check out the [documentation](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http).

## Using the ModelContextProtocol.AspNetCore package

The MCP C# SDK provides the `ModelContextProcotol.AspNetCore` package that enables .NET applications to implement MCP server capabilities. Installing the NuGet package can be done using the dotnet cli like so:

```bash
dotnet add package ModelContextProtocol.AspNetCore --prerelease
```

*N.B - At the time of writing, I'm using version 0.2.0-preview.3*

This will give us the extension we need to build our remote MCP server using the Streamable HTTP transport mechanism. With the NuGet package installed, we can set up our `Program.cs` class like so:

```csharp
// Program.cs
using System.Net.Http.Headers;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly();

builder.Services.AddOpenTelemetry()
    .WithTracing(b => b.AddSource("*")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation())
    .WithMetrics(b => b.AddMeter("*")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation())
    .WithLogging()
    .UseOtlpExporter();

builder.Services.AddSingleton(_ =>
{
    var client = new HttpClient() { BaseAddress = new Uri("https://api.squiggle.com.au/") };
    client.DefaultRequestHeaders.UserAgent.Add(new ProductInfoHeaderValue("mcp-afl-server", "1.0"));
    return client;
});

var app = builder.Build();

app.MapGet("/api/healthz", () => Results.Ok("Healthy"));

app.MapMcp();

app.Run();
```

There's two important blocks of code that we need to examine here:

```csharp
builder.Services.AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly();
```

In this block, we're registering the MCP server and its dependencies into our application's dependency injection container.

The `.WithHttpTransport()` method configures the server to use HTTP as its transport mechanism, meaning that we'll be able to communicate with it over HTTP POST and GET. The `WithToolsFromAssembly()` method is the same as before when we used `stdio` transport, as it just registers the tools that we've defined in our application to the MCP server.

The other important line of code is this one:

```csharp
app.MapMcp();
```

This extension will set up the HTTP endpoints required for our remote MCP server. We can dive into the definition of this extension method using the **F12** key in VS Code:

```csharp
// https://github.com/modelcontextprotocol/csharp-sdk/blob/main/src/ModelContextProtocol.AspNetCore/McpEndpointRouteBuilderExtensions.cs
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.Metadata;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;
using ModelContextProtocol.AspNetCore;
using ModelContextProtocol.Protocol;
using System.Diagnostics.CodeAnalysis;

namespace Microsoft.AspNetCore.Builder;

/// <summary>
/// Provides extension methods for <see cref="IEndpointRouteBuilder"/> to add MCP endpoints.
/// </summary>
public static class McpEndpointRouteBuilderExtensions
{
    /// <summary>
    /// Sets up endpoints for handling MCP Streamable HTTP transport.
    /// See <see href="https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http">the 2025-03-26 protocol specification</see> for details about the Streamable HTTP transport.
    /// Also maps legacy SSE endpoints for backward compatibility at the path "/sse" and "/message". <see href="https://modelcontextprotocol.io/specification/2024-11-05/basic/transports#http-with-sse">the 2024-11-05 protocol specification</see> for details about the HTTP with SSE transport.
    /// </summary>
    /// <param name="endpoints">The web application to attach MCP HTTP endpoints.</param>
    /// <param name="pattern">The route pattern prefix to map to.</param>
    /// <returns>Returns a builder for configuring additional endpoint conventions like authorization policies.</returns>
    public static IEndpointConventionBuilder MapMcp(this IEndpointRouteBuilder endpoints, [StringSyntax("Route")] string pattern = "")
    {
        var streamableHttpHandler = endpoints.ServiceProvider.GetService<StreamableHttpHandler>() ??
            throw new InvalidOperationException("You must call WithHttpTransport(). Unable to find required services. Call builder.Services.AddMcpServer().WithHttpTransport() in application startup code.");

        var mcpGroup = endpoints.MapGroup(pattern);
        var streamableHttpGroup = mcpGroup.MapGroup("")
            .WithDisplayName(b => $"MCP Streamable HTTP | {b.DisplayName}")
            .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status404NotFound, typeof(JsonRpcError), contentTypes: ["application/json"]));

        streamableHttpGroup.MapPost("", streamableHttpHandler.HandlePostRequestAsync)
            .WithMetadata(new AcceptsMetadata(["application/json"]))
            .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status200OK, contentTypes: ["text/event-stream"]))
            .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status202Accepted));

        if (!streamableHttpHandler.HttpServerTransportOptions.Stateless)
        {
            // The GET and DELETE endpoints are not mapped in Stateless mode since there's no way to send unsolicited messages
            // for the GET to handle, and there is no server-side state for the DELETE to clean up.
            streamableHttpGroup.MapGet("", streamableHttpHandler.HandleGetRequestAsync)
                .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status200OK, contentTypes: ["text/event-stream"]));
            streamableHttpGroup.MapDelete("", streamableHttpHandler.HandleDeleteRequestAsync);

            // Map legacy HTTP with SSE endpoints only if not in Stateless mode, because we cannot guarantee the /message requests
            // will be handled by the same process as the /sse request.
            var sseHandler = endpoints.ServiceProvider.GetRequiredService<SseHandler>();
            var sseGroup = mcpGroup.MapGroup("")
                .WithDisplayName(b => $"MCP HTTP with SSE | {b.DisplayName}");

            sseGroup.MapGet("/sse", sseHandler.HandleSseRequestAsync)
                .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status200OK, contentTypes: ["text/event-stream"]));
            sseGroup.MapPost("/message", sseHandler.HandleMessageRequestAsync)
                .WithMetadata(new AcceptsMetadata(["application/json"]))
                .WithMetadata(new ProducesResponseTypeMetadata(StatusCodes.Status202Accepted));
        }

        return mcpGroup;
    }
}
```

There's quite a bit to unpack here, so let's break this down:

```MapMcp()``` starts by retrieving the `StreamableHttpHandler` from the dependency injection container. It'll throw an exception if we haven't registered our MCP Server with HTTP Transport. If we have registered the required services in our DI container, it'll create a route group using the provided pattern (or an empty string default) for the base path for all MCP endpoints.

Within this route group, *Streamable HTTP* endpoints are created to map the POST endpoints to handle incoming requests to our MCP server, accepting JSON requests and producing either *Server-Sent Events* for streaming responses or standard accepted responses.

There's also a conditional check based on the `Stateless` configuration option. In Stateful mode, additional `GET` and `DELETE` endpoints are mapped for the streamable HTTP handler. The `GET` enables server-initiated communication through streaming, and the `DELETE` endpoint allows clients to clean up server-side resources.

There's also conditional mapping for legacy `HTTP` with `SSE` when our server is not in `Stateless` mode, which includes a dedicated `/sse` endpoint for handling `SSE` connections and a `/message` endpoint for sending messages.

The method returns an `IEndpointConventionBuilder` to allow us to chain additional configuration such as authorization policies, CORS settings, or any other endpoint conventions we want to add.

## Deploying to Azure Container Apps

With our application code taken care off, we can containerize our MCP server just like we would with any other ASP.NET Core application through a `Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080


# This stage is used to build the service project
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["mcp-afl-server.csproj", "."]
RUN dotnet restore "./mcp-afl-server.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./mcp-afl-server.csproj" -c $BUILD_CONFIGURATION -o /app/build

# This stage is used to publish the service project to be copied to the final stage
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./mcp-afl-server.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# This stage is used in production or when running from VS in regular mode (Default when not using the Debug configuration)
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "mcp-afl-server.dll"]
```

Our application will be exposed on port `8080`, so in our Bicep code for our MCP server, we need to make sure our Container App's will accept incoming traffic on port 8080:

```bicep
resource mcpServer 'Microsoft.App/containerApps@2025-01-01' = {
  name: containerAppName
  location: location
  tags: tags
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${uai.id}': {}
    }
  }
  properties: {
    managedEnvironmentId: containerAppEnvironmentId
    configuration: {
      activeRevisionsMode: 'Single'
      ingress: {
        external: true
        targetPort: 8080
        allowInsecure: false
        traffic: [
          {
            latestRevision: true
            weight: 100
          }
        ]
      }
      registries: [
        {
          server: containerRegistry.properties.loginServer
          identity: uai.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: baseName
          image: imageName
          env: [
            {
              name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
              value: appInsights.properties.ConnectionString
            }
            {
              name: 'managedidentityclientid'
              value: uai.properties.clientId
            }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: {
                port: 8080
                path: '/api/healthz'
                scheme: 'HTTP'
              }
              initialDelaySeconds: 10
              periodSeconds: 5
              failureThreshold: 30
              timeoutSeconds: 2
            }
          ]
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 2
        rules: [
          {
            name: 'http-scale-rule'
            http: {
              metadata: {
                concurrentRequests: '100'
              }
            }
          }
        ]
      }
    }
  }
}
```

Deploying an MCP server to Azure Container Apps isn't any different to any other type of application. To see the full code sample (Both C# and Bicep), check out [my repository on GitHub](https://github.com/willvelida/mcp-afl-server).

I use GitHub Actions to deploy the sample, and hopefully you'll see that apart from setting up the endpoints in our MCP server app, there's no extra configuration that we need to do to host our remote server on Azure Container Apps, or any other platform (such as [Azure Functions](https://devblogs.microsoft.com/dotnet/build-mcp-remote-servers-with-azure-functions/) or [App Service](https://techcommunity.microsoft.com/blog/appsonazureblog/host-remote-mcp-servers-in-azure-app-service/4405082)).

## Testing our remote MCP server

Once our MCP Server has been deployed, we can interact with it using an MCP client, like Visual Studio Code.

*N.B - I know, asking GitHub Copilot in VS Code for AFL data seems strange. It's just a demo at the end of the day. Imagine I'm using an MCP server for [Azure DevOps](https://github.com/microsoft/azure-devops-mcp), or [MS Docs](https://github.com/microsoftdocs/mcp), and you'll appreciate why this functionality is handy.*

To add our MCP Server to Visual Studio Code, press *Ctrl + Shift + P* and run the *MCP: Add Server* command. Choose *HTTP (Server-sent events)* as the transport and enter the URL of your Container App with the `/sse` endpoint on the server.

VS Code will then add the MCP server to the `mcp.json` configuration file. It should look something like the following:

```json
"mcp": {
        "servers": {
            "aflmcpserver": {
                "url": "https://yourcontainerappurl.io",
                "type": "sse"
            }
        }
    },
```

*N.B - I orginally had the `/sse` endpoint appended to the `url` value, which was causing some 405 errors to be thrown. It still worked, but by removing it, the client (VS Code), can now call GET on `/sse` to establish the sse connection, and use POST on `/message` to send MCP requests.*

Next to the server in the JSON file, click `Start` and your MCP server should be connected to the server.

Open up GitHub Copilot chat, and you can start interacting with your MCP server in **Agent** Mode. Let's see how Carlton (*Go Baggers!*) did in the AFL last year:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rl2fd8v8ato56boa10sq.png)

When we ask our question, GitHub Copilot chat will use the `GetStandingsByRoundAndYear` tool on our AFL MCP Server, and use the data it retrieves as context to answer our question. Press *Continue* to use the tool, and we get the following response from GitHub Copilot:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/plcava9c2ud6l2e1ycbx.png)

## Conclusion

Hopefully after reading this article you have a better understanding on how we can build and host remote MCP servers using the `ModelContextProtocol.AspNetCore` package in the MCP C# SDK and then host our servers on a PaaS service like Azure Container Apps.

We haven't even touched on security yet, which is *extremely* important for implementing MCP servers. At the time of writing, [work](https://github.com/modelcontextprotocol/csharp-sdk/pull/377) is being done on Authorization support in the C# SDK. In a future article, I'll discuss how we can secure our MCP servers to prevent misuse.

Don't forget to check out the [full code for this sample on my GitHub!](https://github.com/willvelida/mcp-afl-server)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)!

Until next time, Happy coding! 🤓🖥️