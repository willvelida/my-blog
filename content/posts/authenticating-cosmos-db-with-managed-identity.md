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

Bringing this back to Azure Cosmos DB, we can use 

## Creating our Cosmos DB account with a System-Assigned Managed Identity

## Creating our Azure Function with a System-Assigned Managed Identity

## Creating Role Assignments

## Configuring our CosmosClient to use Managed Identities

## Testing our Function

## Conclusion