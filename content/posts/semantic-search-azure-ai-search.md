---
title: "Improving Azure AI Search results with semantic search"
date: 2024-01-04
draft: false
tags: ["Azure","AI", "csharp","Tutorial", "Bicep", "ML"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nhoi8faevganeh17socr.png
    alt: "Improving Azure AI Search results with semantic search"
    caption: 'In Azure AI Search, semantic ranking measurably improves search relevance by using language understanding to rerank search results'
---

In Azure AI Search, semantic ranking improves our searches by using language understanding to rerank search results. Semantic search is a collection of query capabilities that improve the quality of search results using text-based queries. Using semantic search we can:

- **Improve Search Results** by adding a ranking over initial search results using advanced algorithms that consider the context and meaning of the query, resulting in more relevant search outcomes.
- **Provides Additional Information** by extracting and displaying *captions* and *answers* from search results, which can be used to improve the user's search experience.

In this article, we'll talk a little about Semantic Search is in Azure AI Search, its advantages and limitations, how to set it up in Azure AI search, and how we can perform semantic searches using C#.

## What is Semantic Search?

Semantic search is a feature within Azure AI Search that aims to improve the ranking of search results. Semantic search improves the ranking of search results by using language understanding to match the context of the original query.

AI Search uses BM25 ranking, which is a function that ranks search results based on the frequency that the search term appears within a document. This is usually quite effective, because documents that include a particular search terms is often the most relevant, but not always the case. BM25 ranking functions don't place any relevance on the semantics of the query and ranking can sometimes be improved by adding language understanding.

Semantic search has 2 functions:

1. It improves the ranking of the query results based on language understanding.
2. It improves the response to the query by providing captions and answers in the results.

Semantic ranking uses BM25 ranking and calculates a new relevance score using the original BM25 ranking combined with the language understanding models to extract the context and meaning of the query.

Semantic captions and answers provide additional results alongside your ranked search results that you can display to improve the understanding of the results to your users.

Captions will extract the summary sentences from the document and highlight the most relevant text in the summary sentences. Answers are an optional feature of semantic search that provides answers to questions. If the search query appears to be a question, and the search result contains text that appears to be a relevant answer, then the semantic answer is returned.

In Azure AI Search, semantic ranking takes the top 50 results from the BM25 ranking results. The results are split into multiple fields as defined by semantic configuration. These fields are then converted into text strings and trimmed into 256 unique tokens. In a document, a token is equivalent to a word.

Once these strings are prepared, they are passed to machine reading comprehension models to find the phrases and sentences that best match the query. The results of this summarization phase is a semantic caption, and optionally, a semantic answer. Semantic captions are now ranked based on the semantic relevance of the caption. Results are then returned in descending order in relevance.

### Advantages of semantic search

Semantic ranking has 2 advantages over traditional search results:

1. It can rank results to more closely match the semantics of the query. This can make it more likely that the most useful documents appear at the top of your results.
2. It can find strings within the results to render as a caption on the search results page and to provide an answer to a question.

### Limitations

Semantic search is applied to results returned from the BM25 ranking function. It can re-rank results provided by the BM25 ranking function, it won't provide any additional documents that weren't returned by the BM25 ranking function.

Also, remember that the BM25 ranking only works for the first 50 results. If more than 50 results are returned, only the top 50 results are considered.

## How to setup Semantic Search

We enable semantic search for Azure AI search at the service level. We'll need to have a search service with at least one index, and the AI search service needs to be at a billable tier. If you don't have one, you'll need to create the service with a billable tier as you cannot change tiers once it's provisioned.

It's also not available in every region. I'll be deploying my resource into Australia East, but you'll want to check out [which regions](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/?products=search&regions=all) support it before deploying it.

To deploy a billable search service, you can use the following bicep template:

```bicep
@description('The suffix applied to deployed resources')
param suffix string = uniqueString(resourceGroup().id)

@description('The name of the Azure Cognitive Search service that will be deployed')
param searchServiceName string = 'search-${suffix}-dev'

@description('The pricing tier of the search service that will be deployed. Default is free')
param sku string = 'basic'

@description('The location of the Azure Cognitive Search service that will be deployed. Default is location of the resource group')
param location string = resourceGroup().location

@description('Replicas distribute search workloads across the service.')
param replicaCount int = 1

@description('Partitions allow for scaling of document count as well as faster indexing by sharding your index over multiple search units')
param partitionCount int = 1

resource searchService 'Microsoft.Search/searchServices@2022-09-01' = {
  name: searchServiceName
  location: location
  sku: {
    name: sku
  }
  properties: {
    replicaCount: replicaCount
    partitionCount: partitionCount
  }
}
```

To deploy the template, you'll need to use the Azure CLI to create the resource group and then deploy the template, which you can do using these commands (just remember to use your own values):

```bash
az group create --name "resource-group-name" --location "azure-region"
az deployment group create --resource-group "resource-group-name" --template-file .\main.bicep
```

Once the search service has been deployed, you can enable Semantic Search. Once you enable it, it'll be available on all indexes.

In your Azure AI Search resource, click on *Semantic ranker* under *Settings* to choose what type of availability plan you want. You can choose either the *Free* tier, which gives you 1,000 requests per month, or *Standard* which gives you the first 1,000 requests free per month, then you'll be charged per additional 1,000 requests. For this tutorial, all we need is the Free tier:

![Choosing Free tier semantic ranker](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e6d9lsju2zp3qeo8sl01.png)

Click on *Select Plan* to get started.

Once that's done, we can use some sample data to test our semantic search. In your Azure AI overview page in the Azure portal, you can do this by clicking on *Import data*.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nsviqpqq9bzfoo5rkrbx.png)

From here, we want the *hotel-samples* source.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yv2hvsawsczov1wquupz.png)

You'll be taken through a whole setup process where you can choose what to import, the schema of your index and the name of your indexer. For our purposes, just use the defaults for now. Once the indexer has run, you should have a new index with 50 documents in them:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mtrf2a1ltui78jox103u.png)

Once we have our data imported, we can configure our semantic ranking in the portal. To do so, select the index that you just created and click on *Add Semantic Configuration*:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a3a64et62n4miuan1oys.png)

From here, we'll need to select our title, content and keyword fields that will be used for semantic ranking, captions, highlights and answers.

We'll create a semantic configuration called *hotels-conf* with the title field of *HotelName*. We'll pick *Description*, *Category*, and *Address/City* as our field names for the Content, and we'll use *Tags* field as our keyword fields. Click save to save your semantic configuration.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yk6byv7wszpa7ayywgk4.png)


Once your semantic configuration is saved, you'll need to save the index again for it to be applied correctly. Just click save before you start making queries against it.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/81f658wyzasbdsx4hv35.png)

Now we can start to write some queries against our index. In the *Search Explorer*, use the JSON view and enter the following payload:

```json
{
  "search": "all hotels near the water",
  "searchFields": "",
  "queryType": "semantic",
  "semanticConfiguration": "hotels-conf",
  "captions": "extractive",
  "answers": "extractive|count-3",
  "queryLanguage": "en-US",
  "count": true
}
```

Hit *Search* and you should see the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0co8namamvdj8272ccmc.png)

In our *search* property, we asked our index to return all hotels near the water, and in our results, we'll see hotels that are near the water, such as Oceanside hotels, or hotels with pools.

## Performing semantic search with C#

Now that we've done this via the portal, let's do this programmatically via C#. For this, we'll just use a regular console app. To create a client that we can use to perform semantic search for Azure AI Search, use the following code:

```csharp
IConfiguration configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

Uri serviceEndpoint = new Uri(configuration["AzureSearchEndpoint"]);
AzureKeyCredential apiKey = new AzureKeyCredential(configuration["AzureSearchKey"]);
string indexName = "hotels-quickstart";

SearchIndexClient searchIndexClient = new SearchIndexClient(serviceEndpoint, apiKey);
SearchClient searchClient = new SearchClient(serviceEndpoint, indexName, apiKey);
```

In this code blok, we're loading a JSON configuration file with a ```IConfiguration``` object. We're then creating variables for our Azure AI Search endpoint, API Key and the name of the index that we'll create with the semantic configuration.

We then create a ```SearchIndexClient``` which we'll use to create our index that will have our semantic configuration applied, and a ```SearchClient``` that we'll use to perform queries against our index.

Let's start with creating our index. I have a ```Hotel``` class that we'll use for the schema of our index, but I also want to apply a semantic configuration for my index so that we can perform semantic queries against our index. To do this, we can write the following code:

```csharp
FieldBuilder fieldBuilder = new FieldBuilder();
var searchFields = fieldBuilder.Build(typeof(Hotel));

var definition = new SearchIndex(indexName, searchFields);
var suggester = new SearchSuggester("sg", new[] { "HotelName", "Category", "Address/City", "Address/StateProvince" });
definition.Suggesters.Add(suggester);

SemanticSettings settings = new SemanticSettings();
settings.Configurations.Add(new SemanticConfiguration("my-semantic-config", new PrioritizedFields()
{
    TitleField = new SemanticField { FieldName = "HotelName" },
    ContentFields = {
                new SemanticField { FieldName = "Description" },
                new SemanticField { FieldName = "Description_fr" }
                },
    KeywordFields = {
                new SemanticField { FieldName = "Tags" },
                new SemanticField { FieldName = "Category" }
                }
}));

definition.SemanticSettings = settings;
searchIndexClient.CreateOrUpdateIndex(definition);
```

Let's break this down a bit:

- We use a ```FieldBuilder``` object that uses the ```Hotel``` class as the basis for our index schema. We create a ```SearchIndex``` object passing in the name of our index and our schema.
- We then create a ```SemanticSettings``` object that we'll use to configure our semantic search. In this object, we'll give our semantic configuration a name *my-semantic-config*, along with some ```PrioritizedFields```, which wil be a self-contained object.
- In our ```PrioritizedFields``` object, we'll provide ```SemanticFields``` for our ```TitleField```, ```ContentFields```, and ```KeywordFields```.
- For our ```TitleField```, this will be a short string (under 25 words). For our tutorial, we'll use *HotelName*.
-  ```SemanticFields``` are larger chunks of text in natural language form. For our example, we'll use the English and French description fields. ```KeywordFields``` will be a list of keywords or descriptive terms.

For our content and keyword fields, you need to list these in order of priority, because lower priority items might get truncated. Across all the properties in your semantic configuration, the fields that you assign must be attributed as *searchable* and *retrievable*, and either be strings of type ```Edm.String```, ```Collection(Edm.String)```, or string subfields of ```Collection(Edm.ComplexType)```.

With our semantic configuration set up, we can now use it to start querying our search resource. To do so, we can write the following code:

```csharp
// Run queries
SearchOptions searchOptions = new SearchOptions()
{
    QueryType = SearchQueryType.Semantic,
    SemanticConfigurationName = "my-semantic-config",
    QueryCaption = QueryCaptionType.Extractive,
    QueryCaptionHighlightEnabled = true,
};

searchOptions.Select.Add("HotelName");
searchOptions.Select.Add("Category");
searchOptions.Select.Add("Description");

SearchResults<Hotel> searchResults = searchClient.Search<Hotel>("what hotel has a good restaurant on site", searchOptions);
foreach (SearchResult<Hotel> result in searchResults.GetResults())
{
    Console.WriteLine($"Hotel Name: " + result.Document.HotelName);
    Console.WriteLine($"Hotel Category: " + result.Document.Category);
    Console.WriteLine($"Hotel Description: " + result.Document.Description);
    Console.WriteLine();
}
```

To perform a semantic search query, we need to specify a few things in the ``SearchOptions`` class:

- We first need to set the ``QueryType`` as ``Semantic``.
- We then use the same configuration name that we defined earlier when creating the index.
- Additionally, I've enabled ``QueryCaption`` so that if we want, we can retrieve the captions in the response. You can also do this for ``QueryAnswers``.

We can then pass in a semantic search query into our ``Search()`` method. For our purposes, we pass in the query *what hotel has a good restaurant on site?* In our ``SearchOptions``, we have specified that we want the *HotelName*, *Category*, and *Description* properties back, so when we receive a response from our query, it look look like the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1tr1b6wuy0vvi300tp78.png)

## Conclusion

In this article, we learnt about Semantic Search in Azure AI Search, its advantages and limitations, how to set it up in Azure AI search, and how we can perform semantic searches using C#.

If you want to learn more about Semantic Search in Azure AI Search, I recommend you take a look at the following resources:

- [Semantic ranking in Azure AI Search](https://learn.microsoft.com/en-us/azure/search/semantic-search-overview)
- [Enable or disable semantic ranker](https://learn.microsoft.com/en-us/azure/search/semantic-how-to-enable-disable?tabs=enable-portal)
- [Configure semantic ranking](https://learn.microsoft.com/en-us/azure/search/semantic-how-to-query-request?tabs=sdk%2Cdotnet-query#3---avoid-features-that-bypass-relevance-scoring)
- [Semantic Search sample](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Search.Documents_11.5.1/sdk/search/Azure.Search.Documents/samples/Sample08_SemanticSearch.md)

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️