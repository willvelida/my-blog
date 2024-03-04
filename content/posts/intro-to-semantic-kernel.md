---
title: "Building AI agents with the Semantic Kernel SDK and Azure OpenAI"
date: 2024-03-04
draft: false
tags: ["Azure", "AI", "Azure OpenAI", "CSharp", "Semantic Kernel SDK"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rjuwudgu0fz564twrvf3.png
    alt: "Building AI agents with the Semantic Kernel SDK and Azure OpenAI"
    caption: "Using the Semantic Kernel SDK, we can integrate large language models in our applications without having to know the finer details of the models themselves."
---

The Semantic Kernel SDK is an open-source SDK that allows developers to integrate large language models (also known as LLMs) in their applications. The Semantic Kernel SDK allows developers to integrate prompts to LLMs and results in their own applications. 

{{< youtube ZIBSD8ECuiw >}}

For example, say we're developing a booking system for a medical clinic. Instead of creating a LLM from scratch, we can use the Semantic Kernel to use existing LLMs and create an AI agent that can understanding the natural language queries that our patients make, provide recommendations based on their queries, and book them in to appointments.

The Semantic Kernel SDK allows you to use LLMs like OpenAI, Azure OpenAI and Hugging Face so you can integrate them into your C#, Python and Java applications.

In this article, I'll explain why Semantic Kernel exists, why we should use it and how we can use it to build Kernel agents.

## What is Semantic Kernel?

The Semantic Kernel SDK allows developers to build their own custom **AI agents**. AI Agents are programs that can achieve predetermined goals. The LLMs are trained on massive amounts of data, and agents can perform a wide variety of tasks with some or minimal human intervention.

To use an example, a copilot is a type of AI agent. It won't do all the work for you, but it can help you by providing suggestions and recommendations. GitHub Copilot is a great example of this!

Semantic Kernel integrates LLMs like Azure OpenAI in your application. We create plugins that interface with LLMs and perform all sorts of tasks.

There are built-in plugins that you can drop into your application using the Semantic Kernel SDK, meaning that you don't have to know the details of each model's API to use them.

The following diagram illustrates the key components of the SDK:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o2qb1p7rke1mtyrijia6.png)

1. AI Orchestration layer

At the very core of the Semantic Kernel stack lies a AI orchestration layer that enables the integration of AI models and plugins.

2. Connectors

The SDK provides a set of connects that enable us to integrate LLMs in our applications. These connectors help bridge the gap between our application code and the AI models.

3. Plugins

The SDK operates on plugins, which consists of prompts that you want the AI model to respond to and functions that can complete specialized tasks. We can use pre-built plugins, or create our own.

## What's the point?

Integrating AI into applications can be complex. Developers have to learn multiple service-specific APIs and the details of the LLM that they want to integrate.

The Semantic Kernel SDK solves this problem by offering a simplified integration of AI capabilities into existing applications. The SDK provides an abstraction layer which lowers the barrier of entry for new developers, and it supports the ability to fine tune prompts to provide a predictable user experience.

## Building a simple Semantic Kernel application.

Let's build a basic application using the Semantic Kernel SDK. For this, you're going to need the following installed/provisioned:

- .NET 8 SDK installed.
- Visual Studio Code.
- Azure OpenAI deployed (You need to have this enabled in your Azure subscription)

We're just going to create a new console application, which you can do by running the following:

```bash
dotnet new console -o MyFirstSKProject
# After running the above, navigate into that directory
cd MyFirstSKProject
```

We now need to install the Semantic Kernel SDK by running the following:

```bash
dotnet add package Microsoft.SemanticKernel
```

Since we are also using Configuration in our application, you'll want to install the following:

```bash
dotnet add package Microsoft.Extensions.Configuration --version 8.0.0
```

Before we right any code, we'll need to create an endpoint for the LLM model that we'll be using in our application. Using Azure OpenAI, you can create a new model to use within the Azure OpenAI Studio.

For this tutorial, we'll create a new model that uses the **gpt-35-turbo-16k** model. I talked about the base models in a [previous blog post](https://www.willvelida.com/posts/building-nlp-azure-open-ai/). This model generate natural language and code completions based on natural language prompts.

In Azure OpenAI Studio, select **Create New Deployment** and then **Deploy Model**. Under **Select a model**, select **gpt-35-turbo-16k** and use the default model. Enter a name of your deployment and then retrieve the **Keys** and **Endpoint** for your OpenAI resource. You will need these to call the model in your app.

Now in our application code, we can write the following:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.Extensions.Configuration;

var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var builder = Kernel.CreateBuilder();

builder.Services.AddAzureOpenAIChatCompletion(
        configuration["DEPLOYMENT_MODEL"],
    configuration["AZURE_OPEN_AI_ENDPOINT"],
    configuration["AZURE_OPEN_AI_KEY"]
);

var kernel = builder.Build();
```

For our configuration, we will use the following `appsettings.json` file. Remember to replace these values with your own:

```json
{
    "DEPLOYMENT_MODEL":"",
    "AZURE_OPEN_AI_ENDPOINT":"",
    "AZURE_OPEN_AI_KEY":""
}
```

To test that our endpoint and kernel is working, we can write the following code:

```csharp
var result = await kernel.InvokePromptAsync("Give me a list of leg exercises that I can do in the gym");

Console.WriteLine(result);
```

Run the code by running `dotnet run` in the terminal and we should see the following output:

```bash
Sure, here are some exercises you can do to target your legs in the gym:

1. Squats - barbell squats, goblet squats, single-leg squats
2. Lunges - walking lunges, reverse lunges, curtsy lunges
3. Deadlifts - conventional deadlifts, sumo deadlifts, Romanian deadlifts
4. Leg press
5. Leg extensions
6. Leg curls
7. Calf raises - standing calf raises, seated calf raises, donkey calf raises
```

This response comes from the Azure OpenAI model that we've passed to the kernel. The Semantic Kernel SDK connects to the LLM and runs the prompt. So instead of having to use the Azure OpenAI SDK directly, we have added the plugin for Azure Open AI, and the Semantic Kernel SDK has acted as an AI orchestration layer between our code and the LLM we used from Open AI.

## Conclusion

In this article, I'll explain why Semantic Kernel exists, why we should use it and how we can use it to build Kernel agents.

In this article, we talked about what the Semantic Kernel is and how we can use it to build applications. We then built a basic application that uses the Semantic Kernel SDK to call a LLM that's been provisioned in Azure OpenAI.

If you want to learn more about the Semantic Kernal SDK, check out the following resources:

- [What is Semantic Kernel?](https://learn.microsoft.com/en-us/semantic-kernel/overview/)
- [Getting started with the Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/get-started/quick-start-guide?toc=%2Fsemantic-kernel%2Ftoc.json&tabs=Csharp)
- [Semantic Kernel on GitHub](https://github.com/microsoft/semantic-kernel)

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️

