---
title: "Implementing Blue/Green Deployments with Azure Web Apps for Containers"
date: 2022-02-21T21:25:40+13:00
draft: true
tags: ["Azure","Bicep","App Service","Containers", "Infrastructure as Code", "GitHub Actions"]
ShowToc: true
TocOpen: true
---

Application uptime is critical for our cloud applications. Using Azure App Service slots, we can implement the Blue/Green deployment pattern to validate that new versions of our application will perform as expected in a production environment, without causing downtime to our existing version of our application. With App Service slots, we can deploy new versions of our container images to our Green slot, run tests against that slot to ensure that everything is working and then direct incoming traffic to our updated container image.

In this article, I'll explain what Blue/Green Deployments are and show you how can implement Blue/Green deployments for your App Services that host your containers. This includes:

- Creating our Azure resources with Bicep.
- Deploying our resources with GitHub Actions.
- Pushing our container image to Azure Container Registry.
- Pulling our container image into our App Service Blue Slot.
- Swapping from the Blue slot into our Green Slot.

Throughout this article, I'll be referring to code that you can find in this GitHub [repo](https://github.com/willvelida/azure-apps-for-containers-bluegreen-poc). If you want to follow along, or use that repo as a base for experimenting with Blue/Green deployments with App Service, please feel free to fork it and have a play around with it! If you see something wrong or want to make an improvement to it, PRs are welcome! 😊

## What are Blue/Green Deployments?

As I mentioned, ensuring that our applications are highly available is critical for applications running in the cloud. There are a couple of strategies that we can implement to ensure HA for our apps, such as spinning up our application in a different region in case of a regional-disaster affecting the datacenter where our app is hosted or using high availability configurations. With both of these options, there are time and cost implications that we would need to consider.

We can ensure high availability when deploying our application. With Blue/Green deployments, we can deploy a new version of our application next to the existing one.

Imagine you're working on an application and you build a container image that packages the new version of your app. With Blue/Green deployments, we can deploy the new image to the Blue slot alongside our Green slot. Production traffic is still being routed to Green slot, but now that our new version has been deployed to the Blue slot, we can run tests against the Blue slot in a production environment to ensure that the new image works as expected. If everything works, we can swap the image from the blue slot to the green slot, redirecting our production traffic to our new image, which happens with no visible downtime for our users. 

Should anything not work as expected, we can abandon the deployment of our new container image without affecting our users, since traffic will be redirected to our green slot containing our exiting image.

In Azure App Service, we can set up staging environments using deployment slots. By using Blue/Green deployments, we can warm up instances in a slot before we swap slots to production traffic, elminating downtime for our application. 

Please note, that your App Service Plan must be either at the **Standard**, **Premium** or **Isolated** tier. 

## Preparing our infrastrucutre

Let's start off by setting up our infrastructure pipeline. Here, I want to create my App Service that has both a blue and green slot, an Azure Container Registry to store my images, then I want to be able to deploy my Bicep template using GitHub Actions.

### Creating our Azure Resources

First, let's create the Azure Container Registry that we need to store our container images. We can do this in Bicep like so:

Azure Container Registry Bicep code:

```bicep
param registryName string
param registryLocation string
param registrySku string

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2021-09-01' = {
  name: registryName
  location: registryLocation
  sku: {
    name: registrySku
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

Here, we're giving our container registry a name, location and sku which we will pass through as parameters.

The important thing to note here is that we're creating a [System Assigned Managed Identity for our Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity). So rather than enabling admin access on this Container Registry, we'll be using a managed identity for our ACR which will allow us to assign permissions and roles for the ACR without having to supply registry credentials. 

There are a number of ways that we can authenitcate with Azure Container Registry, so I'd encourage you to check [this guide](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli) out to see what options are best for your scenario.

Next up. we'll create our App Service Plan. To do this, we can write the following Bicep code:

```bicep
param appServicePlanName string
param appServicePlanLocation string
param appServicePlanSkuName string
param appServicePlanCapacity int

resource appServicePlan 'Microsoft.Web/serverfarms@2021-02-01' = {
  name: appServicePlanName
  location: appServicePlanLocation
  sku: {
    name: appServicePlanSkuName
    capacity: appServicePlanCapacity
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}



output appServicePlanId string = appServicePlan.id
```

App Service Bicep code:

```bicep
@description('Name of the app service plan')
param appServiceName string

@description('Location of the app service plan')
param appServiceLocation string

@description('Name of the slot that we want to create in our App Service')
param appServiceSlotName string

@description('The Server Farm Id for our App Plan')
param serverFarmId string

@description('Name of the Azure Container Registry that this App will pull images from')
param acrName string

@description('The docker image and tag')
param dockerImageAndTag string = '/hellobluegreenwebapp:latest'

// This is the ACR Pull Role Definition Id: https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpull
var acrPullRoleDefinitionId = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')

var appSettings = [
  {
    name: 'WEBSITES_ENABLE_APP_SERVICE_STORAGE'
    value: 'false'
  }
  {
    name: 'WEBSITES_PORT'
    value: '80'
  }
  {
    name: 'DOCKER_REGISTRY_SERVER_URL'
    value: 'https://${containerRegistry.properties.loginServer}'
  }
]

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2021-09-01' existing = {
  name: acrName
}

resource appService 'Microsoft.Web/sites@2021-02-01' = {
  name: appServiceName
  location: appServiceLocation
  kind: 'app,linux,container'
  properties: {
    serverFarmId: serverFarmId
    siteConfig: {
      appSettings: appSettings
      acrUseManagedIdentityCreds: true
      linuxFxVersion: 'DOCKER|${containerRegistry.properties.loginServer}/${dockerImageAndTag}'
    }
  }
  identity: {
    type: 'SystemAssigned'
  }

  resource blueSlot 'slots' = {
    name: appServiceSlotName
    location: appServiceLocation
    kind: 'app,linux,container'
    properties: {
      serverFarmId: serverFarmId
      siteConfig: {
        acrUseManagedIdentityCreds: true
        appSettings: appSettings
      }
    }
    identity: {
      type: 'SystemAssigned'
    }
  }
}

resource appServiceAcrPullRoleAssignment 'Microsoft.Authorization/roleAssignments@2020-08-01-preview' = {
  scope: containerRegistry
  name: guid(containerRegistry.id, appService.id, acrPullRoleDefinitionId)
  properties: {
    principalId: appService.identity.principalId
    roleDefinitionId: acrPullRoleDefinitionId
    principalType: 'ServicePrincipal'
  }
}

resource appServiceSlotAcrPullRoleAssignment 'Microsoft.Authorization/roleAssignments@2020-08-01-preview' = {
  scope: containerRegistry
  name: guid(containerRegistry.id, appService::blueSlot.id, acrPullRoleDefinitionId)
  properties: {
    principalId: appService::blueSlot.identity.principalId
    roleDefinitionId: acrPullRoleDefinitionId
    principalType: 'ServicePrincipal'
  }
}
```

Our main.bicep code:

```bicep
param webAppName string = uniqueString(resourceGroup().id)
param acrName string = toLower('acr${webAppName}')
param acrSku string
param appServicePlanName string = toLower('asp-${webAppName}')
param appServiceName string = toLower('asp-${webAppName}')
param appServicePlanSkuName string
param appServicePlanInstanceCount int

var appServiceSlotName = 'blue'

param location string = resourceGroup().location

module containerRegistry 'containerRegistry.bicep' = {
  name: 'containerRegistry'
  params: {
    registryLocation: location 
    registryName: acrName
    registrySku: acrSku
  }
}

module appServicePlan 'appServicePlan.bicep' = {
  name: 'appServicePlan'
  params: {
    appServicePlanLocation: location
    appServicePlanName: appServicePlanName
    appServicePlanSkuName: appServicePlanSkuName
    appServicePlanCapacity: appServicePlanInstanceCount
  }
}

module appService 'appService.bicep' = {
  name: 'appService'
  params: {
    appServiceLocation: location 
    appServiceName: appServiceName
    serverFarmId: appServicePlan.outputs.appServicePlanId
    appServiceSlotName: appServiceSlotName
    acrName: acrName
  }
}
```

paramters.json file

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "acrSku":{
            "value": "Basic"   
        },
        "appServicePlanSkuName": {
            "value": "S1"
        },
        "appServicePlanInstanceCount": {
            "value": 1
        }
    }
}
```

### Deploying our resources with GitHub Actions

Now that our Bicep code has been written, we can deploy it using GitHub Actions.

Here is our workflow file:

```yaml
name: Deploy Azure Infrastructure

on:
  push:
    paths:
      - 'deploy/*'
  workflow_dispatch:

jobs:

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run Bicep Linter
        run: az bicep build --file ./deploy/main.bicep

  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: azure/login@v1
        name: Sign in to Azure
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - uses: azure/arm-deploy@v1
        name: Run preflight validation
        with:
          deploymentName: ${{ github.run_number }}
          resourceGroupName: ${{ secrets.AZURE_RG }}
          template: ./deploy/main.bicep
          parameters: ./deploy/parameters.json
          deploymentMode: Validate

  preview:
    runs-on: ubuntu-latest
    needs: [lint, validate]
    steps:
      - uses: actions/checkout@v2
      - uses: azure/login@v1
        name: Sign in to Azure
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - uses: Azure/cli@v1
        name: Run what-if
        with:
          inlineScript: |
            az deployment group what-if --resource-group ${{ secrets.AZURE_RG }} --template-file ./deploy/main.bicep --parameters ./deploy/parameters.json
  deploy:
    runs-on: ubuntu-latest
    environment: Dev
    needs: preview
    steps:
      - uses: actions/checkout@v2

      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
        
      - name: Deploy Bicep File
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION }}
          resourceGroupName: ${{ secrets.AZURE_RG }}
          template: ./deploy/main.bicep
          parameters: ./deploy/parameters.json
          failOnStdErr: false
```

## Deploying our container to App Service

### Pushing our container image to Azure Container Registry

### Deploying our App to the Blue slot

### Verify that the blue slot works

### Swap to Green slot

## Conclusion
