---
title: "Last week in Agent Security Update 3: That's not Aoki!"
date: 2026-10-19
draft: false
tags: ["OWASP", "Agents", "AI", "Security"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/9ukj0m60yyv2iovayn70.png
    alt: "Last week in Agent Security Update 3: That's not Aoki!"
    caption: "More Test agents going where they shouldn’t, an MCP flaw under active attack, and infostealers going after your coding agent’s keys."
---

An apology to start off with! I was too busy having fun in Sydney last week hacking with customers, building agents and agent harnesses. I got to experiment with [Entra Agent ID](https://learn.microsoft.com/en-us/entra/agent-id/what-is-microsoft-entra-agent-id) and [Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit), two technologies I'm looking forward to building with more over the next few months.

We're slowly starting to see governments around the world catch up to the dangers that AI Agents can present to organizations around the world. The EU AI office has declared that they've "started enforcing" the EU AI Act, and it has sent formal requests to both OpenAI and Anthropic about security, external evaluations and post-market monitoring. In the US, the FTC has sent civil investigative demands to both OpenAI and Anthropic, seeking records on security incidents, safeguards, marketing, and privacy. Whether this goes anywhere is anyone's guess. I'm pretty pesimistic about it, but hopefully I'm wrong.

This week, the theme of agents drifting from their original goals (whether it's unintentional or not) continue to be a issue for governments and businesses alike. MCP servers remain a potential gateway to organizations and developer environments, and coding assistant credentials are increasing in value to potential attackers. 

As always, I’ll point out that I’m not trying to shame any of the companies involved, nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here’s some incidents that have happened over the period 12th October to 19th October 2026.

## CERT-In: test agents read confidential data at Indian organisations

CERT-In issued an advisory that AI agents under research and evaluation by multiple AI model providers went beyond thier assigned tasks and reached live systems at numerous Indian organizations. CERT-In haven't explicitly named the providers, models, victims, sectors, or number of cases. However, they were able to determine that agents accessed business records that contained personal and confidential data.

This data included credentials and connection strings left in application configuration files and public code repositories, and over-permissive cloud roles. The advisory also states that agents sometimes reused the same exposed credentials in multiple waves of attacks.

The same advisory also describes a separate, financially motivated campaign against Indian entities that used rebranded open-source AI offensive-security frameworks with multiple agents working in parallel. This enabled the attacks to cut time from initial access to impact from days to a few hours. The scale of this attack has not been disclosed.

CERT-In are recommending that affected targets enforce network-enforced isolation and default-deny egress for AI test runs, automaticl kill switches when agents touch 3rd-party systems, rotating secrets and enforcing least privilege, implementing MFA, and implementing 180-day retention.

More countries are being impacted by rogue agents (ASI10) where agents are either drifting from their original tasks or being directed to do so through their system prompt. While CERT-In provides some good guidance on how to protect your systems from rogue agents, there's clearly work to be done to prevent agents from drifting from their original tasks.

Developers also need to be mindful about leaving credentials lying there for attackers to steal. Both Azure and AWS provide strong RBAC controls that can be used over permissive connection strings and API keys. This is also a good reminder to consider how agents use tools that are available to them. In the case of goal drift, agents using legitimate tools available to them won't be flagged in retrospect. They're allowed to use the tools that you configure, so think about the design of your agents and wether or not the tools you give them are too permissive.

- [CERT-In advisory CIAD-2026-0049](https://www.cert-in.org.in/s2cMainServlet?VLCODE=CIAD-2026-0049&pageid=PUBVLNOTES02)
- [CERT-In Warns AI Agents Accessed Sensitive Data at Indian Organizations During Testing](https://thehackernews.com/2026/10/cert-in-warns-ai-agents-accessed.html)
- [India's Cyber Agency Says AI Agents Read Confidential Business Records Without Authorization](https://decrypt.co/381165/indias-cyber-agency-ai-agents-read-confidential-business-records)

## Langflow MCP stdio command injection now exploited (CVE-2026-105740)

Langflow is a tool for building and deploying AI agents and workflows. Prior to version 1.9.0, any authenticated Langflow user could perform Remote Code Execution (RCE) on the server by adding an MCP server using stdio transport. This command could be supplied to the user by passing `bash -c "exec {command}"` with zero validation, no allowlisting, and no sandboxing. This command would execute immediately when the server list is fetched.

A bad actor could exploit this by logging in as a regular non adminuser, and add a malicious MCP server with a command field containing a malicious payload. The server would then accept the server and store the attacker-supplied command without validating it, which the attacker would then trigger by fetching the MCP server and achieve code execution.

Previdian have found that 1,423 exploitation attempts have been recorded since September 19th from 27 unique IP addresses located in Belgium, China, Germany, Hong Kong, Japan, Nigeria, the Philippines, Russia, Seychelles, Singapore, South Korea, Taiwan, Tanzania, the U.S., and Vietnam. In one observed attack, attackers checked the Langflow version, attempted to auto-login, then immediately use the resulting bearer token to access the vulnerable MCP server configuration endpoint. Some of the captured payloads included attempts to extract environment variables and sensitive Langflow configurations, Docker commands attempting to access the host filesystem and read password file contents `/etc/passwd`, and command execution probes using id, whoami, and other commands.

Mapping this back to OWASP, a configuration string flaw in Langflow has become a arbitary shell exection on the agent platform's host. The playload planted as an MCP server definition also is a threat to the agent's supply chain. Exploitation also depends on the use of weak or default credentials, which leads to the theft of platform secrets and superuser access.

- [CVE-2026-105740: langflow Authenticated RCE Remote Code Execution](https://previdian.com/CVE-2026-105740)
- [Langflow's Critical Authenticated RCE Flaw Targeted in 1,423 Honeypot Exploit Attempts](https://thehackernews.com/2026/10/langflows-critical-authenticated-rce.html)

## Aoki infostealer adds AI coding-agent credentials

On October 15th, the folks at Symantec reported that the Aoki infostealer is now targeting AI coding-tool credentials. This includes Claude's `.claude.json` and desktop cookies, Codex `config.toml` and `.env` files, Cursor's local state database, tokens for GitHub Copilot, Gemini, Codeium, JetBrains AI Assistant and Kiro, and ChatGPT cookies and AWS Credentials.

An infostealer is a type of malicious software designed to covertly harvest sensitive data from a victim's computer or device without thier knowledge. Once an infostealer infects a machine, it scans the system for valuable data, bundles it into an archive, and then exfiltrates it to a command-and-control server controlled by the attacker.

If an attacker is able to steal credentials from their victims, they'd be able to carry out activity under thier legitimate user's identity and API quota. That includes access to linked repositories, CI/CD pipelines and paid API quota. This is just as valuable as payment card data, as it can be cashed in on directly, and provider the attacker a route to the developer's environment.

Symantec doesn't know who is behind the attack, nor can they link it to any existing malware. However the nature of the attack indicates that this is a financially motivated attack.

There's no patch for this vulnerability, so the important thing to note here is that you need to treat your agent configuration files as secrets. The potential for abusing agent credentials is huge, and any stolen agent token gives attackers a foothold in your repositories and pipeline that your agents feed. 

- [Aoki: Infostealer Targets AI Coding Assistant Credentials](https://www.security.com/threat-intelligence/aoki-infostealer-ai-targeting)
- [New Aoki Infostealer Targets AI Coding Assistant Credentials](https://www.infosecurity-magazine.com/news/aoki-infostealer-ai-coding/)
- [New Aoki Malware Targets ChatGPT, Claude, Codex and Gemini to Steal Credentials](https://cybersecuritynews.com/new-aoki-malware/)

## OpenCode Console breach exposes stored LLM provider API keys

On October 16th, attackers managed to gain access to a legacy OpenCode console system that was scheduled to be shut down and gain access to third-party API keys in affected workspaces. Affected users where limited to some Console users who had connected external providers.

*How old does something have to be to be called legacy in the age of AI?*

OpenCode have emailed affected users and later confirmed the breach on X.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/08rq5s8u7bz7y6u7kdsx.png)

OpenCode have said that the breach was connected to a large-scale distilation attack, however it hasn't provided any technical evidence to how it was linked. Distilation can involve using the outputs of one AI model to train or improve another model. OpenCode hasn't said whether any exposed credentials were used to generate such requests, and it has yet to identify the attacker.

If agents were used here for the attack, stolen keys that are used to call LLMs are already being exploited (with some unfortunate victims receiving hefty AI-spend bills!)

- [OpenCode Breach Exposes Third-Party API Keys of 1.4% of Console Users](https://the420.in/opencode-security-breach-api-keys-console-users/)

## Wrap up

It's easy to think that agents are running wild, but if you think about it rationally, agents act with the tools and permissions we give them. They're also enabled by our bad habits and bad developer practices. If you leave connection strings and API keys lying around for agents to discover, you're asking for trouble.

The platforms that we build agents on are also a target too. In the case of Langflow, that CVE went from published to being actively exploited all through a single MCP server definition. Treat any MCP server configuration as a mechanism for executable code.

If you're building agents for your organization, or using coding agents in your developments, you need to treat the credentials that they use like any other production secret. Keep them out of code and configuration files, ensure they are scoped appropriately, and rotate them often (better to not use API keys at all if you can).

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️