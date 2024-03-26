---
title: "Managing Secrets in Azure Container Apps"
date: 2024-03-26
draft: false
tags: ["Azure", "Azure Container Apps", "Bicep", "GitHub Actions", "DevOps"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8n6l3sttn0x63o6gq417.png
    alt: "Managing Secrets in Azure Container Apps"
    caption: "In Azure Container Apps, we can use secrets to secure sensitive configuration. We can reference secrets directly in our Container Apps, or reference a secret in Azure Key Vault"
---

Azure Container Apps allows your apps to secure sensitive configuration values as *secrets*. Once you define your secrets, you can pass them as configuration to revisions of your Container Apps, and as secured values to your scale rules.

In this article, I'll discuss what secrets are, where we can define secrets, and how we can reference them in our application's environment variables.

**If you want to watch a video that talks about these concepts, check out the video below!**

{{< youtube kml1WL30Cks >}}

## What are Secrets in Azure Container Apps?

Secrets are sensitive configuration values in our Container App applications. The secrets we defined are scoped at the application level, meaning that they aren't tied to a specific revision of our application.

Adding, removing and changing secrets don't generate new revisions of our application, and each app revision can reference one or more secrets. You can also have multiple revisions reference the same secret.

You will inevitably change the values of your secrets, but since changing secrets don't generate new revisions of your application, you can either deploy a new revision with the new secret value, or restart an existing revision.

If you need to delete secrets from your Container App, deploy a new revision that doesn't reference the secret, then deactivate all revision that reference the old secret.

## How can we define our secrets in Azure Container Apps?

Secrets are defined as a set of name/value pairs. We can either specify the secret in our Container App directly, or as a reference to a secret stored in Azure Key Vault.

### Storing the secret value in Container Apps

Using Bicep, we can store our secret value directly in our Container App like so:

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-08-01-preview' = {
  name: containerAppName
  tags: tags
  location: location
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      // Omitted Code  
      secrets: [
        {
          name: 'cosmos-db-connection-string'
          value: cosmosDbAccount.listConnectionStrings().connectionStrings[0].connectionString
        }
        {
          name: 'app-insights-key'
          value: appInsights.properties.InstrumentationKey
        }
        {
          name: 'app-insights-connection-string'
          value: appInsights.properties.ConnectionString
        } 
      ]
      activeRevisionsMode: 'Multiple'
    }
    // Omitted Code
}
```

Secrets are defined at the application level in the `resources.properties.configuration.secrets` section. So in this block, I've set the connection string to a Cosmos DB account directly in the `secrets` array.

## Referencing a secret from Azure Key Vault

We can also reference secrets that are stored in Azure Key Vault. Azure Container Apps will retrieve that secret from Key Vault, and then make it available to your Container App.

There's a couple of things that we need to do before we can retrieve secrets from Key Vault:

1. Enable Managed Identity in your Container App.
2. Grant the identity the role to read secrets from Azure Key Vault.

We need to grant our Container App the **Key Vault Secret User Role**. To do this in Bicep, we can do the following:

```bicep
var keyVaultSecretUserRoleId = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')

resource keyVaultSecretUserRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(containerApp.id, keyVaultSecretUserRoleId)
  scope: keyVault
  properties: {
    principalId: containerApp.identity.principalId
    roleDefinitionId: keyVaultSecretUserRoleId
    principalType: 'ServicePrincipal'
  }
}
```

In this block, we are giving our Container App the **Key Vault Secret User Role** using the role definition id (as a guid), over the scope of our Key Vault.

Once this is done, we can reference the secret in Key Vault like so:

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-08-01-preview' = {
  name: containerAppName
  tags: tags
  location: location
  properties: {
    // Omitted Code
      secrets: [
        {
          name: 'cosmos-db-connection-string-kv'
          keyVaultUrl: 'https://${keyVault.name}.vault.azure.net/secrets/CosmosDbConnectionString'
          identity: 'system'
        }  
      ]
      activeRevisionsMode: 'Multiple'
    }
    // Omitted Code
}
```

In this code block, we are referencing a secret for our Cosmos DB connection string in our Key Vault. We use `system` for our `identity` property. If we were to use a user managed identity, we can replace `system` with the identity's resource ID.

For our Key Vault secret URI, we can either reference a specific version of the secret, or the latest version of a secret.

If we don't specify the version of the secret in the URI, then the app will use the latest version that exists in the key vault. As newer versions of the secret are published, your application will retrieve the latest version within 30 minutes. 

If you want more control over which version of the secret is used, specify the version in the URL.

## How can we reference secrets in our environment variables?

Once we have defined our secrets, we can reference them in our Container Apps environment variables. When our environment variables reference a secret, its value will be the value defined in the secret.

To do this in our Bicep code, we can write the following:

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-08-01-preview' = {
  name: containerAppName
  tags: tags
  location: location
  properties: {
    // Omitted code
      secrets: [
        {
          name: 'cosmos-db-connection-string'
          value: cosmosDbAccount.listConnectionStrings().connectionStrings[0].connectionString
        }
        {
          name: 'app-insights-key'
          value: appInsights.properties.InstrumentationKey
        }
        {
          name: 'app-insights-connection-string'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'cosmos-db-connection-string-kv'
          keyVaultUrl: 'https://${keyVault.name}.vault.azure.net/secrets/CosmosDbConnectionString'
          identity: 'system'
        }  
      ]
      activeRevisionsMode: 'Multiple'
    }
    template: {
      containers: [
        {
          name: containerAppName
          image: 'mcr.microsoft.com/k8se/quickstart:latest'
          env: [
            {
              name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
              secretRef: 'app-insights-key'
            }
            {
              name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
              secretRef: 'app-insights-connection-string'
            }
            {
              name: 'COSMOS_DB_CONNECTION_STRING'
              secretRef: 'cosmos-db-connection-string'
            }
            {
              name: 'COSMOS_DB_CONNECTION_KV'
              secretRef: 'cosmos-db-connection-string-kv'
            }
          ]
        }
      ]
      // Omitted code
    }
  }
}
```

So in this block, we define environment variables that use secrets in the `secretRef` property. For example, the `COSMOS_DB_CONNECTION_KV` environment variable using the secret value of the `cosmos-db-connection-string-kv` secret.

When we look at our Container App in the portal, we can see that our secrets are defined under *Secrets*:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fzbr08nrcwnojczr22uj.png)

And then these secrets are referenced in our Container Apps environment variables:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ki5obpu4m545xkhrvyco.png)

## Conclusion

In this post, we talked about secrets in Azure Container Apps. Using Bicep, we can reference secrets directly in our Container App, and we can reference secrets that are stored in Azure Key Vault.

I'm building a sample application that uses Azure Container Apps for a video series on my YouTube, so here are a couple of resources that you can use to learn more about secrets:

- [Let's Build Azure Container Apps series](https://www.youtube.com/watch?v=6dDJpkLiIFU&list=PLvtybS2EHFJg459xQ6ApYUZrDae3lRfx4&ab_channel=WillVelida)
- [GitHub Code](https://github.com/willvelida/lets-build-aca)

If you want to learn more about secrets in Azure Container Apps, check out the following [documentation](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets?tabs=arm-template).

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️