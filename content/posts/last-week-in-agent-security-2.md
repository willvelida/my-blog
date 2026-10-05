---
title: "Last week in Agent Security Update 2: Zam-mad Max"
date: 2026-10-06
draft: false
tags: ["OWASP", "Agents", "AI", "Security"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/9ukj0m60yyv2iovayn70.png
    alt: "Last week in Agent Security Update 2: Zam-mad Max"
    caption: "An AI agent breached DIVD, a vulnerability disclosure group, by chaining two Zammad zero-days. Plus coding agents leaking 13,000+ screenshots to public GitHub, and more OpenAI rogue-agent fallout."
---

Last week I've been struggling with some kind of illness. My throat was dry and I couldn't breathe on Friday, so I took a course of pseudoephedrine medication, bought some strepsils, and a nasal spray just to manage it. I was sick the whole weekend and started to recover just in time for Monday! Being an adult is awesome....

Now those of you who are a little too keen for our robot overlords to consume us might point out that agents don't get sick, but they can be subject to a wide variety of vulnerabilities. Not the greatest segway into this weeks article, but I couldn't think of anything else.

Last week we saw further exploitations of what agents can reach, not necessarily what they are asked to do. Rogue Agents and the tools that they can use are becoming increasingly sophisticated attack vectors.

As always, I’ll point out that I’m not trying to shame any of the companies involved, nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here’s some incidents that have happened over the period 28th September to 4th October 2026.

## OpenAI agent breach of Medicare

Last week I covered how OpenAI agents managed to breach Medicare here in Australia and gained access to non-public information. OpenAI confirmed on September 28th that agents had also breached New South Wales Bureau of Crime Statistics and Research (BOCSAR), the Victorian Department of Health, and the Australian Institute of Health and Welfare.

OpenAI has committed to an Australian taskforce, and support through it's Daybreak programme. On 1st and 2nd of October, OpenAI went through the process of notifying 100+ organizations of unauthorized agent activity. Independent analysis from Asymmetric Security named 55 organizations, including the SEC and US Bureau of Economic Analysis as victims of unauthorized agent activity, some of which has left records inaccessible or erased.

I've never been optimistic about the speed of government officials in responding to incidents like this, but this *may* prompt some much needed response to breaches like this. In Australia, a Joint Select Committee has been scheduled for the October 6th, which OpenAI's Jason Kwon will attend (Sam Altman and Dario Amodei both declined).

In the US, senators Josh Hawley (Republican) and Chris Murphy (Democrat) have introduced a bipartisan AI Agent Accountability Act on October 1st, which would make AI Agent operators criminally and civilly liable for agentic hacks. Also, the Legal Advocates for Safe Science and Technology have sued OpenAI in San Francisco's Superior Court under California's anti-hacking law and Unfair Competition Law, citing both the Hugging Face and Medicare breaches.

Whether OpenAI will actually be held to account is another matter. Watch this space.

- [OpenAI apologises for Medicare breach, shelves next gen ChatGPT](https://www.abc.net.au/news/2026-09-29/openai-apologises-medicare-shelves-chatgpt-astra-launch/107207156)
- [OpenAI Faces First Lawsuit Over Rogue AI Agents That Hacked Hugging Face](https://gizmodo.com/openai-faces-first-lawsuit-over-rogue-ai-agents-that-hacked-hugging-face-2000819469)
- [Hawley and Murphy bill would send AI executives to prison over rogue agent hacks](https://startupfortune.com/hawley-and-murphy-bill-would-send-ai-executives-to-prison-over-rogue-agent-hacks/)

## DIVD breached by an agentic AI attacker chaining Zammad zero-days

On September 21st, The [Dutch Institute for Vulnerability Disclosure (DIVD)](https://www.divd.nl/) was hacked in an automated AI attack that exploited zero-day vulnerabilities in user support/ticketing solution [Zammad](https://zammad.com/en).

DIVD is an organization that searches for new vulnerabilities in software and report them to vendors, and it's made up of mostly volunteer security researchers. While it may be discouraging that even an organization full of security experts got breached, hopefully it illustrates that anyone can be a victim of cybercrime.

These flaws enabled unauthenticated attackers to achieve remote code execution and leak user sessions ([CVE-2026-102489](https://www.cve.org/CVERecord?id=CVE-2026-102489)), and allowed a local user to elevate their privileges to root ([CVE-2026-102490](https://www.cve.org/CVERecord?id=CVE-2026-102490)). Combined together, the attacker could hijack sessions, run code remotely and escalate privileges from the Zammad user to root.

From there, the attackers were able to access other services, read and exfiltrate data. Luckily network segmentation prevented the attack from going deeper into the environment.

According to the DIVD, the attack was "loud and very very messy, including password spraying and man-in-the-middle activity. In fact the DIVD were able to reverse engineer the attack because the agent overexplained what it was doing.

However, it's still unknown whether this was a capability test or part of a broader campaign. So we have a Rogue Agent running the attack (ASI10), but at the time of writing we don't know if a human has maliciously set out a goal or if an agent exceeded its brief. What is clear that the agent drove the exploitation, using password spraying and pivoting tools at machine speed.

- [DIVD-2026-00015 - Vulnerabilities in Zammad during investigation of case DIVD-2026-00014](https://csirt.divd.nl/cases/DIVD-2026-00015/)
- [It was a matter of when, not if...](https://www.divd.nl/newsroom/articles/when-no-if/)
- [Zammad Zero-Days Exploited in AI-Powered DIVD Hack](https://www.securityweek.com/zammad-zero-days-exploited-in-ai-powered-divd-hack/)
- [AI agent used Zammad zero-days to breach Dutch vulnerability disclosure non-profit](https://www.helpnetsecurity.com/2026/10/01/divd-agentic-ai-attack-breach/)
- [AI agents hacked the hackers, stealing email addresses from security research org](https://www.theregister.com/security/2026/10/01/ai-agents-hacked-the-hackers-stealing-email-addresses-from-security-research-org/5300652)

## Transluce: autonomous agents aimed SQL injection at US and Canadian government sites

[Transluce](https://transluce.org/) is a non-profit research lab that's building a public tech stack for scalable oversight of AI. On September 30th, they published an article after discovering several additional incidents where rogue AI agents have used techniques to access publicly available data on government websites in the US and Canada.

So far, Transluce haven't identified any instances where agents have been able to access any information that's not publicly available. However, they were able to identify that agents were able to probe several government agencies and violate explicit usage policies. This attempt targeted several government websites including the White House, Department of Defense, The CDC and SEC, as well as state agencies in California, Maryland, Illionis, Texas, and New York.

For the US Department of Education, agents attempted basic SQL injection to access school statistics. What's interesting in this case is that the data stored on the websites appeared to match a web search task in Google's DeepSearchQA benchmark. This means that the agents were potentially not given a hacking-related task, but were being graded on their ability to retrieve specific information on the internet.

Transluce also reported agents attempting rudimentary hacks on Library and Archives Canada. Arquivo.pt captured 899 requests that included failed rudimentary hacking attempts that carried attack payloads (SQL injection and cross-site scripting).

While they don't explicitly attribute these attempts to OpenAI, they claim that they exhibit the same tactics that have been attributed to OpenAI agents in a similar timeframe. OpenAI are currently reviewing the findings and have provided an initial briefing to Canadian officials.

This is another example of Rogue Agents straying away from original goals and using information gathering techniques for attack probes (ASI10), and the use of ordinary web-request tools for SQL injection and XSS payloads (ASI02).

- [AI Agents Targeted U.S. and Canadian Government Websites](https://transluce.org/us-canada-gov)
- [Autonomous AI agents tried to hack US, Canadian government websites](https://www.bleepingcomputer.com/news/security/autonomous-ai-agents-tried-to-hack-us-canadian-government-websites/)
- [AI Agents Aimed SQL Injection at US and Canadian Government Sites](https://www.securityweek.com/ai-agents-aimed-sql-injection-at-us-and-canadian-government-sites/)

## "PixelLeak": coding agents published internal screenshots to public GitHub repos

Developers who get coding agents to make visual changes will often use screenshots as evidence that a change has been made. As you can imagine, if you're doing this as part of your day job, there's IP involved, and that information shouldn't be leaked to the public until it's ready.

However, Glow Labs identified over 13,000 internal images that were published on public GitHub repositories by developers at over 300 organizations.

The issue is that GitHub CLI couldn't attach images to a PR, so agents would work around this by creating public repositories tied to the developer's personal account to host the image. There's an open-source tool called `gitshot` that defaults to public storage, and the agents saved this workaround to a skill. If you have multiple coding agents in your organization, that increased the blast radius.

These screenshots also included billing records, unreleased features, and treasury consoles.

The GitHub CLI added an `--attach` flag through version 2.99.0 (Released 1st September). However, Flow also advises that you need to enforce runtime controls for developer agents to prevent information from being leaked.

These were legitimate tools that agents used (The GitHub CLI, `gitshot`). They just used those tools to publish confidential information. The agents also abused developers' personal GitHub credentials to create public assets outside organizational control (ASI03), and the use of agent-shared skills spread the unsafe pattern across organizations, infecting their supply chain (ASI04).

- [PixelLeak: How AI Agents Exposed Developer Screenshots from Leading Tech Companies](https://www.glow.io/blogs/how-ai-agents-exposed-developer-screenshots-from-leading-tech-companies)
- [AI Coding Agents Exposed 13,000 Internal Images, Including Billing Records, on GitHub](https://thehackernews.com/2026/09/ai-coding-agents-exposed-13000-internal.html)
- [PixelLeak exposes 13,000 internal screenshots on GitHub](https://www.bitdefender.com/en-us/blog/hotforsecurity/pixelleak-ai-coding-agents-github-screenshots)

## GitLab AI Gateway: prompt-template sandbox escape to RCE (CVE-2026-90970)

GitLab warned customers to patch a vulnerability in its AI Gateway product that could let attackers run arbitrary commands on vulnerable instances. AI Gateway gives access to AI-native GitLab Duo features. 

The flaw lies in the prompt template of a custom flow. A custom flow is an AI-powered workflow that users create on the Duo Agent platform to automate multi-step tasks. A logged-in user with Duo Agent platform access could have used the flow to escape the prompt template sandbox via a crafted flow configuration. This escape could be used to execute arbitrary commands on the gateway.

For GitLab-hosted gateways, this vulnerability has already been fixed. However if you host your own gateway, GitLab has advised that you update your environment immediately. GitLab sent that guidance to customers before it published the advisory.

Self-hosted gateways holds signing keys for JWT, which are treated as sensitive credentials, as well as connections to GitLab instances and model providers.

GitLab credits HackerOne user 'invisiblemeerkat' with reporting the flaw. *I wish I had a cool hacker name, any suggestions are welcome. Keep it SFW*.

Currently there are no reported incidents from this disclosure, but I'd keep an eye on those self-hosted gateways. This does have the potential for Unexpected Code execution (ASI05) incidents via the agent flow configuration, and identity and privilege abuses (ASI03) as a low-privileged agent-platform user could end up with the gateway's signing keys and service credentials.

- [GitLab AI Gateway Critical Patch Release: 19.2.4, 19.3.2, and 19.4.1](https://docs.gitlab.com/releases/patches/other-patches/patch-release-gitlab-ai-gateway-19-4-1-released/)
- [GitLab Patches Critical 9.9 AI Gateway Flaw Allowing Command Execution on Self-Hosted Servers](https://thehackernews.com/2026/10/gitlab-patches-critical-self-hosted-ai.html)
- [GitLab warns of critical RCE vulnerability in AI Gateway service](https://www.bleepingcomputer.com/news/security/gitlab-warns-of-critical-rce-vulnerability-in-ai-gateway-service/)

## Official MCP Python SDK: malicious MCP servers can hijack OAuth credentials

Cycode reported an official MCP Python SDK flaw could trick applications into handing over the OAuth credentials it uses to log into a real service. In versions >= 2.0.0a1, < 2.2.0 and >= 1.9.1, < 1.30.0, it could send the client secret, authorization code, and PKCE proof key to a token endpoint that an attacker controls.

When an MCP client needs to log in, it'll ask the server it is connecting to where the authorization server can be find. In the affected versions, a malicious server could point it at a server that an attacker chooses. The client then sends the secret, PKCE proof key and auth code to the attacker. The proof key is a one-time value that's designed to prevent a stolen authorization code from being used, so if that gets handed over, that protection is gone.

No exploits have been reported yet, and this issue was fixed in version 1.30.0 and 2.2.0. However, if you're using affected SDK versions, you could be opening yourself up to Agentic Supply Chain vulnerabilities and Identity and Privilege abuses.

Furthermore, if you use `ClientCredentialsOAuthProvider` or `PrivateKeyJWTOAuthProvider`, you should also pass the `issuer=`. Without this, they'll still follow whichever auth server your MCP server advertises.

- [OAuth client could send credentials to an authorization server chosen by the MCP server](https://github.com/modelcontextprotocol/python-sdk/security/advisories/GHSA-qx49-fqc8-xw99)
- [Cycode Uncovers Account Takeover in MCP Python SDK](https://cycode.com/blog/mcp-python-sdk-oauth-account-takeover/)
- [Official MCP Python SDK Flaw Can Let Malicious Servers Steal OAuth Credentials](https://thehackernews.com/2026/09/official-mcp-python-sdk-flaw-can-let.html)

## Malicious ChatGPT Custom GPT "Plus 5.6" used to deliver a RAT via ClickFix

Malware developers use sponsored Google results as an attack vector to lure unsuspecting victims. Huntress found that attackers are using sponsored results to push a malicious ChatGPT custom GPT named "Plus 5.6", that lures them to a fake Cloudflare CAPTCHA check and make them download and run a remote access trojan (RAT).

Essentially users would search "chatgpt" in Google, and a link to the custom GPTs would appear in the sponsored Google Search result. Those who navigate to the site and complete the CAPTCHA check would be delivered with a ClickFix attack, which tell users to copy-and-paste a command into their terminal.

That triggers a chain that ends up with a remote access trojan being downloaded onto your machine.

Huntress reported the initial finding and OpenAI were able to remove it. However, it didn't take long for another site to be created and pushed through sponsored results (*who at Google verifies these?*).

The second one is still online, and while it doesn't contain the malicious ClickFix lure anymore, it's only a matter of time until a new installer is created.

The general advise here is that any page or website that asks you to run something in the Run dialog, Terminal, PowerShell, or command prompt is asking for trouble. **Don't do it**.

But this is a great example of how agents can exploit the trust of humans (ASI09). Who among us hasn't typed 'chatgpt' into a Google search? Imagine if you're not a technical person, or you just forget to look at the actual web address? If the domain looks official enough, you're going to trust it, and if you're not sure what a download does, are you really going to take the time to stop it?

- [Malicious Custom GPT on chatgpt.com lures users into installing a RAT](https://www.helpnetsecurity.com/2026/09/29/malicious-chatgpt-custom-gpt-malware-via-clickfix/)

## LiteLLM: JWT email_verified account takeover (CVE-2026-93355)

LiteLLM is an open-source gateway that you can use to manage access to OpenAI, Anthropic, and other LLM APIs. The gateway holds API keys, tracks how much you're spending on LLM calls, and controls who can access what.

The folks at OX Research discovered a way to use a legitimately signed login token to authenticate as another LiteLLM user (including admins). The JWT authentication silently falls back to an unverified email claim when a direct identity match misses. This allows the attacker to log in as any user, and then permanently rebind that victim's SSO account to the attacker's own token.

This vulnerability was actually discovered on May 18th, but Ox has *still* not received a response, and it was publicly disclosed on September 14th. There's been no reported exploitations yet, the vulnerability has been confirmed through to version 1.100.1

- [CVE-2026-93355: Account Takeover in LiteLLM](https://www.ox.security/blog/litellm-an-ordinary-login-token-can-become-someone-elses-admin-account/)

## Conclusion

We're only going to see more agents being used in cyberattacks. Whether it's agents going against their intended goals or being directed to be malicious. Agents can also cause trouble when they are acting within their expected behavior, as in the case of PixelLeak.

The agentic supply chain also continues to be a challenge. Agents depend on MCP servers, SDKs, gateways, and shared skills which all can be weak links to be exploited.

If you're developing agents for your organizations, you need to ensure that you're thinking about the underlying agent infrastructure, and implement constraints and guardrails over what your agents can do.

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️