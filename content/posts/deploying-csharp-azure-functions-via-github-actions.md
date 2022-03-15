---
title: "Deploying C# Azure Functions via GitHub Actions"
date: 2022-03-15
draft: false
tags: ["Azure","Azure Functions","Serverless","GitHub Actions", "C#"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s6dx3876bondcx8rpikj.png
    alt: "Azure Functions GitHub Actions workflow output"
    caption: 'We can deploy Azure Functions using GitHub Actions by defining a simple workflow file in YAML'
---

I've spent a lot of time with GitHub Actions lately and it's been a lot of fun. I've had quite a bit of experience using Azure DevOps in my previous jobs and before GitHub Actions were a thing, I'd create Service Connections in Azure DevOps so that I could host my code in GitHub, but still run my build and deploy pipelines in Azure DevOps.

This isn't to say that GitHub Actions is better than Azure DevOps, nor vice-versa. This article is purely an informational piece on *HOW* you can use GitHub Actions to deploy your Functions to Azure. Specifically we'll talk about:

- Setting up our GitHub Action
- Building our Function App
- Deploying our Function to Azure

## Setting up our GitHub Action

Before we can create our GitHub Actions workflow, we'll need to create a service principal so that our GitHub Action can authenticate and perform operations in our Azure environment. To do this we can use the AZ CLI. Run the following commands to create our service principal:

```bash
# If you aren't logged in, you can login in like so
az login

# if you don't have a resource group to deploy to, you can create one
az group create -n <resourceGroupName> -l <location>

# create your service principal
az ad sp create-for-rbac --name <nameOfYourApp> --role owner --scopes /subscriptions/<subscriptionId>/resourceGroups/<resourceGroupName> --sdk-auth
```

Replace the ```--name``` parameter with the name of your application. In the ```az ad sp create-for-rbac``` command, the scope of the service principal is limited to our resource group. You'll need to replace the ```subscriptionId``` and ```resourceGroup``` parameters with the Id of your Azure Subscription and the name of your resource group that you'll be deploying to respectively.

The ```az ad sp create-for-rbac``` command will generate a JSON output that looks like this:

```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
    // output omitted for brevity
}
```

We need to save this JSON output as a **Secret** in our GitHub repo. You can do this by selecting **Settings** then **Secrets** and clicking on **New** to create the secret. We'll need to create the following secrets:

| **Secret** | **Value** |
| ---------- | --------- |
| AZURE_CREDENTIALS | The entire JSON output that was generated as part of the service principal creation step |

Now we can move onto creating our GitHub Action file in our git repository. This is a ```.yml``` file (Yay YAML 😝) that contains our instructions to the build agent on how we want to build our Function app and deploy it to Azure.

To create this file, we can head to our repository in GitHub and click on **Actions** then **New workflow**. From here, we can choose a template workflow that other developers have built or we can set up our own workflow. For the purposes of this tutorial, we're going to create our own one from scratch.

![Creating a workflow in GitHub Actions](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w0od91dp64e4u9ssnbsh.png)

We'll be presented with a web editor in which we can start to write our GitHub Actions workflow file. Before we discuss our ```build``` and ```deploy``` jobs, let's enter the following in our workflow file:

```yaml
name: Build and Deploy Function App
env:
  AZURE_FUNCTIONAPP_PACKAGE_PATH: '.'
  DOTNET_VERSION: 6.0.x
  OUTPUT_PATH: ${{ github.workspace }}/.output
  
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

Let's break this down:

- ```name``` is the name of our workflow. GitHub will display the names of our workflow in our repository actions page. This way, if we have multiple workflows in one repository, we can use the name of our workflow to distinguish them.
- We can define environment variables for our workflow file under ```env```. Here we write key/value pairs that acts as environment variables for our workflow file. In this case, I'm defining .NET 6 as the .NET version I want to build my Function in, the root directory as my package path and I'm defining an output path which I will use to publish my Function package to. More on that later.
- To automatically trigger a workflow, we can use the ```on``` key here to define what will cause our workflow to run. GitHub Actions provides us a bunch of different method to trigger a workflow file, but in our case whenever we push to our ```main``` branch, this workflow file will trigger. I've also added ```workflow_dispatch``` which will allow us to manually trigger our workflow in GitHub.

## Creating our Build Stage

Let's define our build stage. To do this, we create a **job** in our workflow file. A workflow run is made up of one or more jobs (in our case, two jobs). By default, we can run these in parallel, but we'll first build our Function to build a package file and then create a second job to deploy that package.

We can write our build job as follows:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@v2
      
    - name: Setup DotNet ${{ env.DOTNET_VERSION }} Environment
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Publish Functions
      run: dotnet publish <your-function>.csproj --configuration Release --output ${{ env.OUTPUT_PATH }}
      
    - name: Package Functions
      uses: actions/upload-artifact@v1
      with:
        name: functions
        path: ${{ env.OUTPUT_PATH }}
```

Let's break this down:

- Each job runs in a runner environment which we define using ```runs-on```. In our workflow, we'll be using a Ubuntu environment.
- We then define the steps that our job will take under ```steps```. We can run commands, perform setup or run actions in our repository. Each step runs it its own process in the environment and has access to the workspace and filesystem.
- In our first step, we check out our repository using the ```actions/checkout@v2``` action so that our workflow can access our repository.
- We then use the ```actions/setup-dotnet@v1``` action to setup a .NET CLI environment for our runner to use. In this step, we use our ```DOTNET_VERSION``` environment variable to tell this action to use .NET 6 as our .NET version.
- We then run a CLI command to publish a Release artifact to our ```OUTPUT_PATH``` environment variable for our Function. Here, we're just running a inline .NET CLI command to publish our artifact.
- Finally, we use the ```actions/upload-artifact``` action to upload our published Function package as an artifact that we can use in our Deploy job.

To read more about the tasks that I'm using here, checkout the following docs:

- [Checkout](https://github.com/actions/checkout)
- [Setup Dotnet](https://github.com/actions/setup-dotnet)
- [Upload Artifact](actions/upload-artifact@v1)

This is a very basic build job that we've defined here. In production scenarios, you'll want to use this stage to run your unit tests for your Functions, perform static code analysis for security checks and perhaps compliance checks to ensure that your code meet the compliance requirements that your company has to adhere to. I've kept this workflow simple, but do make sure you include these tasks in to improve your confidence that your code is reliable and secure. 

Those types of checks are the equivalent of eating your vegetables, but it's good for you, so do it.

## Deploying our Function App

We now have our build job written up, so we can start to create our deploy job. For this, we can write the following YAML:

```yaml
deploy:
    runs-on: ubuntu-latest
    needs: [build]
    env:
      FUNC_APP_NAME: <your-function-app-name>
      
    steps:
      - name: Download Artifact
        uses: actions/download-artifact@v1
        with:
          name: functions
          path: ${{ env.OUTPUT_PATH }}
        
      - name: "Login via Azure CLI"
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
        
      - name: Deploy Function to Azure
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ env.FUNC_APP_NAME }}
          package: ${{ env.OUTPUT_PATH }}
```

Again, let's break this down:

- Similar to our ```build``` job, we define our runner environment, but this time we add ```needs``` to our setup. This states that we need the ```build``` job to complete successfully before our ```deploy``` job will run.
- We also define an environment variable for the name of our Function App within the context of this specific job. If we had more than one environment variable defined with the same name, GitHub will use the most specific environment variable. So for example, if we had an environment variable defined for our Function App at the workflow level, our environment variable defined in our Job would take priority.
- In our first step, we download our Functions artifact from our ```OUTPUT_PATH``` that we published to in our build stage.
- We then login to Azure using our service principal credentials that we generated for our GitHub Action workflow earlier using the ```azure/login@v1```
- Finally, we deploy our Azure Function using the ```Azure/function-actions``` Action. Here, we provide the name of the app that we want to deploy to (using our environment variable that we defined in our job), along with the package that we will deploy (that we saved our Publish artifact to in our ```build``` job).


To read more about the tasks that I'm using here, checkout the following docs:

- [Download Artifact](https://github.com/actions/download-artifact)
- [Azure Login](https://github.com/Azure/login)
- [GitHub Actions for deploying to Azure Functions](https://github.com/Azure/functions-action)

## Wrapping up

In this article, we talked about how we can deploy our C# Azure Functions using a simple GitHub Action workflow. 

This was a very basic example, so in our production apps, we'll want to run unit tests, security scans, integration tests, automation tests etc to ensure that our Functions are well tested before our users use them in our applications.

Hopefully this article will help you when you're using GitHub Actions to deploy Functions. If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️