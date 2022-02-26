---
title: "Implementing Blue/Green Deployments with Azure Web Apps for Containers"
date: 2022-02-21T21:25:40+13:00
draft: false
tags: ["Azure","Bicep","App Service","Containers", "Infrastructure as Code", "GitHub Actions"]
ShowToc: true
TocOpen: true
cover:
    image: https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp7.png
    alt: "Implementing Blue/Green Deployments with Azure Web Apps for Containers"
    caption: 'Using Deployment slots, we can perform Blue/Green deployments in Azure App Service to achieve zero-downtime deployments for our containerized workloads.'
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

We're running our containers on Linux, so we'll need to provision a Linux App Service Plan. I've parameterized the App Plan Sku name, but as I mentioned before, deployment slots are only available at **Standard** or above plan. Since we're provisioning a Linux plan, we'll need to set the ```reserved``` property to true.

To see the full Bicep reference template for App Service Plans, check out this [page](https://docs.microsoft.com/en-us/azure/templates/microsoft.web/2019-08-01/serverfarms?tabs=bicep).

With our App Plan template complete, we can start to create our Bicep template for our App Service with both Blue and Green slots.

Here is our full Bicep file for our App Service. Don't worry! we will break this down:

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

- Let's start with our parameters. We'll need to import both the App Service Plan and the Docker Registry into this Bicep file, so I've added some parameters that we can use in the template. This may be scope creep for this tutorial, but say we have Container Registries across our different environments (DEV, UAT, Prod) and different app service plans, we can provision new App Services that use our different resources across our different environments taking this approach.
- We then create a variable for our *AcrPull* role definition. We do this by getting the unique identifier for a resource using the ```subscriptionResourceId``` [function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#subscriptionresourceid).
- We create another variable for our App Settings. For our example, both our Blue and Green slot will need to have the same app settings. We could use different app settings for our slots if we needed/wanted to, but for simplicity, I've created a variable and applied them to both slots just so I don't have to repeat myself.
- Our Container Registry needs to be imported so I can reference it in our App Service. All we need to do here is to create a resource block for our Container Registry and use the keyword ```existing```. We reference the name of our registry using a parameter.
- For our App Service, we give it a name and location. We also need to provide the following information:
  - First we set the ```kind``` property to ```app,linux,container```. This tells the App Service that we want to host Linux Containers.
  - In the properties section, we set the ```serverFarmId``` to the Id of our Server Farm. We're setting this as a parameter, so if we wanted to use this module for other App Services, we could deploy it to another App Service Plan.
  - For our site config section, we set the ```appSettings``` to the App Settings we defined earlier, the ```acrUseManagedIdentityCreds``` property tells the App Service that we want to use the managed identity credentials to pull images from our container registry. We finally set the ```linuxFxVersion``` to ```DOCKER|${containerRegistry.properties.loginServer}/${dockerImageAndTag}```. This sets the Linux App Framework and version.
  - We finally create a System Managed Identity for this App Service and the Blue slot. We'll need this for our role assignments.

We then create two role assignments that allow our App Service Blue and Green slot to pull images from our container registry. The ```roleDefinitionId``` uses the *AcrPull* role definition we defined in our variable at the start, and the ```scope``` property assigns this role to the App Service over our Azure Container Registry. 

To learn more about how we can create role assignments using Bicep, check out this [article](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/scenarios-rbac).

With our modules created, we can now write up our ```main.bicep``` file like so:

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

We'll also need a parameters file. We can create a ```parameters.json``` file like so:

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

Now that our Bicep code has been written, we can deploy it using GitHub Actions. Before we can do this, we'll need to set up a couple of things.

First let's set up a resource group that we'll deploy our resources to. We can do this by running the following AZ CLI command:

```bash
az group create -n <resource-group-name> -l <location>
```

With our resource group created, we'll need to generate a Service Principal that will allow our GitHub Action workflow to deploy resources to Azure. We can do so by running the following:

```bash
az ad sp create-for-rbac --name yourApp --role owner --scopes /subscriptions/{subscription-id}/resourceGroups/exampleRG --sdk-auth
```

Use your resource group name and subscription Id in the command. The output of this command will generate a JSON object that you'll need to copy. It should look similar to this:

```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
}
```

We need to save this JSON ouput as a **Secret** in our GitHub repo. You can do this by selecting **Settings** then **Secrets** and clicking on **New** to create the secret. We'll need to create the following secrets:

| **Secret** | **Value** |
| ---------- | --------- |
| RESOURCE_GROUP | Name of the resource group that we're deploying resources to |
| AZURE_CREDENTIALS | The entire JSON output that was generated as part of the service principal creation step |

Now that we have our secrets, we can create the following workflow file that will deploy our resources:

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

This file does the following:

- We first validate our file to ensure that it's a valid Bicep template.
- We then run a preview job on our Bicep template to see what will be deployed as part of our workflow.
- We then run a deploy stage that uses a manual approval step. This is done by putting the ```environment``` parameter in this stage. When we approve this deployment, the stage deploys our Bicep template using the parameters file to deploy our resources to Azure.

![Our Bicep GitHub Actions Deployment Pipeline](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp1.png)

If we head into Azure, we can see that our resources have been created:

![Our required resources](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp2.png)

## Deploying our container to App Service

We have our Container Registry and App Service ready to go in Azure! We just need to allow our GitHub Action to push and push images to our Azure Container Registry.

First, we need to get the resource Id of our container registry. We can do this by running the following command (Replace ```<registry-name>``` with the name of your Azure Container Registry):

```bash
registryId=$(az acr show --name <registry-name> --query id --output tsv)
```
Now that we have our resource Id, we can use the following AZ CLI command to assign the AcrPush role (Replace ```<ClientId>``` with the client ID of your service principal):

```bash
az role assignment create --assignee <ClientId> --scope $registryId --role AcrPush
```

Once our role has been created, we can set up the following secrets in GitHub Actions that we'll need to use in our GitHub Action workflow file:

| **Secret** | **Value** |
| ---------- | --------- |
| REGISTRY_LOGIN_SERVER | The login server name of your registry (all lowercase). Example: myregistry.azurecr.io |
| REGISTRY_USERNAME | The ```clientId``` from the JSON output from the service principal creation |
| REGISTRY_PASSWORD | The clientSecret from the JSON output from the service principal creation |

### Pushing our container image to Azure Container Registry

We can now create our workflow file. This file will do quite a bit so let's focus on pushing our Container Image to Azure Container Registry to begin with. 

```yaml
name: Build and Deploy Container Image to App Service

on:
  workflow_dispatch:

defaults:
  run:
    working-directory: ./src

# Note: The use of :latest for the container image is not recommeded for production environments.
jobs:
  build-container-image:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main
      
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: 'Build and Push Image to ACR'
      uses: azure/docker-login@v1
      with:
        login-server: ${{ secrets.REGISTRY_LOGIN_SERVER }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    - run: |
        docker build . -t ${{ secrets.REGISTRY_LOGIN_SERVER }}/hellobluegreenwebapp:latest
        docker push ${{ secrets.REGISTRY_LOGIN_SERVER }}/hellobluegreenwebapp:latest
```

We login to Azure using our Service Principal connection and then log into our Azure Container Registry. Once that's successfull, we build and push our container image to our ACR.

You can view the Dockerfile that I'm using [here](https://github.com/willvelida/azure-apps-for-containers-bluegreen-poc/blob/main/src/Dockerfile).

One thing to note here is the image tag that we're using. This is a basic demo, so I'm just using ```latest```. For your container images, make sure you follow [best practices](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version).


Once this has been built and pushed to our Azure Container Registry, we should be able to see our container image by navigating to our ACR, and searching for the image under **Repositories**

![Our deployed image to ACR](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp3.png)

### Deploying our App to the Blue slot

Once our image has been pushed to Azure Container Registry, we can deploy it to our Blue slot. In the snippet below, we log into Azure, then retrieve our Application name using a inline script and set it to a output variable that we can use in a later step in this stage.

We then deploy our image to our Blue slot using the output variable we set as the App name, with the image that we've pushed to our ACR and explicity tell the task to deploy the image to our ```blue``` slot:

```yaml
deploy-to-blue-slot:
    needs: build-container-image
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main
      
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: 'Get App Name'
      id: getwebappname
      run: |
        a=$(az webapp list -g ${{ secrets.AZURE_RG }} --query '[].{Name:name}' -o tsv)
        echo "::set-output name=appName::$a"

    - name: 'Deploy to Blue Slot'
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ steps.getwebappname.outputs.appName }}
        images: ${{ secrets.REGISTRY_LOGIN_SERVER }}/hellobluegreenwebapp:latest
        slot-name: 'blue'
```

### Verify that the blue slot works

n this sample, we can verify whether or not our deployment to the blue slot was successful by simply navigating to the blue slot of our App Service.

The URL for our blue slot will take the following format:

```https://<name-of-app-service>-<slot-name>.azurewebsites.net```

In this sample, we're just deploying to our blue slot and manually verifying whether or not our container image deployed successfully. In production scenarios, we'll need to include tasks, such as automation tests, to ensure that our deployed container image works as expected before we deploy to our production slot.

In our GitHub actions, we should see the URL to our blue slot like so:

![GitHub Action showing Blue Slot url](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp4.png)

We can use this URL to navigate to the blue slot and see that our image has been successfully deployed:

![Blue slot screenshot](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp5.png)

If we navigate to our green slot, we see that nothing has been deployed yet! Let's swap the slots now.

### Swap to Green slot

Once we have verified that our deployment to the blue slot has been successful, we can deploy our container image to the green slot. We can do this using the following job in our GitHub Actions workflow:

```yaml
swap-to-green-slot:
    runs-on: ubuntu-latest
    environment: Dev
    needs: deploy-to-blue-slot
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main
      
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: 'Get App Name'
      id: getwebappname
      run: |
        a=$(az webapp list -g ${{ secrets.AZURE_RG }} --query '[].{Name:name}' -o tsv)
        echo "::set-output name=appName::$a"

    - name: 'Swap to green slot'
      uses: Azure/cli@v1
      with:
        inlineScript: |
          az webapp deployment slot swap --slot 'blue' --resource-group ${{ secrets.AZURE_RG }} --name ${{ steps.getwebappname.outputs.appName }}
```

In this job, we use environments to manually approve the deployment to our production slot provided that our deployment to the blue slot was successful. We can assign a user or users that need to approve the deployment before this job runs.

In this sample, we just use this as a manual approval step. In production scenarios, it's a good idea to run automation tests to ensure that your new container image works as expected before deploying to your production slots.

To learn more about how environments work in GitHub Actions, check out the following [documentation](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment).

Finally, we initiate the swap to the production slot by using the AZ CLI. Here are just swapping from our blue slot into our green slot. To learn more about how we can work with slots using the AZ CLI, please review the following [documentation](https://docs.microsoft.com/en-us/cli/azure/webapp/deployment/slot?view=azure-cli-latest#commands).

As with out green slot, we should get a URL. Since this is our production slot, our URL won't have a slot name within it's URL.

Navigate to the URL and you can see that we've successful swapped to our green slot!

![Our shiny new production app](https://willvelidastorage.blob.core.windows.net/blogimages/bluegreenapp6.png)

## Conclusion

If you're deploying containers to Azure App Service, hopefully this has helped you see that you can use Blue/Green deployments to achieve zero-downtime deployments for your container images.

This was a basic example, so in your production scenarios you'll want to run comprehensive automation tests against your blue slot before swapping to the green slot. 

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️