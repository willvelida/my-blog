---
title: "Last week in Agent Security 1: We are not-a-mused!"
date: 2026-09-29
draft: false
tags: ["OWASP", "Agents", "AI", "Security"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/7sq1qts9agjrx3erdgi6.png
    alt: "Running multiple GitHub Copilot agents in parallel with tmux"
    caption: "Muse AI exploits on macOS, OpenAI breaches Medicare, and Gambit finds a sophisticated multi-agent hack that's still ongoing!"
---

Last week I talked at NDC Oslo about the OWASP Top 10 for Agents where I showed folks the different ways that agents can be exploited. If you've turned the news on recently, you've probably heard about AI will kill us all and how we're all doomed.

Call me optimistic, but I think we're currently in the middle of a hype cycle where AI labs are overstating the abilities of the models, and we're at a point where we've seen how powerful agents can be, but we need to be more serious about the harms agents can do, and how we can mitigate those risks.

With that in mind, I'll aiming to do a weekly blog post on different agentic attacks and incidents, what happened, how it happened, and what OWASP risks they fall under. Depending on demand or the severity/publicity of the attack, I may do a deep dive on how you *could* prevent such an attack in the first place.

I'll point out that I'm not trying to shame any of the companies involved, nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here's some incidents that have happened over the period 21st-27th September 2026.

## Meta Muse macOS zero-day ("not-a-mused")

On September 21st 2026, macOS security researcher [Patrick Wardle](https://x.com/patrickwardle?lang=en) (also founder of the Objective-See Foundation) created a POC (amusingly called "not-a-mused") showing that malware already running inside a Mac can hijack Meta's Muse AI assistant.

The flaw centers on a undocumented configuration setting `endo_voyager_dictation_endpoint` that an unpriviledged local process can flip to reroute Muse's dictation traffic to an attacker's controlled server (via your own microphone). From there, you could inject extra instructions that Muse trusts, read dictated content from Muse, and steal the authentication token that signs into the user's Muse account. You can use Muse on multiple devices, which extends the blast radius.

Meta have apparently released a hotfix, but we see a number of risks being highlighted here. Theft and reuse of the agent's token (Identity & Privilege Abuse), injected goal hijack of the instructions that the agent acts on (Agent Goal Hijack), and how a trusted assistant becomes an attack surface (Human-Agent Trust Exploitation).

- [Wardle's X thread on Muse](https://x.com/patrickwardle/status/2102045926474785265)
- [One Hidden Meta Muse Setting Could Let Attackers Turn the AI Assistant Into a Backdoor](https://thehackernews.com/2026/09/one-hidden-meta-muse-setting-could-let.html)
- [Meta Muse Hits No. 1 on App Store, Then a Mac Zero-Day Hijacks Its Permissions](https://www.gpts24.com/en/news/meta-muse-hits-no-1-on-app-store-then-a-mac-zero-day-hijacks-its-permissions)

## Gambit "Strix/Cairn/Hermes" autonomous card-skimming campaign

Gambit Security published an interim report about a financially motivated campaign in which a single operator chained three open-source AI agent frameworks to attack online retailers autonomously. They've traced the attack to July 2026 and it's still ongoing (at time of writing).

The first framework involved is [Strix](https://www.strix.ai/), which is an open source AI penetration testing tool. Between 23rd and 31st August, Strix was run in deep mode against various hosts to provide reconnaissance and vulnerability discovery.

[Cairn](https://cairnsecurity.com/), which is an automated penetration testing engine, was then used to launch attacks. Each attack path was chosen by the harness in real time through extensive probing and exploitation attempts, resulting in dynamic and mostly different TTPs across victims.

Then finally, [Hermes](https://hermes-agent.nousresearch.com/), is being used to orchestrate the attack. Gambit attributes a Chinese-speaking operator running a `SOUL - Red Team Operator` persona with 121 skills. The operator has also added a skill that removes the content security filters of Hermes itself. It used Opus 4.6 (after newer models had refused its requests), with just under 2,000 prompts typed by the human operator across 260 sessions. Some example prompts include:

- 看漏洞报告 开干 (“read the vulnerability report and start”)
- 看看进web后台 (“get into the web backend”)
- 你去搜一下wp2shell (“go and search for wp2shell”)

So far, more than 600,000 valid credit-card records have been exfiltrated from two victims, with the vast majority being US records, card-skimmer scripts planted on the sites of five organisations, and 119+ compromised websites overall; victims include a Fortune 500 hospitality company, a major US airline, a large US industrial-supplies distributor and an online fashion retailer. One Hermes "skill" file instructed the agent to wipe card fields from victims' Magento databases after exfiltration, causing operational disruption.

So another example of Rogue Agents in action, but this time the agents belong to the attacker, rather than a defender's own agent being subverted.

- [Autonomous AI Agents are breaking into hundreds of Online Retailers for $25 a target in an ongoing campaign](https://gambit.security/blog-posts/autonomous-ai-agents-online-retailers-25-a-company)

## Bifrost AI-gateway unauthenticated RCE

AI Gateways are an effective mechanism to protect calls to LLM models, but they aren't foolproof. [CVE-2026-90898](https://www.cve.org/CVERecord?id=CVE-2026-90898) was reported in open-source AI gateway Bifrost which allows unauthenticated attackers to run arbitary commands on the gateway server with a single HTTP request. This is a good example of ASI05, Unexpected Code Execution.

Yuval Moravchick of JFrog Security Research found that an attacker can register a stdio-type MCP client through a single unauthenticated POST request to the management API endpoint `/api/mcp/client`. Before any MCP handshake, Bifrost starts the specified command immediately as the gateway process user.

The Bifrost binary binds the management API to localhost by default, which limits exposure to the local machine. The official Docker image binds it to 0.0.0.0, making the management API reachable from outside the container if the port is published.

Upgrading to transport versions v2.1.0 will result in a 403 response when unauthenticated callers try to register a stdio MCP client. Now the CTO and co-founder of Maxim says that the risk is lower than the CVE rating suggests, since exploitation requires the management interface to be reachable from an untrusted network and the operator to have left authentication unset. The default and documented deployment places the gateway inside a private network. Maxim and JFrog are working together to revise the severity assessment.

- [Critical Bifrost AI Gateway Flaw Lets Attackers Run Commands Without Credentials](https://thehackernews.com/2026/09/critical-bifrost-ai-gateway-flaw-lets.html)

## Carbonato Docker botnet built on the Hermes "GH0ST" agent

On September 23rd, The researchers at ThreatDown disclosed a new botnet malware called Carbonato, which targets insecure hosts running Docker daemons exposing an unauthenticated daemon API on port 2375 to launch a privileged container to gain host access, and then install the Hermes Agent AI framework running an agent persona named GHOST with instructions that overwrite the default `SOUL.md` persona file.

![](https://www.bleepstatic.com/images/news/u/1220909/2026/September/agent.jpg)

It then opens a reverse SSH tunnel, installs an SSH server with the operators' key, and reports the new deployment through Telegram. Operators issue tasks over a Telegram interface, which is then interpreted by the model, and writes terminal commands, reads the output and decides to do what next. These tasks prioritize tasks such as harvesting AI API keys, SSH credentials, access tokens etc. and then returns those via Telegram.

This is an example of a Rogue Agent (ASI10) being weaponized as the malware core combined with tool misuse and exploitation (ASI02). The recommendation here is to keep Docker daemon APIs off the network and require authentication on registries.

- [New Carbonato malware uses AI agents to hijack exposed Docker hosts](https://www.bleepingcomputer.com/news/security/new-carbonato-malware-uses-ai-agents-to-hijack-exposed-docker-hosts/)

## OpenAI agent breached Australia's Medicare statistics portal

In local news, OpenAI breached Medicare!

So back in June, An OpenAI agent breached a Medicare statistics reporting portal operated by Services Australia, accessing public and non-public files and writing data to an internal server.

The agent was running a research task on public medical-spending data and when the portal's controls refused its requests, the agent found a workaround. The portal holds aggregated Medicare/Pharmaceutical Benefits Scheme statistics and is separate from claims and personal-record systems. 

While no evidence that individual medical records were accessed, it didn't help that OpenAI found out about the attempt in June, and only told the Australian government last week. Despite the outrage from Australian media and general public, this isn't the worst example of Rogue Agent attacks. OpenAI should have acted quicker. This also shows how agents can be hijacked through goal drift, and how agents can bypass access controls to reach non-public files.

The Australian Government has stood up a cross-government task force, so watch this space.

- [OpenAI hacked Australian Medicare govt site, probed data providers](https://www.bleepingcomputer.com/news/security/openai-hacked-australian-medicare-govt-site-probed-data-providers/amp/)

## Salesforce Agentforce "SalesBleed" 0-click data exfiltration

On 24 September 2026, Zenity Labs disclosed SalesBleed, a set of zero-click vulnerabilities in Salesforce Agentforce that allowed attackers to exfiltrate CRM data with no victim interaction and without authenticating into the target's Salesforce environment.

An attacker plants a hidden prompt-injection payload inside a public-facing Web-to-Lead form. The malicious lead persists in CRM as a dormant record, and when an Agentforce agent later processes it during normal operations, the injected instructions hijack the agent.

The payload then queries and exfiltrates sensitive account data using DNS-based exfiltration that evaded Salesforce's Trusted URLs redaction control. So any external party who submits a lead could silently exfiltrate CRM data.

Zenity Labs and Salesforce have worked together to fix the bugs in Agentforce, but Zenity were quick to stress that the vulnerabilities found aren't unique to Agentforce. Any agent that ingests untrusted external records, renders links back to users, and holds sensitive tool access can be hijacked in the same way.

Memory and Context Poisoning through poisoned CRM records that resurface as trusted context (ASI06), indirect prompt injection redirecting agent behavior through those poisoned records (ASI01), and the agent's tool becoming the blast radius (ASI03) can occur in anyone's agent.

- [Zero-Click Vulnerabilities in Salesforce Agentforce Expose Wider AI Agent Risk](https://www.infosecurity-magazine.com/news/vulnerabilities-salesforce-ai/)
- [Salesforce Indirect Prompt Injection Vulnerability Enables 0-click Data Exfiltration](https://cybersecuritynews.com/salesforce-salesbleed-vulnerability/)

## Conclusion

As you can probably see, these incidents came from agents that could make real tool calls, use credentials, and had the autonomy to perform consequential actions. Some agents were developed by the attackers themselves, while others drifted from their intended goals.

That's why it's critical for developers to control **what** your agent can do, not just what you tell them do. You have to assume that prompt injection attacks will eventually prevail, so you can design your agents to stop the damage early when it does.

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️

