---
title: "Making sense of your logs with Log Analytics: A Beginner's Guide"
date: 2023-04-26
draft: false
tags: ["Azure","Azure Monitor","Log Analytics"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y5xq8jse89z335fsb5ss.png
    alt: "Availability tests in Application Insights"
    caption: 'Automate your availability tests in App Service with Bicep'
---

## What is Log Analytics?

Azure Log Analytics is a tool as part of Azure Monitor that we can use to query data stored in the Azure Monitor Logs store. As administrators or developers, we would use Log Analytics in the Azure portal to configure input data sources (such as Container Apps, App Service, Cosmos DB etc.) and then query thier Azure Monitor logs to gain insights on those resources.

If you've worked with Azure for while, you've probably heard of (or even used) Log Analytics. Sending logs to Log Analytics is relatively straight forward, but many people just use Log Analytics as a dumping ground for their logs, and don't take advantage of all its capabilities.

In this article, I'll start by highlighting why you would use Log Analytics, then dive into creating a workspace using Bicep, sending logs from Azure resources to our workspace (I'll use Container Apps for this article) and how we can query our logs using Kusto queries (KQL for short).

Let's dive in!

## Why Log Analytics?

As I mentioned earlier, Log Analytics is a tool for Azure Monitor that we can use in the Azure Portal to query our log data that's collected in Azure Monitor logs.

When querying our data in Log Analytics, we use the Kusto Query Language (KQL), which can be used to perform simple or complex queries. Some examples includes:

- Aggregating results from large data sets.
- Join data from mulitple tables
- Searching by time, value, state etc.

Azure Monitor logs can contain a significant amount of data. If you understand how to query this data correctly, Log Analytics can provide extensive details of what's going on in your Azure workloads, enabling you to solve issues and problems quickly.

For example, say you're running a multitenanted solution that's hosted on Azure Container Apps. Each customer has their own dedicated Container Apps environment (purely for network isolation purposes) and you notice that one customer is running the maximum number of replicas for their Container Apps thanks to their HTTP traffic. Using this information, you can use Log Analytics to query all of your customers environments to see if there are any other customers who are experiencing this, see if there are any patterns or trends and adjust our scaling rules or maximum replica count accordingly.

Let's use this example to create a Log Analytics workspace in Bicep, configure a Container App environment to send logs to our Log Analytics workspace and use the features in Log Analytics to analyze our data. For this article, I'm going to use Bicep, but you can create workspaces using Terraform, or via ClickOps in the Azure Portal (Not mocking ClickOps, it's a great way to start out! But as you progress and get more comfortable with resources, you should look to adopt an Infrastructure-as-Code tool).

## Creating a workspace with Bicep

To create a Log Analytics workspace in Bicep, we can write the following:

```bicep
@description('Specifies the name of the Log Analytics workspace')
param logAnalyticsName string

@description('Specifies the location for our Log Analytics workspace')
param location string

@description('Specifies the tags that will be applied to this Log Analytics workspace')
param tags object = {
    Environment = 'Production'
}

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: logAnalyticsName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    features: {
      immediatePurgeDataOn30Days: true
    }
    workspaceCapping: {
      dailyQuotaGb: 5
    }
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}
```

Here we're defining a very simple Bicep resource block for our workspace. In this resource, I'm defining:

- The name of my Log Analytics workspace
- The tags I want to apply to the resource
- Which location to deploy the workspace to.
- The SKU (Stock Keeping Unit)
- Enabling public network access for ingestion and querying.
- A 5GB daily volume cap for ingestion.
- Purging the data from my workspace after 30 days.

Log Analytics has various different SKUs on offer, which I won't go into too much detail here as I want to focus on the technical stuff. The SKU I've chosen 'PerGB2018' will essentially charge me a certain amount per GB of logs that I'm storing. For full pricing details for Log Analytics, check [this page](https://azure.microsoft.com/en-us/pricing/details/monitor/#pricing).

*Before we continue*, I wrote an [article](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/scenarios-monitoring) sometime back on creating monitoring resources in Bicep on Microsoft Docs. This is something that a lot of engineers don't do, but should! Say if Azure has a regional failure, and you need to stand up a new environment quickly to ensure business continuity, it'll be easier to deploy a template with all your monitoring resources and rules already defined, rather than trying to stand everything up again with ClickOps!

For other Log Analytics properties you can configure in Bicep, check out the [reference documentation](https://learn.microsoft.com/en-us/azure/templates/microsoft.operationalinsights/workspaces?pivots=deployment-language-bicep)

## Sending logs to your workspace

With your workspace defined, you can send logs from your Container App environment to your Log Analytics workspace.

**There is a bit of a catch here**. Log Analytics is the default storage and viewing option for logs in Container Apps. Different resources in Azure emit different kinds of diagnostic logs, which can be sent to different locations. You can store these in a Azure Storage account, stream to Azure Event Hubs or send them to a partner solution.

This means that the way you can currently configure logs from your Container App environment to Log Analytics isn't the same as we would normally configure it in Bicep. Let's configure the logs for your Container App environment to be sent to Log Analytics:

```bicep
resource containerAppEnv 'Microsoft.App/managedEnvironments@2022-10-01' = {
  name: containerAppEnvName
  location: location
  sku: {
    name: 'Consumption'
  }
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}
```

As you can see, this is pretty straight forward. In the ```appLogsConfiguration``` property, I'm defining Log Analytics as the destination of our logs, and providing the customer ID and shared key from my Log Analytics resource that I defined earlier so that logs will be sent to that workspace.

For other Azure resources, we can configure sending logs to Log Analytics something like so:

```bicep
var appPlanSkuName = 'S1'

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2021-12-01-preview' existing = {
  name: logAnalyticsWorkspace
}

resource appServicePlan 'Microsoft.Web/serverfarms@2021-03-01' = {
  name: appPlanName
  location: location
  sku: {
    name: appPlanSkuName
    capacity: 1
  } 
}

resource diagnosticLogs 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: appServicePlan.name
  scope: appServicePlan
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'AllMetrics'
        enabled: true
        retentionPolicy: {
          days: 30
          enabled: true 
        }
      }
    ]
  }
}
```

This is a basic example of sending *'AllMetrics'* diagnostics from an App Service Plan. The important thing to note here is that the resource type ```Microsoft.Insights/diagnosticSettings``` is an extension resource, meaning that we apply these resources to another resource via the ```scope``` property. In this block, we use the ```workspaceId``` property to send our logs to our Log Analytics workspace by defining the id.

It's a bit confusing that there are two ways of doing things. AFAIK (and at the time of writing), Azure Monitor integration is still in preview for Container Apps. If/When it becomes generally avaialble, you'll be able to configure in the standard way using ```Microsoft.Insights/diagnosticSettings```. But for more, check out the following resources for more information:

- [Diagnostic settings in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/scenarios-monitoring#diagnostic-settings)
- [Microsoft.Insights diagnosticSettings](https://learn.microsoft.com/en-us/azure/templates/microsoft.insights/diagnosticsettings?pivots=deployment-language-bicep)
- [Microsoft.App managedEnvironments](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/managedenvironments?pivots=deployment-language-bicep) 

## Analyzing your logs with Kusto queries

Once you've deployed your Log Analytics workspace resource and configured a resource to send some logs to it, you can start to query your data using [Kusto Query Language (or KQL)](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/). When you query Log Analytics, you'll be querying dedicated tables that data is stored in. With Log Analytics, you can use it to query all of our resources in Azure, or you can scope it to a particular resource. 

When you start to write a KQL query, you'll start by defining the source table from which you'll be querying from, followed by a series of operators that you will use to query the source table.

KQL queries can have a chain of multiple operators that help refine the results of your query and enable you to perform advanced queries. You also have the ability to query and include data from multiple tables.

Let's illustrate this with a basic example. Container Apps expose two different log types that we can query in Log Analytics:

- Console Logs, which are emitted by your app.
- System Logs, which are emitted by the Container Apps Service.

Let's create a query that returns the last 100 logs in our System Logs that were emitted from our Container App. We can write this like so:

```kusto
ContainerAppSystemLogs_CL
| where ContainerAppName_s == 'hello-aca'
| take 100 
```

The ```take``` operator returns the last 100 logs, while the ```where``` operator filters our query to just show the logs from our ```hello-aca``` Container App. Our results should look like this:

![A picture of Log Analytics showing results from a KQL query](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tbjem5i00r2lxnr627s5.png)

You might be thinking that there's a lot data that our query has returned that doesn't mean a lot to us. We can refine our query to include certain columns and rename them to something a bit more meaningful. We can rewrite this like so:

```kusto
ContainerAppSystemLogs_CL
| where ContainerAppName_s == 'hello-aca'
| project Time=TimeGenerated, AppName=ContainerAppName_s, Revision=RevisionName_s, Message=Log_s
| take 100
```

![Our nicer looking query results](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a5du70gxrgrcr6n6sjlr.png)

As you can see, we now have a nicer result set, simply by using the ```project``` operator.

For the purposes of this article, I've kept the query simple. But what if this was a query that we ran quite often, and other members of our team wanted to run the same query. Rather than them having to write the same KQL query everytime, we can save the log query, by clicking on **Save** (highlighted in the red box below)

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d9lb77bv54y4n3r1g33k.png)

This will save the log query into something called a **query pack**. Saving our queries into query packs allow us to use the query in all Log Analytics contexts, whether that be in the workspace or when we are using Log Analytics scoped to a resource. Other users can use our query, and we can even create our own library of common queries that our team or organization can use.

With query packs, we can discover our queries by being able to filter and group queries by different properties. Our queries will also be available to us across subscriptions, which is great if you've set up different subscriptions for different environments (DEV, Prod, QA etc.).

When we save a query, we give it a name, a description and a category to store the query in:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sq1wn35z5nkp7bgdvdbf.png)

Once our query has been saved, we can search for our query in Log Analytics by searching for it under the *Queries* tab.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ptg1shd3wsnqw4c5dps.png)

We can then run the query as and when we need to. We also have the options of editing a saved query or modify any of the properties of the saved query itself.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0o7kw0fub07icqo38p58.png)

## Wrap up

This was a very gentle primer on getting started with Log Analytics. If you're keen to dive into Kusto queries a more, I recommend that you check out the following resources:

- [Structure Log Analytics queries](https://learn.microsoft.com/en-us/training/modules/configure-log-analytics/5-structure-queries) - This will help you structure your Log Analytics queries, with some common operators that you will use when performing KQL queries.
- [Kusto Query Language overview](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/) - Use this as a reference for all operators that you'll use for your KQL queries.
- [Design a Log Analytics workspace architecture](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design) - For all you architects thinking about using Log Analytics, this is a grat resource! You can't just dump logs somewhere and that's it, you'll need to think about security, data ownership, data retention and more, so this is a fantastic document for helping you figuring all that out.

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️