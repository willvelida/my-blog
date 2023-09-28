---
title: "Getting started with Azure Cognitive Search in C#"
date: 2023-09-27
draft: false
tags: ["Azure","Azure Cognitive Search","AI", "C#", "Bicep"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8r1p8foatub526q6lbpv.png
    alt: "Getting started with Azure Cognitive Search in C#"
    caption: "In this article, let's dive into Azure Cognitive Search and how we can query it using C# code!"
---

Azure Cognitive Search is a search service in Azure that gives you as a developer the tools for building search experiences over private data in your enterprise, web and mobile applications.

Common use cases for search can include any scenario that surfaces text to users, such as searching a product catalog, or searching documents within your organization. Azure Search provides you with a bunch of cool capabilities, including:

- Full text and vector search over search indexes containing your content.
- Indexing, with optional AI enhancement for extracting and transforming your content.
- Query syntax for text search, autocomplete, geo-search, vector queries and more.
- Access to search via SDKs and REST APIs
- Integration with Azure AI services.

In this article, I'll cover what Azure Cognitive Search is, what components make up Azure Search, and where it can fit within your architecture. We'll then create a Cognitive Search service using Bicep along with a storage account that we will pull our data from into the index. Once that's deployed, we'll start to work with our Cognitive Search service using C#, creating our data source for our indexer, creating the index and then querying our index with some simple queries.

Let's dive in! 🚀

## What is Azure Cognitive Search and what can I use it for?

Within your architecture, Azure Cognitive Search sits between your data source that contains your un-indexed data, and your client app that sends query requests to a search index and handles the response.

Azure Cognitive Search has two primary workloads: **Indexing** and **Querying**:

- **Indexing** - This is the process of loading your data into the search service and making it searchable. Inbound text is processed into tokens and stored in inverted indexes, and inbound vectors are stored in vector indexes. Documents that can be indexed by Cognitive Search are stored in JSON format, which you can either upload JSON documents directly, or use a indexer to serialize your data into JSON.
- **Querying** happens once your index is populated with searchable text. When client apps send queries to your search service, it will handle the response. The query execution happens over a search index.

You integrate Azure Cognitive Search with other Azure services using **indexers**. These automate data ingestion from other Azure data sources, such as Azure Cosmos DB, Blob Storage, Azure SQL and more. Indexers will crawl cloud data sources and populates a search index using mapping between your source data and the search index.

Azure Cognitive Search can also use **skillsets** that use Azure AI services or custom AI that you can create in Azure ML to enrich their search with AI functionality. Skillsets are attached to indexers, and can contain one or more skills that uses built-in or custom AI over your external data.

According to the [docs](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search), Cognitive Search is great for the following:

- Searching over your content (which is what you'd hope for a search service)
- Offloading the indexing and querying workloads onto a dedicated search service (rather than doing this directly on your data source).
- Implementing search-related features easily, such as filters, autocomplete, synonym mapping etc.
- Transforming image and text files into search chunks.
- Adding custom or linguistic text analysis.

## Provisioning our Azure Cognitive Search resource with Bicep

Let's get our hands dirty with Cognitive Search! Cognitive Search has a free tier that you can use to explore some of the functionality that it provides (Obviously, there are limitations on what you can do with the free tier, but for dev/test, it's fine for now). 

In Cognitive Search, you execute queries over content that's loaded into a search index. There are two models that we can load data into a index:

1. **Push Model** - This is when we push data programmatically into Cognitive Search. This model provides the most flexible approach for pushing data into Cognitive Search, and gives us benefits such as no restrictions on data source type, no restrictions on frequency of execution, uploading documents individually or in batches, and full control over connectivity and secure retrieval of documents.
2. **Pull Model** - This model crawls supported data sources, and automatically uploads the data into your index. This is done through *Indexers*, which will connect an index to a data source and maps the source fields to equivalent fields in the index.

For this demo, we'll provision a Cognitive Search service and a storage account using Bicep. Then we'll dive into some C# code that uploads some JSON documents to our Blob storage account, populates an index in Cognitive Search using the data from our Blob Storage account, and then run queries against our index.

If you want to follow along with the code, check out this [GitHub repository](https://github.com/willvelida/azure-cognitive-search-getting-started-sample).

Let's start by provisioning our Azure Search service and Blob Storage account:

```bicep
@description('The suffix applied to deployed resources')
param suffix string = uniqueString(resourceGroup().id)

@description('The name of the Azure Cognitive Search service that will be deployed')
param searchServiceName string = 'search-${suffix}-dev'

@description('The pricing tier of the search service that will be deployed. Default is free')
param sku string = 'free'

@description('The location of the Azure Cognitive Search service that will be deployed. Default is location of the resource group')
param location string = resourceGroup().location

@description('Replicas distribute search workloads across the service.')
param replicaCount int = 1

@description('Partitions allow for scaling of document count as well as faster indexing by sharding your index over multiple search units')
param partitionCount int = 1

@description('The name of the Azure Storage account that will be deployed')
param storageAccountName string = 'storage${suffix}'

@description('The name of the blob container that will be deployed')
param blobContainerName string = 'hotel-rooms'

@description('The SKU of the Azure Storage account that will be deployed. Default is Standard_LRS')
param storageAccountSku string = 'Standard_LRS'

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

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageAccountSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}

resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${storageAccount.name}/default/${blobContainerName}'
}
```

We can provision a Cognitive Search service by using the ```Microsoft.Search/searchServices``` namespace. We're provisioning a free tier account for this tutorial (some features aren't available for the free tier, but that's out of scope for now).

We're also deploying a storage account with a blob container called *hotel-rooms*.

To deploy this, run the following commands in your terminal:

```bash
# Create the resource group
az group create --name <name-of-your-resource-group> --location <azure-region-close-to-you>

# Deploy the Bicep template
az deployment group create --resource-group <name-of-your-resource-group> --template-file .\deploy\main.bicep
```

Once your template has been successfully deployed, you should see it in the portal, like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/klbalqt6nwo6gnchbv8o.png)

## Interacting with our Cognitive Search Service with C#

In the GitHub repository, I've added some JSON documents that I've 'borrowed' from the Microsoft samples in the ```/data``` folder that you can use for this demo. I've uploaded them manually in this sample, so do the same and you should see them in the container like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mp9v4ezfuhz7hcms2mc6.png)

The content of these files contain some information about Hotels and the types of rooms available in those hotels. This includes information about how much they cost, whether smoking is allowed, what types of beds are in the rooms etc. The following is a sample JSON payload of our data:

```json
{
	"Id": "1",
	"HotelName": "Secret Point Motel",
	"Rooms": [
		{
		"Description": "Budget Room, 1 Queen Bed (Cityside)",
		"Description_fr": "Chambre �conomique, 1 grand lit (c�t� ville)",
		"Type": "Budget Room",
		"BaseRate": 96.99,
		"BedOptions": "1 Queen Bed",
		"SleepsCount": 2,
		"SmokingAllowed": true,
		"Tags": [ "vcr/dvd" ]
		}
	]
}
```

In this tutorial, we'll be building a simple C# console application that creates a index for our hotel room data, uses our blob storage container as our data source for our indexer, creates our indexer and then run a few queries over our Search service.

### Setting up our console app

For this console application, we're going to need some keys/settings from both our Cognitive Search service and our storage account. For our search service, we'll need the following:

- *Search Service URL*: You can find this under the **Essentials** tab in the Overview of your service. It should be in the form of ```https://{name-of-your-search-service}.search.windows.net```
- *API Key*: You can find these under **Keys**. You have two admin keys that you can use, and regenerate as required.

Before you ask, yes you can use Azure RBAC to authenticate to your search service. I'll cover this in a future post.

For our storage account, we'll need the connection string. To get this, go into your storage account and under **Access Keys**, you'll see two connection strings that you can use.

You can store all these settings in a basic *appsettings.json* file like so:

```json
{
  "SearchServiceUri": "<your-search-service-uri>",
  "SearchServiceApiKey": "<your-search-service-api-key>",
  "BlobStorageAccountName": "<name-of-storage-account>",
  "BlobStorageConnectionString": "<primary-connection-string-of-storage-account>"
}
```

With our configuration sorted, we'll need to install a couple of packages for our console app. I did this by running the following dotnet CLI commands:

```bash
dotnet add package Azure.Search.Documents
dotnet add package Microsoft.Extensions.Configuration.Json
```

Just to explain the purpose of these packages:

- **[Azure.Search.Documents](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/search.documents-readme?view=azure-dotnet)** - This is the .NET SDK client library that we'll use for interacting with Cognitive Search. You can use the SDK to query data in your indexes, create indexes, manage indexers, skillsets, and more.
- **Microsoft.Extensions.Configuration.Json** - This is the JsonConfigurationProvider class that allows us to load configuration from a JSON file. Since we're using a *appsettings.json* file in our console app, we will use this to load our configuration from that file.

### Creating our Indexes and Indexer

With those packages installed, we can create some clients in our console app to interact with our Cognitive Search service. We'll be creating 2 clients:

- **SearchIndexClient** - This client is used to create, update and delete indexes.
- **SearchIndexerClient** - This client is used to work with both indexers and skillsets.

To create both clients, we'll need the Cognitive Search service URI and the API key. To load this from our configuration file, we can write the following:

```csharp
IConfigurationRoot configuration = new ConfigurationBuilder().AddJsonFile("appsettings.json").Build();

string searchServiceUri = configuration["SearchServiceUri"];
string searchServiceApiKey = configuration["SearchServiceApiKey"];
string indexName = "hotel-rooms";

SearchIndexClient searchIndexClient = new SearchIndexClient(new Uri(searchServiceUri), new AzureKeyCredential(searchServiceApiKey));
SearchIndexerClient searchIndexerClient = new SearchIndexerClient(new Uri(searchServiceUri), new AzureKeyCredential(searchServiceApiKey));
```

With our clients created, we'll start by creating our index using the **SearchIndexClient**. In this application, I'm resetting the index each time it runs by retrieving the index (via ```GetIndexAsync```) and then deleting it (via ```DeleteIndexAsync```). If the index doesn't exist, we'll create a new index by creating a new **SearchIndex** object. To do this, we can write the following:

```csharp
try
{
    await searchIndexClient.GetIndexAsync(indexName);
    await searchIndexClient.DeleteIndexAsync(indexName);
}
catch (RequestFailedException ex) when (ex.Status == 404)
{
    // if the index doesn't exist, throw 404
}

FieldBuilder builder = new FieldBuilder();
var definition = new SearchIndex(indexName, builder.Build(typeof(Hotel)));
await searchIndexClient.CreateIndexAsync(definition);
```

In this code block, we're creating a new **SearchIndex** object, passing in the name of the index, as well as a **FieldBuilder** object. This object uses reflection to create a list of **SearchField** objects for the index by examining the public properties and attributes given to the class that we pass through it. In our case, it's our ```Hotel``` class:

```csharp
﻿using Azure.Search.Documents.Indexes;
using Azure.Search.Documents.Indexes.Models;
using Microsoft.Spatial;
using System.Text.Json.Serialization;

namespace CognitiveSearchDemo.Models
{
    public partial class Hotel
    {
        [SimpleField(IsFilterable = true, IsKey = true)]
        public string HotelId { get; set; }

        [SearchableField(IsFilterable = true, IsSortable = true)]
        public string HotelName { get; set; }
        public Room[] Rooms { get; set; }
    }
}
```

With Cognitive Search, your C# data models should support the search experience that you want to provide to your users. Top level objects in .NET corresponds to a search result that you want to present in your search UI.

In each of your classes, fields are defined with a data type and attribute that determine how it's used. Each public property in each class maps to a field with the same name in the definition of your index.

There are 3 field types that you can use for Cognitive Search:

1. **SearchField** - This is the base class. Most properties are set to null, expect ```Name``` which is required.
2. **SimpleField** - This is a helper model. It can be any data type, is always non-searchable (meaning that the field is ignored for full text search queries), and is retrievable (not hidden).
3. **SearchableField** - Another helper model, expect it must be a string type and is always searchable and retrievable.

With our index created, we can now create our data source for our search indexer so that we can pull data from our storage account into our index, which we can do so by writing the following:

```csharp
SearchIndexerDataSourceConnection blobDataSource = new SearchIndexerDataSourceConnection(
    name: configuration["BlobStorageAccountName"],
    type: SearchIndexerDataSourceType.AzureBlob,
    connectionString: configuration["BlobStorageConnectionString"],
    container: new SearchIndexerDataContainer("hotel-rooms"));

await searchIndexerClient.CreateOrUpdateDataSourceConnectionAsync(blobDataSource);
```

Here, we create the data source by creating a **SearchIndexerDataSourceConnection** object by giving it a name, the container of our Blob Storage account, the connection string of the storage account and giving the **SearchIndexerDataSourceConnection** the type of ```SearchIndexerDataSourceType.AzureBlob```. There are [several types of data sources](https://learn.microsoft.com/en-us/dotnet/api/azure.search.documents.indexes.models.searchindexerdatasourcetype?view=azure-dotnet#properties) that you can use, including:
- Azure Data Lake
- Cosmos DB
- Azure SQL
- Table and Blob Storage
- Azure MySQL

Now that our data source has been created, we can set up an indexer that pulls data from our blob storage account, like so:

```csharp
IndexingParameters indexingParameters = new IndexingParameters()
{
    IndexingParametersConfiguration = new IndexingParametersConfiguration()
};
indexingParameters.IndexingParametersConfiguration.Add("parsingMode", "json");

SearchIndexer blobIndexer = new SearchIndexer(name: "hotel-rooms-blob-indexer", dataSourceName: blobDataSource.Name, targetIndexName: indexName)
{
    Parameters = indexingParameters,
    Schedule = new IndexingSchedule(TimeSpan.FromDays(1))
};

blobIndexer.FieldMappings.Add(new FieldMapping("Id") { TargetFieldName = "HotelId" });

try
{
    await searchIndexerClient.GetIndexerAsync(blobIndexer.Name);
    await searchIndexerClient.ResetIndexerAsync(blobIndexer.Name);
}
catch (RequestFailedException ex) when (ex.Status == 404) { }

await searchIndexerClient.CreateOrUpdateIndexerAsync(blobIndexer);

try
{
    await searchIndexerClient.RunIndexerAsync(blobIndexer.Name);
}
catch (RequestFailedException ex) when (ex.Status == 429)
{
    Console.WriteLine($"Failed to run indexer: {ex.Message}");
    throw;
}
```

There's a lot going on here, so let's break it down.

In our JSON blobs, we have a field called ```Id``` instead of ```HotelId```. If we have documents that don't match fields in our index, we can use the ```FieldMapping``` class to tell our indexer to direct the ```Id``` field value to our ```HotelId``` property.

The ```IndexingParameters``` object can be used to specify a parsing mode. In Blob Storage, this is handy depending on if you have blobs that represent multiple documents in the same blob, or if they are represented in a single blob.

You can also define a schedule for the indexer. In this tutorial, we are defining a schedule to run the indexer once a day. You can also remove this if you want to run this manually yourself.

*Note* - Since we're running the free tier here, there's a limit of running the indexer every 180 seconds.

Once the indexer has been created, we can see if it has been successful via the portal. Her we can see when it was last run, how many documents were successfully pulled into the index and if there's any errors.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lf7f6l4bzx3dl3vzg4lz.png)

We can dive into the indexer and see the execution history of the indexer itself. Here we can see the number of runs that were successful or failures, when they were last run, how long it look etc.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ewyigyc2qvkj0754rz5.png)

With our blob documents being pulled into the index, we can see them and do some basic queries in the portal.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p9g45o9hhej6ozqoe6vf.png)

We can also view the fields that are available to us in the search index, whether they are retrievable or not, filterable, sortable etc.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5l5nm547powb9vwhh68o.png)

### Querying our Cognitive Search service

Now that we have some data in our index, we can start to write some queries for them. First, we'll need to set up a ```SearchClient``` that will allow us to make queries against our indexes.

For this, we need to pass the URI for our Cognitive Search service, the name of the index that we will query against, and the API key of our Search service.

Once we've set up our ```SearchClient```, we can start to define our query. We create a ```SearchOptions``` object that we can use to specify options such as sorting, filters, paging etc. 

We then execute our query using the ```Search``` method. We pass in the search text to use as a string, along with the search options that we created earlier. We then use our ```Hotel``` type as the type parameter for our ```Search``` method, which will deserialize the documents in the results into ```Hotel``` objects.

```csharp
SearchClient searchClient = new SearchClient(new Uri(searchServiceUri), indexName, new AzureKeyCredential(searchServiceApiKey));

Console.WriteLine("Query 1: Return all documents, returning only HotelId and HotelName fields");

SearchOptions options = new SearchOptions()
{
    IncludeTotalCount = true,
    Filter = "",
    OrderBy = { "" }
};

options.Select.Add("HotelId");
options.Select.Add("HotelName");

SearchResults<Hotel> result = searchClient.Search<Hotel>("*");

foreach (var doc in result.GetResults())
{
    Console.WriteLine($"Hotel Id: {doc.Document.HotelId} | Hotel Name: {doc.Document.HotelName}");
}
```

This query will return the following results:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/co3moamtol7x0cia8d3r.png)

We can change our ```SearchOptions``` object to pass in filters. So in the code snippet below, we pass in a filter to return hotels that have rooms with a rate of less than $100 per night:

```csharp
Console.WriteLine("Query 2: Find all hotels with rooms cheaper than $100 per night");

options = new SearchOptions()
{
    Filter = "Rooms/any(r: r/BaseRate lt 100)"
};
options.Select.Add("HotelId");
options.Select.Add("HotelName");

result = searchClient.Search<Hotel>("*", options);

foreach (var doc in result.GetResults())
{
    Console.WriteLine($"Hotel Id: {doc.Document.HotelId} | Hotel Name: {doc.Document.HotelName}");
}
```

This query will return the following results:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s03ihop9mn8lnyxu1cob.png)

Finally, we can pass in a search text as a string to search for documents with a specific term. For example, we can pass in the term *hotel* to search for all hotels that have the term *hotel* in their name like so:

```csharp
Console.WriteLine("Query 3: Search the HotelName field for the term 'hotel'");

options = new SearchOptions();
options.SearchFields.Add("HotelName");

options.Select.Add("HotelId");
options.Select.Add("HotelName");

result = searchClient.Search<Hotel>("hotel", options);

foreach (var doc in result.GetResults())
{
    Console.WriteLine($"Hotel Id: {doc.Document.HotelId} | Hotel Name: {doc.Document.HotelName}");
}
```

This query will return the following results:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tl55xvsi1cepk528cp6j.png)

## Conclusion

In this article, I talked about what Azure Cognitive Search can do, what components make up Cognitive Search and where it sits within our architecture. I then showed you how you can provision a Cognitive Search service using Bicep, how we can pull data from blob storage into our Cognitive Search service using indexers and how we can query our index using C#.

If you want to learn more about Cognitive Search, I recommend taking a look at the following:

- [What's Azure Cognitive Search?](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search)
- [How to use Azure.Search.Documents in a C# .NET Application](https://learn.microsoft.com/en-us/azure/search/search-howto-dotnet-sdk)
- [Data import in Azure Cognitive Search](https://learn.microsoft.com/en-us/azure/search/search-what-is-data-import)

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️