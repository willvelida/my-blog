---
title: "Creating and Provisioning Azure Container Apps with Bicep"
date: 2022-02-18T20:47:30+13:00
draft: false
tags: ["Azure","Bicep","Serverless","Azure Container Apps","Containers", "Infrastructure as Code"]
ShowToc: true
TocOpen: true
cover:
    image: https://willvelidastorage.blob.core.windows.net/blogimages/deploycontainerappsbicep4.png
    alt: "Azure Container Apps Logo"
    caption: 'Using Bicep, we can deploy Azure Container Apps quickly and easily!'
---

Using Bicep, we can deploy and manage all the resources required for our Azure Container Apps. In this post, we'll write a Bicep template that defines all the infrastructure required for our Container App and deploy it using the AZ CLI. In our Bicep template, we'll be deploying the following resources:

- A Log Analytics workspace.
- Azure Container Registry
- An Azure Container App Environment
- A Container App.

Before we kick this off, it's **REALLY IMPORTANT** to note that Container Apps is **currently in preview**! That means what I publish in my present might/will change in your future! One example of this is that the namespace that Container Apps currently reside in will be changing in March 2022, as referenced in [this GitHub issue](https://github.com/microsoft/azure-container-apps/issues/109) (*Note to self, update this article!*)

## Creating our Container App resources in Bicep

Bicep is a domain-specific language that we can use as Infrastructure as Code (IaC) that Microsoft created to deploy Azure resources. Bicep code is complied to ARM templates and in my opinion, is a lot nicer to work with than ARM.

To work with Bicep, there's a fantastic [Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep) that we can use to work with Bicep. Open up Visual Studio Code and click the **extensions** tab. Enter *Bicep* into the search bar and you should see the extension.

![image](https://willvelidastorage.blob.core.windows.net/blogimages/deploycontainerappsbicep1.png)

To work with Bicep, you'll also need to have Azure CLI version 2.20.0 or later. Check out [this page](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/install) for more details on how to get Bicep set up on your machine.

Now that we have everything set up, we can start writing our Bicep templates with all the resources we need for our Container Apps.

### Log Analytics

Container Apps gathers data about your container app and stores it in Log Analytics. We can write single text strings or a line of serialized JSON data to Log Analytics. For our Container App we'll need to create a Log Analytics workspace to send our Container Apps logs to.

```bicep
@description('Name of the log analytics workspace')
param logAnalyticsName string

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2021-12-01-preview' = {
  name: logAnalyticsName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
  }
}
```

In this resource block, we're creating our Log Analytics workspace, providing it with the name of the workspace using a parameter, a location (which I'll discuss in a bit) and the SKU defined in the properties block.

### Azure Container Registry

Next up we will need to create an Azure Container Registry. In future posts, I'll show you how we can pull images from our container registry and deploy it to a Container App. But for now, we can create our Container Registry in Bicep like so:

```bicep
@description('Name of the connected Container Registry')
param containerRegistryName string

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2021-12-01-preview' = {
  name: containerRegistryName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
}
```

Here, I'm giving my ACR instance a name via a parameter, deploying it in the same location as my Log Analytics workspace, setting the SKU to basic and then enabling the Admin User.

Usually, we *shouldn't* do this. Instead we'd create a managed identity for our Container Registry and our Container App, and then assign a [role](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-roles?tabs=azure-cli) to our Container App of 'AcrPull' which will allow our Container App to pull images from our registry.

At the time of writing however, managed identities can't be created for our Container Apps, so as a work around, we'll enable the admin user in our container registry and use the login to authorize our Container App to pull images from the registry.

### Azure Container App Environment

Let's move onto creating an environment for our Container Apps. Individual Container Apps are deployed to a single [Container Apps environment](https://docs.microsoft.com/en-us/azure/container-apps/environment). The environment acts as a secure boundary around groups of container apps.

Container Apps in the same environment are deployed to the same virtual network and they write logs to the same Log Analytics workspace.

For our Bicep template, we'll define our Container App environment and configure it to send logs to our Log Analytics workspace that we've just defined.

```bicep
@description('Name of the Container App Environment')
param containerAppEnvName string

resource containerAppEnvironment 'Microsoft.Web/kubeEnvironments@2021-03-01' = {
  name: containerAppEnvName
  location: location 
  kind: 'containerenvironment'
  properties: {
    environmentType: 'managed'
    internalLoadBalancerEnabled: false
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

Here we're providing a name for our Container Apps environment with a parameter and deploying it in the same location as our other resources. For this Container Apps environment, we are disabling the internal load balancer, because we're not working with a vNET for this environment. We can set this flag to ```true``` to make this environment visible only within the vNET subnet, but that's out of scope for this article.

We set the environment type to ```managed``` as this is the only supported type for Container App Environments. We then configure our environment to send logs to Log Analytics with the ```logAnalyticsConfiguration``` block. Here we use the ```customerId``` and the ```primarySharedKey```, which we can access as outputs of our ```logAnalytics``` resource.

To see what else we can configure when creating Container App Environments in Bicep, check out the template reference [here](https://docs.microsoft.com/en-us/azure/templates/microsoft.web/kubeenvironments?tabs=bicep). 

### Container App

We now have an environment that we can deploy our Container Apps to, so let's write up some Bicep for a single Container App.

```bicep
@description('Name of the TodoApi Container App')
param todoApiContainerName string

resource todoApiContainerApp 'Microsoft.Web/containerApps@2021-03-01' = {
  name: todoApiContainerName
  location: location
  properties: {
    kubeEnvironmentId: containerAppEnvironment.id
    configuration: {
      ingress: {
        external: true
        targetPort: 80
        allowInsecure: false
        traffic: [
          {
            latestRevision: true
            weight: 100
          }
        ]
      }
      registries: [
        {
          server: containerRegistry.properties.loginServer
          username: acrUserName
          passwordSecretRef: 'container-registry-password'
        }
      ]
      secrets: [
        {
          name: 'container-registry-password'
          value: containerRegistry.listCredentials().passwords[0].value
        }
      ]
    }
    template: {
      containers: [
        {
          name: todoApiContainerName
          image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
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

There's quite a bit happening here so let's break it down a bit:

- Starting with the basics, I'm giving my Container App a name that I will set as a parameter and deploying it in the same location as the rest of my resources.
- We then move onto our Container App specific properties. We need to provide the resource ID of the Container App's kubeEnvironment to tell our Bicep code that this Container App belongs to our Container App environment. We do this by accessing the ID as an output of our ```containerAppEnvironment``` resource.
- Within our ```configuration``` block, we're enabling external Ingress and setting our target port for this Container to port 80. We deny insecure traffic and set the traffic level to our latest revision at 100%. If we had [multiple revisions](https://docs.microsoft.com/en-us/azure/container-apps/revisions), we split traffic between our revisions. Since we're only working with one revision, we'll send all traffic there.
- Next up, we define the registry that our container app will pull images from. We do this by providing the server name and username as outputs from our ```containerRegistry``` resource block that we defined earlier. We then provide a **reference** to the name of secret that we will use to store the login password for our Azure Container Registry.
- We then define a ```secret``` block that will create our secret that has a value containing the password needed to access our Azure Container Registry. Again, we use an output from our ```containerRegistry``` to get this password.
- In our ```template``` block, we define the container that we'll deploy to our Container App. In my ```container``` array, I'm using the same name as my container app for my image image, using a simple hello-world image that Microsoft have provided and setting both CPU and Memory resources (You can read how to configure your containers in Container Apps [here](https://docs.microsoft.com/en-us/azure/container-apps/containers#configuration)). Within my ```scale``` block, I'm just setting the minimum and maximum number of replicas to 1, but at the time of writing we could set a [maximum of up to 25 replicas](https://docs.microsoft.com/en-us/azure/container-apps/scale-app).

For more information on what we can define in a Container Apps Bicep template, check out the template guide [here](https://docs.microsoft.com/en-us/azure/templates/microsoft.web/containerapps?tabs=bicep).

## Deploying our Bicep Template

Our full Bicep template should look like this (I've moved things around just to tidy it up a bit, so feel free to copy and paste this if you want, or fix your template to make it look similar):

```bicep
@description('Name of the log analytics workspace')
param logAnalyticsName string

@description('Name of the connected Container Registry')
param containerRegistryName string

@description('Name of the Container App Environment')
param containerAppEnvName string

@description('Name of the TodoApi Container App')
param todoApiContainerName string

param location string = resourceGroup().location

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2021-12-01-preview' = {
  name: logAnalyticsName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
  }
}

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2021-12-01-preview' = {
  name: containerRegistryName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
}

resource containerAppEnvironment 'Microsoft.Web/kubeEnvironments@2021-03-01' = {
  name: containerAppEnvName
  location: location 
  kind: 'containerenvironment'
  properties: {
    environmentType: 'managed'
    internalLoadBalancerEnabled: false
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

resource todoApiContainerApp 'Microsoft.Web/containerApps@2021-03-01' = {
  name: todoApiContainerName
  location: location
  properties: {
    kubeEnvironmentId: containerAppEnvironment.id
    configuration: {
      ingress: {
        external: true
        targetPort: 80
        allowInsecure: false
        traffic: [
          {
            latestRevision: true
            weight: 100
          }
        ]
      }
      registries: [
        {
          server: containerRegistry.name
          username: containerRegistry.properties.loginServer
          passwordSecretRef: 'container-registry-password'
        }
      ]
      secrets: [
        {
          name: 'container-registry-password'
          value: containerRegistry.listCredentials().passwords[0].value
        }
      ]
    }
    template: {
      containers: [
        {
          name: todoApiContainerName
          image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
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

We will also need to provide a parameter file to accompany our Bicep template. Like ARM, this is just a JSON file. We could provide parameters via the AZ CLI, but in future posts I want to deploy my Bicep template via GitHub Actions, so we'll start with a parameter file. 

Our parameter file will look like this (You'll need to provide your own values):

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "logAnalyticsName": {
        "value": "<log-analytics-workspace-name>"
      },
      "containerAppEnvName": {
          "value": "<container-environment-name>"
      },
      "todoApiContainerName": {
          "value": "<container-app-name>"
      },
      "containerRegistryName": {
          "value": "<container-registry-name>"
      }
    }
  }
```

We can now deploy our Bicep tempalte. As you can see within the Bicep template, I've set the ```location``` parameter to be the location of the resource group that we will deploy our Container App resources to.

**At the time of writing, Container Apps can only be provisioned in the following environments:**

- East US
- North Europe
- Canada Central
- East US 2
- West Europe

With that in mind, we'll create a resource group that we'll deploy our resources to by running the following AZ CLI command.

```bash
az group create --name <name-of-your-resource-group> --location <one-of-the-above-locations>
```

We now have a resource group to deploy our Container App resources to. Let's do that by running the following command:

```bash
az deployment group create --resource-group <name-of-your-resource-group> --template-file <your-bicep-file>.bicep --parameters --<your-parameter-file>.json
```

Give that a couple of minutes to run and we should see the following output:

![image](https://willvelidastorage.blob.core.windows.net/blogimages/deploycontainerappsbicep2.png)

Click on **Go to resource group** and we should see that all our resources we defined in our Bicep template have been deployed to Azure!

![image](https://willvelidastorage.blob.core.windows.net/blogimages/deploycontainerappsbicep3.png)

Let's see how our Container App is doing! Click on your Container App resource and you should see an Application Url (since we enabled external ingress in our Container App). Navigate to it and you should see the following web page:

![image](https://willvelidastorage.blob.core.windows.net/blogimages/deploycontainerappsbicep4.png)

Awesome! We've just deployed a Azure Container App in a Container App environment that sends logs to our Log Anaytics workspace and can pull images from our Azure Container Registry in just once Bicep file! Good job! 👏👏

## Wrapping Up

As you can see, we can manage and deploy our Container Apps using Bicep! As this awesome product grows, more features will be released meaning that our we can extend our Bicep template further ([vNet integration comes to mind](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/azure-container-apps-virtual-network-integration/ba-p/3096932))

If you want a reference to the code that we've written in this post, you can do so in this [GitHub repository](https://github.com/willvelida/bookstore-containerapps).

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️

