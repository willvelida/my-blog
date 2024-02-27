---
title: "Building NLP applications with Azure OpenAI"
date: 2024-02-27
draft: false
tags: ["Azure","Cloud Native", "AI", "Azure OpenAI", "CSharp", "NLP"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fpc9gysrlyioo6ma1pn9.png
    alt: "Building NLP applications with Azure OpenAI"
    caption: "Azure OpenAI provides developers with the ability to ad AI to their applications, with various AI models that can be used for a variety of different tasks"
---

Azure OpenAI provides developers with the ability to add AI to their applications using a variety of different models from OpenAI. This includes GPT-4, GPT-4 Turbo with Vision, GPT-3.5-Turbo and Embedding models.

We can add AI functionality to our applications using C#, Python or REST APIs. The Generative AI capabilities that are available in Azure OpenAI are provided through the models, which belong to different families.

This article assumes that you already have access to Azure OpenAI. To use Azure OpenAI, you need to be approved. Luckily for me, my work already has a resource for me to use. If you haven't got one, check out [this guide](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/create-resource?pivots=web-portal) to get started.

## Choosing and deploying a model

To build applications with Azure OpenAI, we need to choose a model and deploy it. There are several base models that come out of the box, and there is also the option to create customized base models.

Azure OpenAI includes the following model types:

- **GPT-4 models** - These are the latest *generative pretrained* models that can generate language and code completions based on natural language prompts.
- **GPT 3.5 models** - These can generate natural language and code completions based on natural language prompts. GPT-35-Turbo models are optimized for chat-based interactions.
- **Embeddings models** - These convert text into numeric vectors, and are useful in situations where we need to compare text sources for similarities.
- **DALL-E models** - These models are used to generate images based on natural language prompts. They're currently in preview. I like using these to generate images for my social media and blog posts 😁

Each model family performs different tasks well, and each model within each family has different capabilities. We can break down the model families into 3 main one:

1. **Text or Generative Pre-trained Transformer (GPT)** - These models understand and generate natural language and code. These models are great at general tasks, conversations, chats etc.
2. **Code** - These models are built on top of GPT models, and trained on millions of lines of code. These models can generate and understand code, including interpreting comments to generate code.
3. **Embeddings** - These models can understand and use embeddings, which are a special type of data that we can use in ML models and algorithms.

In this article, I'm going to focus on the GPT models.

When we select a model, we can see which family it belongs to and it's capability by the name. For example `text-embedding-3-large`. You can see all the models, capability levels, and naming conventions in this [documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models#gpt-4-and-gpt-4-turbo-preview).

To deploy, manage and explore models, we can use the Azure OpenAI Studio to do so. To 

In the Azure OpenAI Studio, click on the **Deployments** page.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ix1opqoo8jqxracruk5g.png)

From here, we can view our existing model deployments, and create new ones. To create a new deployment, you can click on **Create new deployment**, and create a model to suit your needs:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x1jxwrvmzmpyl5dekwqo.png)

For this article, I created a *gpt-35-turbo** model with the following settings:

- Model Name = gpt-35-turbo
- Model version = 0613
- Deployment name = wv-gpt-35-turbo
- Content filter = Default
- Deployment type = Standard
- Token per minute rate limit = 5K
- Enable dynamic quota = Enabled

## Using our model in our C# application

Once our model is available we can consume it in our code. Azure OpenAI models can be accessed either via the REST API or the Python or C# SDK. To use the C# SDK, we need to install the following NuGet package:

```code
dotnet add package Azure.AI.OpenAI
```

Once that package is installed, we can initialize the Azure OpenAI client and create a `ChatCompletionsOptions` object so we can send requests to our Azure OpenAI model:

```csharp
// Add Azure OpenAI package
using Azure.AI.OpenAI;

// Build a config object and retrieve user settings.
IConfiguration config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();
string? oaiEndpoint = config["AzureOAIEndpoint"];
string? oaiKey = config["AzureOAIKey"];
string? oaiModelName = config["AzureOAIModelName"];

// Initialize the Azure OpenAI Client
OpenAIClient client = new OpenAIClient(new Uri(oaiEndpoint), new AzureKeyCredential(oaiKey));

// Build completion options object
ChatCompletionsOptions chatCompletionsOptions = new ChatCompletionsOptions()
{
    Messages = {
        new ChatMessage(ChatRole.System, "You are a helpful assistant"),
        new ChatMessage(ChatRole.User, "Summarize the following text in 20 words or less:\n" + text),
    },
    MaxTokens = 120,
    Temperature = 0.1f,
    DeploymentName = oaiModelName
};
```

Let's break down what we've added here.

To make authenticated calls to Azure OpenAI via the SDK, we'll need 3 things:

1. **The endpoint name** - This is the base endpoint of your Azure OpenAI resource.
2. **The API Key** - This is the API key that you use to authenticate to your Azure OpenAI resource. Both the endpoint and API key can be found in the **Keys & Endpoint** section in the Azure portal.
3. **The name of your model** - This is the deployment name of your deployed model in Azure OpenAI Studio.

I've added these three files in a `appsettings.json` file. We then build a `IConfiguration` object to use that configuration in our application. We can then instantiate our OpenAI client by doing the following:

```csharp
OpenAIClient client = new OpenAIClient(new Uri(oaiEndpoint), new AzureKeyCredential(oaiKey));
```

Now that we've created a client, we can access different endpoints for different models. Only some endpoints can be used for certain models. These are:

- **Completion** - This model takes an input prompt, and generates one or more predicted completions.
- **ChatCompletion** - This model takes input in the form of a conversation. We define the roles with the message they send, and the next chat completion is generated.
- **Embeddings** - This model takes input and returns a vector representation of that input.

In our code, we've created a `ChatCompletionsOptions` object to define roles in our application. We've told our system that they're a helpful assistant, and our user has told the system to summarize our input in 20 words or less.

We've also defined our `MaxTokens`, `Temperature` and `DeploymentName` properties. Just to explain what these mean:

- **MaxTokens** = This sets a limit on the number of tokens per model response. The API supports a maximum of 4000 tokens shared between the prompt, and the model response. One token is equal to roughly 4 characters in English text.
- **Temperature** - This controls the randomness of the response. The lower the temperature, the more repetitive and deterministic our responses will be. The higher the temperature, the more unexpected the responses will be.
- **DeloymentName** - This is the name of our model that we'll be using for our Azure OpenAI client.

Now that we've set up our client, we can make requests to our Azure OpenAI model by doing the following:

```csharp
// Send request to Azure OpenAI model
ChatCompletions response = client.GetChatCompletions(chatCompletionsOptions);
string completion = response.Choices[0].Message.Content;

Console.WriteLine("Summary: " + completion + "\n");
```

I have a text file that describes the process of making whiskey. When I pass that text file through as a request, I get the following response:

```bash
Summary: Whiskey is made through malting, mashing, fermentation, distillation, aging, and bottling, with each step contributing to the final product's unique flavor.
```

So the output of our program has generated a response from Azure OpenAI. We can make the response more random by increasing our Temperature. This will need to be a value between 0.0 and 1.0, so let's crack it up to the max, and run three times:

```console
# run 1
Summary: Whiskey is made through malting, mashing, fermentation, distillation, aging, and bottling processes, each contributing to the final product's unique flavors.

# run 2
Summary: Whiskey is made through malting, mashing, fermentation, distillation, aging, and bottling, each step contributing to its unique flavor and character.

# run 3
Summary: Whiskey is made through malting, mashing, fermentation, distillation, aging, and bottling, each step adding flavor and complexity.
```

As you can see, we have different outputs each time we run it.

## Conclusion

Azure OpenAI offers different models for text, code and embeddings which we can use via the REST API or C# and Python SDKs. Here I showed a small code example of how we can use the C# SDK to building NLP applications using Azure OpenAI service.

As I get more time on my hands, I'm looking to do more in-depth work with Azure OpenAI service and build out more complex samples. While I don't enjoy the hype around AI at the moment, I'm interested to see what practical examples folks are building with Azure OpenAI service, so I'll be posting more Azure OpenAI content in the future.

If you're interested to learn more about Azure OpenAI, I recommend that you check out the following resources:

- [Develop Generative AI solutions with Azure OpenAI Service MS Learn path](https://learn.microsoft.com/en-us/training/paths/develop-ai-solutions-azure-openai/)
- [Azure OpenAI Service documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️


