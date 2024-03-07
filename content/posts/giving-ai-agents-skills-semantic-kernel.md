---
title: "Giving our AI Agents skills using native functions in the Semantic Kernel SDK"
date: 2024-03-07
draft: false
tags: ["Azure", "AI", "Azure OpenAI", "CSharp", "Semantic Kernel SDK"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xf5kt26h7m005qomtt5x.png
    alt: "Giving our AI Agents skills using native functions in the Semantic Kernel SDK"
    caption: "Native functions are functions written in native code that can be called by the Semantic Kernel SDK in an artificially intelligent (AI) application. They're useful for performing tasks that large language models can't do on their own."
---

[In my last post on Semantic Kernel](https://www.willvelida.com/posts/create-plugins-semantic-kernel/), I talked about how we interact with large language models (LLMs) through plugins that run with NLP prompts.

We can also create plugins using native code. We can use our code to integrate with LLMs for calculations, data manipulation or any other functionality that our applications require.

In this article, I'll talk about how we can use native functions in the Semantic Kernel SDK, how we can create native function plugins, and how we can combine native functions with prompts.

**If you want to watch a video that talks about these concepts, check out the video below!**

{{< youtube g2MuT1V3FYk >}}

## What are native functions?

Put simply, native functions are functions that we write in our native code that can be called by the Semantic Kernel SDK in our AI applications.

They're useful for performing tasks that LLMs can't do on their own. We can combine skills with existing services that enable our AI agents to perform a wide range of tasks.

Native functions have a certain format and a recommended file structure that will be used by the kernel. Native functions have a `KernelFunction` decorator in their definitions, and it's recommended to store them in `Plugins` directory. You should keep related functions in the same file or subdirectory.

Let's use an example. Say we're building a AI Fitness instructor that suggests new exercises based on exercises that we have logged during a workout. We'll create a plugin that uses native functions to interact with a database of existing exercises, and exercises that a user has done to recommend new exercises that a user can do during their workout.

We can create a plugin for our exercises and add native functions to it to perform functionality that LLMs cannot perform:

```csharp
public class ExerciseLibraryPlugin
{
    [KernelFunction, Description("Get a list of exercises to the user")]
    public static string GetExerciseLibrary()
    {
        string dir = Directory.GetCurrentDirectory();
        string content = File.ReadAllText($"{dir}/exerciselibrary.txt");
        return content;
    }

    [KernelFunction, Description("Get a list of exercises recently logged by the user")]
    public static string GetRecentExercises()
    {
        string dir = Directory.GetCurrentDirectory();
        string content = File.ReadAllText($"{dir}/recentexercises.txt");
        return content;
    }

    [KernelFunction, Description("Add a exercise to the recently logged list")]
    public static string AddToRecentExercises(
        [Description("The name of the exercise")] string exercise,
        [Description("The number of reps that was performed")] string reps,
        [Description("The number of sets that was performed")] string sets)
    {
        string filePath = $"{Directory.GetCurrentDirectory()}/recentexercises.txt";
        string jsonContent = File.ReadAllText(filePath);
        var recentlyLogged = (JsonArray) JsonNode.Parse(jsonContent);

        var newExercise = new JsonObject
        {
            ["exercise"] = exercise,
            ["reps"] = reps,
            ["sets"] = sets
        };

        recentlyLogged.Insert(0, newExercise);
        File.WriteAllText(filePath, JsonSerializer.Serialize(recentlyLogged, new JsonSerializerOptions { WriteIndented = true}));

        return $"Added '{exercise}' to recently played";
    }
}
```

In this code, we use the `KernelFunction` decorator to indicate that this method is a native function. We also use the `Description` decorator to add a description of what the function does.

We've defined two native functions that retrieves a list of exercises to the user, and a list of exercises that have been logged by the user. Taking the `recentexercises.txt` file as an example, here is the file format that I'm using:

```json
[
    {
        "exercise": "Bench Press",
        "reps": "8",
        "sets": "4"
    },
    {
        "exercise": "Squats",
        "reps": "12",
        "sets": "6"
    },
    {
        "exercise": "Deadlifts",
        "reps": "5",
        "sets": "5"
    },
    {
        "exercise": "Lunges",
        "reps": "20",
        "sets": "4"
    }
]
```

We can then import the `ExerciseLibraryPlugin` to the kernel using `ImportPluginFromType`. We can then use the `InvokeAsync` using our plugin name and function name that we want to call:

```csharp
var kernel = builder.Build();
kernel.ImportPluginFromType<ExerciseLibraryPlugin>();

var result = await kernel.InvokeAsync(
    nameof(ExerciseLibraryPlugin),
    nameof(ExerciseLibraryPlugin.AddToRecentExercises),
    new()
    {
        ["exercise"] = "Shoulder Press",
        ["reps"] = "12",
        ["sets"] = "5"
    });

Console.WriteLine(result);
```

## Combining native functions and prompts

Plugins can consist of both native code and semantic prompts to LLMs. By combining the two, we can build some pretty intelligent applications quickly.

We can call our own plugin functions instead our custom prompts. The Semantic Kernel SDK allows us to create prompts that can call other plugins, meaning that you can create prompts that call other prompts, or prompts that call your own custom functions.

Going back to our `ExerciseLibraryPlugin`, let's examine the following function

```csharp
[KernelFunction, Description("Get a list of exercises to the user")]
public static string GetExerciseLibrary()
{
    string dir = Directory.GetCurrentDirectory();
    string content = File.ReadAllText($"{dir}/exerciselibrary.txt");
    return content;
}
```

Say we have a file that contains a catalog of exercises that the user can perform, we can combine our native functions with a semantic prompt. The function will be able to retrieve data that the LLM that we are using couldn't access on its own. 

That function will be able to pass the retrieved data to the LLM to recommend an exercise to perform:

```csharp
string prompt = @"This is a list of exercises available to the user:
                {{ExerciseLibraryPlugin.GetExerciseLibrary}}

                This is a list of exercises that the user has recently performed:
                {{ExerciseLibraryPlugin.GetRecentExercises}}

                Based on their recent activity, suggest an exercise from the list to do next";

var combineResult = await kernel.InvokePromptAsync(prompt);

Console.WriteLine(combineResult);
```

## Conclusion

In this article, we learnt how we can create and use native functions that integrate with LLMs to extend their functionality. We also learnt how we can combine native functions with semantic prompts to build AI applications.

If you want to learn more about native functions in the Semantic Kernel SDK, check out the following resources

- [Creating native functions for AI to call](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/using-the-kernelfunction-decorator?tabs=Csharp)
- [Calling functions within a prompt](https://learn.microsoft.com/en-us/semantic-kernel/prompts/calling-nested-functions?tabs=Csharp)

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️