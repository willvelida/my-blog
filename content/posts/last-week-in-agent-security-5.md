---
title: "Last week in Agent Security Update 5: The Pentagon Prompts"
date: 2026-11-02
draft: false
tags: ["OWASP", "Agents", "AI", "Security"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/nyjbums24ih0cqysj6ar.png
    alt: "Spider-Man pointing meme with Cursor and Anthropic pointing at each other."
    caption: "This week: a contractor’s Claude agent allegedly leaks Pentagon network details, Cursor serves a backdoored Claude Code extension, Cathedral’s agents get inside Anthropic, 123 Reg reports an “AI-driven” breach, a worm arms 42 GitHub repos with agent hooks, embedded browsers expose five AI dev tools, and CyberStrikeAI sends 44 agents at Japan.."
---

Last week I spoke at API Days in Melbourne, where I gave a talk on the OWASP Top 10 for Agents and how you can mitigate the risks using tools such as the [Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit) and [Microsoft Entra Agent ID](https://learn.microsoft.com/en-us/entra/agent-id/what-is-microsoft-entra-agent-id).

My talk was scheduled to be in the last timeslot of the day, which was 4.50pm. That's a tough spot for speakers and audiences, as conferences can be a long and tiring day. I was fully prepared to speak to about 2 people in the room who decided not to beat the traffic and get their money's worth.

![Will presenting a slide about Cairn agent attacks at API Days Melbourne.](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/53hn7vi6e7ga5le44uj0.jpg)

*Presenting the OWASP Top 10 for Agents at API Days Melbourne.*

Luckily about 40-50 people showed up just before I was about to present, which injected some much needed energy into the room. I'd given the same talk before at NDC Oslo, but I decided to rebuild the entire slide deck and demo since I only had about 20 minutes to talk (as opposed to 1 hour in Oslo). I'm giving a [longer version of the talk at NDC London](https://ndclondon.com/agenda/applying-the-owasp-top-10-for-agentic-applications-to-your-ai-agents) in January, so if you're in London and are planning to attend, make sure you get your [tickets](https://ndclondon.com/tickets)!

The sample I showcased is available on my [GitHub](https://github.com/willvelida/owasp-agents-sample) if you're interested. There's a couple of things I need to do before I share it more widely, but I'm looking for contributors and feedback, so please take a look and let me know what you think.

# This week's attacks

Big focus on the agentic supply chain this week, as there were several attacks that targeted vulnerabilities in how agents are set up, how extensions are configured for agents, and how agents can accidently executed code just by opening up configuration files in a project.

As always, I’ll point out that I’m not trying to shame any of the companies involved (maybe just the ones who deserve it), nor do I condone any of the methods used in any of these incidents. My goal here is just to spread awareness of the different ways that agents can be exploited, and hopefully trigger a conversation with your teams on how you should protect your agents and the people who use them from potential vulnerabilities.

Here’s some incidents that have happened over the period 26th October to 1st November 2026.

## Trojanised Claude Code extension in Cursor Marketplace

On October 29th, versions 2.1.312 and 2.1.313 of the Clade Code VS Code extension were published to Cursor's VS Code extension marketplace for macOS and Windows that carried a malicious loader embedded in its `extension.js` bundle.

This `extension.js` file carried a small self-contained loader embedded in the legitimate bundle. It was designed to activate after VS Code startups and when its WebView opens, the loader proceeds to add an attacker-controlled key to the user's `~/.ssh/authorized_keys` file. This key is labeled "svc-telemetry" to hide its true intentions. It then attempts to make the victim reachable through SSH across Windows, Linux, and macOS, establishes a reverse tunnel, installs platform-specific persistence and then transmits host recon data back to the reverse tunnel over an unencrypted HTTP request.

![Attack chain for the trojanised Claude Code extension, showing SSH access, persistence, and host reconnaissance.](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj4Ew0Bj3BoXpUeH6TD2jNswGM1qWxJ956j4_sI9otnOjW_hHhyrbfmuhd-kovwe07nO3rsiuAjjOAq1kZXc4x5Gk_g8uqiYAyrm-RCnaGabGsNC-D1W_92UaKoUZ56AGraAszeX_pkW5hMAq1lFGu7VBicGP1jbBqNsd9BhTkjqWJT6CYNE0YcyF-u3mih/s1700-nu-rw-lo-l85-e365/aa.png)

*How the trojanised Claude Code extension plants an SSH backdoor, maintains persistence, and sends host details to the attacker. Credit: [OpenSourceMalware](https://opensourcemalware.com/blog/trojanized-claude-code-extension-plants-persistent-ssh-backdoor), reproduced in [The Hacker News](https://thehackernews.com/2026/10/official-claude-code-vs-code-extension.html).*

The collected data includes public IP addresses, hostname and local usernames, OS and CPU architecture, home directory and current working directory information, global git email from git config files, and environment variables containing things like KEY, TOKEN, SECRET, PASS, AWS, or GCP.

Cursor's security team have published a blog notifying customers of the incident, and stressed that this trojanised extension does not impact other extensions downloaded from the Cursor marketplace, Grok Bot, Grok, or any other of their AI products. In response, Cursor had to rebuild the marketplace and take the server offline so that the reverse shell would no longer work.

Anthropic have stated that this was not a compromise of Anthropic's systems, as the malicious actor used access to Cursor's marketplace to publish a tampered copy of the extension. The malicious update was only distributed through Cursor's marketplace, with no evidence that it reached Open VSX or Microsoft's official VS Code marketplace.

The Microsoft Security Research team are advising affected users to remove confirmed unauthorized SSH keys, tunnel identities, scripts, and startup entries separately. Removing or updating the extension won't remove artifacts written outside the installation directory.

Here, we see a trusted distribution channel for installing extensions for coding agents being exploited by a backdoor trojan (ASI04). Securing the agentic supply chain is a challenge, because there's no real attestation being conducted in this space, and with package managers being administrated by vendors over foundations, the focus on innovation at any cost comes at the cost of security. 

By injecting threats in the agentic supply chain, attackers can use access to developer environments to perform other types of attacks, such as RCE and Identity abuses. Until there's a more unified approach to how we secure the agentic supply chain, we're going to see more attacks like these slip through the cracks.

- [Official Claude Code Extension on Cursor Marketplace Trojanized with Persistent SSH Backdoor: The Hacker News](https://thehackernews.com/2026/10/official-claude-code-vs-code-extension.html)
- [Trojanized Claude Code Extension Delivered Persistent SSH Backdoor to Developer Machines: StepSecurity](https://www.stepsecurity.io/blog/claude-code-extension-trojanized)
- [Security notice — malicious version of Anthropic.claude-code (2.1.312 and 2.1.313) identified in the Cursor VS Code Extension Marketplace: Cursor](https://cursor.com/blog/claude-code-extension-security-notice)

## Accenture contractor's personal Claude Code agent on a Pentagon system

WIRED reported on October 29th that a contractor from Accenture connected their personal Claude Code account to a Department of Defense system which held controlled unclassified information over a period of two months. The agent appears to have sent login credentials and a network map to outside the Pentagon's network.

The system in questions hold sensitive data that needs to be reviewed before it's released (unclassified information, or CUI for short) and it's used by several intelligence agencies. The system runs on NIPRNet, which is the Pentagon's network for unclassified information. It does connect to the public internet, but it has heavy restrictions (as you can imagine).

The contractor gave Claude access to their login credentials so it could write code, and set up cloud servers. This granted the agent admin access, and it also had the ability to talk to the internet to do its work. This resulted in credentials, passwords, and network maps being sent to a third-party website, which the US National Institute of Standards and Technology has flagged as high risk.

What's worse is that Defense Department staff and federal contractors must report cybersecurity incidents within 72 hours, which Accenture didn't do. They also denied any wrongdoing. There's also a political element to this, after the debacle that cause the defense department to [label Anthropic a supply chain risk](https://www.bbc.com/news/articles/cn5g3z3xe65o), staff at the Pentagon may not use Claude.

Whether this is a case of the contractor accidently using their personal account for work use by mistake or something more malicious is unconfirmed at this time. There's no indication at this time that hackers have reached any classified systems, but this does open up the possibility that code operate normally within NIPR and perform malicious acts inside a SIPR network.

Here a human's admin credentials were handed to an unsactioned agent, which then acted under that identity in highly sensitive networks. This opens up the possibility of rogue agents sending data outside a secure network and performing malicious acts inside the secure network.

So if you're contractor working for a defense department, or any other customer in a highly regulated environment, just make sure you're logged in with the right credentials and use the right tools for the job.

- [A Defense Contractor Gave Their Personal AI Agent Access to a Secure Pentagon System](https://www.wired.com/story/a-defense-contractor-gave-their-personal-ai-agent-access-to-a-secure-pentagon-system/)
- [A Claude agent ran inside a Pentagon network for two months, WIRED reports: TNW](https://thenextweb.com/news/accenture-contractor-claude-agent-pentagon-network-wired)
- [Accenture Contractor Allegedly Gave Claude AI Agent Access to Secure Pentagon Network: Cyber Security News](https://cybersecuritynews.com/accenture-contractor-allegedly-gave-claude-access/)

## Cathedral's agent-led intrusion into Anthropic

Remember those douchebags from DOGE? Well they're doing a sequel, and this time they're taking on Anthropic.

The founders of Cathedral informed Anthropic that they had penetrated sensitive internal data and messaging systems through third-party business software. Cathedral is cybersecurity startup founded by former DOGE employees and sells AI-powered security tools to government agencies and businesses.

Anthropic confirmed that the researchers accessed a limited set of sales, HR, and IT systems, including contact information for some customers and partners. Chats with Claude, source code, and model weights were never exposed. Anthropic shut down the affected access points, disabled third-party programs until the issue was resolved, and notified U.S federal authorities.

There's a bit of disagreement between Anthropic and Cathedral as to how long the breach lasted, and what data was breached. Anthropic also accuses Cathedral of violating their bug bounty program's rules through incomplete disclosure, and through targeting third-party providers.

Now, as much as I dislike anyone who's worked for DOGE, if there's a vulnerability in a third-party or external system that you use that could cause a potential breach or security incident, you need to take it seriously. As you can imagine with the actors involved, there's a certain amount of dick measuring going on regarding the extent of the breach (*even a former OpenAI employee is getting involved*), but as much as it pains me to say, integration with third-party applications are a problem when it comes to security, and Anthropic could have done a better job making sure that the external services they use were secure.

Regardless of the rules of a bug bounty program, you can see that it's easy for anyone with an agent to use legitimate tools to cause cybersecurity incidents, whether they are researchers from startups like Cathedral, or malicious state aligned attackers. It's important to factor in how AI is used with your various different systems, as one rogue agent with access to legitimate tools can cause real damage.

*Just going to take a brief break so I can shower off the fact I'm kinda on the side of ex-DOGE employees.*

- [Cathedral Says AI Agents Breached Anthropic in 12 Hours: Claimed vs Verified: Agentbeam](https://agentbeam.com/blog/cathedral-anthropic-12-hour-ai-agent-breach-claimed-vs-verified)
- [AI agents hacked Anthropic, maker of arguably the most capable AI models for cyberdefense: The decoder](https://the-decoder.com/ai-agents-hacked-anthropic-maker-of-arguably-the-most-capable-ai-models-for-cyberdefense/)

## 123 Reg "AI-driven" breach

UK domain registrar 123 Reg disclosed that they were a victim of an AI-driven cyberattack that exposed customer data.

An initial investigation by 123 Reg identified attack servers in both China and Singapore that conducted the attacks. They observed the use of AI attack skills (including copies of Anthropic's open-source skills) and the use of ARTEX, an open-source AI pentest framework.

*If you're a regular reader of the series, you would have seen attackers using ARTEX for attacks before*.

123 Reg suspect that a Chinese-state group who conducted similar attacks on finance companies in South Korea are behind the attack. Nothing has been confirmed yet, and 123 Reg have notified the FTC, UK, and EU regulators. Customers have been notified that information such as company VAT (tax) numbers, names, phone numbers, account creation dates etc. have been compromised.

- [123 Reg discloses 'AI cyberattack' on customer data, suspects China nexus: The Register](https://www.theregister.com/cyber-crime/2026/10/28/123-reg-discloses-ai-cyberattack-on-customer-data-suspects-china-nexus/5305936)

## testmetest worm plants agent hook files in 42 GitHub repos

The folks at JFrog Security identified 42 GitHub repositories across 11 owners connected to the `testmetest` npm campaign. Essentially, the `testmetest` is a worm that planted configuration files in GitHub repositories so that when a developer opened a project in an AI coding tools, it would execute the attackers code.

![Diagram linking malicious testmetest npm packages, stolen credentials, hijacked GitHub repositories, and malicious package publishing.](https://research.jfrog.com/img/RealTimePostImage/post/beyond-testmetst/diagram.png)

*The testmetest campaign links malicious npm packages to credential theft, hijacked GitHub repositories, and further malicious releases. Source: [JFrog Security Research, Beyond testmetest](https://research.jfrog.com/post/beyond-testmetest-42-hijacked-github-repos/).*

The attackers used stolen credentials to take over the target's real GitHub account (including one organizational account!) and used scripts to copy old commit messages and dates so that the attack changes would blend in. Each repository was infected with a set of configuration files that pointed at a single script `.github/init.js`.

The attack would change depending on the tool used:

- In **Claude Code**, a `SessionStart` hook in `.claude/settings.json` would run `node .github/init.js` whenever a session starts.
- In **Gemini CLI**, a similar trigger exists in `.gemini/settings.json`.
- In **Cursor**, the script would trigger in `.cursor/mcp.json` and `.cursor/rules/setup.mdc`.
- In **VS Code**, a task would run in `.vscode/tasks.json` whenever the folder is opened.

The tools simply follow project settings they trust by default and runs in the background, so the developer would see nothing. The `init.js` would then proceed to steal GitHub tokens, swap them for npm publishing rights, and then publish malicious packages whenever someone installs them. One infected developer would become a new way to infect others. The file was over oversized, which kept the file payload out of GitHub's code search.

It's a novel method to hijack the supply chain. Poisoned repositories that contained agent configuration that tools trust by default is a simple but effective way to deliver a vulnerability. Once the workspace opens, attackers can run code that performs a variety of malicious activities.

- [Beyond testmetest: Native Addon Camouflage and 42 Armed GitHub Repositories Reveal a Hidden npm Publisher: JFrog Security Research](https://research.jfrog.com/post/beyond-testmetest-42-hijacked-github-repos/)

## "Embrace" embedded-browser flaws in five AI dev tools

The team at Forever Security have discovered vulnerabilities in Cursor, GitHub Copilot, Claude Desktop, VS Code, and Codex that can allow attackers to execute code on host machines, read and steal files, or spoof browser address bars.

If you've been using AI coding assistants recently, you would have noticed that most of the tools now have their own built-in browsers so that the agent can preview apps and read web pages. However, in this case each tool gives its own browser powers a normal browser may block, meaning that the browser is a way to get access into the agent and the host machine.

In Cursor, the built-in browser lets every page send messages to the app. A hostile page could use this to slip hidden text into the agent's message box, and hijack the goal of the agent. In GitHub Copilot, the preview tab could tell the app what the address was, but Copilot would only check if the text *looked* like a web address, not where the page really came from. In Claude Desktop, Codex, and VS Code, a local HTML file opened in the tool can read other files. Now in both Claude and VS Code, the workspace trust boundary applied to the web pages, but not the web workers. So when those pages load, the web workers could read anything. The whole disk is readable, which can include SSH keys, `.env` files and tokens.

No CVEs have been raised because of this, and the worrying thing here is that these tools are being fixed quietly or dismissed as outside their trust models. Context from a malicious web page could easily redirect coding agents to perform malicious acts. There's also the human-agent trust exploitation issue, as a spoofed address bar abuses the trust that developers have placed in the tool, and the agent's browser.

It's disappointing that these companies either quietly fix these vulnerabilities without giving developers the heads up, or they just ignore or dismiss it entirely. 

- [Researchers find bugs in 5 AI developer tools, with OpenAI and Microsoft flaws still unpatched: The Register](https://www.theregister.com/security/2026/10/29/researchers-find-bugs-in-5-ai-developer-tools-with-openai-and-microsoft-flaws-still-unpatched/5306508)

## CyberStrikeAI: 44 agents against Japanese organisations

Cybersecurity researchers from TrendAI have disclosed details of AI-driven cyberattacks targeting organizations in Japan. TrendAI analyzed three attacks used by threat actors to conduct malicious operations.

The first used a tool called CyberStrikeAI, which is an open-source penetration testing tool, to automate attacks into a repeatable workflow, including activities such as reconnaissance, vulnerability detection and validation, exploitation, command-and-control, data exfiltration, and automated reporting. The work was delegated among 44 agents to handle various aspects of the attacks. At least 19 organizations where subjected to these attacks.

The second was characterized as a human-led attack using LLM models to accelerate planning and tool development. Analysis of these artifacts seem to indicate that the campaign was designed to target AI workflow platforms and inference services in an attempt to obtain a large amount of inference capacity at a low cost. One instance targeted a geographic information system server exposed with default credentials, resulting in data exfiltration.

In the third attack, AI was used to develop the operator's tools and scripts. The attack then used these tools to target edge devices from Cisco, Citrix, F5, and Yamaha using published vulnerabilities, management functions, or credentials present in configuration files.

It's unknown at this stage if these attacks share the same operator, or if these attacks have targeted other regions.

- [CyberStrikeAI Uses 44 AI Agents to Automate Hacking Operations Against Japan Targets: The Hacker News](https://thehackernews.com/2026/10/cyberstrikeai-uses-44-ai-agents-to.html)

# Conclusion

Quite a few attacks last week focused on the developer's agent setup. Agent configuration files are acting as executable code inside coding assistants that most tools trust by default. Whether that's through backdoors in official marketplaces, or malicious hook files in GitHub repositories that run malicious code as soon as projects are opened, there are gaps in the agentic supply chain that need urgent attention.

It's not good enough for Big Tech firms and Frontier labs to quietly patch systems or ignore threats as outside their trust boundary without issuing advisories. The only way to get these companies to act properly is through government intervention and legislation. Which I don't see happening anytime soon.

*As always, opinions are my own*

Remember, I'm not trying to scare folks or discourage people from using these tools. Just be mindful of what your agent is installing. Treat agent configuration like any other code and review it like any other executable. Lock down where agent extensions and models come from, and restrict agent traffic to approved model endpoints.

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️