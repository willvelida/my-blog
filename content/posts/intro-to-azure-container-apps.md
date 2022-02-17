---
title: "Introduction to Azure Container Apps"
date: 2022-02-14T19:35:31+13:00
draft: true
ShowToc: true
TocOpen: true
---

Azure Container Apps is a serverless platform that allows us to run containerized applications. We can deploy our containers as if we were going to deploy them to Kubernetes, without actually having to manage or operate a Kubernetes cluster! 

Container Apps can dynamically scale based on HTTP traffic, Incoming events, CPU/Memory load or by using any [KEDA (Kubernetes Event-driven Autoscaling) supported scaler](https://keda.sh/docs/2.6/scalers/)

In this article, I'll introduce some key concepts of Azure Container Apps and show you how you can create a Container App using Bicep. A word of warning, at the time of writing this post Container Apps are still in preview, so if you're reading this in the future it's likely that a lot of this information will be out-of-date.

## Key Container Apps Concepts

### Container Apps Environments

With Azure Container Apps, we have individual containers that we will deploy to a single Container App environment. The environment acts a secure boundary around groups of container apps and they are deployed in the same vNET and they write logs to the same Log Analytics instance. This is great for grouping related services, deploying our apps to the same network and having our apps share the same log analytics workspace.

So if we were to visualize this:

[INSERT DIAGRAM HERE]

Here we have multiple Container Apps within our environment. The environment is the isolated boundary around our two container apps. When we deploy revisions of our Container Apps, the environment still acts as a boundary as our Container Apps change over time (we'll discuss revisions later).

If we're using Dapr, we can deploy our container apps to the same environment which can then communicate with each other using Dapr. This is also ideal when we have applications that share the same Dapr configuration.

If we have applications that don't share compute, or they don't/can't communicate with each other via Dapr, then these are good reasons to deploy our container apps to different environments.

### Containers in Container Apps

As I mentioned earlier, deploying apps to Container Apps is like deploying to Kubernetes, except we don't need to worry about managing the details of Kubernetes as this is done for us (That being said, if you do need to access the Kubernetes API as part of operating your applications, it's best to stick to Kubernetes).

Containers are grouped together in Pods inside a revision snapshot. Container Apps support Linux-based container images and we can deploy our images from public or private registries. 

We can even define multiple containers in a single container app. This is similar to how [Pods work in Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods), they share hard disk and network resources and go through the same lifecycle. Running multiple containers within the same Container App is good when we're implementing the sidecar pattern, sharing the same scaling rules among our containers or enabling communication on the same host among our containers.

At the time of writing, we can't run privileged containers and we are limited to Linux-based images.

### Revisions and Handling Lifecycle

In Container Apps, revisions are immutable snapshots of a container app. As we change the configuration of our Container App, new revisions are made. This is useful when we expose our Container Apps via HTTP. We can use revisions to direct traffic to different revisions, as we would in A/B testing or when conducting Blue/Green deployments.

When we change our Container App, we either make a *revision-scope* change or a *application-scope* change. 

Revision-scope changes are changes that will create a new revision (hence the name) of our Container App and include the following:

- Changes to our containers.
- Adding new or updating existing scaling rules
- If we're using Dapr, and we change our settings.
- Any change that affects the```template```section our our Container App configuration.

Application-scope changes include the following:

- Changes to our traffic splitting rules.
- Turning ingress for our Container App on or off.
- Changing secret values (The catch with this is that when we change our secrets, we need to restart the revision before our container uses the new values).
- Any change outside the```template``` section of our configuration.

We can configure our Container App to deactivate revisions automatically, or we can keep them in case we need to roll back to a specific version (Up to 100 revisions can be kept before Container Apps purge them). When we deactivate a revision, the container will be shut down. 

## Creating our Container App with Bicep

Let's dive into some code. First up, we'll need to provision the infrastructure for our Container App

Log Analytics Workspace

```bicep
@description('Name for the Log Analytics Workspace')
param lawName string

@description('Location that the Log Analytics Workspace will be deployed to')
param lawLocation string

resource law 'Microsoft.OperationalInsights/workspaces@2021-06-01' = {
  name: lawName
  location: lawLocation
  properties: {
    sku: {
      name: 'PerGB2018'
    }
  }
}

output clientId string = law.properties.customerId
```

Our container app environment:

```bicep
@description('Name of the Container App environment')
param environmentName string

@description('Location that the Environment will be deployed to')
param location string

@description('The name of the Log Analytics Workspace that this Environment will use')
param lawName string

resource law 'Microsoft.OperationalInsights/workspaces@2021-06-01' existing = {
  name: lawName
}

resource env 'Microsoft.Web/kubeEnvironments@2021-02-01' = {
  name: environmentName
  location: location
  kind: 'containerenvironment'
  properties: {
    type: 'managed'
    internalLoadBalancerEnabled: false
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: law.properties.customerId
        sharedKey: law.listKeys().primarySharedKey
      }
    }  
  }
}

output id string = env.id
```

Our container registry

```bicep
resource azureContainerRegistry 'Microsoft.ContainerRegistry/registries@2021-09-01' = {
  name: todoAcrName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

Our container app

```bicep
resource webContainerApp 'Microsoft.Web/containerApps@2021-03-01' = {
  name: 'todoweb'
  location: location
  properties: {
    kubeEnvironmentId: kubeEnvironment.outputs.id
    configuration: {
      ingress: {
        external: true
        targetPort: 80
      }
      secrets: [
        {
          name: 'acr-password'
          value: azureContainerRegistry.listCredentials().passwords[0].value
        }
      ]
      registries: [
        {
          server: azureContainerRegistry.properties.loginServer
          username: todoAcrName
          passwordSecretRef: 'acr-password'
        }
      ]
    }
    template: {
      containers: [
        {
          image: 'velidatodoacr.azurecr.io/todo-list:latest'
          name: 'todo-list'
          resources: {
            cpu: '0.5'
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 1
      }
    }
  }
}
```