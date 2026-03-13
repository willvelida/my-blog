---
title: "Securing AI Agents: Implementing the OWASP Top 10 for Agentic Applications to my Health Data Agent"
date: 2026-03-13
draft: false
tags: ["Agents", "AI", ".NET", "OWASP", "Security", "Microsoft Agent Framework"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hzwe4hgc9p98fey45mkt.png
    alt: "Overview of the OWASP Agentic Top 10 implemented in a .NET AI agent with practical security controls."
    caption: "A practical walkthrough of the OWASP Agentic Top 10, implemented in a real .NET 10 AI agent built with the Microsoft Agent Framework."
---

The OWASP Top 10 for Agentic Applications (2026) identifies the most critical security risks facing AI agents. From prompt injection and tool misuse to identity abuse and cascading failures. The guidance is thorough, but what does it actually look like to implement these controls in a .NET agent?

This series answers that question by walking through every applicable control from the OWASP Agentic Top 10, showing how each was implemented in [Biotrackr](https://github.com/willvelida/biotrackr), my personal health data tracker with a Claude-powered chat agent built on Microsoft Agent Framework, .NET 10, and Azure.

## Why Agents Need Their Own Threat Model

Traditional web apps have clear request-response boundaries. You validate input, sanitize output, and apply authorization at well-defined checkpoints. AI agents blur all of these lines.

Agents make autonomous decisions: which tools to call, what parameters to use, how to interpret results. The LLM is both the brain and the attack surface. It processes user input AND tool results as context. A malicious payload in a tool result is just as dangerous as one in a user message.

The standard OWASP Top 10 for web apps doesn't cover agent-specific risks like goal hijacking, tool misuse, or memory poisoning. That's why the OWASP Agentic Top 10 exists, and why I've spent time implementing these controls in my own project.

## The Biotrackr Chat Agent

Biotrackr is my side project that tracks health data from Fitbit, which includes data for sleep, activity, food, and weight. The chat agent is a .NET 10 Minimal API running as an Azure Container App, using Microsoft Agent Framework with Claude Sonnet 4.6 via the Anthropic provider. It has 12 function tools that call existing health data APIs through Azure API Management, persists chat history in Cosmos DB, and streams responses to a Blazor UI via the AG-UI protocol.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zvglbru4j9p5jww6pbuw.png)

There's a chat interface that I use to query to agent, and the agent decides which tools to call, tool results come back as LLM context, and the agent responds. Every step in that pipeline is a potential attack surface.

## The OWASP Agentic Top 10 — Applied to Biotrackr

Here's a quick reference of each vulnerability and how it applies to Biotrackr, with links to the detailed posts.

| ID | Vulnerability | Biotrackr Risk |
|----|--------------|----------------|
| ASI01 | Agent Goal Hijack | HIGH — user input + tool results as LLM context |
| ASI02 | Tool Misuse and Exploitation | HIGH — 12 tools calling external APIs |
| ASI03 | Identity and Privilege Abuse | MEDIUM — solved with Entra Agent ID |
| ASI04 | Agentic Supply Chain Vulnerabilities | MEDIUM — preview NuGet packages |
| ASI05 | Unexpected Code Execution | LOW — no code execution tools |
| ASI06 | Memory and Context Poisoning | MEDIUM — multi-turn conversations + persistence |
| ASI07 | Insecure Inter-Agent Communication | N/A — single agent architecture |
| ASI08 | Cascading Failures | MEDIUM — Claude API + APIM dependencies |
| ASI09 | Human-Agent Trust Exploitation | MEDIUM — health data = high trust risk |
| ASI10 | Rogue Agents | LOW — constrained scope |

### ASI01 — Agent Goal Hijack

**What is it?** An attacker manipulates the agent's objectives, task selection, or decision pathways through prompt-based manipulation, deceptive tool outputs, or poisoned external data.

**Why it matters for Biotrackr:** The chat agent processes user messages and API responses as context for the LLM. A compromised data source could carry a prompt injection payload that redirects the agent's behaviour.

**What I implemented:** Strict input validation on all tool parameters (dates validated as `DateOnly`, page sizes capped at 50), immutable system prompts loaded from Azure App Configuration at startup, structured JSON-only tool responses with no free-text fields, and comprehensive logging of every tool call for audit.

👉 [Read the full ASI01 post — Preventing Agent Goal Hijack in .NET AI Agents](https://www.willvelida.com/posts/preventing-agent-goal-hijack/)

### ASI02 — Tool Misuse and Exploitation

**What is it?** An attacker exploits tools accessible to the agent. Excessive permissions, lack of rate limiting, or unvalidated parameters allow the agent to perform actions beyond its intended scope.

**Why it matters for Biotrackr:** The agent has 12 tools calling external APIs. Without constraints, a hijacked agent could exfiltrate data through large result sets or abuse tool calls at high frequency.

**What I implemented:** All tools are read-only (HTTP GET only), date range queries are capped at 365 days, page sizes are hard-capped at 50, and every tool call is logged with OpenTelemetry tracing. The agent simply cannot mutate data.

👉 [Read the full ASI02 post — Preventing Tool Misuse in AI Agents](https://www.willvelida.com/posts/preventing-tool-misuse-ai-agents/)

### ASI03 — Identity and Privilege Abuse

**What is it?** An agent operates with excessive privileges or uses a shared identity, allowing it to access resources beyond its intended scope. This is the classic "over-privileged service account" problem, amplified by autonomous decision-making.

**Why it matters for Biotrackr:** The chat agent calls downstream APIs and accesses Cosmos DB. If it shared the app's identity with broad permissions, a compromised agent could access anything the app can.

**What I implemented:** Microsoft Entra Agent ID gives the agent its own first-class identity with federated credentials. RBAC is scoped to the minimum required; read-only access to the specific APIs and data stores it needs, nothing more.

👉 [Read the full ASI03 post — Preventing Identity and Privilege Abuse in AI Agents](https://www.willvelida.com/posts/preventing-identity-and-privilege-abuse-ai-agents/)

### ASI04 — Agentic Supply Chain Vulnerabilities

**What is it?** Vulnerabilities in the agent's dependencies; frameworks, plugins, model providers, or tools that can be exploited to compromise the agent. AI frameworks are evolving rapidly, and many packages are in preview.

**Why it matters for Biotrackr:** The agent depends on preview-stage NuGet packages from Microsoft Agent Framework and the Anthropic provider. Preview packages can have breaking changes, undiscovered vulnerabilities, or unstable APIs.

**What I implemented:** Pinned NuGet package versions, lock files committed to source control, Dependabot configured for automated dependency updates, and a clear governance process for upgrading preview packages.

👉 [Read the full ASI04 post — Preventing Agentic Supply Chain Vulnerabilities](https://www.willvelida.com/posts/preventing-agentic-supply-chain-vulnerabilities/)

### ASI05 — Unexpected Code Execution

**What is it?** The agent executes code that wasn't intended by its designers. Either through code generation tools, dynamic evaluation, or injection into executable contexts.

**Why it matters for Biotrackr:** Mostly it doesn't. The agent has no code execution tools, no `eval()`, and no dynamic compilation. This is a deliberate architectural decision that eliminates an entire class of vulnerabilities.

**What I implemented:** The absence of code execution tools IS the control. Tools only perform HTTP GET requests and return structured data. If code execution tools were ever needed, they'd require sandboxed execution environments and human approval.

👉 [Read the full ASI05 post — Preventing Unexpected Code Execution in AI Agents](https://www.willvelida.com/posts/preventing-unexpected-code-execution-in-agents/)

### ASI06 — Memory and Context Poisoning

**What is it?** An attacker corrupts the agent's memory or conversation context to influence future behaviour. In multi-turn conversations, earlier messages become "trusted" context that shapes how the agent interprets later inputs.

**Why it matters for Biotrackr:** Chat history is persisted in Cosmos DB and loaded as context for subsequent interactions. A poisoned conversation turn could influence the agent's behaviour across an entire session.

**What I implemented:** Cosmos DB TTL on conversation documents to limit the blast radius of poisoned context, bounded context windows so the agent only loads recent history, and structured message format that separates user messages from system context.

👉 [Read the full ASI06 post — Preventing Memory and Context Poisoning in AI Agents](https://www.willvelida.com/posts/preventing-memory-context-poisoning/)

### ASI07 — Insecure Inter-Agent Communication

**What is it?** When multiple agents communicate, messages between them can be intercepted, spoofed, or manipulated if communication channels lack authentication, encryption, or message integrity.

**Why it matters for Biotrackr:** It doesn't. Biotrackr uses a single agent with no inter-agent orchestration. This is worth calling out explicitly because choosing a single-agent architecture eliminates an entire vulnerability class. If I ever add multi-agent orchestration, this becomes a priority. In the article below, I discuss what this *might* look like in the context of Biotrackr.

👉 [Read the full ASI07 post — Preventing Insecure Inter-Agent Communication in AI Agents](https://www.willvelida.com/posts/preventing-insecure-inter-agent-communication/)

### ASI08 — Cascading Failures

**What is it?** A failure in one component (the LLM provider, a downstream API, or a tool) propagates through the agent system, causing widespread outages or degraded behaviour. Agents are particularly susceptible because they chain multiple service calls autonomously.

**Why it matters for Biotrackr:** The agent depends on Claude's API (via Anthropic) and multiple downstream health data APIs through APIM. If Claude goes down or an API times out, the agent could hang, retry endlessly, or return garbage.

**What I implemented:** `AddStandardResilienceHandler()` on all HTTP clients for circuit breaking, retry with exponential backoff, and timeout policies. Graceful degradation in tool responses. If an API is unavailable, the tool returns a structured error, not an exception. Token budgets prevent runaway LLM calls.

👉 [Read the full ASI08 post — Preventing Cascading Failures in AI Agents](https://www.willvelida.com/posts/preventing-cascading-failures-ai-agents/)

### ASI09 — Human-Agent Trust Exploitation

**What is it?** Users over-trust the agent's output, treating it as authoritative when it shouldn't be. This is especially dangerous in health, legal, and financial domains where misplaced trust can lead to real harm.

**Why it matters for Biotrackr:** This is a health data agent. If a user asks "should I be worried about my heart rate?" and the agent gives a confident answer, that's genuinely dangerous. Users may treat the agent's analysis as medical advice.

**What I implemented:** The system prompt explicitly states the agent is not a medical professional and directs users to consult healthcare providers. The UI visually differentiates agent responses from factual data. Health disclaimers are baked into the agent's behaviour, not bolted on.

👉 [Read the full ASI09 post — Preventing Human-Agent Trust Exploitation in .NET AI Agents](https://www.willvelida.com/posts/preventing-human-agent-trust-exploitation/)

### ASI10 — Rogue Agents

**What is it?** The agent itself becomes the threat. Whether through compromised training data, manipulated system prompts, or lack of runtime constraints, the agent operates outside its intended boundaries.

**Why it matters for Biotrackr:** Even though the risk is low for a constrained side project, defence in depth means we plan for the worst. If the agent's behaviour drifted due to a poisoned system prompt update or a compromised dependency, we need detection and kill switches.

**What I implemented:** All tool calls are logged and persisted for audit, the system prompt is version-controlled in Azure App Configuration with RBAC, the agent has no self-modification capabilities, and the architecture supports a kill switch through configuration changes without redeployment.

👉 [Read the full ASI10 post — Preventing Rogue AI Agents](https://www.willvelida.com/posts/preventing-route-agents/)

## Cross-Cutting Themes

A few patterns show up across nearly every control:

- **Structured JSON responses** — tools return minimal, structured data with no free-text fields that could carry injection payloads (ASI01, ASI02, ASI06)
- **Input validation at every boundary** — tool parameters, API responses, user messages are all treated as untrusted (ASI01, ASI02)
- **Principle of least privilege** — the agent identity has read-only access, tools can only query data, and RBAC is scoped to the minimum required (ASI03, ASI10)
- **Observability** — OpenTelemetry tracing and conversation persistence create a full audit trail of every tool call and agent response (ASI01, ASI02, ASI08, ASI10)
- **Defence in depth** — no single control is relied upon in isolation; multiple layers work together (all)

## Wrapping Up

The OWASP Agentic Top 10 gives us a structured framework for thinking about agent security, and this series shows how to put it into practice.

Start with [Part 1 — Preventing Agent Goal Hijack](https://www.willvelida.com/posts/preventing-agent-goal-hijack/), or jump to whichever vulnerability is most relevant to your architecture. If you're building agents, I'd strongly encourage you to assess them against the OWASP Agentic Top 10.

If you have any questions about the content here, please feel free to reach out to me on [Bluesky](https://bsky.app/profile/willvelida.com) or comment below.

Until next time, Happy coding! 🤓🖥️
