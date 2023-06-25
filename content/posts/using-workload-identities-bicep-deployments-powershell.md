---
title: "Using Workload Identities for Bicep Deployments in GitHub Actions"
date: 2023-06-24
draft: false
tags: ["Bicep","GitHub Actions","DevOps","Azure", "Azure AD"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/31rqp6mtxrbb6v4gibi0.png
    alt: "Using Workload Identities for Bicep Deployments in GitHub Actions"
    caption: "With workload identities, we can authenticate GitHub Action workflows to deploy resources to Azure, without needing to manage secrets"
---

As I've been working on my side project, I've been trying to work on my CI/CD skills and deploy all my resources through GitHub Actions. This project is made up of a couple of services, which each have their own infrastructure and application code. I'm deploying my resources to a single resource group in Azure.

To deploy infrastructure to Azure via GitHub Actions, we need to authenticate to our Azure subscription. Via the command line, we can do this using PowerShell or the CLI, but that's possible due to us being able to interact with the authentication process.

In GitHub Actions, we need an automated process to handle the authentication process for us. This is where **Workload Identities** come into play.

In this article, I'll talk about what workload identities are and how they work. We'll then set one up, integrate it with a GitHub Actions workflow file and discuss how we can use workload identities in our GitHub Actions workflows.

This article will use PowerShell code, but you can also do this using the Azure CLI.

## What are workload identities?

When we deploy Bicep templates via GitHub Actions, we need to use workload identities to authenticate to Azure since we will be deploying resources to Azure without direct involvement from us.

Workloads are automated process that doesn't have a human directly running it. This workload can sign into Azure AD, without a human signing in and interacting with the authentication process. 

## How do workload identities work?

Workload identities are a feature of Azure AD. In Azure AD, there will be applications which can represent systems, non-human agents, processes etc. In this scenario, our deployment workflow can also be classified as an application.

When we create and tell Azure AD about an application, we register it and create an object called an application registration. This registration will represent the application in Azure AD.

This application registration can have *federated credentials* associated with it. Federated credentials don't require us to store secrets, which is great for when we want to use Azure AD applications with services like GitHub!

When we authenticate in our GitHub Actions workflow, it will contact Azure AD through GitHub, which tells Azure AD the name of our GitHub organization (our username on GitHub) and repository, as well as some other information. If our federated credential matches the repository's details, our deployment workflow will authenticate using the permissions that we've assigned to our application.

## Setting up our workload identities

Let's set up a workload identity that we will use to deploy resources to our Azure environment. First we'll need to set two variables for our GitHub username and the repository name that our code is stored in. We can set this variables like so:

```powershell
$githubOrganizationName = '<your-github-username>'
$githubRepositoryName = 'myreponame'
```

We now need to create an application registration for our workload identity. We'll need to store it in a variable, which we can do like so:

```powershell
$productionApplicationRegistration = New-AzADApplication -DisplayName 'myreponame-production'
```

Through the variable, we can now access important pieces of information from our application registration, such as:

* **Application ID**: The unique identifier of our app registration. We use this when our workflow needs to sign into Azure.
* **Object ID**: The unique identifier that Azure AD assigns.

Now that our application registration has been created, we need to assign it with *Federated Credentials*. This is an application credential that don't require us to manage any secrets like passwords or keys (which can expire after a certain amount of time). These federated credentials allow Azure AD and GitHub to trust each other (which is referred to as federation).

When our GitHub workflow attempts to sign in, GitHub will provide information about the workflow so Azure AD can determine whether or not this workflow has sufficient permissions to sign in. Information that GitHub provides to Azure AD can include:

* The GitHub user.
* The name of the repository.
* The branch of your repository that the workflow is currently running on.
* The environment that the workflow is targeting.

Azure AD can then either allow or deny the sign-in attempt depending on the information that GitHub has provided. 

In my GitHub Action workflow, I'll be deploying off my **main** branch to my **production** environment, so I'll need to create two federated credentials for my workload identity that associates my production environment with my GitHub repository.

We can do this like so:

```powershell
New-AzADAppFederatedCredential `
   -Name 'myreponame-production' `
   -ApplicationObjectId $productionApplicationRegistration.Id `
   -Issuer 'https://token.actions.githubusercontent.com' `
   -Audience 'api://AzureADTokenExchange' `
   -Subject "repo:$($githubOrganizationName)/$($githubRepositoryName):environment:Production"

New-AzADAppFederatedCredential `
   -Name 'myreponame-production-branch' `
   -ApplicationObjectId $productionApplicationRegistration.Id `
   -Issuer 'https://token.actions.githubusercontent.com' `
   -Audience 'api://AzureADTokenExchange' `
   -Subject "repo:$($githubOrganizationName)/$($githubRepositoryName):ref:refs/heads/main"
```

Notice here that for our ``-ApplicationObjectId``, we use the Object Id of our workload identity that we created earlier, and our ``-Subject``, we assign our policies that associate our production environment and GitHub repository to our workload identity.

This covers the *authentication* part of our workflow. We now need to authorize our workload identity to contribute resources to our resource group. For this we can use Azure RBAC (Role-Based Access Control) to assign a role to our workload identity.

We'll need to create a service principal. For this, we can use the Application Id of our resource group that our workflow will deploy resources to. I've already created my resource group, so I'll retrieve it and store it in a variable like so:

```powershell
$productionResourceGroup = Get-AzResourceGroup -Name "rg-myreponame"
```

I can then use the ``AppId`` property to create a service principal for it.

```powershell
New-AzADServicePrincipal -AppId $($productionApplicationRegistration.AppId)
```

Now that our service principal has been created, we can assign it a role. I'm going to give my service principal the owner role over my resource group (I'll be creating role assignments in my Bicep template). There are many considerations of what role and scope you should give your service principal, so for your scenario, do spend some time thinking about this.

To create our role assignment, we can use the ``AppId`` of our workload identity, the ``Owner`` role for our ``-RoleDefinitionName`` and for our ``-Scope``, we can use the resource ID of our resource group:

```powershell
New-AzRoleAssignment `
   -ApplicationId $($productionApplicationRegistration.AppId) `
   -RoleDefinitionName Owner `
   -Scope $($productionResourceGroup.ResourceId)
```

With everything set up, we can now set up our secrets in GitHub Actions. We'll need to use the client ID, tenant ID and subscription ID to authenticate to Azure. We can print these out with the following code:

```powershell
$azureContext = Get-AzContext
Write-Host "AZURE_CLIENT_ID: $($productionApplicationRegistration.AppId)"
Write-Host "AZURE_TENANT_ID: $($azureContext.Tenant.Id)"
Write-Host "AZURE_SUBSCRIPTION_ID: $($azureContext.Subscription.Id)"
```

One by one, copy these values and save them as secrets in your GitHub repository. Here's a screenshot of what mine looks like:

![Our secrets in GitHub Actions](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wlyb1oq97x9w3owh7pnl.png)

## Using our workload identities in GitHub Actions

We can now use our workload identity in our GitHub Actions. First, we'll need to allow our deployment workflow with the ability to request tokens. We can do this by adding the ``permissions`` property:

```yaml
permissions:
  id-token: write
  contents: read
```

In our various steps, we can then use our secrets that we saved earlier to sign into Azure. When we use workload identities, we need to specify these three inputs like so:

```yaml
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

We can use this to sign into Azure when we deploy Bicep templates to Azure. For example, we could have a step in our GitHub Actions workflow like this:

```yaml
deploy-infra:
    runs-on: ubuntu-latest
    environment: Production
    needs: preview
    steps:
      - uses: actions/checkout@v2

      - uses: azure/login@v1
        with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        
      - name: Deploy Bicep File
        uses: azure/arm-deploy@v1
        with:
          resourceGroupName: ${{ secrets.AZURE_RG }}
          template: ./main.bicep
          parameters: ./parameters.prod.json
          failOnStdErr: false
```

When we run our GitHub Actions workflow, we're now using workload identities to authenticate to Azure

![Using workload identities in GitHub Actions](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mjna1eaifoae6n0mjz1k.png)

## Conclusion

In this article, I talked about workload identities, what they are and how we can use them in GitHub Actions deployment workflows.

We learned how we can use federated credentials to authenticate our workflows without having to store any secrets. We then learnt how we can use our workload identities in our GitHub Actions workflows.

If you want to learn more about workload identities, check out the following resources:

* [What are workload identities?](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identities-overview)
* [Application and service principal objects in Azure Active Directory](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals?tabs=browser)
* [Workload identity federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation)
* [Configure an app to trust an external identity provider](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp)
* [Configuring OpenID Connect in Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️