---
title: "Building an event streaming app with Azure Functions, Event Hubs and Azure Cosmos DB"
date: 2022-04-24T14:33:51+12:00
draft: false
tags: ["Azure","Azure Functions","Azure Event Hubs","Azure Cosmos DB","C#", "Bicep", "Events"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g9jpaddirchndoazlzap.png
    alt: "Building an event streaming app with Azure Functions, Event Hubs and Azure Cosmos DB"
    caption: 'Using Azure Functions, we can integrate with Event Hubs and Cosmos DB to build high-scale streaming applications'
---

Back in 2020, I wrote an article on how you can build a simple streaming app using Azure Functions, Event Hubs and Azure Cosmos DB. I've been meaning to update some old samples that I created while I was an MVP, so the long weekend seemed like a good time to update this one.

Again, this is a relatively simple sample that I've created here. Previously, I developed all of this locally and created all my Azure resources via the portal. In this post, we'll be doing things a little differently. In this article we will:

- Outline what we'll be building.
- Create our resources using Bicep.
- Create a Function that will generate events.
- Create a Function that will listen to the events and persist them to Azure Cosmos DB.
- Test our Function App End-to-End.

## What are we building?

With Azure Functions, we can build event-driven applications that integrate different components teogether with ease. These range from simple APIs that react to HTTP events, to events that happen inside a database or we can build Functions that listen to messages or events on a queue and then perform some logic to react to that event.

In this sample, I'm going to create two Functions: One that generates events based on an HTTP event that sends events to Azure Events Hubs and the second will listen to the events and persist them into Azure Cosmos DB. 

For this sample, Let’s imagine we have various devices situated around the world that capture temperature, how much damage they’ve received by level and how long that device has been out in the field. We aren’t going to do too much logic based on this scenario, we’ll use this concept as a basis to demonstrate how generated events can be processed by Event Hubs.

Here's a basic diagram that demonstrates this:

![Event-streaming diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g9jpaddirchndoazlzap.png)

In this sample, I'll using C# to demonstrate a number of features in Azure Functions that you can use to integrate different components together. For Azure Cosmos DB, I'll show you how we can use dependency injection to inject our services into our Azure Functions and for Event Hubs, I'll show you how we can use the Output bindings for Event Hubs to send our events to the hub in our Event Hub namespace. We'll be connecting to both Azure Cosmos DB and Event Hubs using Managed Identities, which will eliminate the need for me to store any secrets within my Function App.

If you want to deploy this sample as we follow along, you can do so by clicking the button below. You will need your own Azure Subscription.

## Creating our infrastructure with Bicep

For our application, we'll be deploying the following resources:

- Azure Function App (along with a App Service Plan, Storage Account, App Insights instance)
- Azure Event Hubs Namespace (along with a event hub).
- Azure Cosmos DB account (along with a database and container).
- Our Azure Role Assignments.

### Event Hubs Namespace

To send our device reading events, we'll use Azure Event Hubs as our message broker. Events Hubs is a fully managed, real-time data ingestion service that scales (Event Hubs has the ability to process millions of events per second). In our application, we will create two functions: one that will produce events that sends data to our event hub and another that receives events from our event hub.

Here is our Bicep file for our Event Hubs Namespace:

```bicep
@description('Name of the Event Hubs Namespace')
param eventHubsName string

@description('The location to deploy the Event Hub Namespace to')
param location string

@description('The SKU that we will provision this Event Hubs Namespace to.')
param eventHubsSkuName string

@description('The name of our event hub that we will provision as part of this namespace')
param hubName string

resource eventHubNamespace 'Microsoft.EventHub/namespaces@2021-11-01' = {
  name: eventHubsName
  location: location
  sku: {
    name: eventHubsSkuName
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource eventHub 'Microsoft.EventHub/namespaces/eventhubs@2021-11-01' = {
  name: hubName
  parent: eventHubNamespace
  properties: {
    messageRetentionInDays: 1
  }
}

output eventHubNamespaceName string = eventHubNamespace.name
output eventHubNamespaceId string = eventHubNamespace.id
output eventHubName string = eventHub.name
```

I've written this as a module with two resources. Our first resource is our Event Hub Namespace. This is the management container for event hubs. The namespace is where we provision all our event hubs to and has a range of network and access control features that we can implement for our namespace.

For our namespace, I've kept it simple. I've given it a name, SKU and location and a System-Assigned identity. I'll talk about this a little more when we discuss role assignments.

I've also created an Event Hub that will be provisioned to our Event Hubs namespace. In Bicep, we use the ```parent``` keyword to indicate that we will provision our Event Hub to our namespace resource.

When we publish events to event hubs, we can configure how long they will stay on the hub until they are removed. This is out of scope for this article, so I just set it to a day.

For this module, I've created three outputs: One for the name of our namespace, one for the Id of the namespace, and another for the event hub name. More on this later.

### Azure Cosmos DB Account

To persist our events, we're going to use Azure Cosmos DB, which is Microsoft's globally distributed, multi-model database service that provides low latency and high availability.

Cosmos DB provides 5 different API's that we can use to store data in. Document data can be stored using the SQL or MongoDB API, graph data can be stored using the Gremlin API, tabular data using the Table API (Essentially a globally distributed Table Storage offering) or columnar data using the Cassandra API.

For this sample, we'll create our Cosmos DB account that uses the SQL API along with a database and a container that we will write our events to (If you're completely new to Cosmos DB, you can think of a container in Cosmos DB as a table, or if you're coming from MongoDB, it's a collection).

Here is our Bicep file for our Cosmos DB account:

```bicep
@description('The location that these Cosmos DB resources will be deployed to')
param location string

@description('The name of our Cosmos DB Account')
param cosmosDbAccountName string

@description('The name of our Database')
param databaseName string

@description('The name of our container')
param containerName string

@description('The amount of throughput to provision in our Cosmos DB Container')
param containerThroughput int

resource cosmosDbAccount 'Microsoft.DocumentDB/databaseAccounts@2021-11-15-preview' = {
  name: cosmosDbAccountName
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: true
      }
    ]
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2021-11-15-preview' = {
  name: databaseName
  parent: cosmosDbAccount
  properties: {
    resource: {
      id: databaseName
    }
  }
}

resource container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2021-11-15-preview' = {
  name: containerName
  parent: database
  properties: {
    options: {
      throughput: containerThroughput
    }
    resource: {
      id: containerName
      partitionKey: {
        paths: [
          '/id'
        ]
        kind: 'Hash'
      }
      indexingPolicy: {
        indexingMode: 'consistent'
        includedPaths: [
          {
            path: '/*'
          }
        ]
      }
    }
  }
}

output cosmosDbAccountName string = cosmosDbAccount.name
output databaseName string = database.name
output containerName string = container.name
output cosmosDbEndpoint string = cosmosDbAccount.properties.documentEndpoint
```

I've created another module for our Cosmos DB resources. First we create our Cosmos DB account. Again, we provide a name, location and enable a System-Assigned managed identity. Bicep requires us to provide an array of locations to our Cosmos DB accounts when creating one. This enables us to make our Cosmos DB account multi-regional, assign a failover priority to a region and making it zone redundant. For this demo, it's overkill so we just use our proivde our location as a region to provision this account a single region for now.

We then create our database for our Cosmos DB account and our container. The parent for our database will be the account and the parent for our container is our database.

Inside our container resource, we provision the throughput at the container level. In Azure Cosmos DB, we can provision throughput at both the database level (All containers provisioned in the database will share the throughput) or the container level (dedicated throughput for that container). We also specify the ```/id``` property as our partition key and our indexing policy is set to index on all properties in our document.

For this module, I've created 4 outputs: The name for our account, database and container and the endpoint of our Cosmos DB account. More on this later.

### Azure Role Assignments

Azure has a powerful role-based access control system. In Bicep, we can create our RBAC role assignments and definitions to include these RBAC controls within our Infrastructure code.

The ```roleAssignment``` resource type is an **extension resource** in Bicep, meaning that we can apply this resource to extend the capability to another resource.

Let's start with our Event Hub Role Assignments:

```bicep
@description('The Name of the Event Hubs Namepace')
param eventHubNamespaceName string

@description('The Id of the Function App')
param functionAppId string

@description('The Principal Id of the Function App')
param functionAppPrincipalId string

var eventHubsDataReceiverRoleId = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2b629674-e913-4c01-ae53-ef4638d8f975')
var eventHubsDataSenderRoleId = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'a638d3c7-ab3a-418d-83e6-5f17a39d4fde')

resource eventHub 'Microsoft.EventHub/namespaces@2021-11-01' existing = {
  name: eventHubNamespaceName
}

resource eventHubsDataReceiverRole 'Microsoft.Authorization/roleAssignments@2020-10-01-preview' = {
  scope: eventHub
  name: guid(eventHub.id, functionAppId, eventHubsDataReceiverRoleId)
  properties: {
    principalId: functionAppPrincipalId
    roleDefinitionId: eventHubsDataReceiverRoleId
    principalType: 'ServicePrincipal'
  }
}

resource eventHubsDataSenderRole 'Microsoft.Authorization/roleAssignments@2020-10-01-preview' = {
  name: guid(eventHub.id, functionAppId, eventHubsDataSenderRoleId)
  scope: eventHub
  properties: {
    principalId: functionAppPrincipalId
    roleDefinitionId: eventHubsDataSenderRoleId
    principalType: 'ServicePrincipal'
  }
}
```

Let's break this down a bit.

In this module, I've imported our Event Hub Namespace and used it in our ```scope``` property. Remember that role assignments are extension resources, so we need to use the ```scope``` property to apply these role assignments to our Event Hub Namespace.

A role assignment's resource name must have a gloablly unique identifier. It's good practice to create a guid that uses the scope, principal Id and role Id together, which we can achieve by using the ```guid()``` function.

We need to provide the **Event Hubs Data sender** and **Event Hubs Data receiver** roles to the Function App, since our Function App will have a function that sends the data from Event Hubs and a function that recieves data from Event Hubs. We first set these up as variables inside our module with the proper resource ids and then pass these variables to the ```roleDefinitionId``` property.

*There is a resource type that we can also use for this ```Microsoft.Authorization/roleDefinitions```. Either method is fine*.

The ```principalId``` property has to be set to a GUID that represents the Azure AD identifier for the principal. We set this up for our Event Hub Namespace and Cosmos DB account when we defined a System-Assigned identity block within our resources. For these roles, we pass in the ```functionAppPrincipalId``` parameter as the ``principalId``. When we create our Function, we'll be creating a System-Assigned identity for it so this is how we will pass that Object Id to our role assignment and provide the Function with these roles in Event Hubs.

The ``principalType`` property specifies whether the principal is a user, group or service principal. Since managed identities are a form of service principal, we set our ``principalType`` to 'ServicePrincipal`.

Now let's create our role assignments for Cosmos DB:

```bicep
@description('The name of the Cosmos DB account that we will use for SQL Role Assignments')
param cosmosDbAccountName string

@description('The Principal Id of the Function App that we will grant the role assignment to.')
param functionAppPrincipalId string

var roleDefinitionId = guid('sql-role-definition-', functionAppPrincipalId, cosmosDbAccount.id)
var roleAssignmentId = guid(roleDefinitionId, functionAppPrincipalId, cosmosDbAccount.id)
var roleDefinitionName = 'Function Read Write Role'
var dataActions = [
  'Microsoft.DocumentDB/databaseAccounts/readMetadata'
  'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*'
] 

resource cosmosDbAccount 'Microsoft.DocumentDB/databaseAccounts@2021-11-15-preview' existing = {
  name: cosmosDbAccountName
}

resource sqlRoleDefinition 'Microsoft.DocumentDB/databaseAccounts/sqlRoleDefinitions@2021-11-15-preview' = {
  name: '${cosmosDbAccountName}/${roleDefinitionId}'
  properties: {
    roleName: roleDefinitionName
    type: 'CustomRole'
    assignableScopes: [
      cosmosDbAccount.id
    ]
    permissions: [
      {
        dataActions: dataActions
      }
    ]
  }
  dependsOn: [
    cosmosDbAccount
  ]
}

resource sqlRoleAssignment 'Microsoft.DocumentDB/databaseAccounts/sqlRoleAssignments@2021-11-15-preview' = {
  name: '${cosmosDbAccountName}/${roleAssignmentId}'
  properties: {
    roleDefinitionId: sqlRoleDefinition.id
    principalId: functionAppPrincipalId
    scope: cosmosDbAccount.id
  }
}
```

Azure Cosmos DB exposes an RBAC system that lets users authenticate and authorize data requests using Azure AD. We essentially create role definitions containing a list of allowed actions that users are allowed to perform, such as reading items from a container or writing an item to a container.

In our role, create a custom SQL Role that allows the user to perform actions on items in our container and read the metadata from our account. Since we'll be using the Cosmos DB SDK in our function, we need to give our Function the ability to read the account metadata to fetch configuration details of that account such as the region that the account has been provisioned to and the partition key of our container.

You can read more about Metadata requests [here](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac#metadata-requests).

### Function App

We can now start to write our Bicep code for our Azure Function. For this we'll be creating an App Service Plan to host our Function, a Storage Account for our Function, an App Insights instance and the function itself.

Let's start with our App Service Plan:

```bicep
@description('The name of our App Service Plan')
param appServicePlanName string

@description('The location to deploy our App Service Plan')
param location string

@description('The SKU that we will provision this App Service Plan to.')
param appServicePlanSkuName string

resource appServicePlan 'Microsoft.Web/serverfarms@2021-03-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: appServicePlanSkuName
    tier: 'Dynamic'
  }
  properties: {
  } 
}

output appServicePlanId string = appServicePlan.id
```

This is a fairly straightforward module. We're hosting our Function App on the Consumption plan, so we apply the 'Dynamic' tier to our App Service Plan as well as passing through the SKU, name of the App Service Plan and location to our resource.

We have a single output for this module, which is the id of the App Service Plan. This will be used in our Bicep code for our Function.

Here's the remaining code we need for our Function App:

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-08-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

resource functionApp 'Microsoft.Web/sites@2021-03-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlan.outputs.appServicePlanId
    siteConfig: {
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: appInsights.properties.InstrumentationKey
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: 'InstrumentationKey=${appInsights.properties.InstrumentationKey}'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: functionRuntime
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'DatabaseName'
          value: cosmosDb.outputs.databaseName
        }
        {
          name: 'ContainerName'
          value: cosmosDb.outputs.containerName
        }
        {
          name: 'CosmosDbEndpoint'
          value: cosmosDb.outputs.cosmosDbEndpoint
        }
        {
          name: 'EventHubConnection__fullyQualifiedNamespace'
          value: '${eventHub.outputs.eventHubNamespaceName}.servicebus.windows.net'
        }
        {
          name: 'ReadingsEventHub'
          value: eventHub.outputs.eventHubName
        }
      ]
    }
    httpsOnly: true
  } 
  identity: {
    type: 'SystemAssigned'
  }
}
```

We define our Function App resource block by using the ``Microsoft.Web/sites`` type and setting the ``kind`` property to *functionapp*. We then use the App Service Plan Id (which we defined as an output in our App Service Plan module) as the ```serverFarmId``` property.

For our configuration, we use the storage account connection strings for our Function Storage, the App Insights instrumentation key to connect our Function to App Insights and set the Function Runtime to dotnet and v4+.

We use the outputs of our Cosmos DB module to set the Cosmos DB endpoint, Container Name and Database name so our Function can write items to our Cosmos DB account.

For Event Hubs, since we are using a managed identity to authenticate to it, we can set our environment variable ``EventHubConnection__fullyQualifiedNamespace`` to the endpoint of our namespace, instead of using a connection string. We'll need to install a package in our Function to make this work. We also define the event hub that we will be sending and receiving events from (Using an output from our Event Hub module).

When we deploy our Function, the configuration settings will look like this:

![Function Configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n0t1sp1i2gb8cf2s4imm.png)

As you can see, we just use the endpoints for our Cosmos DB account and the FQDN for our Event Hubs namespace. No secrets involved.

Our full ``main.bicep`` file should look like this:

```bicep
@description('The location where we will deploy our resources to. Default is the location of the resource group')
param location string = resourceGroup().location

@description('Name of our application.')
param applicationName string = uniqueString(resourceGroup().id)

@description('The SKU for the storage account')
param storageSku string = 'Standard_LRS'

var appServicePlanName = '${applicationName}asp'
var appServicePlanSkuName = 'Y1'
var storageAccountName = 'fnstor${replace(applicationName, '-', '')}'
var functionAppName = '${applicationName}func'
var functionRuntime = 'dotnet'
var cosmosDbAccountName = '${applicationName}db'
var databaseName = 'ReadingsDb'
var containerName = 'Readings'
var containerThroughput = 400
var appInsightsName = '${applicationName}ai'
var eventHubsName = '${applicationName}eh'
var eventHubsSkuName = 'Basic'
var hubName = 'readings'

module cosmosDb 'modules/cosmosDb.bicep' = {
  name: 'cosmosDb'
  params: {
    containerName: containerName 
    containerThroughput: containerThroughput
    cosmosDbAccountName: cosmosDbAccountName
    databaseName: databaseName
    location: location
  }
}

module appServicePlan 'modules/appServicePlan.bicep' = {
  name: 'appServicePlan'
  params: {
    appServicePlanName: appServicePlanName 
    appServicePlanSkuName: appServicePlanSkuName
    location: location
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-08-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

module eventHub 'modules/eventHubs.bicep' = {
  name: 'eventHub'
  params: {
    eventHubsName: eventHubsName 
    eventHubsSkuName: eventHubsSkuName
    hubName: hubName
    location: location
  }
}

resource functionApp 'Microsoft.Web/sites@2021-03-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlan.outputs.appServicePlanId
    siteConfig: {
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: appInsights.properties.InstrumentationKey
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: 'InstrumentationKey=${appInsights.properties.InstrumentationKey}'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: functionRuntime
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'DatabaseName'
          value: cosmosDb.outputs.databaseName
        }
        {
          name: 'ContainerName'
          value: cosmosDb.outputs.containerName
        }
        {
          name: 'CosmosDbEndpoint'
          value: cosmosDb.outputs.cosmosDbEndpoint
        }
        {
          name: 'EventHubConnection__fullyQualifiedNamespace'
          value: '${eventHub.outputs.eventHubNamespaceName}.servicebus.windows.net'
        }
        {
          name: 'ReadingsEventHub'
          value: eventHub.outputs.eventHubName
        }
      ]
    }
    httpsOnly: true
  } 
  identity: {
    type: 'SystemAssigned'
  }
}

module eventHubRoles 'modules/eventHubRoleAssignment.bicep'  = {
  name: 'eventhubsroles'
  params: {
    eventHubNamespaceName: eventHub.outputs.eventHubNamespaceName 
    functionAppId: functionApp.id
    functionAppPrincipalId: functionApp.identity.principalId
  }
}

module sqlRoleAssignment 'modules/sqlRoleAssignment.bicep' = {
  name: 'sqlRoleAssignment'
  params: {
    cosmosDbAccountName: cosmosDb.outputs.cosmosDbAccountName
    functionAppPrincipalId: functionApp.identity.principalId
  }
}
```

### Deploying our Infrastructure

Now that our Bicep code is complete, we can deploy it to Azure. For this article, we'll just use the AZ CLI to do so.

First up, we need to create a resource group to deploy our resources to. We can do this using the following command:

```bash
az group create -n <name-of-the-resource-group> -l <azure-region-of-choice>
```

Once our resource group has been created, we can deploy the resources. In the terminal, navigate to where your Bicep file is location and run the following:

```bash
az deployment group create --template-file main.bicep --resource-group <name-of-the-resource-group>
```

Give it a couple of minutes to deploy your resources. You should see your resoures deployed to Azure by navigating to your resource group and seeing the following:

![Azure Resources in Azure](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qjt20xksiicovc97aual.png)

Once everything has been deployed, we can verify that the role assignments for Function have been created so that the Function can send and receive events from our Event Hub. We can do this by navigating to the *Access Control* page in our Function App and clicking on *role assignments*. You should see something similar to the below:

![Event Hubs Permissions](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7vdj7ghqt33negaop9iu.png)

## Creating our Event Generator Function

Before we start writing our Functions, let’s explore the model that we’ll use for our scenario. Here I have a **DeviceReading** class that we’ll use for our scenario. In v3 of the Cosmos DB .NET SDK, we need to explicitly specify our id of the document that we are trying to persist. In order to do this, I have a property in our **DeviceReading** class called DeviceId which I have decorated with a JsonProperty called id. To do this, you’ll need to add a using statement for **Newtonsoft.Json**.

Here is the data model that I'll be using for my device reading:

```csharp
using Newtonsoft.Json;

namespace DeviceReaderSample.Models
{
    public class DeviceReading
    {
        [JsonProperty("id")]
        public string DeviceId { get; set; }
        public decimal DeviceTemperature { get; set; }
        public string DamageLevel { get; set; }
        public int DeviceAgeInDays { get; set; }
    }
}
```

We'll also need to install the following NuGet packages into our Function App:

- **Bogus**. This is a neat little library that allows us to generate random data based on our C# objects. I first saw this library in action when I was going through [the labs that the Cosmos DB Engineering team have put together](https://azurecosmosdb.github.io/CosmosDBWorkshops/). I’m just using it here to generate lots of events for our Device Readings that I can send to Event Hub.
- **Microsoft.Azure.Cosmos**. I’ll be using v3 of the .NET SDK to persist my events to my Cosmos DB collection.
- **Microsoft.Azure.WebJobs.Extensions.EventHubs** **USING VERSION 5+**. This version introduces the ability to connect to our Event Hub namespace using an identity instead of a secret. So we'll need to use this for our Function to work.
- **Azure.Identity**. This will enable us to connect to both our Cosmos DB account and Event Hub namespace using a credential for our System-Assigned identity.

### Injecting our services

In order to register my services, I’ll need to create a Startup class. Here’s the Startup class for my Function app.

```csharp
using Azure.Identity;
using DeviceReaderSample;
using Microsoft.Azure.Cosmos;
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.IO;

[assembly: FunctionsStartup(typeof(Startup))]
namespace DeviceReaderSample
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            var config = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
                .AddEnvironmentVariables()
                .Build();

            builder.Services.AddLogging();
            builder.Services.AddSingleton<IConfiguration>(config);
            builder.Services.AddSingleton(sp =>
            {
                IConfiguration configuration = sp.GetService<IConfiguration>();
                CosmosClientOptions cosmosClientOptions = new CosmosClientOptions
                {
                    MaxRetryAttemptsOnRateLimitedRequests = 3,
                    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(60)
                };
                return new CosmosClient(configuration["CosmosDbEndpoint"], new DefaultAzureCredential(), cosmosClientOptions);
            });
        }
    }
}
```

The main thing I want to highlight here is the following line:

```csharp
return new CosmosClient(configuration["CosmosDbEndpoint"], new DefaultAzureCredential(), cosmosClientOptions);
```

Instead of using a connection string which has the primary key in it, we use the endpoint of our Cosmos DB account, a **DefaultAzureCredential** object and our Cosmos DB client options. By using a **DefaultAzureCredential**, we are telling our client to use the System Assigned identity and the roles that we have granted that identity to authenticate to our Cosmos DB account. 

That means that the service principal can only perform operations on our Cosmos DB account with the permissions that we have granted it. This provides a more granular permissions model to our Function, instead of providing the connection string, which has a much wider scope of permissions.

Now that we’ve injected our services and got a model to work with, we can build our Function to send events to our Event Hub.

I’ve set up an **HttpTrigger** Function just so I can trigger the sending of events through an HTTP call.

In order to mock up events, I’m using **Bogus.Faker** to create a IEnumerable collection of Device Reading and defining the rules that I want to set for each property. We use the ```numberOfEvents``` parameter to set the number of events we want to generate.

I’m then iterating through my IEnumerable collection and converting each reading within that collection into an EventData type and sending that to my Event Hub.

```csharp
using Bogus;
using DeviceReaderSample.Models;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace DeviceReaderSample.Functions
{
    public class GenerateReadings
    {
        private readonly ILogger<GenerateReadings> _logger;

        public GenerateReadings(ILogger<GenerateReadings> logger)
        {
            _logger = logger;
        }

        [FunctionName(nameof(GenerateReadings))]
        public async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = "GenerateReadings/{numberOfEvents}")] HttpRequest req,
            [EventHub("readings", Connection = "EventHubConnection")] IAsyncCollector<DeviceReading> outputEvents,
            int numberOfEvents)
        {
            try
            {
                var deviceIterations = new Faker<DeviceReading>()
                .RuleFor(i => i.DeviceId, (fake) => Guid.NewGuid().ToString())
                .RuleFor(i => i.DeviceTemperature, (fake) => Math.Round(fake.Random.Decimal(0.00m, 30.00m), 2))
                .RuleFor(i => i.DamageLevel, (fake) => fake.PickRandom(new List<string> { "Low", "Medium", "High" }))
                .RuleFor(i => i.DeviceAgeInDays, (fake) => fake.Random.Number(1, 60))
                .GenerateLazy(numberOfEvents);

                foreach (var reading in deviceIterations)
                {
                    await outputEvents.AddAsync(reading);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(GenerateReadings)}: {ex.Message}");
                throw;
            }

            return new OkResult();
        }
    }
}
```

## Creating our Event Listener Function

Now let’s take a look at the function that will persist events to Cosmos DB:

```csharp
using Azure.Messaging.EventHubs;
using DeviceReaderSample.Models;
using Microsoft.Azure.Cosmos;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System;
using System.Text;
using System.Threading.Tasks;

namespace DeviceReaderSample.Functions
{
    public class PersistReadings
    {
        private readonly ILogger<PersistReadings> _logger;
        private readonly IConfiguration _configuration;
        private readonly CosmosClient _cosmosClient;
        private readonly Container _container;

        public PersistReadings(ILogger<PersistReadings> logger, IConfiguration configuration, CosmosClient cosmosClient)
        {
            _logger = logger;
            _configuration = configuration;
            _cosmosClient = cosmosClient;
            _container = _cosmosClient.GetContainer(_configuration["DatabaseName"], _configuration["ContainerName"]);
        }

        [FunctionName(nameof(PersistReadings))]
        public async Task Run([EventHubTrigger("readings", Connection = "EventHubConnection")] EventData[] events, ILogger log)
        {
            foreach (EventData eventData in events)
            {
                try
                {
                    string messageBody = Encoding.UTF8.GetString(eventData.EventBody.ToArray());

                    var telementryEvent = JsonConvert.DeserializeObject<DeviceReading>(messageBody);

                    // Persist to cosmos db
                    await _container.CreateItemAsync(telementryEvent);
                    _logger.LogInformation($"{telementryEvent.DeviceId} has been persisted");
                }
                catch (Exception ex)
                {
                    _logger.LogError($"Something went wrong. Exception thrown: {ex.Message}");
                }
            }
        }
    }
}
```

I’m also using constructor injection to make my dependencies available in this function, but this time I’m using my **CosmosClient**, then setting the Container that I want to use by retrieving the Container I want to persist my events to by using the ```.GetContainer()``` method. I need to pass through the database and the container name in this method, which I’m getting from my **IConfiguration** object.

For this Function, I’m using an **EventHubTrigger** which will trigger every time an event gets sent to our Event Hub. In order to connect to my Event Hub, I’ve specified the Event Hub name that processes our events and defining the connection to our Event Hub Namespace. This will send events into an array of EventData.

We will iterate through this array and for each event, we’ll try to convert out incoming event into a DeviceReading object, then persist that to our container in Cosmos DB.

### Deploying our Function

We can now deploy our function code to Azure! To do this, you can follow [this tutorial](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs?tabs=in-process) on Microsoft Docs to publish our code from Visual Studio.

## Testing our Function

Once our Function has been deployed. We can trigger our **GenerateReadings** function by navigating to it in the portal and clicking on *test*. We should see the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cpzv4c2mynirv3ax6xbs.png)

In the **Query** section, enter the number of events that you want to send to the Function by entering a number in the value box. Click **Run** to trigger the function.

This will send events to our **GenerateReadings** function, which will send events to our **readings** event hub. This will triggger our **PersistReadings** function which will write our events to our Cosmos DB account. Navigate to your Cosmos DB account and open up the container. You will see that events have been written to your Cosmos DB account like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dq94f38a2igh0l157dcf.png)

## Conclusion

In this article, we built a simple event streaming application using Azure Functions, Event Hubs and Azure Cosmos DB. We used Managed Identities to authenticate to our Cosmos DB and Event Hubs namespace instead of using secrets and we created our infrastructure using Bicep.

Even though this is a simple concept app, hopefully you now understand how we can build something like this using Infrastructure as Code, Managed Identities instead of secrets and dependency injection to inject our services instead of bindings.

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️