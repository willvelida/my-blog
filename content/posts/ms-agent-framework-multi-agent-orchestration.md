---
title: "Orchestrating a multi-agent solution with the Microsoft Agent Framework"
date: 2026-07-21
draft: false
tags: ["Microsoft Agent Framework", "AI Agents", ".NET", "AI"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/vkrb2slxsrn8tr6s5uf8.png
    alt: "Orchestrating a multi-agent solution with the Microsoft Agent Framework"
    caption: "The Microsoft Agent Framework ships five orchestration patterns for coordinating multiple agents: sequential, concurrent, group chat, handoff, and Magentic."
---

The World Cup is over, and despite England losing in the semi-finals, there were plenty of successful hatewatches to be enjoyed (Thank you Belgium and Spain for your service). If you're sad that it's all over, the Premier League is just over 1 month away.

Last season, my team Arsenal finally won the title after a 22-year drought. Arguably, we should have won the EFL Cup and the Champions League. But as any Arsenal fan will tell you, we do have a tenancy to lead by a goal, and just kill the game until the final whistle. Our defence is probably the best in the world (Our biggest defeat last season was a 2-0 loss to Man City), but our attacking could, and should be much better.

As a manager, what approach do you take to regain the title?

I've been learning about multi-agent architectures, and how we can orchestrate between multiple agents, so I'd thought it'd be a bit of fun to see if I could build a multi-agent system that could provide some insights as to who Arsenal *might* want to sign to defend their title, as well as learn different multi-agent orchestration patterns using the Microsoft Agent Framework.

If you want to follow along with the code, please check out the sample on my [GitHub](https://github.com/willvelida/football-scout)

## What is the Microsoft Agent Framework?

The Microsoft Agent Framework (MAF) helps developers build agents that can process inputs, make decisions, and execute tasks autonomously by using LLMs. It's open source, and you can [build agents using MAF in .NET, Python, and now Go!](https://learn.microsoft.com/en-us/agent-framework/overview/?pivots=programming-language-csharp)

If you're new to MAF, I've written a [few blog posts](https://www.willvelida.com/tags/microsoft-agent-framework/) in the past that cover the basics, and how I use MAF to build my own health agent.

## Agents vs workflows

Agents are backed by a LLM and has access to various tools that help the agent accomplish tasks. The steps that an Agent takes are dynamic and determined by the LLM based on the context of the conversation and the tools available.

Within MAF, we can build complex workflows that include AI Agents as components. They're designed to handle complex business processes that can include multiple agents, human-in-the-loop interactions, and integrations with external systems. The workflow is explicitly defined, which can give us more control over the overall execution path.

Some of the key features that MAF workflows include are:

- **Type Safety**, where strong typing ensures messages flow correctly between components with validation that help prevent runtime errors.
- **Control Flow**, a graph-based architectures that model the complex workflow with executors and edges.
- **External Integration**, built-in request/response patterns that enable integrations with external APIs and human-in-the-loop scenarios.
- **Checkpointing**, where you can save workflow states via checkpoints. Useful for when you need to recover or resume long-running processes.
- **Multi-Agent Orchestration**, built-in patterns for coordinating multiple AI agents, including sequential, concurrent, hand-off, and magentic. Which we're covering in this blog post!

## Why we need orchestration in multi-agent architectures

Single agents can be limited in complex scenarios. They are constrained by their instructions or the capabilities of the underlying LLM. When we need to deploy multiple agents within our architectures, orchestration can help us in the following ways:

- Assign distinct skills, responsibilities, or perspectives to each agent.
- Combine outputs from multiple agents to improve decision-making and accuracy.
- Coordinate steps in a workflow so work from one agent can build on the last.
- Dynamically route control between agents based on context or rules.

That being said, more agents, more problems. Just because we have a bunch of agents in our architecture that have different skills, responsibilities, or perspectives, doesn't necessarily mean that we avoid the limitations of using a single agent. 

### Before we start

The agents work off two JSON files in the demo repo. `arsenal-squad.json` is the current squad, 24 players plus the money I've got to spend. `transfer-targets.json` is the shortlist, 40 players: the senior names Arsenal have actually been linked with, and a longer list of wonderkids and bargains worth a look.

Both files share the same shape, so every agent reads them the same way. Here's the squad file, with the metadata notes trimmed and one player shown:

```json
{
  "team": "Arsenal",
  "list": "current squad",
  "source": "EA Sports FC 26 player ratings",
  "asOf": "2026-07-19",
  "finances": {
    "remainingTransferBudget": 110000000,
    "remainingWageBudget": 380000
  },
  "players": [
    { "name": "Bukayo Saka", "nationality": "England", "position": "RW", "club": "Arsenal", "age": 24, "overall": 88, "current": 87, "predicted": 89, "pace": 84, "shooting": 82, "passing": 85, "dribbling": 88, "defending": 60, "physical": 73, "value": 140000000, "wage": 300000 }
  ]
}
```

Each player has the same fields. The `overall` and the six face-stats (pace, shooting, passing, dribbling, defending, physical) come from EA Sports FC 26's player ratings. `current` and `predicted` are from fifacm.com, which tracks a rating now and where it's heading, so a scout can tell a finished article from a project. `value` and `wage` are my own real-world-scale GBP estimates, calibrated to the budget, since EA and fifacm don't publish transfer fees or wages.

The `finances` block defines what the transfer and wage budgets are: 110 million to spend and 380k a week respectively (Taken from the starting amount in EA FC 26 Career Mode). The window planner on the Magentic desk has to keep its shortlist inside both, which is the fun part, because the best player is rarely the one you can afford.

`transfer-targets.json` is the same shape without the budget block. The senior names (Julián Álvarez, Nico Williams, Bruno Guimarães) come from BBC Sport's Football Gossip column, filtered to the Arsenal links from June 2026 on. The prospects come from the fifacm wonderkids and bargains lists. All players statistics were taken from their EA card and a fifacm rating.

*N.B, while I've tried to make the data as realistic as possible, it's still demo data. If you see your favorite player and think he's underpriced or underpaid, it's only a demo. Just have fun with it. There's also a couple of players who have already signed for new clubs. Rodgers to Chelsea. Seriously. How much money do you think they're paying him to NOT play in the Champions League? Absolute Boehly Ball madness 😂*

I've also created a markdown file called `brief.md`, which is my guess of what Mikel Arteta will be facing when everyone returns from the World Cup.

```markdown
# The Brief

You are Arsenal's manager. You have just won the 2025/2026 Premier League title, and are looking to strengthen your squad for further glory.

You are concerned about the following:

- Ageing squad. Some players are reaching 30 years old, or are over 30 years old. Getting rid of some of these players may be needed to generate transfer funds and free up wage budgets.
- Attacking ability. While you won the league, you are concerned how some wins were too close, and that better attacking options would have made some wins more devisive. This is particularly the case in big games, as Arsenal lost both the League Cup final and Champions League Final.
- Defensive options. While the center back position is solid, you are concerned about your wing backs. While Jurriën Timber is your top RB, and Myles Lewis-Skelly is a talented prospect for the future, you may need to strengthen this area.

The following players are indispensible:

- Bukayo Saka
- Declan Rice
- William Saliba
- Martin Ødegaard
- David Raya

The following players are considered dispensible:

- Leandro Trossad
- Kepa
- Christian Nørgaard
- Gabriel Jesus

For last season's results, use the following resource: https://en.wikipedia.org/wiki/2025%E2%80%9326_Arsenal_F.C._season
```

## The five orchestration patterns

MAF provides orchestration patterns directly via the SDK:

- **Concurrent orchestration**. This is when we broadcast the same task to multiple agents at once and collect their results independently. This is useful when we need agents to conduct parallel analysis, independent subtasks, or ensemble decision making.
- **Sequential orchestration**. This is when we pass the output from one agent to the next in a fixed order. Handy, if we have step-by-step workflows, pipelines, or progressive refinement.
- **Handoff orchestration**. This is when we dynamically transfer control between agents based on context or rules. Great for escalation, fallback, and expert routing scenarios where one agents needs to work at a time.
- **Group chat orchestration**. This is when we coordinate a shared conversation among multiple agents (and optionally humans, because we still matter 😅), managed by a chat manager who chooses who speaks next. This is great when we need agents to brainstorm, conduct collaborative problem solving, and build consensus.
- **Magnetic orchestration**. This is a manager-driven approach that plans, delegates, and adapts across specialized agents. This is suited to complex, open-ended problems where the solution path evolves.

While these orchestration patterns exist, the typical orchestration flow looks like this:

1. We define our agents and describe their capabilities.
2. Select and create an orchestration pattern.
3. Configure callbacks or transforms for custom input and output handling.
4. Start a runtime to manage execution.
5. Invoke the orchestration with your task.
6. Retrieve the results.

### Sequential

Sequential orchestration means that agents are arranged in a logical pipeline where each agent processes the task one after the another. The output from one agent is used as input for the next.

This is great for worflows where each step depends on the previous one. For our scouting example, say we want to scout a particular player, and we have three agents that analyze the player: one for determining what type of player they are based on their stats, another who weighs that profile against their asking transfer fee and wage demands vs our budget, and finally an agent that write the complete scouting profile for the player with a recommendation:

```csharp
AIAgent statReader = agents.CreateAgent(
            name: "StatReader",
            description: "Turns raw ratings into a playing profile.",
            instructions:
                "You are a football scout who reads EA Sports FC ratings. Given one player's data, describe their " +
                "playing profile in 3-4 sentences: what the pace, shooting, passing, dribbling, defending and " +
                "physical stats tell you, their age, and what the current-to-predicted overall says about their " +
                "growth. Do not discuss money. Output only the profile.");

        AIAgent valuer = agents.CreateAgent(
            name: "Valuer",
            description: "Weighs the profile against fee, wage and budget.",
            instructions:
                "You are a recruitment analyst. You receive a player's playing profile followed by their financial " +
                "data and Arsenal's remaining budget. Weigh the transfer value and weekly wage against that budget " +
                "and judge whether the fee is fair for the profile. Reference the actual numbers. Give a 2-3 " +
                "sentence value assessment.");

        AIAgent reportWriter = agents.CreateAgent(
            name: "ReportWriter",
            description: "Writes the final scouting report with a verdict.",
            instructions:
                "You are the chief scout. Using the playing profile and the value assessment already in the " +
                "conversation, write a short plain-English scouting report of about six sentences and end with a " +
                "verdict line that is exactly one of 'VERDICT: SIGN', 'VERDICT: MONITOR' or 'VERDICT: PASS', " +
                "followed by one line of reasoning.");

Workflow workflow = AgentWorkflowBuilder.BuildSequential([statReader, valuer, reportWriter]);
```

Let's run this on Nico Williams and see what our agents say! (we've been linked on and off to him for a while now, and personally I'd love to see him sign for us)

```bash
┌─ StatReader
│ Nico is an explosive, direct left winger — his 93 pace and 87 dribbling make him a constant threat in 1v1s and on the counter. Shooting 76 and passing 80 mean he can both finish and deliver qualityfinal balls from wide areas, offering reliable end product rather than elite playmaking. Low defending (36) and only average physicality (66) point to limited defensive work-rate and susceptibility in physical duels, so he fits best as an attack-first winger in systems that protect him defensively.At 24 with a current 85 overall rising to 88, he’s entering his prime and still has clear upside tobecome a top-tier wide attacker.

┌─ Valuer
│ Paying £55m (50% of your remaining £110m transfer budget) with a £140k/wk wage (≈37% of the £380k/wk headroom) is financially feasible — you’d retain £55m transfer budget and £240k/wk wage headroomafter the deal. Football-wise, a 24‑year‑old with current 85→predicted 88, elite pace (93) and dribbling (87) plus decent end product (76 shooting, 80 passing) is strong value for that price; his lowdefending (36) and moderate physicality (66) limit him defensively, but the fee/wage represents fair-to-good value given his upside and role as an attack‑first winger.

┌─ ReportWriter
│ Nico Williams is an explosive, attack-first left winger whose elite pace (93) and excellent dribbling (87) make him a constant 1v1 and counter-attacking threat. His shooting (76) and passing (80) mean he provides reliable end product from wide positions rather than creative midfield play. He offers clear upside at 24 with current 85 and a predicted 88, so he should still improve into a top-tierwide attacker. Defensively he is limited (36) and only average physically (66), leaving questions over his contribution in our high-press and transition phases. The reported £55m fee and £140k/wk wage are affordable within our remaining budget and headroom, so the deal is financially feasible. Given Arsenal already have two high-quality wide forwards and demand defensive work-rate from wide players, he’s a valuable option but not an urgent buy.
│ 
│ VERDICT: MONITOR
│ Good value and high upside as a pure attacker, but limited defensive fit and squad redundancy mean we should keep him on the shortlist rather than push to sign now.
```

Each agent passes their output as input in the pipeline, and produces a verdict of whether we should sign him or not.

*N.B. Hard disagree on the MONITOR verdict here. With Mr Clutch Leandro Trossard gone, we need someone like Nico Williams to boost our attack, especially after winning the World Cup!* 

You should choose sequential orchestration in your workflow if your pipeline is made up of multiple steps that happen in a specific order, where each step relies on the one before it.
![Sequential orchestration: a transfer target flows top to bottom through StatReader (ratings to profile), then Valuer (profile vs budget), then ReportWriter, producing a scouting report](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/j2qrgtr2hb11g16tasjr.png)

If you have tasks that can't be performed at the same time, and have to build on each other, sequential orchestration is the pattern that you should use. If you need tasks to run in parallel, then concurrent orchestration would serve you better.

### Concurrent

So like I said, concurrent orchestration lets multiple agents run in parallel on the same task. Each agent handles the task independently, and then their outputs are aggregated.

If we have tasks that can be run at the same time, we can use a fixed group of agents, or bring them in dynamically, to solve the task. This is particularly useful when we need different agents with different skills that can work independently on a common goal.

For our example, we have 4 expert scouts, and we want them to assess a particular target on the shortlist. We can define the agents like so:

```csharp
AIAgent attackScout = agents.CreateAgent(
            name: "AttackScout",
            description: "Judges attacking output.",
            instructions:
                "You are an attacking scout. Judge ONLY the target's attacking threat from their pace, shooting and " +
                "dribbling. Two or three sentences, then end with a sub-score out of 10 for attacking output.");

        AIAgent defenceScout = agents.CreateAgent(
            name: "DefenceScout",
            description: "Judges defensive contribution.",
            instructions:
                "You are a defensive scout. Judge ONLY the target's defensive work and duels from their defending " +
                "and physical stats. Two or three sentences, then end with a sub-score out of 10 for defence.");

        AIAgent fitScout = agents.CreateAgent(
            name: "SystemFitScout",
            description: "Judges fit for Arsenal's system.",
            instructions:
                "You are a tactical scout for Arsenal. Judge how the target's position and profile fit Arsenal's " +
                "current squad shape below. Two or three sentences, then end with a sub-score out of 10 for fit.\n\n" +
                "Arsenal squad:\n" + data.SquadSummary());

        AIAgent valueScout = agents.CreateAgent(
            name: "ValueScout",
            description: "Judges value for money.",
            instructions:
                "You are a value scout. Judge value for money from the target's transfer value, weekly wage, age and " +
                "predicted growth against Arsenal's budget (" + data.FinancesSummary() + "). Two or three sentences, " +
                "then end with a sub-score out of 10 for value.");
```

We use the concurrent orchestration pattern when we need different approaches to solve the same problem. Instead of them working one after the other, the agents will fan-out and work at the same time.

Results from each agent are then combined to make a final answer. We define the `Workflow` and pass our agents to it. We execute our workflow and then fan-back into a PanelChair agent, who reduces their four assessments into a single recommendation:

```csharp
Workflow workflow = AgentWorkflowBuilder.BuildConcurrent([attackScout, defenceScout, fitScout, valueScout]);

AIAgent chair = agents.CreateAgent(name: "PanelChair", description: "Merges the panel into one recommendation.", instructions: "...");
```

Let's judge Bruno Guimarães (someone who we've been linked with for ages...). Do our agents think he'd be a good fit for the Arsenal?

```bash
┌─ AttackScout
│ Guimarães offers a strong attacking profile for a central midfielder: his 84 dribbling provides excellent on-ball progression and ability to beat opponents in tight spaces, while 75 shooting makes him a real threat from late runs and long-range efforts. His 66 pace isn't electric but is sufficient when paired with close control and agility, so he compensates for lack of top speed by creating and finishing chances through technique. Attacking output: 8/10

┌─ DefenceScout
│ With 79 defending and 81 physical Bruno is a strong, aggressive ball-winner who wins tackles and aerial/body duels reliably and shields the back line well in central areas. Limited top-end pace (66) can hinder recovery against quick transition plays, but his strength and stamina make him a consistently effective presence in midfield duels.  
│ Defence: 8/10

┌─ ValueScout
│ Bruno is an immediate-quality, versatile 86-rated midfielder who would slot straight into Arsenal's XI and shore up control in midfield; his technical and defensive numbers match the profile of a ready-made starter. Financially he’s affordable within Arsenal’s remaining £110m transfer pot and £380k/wk wage headroom (would cost £70m and £160k/wk), but at 28 with no projected growth and eating a large slice of the budget he offers limited long-term upside.
│ 
│ Value sub-score: 7/10.

┌─ SystemFitScout
│ Bruno Guimarães would slot neatly as a dynamic interior in our 4-3-3, offering ball progression, driving runs and solid defensive work that complements Declan Rice and gives tactical flexibility alongside Ødegaard or Zubimendi. At 28 and on a sizeable price/wage, he immediately upgrades depth andmidfield control but overlaps with existing options and could block younger development, so he's useful but not an essential long‑term priority. 
│ 
│ 7/10

   · PanelChair [gpt-5-mini]

┌─ PanelChair (aggregator)
│ MONITOR

His attacking 8/10 and defensive 8/10 underline he is an immediate, two-way starter who fits our 4-3-3 and complements Declan Rice/Ødegaard with ball progression and midfield protection. The overall 7/10 reflects that he is useful but not essential — he overlaps with existing options and risks blocking younger development at age 28. The value 7/10 (price/wage vs no growth) means we keep him on the shortlist and only push to sign if the fee/wage comes down or a long-term younger target cannot besecured.
```

From the output, we can see that the expert agents have each given their recommendation on Bruno Guimarães from their perspective (the fan-out) and have fed their results to our `PanelChair` agent (fan-in), who has recommended that we monitor the situation.

![Concurrent orchestration: a transfer target fans out to four scouts (attack, defence, system fit, value) running in parallel, then fans in to a PanelChair that merges their four scores into one recommendation](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/1um19mgqrr9wqztbms77.png)

### Group chat

Group chat orchestration essentially models a collaborative conversation. Multiple AI Agents can participate in a conversation alongside humans to work on tasks such as debates, or collaborative problem-solving.

A central chat manager controls the flow, and decides which agent responds next in the conversation, and when to request inputs from a human.

Group chats can support free-flowing iteration conversation to formal workflows with defined roles and approval steps. This is great for human-in-the-loop setups where humans have to approve an idea or proposal from an agent.

In group chat orchestration, I'd recommend that agents in the conversation don't directly change running systems at all. Just let them contribute to the conversation.

In our example, let's say we have two agents: one who wants to find the best available player from the shortlist and make a case why Arsenal should sign him, and another who will look at the squad, find the biggest gap in the squad and recommend that Arsenal sign that player instead.

![Group chat orchestration: the BestAvailable and SquadNeeds agents debate in turns, moderated by a round-robin chat manager, producing a shortlist recommendation](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/elain7erwlviwjohxmqf.png)

In the code, it looks like this:

```csharp
AIAgent bestAvailable = agents.CreateAgent(
            name: "BestAvailable",
            description: "Backs the highest-rated target.",
            instructions:
                "You argue Arsenal should sign the best available player: the shortlist target with the highest " +
                "CURRENT rating that the transfer budget allows. Two or three sentences per turn, name your pick, " +
                "and rebut the squad-needs case. Stay within the budget.\n\n" +
                "Shortlist:\n" + data.TargetsSummary() + "\n\n" + data.FinancesSummary());

        AIAgent squadNeeds = agents.CreateAgent(
            name: "SquadNeeds",
            description: "Backs the biggest squad need.",
            instructions:
                "You argue Arsenal should spend on their thinnest position rather than the flashiest name. Weigh the " +
                "club's stated priorities in the manager's brief below, pick the shortlist target that best fills the " +
                "biggest need for a sensible fee, and where it helps the budget, name an ageing squad player to sell " +
                "to fund it. Name your pick and rebut the best-available case. Two or three sentences per turn. Stay " +
                "within the budget.\n\n" +
                (string.IsNullOrEmpty(data.Brief) ? "" : "Manager's brief:\n" + data.Brief + "\n\n") +
                "Arsenal squad:\n" + data.SquadSummary() + "\n\nShortlist:\n" + data.TargetsSummary() + "\n\n" +
                data.FinancesSummary());

        Workflow workflow = AgentWorkflowBuilder
            .CreateGroupChatBuilderWith(participants => new RoundRobinGroupChatManager(participants)
            {
                MaximumIterationCount = 4,
            })
            .AddParticipants([bestAvailable, squadNeeds])
            .Build();
```

We create our agents with different instructions, and then add those agents to the group chat as participants `AddParticipants()`. We can further customize the group chat manager if we need to. We can control how agents are selected, when humans need to provide input, how results are summarized and filtered, and when to terminate the conversation.

Our response looks like this:

```bash
┌─ BestAvailable
│ Pro pick: Julián Álvarez. He’s the highest-current-rated target Arsenal can afford (cur 86, value £85m, wage £250k/wk) and immediately upgrades our goalscoring, pressing and rotation up front — that marginal goals boost protects a title defence more than another youth CB. Signing a striker of Álvarez’s level is higher-impact than doubling down on depth at centre-back, because goals win tight title races.
│ 
│ Con pick: Pau Cubarsí. Arsenal should prioritise a top young left-footed centre-back (value £75m, wage £90k/wk) to future-proof the backline and preserve Saliba long-term; elite defensive continuity is rarer and harder to replace than rotating forwards. Álvarez is a luxury given we already have Gabriel Jesus as cover, and spending big on a 26‑year‑old forward risks blocking younger attacking talent.
│ 
│ Rebuttal (pick: Julián Álvarez). Álvarez fits the budget and wage headroom and offers immediate, non-redundant upgrade — he brings different movement, finishing and link-up that raises the whole attack and mitigates injury risk to Jesus; that direct points impact is decisive for a reigning champion. Defensive depth exists (White, Saliba, Gabriel and youngsters), whereas a high-end striker producing 15–25 goals a season is the faster, clearer way to protect the title.

┌─ SquadNeeds
│ Pick: Malo Gusto. He directly fills the thinnest slot in the squad — right wing‑back cover behind Timber — at a sensible market price (~£40m, £80k/wk) and brings the pace and one‑v‑one ability we lacked in big finals; he’s an immediate tactical upgrade that preserves our pressing shape without blowing the transfer or wage budget. To help the books and wage structure, sell Gabriel Jesus (estimated £35m fee, frees ~£265k/wk) — he’s listed as dispensable and offloads an ageing high earner while keeping Gyökeres, Havertz and Kai as forward options.
│ 
│ Rebuttal of Julián Álvarez: Álvarez is a high‑impact striker but arguably redundant given our depth up front and would cost ~£85m plus ~£250k/wk, which would seriously erode wage flexibility and leave the wing‑back weakness unaddressed — that structural imbalance cost us in decisive games. A £40m right‑back buys immediate, system‑wide gains (defensive solidity, better transitions and wide chance creation) that protect a title defence more reliably than another high‑cost forward.
│ 
│ Rebuttal of Pau Cubarsí: Pau is a great long‑term CB but at ~£75m he’s an expensive fix for a position where Saliba, Gabriel and Piero (plus Mosquera rising) already provide strong short‑to‑medium term cover. Prioritising a top wing‑back delivers bigger immediate returns per pound and addresses the specific tactical vulnerability flagged in the manager’s brief.
```

As you can see, the agents have discussed potential transfer targets, arguing their case based on their instructions. The agents take their turns within the conversation, putting forward their own arguments and providing counter arguments to the other.

### Handoff

In handoff orchestration, we let agents transfer control to one another based on the task they are trying to accomplish, or a request from a human user.

Essentially, each agent "handsoff" the conversation to an agent with the right skillset, ensuring that the best-suited agent is handling each part of the task.

![Handoff orchestration: a question feeds a triage agent that hands off to one of four position specialists (goalkeeper, defence, midfield, attack) and back, with an optional human in the loop, returning the answer](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ec2wt7gtzjhinapi3uco.png)

For our Arsenal transfer room, we create agents who are specialized in a certain position (goalkeeping, defence, midfield, and attack). We create a triage agents who takes our input, and then "handsoff" the task to the appropriate the agent:

```csharp
        AIAgent triage = agents.CreateAgent(
            name: "triage",
            description: "Routes questions to the right position specialist.",
            instructions:
                "You are the transfer desk triage. Read the question and hand off to exactly one specialist: " +
                "goalkeeping, defence, midfield or attack. Do not answer the question yourself; ALWAYS hand off.");

        AIAgent goalkeeping = Specialist(agents, data, "goalkeeping", "Goalkeeper");
        AIAgent defence = Specialist(agents, data, "defence", "Defence");
        AIAgent midfield = Specialist(agents, data, "midfield", "Midfield");
        AIAgent attack = Specialist(agents, data, "attack", "Attack");

        Workflow workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triage)
            .WithHandoffs(triage, [goalkeeping, defence, midfield, attack])
            .WithHandoffs([goalkeeping, defence, midfield, attack], triage)
            .Build();
```

This pattern is great when we have tasks that need specialized knowledge or tools, but the order of agents that are needed to solve the task isn't known in advance, or if requirements emerge dynamically during processing.

Let's use an example of seeing which goalkeeper from the shortlist would be best for Arsenal to sign. We'll ask our triage agent that question, who will route it to our goalkeeping specialist:

```bash
Your question [Enter for: Which goalkeeper on the shortlist is the best value for money?] > 
Routing: "Which goalkeeper on the shortlist is the best value for money?"

┌─ goalkeeping
│ Best value: Mike Penders (Strasbourg).
│ 
│ Why:
│ - Fee £12m, wage £20k/wk (≈£1.04m/yr) → estimated 5‑yr cost ≈ £17.2m.
│ - Predicted OVR 86 (current 77) — highest upside ( +9 ).
│ - Cost per predicted OVR ≈ £17.2m / 86 ≈ £0.20m per OVR point.
│ 
│ Comparisons:
│ - Guillaume Restes: fee £25m, wage £30k/wk → 5‑yr cost ≈ £32.8m; predicted OVR 85 → ≈£0.39m/OVR.
│ - Maarten Vandevoordt: fee £18m, wage £25k/wk → 5‑yr cost ≈ £24.5m; predicted OVR 85 → ≈£0.29m/OVR.
│ 
│ Bottom line: Penders delivers the highest projected quality for the lowest total cost — best value for money.
```

Handoff orchestration provides us with a flexible way to route tasks to the proper agent, ensuring that agent is the best one to handle the task.

### Magentic

Magentic orchestration is a flexible, general-purpose multi-agent pattern designed for open-ended tasks that require dynamic collaboration.

A Magentic manager coordinates a team of specialized agents, and decides who should act next based on the context as it evolves, the capability of the agents, and how the task is progressing. This pattern is based on [Magentic-One](https://www.microsoft.com/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/) from Microsoft Research.

![Magentic orchestration: the WindowPlanner manager runs a plan-and-progress loop. It builds a task ledger, updates a progress ledger, then checks whether the task is complete (yes, report the final transfer plan), whether progress is being made, and whether the stall count is over two (yes, re-plan the task ledger), delegating to the Scout and Analyst agents each round](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/167ssv7y9ig6q3hzop6d.png)

To illustrate this, imagine Arsenal's manager (Mikel Arteta), is working alongside a scout and a financial analyst to coordinate the transfer window.

In the magentic orchestration, the manager uses a reasoning model (Phi-4) to build the plan, judge progress towards the plan, pick the next speaker, and decide when the task is completed. These are all emitted as JSON ledgers, so we need a reasoning model to do this.

The scout and financial analyst are just workers who do repeatable tasks (suggest players, basic arithemtic, etc.), so we can use a cheap GPT model for those agents.

```csharp
// Mixed models: the manager plans on a heavy reasoning model, the workers run on a fast one.
        AIAgent manager = agents.CreateAgent(
            name: "WindowPlanner",
            description: "Plans and coordinates the transfer window.",
            role: ModelRole.Reasoning,
            instructions:
                "You coordinate Arsenal's transfer window. Delegate to Scout (who recommends who to sell and who to " +
                "sign) and Analyst (who nets the sales and signings against the budget). Keep a running tally, and " +
                "finish with a final plan listing outgoings (sales) and incomings (signings) that respects the budget " +
                "and wage limits after sales.");

        AIAgent scout = agents.CreateAgent(
            name: "Scout",
            description: "Recommends targets that fill squad gaps.",
            role: ModelRole.Fast,
            instructions:
                "You are Arsenal's scout. When asked, recommend both (a) shortlist targets to SIGN that fill the " +
                "squad's weakest positions, and (b) current squad players to SELL \u2014 ageing (30+) or surplus \u2014 to " +
                "raise funds and free wages, each with a one-line reason. Leave the budget arithmetic to Analyst.\n\n" +
                "Arsenal squad:\n" + data.SquadSummary() + "\n\nShortlist:\n" + data.TargetsSummary());

        AIAgent analyst = agents.CreateAgent(
            name: "Analyst",
            description: "Keeps spend and wages within the limits.",
            role: ModelRole.Fast,
            instructions:
                "You are Analyst, Arsenal's finance specialist. Given proposed sales and signings, add up the signing " +
                "fees and wages, subtract the sale proceeds and the wages they free, and net against the limits: a " +
                "plan is affordable when signing fees are within the remaining budget plus sale proceeds, and the " +
                "added wages are within the headroom plus the freed wages. Flag any plan that busts either limit and " +
                "suggest cheaper alternatives or extra sales.\n\n" + data.FinancesSummary() +
                "\n\nArsenal squad (sale candidates with their value and wage):\n" + data.SquadSummary() +
                "\n\nShortlist:\n" + data.TargetsSummary());

        Workflow workflow = new MagenticWorkflowBuilder(manager)
            .AddParticipants([scout, analyst])
            .WithName("Window Planner")
            .WithDescription("Plans Arsenal's summer window within budget.")
            .RequirePlanSignoff(false)
            .WithMaxRounds(6)
            .WithMaxStalls(2)
            .WithMaxResets(2)
            .Build();
```

Between every agent turn, the manager runs. It reads the transcript of the progress that has been made and decides three things:

- Is the task done?
- Has progress stalled?
- Who speaks next?

The manager keeps two ledgers:

1. **Task ledger**. This is built up front, and includes the facts it has, plus a step-by-step plan.
2. **Progress ledger**. This keeps tracks of progress being made towards completing the plan.

Each reply from the worker is appended to the shared transcript, and the manager adds it into the next progress-ledger update. When our scout produces a shortlist of players, and the analyst confirms affordability, this counts as progress:

```bash
─ Scout
│ Shortlist to SIGN
│ - Julián Álvarez — proven big‑game striker: scores, links play and presses; immediate upgrade for decisive matches.
│ - Malo Gusto — attack‑minded right wing‑back: elite pace and progressive play with improving defensive metrics for the flank.
│ 
│ Current squad to SELL
│ - Leandro Trossard (31) — surplus wide option; sell to free wages and recoup fee.
│ - Christian Nørgaard (32) — ageing backup CDM; sell to free wage and make room for younger midfielders.
│ - Kepa (31) — expensive backup GK with low resale; sell to cut wages and free squad spot.
│ - Gabriel Jesus (29) — high wage, often redundant with our striker mix; sell to raise funds and reduce wage bill.
   … planning step 3…
   … planning step 4…

┌─ Analyst
│ Short answer: Affordable — proceed.
│ 
│ Key numbers
│ - Sale proceeds: Leandro Trossard £30m + Christian Nørgaard £10m + Kepa £8m + Gabriel Jesus £35m = £83m.
│ - Wages freed by sales: 110k + 70k + 130k + 265k = 575k/wk freed.
│ - Signing costs: Julián Álvarez £85m + Malo Gusto £40m = £125m.
│ - New wages added: Álvarez 250k/wk + Gusto 80k/wk = 330k/wk.
│ - Transfer budget math: Starting £110m + £83m (sales) − £125m (signings) = £68m remaining.
│ - Wage headroom math: Starting headroom 380k/wk + 575k/wk (freed) − 330k/wk (added) = 625k/wk new headroom.
│ - Verdict vs limits: Signing fees (125m) ≤ available funds (193m) ✓; added wages (330k/wk) ≤ headroom+freed wages (955k/wk) ✓. No limit breached.
│ 
│ Squad impact (concise)
│ - Julián Álvarez: clear attacking upgrade over Gabriel Jesus (higher OVR, better pace/shooting/link play). Improves goalscoring/pressing in big matches and offers immediate step up in decisive games.
│ - Malo Gusto: adds elite flank pace and progressive attacking output from the right wing‑back/RB role while being cheaper in wages than current senior backups; strengthens both attacking width and defensive recovery on the right.
│ 
│ Recommendation
│ - Proceed with both signings. They fit the financial constraints (leaving £68m transfer buffer and increasing weekly headroom to ~£625k/wk) and deliver tangible attacking and right‑side defensive upgrades.
```

If an agent repeats itself, this counts as a *stall*, and this triggers a reset, where the manager agent will rebuild the task ledger, and restart the inner loop.

Each round the progress ledger determines if the request has been fully satisfied? If it has, then the manager will synthesize the final answer from the final transcript

Planning a transfer window is a complex task (even more so in real life). There are a lot of open-ended questions that our agents need to answer here (should we prioritize selling players who are at or past their peak? What gaps are there in our squad? etc.).

Multiple specialized agents can provide input and feedback to the plan to shape our solution. These are just simple agents reading off JSON files, but imagine if we have multiple agents with access to their own specialized tools and external systems. Using Magentic orchestration, we can build a solution to solve complex problems that don't have a fixed or deterministic pathway.

## Conclusion

I don't think my dream of being Arsenal's manager is going to be realized anytime soon, but hopefully you now have a better understanding of different orchestration workflows in the Microsoft Agent Framework SDK.

Take a look at the sample code, set up your own Foundry and run the [samples](https://github.com/willvelida/football-scout) yourself!

If you have any questions about this, please feel free to reach out to me on BlueSky!

Until next time, Happy coding! 🤓🖥️