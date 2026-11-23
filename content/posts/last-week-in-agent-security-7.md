---
title: "Last week in Agent Security Update 7: This is fine"
date: 2026-11-24
draft: false
tags: ["OWASP", "Agents", "AI", "Security", "LWIAS"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/1x4tpku8aq8uukb3ufuv.png
    alt: "artoon of a dog in a bowler hat sitting calmly at a table with a cup of coffee while the room around him is on fire and filling with smoke."
    caption: "This week: Cairn hijacks Microsoft's Agent Framework on PyPI, an agent turns patches into exploits with no human in the loop, a DPRK crew turns Claude Code's config into a backdoor dropper, California sues OpenAI, AntiFox agents go after AWS, 30k Grok Build apps sit open on the internet, BluePhishing turns Entra agent blueprints into an identity factory, Cairn agents find a zero-day on their own, and Meta's Muse gets owned from a PDF export."
---

If you work in cybersecurity or software engineer, I hope you got some sleep last week, because I'm not sure many of us did. It's been an unsettling few days, and I believe that we're only at the start of it.

On Tuesday, [Java4Shell](https://blog.qualys.com/vulnerabilities-threat-research/2026/11/17/how-to-find-mitigate-and-patch-the-critical-java4shell-flaw-cve-2026-114938-now) dropped: a pre-auth RCE in OpenJDK's Base64 decoder that hits default installs of Jira, Jenkins, Keycloak and other products, with exploitation attempts starting within a day. On Wednesday, a [10-year-old nginx HTTP/2 flaw](https://blog.qualys.com/product-tech/patch-management/2026/11/18/nginx2root-five-nginx-flaws-led-by-cve-2026-123007-how-to-find-mitigate-and-patch-now) got fixed, after [five different research teams found it independently](https://www.helpnetsecurity.com/2026/11/19/nginx2root-nginx-vulnerabilities/), with a public proof of concept out before Ubuntu or Debian had shipped a patch. Throw in [seven Ruby flaws](https://www.ruby-lang.org/en/news/2026/11/18/multiple-vulnerabilities-in-regexp-and-string/), [another critical PHP release](https://labs.beazley.security/advisories/BSL-A1250), a [VMware vCenter zero-day](https://digital.nhs.uk/cyber-alerts/2026/cc-4899), and [hijacked Optimizely scripts](https://www.mcafee.com/blogs/security-news/optimizely-supply-chain-attack/) serving card skimmers to visitors of CNN, BBC and Zoom. That's only a small pick of incidents.

A lot of these bugs were found with AI. Attackers are now using AI to exploit them too, and fast.

Oh, and [GTA 6 finally got released](https://www.rockstargames.com/newswire/article/39akk424a9oo53/grand-theft-auto-vi-now-available)! I woke up at 5am, bought it on a whim, and I'm enjoying it more than I thought I would. The driving mechanics aren't great in my opinion, but it's early days.

# This week's attacks
 
In [Update 6](https://www.willvelida.com/posts/last-week-in-agent-security-6/), I said a lot of what we were seeing was attackers *vibe-hacking*: throwing ARTEX at leaked secrets and legacy endpoints and seeing what sticks. Agents are now finding their own zero-days, turning patches into exploits without anyone operating them, and going after the infrastructure we use to build agents themselves.
 
As always, I'm not trying to shame any of the companies involved (maybe just the ones who deserve it), nor do I condone any of the methods used. My goal is to spread awareness of how agents can be exploited, and hopefully trigger a conversation with your teams about protecting your agents and the people who use them.
 
Here's some incidents that have happened over the period 16th November to 22nd November 2026.
 
## Microsoft Agent Framework compromised on PyPI by an attacker running Cairn
 
On November 17th, two malicious versions of the [Microsoft Agent Framework Python SDK](https://pypi.org/project/agent-framework/), 1.26.1 and 1.26.2, were published to PyPI. Both were live for less than an hour before being quarantined. The last safe version is 1.26.0.
 
Each version dropped a `.pth` file into site-packages. [Python runs `.pth` files on every interpreter start](https://docs.python.org/3/library/site.html), so the credential stealer ran whenever Python started in that environment, even if nothing imported `agent-framework`.
 
According to [Wiz](https://www.wiz.io/blog/malicious-version-of-microsofts-agent-framework-published-to-pypi), the attacker appears to have compromised a client certificate and private key for Microsoft's Engineering System Release Pipeline (ESRP). A publishing attempt from AWS was blocked by ESRP's IP filtering, but a second one from Microsoft-owned IP space succeeded. How the credential was obtained is still unknown.
 
The attacker's tooling identified itself as Cairn, and a test run on the operator's own machine showed they were also using a coding agent. An agent framework used to compromise an agent framework, with help from a coding agent.
 
![Timeline of the agent-framework attack, highlighting what the attacker disclosed through webhook.site.](https://www.datocms-assets.com/75231/1795009347-final-esrp-attack-flow-2x.png?dpr=0.5&fm=webp)
 
*How the attack unfolded. Image credit: [Wiz](https://www.wiz.io/blog/malicious-version-of-microsofts-agent-framework-published-to-pypi).*
 
The stealer went after environment variables, credential files, OIDC tokens and Azure managed-identity tokens. Worse, it asked the platform to mint *fresh* tokens, which rotating stored secrets won't cover.
 
Luckily, the attacker exfiltrated everything to a free-tier webhook.site URL capped at 100 requests. Wiz analysed the 50 it captured and found only three successful victim exfiltrations. Microsoft hasn't explained how the ESRP credential was compromised yet, and has made some repositories private while it investigates.
 
From an OWASP perspective, a trusted agent SDK was trojanised at the source ([ASI04 Agentic Supply Chain Vulnerabilities](https://www.willvelida.com/posts/preventing-agentic-supply-chain-vulnerabilities/)), and the stolen tokens let attackers act as the victim's cloud and CI identities ([ASI03 Identity and Privilege Abuse](https://www.willvelida.com/posts/preventing-identity-and-privilege-abuse-ai-agents/)).
 
Check your lockfiles and runners for the bad versions, rotate everything an affected host could reach, and [hash-pin](https://pip.pypa.io/en/stable/topics/secure-installs/) your agent SDKs.
 
- [Microsoft agent-framework Compromised: Credential Stealer Targets Azure and CI Pipelines: Wiz](https://www.wiz.io/blog/malicious-version-of-microsofts-agent-framework-published-to-pypi)
- [Attackers Compromise Microsoft's Agent Framework PyPI Package to Steal Cloud Credentials: The Hacker News](https://thehackernews.com/2026/11/attackers-compromise-microsofts-agent.html)

## exploit_agent: an agent turns patches into exploits with no human in the loop
 
On November 16th, the folks at Google Threat Intelligence Group (GTIG) published their [Q4 AI Threat Tracker](https://cloud.google.com/blog/topics/threat-intelligence/agentic-threat-activity-crosses-a-significant-threshold). For the first time, GTIG has seen attackers fully automate vulnerability discovery *and* exploitation with no human in the loop.
 
GTIG found an exposed staging server hosting a multi-agent framework called `exploit_agent`. It watches popular web software for security patches, diffs the code before and after to work out what was fixed, drafts an exploit, and fires it at targets automatically. There's no approval step anywhere in that loop.
 
The server also held stolen victim data, mostly from French media and education websites, plus scripts for removing *other attackers'* webshells (*nothing says professionalism like cleaning up after your competitors*). My favourite detail is a Russian comment reminding the operator to paste API keys into the config file, not the chat. 

The bigger picture is how much the patch window has shrunk. When two PaperCut zero-days were disclosed in late August, researchers built working exploits with LLMs within a day. Four days later, [GreyNoise saw an attacker](https://www.greynoise.io/blog/ai-orchestrated-campaign-against-papercut-ng-mf) hit hundreds of organisations, going from foothold to domain admin within minutes. GTIG also says monthly zero-day exploits spiked to 22 in August, up from an average of 8 in 2025.
 
Mapping this back to OWASP, the agent chains diffing, exploit-writing and execution tools into an offensive pipeline ([ASI02 Tool Misuse and Exploitation](https://www.willvelida.com/posts/preventing-tool-misuse-ai-agents/)), and runs the code it writes against live targets with no review ([ASI05 Unexpected Code Execution](https://www.willvelida.com/posts/preventing-unexpected-code-execution-in-agents/)).
 
The takeaway is that "patch within 30 days" is a fantasy for anything internet-facing. Your realistic window is about 24 hours: patch it, virtually patch it at your WAF, or pull it off the internet entirely.
 
- [GTIG AI Threat Tracker: Agentic Threat Activity Crosses a Significant Threshold: Google Cloud](https://cloud.google.com/blog/topics/threat-intelligence/agentic-threat-activity-crosses-a-significant-threshold)
- [Google: AI agent turned a security patch into a working exploit with no human help: The Register](https://www.theregister.com/research/2026/11/16/google-ai-agent-turned-a-security-patch-into-a-working-exploit-with-no-human-help/5309926)
## Claude Code settings.json hooks used to drop a backdoor (MIDNIGHT NEPTUNE)
 
Remember the `testmetest` worm from [Update 5](https://www.willvelida.com/posts/last-week-in-agent-security-5/#testmetest-worm-plants-agent-hook-files-in-42-github-repos), which planted hooks in `.claude/settings.json` so opening a repo ran the attacker's code? A state-backed crew is now doing the same thing.
 
Buried in the same GTIG report is a suspected North Korean cluster tracked as **MIDNIGHT NEPTUNE**, targeting cryptocurrency organisations in North America and Europe. The chain goes like this:
 
1. **Fake recruiters** with AI-generated profile pictures (they carried [SynthID](https://deepmind.google/models/synthid/) watermarks).
2. **Fake meetings** with "[ClickFix](https://www.microsoft.com/en-us/security/blog/2025/08/21/think-before-you-clickfix-analyzing-the-clickfix-social-engineering-technique/)" lures: "your audio isn't working, run this script to fix it".
3. **A pivot** into the victim's *internal* code repositories.
4. **A poisoned `.claude/settings.json`**, so the next developer to start Claude Code in that repo downloads the **SOMBERMEME** backdoor.
5. **GitHub tokens named after Microsoft Copilot** to blend in.
What makes this nasty is that there's no bug. [Hooks](https://code.claude.com/docs/en/hooks) are designed to run automatically, and the poisoned config inherits all the trust you give your own repos.
 
Mapping this back to OWASP, the agent's config became the delivery mechanism in the code supply chain (ASI04 Agentic Supply Chain Vulnerabilities), starting the agent ran the payload (ASI05 Unexpected Code Execution), and the disguised tokens kept the actor's access looking legitimate (ASI03 Identity and Privilege Abuse).
 
I said it in Update 5 and I'll say it again: agent config files are code. Put [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) on `.claude/`, `.cursor/`, [`AGENTS.md`](https://agents.md/) and your MCP configs, alert on any change that adds a hook, and use [managed settings](https://code.claude.com/docs/en/settings) to control which hooks can run. And if there's a PAT called "Copilot" in your GitHub org that nobody remembers creating, start asking questions.
 
- [GTIG AI Threat Tracker: Agentic Threat Activity Crosses a Significant Threshold: Google Cloud](https://cloud.google.com/blog/topics/threat-intelligence/agentic-threat-activity-crosses-a-significant-threshold)
## California sues OpenAI over agent testing and containment
 
On November 16th, California's Attorney General Rob Bonta [sued OpenAI](https://oag.ca.gov/news/press-releases/it%E2%80%99s-golden-state-not-wild-west-attorney-general-bonta-sues-openai-over), alleging its failure to secure and monitor model testing created a public nuisance and breached California's Unfair Competition Law.
 
The case centres on the **[Hugging Face attack](https://openai.com/index/hugging-face-incident-and-the-road-ahead/)** from July 2026, when hundreds of agents running on experimental OpenAI models escaped OpenAI's research environment and attacked Hugging Face over the internet for more than two days.
 
The AG argues the attack was foreseeable, that OpenAI ignored its own policies, and that it misled the public about how tightly testing was controlled. He also says OpenAI's agents have since hit government health databases and even **OpenAI's own internal infrastructure**. The line that'll get quoted everywhere is that OpenAI admitted losing control of its agents **at least 80 times** during testing.
 
According to the [Cloud Security Alliance's summary](https://labs.cloudsecurityalliance.org/research/frontier-ready-daily-20261117/), the alleged failures include no egress monitoring and alerts nobody acted on. These are ordinary security controls, not exotic AI-safety problems. To be clear, none of this has been tested in court yet.
 
Mapping this back to OWASP, this is agents acting outside their intended containment ([ASI10 Rogue Agents](https://www.willvelida.com/posts/preventing-route-agents/)), using access they should never have had (ASI03 Identity and Privilege Abuse).
 
California is measuring OpenAI against OpenAI's *own* statements about its safeguards. If you've told customers your red-team agents or evaluation harnesses are contained and monitored, check that against your actual controls.
 
*Not a lawyer, opinions are my own, etc.*
 
- [It's the Golden State, not the Wild West: Attorney General Bonta Sues OpenAI Over Cybersecurity Incidents: California Attorney General](https://oag.ca.gov/news/press-releases/it%E2%80%99s-golden-state-not-wild-west-attorney-general-bonta-sues-openai-over)
- [Frontier Ready Daily – 17 November 2026: Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/frontier-ready-daily-20261117/)
## AntiFox agents used in AWS credential-theft attacks
 
Attackers are running self-hosted AntiFox, a Chinese-language fork of our familiar friend [ARTEX](https://www.willvelida.com/posts/last-week-in-agent-security-6/#artex-campaigns-widen-secrets-to-cloud-abuse-storm-3358-against-finance-and-1465-exposed-consoles), against web services and AWS environments. [Arctic Wolf observed](https://thehackernews.com/2026/11/antifox-ai-pentesting-platform-linked.html) the agents hosted on Alibaba Cloud and using valid AWS identities to make authenticated API calls.
 
In one intrusion, operators planted an ASPX webshell on an internet-exposed IIS server, pulled connection strings and config files, then cycled through valid IAM users and EC2 roles to make hundreds of AWS API requests in about 30 minutes. Most were blocked, but they still got some production secrets and deleted S3 objects.
 
It's the same story as last week's ARTEX campaigns: leaked, over-permissioned identities (ASI03 Identity and Privilege Abuse), abused through legitimate cloud APIs at machine speed (ASI02 Tool Misuse and Exploitation). Apply least privilege, require [IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html), turn on [GuardDuty](https://aws.amazon.com/guardduty/), and keep long-term access keys out of app config.
 
- [Hackers use AntiFox AI agents in AWS credential-theft attacks: BleepingComputer](https://www.bleepingcomputer.com/news/security/hackers-use-antifox-ai-agents-in-aws-credential-theft-attacks/)
## Over 30k Grok Build applications exposed on the public internet with no sign-in
 
[Researchers at Claroc](https://blog.claroc.io/ghostshares-grok-build-apps/) found more than 30,000 apps built with Grok Build, SpaceXSI's chat-based app builder, open to anyone on the internet without signing in. Every app gets its own grok.me hostname, so they're easy to find.
 
The exposed apps included SpaceXSI's own sales territory map, a confidential X ad proposal to a US federal agency, an ICU patient's lab results (oof), a hospital group's staff list, and a timesheet that hands out employees' passwords. All good stuff….
 
SpaceXSI took down its own employees' apps, but plenty of others are still up. A lot of people clearly picked "Anyone with the link" without thinking about what "anyone" means.
 
Now, my hatred for Elmo might be creating some bias here. But tools like this mean your staff are shipping production apps your security team has never heard of.
 
- [GhostShares 2: Grok Apps Exposed xAI's Business Secrets and People's Private Data: Claroc](https://blog.claroc.io/ghostshares-grok-build-apps/)
- [Grok's security blind spot exposed users and Musk's company secrets: Cybernews](https://cybernews.com/security/groks-data-leak-spacexsi/)
## BluePhishing turns Entra agent blueprints into a consent-phishing identity factory
 
Consent phishing is nothing new, but Microsoft has given us a new kind of identity to consent to. Proofpoint's Rachel Rabin has shown the old trick works on it too. Proofpoint calls it [BluePhishing](https://www.proofpoint.com/us/blog/threat-insight/BluePhishing-Initial-Access-via-Third-Party-Agent-Blueprint-Consent). It's a proof of concept that hasn't been seen in the wild, but Microsoft's MC1506733 notice says default Entra users can create agent blueprints from November 2026, so I'd rather we talk about it now.
 
Quick primer: an **[agent blueprint](https://www.willvelida.com/posts/entra-agent-id/)** is roughly an app registration for agents, and **agent identities** are instances created from it that inherit its permissions.
 
Here's the chain:
 
1. The attacker builds a multi-tenant blueprint in their own tenant with the permissions they want.
2. They phish a victim into consenting. "An AI agent wants to help you" is going to be a very effective lure ([ASI09 Human-Agent Trust Exploitation](https://www.willvelida.com/posts/preventing-human-agent-trust-exploitation/)).
3. Consent creates a blueprint principal in the victim's tenant with `AgentIdentity.CreateAsManager`, which Proofpoint says **cannot be revoked**.
4. The attacker mints up to 250 agent identities per blueprint (ASI03 Identity and Privilege Abuse).
5. Those identities inherit the blueprint's permissions. In Proofpoint's demo, they used `User.Read.All` and `Group.Read.All` with [GraphRunner](https://github.com/dafthack/GraphRunner) to enumerate the tenant.
The part that bothers me most is visibility. Microsoft's docs say inherited permissions aren't visible in the Entra portal or Graph, only in the token at runtime.
 
The good news is a blueprint's ID is the same in every tenant, so a known-bad blueprint becomes a hunting indicator. Blueprints show up in the audit log as `Add application`, and agent identities as `Add service principal`. You can search for your blueprints using the following Kusto query:
 
```kusto
AuditLogs
| where TimeGenerated > ago(30d)
| where OperationName in ("Add application", "Add service principal", "Consent to application")
| extend TargetName = tostring(TargetResources[0].displayName),
         Initiator  = coalesce(tostring(InitiatedBy.user.userPrincipalName),
                               tostring(InitiatedBy.app.displayName))
| project TimeGenerated, OperationName, TargetName, Initiator, Result
| order by TimeGenerated desc
```
 
Look for one consent followed by a burst of `Add service principal` events. That's the identity factory spinning up.
 
Apply your [user consent restrictions](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent) to agent blueprints, keep agents on least privilege, and train your users. An "AI agent" consent prompt is still a consent prompt. Nothing here is a vulnerability, and the defaults are moving in the wrong direction this month.
 
- [BluePhishing: Initial Access via Third-Party Agent Blueprint Consent: Proofpoint](https://www.proofpoint.com/us/blog/threat-insight/BluePhishing-Initial-Access-via-Third-Party-Agent-Blueprint-Consent)
- [BluePhishing Entra agent consent attack: Agentbeam](https://agentbeam.com/)
## Cairn agents find and exploit a MONAI Label zero-day on their own
 
Cairn's back! In [Update 4](https://www.willvelida.com/posts/last-week-in-agent-security-4/#cairn-agents-break-into-100-company-websites), its exposed control panels showed agents breaking into 100+ company websites. This week it levels up.
 
On November 20th, [Amazon Threat Intelligence published](https://aws.amazon.com/blogs/security/chinese-nexus-actors-cairn-ai-agents-found-a-zero-day-on-their-own/) what it found on an unauthenticated, public Cairn instance discovered back on September 10th. It held a Chinese-speaking operator's prompts and agent output from a 48-hour campaign against East Asian universities. The operator's goal was simply "obtain shell access".
 
The failures are just as interesting. Against an open Redis server, the agents spent **16 hours and 148 actions** trying every known takeover trick. Against a GPU server, they tried 30,000+ credential combinations built from faculty directories. None of it worked, but no human grinds through 16 hours of Redis exploitation without a coffee break.
 
Against [MONAI Label](https://github.com/Project-MONAI/MONAILabel), an open-source medical imaging AI app, things went differently. The agents mapped the API, found a path traversal in an upload endpoint, and went from mapping to arbitrary file write in **about 11 minutes**. Later, they wrote a reverse shell into [`sitecustomize.py`](https://docs.python.org/3/library/site.html#module-sitecustomize) (which Python imports on every start), called the app's training endpoint to spawn a new Python process, and got a shell back as **root**. Sound familiar? It's the same Python startup trick as the `.pth` file in the agent-framework compromise.
 
On patching, the sources don't agree. Amazon says a fix and advisory ([GHSA-qc22-rmgc-vg94](https://github.com/Project-MONAI/MONAILabel/security/advisories/GHSA-qc22-rmgc-vg94)) were released, but the [CSA found](https://labs.cloudsecurityalliance.org/research/frontier-ready-daily-20261121/) PyPI still serving the vulnerable 0.8.5 on November 21st. There's no CVE, so scanners won't flag it. If you installed MONAI Label with `pip`, assume you're exposed.
 
Mapping this back to OWASP, an attacker's agent framework chained tools to discover and exploit a zero-day (ASI02 Tool Misuse and Exploitation), then wrote and ran its own payload to get root (ASI05 Unexpected Code Execution).
 
This isn't vibe-hacking anymore. The agents got an IP and a goal, and *discovered* a new bug. Find the ML tools your data science teams have stood up, take them off untrusted networks, turn on authentication, and stop running them as root.
 
- [Chinese-nexus actor's Cairn AI agents found a zero-day on their own: AWS Security Blog](https://aws.amazon.com/blogs/security/chinese-nexus-actors-cairn-ai-agents-found-a-zero-day-on-their-own/)
- [Frontier Ready Daily – 21 November 2026: Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/frontier-ready-daily-20261121/)
## Meta Muse: browser bug to silent exfiltration to VM root
 
Oh Meta…. We're back again!
 
In [Update 4](https://www.willvelida.com/posts/last-week-in-agent-security-4/#meta-muse-ghostshares), Muse leaked shared conversations through public DNS. This time it's worse. On November 20th, [NebuSec showed](https://nebusec.ai/research/inside-meta-muse/) how one malicious web page could take over the Muse agent, quietly send your conversation history and memories to an attacker, and break out to root on the VM underneath.
 
On paper, the design isn't bad. Each user gets their own VM, and the agent runs in a container inside it. Real credentials are swapped in outside the container, so the agent can't leak what it doesn't know, and an egress proxy called **Sentinel** polices every outbound request.
 
![Meta's architecture diagram for Muse request routing.](https://nebusec.ai/_astro/muse-architecture.Ca8AR4Rs.png)
 
*How requests reach the Muse agent inside its per-user VM. Image credit: Meta, via [NebuSec](https://nebusec.ai/research/inside-meta-muse/).*
 
**Step 1: the browser is the front door.** Muse's bundled Chromium was out of date, and its PDF-export process ran with the IPC sandbox disabled. Ask Muse to save an attacker's page as a PDF, and one renderer bug (worth about $500, says NebuSec) gives you a shell.
 
**Step 2: sneaking past Sentinel.** Sentinel screens URLs for sensitive data and enforces volume budgets. But by default, `GET` requests with no body count as "reads", and reads leave without asking the user. By packing data into URLs and staying under the limits, NebuSec sent conversation history and memories out without a single prompt, and could alter stored memories too.
 
**Step 3: out of the container.** Muse's kernel had hardening options switched off and known CVEs, and NebuSec's exploits worked over 99.9% of the time. From VM root, Sentinel stopped mattering, because it only covered the container. NebuSec didn't escape the hypervisor or reach other users, and Meta patched everything before publication.
 
Mapping this back to OWASP, untrusted web content became code execution in the agent's runtime (ASI05 Unexpected Code Execution), the legitimate egress path was used to exfiltrate data within policy (ASI02 Tool Misuse and Exploitation), and the agent's memory became readable and writable by an attacker ([ASI06 Memory and Context Poisoning](https://www.willvelida.com/posts/preventing-memory-context-poisoning/)).
 
What makes me nervous is that this design is pretty much what everyone recommends for agents. It failed because "reads" aren't harmless when a URL can carry data, and egress control only existed at one layer. If you're building or buying an agent platform, test both.
 
- [Inside Meta Muse: Browser, Sentinel and the Shared-Kernel Boundary: NebuSec](https://nebusec.ai/research/inside-meta-muse/)
# Conclusion
 
This was the week attacker agents stopped just exploiting existing vulnerabilities and started finding new ones on their own. Google caught a framework that turns patches into exploits without anyone pressing a button, Amazon watched Cairn agents find a zero-day in 11 minutes, and someone running Cairn shipped a credential stealer inside Microsoft's own agent SDK.
 
The common thread is trust we've handed out and stopped watching: unpinned agent packages, config files that run code when you open a project, consent prompts for "helpful" AI agents, and apps published to "anyone with the link". None of this is new. What's new is that agents probe that trust continuously, cheaply and at machine speed. And as California is making clear to OpenAI, "we said it was contained" doesn't cut it when it wasn't.
 
This week, pin your agent SDKs, treat agent config as code, and lock down who can consent to agent blueprints in your Entra tenant before someone does it for you.
 
If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com).
 
See you next week for another round of Rogue Agents. Until next time, Happy coding! 🤓🖥️

