---
title: "What are Agent Plugins?"
date: 2026-08-07
draft: false
tags: ["AI", "Agents", "MCP", "Agent Plugins"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/pgqd3k17h0wjbf2zi8z1.png
    alt: "The Agent Plugins portable package layout: plugin.json, skills/, and mcp.json"
    caption: "One package format for Agent Skills and MCP servers that works across VS Code, Copilot, Cursor, and more"
---

Implementing [Agent Skills](https://agentskills.io/) and [MCP servers](https://modelcontextprotocol.io/) as part of my development workflow has been a huge boost to my productivity. Long gone are the days where I'd actually have to leave my IDE to raise a pull request in GitHub, and then manually add that PR to a Jira ticket that's still impossible to find in 2026. 

Now it's just a matter of me asking GitHub Copilot to create the PR using my `create-pr` Agent Skill, then use the Atlassian MCP Server to link the PR to the appropriate Jira ticket. All of which I can do without ever having to leave my IDE.

However, I use different coding agents for my development. I'll use GitHub Copilot heavily for work, but I'll use Claude Code and Cursor for my side-projects. For those of you who've used both tools in anger will know that configuring Agent Skills, MCP servers, agents and other agentic components differ slightly between the two tools, which was a bit of a pain to deal with every time I wanted to use the same components in different tools.

The [Agent Plugins](https://agent-plugins.org/) standard was announced on August 6th 2026, which defined a packaging layer that sits on top of both Agent Skills and MCP Servers, giving developers a way of installing agentic coding components in a standardized format across different coding agent clients.

Considering it's only a day old (at time of writing), I decided to look at the docs, and see if I could develop a plugin for my health project using it's MCP server. There are some gaps that need to be addressed, but I think it's a big step towards a standard for giving developers what they need, rather than having to configure every individual component themselves.

In this blog post, I'll talk about what Agent Plugins are, why we need them, how we can build one, how we can install plugins across different clients, and what the gaps are that need to be addressed.

## What are Agent Plugins?

Agent Plugins are an open, vendor-neutral standard for packaging reusable agent components into portable plugins. It defines a shared format for Agent Skills and MCP servers that compatible clients can discover and load.

This is an initiative from AWS, Cursor, Microsoft, OpenAI, and Vercel, who are part of the [Technical Steering Committee](https://github.com/agentplugins/agent-plugins-spec/blob/main/GOVERNANCE.md). Google has also joined as a Core Maintainer.

Agent Plugins are just directories that include a required manifest and optional components. Take the following as an example:

```text
my-plugin/
├── plugin.json
├── skills/
│   └── weekly-health-brief/
│       ├── SKILL.md
│       ├── scripts/
│       └── references/
├── mcp.json
└── com.biotrackr.client/
    └── hooks/
```

The `plugin.json` identifies the plugin and the Agent Plugins version it targets.  `skills/` contains Agent Skills that the plugin will use. `mcp.json` describes MCP servers that the plugin has access to. These can be stdio, Streamable HTTP, or legacy HTTP+SSE servers.

![Diagram of the Agent Plugins portable package (plugin.json, skills/, and mcp.json) loading into compatible clients like VS Code, GitHub Copilot, and Cursor](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ovtwuu67aqze6e6s32of.png)

## Why do we need them?

Agent Skills and MCP servers have been fantastic in augmenting our coding agents. The problem was that each client had their own way of implementing them. For example, in VS Code you have [native support](https://code.visualstudio.com/docs/agent-customization/agent-skills) for the `SKILL.md` format. They live in `.github/skills/<name>`, `.agents/skills/` (project level), or `~/.copilot/skills/` (personal), and you define them like so:

```markdown
---
name: webapp-testing
description: Run and debug integration tests for the web app. Use when asked to test the login page or a UI flow.
---

# Web application testing

1. Start the dev server with `npm run dev`.
2. Use the template at [test template](./test-template.js).
...
```

In Cursor, you don't have native support for `SKILL.md`. Instead, Cursor's equivalent is something called [**Rules**](https://cursor.com/docs/context/rules) (that live in `.cursor/rules/*.mdc`) or just the `AGENTS.md` file. A rule is markdown and frontmatter controlling *when* it applies:

```markdown
---
description: RPC service conventions for the backend
globs: src/services/**/*.ts
alwaysApply: false
---

- Define each service in its own file under `src/services/`.
- Validate inputs at the service boundary.
- Reference @service-template.ts when creating a new service.
```

Agent Plugins is a good step towards solving this problem, as the [specification](https://agent-plugins.org/specification) defines a small interoperability floor for parts that can be portable across different coding agents. Shared components can use a predictable structure, while things like distribution, installation, permissions, UX and client-specifics can be handled by the clients themselves.


## Building a minimal plugin

For [Biotrackr](https://github.com/willvelida/biotrackr), I have an MCP server that exposes tools from across my various APIs. All this plugin will do is use that MCP server to retrieve information from my health data, as there are no write operations that the MCP Server exposes. Pretty basic, but good enough to see how Plugins work without potentially deleting any data.

Since we're just exploring Plugins, I'll keep the directory simple:

```text
biotrackr-brief/
├── plugin.json
├── skills/
│   └── weekly-health-brief/
│       └── SKILL.md
├── mcp.json
├── README.md
└── LICENSE
```

We'll define the `plugin.json` manifest, give it a single skill, create a `mcp.json` configuration file, and add a `README.md` and `LICENSE` file (even though I'm not publishing this 😅).

Let's take a look at the `plugin.json` manifest:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "biotrackr-brief",
  "version": "1.0.0",
  "description": "Turns Biotrackr health data into a short weekly brief. Bundles a weekly-health-brief skill with Biotrackr's MCP server (activity, sleep, vitals, and nutrition tools).",
  "author": {
    "name": "Will Velida",
    "url": "https://github.com/willvelida"
  },
  "homepage": "https://github.com/willvelida/biotrackr",
  "repository": "https://github.com/willvelida/biotrackr",
  "license": "MIT",
  "keywords": [
    "health",
    "fitbit",
    "withings",
    "mcp",
    "weekly-brief",
    "activity",
    "sleep",
    "vitals",
    "nutrition"
  ]
}
```

The `SKILL.md` file is pretty simple, we're just going to ask it to retrieve a weekly summary of our data for us:

```markdown
---
name: weekly-health-brief
description: Assembles a concise weekly health brief from Biotrackr activity, sleep, vitals, and nutrition data. Use when the user asks for a weekly summary, a health recap, "how was my week", or a brief covering steps, sleep, weight, or diet trends over the last seven days.
---

# Weekly health brief

etc. etc.
```

We can also configure the MCP servers that the plugins will use in the `mcp.json` file like so:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "biotrackr": {
      "type": "streamable-http",
      "url": "https://biotrackr-mcp-server-dev.io",
      "headers": {
        "X-Api-Key": "API_KEY"
      }
    }
  }
}

```

## Loading the plugin across clients

In VS Code, we can load our plugin via our user settings json file. Within it, we need to add the following:

```json
{
    "chat.plugins.enabled": true,
    "chat.pluginLocations": {
        "<directory-path>": true
    }
}
```

The `chat.plugins.enabled` setting turns the Agent Plugin feature on. The `chat.pluginLocations` maps the plugin's root directory to `true`. VS Code [auto-detects the Agent Plugins format](https://code.visualstudio.com/docs/agent-customization/agent-plugins) from the `$schema` in the `plugin.json` file, then it loads the `weekly-health-brief` skill and the `biotrackr` MCP server from the `mcp.json`.

You can manage the plugin in the **Extensions** view. Search `@agentPlugins` in the Extensions view to see the **Agent Plugins** that have been installed:


![The VS Code Extensions view filtered by @agentPlugins, showing the installed biotrackr-brief Agent Plugin](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/3gi9j9mvexyvhjwyvkws.png)

Our Agent Plugin can now be added to other Agent-Plugin compatiable clients, like Cursor. We can add our plugin using the **Customize** menu, and then adding the plugin from our local repository:

![Cursor's Customize menu with the biotrackr-brief plugin being added from a local repository](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/4d20mf2723jv24xmru7g.png)

We can inspect the plugin and see what skills are available to the plugin, as well as any MCP Servers that are packaged as part of the plugin. For my Biotrackr plugin, I can see the `weekly-health-brief` skill and biotrackr MCP as part of the plugin.

![Cursor showing the biotrackr-brief plugin's contents: the weekly-health-brief skill and the biotrackr MCP server](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/cme2b63yqicsam1iijh7.png)

I can also see what MCP tools are available as part of the plugin:

![The list of MCP tools the biotrackr MCP server exposes to the plugin in Cursor](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/6kn0vqz0qu9ylve5xwqq.png)

From there, we can start a new chat, and use the skill to get a weekly view of my health data. For example, let's take a look at my data for the week:

![A Cursor chat using the weekly-health-brief skill to produce a weekly view of Biotrackr health data](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/jx4fosie8j6d4nsnb2v2.png)

Drilling down into Cursor's thought process, we can see that Cursor has taken a look at the `SKILL.md` file, and used the MCP server to fetch our weekly data from all our domains:

![Cursor's reasoning trace showing it read the SKILL.md file and called the biotrackr MCP server to fetch the week's data across all health domains](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/96xgrszvyzfl9ahz8yph.png)

Using the same file format and specification, we can use skills and MCP servers bundled as a plugin across two clients without having to change any of the files to be compliant with a particular coding assistant.

## Limitations

It's important to point out that this is just a starting point. I think it's a step in the right direction, but there's a couple of features missing that still need to be addressed.

Currently, there's no permission model, no sandboxing, no trust/provenance and no approval UX for MCP servers that execute commands or access external services. As you can probably imagine, that's a massive task, as install, policy, and enterprise controls will differ across developer environments, CLIs and managed platforms.

There's also no portable secret mechanism. The `mcp.json` schema is closed, and it forbids any credentials in `headers` or `env`. This makes sense from a package perspective, as packages are intended to be for public consumption.

My MCP Server sits behind Azure APIM, and there's no portable slot for the subscription key in the plugin's `mcp.json` schema.

For my local plugin, it was fine to hardcode the `X-Api-Key` header just to get it running on my machine. If this was a package that I published, authentication would have to be handled by the client. Taking VS Code as an example, that would come from VS Code's `.vscode/mcp.json` config file.

There are a list of [future considerations](https://github.com/agentplugins/agent-plugins-spec/blob/main/FUTURE_CONSIDERATIONS.md) that the Agent Plugin team are working on, so I expect these questions to be answered over time.

Another gap is support for [Claude Code](https://code.claude.com/docs/en/plugins-reference). Claude has its own plugin system which already comes with skills, agents, hooks, commands, and MCP servers. What differs between Claude Code plugins and Agent Plugins are the manifest and discovery. Agent Plugins uses a root `plugin.json` with the canonical `$schema`; Claude Code uses `.claude-plugin/plugin.json`.

## Conclusion

Agent Plugins are a simple, but positive step forward for sharing Agentic coding capabilities across teams who use different agentic coding tools. It was a pain having to make edits to skills and MCP servers just so they could work from one tool to another.

There are a couple of major gaps that need to be addressed, but I think this is a big step towards having mature package managers for Agent Plugins, rather than having to copy/paste between different repositories and coding environments!

If you have any questions about this, please feel free to reach out to me on Bluesky!

Until next time, Happy coding! 🤓🖥️
