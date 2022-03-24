---
title: "Using Managed Identities to authenticate with Azure Cosmos DB"
date: 2022-03-23T18:06:29+13:00
draft: true
tags: ["Azure","Azure Cosmos DB","Azure Functions","C#", "Bicep", "GitHub Actions", "Azure AD"]
ShowToc: true
TocOpen: true
cover:
    image: https://willvelidastorage.blob.core.windows.net/blogimages/containerappsgithubactions3.jpg
    alt: "Azure Container Apps Logo"
    caption: 'Using GitHub Actions, we can deploy new versions of our Container Apps as our images are updated'
---

In Azure, Managed Identities provide our Azure resources with an identity within Azure Active Directory. We can use this identity to authenticate with any service in Azure that supports Azure AD authentication without having to manage credentials. In Azure Cosmos DB, we can use managed identities to provide resources with the roles and permissions required to perform actions on our data (depending on what role we provide the identity) without having to use any connection strings or access keys to do so.

In this post, I'll show you how we can use Managed Identities to access our data in Azure Cosmos DB via an Azure Function. In this article, we will cover:

- Why we would use a Managed Identity over a connection string.
- How we can create a Cosmos DB account with a System-Assigned Managed Identity with Bicep
- How we can create an Azure Function with a System-Assigned Managed Identity with Bicep
- Create role assignments in Bicep
- Configure our CosmosClient to use our Managed Identity.
- Test our Function to add and read data using the Managed Identity

## Why Managed Identities?

As I mentioned earlier, we can use Managed Identities to provide our applications with an identity that uses Azure Active Directory to authenitcate to other resources that support Azure AD authentication. By using managed identities, we don't need to manage credentials, such as managed connection strings in Cosmos DB and there is no additional cost in using Managed Identities.

In Azure, we can create two types of managed identities; **System-assigned** and **User-assigned**. When we create a system-assigned managed identity, we create an identity within Azure AD which is tied to the lifecycle of that service. When we delete our service, the identity is also deleted. User assigned indentities are standalone resources which we can assign to one or more resources. This identity is managed seperately from our resources.

Bringing this back to Azure Cosmos DB, we can use built-in Azure roles or custom roles to grant or deny access to resources in our Cosmos DB account. This provides us with a mechanism to create granular access to specific identities with the access that they require, rather than using the admin connection string.

Allowing our clients to use the connection string to our Cosmos DB accounts carries a lot of risk. Let's illustrate this with an example using the C# SDK. If we have the connection string, we can create a connection to our Cosmos DB account like so:

```csharp
builder.Services.AddSingleton(sp =>
    {
        IConfiguration configuration = sp.GetService<IConfiguration>();
        CosmosClientOptions cosmosClientOptions = new CosmosClientOptions
        {
            MaxRetryAttemptsOnRateLimitedRequests = 3,
            MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(60)
        };
        return new CosmosClient(configuration["CosmosDBConnectionString"], cosmosClientOptions);
    });
```

Since we've connected to Cosmos DB using our admin connection string, our client can create databases in our accounts, read details about our Cosmos DB accounts and more. As developers, we need to ensure that clients that are making operations against our Cosmos DB accounts only perform operations that they are authorized to.

To that end, we will create our Cosmos DB account with a System-Assigned identity, which will allow us to assign granular roles and permissions for any clients that will perform operations in Cosmos DB.

## Creating our Cosmos DB account with a System-Assigned Managed Identity

We can create an Azure Cosmos DB account with a system-assigned identity like so:

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2021-10-15' = {
  name: cosmosDbAccountName
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: false
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
```



## Creating our Azure Function with a System-Assigned Managed Identity

We'll be performing operations against our Cosmos DB account via an Azure Function. To do this, we'll need to create an Azure Function that also has a System-Managed Identity

```bicep
resource functionApp 'Microsoft.Web/sites@2021-03-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlanId
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
          value: appInsightsInstrumentationKey
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: 'InstrumentationKey=${appInsightsInstrumentationKey}'
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
          value: databaseName
        }
        {
          name: 'ContainerName'
          value: containerName
        }
        {
          name: 'CosmosDbEndpoint'
          value: cosmosDbEndpoint
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

## Creating Role Assignments

```bicep
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

## Configuring our CosmosClient to use Managed Identities

```csharp
return new CosmosClient(configuration["CosmosDbEndpoint"], new DefaultAzureCredential(), cosmosClientOptions);
```

## Testing our Function

## Conclusion