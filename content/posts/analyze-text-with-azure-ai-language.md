---
title: "Introduction to analyzing text with Azure AI Language Service and C#"
date: 2023-11-22
draft: false
tags: ["Azure","AI", "csharp","Tutorial", "Bicep", "ML"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/787y85nuvzwylblu8cxa.png
    alt: "Introduction to analyzing text with Azure AI Language Service and C#"
    caption: 'With Azure AI Language services, we can perform powerful text analytics on text, such as Named-Entity Recognition, and Language Detection'
---

How much text data do you produce on a daily basis? This could be in the form of an email you send to a colleague, or posts you send on social media. You may have gone to a new resturant and given them a review online, or you could have created some documents such as contracts as part of your work. Now imagine all that data being created on a global scale. How can we apply AI to this vast amount of text data to extract insights from it?

The Azure AI Language services provides an API for common text analysis tasks that you can use within your own application code. 

In this blog post, we'll start by learning how to provision an AI Language service using Bicep. We'll then use that service to perform a couple of text analytics tasks, such as detecting the language from some text, analyze the sentiment of that text, and extract key phrases, entities and linked entities from that text.

Let's dive in!

## Creating an Azure AI Language resource in Bicep

To use Azure AI to analyze text, we'll need to provision a resource for it. We can do this with a Bicep template. Create a new Bicep template called ```main.bicep```, and write the following:

```bicep
@description('The suffix that will be applied to all resources in the template.')
param applicationSuffix string = uniqueString(resourceGroup().id)

@description('The name of the Azure AI Language service that will be deployed')
param azureAiServiceName string = 'ai-${applicationSuffix}'

@description('The location that resources will be deployed to. Default is the location of the resource group')
param location string = resourceGroup().location

@description('The SKU that wil be applied to the Azure AI Service. Default is S0')
@allowed([
  'F0'
  'S0'
])
param sku string = 'S0'

resource cognitiveService 'Microsoft.CognitiveServices/accounts@2021-10-01' = {
  name: azureAiServiceName
  location: location
  sku: {
    name: sku
  }
  kind: 'CognitiveServices'
  properties: {
    apiProperties: {
      statisticsEnabled: false
    }
  }
}
```

Let's break down the template:

- For our *parameters*, we give our Azure AI Service a name, location, and sku.
- We can provision either a **Free** resource (F0), or a **Standard** resource (S0). For this template, I've used Standard for now.
- For the ```kind``` property, I've used ```CognitiveServices```. This will create a multi-services AI resource, which gives us access to all the Azure AI API's. If you want to create a resource that has a specific functionality, check out the [following documentation](https://learn.microsoft.com/en-us/azure/ai-services/create-account-bicep?tabs=CLI).
- For our location, I've used the location of the resource group that I'll deploy my resource to as default, you can change this to any supported Azure region

To deploy the template, it's a simple matter of creating the resource group that we'll deploy the Azure AI service to, and then deploying the template. We can do this by running the following commands:

```bash
$ az group create --name <name-of-resource-group> --location <supported-azure-region>
$ az deployment group create --resource-group <name-of-resource-group> --template-file main.bicep
```

Once your template has been successfully deployed, you should see the following in the Azure portal:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s4ns5qo020xxcrm927t2.png)

To complete the rest of the tasks in this blog, we'll need to get the **Service Endpoint** and the **Key** so we can authenticate to the Azure AI Client in our code. We can do this by clicking on the **Keys and Endpoint** tab in the menu. Copy and paste the *Endpoint* and one of the *Key* values:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o5rvptqkqqjydrazhofo.jpg)

Before we dive into some code, let's take a step back and find out just exactly what we can do with Azure AI Language.

## What can Azure AI Language do?

As I mentioned in the intro, Azure AI Language is designed to help you extract information from text. You can use it for:

- **Language detection** - What language is this text written in?
- **Key phrase extraction** - identifying important words and phrases in the text.
- **Sentiment analysis** - Is this text positive or negative?
- **Named entity recognition** - detecting references to entities, including people, locations, organizations etc.
- **Entity linking** - identifying specific entities by providing reference links to Wikipedia articles.

Let's dive into these capabilities one-by-one to discover how this all works:

### Language Detection

With Azure AI Language services, we can evaluate the text input and determin which language the text has been written in. This is great for when we encounter text where we don't know what the language is.

You essentially provide the service with some text, and the API will respond with a confidence score between 0 and 1 (the closer to 1, the higher the confidence) as to what language the text has been written in.

You can pass in either single phrases, or entire documents. You are limited to document sizes of **under 5,120 characters** per document, and **1,000** items.

Let's take the following JSON request that we might provide the service:

```JSON
{
    "kind": "LanguageDetection",
    "parameters": {
        "modelVersion": "latest"
    },
    "analysisInput":{
        "documents":[
              {
                "id": "1",
                "text": "Hello world",
                "countryHint": "US"
              },
              {
                "id": "2",
                "text": "Bonjour tout le monde"
              }
        ]
    }
}
```

The AI service will return a JSON response that contains the result of each document in our request, including the **predicted language** and a confidence value of the prediction. So for this request, we might get the following response:

```JSON
{   "kind": "LanguageDetectionResults",
    "results": {
        "documents": [
          {
            "detectedLanguage": {
              "confidenceScore": 1,
              "iso6391Name": "en",
              "name": "English"
            },
            "id": "1",
            "warnings": []
          },
          {
            "detectedLanguage": {
              "confidenceScore": 1,
              "iso6391Name": "fr",
              "name": "French"
            },
            "id": "2",
            "warnings": []
          }
        ],
        "errors": [],
        "modelVersion": "2022-10-01"
    }
}
```

This is relatively straight forward, but what if we passed in content with multilingual content? We may get a different result. If our text was predominantly written in English, that will be the language returned as the largest representation in the content, but the confidence rating would be lower. Take the following request as an example:

```JSON
{
  "documents": [
    {
      "id": "1",
      "text": "Hello, I would like to take a class at your University. ¿Se ofrecen clases en español? Es mi primera lengua y más fácil para escribir. Que diriez-vous des cours en français?"
    }
  ]
}
```

Then our response would be as follows:

```JSON
{
    "documents": [
        {
            "id": "1",
            "detectedLanguage": {
                "name": "Spanish",
                "iso6391Name": "es",
                "confidenceScore": 0.9375
            },
            "warnings": []
        }
    ],
    "errors": [],
    "modelVersion": "2022-10-01"
}
```

There may also be content where the language is difficult to detect. This can happen due to characters that the analyzer is unable to parse. In this scenario, the response for th language name would be ```(Unknown)``` and the confidence score would be 0:

```JSON
{
    "documents": [
        {
            "id": "1",
            "detectedLanguage": {
                "name": "(Unknown)",
                "iso6391Name": "(Unknown)",
                "confidenceScore": 0.0
            },
            "warnings": []
        }
    ],
    "errors": [],
    "modelVersion": "2022-10-01"
}
```

### Key Phrase Extraction

Key phrase extraction is fairly straightforward. This is when Azure AI will look at the text of a document or input, and then identify the main points around the context of the document. This works best for larger documents (still taking the limit of 5,120 characters into account).

Let's take a look at the following request:

```JSON
{
    "kind": "KeyPhraseExtraction",
    "parameters": {
        "modelVersion": "latest"
    },
    "analysisInput":{
        "documents":[
            {
              "id": "1",
              "language": "en",
              "text": "You must be the change you wish 
                       to see in the world."
            },
            {
              "id": "2",
              "language": "en",
              "text": "The journey of a thousand miles 
                       begins with a single step."
            }
        ]
    }
}
```

The response from Azure AI Language service would contain a list of key phrases detected in each of our ```documents```:

```JSON
{
    "kind": "KeyPhraseExtractionResults",
    "results": {
    "documents": [   
        {
         "id": "1",
         "keyPhrases": [
           "change",
           "world"
         ],
         "warnings": []
       },
       {
         "id": "2",
         "keyPhrases": [
           "miles",
           "single step",
           "journey"
         ],
         "warnings": []
       }
],
    "errors": [],
    "modelVersion": "2021-06-01"
    }
}
```

### Sentiment Analysis

If we want to determine whether a text input or document is positive or negative, we can use **Sentiment Analysis** to perform such tasks. This is helpful when we want to perform tasks such as evaluating reviews of resturants, books, products, or evaluating customer feedback received on emails or social media.

When we analyze sentiment using Azure AI Language, the response will include the sentiment for the overal document and individual sentence sentiment for each document submitted to the service.

Take the following request as an example:

```JSON
{
  "kind": "SentimentAnalysis",
  "parameters": {
    "modelVersion": "latest"
  },
  "analysisInput": {
    "documents": [
      {
        "id": "1",
        "language": "en",
        "text": "Good morning!"
      }
    ]
  }
}
```

The response could look like this:

```JSON
{
  "kind": "SentimentAnalysisResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "sentiment": "positive",
        "confidenceScores": {
          "positive": 0.89,
          "neutral": 0.1,
          "negative": 0.01
        },
        "sentences": [
          {
            "sentiment": "positive",
            "confidenceScores": {
              "positive": 0.89,
              "neutral": 0.1,
              "negative": 0.01
            },
            "offset": 0,
            "length": 13,
            "text": "Good morning!"
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2022-11-01"
  }
}
```

Again, confidence scores are scored between 0 and 1 (the closer to 1, the more confident Azure AI is of the sentiment). Sentences can be positive, neutral, negative and mixed.

### Named Entity Recognition

In Azure AI Language Service, **Named Entity Recognition** (NER), identifies entities that are mentioned in the text. Entities are then grouped into categories and subcategories. Some examples of entities include:

- Person
- Location
- DateTime
- Address

There's so much more to this, and you can find the complete list of entities [here](https://learn.microsoft.com/en-us/azure/ai-services/language-service/named-entity-recognition/concepts/named-entity-categories?tabs=ga-api).

So if we provide Azure AI Language with the following request:

```JSON
{
  "kind": "EntityRecognition",
  "parameters": {
    "modelVersion": "latest"
  },
  "analysisInput": {
    "documents": [
      {
        "id": "1",
        "language": "en",
        "text": "Joe went to London on Saturday"
      }
    ]
  }
}
```

The response from Azure AI Language would include a list of categorized entities found in each document, like so:

```JSON
{
    "kind": "EntityRecognitionResults",
     "results": {
          "documents":[
              {
                  "entities":[
                  {
                    "text":"Joe",
                    "category":"Person",
                    "offset":0,
                    "length":3,
                    "confidenceScore":0.62
                  },
                  {
                    "text":"London",
                    "category":"Location",
                    "subcategory":"GPE",
                    "offset":12,
                    "length":6,
                    "confidenceScore":0.88
                  },
                  {
                    "text":"Saturday",
                    "category":"DateTime",
                    "subcategory":"Date",
                    "offset":22,
                    "length":8,
                    "confidenceScore":0.8
                  }
                ],
                "id":"1",
                "warnings":[]
              }
          ],
          "errors":[],
          "modelVersion":"2021-01-15"
    }
}
```

So from our input, Azure AI Language service was able extract the DateTime entity (*Saturday*), the Location entity (*London*) and the Person entity (*Joe*).

### Entity Linking

Now in some text inputs, the same name might apply to more than one entity. A good example of this is the planet "Mars", or is the the Roman god of war "Mars", or is the yummy chocolate bar "Mars".

In these situations, we can use **Entity Linking** to disambiguate entities of the same name by referencing an article in a knowledge base. In Azure AI Language, Wikipedia provides the knowledge base for text. When we receive a response, we'll be given an article link from Wikipedia based on entity context within the text.

Take the following request as an example:

```JSON
{
  "kind": "EntityLinking",
  "parameters": {
    "modelVersion": "latest"
  },
  "analysisInput": {
    "documents": [
      {
        "id": "1",
        "language": "en",
        "text": "I saw Mars in the sky"
      }
    ]
  }
}
```

The response we get get from Azure AI Language service includes the entities identified in the text, along with links to associated articles:

```JSON
{
  "kind": "EntityLinkingResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "entities": [
          {
            "bingId": "89253af3-5b63-e620-9227-f839138139f6",
            "name": "Mar",
            "matches": [
              {
                "text": "Mars",
                "offset": 6,
                "length": 5,
                "confidenceScore": 0.01
              }
            ],
            "language": "en",
            "id": "Venus",
            "url": "https://en.wikipedia.org/wiki/Mars",
            "dataSource": "Wikipedia"
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2021-06-01"
  }
}
```

### Working with Azure AI Language in C#

Now that we've learned a little bit about how Azure AI Language works, let's create a simple console application that explores the functionality a bit more.

To do this, we'll need to retrieve the endpoint of our Azure AI service along with the API Key. Retrieve these from the portal (Under **Keys and Endpoints** in the portal), and then create a ``appsettings.json`` file, and store them like so:

```JSON
{
  "AzureAIEndpoint": "<your-azure-ai-service-endpoint>",
  "AzureAIKey": "<your-azure-ai-key>"
}
```

In Visual Studio, we'll need to set the *Copy to Output Directory* value to **Copy if newer** in the ``appsettings.json`` properties, like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ix9yf06km02a0rg7gqcq.png)

We now need to install some packages to work with Text Analytics, and to load our configuration. To do this, we can run the following dotnet commands in the project directory like so:

```bash
dotnet add package Azure.AI.TextAnalytics
dotnet add package Microsoft.Extensions.Configuration
```

Alternatively, you can install these packages by right-clicking the Project file in Visual Studio, and selecting *Manage NuGet packages*. 

Let's write some code to create our Azure AI Text Client:

```csharp
using Azure;
using Azure.AI.TextAnalytics;
using Microsoft.Extensions.Configuration;

IConfigurationRoot configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

string azureAiEndpoint = configuration["AzureAIEndpoint"];
string azureAiKey = configuration["AzureAIKey"];

// Create the Azure AI Text Client;
AzureKeyCredential keyCredential = new AzureKeyCredential(azureAiKey);
Uri endpoint = new Uri(azureAiEndpoint);
TextAnalyticsClient textAnalyticsClient = new TextAnalyticsClient(endpoint, keyCredential);
```

In this code block, we're loading our configuration file, and setting two variables. One for our **AzureAIEndpoint** and another for our **AzureAIKey**. With these two variables, we pass them to create a ``AzureKeyCredential`` object for our key, and a ``Uri`` object for our endpoint. We can then pass these two objects as references to create a ``TextAnalyticsClient`` object.

With our client created, we can start to work with some text analytics functionality. Let's start with **Language Detection**. In this program, we're just going to allow the user to enter some text, and let Azure AI do it's thing.

To detect the language of our input, we can do the following:

```csharp
// Provide some input for the AI Text client to analyze
Console.WriteLine("----------");
Console.WriteLine("Welcome to AnalyzeMyText");
Console.WriteLine("----------");

Console.Write("Please enter some text for the AI text client to analyze: ");
string userInput = Console.ReadLine();

Console.WriteLine($"You wrote: {userInput}");

// Get the language of the input
DetectedLanguage detectedLanguage = textAnalyticsClient.DetectLanguage(userInput);
Console.WriteLine($"The detected language of the input is: {detectedLanguage.Name}");
Console.WriteLine($"Confidence score of the detection: {detectedLanguage.ConfidenceScore}");
```

The language detection is happening via the ``DetectLanguage`` method. We pass our ``userInput`` to the method, and it will tell us what language the input is in, and the confidence score of the prediction. So for example, when we run the program, and pass in the sentence *The weather in Rome is quite sunny, I had a fun day*, it produces the following output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/biv0e0hrgggl4beu6654.png)

As you can see, Azure AI Language Service has detected the language as English, with a very high confidence score of 1 😂

If I try and enter the same input again, but in Italian, this is the output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g59qgh4ehn1bd7ikzu9w.png)

Azure AI has picked up that the input was Italian, and again is pretty confident about that!

*(N.B I'm trying to learn Italian in my spare time, but I have no idea if this is correct Italian, so I apologize in advance!)*

Let's move onto sentiment! Here we want to see if our input is positive or negative. We can retrieve the sentiment of the input by writing the following:

```csharp
DocumentSentiment documentSentiment = textAnalyticsClient.AnalyzeSentiment(userInput);
Console.WriteLine($"The overall sentiment of the input is: {documentSentiment.Sentiment}");
Console.WriteLine($"Positive sentiment score: {documentSentiment.ConfidenceScores.Positive}");
Console.WriteLine($"Neutral sentiment score: {documentSentiment.ConfidenceScores.Neutral}");
Console.WriteLine($"Negative sentiment score: {documentSentiment.ConfidenceScores.Negative}");
```

The sentiment is being retrieved through the ``AnalyzeSentiment`` method. We pass in the input to that method, and it returns the overall sentiment, and a confidence score of positive, neutral, and negative sentiment. Taking our same sentence as before, it produces the following output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e6ssgp966m2n3rt5bwx6.png)

Azure AI has determined that the input was overall positive with 93% confidence. It also assigned the sentence with 6% neutrality and 1% negativity.

Let's move on to Key Phrase Extraction! We can do this using the ``ExtractKeyPhrases`` method. This will take our input, and if the service can find any key phrases or topics that's in the sentence, it will extract them out:

```csharp
// Identify key phrases in the text
KeyPhraseCollection phrases = textAnalyticsClient.ExtractKeyPhrases(userInput);
if (phrases.Count > 0)
{
    Console.WriteLine("I found some key phrases in that sentence! Let's print them out: ");
    foreach (var phrase in phrases)
    {
        Console.WriteLine($"\t{phrase}");
    }
}
else
{
    Console.WriteLine("I didn't find any phrases in that sentence");
}
```

Passing in our input, we should see the following output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p2yvqf9pvw8lrxgw1w4s.png)

Our program has taken our input, and Azure AI has determined that *fun day*, *weather*, and *Rome* are the key phrases in our sentence.

We can do something similar for extracting entities. With the ``RecognizeEntities`` method, we can pass in our input and Azure AI will extract the entities, which we can then use to see what categories or subcategories that entity belongs to:

```csharp
// Get the entities from our input
CategorizedEntityCollection entities = textAnalyticsClient.RecognizeEntities(userInput);
if (entities.Count > 0)
{
    Console.WriteLine("I found some entities in that sentence! Let's print them out: ");
    foreach (var entity in entities)
    {
        Console.WriteLine($"\t{entity.Text} ({entity.Category} | {entity.SubCategory})");
    }
}
else
{
    Console.WriteLine("I didn't find any entities in that sentence");
}
```

With our input, we get the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zsyrulncfioq2ssznlzy.png)

Azure AI has identified *Rome* as an entity, and categorized it as a *Location*. It has also extracted to sub-categories, *City* and *GPE* or *Geopolitical Entity*.

Finally, let's extract linked entities from our input. To do this, we can pass our input into the ``RecognizeLinkedEntities`` method, like so:

```csharp
// Extract the linked entities from the input
LinkedEntityCollection linkedEntities = textAnalyticsClient.RecognizeLinkedEntities(userInput);
if (linkedEntities.Count > 0)
{
    Console.WriteLine("I found some interesting articles from Wikipedia that might interest you!");
    foreach (var entity in linkedEntities)
    {
        Console.WriteLine($"\t {entity.Name} ({entity.Url})");
    }
}
else
{
    Console.WriteLine("I didn't find any interesting articles related to this text :(");
}
```

Here, Azure AI will analyze our input and provide us links from Wikipedia on any entities that are identified in the input. Using our sentence from before as an input, we should get the following output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/adyu032tybzgb2uf4n26.png)

Azure AI has found *Rome* as an entity, and has provided us with a link to an article on Wikipedia for Rome.

## Conclusion

In this article, we learnt how we can deploy an Azure AI resource using Bicep templates. We then learnt about what Azure AI Language services can do, and what types of functionality it provides. We then wrote a simple C# console app that showed how we can use these features programmatically, and what outputs it produces.

If you want to learn more about Language Services in Azure AI, I recommend you talk a look at the following resources:

- [What is Azure AI Language?](https://learn.microsoft.com/en-us/azure/ai-services/language-service/overview)
- [AI 102 GitHub Labs](https://microsoftlearning.github.io/AI-102-AIEngineer/)

If you want to play around with this sample, feel free to grab it from my [GitHub repository](https://github.com/willvelida/azure-ai-samples/tree/main/analyze-text-with-azure-ai-language)

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️