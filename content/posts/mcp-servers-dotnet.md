---
title: "Building a Sports-Themed MCP Server Using .NET"
date: 2025-04-16
draft: false
tags: ["Dotnet", "Model Context Protocol", "LLMs", "GenAI", "AI"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p9bh6a4h0bq4dk2imdaa.png
    alt: "Building a Sports-Themed MCP Server Using .NET"
    caption: "Model Context Protocols standardizes how we can provide context to LLMs. I've lived in Australia for 2 years and know nothing about AFL. Can we use MCPs to learn about AFL?"
---

I've lived in Melbourne for almost two years now, and throughout that time I've been trying to get my head around Australian rules, or AFL. I live pretty close to where Carlton Football Club train, so I've decided to support them (or as the Australians say *"Go for them"* 🤷). The more I learn about the game, the more I'm realizing that I may have set myself up for a lifetime of pain and dissappointment, but being an Arsenal fan since I was 6, I'm more than prepared for it 😂

While I was attending MVP Summit in March, people couldn't stop talking about MCP or **Model Context Protocol**, and how it can be used to provide context to Large-Language Models. I have to admit that I felt some apprehension about ANOTHER term that I'd have to get my head around just in case a customer asked for "One MCP please", but after doing a bit of reading on what it was, I saw it as an opportunity to try improve my AFL knowledge and learn about how MCP works (Kill two birds with one stone....metaphorically of course).

## What is the Model Context Protocol?

Model Context Protocol, or MCP for short, is an open protocol that standardizes how applications can provide context to LLMs. MCP provides a standardized way to connect AI models to different data sources and tools to help us build agents and complex workflows on top of LLMs.

When we use LLMs, they'll need to integrate with a variety of different tools and data sources. MCP provides a list of pre-built integrations that we can integrate with LLMs. It also gives us the flexibility to change between different LLMs and different vendors (so from GPT to Deepseek for example).

## How does it work?

MCP follows a client-server architecture where a host can connect to multiple servers. Take a look at the diagram below:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/81zgsv67qbvxkg6tlgf2.png)

MCP hosts like Claude Desktop, IDEs, or other AI tools act as clients that maintain connections to MCP Servers that expose capabilities through the MCP Protocol.

Clients maintain 1-to-1 connections with servers inside the host application.

MCP servers can connect to data that's hosted locally (like your filesystem, databases etc.) or remotely (such as API's or remote databases) over the internet.

MCP has a protocol layer that handles message framing, request/response linking, and high-level communication patterns, and a transport layer that handles communication between clients and servers.

There are two transport mechanisms that MCP supports:

1. *Stdio transport*

This uses standard input/output transport for communication, which is ideal for local processes.

2. *HTTP with SSE transport*

This uses Server-Sent Events (SSE) for server-to-client messages, and HTTP POST for client-to-server messages.

Both transports use JSON-RPC 2.0 to exchange messages.

### Resources

Resources in MCP allow servers to expose data and content that can be read by clients and used as context for LLM interactions.

They represent any kind of data that an MCP server want to make available to clients, including:

- File contents
- API responses
- Database records
- Images
- and more!

Resources can contain two tyoes of content: Text resources, and Binary resources.

Read more about [Resources here](https://modelcontextprotocol.io/docs/concepts/resources)

### Prompts

Prompts enable servers to define reusable prompt templates and workflows that clients can surface to users and LLMs. This provides a mechanism to standardize common interactions with LLMs.

Prompts in MCP can accept dynamic arguments, include context from resources, and chain multiple interactions.

Read more about [Prompts here](https://modelcontextprotocol.io/docs/concepts/prompts)

### Tools

Tools enable LLMs to perform actions through your MCP server. They expose functionality to clients, and allow LLMs to interact with external systems, perform computations, and take actions.

Tools are exposed from servers to clients to enable LLMs to directly invoke them.

Tools can be discovered by clinets, and invoked to perform operations. Tools can range in complexity, from performing simple calculations to complex API interactions.

Read more about [Tools here](https://modelcontextprotocol.io/docs/concepts/tools)

## Building our MCP Server

For my AFL MCP server, I'm going to build a server that'll connect to the [Squiggle API](https://api.squiggle.com.au/) that provides some basic AFL data (such as team information, standings in the ladder, results from games etc.) 

I'll create several tools that I can use within a MCP host (in this case, Claude Desktop) that will enable Claude Sonnet LLM to perform actions through our server.

I'll building a simple console application for this, which we can create using the `dotnet` CLI:

```bash
dotnet new console --name mcp-afl-server
```

We then need to add two NuGet packages to our console app. The first one will be **ModelContextProtocol** which adds the Model Context Protocol SDK to our application. The second will be the **Microsoft.Extensions.Hosting** package. We can install both using the `dotnet` CLI like so:

```bash
# Add the Model Context Protocol SDK NuGet package
dotnet add package ModelContextProtocol --prerelease
# Add the .NET Hosting NuGet package
dotnet add package Microsoft.Extensions.Hosting
```

*At the time of writing, the ModelContextProcotol package is in beta. Check it out on [GitHub](https://github.com/modelcontextprotocol/csharp-sdk) to see how development is going!*

Once that's done, we can start to build our server! In the `Program.cs` file, we can write the following:

```csharp
using System.Net.Http.Headers;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = Host.CreateEmptyApplicationBuilder(settings: null);

// Create the MCP Server with Standard I/O Transport and Tools from the current assembly
builder.Services.AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

builder.Logging.AddConsole(options =>
{
    options.LogToStandardErrorThreshold = LogLevel.Trace;
});

builder.Services.AddSingleton(_ =>
{
    var client = new HttpClient() { BaseAddress = new Uri("https://api.squiggle.com.au/") };
    client.DefaultRequestHeaders.UserAgent.Add(new ProductInfoHeaderValue("mcp-afl-server", "1.0"));
    return client;
});

var app = builder.Build();

await app.RunAsync();
```

As you can see, there's nothing to complex about this! Let's break it down:

- We start by creating a minimal host without any pre-configured services or settings. We use `CreateEmptyApplicationBuilder` to ensure that we don't write any additional messages to the console, which is only necessary for servers that use STDIO transport.
- We then create our MCP server and configure it to use STDIO Transport, and then load Tools from the current assembly (Which we'll get into in a bit).
- Add some logging to the console and set the error threshold to trace.
- We then add a singleton instance of an `HttpClient` that we'll use to connect to the Squiggle API. Ive also defined a custom UserAgent header to identify the client when making requests.

Now let's take a look at how we can create Tools for our MCP Server. I've created a tool that will help a LLM retrieve data from the Squiggle API on AFL standings (where a team is placed in the ladder):

```csharp
using System.ComponentModel;
using System.Net.Http.Json;
using System.Text.Json;
using mcp_afl_server.Models;
using ModelContextProtocol.Server;

namespace mcp_afl_server.Tools
{
    // This class contains tool execution handlers for querying and converting responses from the Squiggle AFL API service
    [McpServerToolType]
    public static class StandingsTools
    {
        [McpServerTool, Description("Gets the current standing")]
        public static async Task<string> GetCurrentStandings(HttpClient httpClient)
        {
            var response = await httpClient.GetFromJsonAsync<JsonElement>("?q=standings");
            var standingsResponse = JsonSerializer.Deserialize<List<StandingsResponse>>(
                response.GetProperty("standings").GetRawText());

            if (!standingsResponse.Any() || standingsResponse == null)
            {
                return "Unable to retrieve current standings";
            }

            return string.Join("\n--\n", standingsResponse.Select(standings => FormatStandingsResponse(standings)));
        }

        [McpServerTool, Description("Get the standings for a particular round and year")]
        public static async Task<string> GetStandingsByRoundAndYear(
            HttpClient httpClient,
            [Description("The round that has been played")] int roundNumber,
            [Description("The year of the standings")] int year)
        {
            var response = await httpClient.GetFromJsonAsync<JsonElement>($"?q=standings;year={year};round={roundNumber}");
            var standingsResponse = JsonSerializer.Deserialize<List<StandingsResponse>>(
                response.GetProperty("standings").GetRawText());

            if (!standingsResponse.Any() || standingsResponse == null)
            {
                return $"No standings found for the year {year} after round {roundNumber}";
            }

            return string.Join("\n--\n", standingsResponse.Select(standings => FormatStandingsResponse(standings)));
        }

        private static string FormatStandingsResponse(StandingsResponse standing)
        {
            return $"""
                Rank: {standing.Rank}
                Name: {standing.TeamName}
                Played: {standing.Played}
                Wins: {standing.Wins}
                Draws: {standing.Draws}
                Losses: {standing.Losses}
                For: {standing.For}
                Against: {standing.Against}
                Points: {standing.Points}
                Goals For: {standing.GoalsFor}
                Goals Against: {standing.GoalsAgainst}
                Behinds For: {standing.BehindsFor}
                Behinds Against: {standing.BehindsAgainst}
                Percentage: {standing.Percentage}
            """;
        }
    }
}
```

Breaking this down:

- The `[McpServerToolType]` attribute that I've applied to the `StandingTools` tasks indicates that the methods in this class should be exposed as MCP Tools.
- We then describe our tools by annotating our method with `[McpServerTool]` and provide a description for that tool using the `[Description]` annotation.
- Each tool will have a property or event that it can use when performing actions. These are provided as parameters within our methods, and are annotated using the `[Description]` annotation.

I've captured the expected JSON response from the API in the `StandingsResponse` class, and created a simple method that formats that response from the API. The LLM will then use this response as context when responding to our prompts.

## Testing our Server

Let's test the server. We can run the server with the `dotnet` CLI like so:

```bash
dotnet run
```

You can develop your own client to interact with your MCP server, but for simplicity I'm going to use Claude Desktop.

To configure Claude Desktop to use our MCP server, we'll need to edit the configuration file (which is a JSON file). To find it on Windows, run the following (assuming you have VS Code installed):

```bash
code $env:AppData\Claude\claude_desktop_config.json
```

If you don't have VS Code installed, just `pushd` into the Claude folder, and open that configuration file from there.

We can add our MCP servers into the `mcpServers` key like so:

```json
{
    "mcpServers": {
        "mcp-afl-server": {
            "command": "dotnet",
            "args": [
                "run",
                "--project",
                "\\ABSOLUTE_PATH_TO_PROJECT\\mcp-afl-server.csproj",
                "--no-build"
            ]
        }
    }
}
```

This MCP configuration tells Claude for Desktop that there's an MCP Server called `mcp-afl-server` and it should launch it by running `dotnet run` on the project path.

Open Claude for Desktop, and if everything is working correctly you should see the tools that have been exposed in with the hammer icon like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n2g1qepdnssdzvuubll0.png)

This MCP Server has 16 tools that I've created, and we can see more details on the tools by clicking that hammer icon:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5k4k4tgic6e8vtmj143y.png)

Each tool exposed is described using the details that you've provided in the `[Description]` annotation, so make sure you've described your tools properly!

Let's see this in action! I'm going to ask Claude where Carlton finished in the AFL standing in 1995 (The last time we won the flag....yep, that long ago 🥲)

```text
Where did Carlton finish in the AFL in 1995?
```

When I ask Claude that question, it'll use the `GetStandingsByRoundAndYear` tool from my MCP server:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3s1rd73x5hwmm0qgwxma.png)

As you can see, Claude has taken my prompt and used the tool to answer my question. From the prompt, it's taken the year that I want the standings for along with the knowledge that there are 22 rounds in the AFL (before playoffs). 

Underneath the hood, these parameters are used in the tool to retrieve that information from the API, and we can see the response from the API within the Claude UI. The following JSON response has been captured:

```json
{
  `year`: 1995,
  `roundNumber`: 22
}
    Rank: 1
    Name: Carlton
    Played: 22
    Wins: 20
    Draws: 0
    Losses: 2
    For: 2357
    Against: 1711
    Points: 80
    Goals For: 340
    Goals Against: 246
    Behinds For: 317
    Behinds Against: 235
    Percentage: 137.755698421975

    // Extra response omitted
```

For some queries, LLM can invoke multiple tools or the same tool repeatedly to get to the final answer. For example, if I wanted to know who Carlton played in the 1995 playoffs, I can ask Claude the following:

```text
Who did they play in the playoffs?
```

From the image below, you can see Claude LLM invoke the `GetRoundByResultsByYear` tool multiple times to retrieve all the information it needs to provide me with the final answer.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j85ha8pr0whvgx6nc5x4.png)

Hard to believe that this was almost 30 years ago! At the time of writing, Carlton are currently 1-4 in the season, so I'll have to wait a little while longer before they capture another flag (*It's what they call the AFL Championship. I'm not sure why either if that helps*).

## Conclusion

Hopefully you now have a better understanding of how MCP works, and how you can build simple MCP servers to do some pretty powerful stuff! This was just a basic abstraction of calling a REST API for some data to supply LLMs with context, but I'm sure you can imagine that through MCP, we can build some pretty simple integrations with LLMs to do some powerful stuff.

I only showed a portion of the code, so check out the [repository on GitHub](https://github.com/willvelida/mcp-afl-server) to see the complete code.

The next steps will be to figure out how I can host this MCP server remotely. In the Azure world, there's support for [Azure Functions](https://techcommunity.microsoft.com/blog/appsonazureblog/build-ai-agent-tools-using-remote-mcp-with-azure-functions/4401059), [Azure Container Apps](https://techcommunity.microsoft.com/blog/appsonazureblog/host-remote-mcp-servers-in-azure-container-apps/4403550), and [Azure AI Foundry](https://devblogs.microsoft.com/foundry/integrating-azure-ai-agents-mcp/). I'll explore how these work and report back.

If you have any questions about this, please feel free to either comment below or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)!

Until next time, Happy coding! 🤓🖥️

