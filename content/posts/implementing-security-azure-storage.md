---
title: "Securing and managing access for Azure Storage"
date: 2026-11-07
draft: false
tags: ["Azure", "Security", "Azure Storage", "SC-500"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/cyw18735a0az70yd07q3.png
    alt: "Comparison of Azure Storage authorization models: Shared Key, service and account SAS, user delegation SAS, and Microsoft Entra ID with RBAC. A flow diagram shows Azure Functions accessing Storage using a managed identity and the Storage Blob Data Reader role."
    caption: "Azure Storage supports key-based, SAS-based, and identity-based authorization. Prefer Microsoft Entra ID with RBAC, using managed identities to avoid storing account keys in application configuration."
---

Azure Storage offers highly available, massively scalable, and durable storage for a variety of different data objects in the cloud. You can access this data from anywhere in the world over HTTPS via REST, or through client libraries using .NET, Python, Java, Node.js, and Go.

Azure Storage platform includes Blob data, which acts as a object store for text and binary data, Azure Files for file shares for cloud or on-prem deployments, Elastic Storage Area Network (SAN) data, Azure Queues, Azure Tables, managed disks, and more!

## Foundational Security

Let's take a look at the foundational security controls we can apply on the storage account to establish a baseline that protects our data in transit, enforces encryption, and prevents public exposure of our data.

### Data transfer

Starting with data transfer, Azure Storage supports both HTTP and HTTPS connections. HTTP is risky, because data transmitted over unencrypted connections is vulnerable to tampering and interception. To protect data in transit, we can require secure transfers and accept only HTTPS connections.

**Secure transfer required** is now enabled by default on all new storage account, however for some old storage accounts, they still allow HTTP. Check your storage account and switch this setting on if it's not already.

![Comparison of Azure Storage authorization models: Shared Key, service and account SAS, user delegation SAS, and Microsoft Entra ID with RBAC. A flow diagram shows Azure Functions accessing Storage using a managed identity and the Storage Blob Data Reader role.](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/tijho1kmpihhhiv1vuuh.png)

*Azure Storage supports key-based, SAS-based, and identity-based authorization. Prefer Microsoft Entra ID with RBAC, using managed identities to avoid storing account keys in application configuration.*

### TLS version

Transport Layer Security (TLS) is the encryption protocol that secures HTTPS connections. Older TLS versions, such as 1.0 and 1.1, have vulnerabilities and are deprecated, so we can specify the minimum TLS version that clients must use to **connect** to Azure Storage.

For example, if we set the minimum TLS version to 1.2, any client that attempts to connect with TLS 1.0 or 1.1 is rejected. This setting applies to all services in the storage account.

### Disabling anonymous access

Blob Storage supports anonymous public read access to containers and blobs, meaning that anyone with the blob or container URL can read the data without authentication. This can be handy for public website assets, but less desirable for business documentation.

By default, anonymous access is disabled at the account level, which provides defense at depth in case someone accidentally misconfigures a individual container to have public access (In this case, the account level permission would apply **over** the container level).

### Disabling cross-tenant replication

With Azure Blob Storage, you can use object replication to copy blobs from a storage account to a destination account. This allows you to replicate blobs to different accounts that reside in different Microsoft Entra tenants.

This is great for multitenant architectures, but if you misconfigured this replication, you can risk exfiltrating your own data to a unknown tenant. You should disable this capability entirely if you don't need it

### Bicep example

Let's create an example of all of these foundational settings in Bicep:

```bicep
param location string = resourceGroup().location
param storageAccountName string = 'st${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2025-06-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    allowCrossTenantReplication: false
  }
}
```

With these account-level settings in place, we can now implemented identity-based access controls.

## Authorization Models in Azure Storage

Azure Storage provides three authorization approaches that provide different security options. These three options are **Shared Key authorization**, **Shared Access Signatures (SAS)**, and **Microsoft Entra ID with RBAC**.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/tijho1kmpihhhiv1vuuh.png)

Shared Key auth uses the storage account access keys which grant full access to the entire account. Anyone who has these keys will have unrestricted access to all blobs, queues, tables, and files in the storage account.

Managing these keys is a pain. If they're exposed, you'll need to revoke them. If you revoke them, and you have workloads that use those keys, all of their access gets broken. Best not to use these keys at all.

### Shared Access Signatures

SAS keys provide a time-limited, granular access to specific storage resources. Azure Storage supports three types of SAS, each with different security implications.

1. **Service SAS**

- This targets a single service (Blob, Queue, Table, or File) and is signed with a storage account key.
- You specify the resource, permissions, and expiration time.
- SAS keys provide more limited access than full account key, but share the weakness that they're signed with the account key. If you need to revoke the SAS key, you'll run into the same issues as a Shared Key.

2. **Account SAS**

- This provides broader access across multiple services and resource type, also signed with a storage key.
- Account SAS tokens are useful for those cross-service operations, but again you'll run into the same revocation challenge as service SAS tokens.

3. **User delegation SAS**

- These are signed with Entra ID credentials instead of the storage account key.
- This combines the granular, time-limited access of SAS with the security benefits of identity-based authentication.
- When we create a user delegation SAS, Azure Storage will issue a user delegation key that's valid for up to 7 days.
- The SAS token is signed with this delegation key, not the storage account key.
- If we have to revoke the Microsoft Entra ID credentials used to create the delegation key, all associated SAS tokens immediately stop working.

### Microsoft Entra with RBAC

RBAC uses identities (users, groups, service principals, managed identities) and assigns them roles that define specific permissions. Unlike Shared Keys, RBAC provides per-identity access control with full audit trails that can be viewed via Azure Monitor.

Azure Storage provides several built-in RBAC roles for data access:

- **Storage Blob Data Reader** grants read access to containers and blobs.
- **Storage Blob Data Contributor** grants read, write, and delete access.
- **Storage Blob Data Owner** grants full control including ability to set permissions.

We can create these roles in our Bicep code, so for example:

```bicep
param location string = resourceGroup().location
param storageAccountName string = 'st${uniqueString(resourceGroup().id)}'

@description('Principal ID of the application managed identity.')
param principalId string

var blobReaderRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1'
)

resource storageAccount 'Microsoft.Storage/storageAccounts@2025-06-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    allowCrossTenantReplication: false
  }
}

resource blobReaderAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, principalId, blobReaderRoleId)
  scope: storageAccount
  properties: {
    roleDefinitionId: blobReaderRoleId
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}

output blobServiceEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

In our application code, we can use the Azure Identity library and the `DefaultAzureCredential` class to authenticate using RBAC. This can be typed to a managed identity of wherever we're running our workload (Azure Function, Web App, Container App, etc.)

In C#, this would look like the following:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

var serviceClient = new BlobServiceClient(
    new Uri("https://<storage-account-name>.blob.core.windows.net"),
    new DefaultAzureCredential());

var containerClient = serviceClient.GetBlobContainerClient("documents");
var blobClient = containerClient.GetBlobClient("example.txt");

var download = await blobClient.DownloadContentAsync();
Console.WriteLine(download.Value.Content.ToString());
```

This approach eliminates the need for a storage account key from our application configuration.

## Stored Access Policies

Stored access policies is a server-side policy associated with a container, queue, table, or file share that defines the start time, expiry time, and permissions of the policy. When we create a service SAS token, we can reference a stored access policy instead of embedding these parameters directly in the SAS token, which gives us the ability to modify or revoke the SAS after we issue it.

Any changes immediately affect all SAS tokens that reference the policy. Without a stored access policy, a SAS token's parameters are fixed at creation time, which can only be invalidated when we rotate the key.

We created stored access policies at the container level using the portal, Azure CLI, or Azure PowerShell. To create it using the CLI, we can do something like the following:

```bash
az storage container policy create `
  --account-name "<storage-account-name>" `
  --container-name "documents" `
  --name "vendor-read-policy" `
  --start "2026-04-01T00:00:00Z" `
  --expiry "2026-05-01T00:00:00Z" `
  --permissions "rl" `
  --auth-mode key
```

Here, we create a shared access policy over our `documents` container that allows both read and list operations. After we create the policy, you generate a service SAS token that references the policy identifier. 

```bash
az storage container generate-sas `
  --account-name "<storage-account-name>" `
  --name "documents" `
  --policy-name "vendor-read-policy" `
  --https-only `
  --auth-mode key `
  --output tsv
```

To revoke access, we can delete the stored access policy, or modify the parameters to deny access. Note that stored access policies work only with service SAS tokens. Account SAS and user delegation SAS tokens don't support stored access policies.

## Wrapping up

In this blog post, we talked about how we can address potential vulnerabilities in our Azure Storage account. We discussed how we can configure foundational security settings, such as requiring HTTPS, enforcing TLS 1.2, disabling anonymous access, and preventing cross-tenant replication.

We then talked about the different authentication models in Azure Storage, and why we should use scoped RBAC roles with Microsoft Entra ID over shared key authentication (you should disable this method of authentication entirely!)

If you have any questions, feel free to reach out to me on X [@willvelida](https://twitter.com/willvelida) or on [Bluesky](https://bsky.app/profile/willvelida.com)

Until next time, Happy coding! 🤓🖥️