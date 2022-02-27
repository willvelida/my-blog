---
title: "Building and Deploying Container Images to Azure Container Apps with GitHub Actions"
date: 2022-02-27T19:24:11+13:00
draft: true
tags: ["Azure","Serverless","Azure Container Apps","Containers", "Docker", "GitHub Actions"]
ShowToc: true
TocOpen: true
---

In a previous [blog post](https://www.willvelida.com/posts/deploy-container-apps-bicep/), I talked about how we can provision an Azure Container App using Bicep and deploying our Bicep template using GitHub Actions.

We'll now turn our attention to updating the images that our Container App uses by building the new image, deploying it to Azure Container registry and then pulling the newly built image from our registry to our Container App.

Before we start, please bear in mind that Azure Container Apps are **currently in preview!**. That means that this post was correct as of the time of writing! As the service reaches GA (please don't ask me when, I don't know), the methods you read here may have changed drastically. (*Note to self, when that happens update this article!*)

With that in mind, this is what we'll cover:

- Ensuring that our GitHub action has sufficient permissions to run the workflow
- Building our container image and pushing it to Azure Container Registry
- Deploying our container image to Azure Container Apps.

If you haven't got a Azure Container Apps environment setup, please check out my article on [Creating and Provisioning Azure Container Apps with Bicep](https://www.willvelida.com/posts/deploy-container-apps-bicep/).

## Ensuring that our GitHub action has sufficient permissions to run the workflow

When we use GitHub Actions to deploy resources to Azure, we need to create a service principal that is authorized to do so. We can create a service principal for our GitHub Actions workflow by running the following AZ CLI command:

```bash
az ad sp create-for-rbac \
  --name <SERVICE_PRINCIPAL_NAME> \
  --role "contributor" \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME> \
  --sdk-auth
```

Replace the following parameters with your own values:

| **Parameter** | **Description** |
| --------- | ----------- |
| SERVICE_PRINCIPAL_NAME | The name of your service principal. Can be anything your want |
| SUBSCRIPTION_ID | Your Azure Subscription Id |
| RESOURCE_GROUP_NAME | The name of your resource group that you'll be deploying to |

This command assigns the *contributor* permission to your GitHub Action workflow over the resource group. This command should generate a JSON output that you'll need to copy. It looks similar to this:

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
| REGISTRY_LOGIN_SERVER | The login server name of your registry (all lowercase). Example: myregistry.azurecr.io |
| REGISTRY_USERNAME | The ```clientId``` from the JSON output from the service principal creation |
| REGISTRY_PASSWORD | The ```clientSecret``` from the JSON output from the service principal creation |

## Building our container image and pushing it to Azure Container Registry

With our secrets created, we can start working on building and pushing our container image to our Azure Container Registry. For this sample, I'm using the following Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app 
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["BookStore.Common/BookStore.Common.csproj", "BookStore.Api/"]
COPY ["BookStore.Repository/BookStore.Repository.csproj", "BookStore.Api/"]
COPY ["BookStore.Services/BookStore.Services.csproj", "BookStore.Api/"]
COPY ["BookStore.Api/BookStore.Api.csproj", "BookStore.Api/"]
RUN dotnet restore "BookStore.Api/BookStore.Api.csproj"
COPY . .
WORKDIR "/src/BookStore.Api"
RUN dotnet build "BookStore.Api.csproj" -c Release -o /app

FROM build AS publish
RUN dotnet publish "BookStore.Api.csproj" -c Release -o /app

FROM base AS final
WORKDIR /app
COPY --from=publish /app .
ENTRYPOINT [ "dotnet", "BookStore.Api.dll" ]
```

In my GitHub Actions workflow, I want to run a job that will build and push this container image into my Azure Container Registry. We can do this using the following YAML:

```yaml
name: Build and Deploy Container Image to Azure Container Apps

on:
  push:
    paths:
      - 'src/*'
  workflow_dispatch:

defaults:
  run:
    working-directory: ./src/BookStore

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
        docker build . -t ${{ secrets.REGISTRY_LOGIN_SERVER }}/bookstoreapi:${{ github.sha }}
        docker push ${{ secrets.REGISTRY_LOGIN_SERVER }}/bookstoreapi:${{ github.sha }}
```

## Deploying our container image to Azure Container Apps.

```yaml
deploy-container-app:
    runs-on: ubuntu-latest
    needs: build-container-image
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main
      
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: 'Deploy Container App'
      uses: Azure/cli@v1
      with:
        inlineScript: |
          echo "Installing containerapp extension"
          az extension add --source https://workerappscliextension.blob.core.windows.net/azure-cli-extension/containerapp-0.2.0-py2.py3-none-any.whl --yes
          echo "Starting Deploying"
          az containerapp update -n velidabookstoreapi -g todocontainerapp-rg -i ${{ secrets.REGISTRY_LOGIN_SERVER }}/bookstoreapi:${{ github.sha }} --registry-login-server ${{ secrets.REGISTRY_LOGIN_SERVER }} --registry-username  ${{ secrets.REGISTRY_USERNAME }} --registry-password ${{ secrets.REGISTRY_PASSWORD }} --debug
```

## Alternative approach

As part of the preview, we can work with Azure Container Apps using the Azure CLI. One of the commands at our disposal allows us to generate a GitHub Actions workflow using the following command:

```bash
az containerapp github-action add \
  --repo-url "https://github.com/<OWNER>/<REPOSITORY_NAME>" \
  --docker-file-path "./dockerfile" \
  --branch <BRANCH_NAME> \
  --name <CONTAINER_APP_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --registry-url <URL_TO_CONTAINER_REGISTRY> \
  --registry-username <REGISTRY_USER_NAME> \
  --registry-password <REGISTRY_PASSWORD> \
  --service-principal-client-id <CLIENT_ID> \
  --service-principal-client-secret <CLIENT_SECRET> \
  --service-principal-tenant-id <TENANT_ID> \
  --token <YOUR_GITHUB_PERSONAL_ACCESS_TOKEN>
```

With this command, we 

## Playing around with the AZ CLI

## Conclusion

If you want a reference to the code that we've written in this post, you can do so in this [GitHub repository](https://github.com/willvelida/bookstore-containerapps).

In a future blog post, I'll talk about the different types of changes we can make in Azure Container Apps and how that affects our Container Apps environments.

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️
