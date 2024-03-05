---
title: "Creating Plugins with the Semantic Kernel SDK and C#"
date: 2024-03-05
draft: false
tags: ["Azure", "AI", "Azure OpenAI", "CSharp", "Semantic Kernel SDK"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hkz0hlcgrwnbnp3lgd98.png
    alt: "Creating Plugins with the Semantic Kernel SDK and C#"
    caption: "Using the Semantic Kernel SDK, we use plugins that act as the building blocks for our AI applications. Plugins essentially define the tasks that the kernel should complete, and the kernel interfaces with large language models and run the plugins we define."
---

When we use the Semantic Kernel SDK, we use plugins that act as the building blocks for our AI applications. Plugins essentially define the tasks that the kernel should complete, and the kernel interfaces with large language models and run the plugins we define.

Plugins can include native code and natural language prompts, allowing us to use generative AI in our application. Plugins give us the flexibility of defining desired behavior in our application, and we can create custom prompt plugins to fine tune our applications precisely as we need to.

In this article, we'll talk about how we can create our own Semantic Kernel SDK plugins to accomplish different custom tasks. We'll also look at how we can use the built-in plugins from the Semantic Kernel SDK to create our applications.

**If you want to watch a video of this instead, check out the video I've posted on my YouTube channel below!**

{{< youtube GELR1bNfK6I >}}

## Working with Built-in Plugins

Plugins are fundamental to the Semantic Kernel SDK, as they define the tasks for the kernel to perform since they interface with LLMs. Plugins can be just native code and prompts that are invoked to the LLM. The great thing about the Semantic Kernel SDK is that it offers built-in plugins for common tasks that are ready to use.

Plugins are classes that defines a task that the kernel should perform. It can be made from a semantic prompt, or native function code. To use a plugin, we can add it to the kernel and call it using `InvokeAsync`. The kernel will then run the plugin that interfaces with the LLM, and returns the result.

Some examples of available built-in plugins include:

- `MathPlugin` = A plugin to perform mathematical operations.
- `TextPlugin` = A plugin to perform text manipulation.
- `TimePlugin` = A plugin to get time and date information.

## Optimizing our prompts

The Semantic Kernel SDK supports a templating language that allows us to complete tasks using NLP prompts. These are conversational cues that we pass to LLMs, shaping responses based on your queries or instructions. 

This involves providing rich instructions to guide the model to generate the desired response. This is important, since we need to provide clarity to our prompts to get the results we want. Here are some tips to keep in mind when we craft prompts:

- **Specific Inputs Yield Specific Outputs** - LLMs will respond based on the input they receive. Having clear and specific prompts is crucial to get the desired output.
- **Try, and try again** - Experimenting is key to understand how the model interprets and generates responses. Even small changes can lead to significant changes in outcomes.
- **The Context Matters** - LLMs consider the context provided in the prompt. We need to ensure that the context is well-defined and relevant to obtain accurate responses.
- **Handling Ambiguity** - Ambiguous queries are always fun, but even LLMs will struggle with them. Provide context and structure to avoid vague or unexpected results.
- **Length of Prompts** - While LLMs can process both short and long prompts, you need to consider the tradeoff between clarity and brevity. Experiment with the prompt length can help you find the right balance.

## Saving prompts to files.

One cool feature we can implement for larger projects is the ability to organize our prompts into separate files, and then import them into the kernel. For our own plugins, it's a good idea to create separate folders for our prompts.

The Semantic Kernel SDK supports a prompt templating language with some simple syntax rules. We don't need to write code or import any external libraries. Semantic Kernel parses your template and runs the logic behind it. This supports adding variables, calling external functions, and passing parameters to functions.

This is configured using a `config.json` file. As the name suggests, this contains the configuration details for the prompt, and is placed in the same folder as the `skprompt.txt` file. It supports the following parameters:

- `type` - The type of prompt.
- `description` - A description of what the prompt does. This can be used by the kernel to automatically invoke the prompt.
- `input_variables` - Defines the variables that are used inside the prompt.
- `execution_settings` - Defines the setting for completion models.

## Building our own prompts

Let's see how everything works through an example. I want to build an AI agent that helps me with building out my exercise routine (especially for my legs). At the same time, I'm trying to improve my Italian, so I'll see if my AI agent can provide me a few recommendations for leg exercises in Italian!

I've created a new Console Application to build my AI agent, and I'll be using the Azure OpenAI model that I created in [this blog post](https://www.willvelida.com/posts/intro-to-semantic-kernel/). To use that model in Azure OpenAI, I'll need to include an `appsettings.json` file that includes the following:

```json
{
    "DEPLOYMENT_MODEL":"",
    "AZURE_OPEN_AI_ENDPOINT":"",
    "AZURE_OPEN_AI_KEY":""
}
```

Once we have that, we'll need to install the following packages:

```bash
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.Extensions.Configuration --version 8.0.0
dotnet add package Microsoft.SemanticKernel.Plugins.Core --version 1.5.0-alpha
```

With all our NuGet packages installed, we can start by building our Kernel object and add a built-in plugin. Write the following:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Plugins.Core;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var builder = Kernel.CreateBuilder();

builder.Services.AddAzureOpenAIChatCompletion(
    config["DEPLOYMENT_MODEL"],
    config["AZURE_OPEN_AI_ENDPOINT"],
    config["AZURE_OPEN_AI_KEY"]);

builder.Plugins.AddFromType<ConversationSummaryPlugin>();
var kernel = builder.Build();
```

In this code block, we are loading our configuration and adding our connector to Azure OpenAI. We then add a `ConversationSummaryPlugin` to our kernel from the `Core.Plugins` package. We create the kernel and then add the plugin to it.

We then start to build our prompt that asks the LLM to provide a list of leg exercises in Italian. We can do this by writing the following:

```csharp
string language = "Italian";
string input = @$"I'm new to the gym and I want to learn about body building.
                Can you give me a list of leg exercises that are beginner friendly and will
                help me build my leg muscles" in ${language}?";

var result = await kernel.InvokePromptAsync(input);

Console.WriteLine(result);
```

If we run this, our response should be similar to the following:

```bash
Certamente! Ecco una lista di esercizi per le gambe per principianti che possono aiutarti a costruire i muscoli delle gambe:

1. Squat a corpo libero
2. Affondi
3. Step up con sovraccarico
4. Leg press sui macchinari
5. Leg extension sui macchinari
6. Curl delle gambe sui macchinari
7. Stacchi da terra con manubri o bilanciere
8. Glute bridge
9. Hip thrust
10. Calf raises sulla macchina apposita

Ricorda che l'esecuzione corretta dell'esercizio è fondamentale per evitare infortuni e massimizzare i risultati. Ti consiglio di iniziare con un peso leggero e poi aumentare gradualmente man mano che acquisisci più forza e resistenza. Buona fortuna!
```

Bravo! We have received a response from the Azure OpenAI model that we passed to the kernel, which the Semantic Kernel SDK connects to and runs the prompt. We can improve the prompt by adding more specific instructions.

Since I'm trying to make my legs stronger, let's update the prompt to show me how many reps and sets I should do to increase my strength:

```csharp
string input = @$"I'm new to the gym and I want to learn about body building.
                Can you give me a list of leg exercises that are beginner friendly and will
                help me build my leg muscles in ${language}?

                Tell me how many reps and sets I should do to increase my strength. Display
                the exercises in the following format: Exercise - Reps x Sets";
```

We should see a similar response to this:

```bash
Certo! Ecco una lista di esercizi per le gambe che sono adatti ai principianti:

1. Squat bodyweight - 12 x 3
2. Affondi statici - 10 x 3 per gamba
3. Calf Raise - 15 x 3

In generale, per aumentare la forza si consiglia di fare 8-12 ripetizioni per serie e 2-3 serie per esercizio. Tuttavia, essendo tu un principiante, potresti cominciare con un numero inferiore di ripetizioni e serie e aumentare gradualmente man mano che acquisisci più forza e resistenza.

Buona fortuna con il tuo programma di allenamento!
```

Nice! We can also change our prompt to include some background information. I've been going to the gym for a while now, so let's change our prompt to include that background:

```csharp
string language = "Italian";
string background = @"I have been going to the gym for over 10 years, and I'm an experienced weightlifter";

string input = @$"Consider the weightlifter's background: ${background}.

                Can you give me a list of leg exercises that will
                help me build my leg muscles in ${language}?

                Tell me how many reps and sets I should do to increase my strength. Display
                the exercises in the following format: Exercise - Reps x Sets";
```

We should see an output similar to the following:

```bash
Certo! Ecco una lista di esercizi per le gambe che possono aiutare a sviluppare i muscoli delle gambe:

1. Squat - 8-10 x 3-4
2. Affondi con manubri - 10-12 x 3-4
3. Stacco da terra - 6-8 x 3-4
4. Curl delle gambe sdraiato - 10-12 x 3-4
5. Estensione delle gambe - 10-12 x 3-4

Per aumentare la forza, ti consiglio di fare meno ripetizioni ma con più set. Inizia con 3-4 set per ogni esercizio e aumenta gradualmente il peso. Quando ti alleni con pesi più pesanti, cerca di fare tra le 6 e le 8 ripetizioni per set.

Buona fortuna!
```

Let's improve the quality of our responses generated by the LLM by adding a **persona** to our prompt. To do this, let's update our prompt (I'm removing the language for now, but I'll add it in later):

```csharp
string input = @$"
                You are a gym instructor. You are very friendly, positive and give as much detail as possible.

                Consider the weightlifter's background: ${background}.

                Can you give me a list of leg exercises that will
                help me build my leg muscles?

                Tell me how many reps and sets I should do to increase my strength. Display
                the exercises in the following format: Exercise - Reps x Sets";
```

Here's the response:

```bash
Absolutely! Here are some leg exercises that will help you build leg muscles:

1. Squats - 8-12 reps x 3-5 sets
2. Lunges - 8-12 reps (per leg) x 3-5 sets
3. Deadlifts - 8-12 reps x 3-5 sets
4. Leg Press - 10-15 reps x 3-5 sets
5. Bulgarian Split Squats - 8-12 reps (per leg) x 3-5 sets
6. Romanian Deadlifts - 8-12 reps x 3-5 sets
7. Step-Ups - 8-12 reps (per leg) x 3-5 sets
8. Calf Raises - 12-15 reps x 3-5 sets

These exercises are great for building leg muscles because they target various muscles in the legs. When performing these exercises, it's important to use proper form and technique to avoid any injuries.

In terms of reps and sets, it's recommended to do 8-12 reps per set and 3-5 sets per exercise to increase strength. However, it's important to listen to your body and adjust the reps and sets based on your fitness level and goals.

Remember to also incorporate rest days into your workout routine to allow your muscles to recover and grow. Happy lifting!
```

I'll add the Italian back in, and I'll get the following response:

```bash
Certo! Ecco una selezione di esercizi per le gambe che puoi provare:

1. Squat - 4 x 8-10
2. Affondi - 3 x 12-15 per gamba
3. Stacchi da terra - 4 x 8-10
4. Estensione delle gambe - 3 x 12-15
5. Curl delle gambe sdraiati - 3 x 12-15

È importante scegliere un peso che sia abbastanza impegnativo da sollevare ma che ti permetta comunque di mantenere una buona forma durante l'esecuzione dell'esercizio. Se vuoi aumentare la tua forza, ti consiglio di fare 4-6 serie da 6-8 ripetizioni con un peso più pesante e con 2-3 minuti di riposo tra le serie. Se invece vuoi aumentare la tua resistenza muscolare, puoi fare 3-4 serie da 12-15 ripetizioni con un peso più leggero e con 1-2 minuti di riposo tra le serie. Ricorda di inserire anche dei giorni di recupero tra le sessioni di allenamento delle gambe per permettere ai muscoli di riposare e recuperare. Buon allenamento!
```

We can provide instructions to our LLM to assume roles when generating responses, and provide requests and responses as an example. Using Semantic Kernel, we use a special syntax to define user roles. Let's change our input and prompt to use this syntax:

```csharp
string input = "@I enjoy compund exercises using the barbell. I tend to avoid exercise machines";

string prompt = $@"
    The following is a conversation with an AI fitness instructor. The instructor is friendly, positive and knowledgeable.

    <message role=""user"">Can you give me a list of leg exercises that will help me build my leg muscles?>/message>

    <message role=""assistant"">Of course! Do you have any exercises that you know and love to do?</message>

    <message role=""user"">${input}</message>";

var result = await kernel.InvokePromptAsync(prompt);

Console.WriteLine(result);
```

Let's see our response:

```bash
Great! Here are some leg exercises that you can do with a barbell:

1. Squats (squat rack required)
2. Deadlifts
3. Lunges (barbell held in front of the chest)
4. Bulgarian split squats (one foot on the bench, the other foot on the ground, holding the barbell on your shoulders)

These exercises are great for building leg muscles and can be tailored to your fitness level and goals. Let me know if you have any questions or need further guidance on how to perform these exercises.
```

These prompts have worked well for us so far, but as our project grows, we'll want to create prompts and save them to files. We'll also use variables and calling functions inside our prompt templates as well.

To do this, I've created the following folders in my project:

- Prompts
- Prompts/ExercisePlugins
- Prompts/ExercisePlugins/GetExercise
- Prompts/ExercisePlugins/SuggestExercise

To create our prompts, we need to create the `config.json` and `skprompt.txt` files. Let's start with the GetExercise plugin. In our `config.json` file, enter the following:

```json
{
  "schema": 1,
  "type": "completion",
  "description": "Identify the body area of the user's exercise plan",
  "execution_settings": {
    "default": {
      "max_tokens": 1200,
      "temperature": 0
    }
  },
  "input_variables": [
    {
      "name": "input",
      "description": "Text from the user that contains their desired body area for exercises",
      "required": true
    }
  ]
}
```

Next, in our `skprompt.txt` file, enter the following text:

```html
<message role="system">
Instructions: Identify the body area the user wants to exercise.
</message>
<message role="user">
Can you give me a list of exercises that will help me build my muscles?
</message>
<message role="assistant">Of course! Do you have any exercises that you know and love to do?</message>

<message role="user"> {{$input}}</message>
```

For our SuggestExercises plugin, we can create our `config.json` like so:

```json
{
  "schema": 1,
  "type": "completion",
  "description": "Recommend weight exercises to the user",
  "execution_settings": {
    "default": {
      "max_tokens": 1200,
      "temperature": 0.3
    }
  },
  "input_variables": [
    {
      "name": "input",
      "description": "Details about the user's body area",
      "required": true
    }
  ]
}
```

And for our `skprompt.txt` file:

```html
The following is a conversation with an AI fitness instructor. The instructor is friendly, positive and knowledgeable.

    <message role=""user"">Can you give me a list of leg exercises that will help me build my leg muscles?</message>

    <message role=""assistant"">Of course! Do you have any exercises that you know and love to do?</message>

    <message role=""user"">${input}</message>"
```

Let's test our new prompts and update our `Program.cs` file to import our new Plugins.

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Plugins.Core;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var builder = Kernel.CreateBuilder();

builder.Services.AddAzureOpenAIChatCompletion(
    config["DEPLOYMENT_MODEL"],
    config["AZURE_OPEN_AI_ENDPOINT"],
    config["AZURE_OPEN_AI_KEY"]);
var kernel = builder.Build();

kernel.ImportPluginFromType<ConversationSummaryPlugin>();
var prompts = kernel.ImportPluginFromPromptDirectory("Prompts/ExercisePlugins");

ChatHistory history = [];
string input = @"I want to build my leg strength. I love barbell exercises and compound exercises.";

var result = await kernel.InvokeAsync<string>(prompts["SuggestExercises"],
    new() { { "input", input } });

Console.WriteLine(result);
history.AddUserMessage(input);
history.AddAssistantMessage(result);
```

In this code, we've imported our created plugins. We can store the user's conversation using the `ChatHistory` object, and pass some information to the `SuggestExercises` prompt and record the results.

We should see the following output.

```bash
Great! Here are some exercises that can help you build your leg muscles:

1. Squats - This is a classic exercise that targets your quadriceps, hamstrings, and glutes. You can do squats with just your bodyweight or add weights for more resistance.

2. Lunges - Lunges are another great exercise that targets your quads, hamstrings, and glutes. You can do walking lunges, reverse lunges, or stationary lunges.

3. Deadlifts - Deadlifts are a compound exercise that work your hamstrings, glutes, and lower back. They can be done with a barbell, dumbbells, or kettlebells.

4. Leg press - The leg press machine is a great way to target your quads, hamstrings, and glutes. You can adjust the weight and foot placement to target different areas of your legs.

5. Calf raises - Calf raises target your calf muscles and can be done with bodyweight or added weight.

Remember to always warm up before exercising and to use proper form to prevent injury. Good luck with your leg workouts!
```

## Conclusion

In this article, we talked about how we can use built-in plugins in Semantic Kernel. We also learned how to prompt and craft prompts to get the best response from a LLM. We also learnt how to create our own custom prompts to further tailor our experience for our users.

If you want to learn more about the Semantic Kernal SDK and how prompts work, check out the following resources:

- [What is Semantic Kernel?](https://learn.microsoft.com/en-us/semantic-kernel/overview/)
- [Getting started with the Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/get-started/quick-start-guide?toc=%2Fsemantic-kernel%2Ftoc.json&tabs=Csharp)
- [Semantic Kernel on GitHub](https://github.com/microsoft/semantic-kernel)
- [What are prompts?](https://learn.microsoft.com/en-us/semantic-kernel/prompts/)
- [Understanding AI plugins in Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/?tabs=Csharp)
- [Saving and sharing prompts](https://learn.microsoft.com/en-us/semantic-kernel/prompts/saving-prompts-as-files?tabs=Csharp)

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️