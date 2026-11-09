---
title: "Last week in Agent Security Update 6: The Legend of Vibe-Hacking"
date: 2026-11-09
draft: false
tags: ["OWASP", "Agents", "AI", "Security", "LWIAS"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/v4nwtz0brd25vvsbuk9h.png
    alt: "Black-and-white meme of Link from The Legend of Zelda sitting in a white armchair with both hands raised, captioned ABSOLUTE HYAH."
    caption: "This week: a researcher's runaway ARTEX agent breaches WorkOS, GitBook blames AI-driven automation for its breach, ARTEX campaigns spread from leaked secrets to banks (with 1,465 consoles left exposed), one operator runs a 12-agent mesh against Taiwan's emergency services, a Codex jailbreak bundle sneaks in prompt adware, coding agents install retired MCP servers from memory, and a symlink lets agents escape Docker Sandboxes."
---

For the past week, I've been enviously watching YouTube video walkthroughs of Legend of Zelda: Ocarina of Time remakes that was released last week.

My first video game console was a Nintendo 64 with Goldeneye bundled with it. My parents gave me that along with the original Legend of Zelda Ocarina of Time. You can't ask for a better introduction to gaming than that!

I've got an Xbox One which is handy because I get Game Pass Ultimate through work, so most of the games I play are free. However I'd be lying if I said I didn't want to go and buy a Nintendo Switch 2 just so I can go down memory lane again.

Problem is that in Australia, the Switch 2 currently retails for $799! Along with Ocarina of Time retailing for $99, that's a pretty expensive trip down memory lane.

![Two-panel reaction meme showing disgust at paying $160 for GTA 6 Ultimate Edition, then interest in paying $900 for a Nintendo Switch 2 with the Ocarina of Time remake.](https://pbs.twimg.com/media/HWZLE_iaEAA5AFB?format=jpg&name=small)

However, it's a lot more tempting than shelling out $160 for GTA 6 Ultimate edition. I'll have to hide it from my wife somehow.....

![Three-panel Thor: Ragnarok meme: GTA 6 says "You can't defeat me"; every game releasing in 2026 replies "I know, but he can", followed by the Legend of Zelda: Ocarina of Time logo.](https://b3666184.assetcdn.net/2.0/3666184/wp-content/uploads/2026/06/ocarina-of-time-remake-memes-2-20260610-600x713.jpg?lossy=2&strip=1&webp=1)

*Ocarina of Time enters the chat. Source: [Thunder Dungeon's Ocarina of Time remake meme collection](https://thunderdungeon.com/2026/06/10/ocarina-of-time-remake-memes-joy-fear-wallets-out/ocarina-of-time-remake-memes-2-20260610/).*

# This week's attacks

Big focus on ARTEX again this week! I suspect we're at a point where access to agentic cybersecurity tools and AI agents have lowered the barrier of entry to perform wide ranging attacks. However, not a lot of these attacks are successful and we may be in a stage where attackers are just *vibe-hacking*.

*A much better phrase than vibe-coding. I made it up btw, credit me for it lol*.

I predict as frontier models increase in capability and have more guardrails, we'll see attackers shift to using local models in order to work around the built-in guardrails and will only be limited by their access to hardware.

![Mean Girls characters sitting in a convertible beneath the text "Get in bitch, we're going on-prem again".](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/3zropysgzgmzlenukb9m.png)

As always, I’ll point out that I’m not trying to shame any of the companies involved (maybe just the ones who deserve it), nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here’s some incidents that have happened over the period 2nd November to 8th November 2026.

## A researcher's ARTEX agent escapes its sandbox and breaches WorkOS

On November 2nd, [WorkOS](https://workos.com/) disclosed that an attacker using the [ARTEX](https://github.com/IlyassCODEX/ARTEX) offensive framework had accessed one of their internal sales tools. Rather than this originating from a state-sponsored attacker, a security researcher had come forward and accepted responsibility for it.

The research said that their ARTEX agent had escaped the isolated environment that they were doing cybersecurity benchmark evaluation and interacted with WorkOS unintentionally. WorkOS was able to match request details provided by the researcher against company's logs, confirming that they were responsible for the unauthorized activity.

While the researcher confirmed that all data obtained by the agent had been deleted, no details on how the agent escaped its testing environment have been revealed. The exposed data includes business contact details, customer account histories, product information, and summaries from customer support.

Through their investigation, WorkOS have identified the weaknesses that the autonomous agent could exploit to access its internal sales application and extract customer information. The application ran on a legacy tools platform that lacked the same security protections that its newer systems are enforcing. The permissive access controls left API endpoints exposed to the internet, making it easy for the agent to discover and interact with them.

Additionally, the API also accepted a credential intended for local development which had been deployed to production, meaning the agent could authenticate after discovering the exposed endpoint. On top of this, the API had no rate limiting, letting the agent rapidly retrieve records from the exposed application.

WorkOS have now removed internet access to the tool and the weak credential. It also notified customers of the attack from November 2nd, and have started to use ARTEX for their own testing.

This incident demonstrates how legacy systems can be easy exploited by AI-driven attacks. If you have endpoints that are exposed to the public internet, they're fair game for any attacker to come in and exploit it, whether they are a rogue agent used for security research, or if they're agents being driven by state actors.

- [Runaway AI hacking agent behind WorkOS breach after escaping sandbox: BleepingComputer](https://www.bleepingcomputer.com/news/security/runaway-ai-hacking-agent-behind-workos-breach-after-escaping-sandbox/)
- [Update for October 30th, 2026 Data Security Incident: WorkOS](https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident)

## GitBook breach that GitBook attributes to AI-driven automation

On November 2nd, attackers gained unauthorized access to GitBook's systems using a flaw in GitBook's SSO login flow. Using this flaw, they were able to act as GitBook staff and query the authentication information of a misconfigured GitHub Application, gaining access to internal code repositories and API keys.

In response, GitBook disabled every affected internal infrastructure service account, then rotated every secret and API key in our infrastructure, and every customer secret managed by GitBook.

While the attacker was not explicitly named, and the tooling used has not yet been identified, in GitBook's postmortem, they say that the speed and precision of the attack suggests that it was largely driven by automation an AI. GitBook also mentioned that the attack was focused on one customer.

If an AI-driven attack is confirmed, this suggests that an attacker-operated agent chain non-human identities (GitHub App, API Keys, service accounts etc.) into full infrastructure control. GitBook's MCP server and integration tokens sit in customer's agent toolkits, so the breach forces every downstream agent integration to re-authenticate.

- [Hackers Breach GitBook Using AI Agents: Forklog](https://forklog.com/en/hackers-breach-gitbook-using-ai-agents/)
- [Security incident: GitBook](https://www.gitbook.com/status/security-incident-2026-11-03)

## ARTEX campaigns widen: secrets-to-cloud abuse, Storm-3358 against finance, and 1,465 exposed consoles

Even more mischief with ARTEX! If you're a regular reader of the series, you have already seen ARTEX being used in different campaigns. 

On 5 November, Microsoft published research on threat actors using ARTEX to find and abuse secrets that developers had accidentally left on GitHub. Since August, the attackers had run large volumes of GitHub searches for credentials: passwords, OAuth application secrets, and AI service API keys in particular. Once they found something, the agents took over and did the follow-up work with very little human involvement.

On October 28th, the attackers signed in as service principals across dozens of applications and multiple tenants to see what each identity could do and reach. This included access to Azure subscriptions, resource groups, permissions, and AI deployments. The attackers then turned their attention to secrets such as storage account keys, App service settings, Kubernetes credentials and Key Vault contents. They were even able to search SharePoints, OneDrive, and Azure DevOps accounts, and use Run Command to execute code on Azure Virtual Machines.

![Attack-chain diagram showing compromised app credentials used for service principal sign-ins, cloud resource discovery, credential access, data collection and exfiltration, ending in Azure AI service abuse.](https://www.microsoft.com/en-us/security/blog/wp-content/uploads/2026/11/AI-tooling-scanning-secrets-attack-chain.webp)

*Leaked application credentials become a route to cloud data theft and Azure AI service abuse. Image credit: [Microsoft Security Blog](https://www.microsoft.com/en-us/security/blog/2026/11/05/threat-actors-use-agentic-ai-tooling-to-find-and-abuse-exposed-secrets/).*

The following day, Microsoft Security Research published another report on a Cambodian-based group attacking financial organizations based mostly in Southeast Asia. They used ARTEX alongside other automated tools to scan internet-facing systems. They then exploited weaknesses such as anonymous file uploads and server-side forgery (SSRF). Once they're inside compromised systems, they would use tools like `curl` and `netcat` to pull down open-source trojans so they can move around the network. Microsoft also saw password spraying, API abuse, database dumps, and web shells.

Finally, Censys found that more than 1,400 ARTEX control consoles were sitting on the open internet. Control consoles in ARTEX act as an admin panel where operators can use it to type in targets and goals, start and stop the agents, watch what they're doing in real time, and analyze the results.

![ARTEX control-console dashboard showing task and findings counts, 28 asset nodes, 1.2 million tokens used, a token-usage chart, and an activity feed.](https://censys.com/wp-content/uploads/image-334.png)

*The ARTEX dashboard gives operators a view of their agents' tasks, findings and token usage. Screenshot credit: [Censys](https://censys.com/blog/artex-agentic-asbestos/).*

“Exposed” here means the console’s web page can be reached from anywhere on the internet, instead of being hidden behind a VPN or locked down to the operator’s own machine. To be clear, Censys doesn’t say whether these consoles also needed a login, so this isn’t a case of anyone being able to stroll in and take control. But it does mean the attackers’ mission-control screens were in public view.

The headline number needs a little context. About 650 of those servers were set up identically across two blocks of hosting, which looks more like one or two people running lots of copies than 650 separate attackers. So Censys estimates roughly 815 distinct deployments, and says the real number is likely higher, since ARTEX works perfectly well with the console hidden away.

Then it gets a bit embarrassing for the attackers (sorry, not sorry). Censys found 32 servers with open directories, which are web folders that anyone can browse and download from. Inside were exploit kits and reverse-shell scripts, an operator’s entire home folder (including their SSH and command history), a private key for a root certificate authority used to intercept encrypted traffic, and, on a server linked to the attacks on South Korean financial firms, the operator’s `.claude` folder containing their AI prompts, session histories and ARTEX configuration files. 

*Yes, at least one attacker left their AI chat logs on the open internet.*

Censys also found that 155 of the hosts (about 10.6%) were running other attack infrastructure such as Cobalt Strike, vshell and AdaptixC2, so ARTEX is being bolted onto existing criminal toolkits rather than used on its own. Most of these directories were only up for hours or days, but Censys kept a record of what it saw.

Strip away the AI, and the root causes are very familiar: secrets in code, over-permissioned identities, and internet-facing services with weak spots. What the agents change is the economics. Finding a leaked key, working out what it can reach and squeezing every secret out of it used to take a skilled human hours or days. Now it takes minutes and costs the attacker next to nothing (ASI03 Identity and Privilege Abuse). The agents also bend legitimate cloud APIs on the fly until something works, which is tool misuse at machine speed (ASI02 Tool Misuse and Exploitation).

So if you’re running anything in Azure with app registrations you haven’t looked at in a while, now’s a good time to go through them. The agents certainly will.

- [Threat actors use agentic AI tooling to find and abuse exposed secrets - Microsoft Security Blog](https://www.microsoft.com/en-us/security/blog/2026/11/05/threat-actors-use-agentic-ai-tooling-to-find-and-abuse-exposed-secrets/)
- [ARTEX abuse expands as cybercriminals target financial sector - Microsoft Security Blog](https://www.microsoft.com/en-us/security/blog/2026/11/06/artex-abuse-expands-as-cybercriminals-target-financial-sector/)
- [ARTEX exposed consoles research - Censys](https://censys.com/blog/artex-agentic-asbestos/)
- [ARTEX AI Framework Linked to Bank Breaches and Cloud Attacks Found Exposed Across Hundreds of Servers - GBHackers](https://gbhackers.com/artex-ai-framework)
- [ARTEX and Friends: Enter Agentic Offensive Frameworks - Google Cloud Threat Intelligence](https://cloud.google.com/blog/topics/threat-intelligence/artex-and-friends-enter-agentic-offensive-frameworks/)

## BOSSAI: one operator, a 12-agent mesh, and Taiwan's emergency services

On November 4th, Acronis Threat Research Unit identified a single operator using a mesh of AI agents called BOSSAI to conduct persistent campaigns in several countries.

Using a mesh of agents, the operator targeted hundreds of organizations spanning government emergency services, banks, universities, military organizations, gambling sites, and e-commerce platforms in eleven countries. It's not clear if the attacker has any specific affiliation or goal beyond compromising as many organizations as possible, but it's assumed that with a wide-reaching attack like this, the motive is probably financial.

![BOSSAI mesh diagram showing one operator directing infrastructure agents Grok, OpenClaw and Claude Code to maintain offensive agents Hermes, CyberStrike and HalberdStrike, which attack victims.](https://staticfiles.acronis.com/images/content/3bff698c47ae087c3f8cd69baf10a21c.webp)

*One operator directs a 12-agent mesh, with infrastructure agents maintaining the offensive agents that target victims. Image credit: [Acronis Threat Research Unit](https://www.acronis.com/en/tru/posts/bossai-inside-the-ai-attack-mesh-that-targeted-taiwans-emergency-services/).*

What's interesting about the attack is that there doesn't seem to be much skill behind it. The BOSSAI mesh made use of commercial models for the infrastructure, which repeatedly refused to engage in actual offensive activity. Chinese models refused hacking instructions as well, so as a result, the operator was forced to turn to locally run models, which were less effective and constrained by resources.

So while the agents were able to create a high volume of attacks, they made numerous mistakes. The models were given excessive autonomy and instructed not to bother the operator, which lead to agents passing over lucrative targets because they were deemed out of scope.

Excessive autonomy cuts both ways. Obviously the danger here is that the entry bar for spinning up agents with cybersecurity capabilities is pretty low, however with the development of commercial models refusing to perform malicious hacking activities, these types of attacks where criminal organizations attempt to use agents to perform illegal activity is becoming less effective.

- [BOSSAI: Inside the AI attack mesh that targeted Taiwan’s emergency services: Acronis Threat Research Unit](https://www.acronis.com/en/tru/posts/bossai-inside-the-ai-attack-mesh-that-targeted-taiwans-emergency-services/)
- [Single hacker uses 12 AI agents to launch global cyberattacks: Cybernews](https://cybernews.com/security/single-hacker-twelve-ai-agents-cyberattack/)

## "Haiou" Codex jailbreak bundle carries prompt adware

On November 3rd, the folks at Check Point Research analyzed "Haiou", a jailbreak and configuration bundle of OpenAI Codex.

What the bundle does is rewrites Codex's global instructions, using role-play and "CTF" framing to weaken guardrails, and installs offensive "security skills". It also injects adverts for a paid OpenAI-compatible gateway, triggered when a user says "hi", "hello", or "yo".

The payload has been described by Check Point as **prompt adware**, and is potentially being used as a acquisition channel for a paid gateway service. So rather than annoying pop-ups or banners, the prompt adware manipulates responses from an agent to promote the gateway service, which overrides the model's safety training and content guardrails.

Third-party configurations and libraries have always been a vector for supply chain attacks. Here, we have a skills bundle that rewrites the agent's behavior. Manipulation of the persistent global instructions redirect the agent to serve the bundle author's goals, and the changes that are persisted in instruction files are loaded into every session. Over time, the adverts are delivered in the trusted voice of their own agent, exploiting the trust between humans and agents.

- [Free the Model, Sell the Gateway: Prompt Adware hidden in Jailbreak Bundles: Check Point Research](https://research.checkpoint.com/2026/free-the-model-sell-the-gateway-prompt-adware-hidden-in-jailbreak-bundles/)
- [Researchers Uncover Prompt Adware Hidden in OpenAI Codex Jailbreak Toolkit: The Hacker News](https://thehackernews.com/2026/11/researchers-uncover-prompt-adware.html)

## MCPhantom: coding agents install deprecated, unpatched MCP servers from memory

On November 2nd, Glow Labs discovered that coding agents install Anthropic's archived reference MCP servers when told by a user to set up an MCP server.

In 98% of observed sessions, the coding agent will install an archived MCP server based on the model's training data set. For example, GitHub maintains their own MCP server, but the agent will pass over the latest version of the MCP server and instead choose a version from it's own training data. Even if the repository of the MCP server is archived and it states that no security updates are being worked on, the agent won't read the archive notices.

Glow Labs calls this pattern MCPhantom, where retired MCP servers keep being installed on developer machines.

This pattern was observed in Claude Code, Cursor, Codex CLI, Claude Cowork, and Claude Chat. Models also included those from Anthropic, OpenAI, and xAI.

This pattern can introduce serious vulnerabilities into developer environments. The most serious example of this was a remote code execution (RCE) bug found in the Puppeteer server, which would let an AI agent open a browser session. Using prompt injection, an attacker could make any coding agent start any process on the machine instead of the browser, and run any command. When Glow Labs reported the bug, the maintainers stated that the server was archived in May 2025, so no fix will be applied to the server.

These type of dependencies are hard to track as they don't always explicitly appear in dependency files. In some cases, agents that would check a package on npm and got "This package is deprecated" would still install the package anyway.

As defenders, the responsibility is on us to create agent harnesses that refuse to install deprecated servers, or provide feedback that lets operators know when a server is deprecated. Pre-execution hooks would also be beneficial here to prevent compromised or unsupported servers being made available to coding agents.

- [MCPhantom: AI Coding Agents Are Installing Vulnerable Deprecated MCP Servers from Training Data: Glow Labs](https://www.glow.io/blogs/mcphantom-ai-coding-agents-are-installing-vulnerable-deprecated-mcp-servers-from-training-data)
- [AI coding agents are installing MCP servers that will never be patched: Help Net Security](https://www.helpnetsecurity.com/2026/11/03/mcphantom-deprecated-mcp-servers/)

## Docker Sandboxes symlink escape (CVE-2026-103916)

Docker Sandboxes allow developers to run coding agents in isolated environments and prevent them from having unrestricted access to the developer's computers. 

Peyton Kennedy at Endor Labs found a vulnerability that breaks the security boundary of the sandbox (CVE-2026-103916). Commands used to copy files between a sandbox and the developer's computer does not enforce access restrictions.

An agent could exploit this by using a symbolic link, essentially a shortcut that points to another file or folder. By placing this shortcut inside a workspace, the agent can trick a subsequent copy command into accessing files outside the sandbox's approved boundaries.

An agent could use this to gain access to sensitive files on the developer's computers (including access to cloud credentials and SSH keys), and it could overwrite files on the developer's computers, which could lead to malicious code being executed from outside the box.

The full fix for this vulnerability has been released on November 3, in v0.51.0.

- [Docker Sandboxes vulnerability lets AI agents access sensitive files outside their sandbox (CVE-2026-103916): Endor Labs](https://www.endorlabs.com/learn/docker-sandboxes-vulnerability-lets-ai-agents-access-sensitive-files-outside-their-sandbox-cve-2026-103916)

# Conclusion

While all of these attacks are pretty simple, I think we're still at the stage where all of our legacy shit and fixes we forgot to implement are coming back to bite us. Secrets, legacy tools, API Keys etc. are just there for the taking, and even the script-kiddies and vibe hackers are getting some small successes in compromising our environments.

This week, have a hunt and look for legacy tools that may be exposed to the internet, and for any secrets that you may have lying around (hopefully not in your public repositories!). If you find anything, rotate your keys, delete secrets (use RBAC where you can!), implement rate-limiting on your public APIs, and take anything off the public internet that you don't want exposed to it.

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️