---
title: "Scaling in Azure Container Apps"
date: 2022-12-17
draft: false
tags: ["Azure","Azure Container Apps","Kubernetes", "Bicep", "Containers", "KEDA"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7klbbyceqsonj6robnd9.png
    alt: "Scaling in Azure Container Apps"
    caption: 'Azure Container Apps uses KEDA to enable automatic horizontal scaling based on HTTP/TCP traffic, CPU/Memory usage, or events supported by KEDA'
---

Azure Container Apps manages automatic horizontal scaling through declarative scaling rules. We can scale out our container apps based on a variety of different scalers.

I recorded a video over the weekend covering how scaling works in Azure Container Apps, what KEDA is and how to set scaling rules within Azure Container Apps, using HTTP and Azure Queue storage scale rules as an example.

{{< youtube EYtNeHF5Mm8 >}}

If you prefer reading rather than watching, continue reading this post. I'll only be covering the storage queue scaler in this post, so if you want to see how the HTTP scaler works, please watch the video.

## How does scaling work in Container Apps?

As I mentioned earlier, Container Apps horizontally scale based on the scaling rules that have been set for that container app. As it scales out, new instances (or replicas) of the container app are created.

There are two scale properties that apply to all the scale rules that we define for a container app:

- **minReplicas**: This is the minimum number of replicas for a container app.
- **maxReplicas**: This is the maximum number of replicas for a container app.

With some container app triggers, we can scale to zero instances. This can be cost-efficient if we have an app that isn't used often, since we won't be billed for usage charges. However, this can create cold-starts for our application. If you want an instance of your app to always run, set the **minReplicas** count to 1. The flip side to this is that if you have replicas that are not processing anything, but are running in memory, you will be billed for 'idle charges'.

Scaling rules in Container Apps are bound to a particular revision. When we change our scaling rules, a new revision of the container app is created. If you're splitting traffic between different revisions, bear in mind that one revision will scale differently depending on the scale rules you set on that revision.

Until App Service Plan, there's currently no support to scale a container app up.

## What scale triggers do Container Apps support?

Azure Container Apps supports the following scale triggers:

- **HTTP and TCP traffic**: These rules will scale the container app based on the number of concurrent HTTP or TCP requests to your container app.
- **Event-driven**: These rules will scale the container app based off events that are supported by KEDA. For example, messages from Azure Storage Queue, or blobs from Azure Blob Storage. The full list of KEDA scalers can be found [here](https://keda.sh/docs/2.9/scalers/).
- **CPU or Memory Usage**: The rules will scale the container app based on the amount of CPU or memory consumption. With these scaling rules, you **cannot scale to zero**. You will need to have at least 1 replica of your application running.

## What is KEDA?

KEDA stands for **K**ubernetes-based **E**vent **D**riven **A**utoscaling. Using KEDA, you can scale containers in Azure Container Apps based on the number of events that need to be processed.

In a Kubernetes world, KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster.

KEDA has a wide range of scalers that it supports. Azure Container Apps supports KEDA ScaledObjects and all of the available scalers.

To learn more about KEDA, check out the [documentation on their website](https://keda.sh/docs/2.9/concepts/).

## Queue Storage Example

For this example, I'll be focusing on the important peices of code of [this sample](https://github.com/willvelida/container-apps-scaling). Please download it, deploy it, and give it a try!

Take a look at the following Bicep code:

```bicep
resource queueReader 'Microsoft.App/containerApps@2022-03-01' = {
  name: queueAppName
  location: location
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      activeRevisionsMode: 'Single'
      secrets: [
        {
          name: 'queueconnection'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
      ]
    }
    template: {
      containers: [
        {
          image: 'mcr.microsoft.com/azuredocs/containerapps-queuereader'
          name: queueAppName
          env: [
            {
              name: 'QueueName'
              value: storageQueue.name
            }
            {
              name: 'QueueConnectionString'
              secretRef: 'queueconnection'
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'myqueuerule'
            azureQueue: {
              queueName: storageQueue.name
              queueLength: 10
              auth: [
                {
                  secretRef: 'queueconnection'
                  triggerParameter: 'connection'
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

Within the ```resources.properties.template.scale``` section of our template, we are defining a scale rule that will scale out our container app for every 10 messages that this app reads from a storage queue.

Each event type features different properties in the ```metadata``` section of the KEDA definition. So if we were to define this in KEDA, it would look something like the following:

```
triggers:
- type: azure-queue
  metadata:
    queueName: queue-name
    queueLength: '10'
    connectionFromEnv: queueconnection
    accountName: storage-account-name
```

KEDA scale rules are defined in YAML. When you are configuring scale rules based off KEDA events, take a look at the [scaler documentation](https://keda.sh/docs/2.9/scalers/), and use that to help define your scale rules in Container Apps.

This scale rule will scale the container app based on the following behavior:

- For every 10 messages placed in the queue, create a new replica.
- To authenticate to the queue, we pass in the ```secretRef``` that contains the connection string to the storage account that we defined as a secret in our template.

At the time of writing, **Managed Identity authentication is not supported for scaling rules**.

When we deploy our Bicep template, we can see the scale rule defined in **Scale** under **Application** within our container app.

![Scale rule in Azure Container App](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uzd7mcep6zcvkuheg8ak.png)

As messages get sent to the queue, the number of replicas will increased until processing is complete, then the replicas will scale back down to the defined ```minReplicas``` count. In our case, this is 1 replica. You can see the replica count change by heading into your Container App and navigating to **Metrics** under **Monitoring**

![Replica Count change in Azure Container App metrics](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1hikql6gyf0f3r9bqzuf.png)

## Conclusion.

In this article, we talked about how scaling works in Azure Container Apps, what we can scale on and we walked through an example of scaling a Container App based on messages being sent to a Azure Storage Queue.

If you have any question about this video or Azure Container Apps in general, feel free to comment below or reach out to me on [Twitter](https://twitter.com/willvelida).

Until next time, Happy coding! 🤓🖥️