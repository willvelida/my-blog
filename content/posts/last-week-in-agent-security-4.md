---
title: "Last week in Agent Security Update 4: Cairn’t Touch This"
date: 2026-10-26
draft: false
tags: ["OWASP", "Agents", "AI", "Security"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/pp1zhcf6vjig5kedb69o.png
    alt: "Last week in Agent Security Update 4: Cairn’t Touch This"
    caption: "Open-source pentest agents, now available to any criminal near you. Welcome to the era of the open-source attack agent."
---

What a fucking week....

If you've been living under a rock this week, [two critical RCE vulnerabilities were found in PHP, collectively known as PHP2Shell](https://www.wiz.io/blog/php2shell-php-rce). This vulnerability allows unauthenticated attackers to remotely execute code on vulnerable web servers without logging in or providing credentials, potentially seizing server control or stealing data.

These vulnerabilities affected all PHP versions released over the last decade, spanning back to version 5.4. This was particularly scary because according to Wiz, PHP powers 70% of the world's websites, including banking and government applications.

Many websites and pages were taken offline as a precaution, including those of the Finnish government, The Netherlands' cyber-security agency, and Wikimedia (the organization behind Wikipedia).

Researchers at Calif and Wiz used models from OpenAI to find the tools, so AI was pretty useful in finding the flaw. The problem was that the [vulnerability was leaked](https://thehackernews.com/2026/10/someone-tried-to-exploit-php2shell-4.html) before the embargo was lifted, so the PHP platform developers were forced to rush out the disclosure of the vulnerability as well as the fix. With attackers being able to use agents to operate at machine speed, this only highlights the dangerous world we now live in.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/zgnn70mquvh3wax32sbp.png)

So far, no breaches have been reported. I'm holding my breath on this one, as we have seen in the past attackers being able to breach a system, lie dormant for months, and then strike. Keep your eyes open on this one.

In more positive news, While I was at NDC Oslo, Richard Campbell invited me on [RunAs Radio](https://runasradio.com/) to talk about the OWASP Top 10 for Agentic Applications, and how the different layers of agentic architecture can be compromised. You can check out the podcast [here](https://runasradio.com/Shows/Show/1059).

# This week's attacks

This week we focus less on misalignment from frontier models and more on attackers who are using agents to drive more targeted attacks using open-source tools. We also take a look at how agentic supply chain vulnerabilities have the potential to cause harm on a wide scale, and finally we make fun of Meta because I hate them.

As always, I’ll point out that I’m not trying to shame any of the companies involved (maybe just the ones who deserve it), nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here’s some incidents that have happened over the period 19th October to 25th October 2026.

## Cairn agents break into 100+ company websites

NorForge found more than 20 publicly exposed control panels for Cairn, which is an open-source AI pentest system, that criminals were using to point autonomous agents at e-commerce, payment and pharmaceutical targets. These agents tried more than 1200 sites and reached websites belonging to more than 100 companies, including Deere, SharkNinja, and Cummins.

The operator provided short instructions to agents to attempt code execution and plant web-shell uploaders. One documentation chain combined unauthorised write access to a memcached session store, unsafe PHP object deserialization (*oh PHP*) and old Zend Framework code. NorForge haven't confirmed which models were used in the attack.

NorForge estimates that 10% of attacks were successful. One large US retailer was independently confirmed that they were compromised, and one shell URL serving a uploader shell has been successful. The volume of data that has been taken is currently unknown. The panels also leaked secrets and some legitimate pentest firms have had client data exposed on those open panels.

![AIA0001 target selection by sector, showing retail and wholesale as the largest target group.](https://norforge.com/blog/inside_cairn/images/aia0001-target-selection.webp)

*Source: [NorForge — Inside Cairn: Open Windows into massive AI-Driven Cyber Operations](https://norforge.com/blog/inside_cairn).*

A couple of weeks ago, I described how [Cairn](https://www.willvelida.com/posts/last-week-in-agent-security-1/#gambit-strixcairnhermes-autonomous-card-skimming-campaign) was being used in a multi-agentic attack targeting retail websites in the US. We're going to see more sophisticated rogue agent attacks using specialized frameworks to perform malicious, specific attacks on their targets, utilizing these specialized tools to successfully breach them with minimal effort from the human operator. The maker of Cairn told Reuters that the tools was not designed for hacking attacks, but we've already seen multiple instances where agents can use legitimate tools to do harm, and how attackers can exploit them to do damage.

- [Inside Cairn: Open Windows into massive AI-Driven Cyber Operations - NorForge blog](https://norforge.com/blog/inside_cairn)
- [Hackers Use Autonomous AI Agents to Breach Company Websites, Target Over 1,200 Sites - The Hacker News](https://thehackernews.com/2026/10/hackers-use-autonomous-ai-agents-to.html)

## Storm-0940 uses ARTEX agents in ConnectWise supply-chain campaign

It's not just Cairn that's been exploited by bad actors. The folks in Microsoft Security have attributed Chinese state-aligned actor dubbed Storm-0940 was behind the recent [supply chain compromise](https://thehackernews.com/2026/10/connectwise-resets-credentials-as-cloud.html) of the ConnectWise Cloud Backup application.

ARTEX, an open-source multi-agent LLM pentest framework, was used to automate the operation from reconnaissance through to exfiltration. The actor abused stolen ConnectWise multi-tenant application identities and the trust that customers had already granted to those identities. It was also able to run password spray attacks against Microsoft 365, Exchange Online, Okta, and Entra ID.

These ARTEX-driven attempts reached more than 2,250 tenants and peaks of up to 2,900 attempts per minute. Between 10th-14th October, the spray generated more than 151,000 failed logins. After the 15th, agents were able to reach Microsoft Graph and were able to change identities, groups, roles, and app credentials. In several cases, they were able to gain privileged access in under 3 hours. 

![Illustration accompanying The Hacker News report on China-linked hackers using ARTEX AI agents.](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEihyvW6Vi5XS9EzEem8mWZ6I1vdQwHj9pQWyIdJMAgk2HxEegqxF3tCKtOw7REdJ_V27mKU6pm6LbGcSKl5zkvy3wdEoNS3hbdGj9LnNXTuuR-g5cWisW5tp7YTHPJTJWEkyl2iDQAF2oAA9IU7tlw7d5n8hNuHwlSq49zCctuAkuI-omxB8NMByUzipUnd/s1700-nu-rw-lo-l85-e365/zz.jpg)

*Source: [The Hacker News — China-Linked Hackers Use ARTEX AI Agents in ConnectWise Supply Chain Attack](https://thehackernews.com/2026/10/china-linked-hackers-use-artex-ai.html).*

Looking at the prompt histories, it appears that the motive is financial, and Wiz have reported that the campaign has expanded to other software vendors. ConnectWise has responded by shutting down services and restoring them, and Microsoft Security have published a [list of immediate actions and mitigations](https://www.microsoft.com/en-us/security/blog/2026/10/21/storm-0940-uses-artex-agentic-ai-connectwise-supply-chain-compromise/) that affected customers should take.

This attack managed to abuse OAuth app identities and delegated trust across multiple tenants at machine speed, using ARTEX as the multi-agentic attack vector, exploiting the tools that the framework provides to repurpose them for an offensive campaign.

- [AI-driven, large-scale nation-state campaign using ARTEX enables supply chain compromise - Microsoft Security Research](https://www.microsoft.com/en-us/security/blog/2026/10/21/storm-0940-uses-artex-agentic-ai-connectwise-supply-chain-compromise/)
- [China-Linked Hackers Use ARTEX AI Agents in ConnectWise Supply Chain Attack - The Hacker News](https://thehackernews.com/2026/10/china-linked-hackers-use-artex-ai.html)

## INC / The Gentlemen ransomware affiliates' "Phantom" AI kill chain

In more examples of rogue agents and tool misuse and exploitation, the folks at SOCRadar traced an exposed command-and-control server, and six more adversary servers, to Russian-speaking operators. This was an AI-driven intrusion ecosystem run by affiliates of the INC Ransomware and The Gentlemen Ransomware groups. The operation runs on an AI-executed, human-approved model, where a human writes a victim-specific prompt and approves each step, while commercial models that are accessed through OpenRouter carry out intrusions via a OpenCode coding agent.

The toolkit includes a tool called "Phantom", which is an autonomous red-team framework, and "WormGPT-MCP", a set of MCP servers that cover reconnaissance through extortion. When a model would refuse the request, the operator would reframe it as an authorized red-team engagement, and the model would continue.

![Tooling overview of an AI-executed operating structure associated with INC and The Gentlemen ransomware affiliates.](https://socradar.io/wp-content/uploads/2026/10/tooling-overview-ai-executed-operating-structure.jpg.webp)

*Source: [SOCRadar — INC / The Gentlemen ransomware AI infrastructure](https://socradar.io/blog/inc-gentlemen-ransomware-ai-infrastructure/).*

62 names organizations have been listed as victims, with 36 in the United States. The operators gained access by exploiting public-facing edge devices, and then moved towards ransomware deployments. This was followed by credential reuse, lateral movement, and data exfiltration.

- [INC and The Gentlemen Ransomware: Exposed AI Infrastructure - SOCRadar](https://socradar.io/blog/inc-gentlemen-ransomware-ai-infrastructure/)

## Autonomous LLMjacking agent captured by Symantec

Symantec's Threat Hunter team have discovered the instruction set of a self-directed offensive agent that hunts for exposed LLM endpoints, AI gateways, and MCP servers to run their own work through them or through stolen keys, so that victims pay for the compute.

The agent was designed to use the hijacked AI models to autonomously run up to 100,000 malicious actions without human approval, including scanning for targets, exploiting vulnerabilities, stealing credentials, deploying cryptocurrency miners and reinfecting compromised systems. Attackers used layered prompts to bypass commercial model safety controls.

The honeypoint logged more than 127,000 requests from tens of thousands of addresses. This telemetry included reconnaissance of model endpoints, spraying of stolen API keys, and probing for exposed configuration files. What makes this attack disturbing is that instead of attackers trying to trick a public chatbot into misbehaving, the operators built AI into the core of an offensive tool and attempted to hijack other people's models to power it.

- [Autonomous LLMjacking: ‘Skynet’ Agent Hijacks Exposed AI Models to Run Attacks - Security.com](https://www.security.com/blog-post/ai-llmjacking-skynet)
- [Symantec Uncovers AI Agent Hijacking Models for Autonomous Cyber-Attacks - Infosecurity magazine](https://www.infosecurity-magazine.com/news/symantec-ai-agent-llmjacking/)

## LiteLLM 1.107.0 supply-chain compromise via leaked Agent Platform key

Turning towards vulnerabilities on the supply chain, a malicious version fo LiteLLM `litelmm==1.107.0` was published to PyPI, Docker Hub, and GHCR images. This was caused by a master key for its internal Agent Platform that was exposed in a `opencode.json` file that was committed to a public repository back on May 27th (*We've all been there before*).

The commit was force-pushed over, but the key wasn't actually revoked. Attackers were able to use the key against the Agent Platform API, which returned a GitHub CLI OAuth token belonging to an organization owner. From there, the attackers were able to steal maintainer tokens, push secret-harvesting workflows, and repoint releases to lookalike repositories.

![Image accompanying The Hacker News report on a leaked LiteLLM master key.](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgxQfhCGuRZSHUo3sGckM9x3uvri__keVPtpF7jfOcKP-VPZnk_7p15ZxWr_N-WMn2ZCzs-VToqNjhTtHKyLBIBXzgWDLvAowr3VLhKxX75P-7E4skEXvq14_XDT4s-wkyinfA7OI7QxOyU7KrTmpJaWLNy4f5fMLzV_9xJIftNrV8dvct2MpBoDPteqDRg/s1700-nu-rw-lo-l85-e365/litellm1.1.jpg)

*Source: [The Hacker News — Report on a leaked LiteLLM master key](https://thehackernews.com/2026/10/leaked-litellm-master-key-let-attackers.html).*

The malicious version runs a credential stealer at start up that takes information like SSH keys, cloud tokens, git tokens, browser data, AI tool credentials, and environment variables, and sends them to `litellm[.]dev`. It also installs a Unix backdoor that polls the code hourly for new code.

The release was quarantined and affected images were removed. Due to the breach, LiteLLM are undertaking the process of revoking internal and vendor tokens. LiteLLM are advising users to rotate credentials from a clean machine and pin 1.106.2 by digest.

Thanks to this supply chain vulnerability, a widely used AI gateway package was able to be trojanised, putting every downstream agent stack that had the affected version installed at risk. The unrevoked agent-platform master key returned an over-scoped owner token, which gave the attackers a wide scope to perform authorized actions for malicious intent.

- [The attack chain behind the v1.107.0 LiteLLM release - LiteLLM](https://docs.litellm.ai/blog/security-incident-oct-21-2026-attack-chain)
- [Leaked LiteLLM Master Key Let Attackers Steal GitHub Tokens and Hijack Release Pipeline - The Hacker News](https://thehackernews.com/2026/10/leaked-litellm-master-key-let-attackers.html)

## OpenCode RCE via malicious MCP server during OAuth sign-in

The folks at Pillar Security have discovered a critical RCE (remote-code execution) flaw in OpenCode that affects Windows and Windows Subsystem for Linux (WSL). This flaw allows an attacker-controlled remote MCP server to execute arbitrary code on a developer's machine during the remote MCP OAuth authentication process, without requiring additional user interaction or consent.

MCP servers connects agents to external tools and services. This vulnerability highlights that connecting coding agents to compromised or malicious remote MCP servers can expose a developer's machine to remote code execution. It's vital that developers and security teams assess the trustworthiness of connected MCP servers and ensure that applications that handle externally supplied URLs do not pass untrusted input into shell commands.

In this case, Anomaly (the company behind OpenCode) have claimed that since they believe that users who configure MCP servers for their agent are "outside our trust boundary", no CVE has been created and no advisory has been published. This means that this vulnerability won't show up in vulnerability scanners or dependency alerts, so the responsibility is on developers and security teams to check the OpenCode version directly.

Pretty poor response from Anomaly in my opinion.

- [Open and Shut: Remote Code Execution in opencode via MCP OAuth: Pillar Security](https://www.pillar.security/blog/open-and-shut-remote-code-execution-in-opencode-via-mcp-oauth)

## Planted AGENTS.md in ~/Downloads hijacks coding agents

Markdown is the hottest new language! It's also the hot new attack vector! The folks at Forever Security found a malicious `AGENTS.md` file that was downloaded into a users `~/Downloads` folder, and discovered that it tried to make Codex install a malicious npm package.

`AGENTS.md` files are files that coding agents are built to trust and obey, so you can imagine that if an attacker than supply a `AGENTS.md` file that gets downloaded onto a users computer that direct the agent to perform malicious activity such as installing malware or handing over login information, that's a pretty powerful attack! Coding agents are getting more capable in their functionality, including having the ability to open up browsers, or even being able to take over control of the computer. This type of attack starts from a simple Agent Goal Hijack using a malicious `AGENTS.md` file, and can lead to other risks like supply chain vulnerabilities, unexpected code execution, and identity abuses.

That's a pretty big design flaw. There's no patch for this, so the way that we can mitigate that is to only run agents from project directories, review instruction files before trusting them, keeping approvals in the hands of humans, and applying sandboxes to our agents.

- [Markdown Is the New Executable: Attackers Are Dropping Agent Instructions Into Your Downloads Folder From the Web: Forever Security](https://forever.security/blog/hijacking-ai-agents-via-downloads-folder)
- [AI coding agents hijacked using AGENTS.md file in Downloads folder: BleepingComputer](https://www.bleepingcomputer.com/news/security/ai-coding-agents-hijacked-using-agentsmd-file-in-downloads-folder/)

## Meta Muse "GhostShares"

Oh Meta....

In Meta's Muse AI assistant, you can share conversations. These conversations are published as public pages that anyone with the link can open. (That's by design, which doesn't surprise me considering it's Meta. *Why* it's by design after all the bad publicity that Meta have received due to their lax attitude toward privacy shouldn't be a surprise to anyone).

However, the folks at Claroc's research team found that these links were pretty easy to find, because what Muse names each share's web address after the conversation's own title, and that address is written into public DNS. One passive-DNS search brings back thousands of them, listed by name, like a table of contents.

*To be fair, this isn't a new type of leak. This has also happened to both ChatGPT and Grok conversations*

![Flow diagram accompanying Claroc’s GhostShares article.](https://blog.claroc.io/img/ghostshares-flow.png)

*Source: [Claroc — GhostShares article](https://blog.claroc.io/ghostshares-muse-passive-dns/).*

This type of leak can lead to some pretty dangerous situations. One conversation included a candidate's notes preparing for a confidential executive-search call with a frontier AI lab, which included naming the lab's internal contact and the meeting time. Another included a to-do list that included a bank fraud alert, with partial account number and transaction information.

Why anyone trusts Meta with anything is a mystery to me. Let alone one of their AI agents.

- [GhostShares: A Shared Muse Chat Exposed Prep for a Confidential Call With a Frontier AI Lab Leader: Claroc Research](https://blog.claroc.io/ghostshares-muse-passive-dns/)
- [Meta Muse exposes users’ personal and business secrets when they share conversations: ArsTechnica](https://arstechnica.com/security/2026/10/meta-muse-exposes-users-personal-and-business-secrets-when-they-share-conversations/)

## Wrapping up

We're seeing an increase in malicious actors using agentic-driven attacks to cause harm. Both incidents that used Cairn and ARTEX in their agentic-driven attack used tools that they were authorized to use, and they were designed by the developers of both those tools to automate pentest tasks. It's just that they were used by malicious actors to perform those tasks at machine speed for malicious purposes.

Hopefully we'll see an increase in watch dog agent capabilities to prevent or limit attacks from these rogue agents. It's difficult to stop the bad guys using open-source frameworks to do harm, but there's a lot of work to be done to make it easier for folks to protect their applications and themselves from harm (*Probably not a bad startup idea in there...*).

With these types of attacks being able to be executed at machine speed, more focus needs to be paid to identity, and how over scoped credentials that are either stolen or mishandled have the potential to do some real harm.

This also wasn't a great week for vendors. I hope that Anomaly come to the table and at least issue an advisory. This helps vulnerability and dependency scanners pick up issues in code, and help make everyone a little safer. Until then, security teams and developers need to be extra vigilant in what models, gateways etc. they are using, so they can prevent vulnerabilities causing harm.

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️