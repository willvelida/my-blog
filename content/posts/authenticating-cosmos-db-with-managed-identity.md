---
title: "Using Managed Identities to authenticate with Azure Cosmos DB"
date: 2022-03-25
draft: false
tags: ["Azure","Azure Cosmos DB","Azure Functions","C#", "Bicep", "GitHub Actions", "Azure AD"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/brunyulntyeh5xg3md5m.png
    alt: "Cosmos DB, Managed Identities, Functions logo"
    caption: 'We can make authenticated calls to Cosmos DB using System Assigned Managed Identities instead of using connection strings.'
---

In Azure, Managed Identities provide our Azure resources with an identity within Azure Active Directory. We can use this identity to authenticate with any service in Azure that supports Azure AD authentication without having to manage credentials. In Azure Cosmos DB, we can use managed identities to provide resources with the roles and permissions required to perform actions on our data (depending on what role we provide the identity) without having to use any connection strings or access keys to do so.

In this post, I'll show you how we can use Managed Identities to access our data in Azure Cosmos DB via an Azure Function. In this article, we will cover:

- Why we would use a Managed Identity over a connection string.
- How we can create a Cosmos DB account with a System-Assigned Managed Identity with Bicep
- How we can create an Azure Function with a System-Assigned Managed Identity with Bicep
- Create role assignments in Bicep
- Configure our CosmosClient to use our Managed Identity.
- Test our Function to add and read data using the Managed Identity

If you want to see the full code sample for this post, check out this [GitHub repo](https://github.com/willvelida/azure-samples/tree/main/cosmosdb-function-managed-identity).

You can also deploy the sample directly to your Azure Subscription by clicking on the button below:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fwillvelida%2Fazure-samples%2Fmain%2Fcosmosdb-function-managed-identity%2Fdeploy%2Fazuredeploy.json)

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

When we deploy our Cosmos DB account, we should see that a System-Assigned Identity for our account has been created by navigating to **Identity** in the sidebar. We'll see that an *Object Id* or *Principal Id* has been generated for our Cosmos DB account (I've blanked it out in the below picture, but it will be a randomly generated GUID):

![system-assigned identity in Cosmos DB](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/crguw31i4s6ycuscl0ga.png)

The *Object Id* is a unique value for an application object that uniquely identifies the object in Azure AD. This Object Id that we have generated will uniquely identify our Azure Cosmos DB account.

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

Again, in our Bicep code we are using the ```identity``` block and creating a managed identity of type ```SystemAssigned```.

Similar to our Cosmos DB account, we can find the Object Id of our Azure Function by navigating to **Identity** in the sidebar:

![system assigned identity Azure Function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bhysdaohhlcnfenemsts.png).

Now that we have enabled our System-assigned identities for both our Cosmos DB and Azure Function, we can now create our role assignments that will allow our Function to perform operations against our Cosmos DB account without having to use the connection string.

## Creating Role Assignments

Azure Cosmos DB provides a number of built-in roles that allow us to authorize and authenticate data requests using Azure AD identities in a granular manner.

We provide our identities with role definitions that allow them to perform a certain list of allowed accounts. We can apply these roles at the account, database or container level.

The list of these built-in role definitions can be found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac#permission-model).

For the purposes of this article, we're going to be creating a Custom Role that includes the following actions that we will allow our role to perform over our data:

```bicep
var dataActions = [
  'Microsoft.DocumentDB/databaseAccounts/readMetadata'
  'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*'
]
```

When we make calls to Cosmos DB using the .NET SDK, the SDK issues read-only metadata requests to serve specific data requests. This includes metadata like the partition key you've set on your containers, the list of Azure regions that the account is set in etc.

Since we are using the .NET SDK to make calls to our Cosmos DB account, we'll need to grant the System-Assigned identity the ability to perform actions that need this permission enabled.

We'll also be performing operations on our items in our containers, so we grant the ```Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*``` permission to our Function so it's able to do so.

We can define our sql roles in Bicep like so:

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

You may have noticed earlier in our App Settings for our Function, I've added a setting called ```CosmosDbEndpoint```. Instead of using our App Setting ```CosmosDbConnectionString``` which contained our connection string earlier, we can now just use our endpoint:

```csharp
return new CosmosClient(configuration["CosmosDbEndpoint"], new DefaultAzureCredential(), cosmosClientOptions);
```

Our Cosmos DB endpoint will look like this ```https://<account-name>.documents.azure.com:443/```. Making an unauthorized call to this endpoint returns the following response:

```json
{
  "code":"Unauthorized",
  "message":"Required Header authorization is missing. Ensure a valid Authorization token is passed.\r\nActivityId: 8a9c8eb2-1915-4466-bc8b-e53eaa768965, Microsoft.Azure.Documents.Common/2.14.0"
}
```

In order to make an authorized call, we pass in a ```new DefaultAzureCredential()``` into our ```CosmosClient```. This provides a default authentication flow for our application. In other words, this will attempt to authenticate our Azure Function to Azure Cosmos DB using the managed identity that we have assigned it. Since we have created a role assignment for our Azure Function, our Function will be authorized to perform operations against our Cosmos DB account.

## Testing our Function

Now that everything has been set up, we can test our Function and make sure that it can perform operations against our Cosmos DB account.

For this test, I have a simple [Function](https://github.com/willvelida/azure-samples/blob/main/cosmosdb-function-managed-identity/src/Todo.Api/Todo.Api/Functions/CreateTodoItem.cs) that uses a HTTP Trigger to make a POST request and add a simple Todo Item into our Cosmos DB container.

In the Azure Portal, we can navigate to this Function and test it out. I pass in the below JSON payload that represents the item that I want to persist to Cosmos DB:

![making a request with our function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7gbmneot3fge78wog2w4.png)

We should receive a 200 OK response, along with our Todo Item that we've just created in Azure Cosmos DB:

![successful request](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a9vwpt7je86vpi4dihz8.png)

Let's navigate to our Container in Cosmos DB. In the Bicep template, I created a ```todos``` container in a ```TodoDB``` database. Navigate to the container and we should see that the Todo item that we created has been successfully persisted to Azure Cosmos DB.

![item in Cosmos DB](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qrlr8uqosqdi6mn6pk0j.png)

## Conclusion

As we've seen in this post, we can use a combination of managed identities and role assignments to authenticate to Azure Cosmos DB without having to use the connection string in our applications. 

In this post, we used Azure Functions as an example, but we could do this for any [Azure resource that supports managed identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-status).

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️