---
title: "Deploying to Azure with Terraform and GitHub Actions"
date: 2024-09-19
draft: false
tags: ["Terraform","GitHub Actions","DevOps","Azure", "Microsoft Entra ID", "Azure Kubernetes Service"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7ufokytrvyebfdhqsyxq.png
    alt: "Deploy to Azure with Terraform and GitHub Actions"
    caption: "What started out as building a simple AKS cluster for my own learning, became a full-on CI/CD pipeline..."
---

I'm building my own Azure Kubernetes Cluster that I can use for my personal development, and I've been wanting to improve my Terraform skills, so I've spent a bit of time over the past couple of days getting a Terraform deployment to work with GitHub Actions.

The AzureRM provider has moved on a bit since I've used it in anger, so I learnt a lot about the different resources that are available, and how we can use GitHub Actions to deploy Terraform templates to Azure.

In this post, I'll cover what I've done so far with my Terraform templates, how I've structured my code, and how I've implemented my GitHub Actions workflow. It's my hope that this blog post has something for everyone, whether you've just started out with Terraform and are looking to see how you can automate deployments through GitHub Actions, or you're someone who's been deep in Terraform for a while and are just looking for a reference to help you with a particular problem.

If you just want to look at the code, you can do so [here](https://github.com/willvelida/my-aks-cluster). If you want extra context, read on!

## What did I want to achieve?

What started out as just an exercise in curiosity to see how I could build an AKS cluster for learning become a bit of a larger task. The purpose of this was to just get a simple GitHub Actions workflow going that would automate the deployment of my AKS cluster.

Over time, I'm going to be adding things to the cluster, such as monitoring, resiliency, tests etc. and thought it'd be cool to automate the deployment rather than run it on my local machine.

*Just a bit of advice - It does take more effort, but it's worth it. I did learn a lot which I can apply to my day-to-day.*

With that in mind, I'll split out what I did into 3 steps:

1. Structuring my Terraform Code
2. Creating the resources needed for GitHub Actions
3. Deploying my AKS cluster via GitHub Actions

## Step 1: Structuring my Terraform Code

I'm put all my code in one repository. I didn't want to separate it out into different repositories, since this will just all be used for my AKS cluster deployments.

At the time of writing, I've implemented the following folder structure in my repository:

```
├── .github
│   ├── workflows
├── cluster-deployment
│   ├── tfvars
├── github-deployment
│   ├── tfvars
├── modules
│   ├── aks-cluster
│   ├── azure-container-registry
│   ├── etc. etc.
├── LICENSE
├── rEADME.md
└── .gitignore
```

The main purpose behind this was to separate the GitHub related terraform code to the AKS terraform code. The pipeline is currently configured to set the working directory to the ```./cluster-deployment``` folder when it runs (more on this later), so splitting these out into two folders made this more manageable. 

I have a separate folder for some Terraform modules that I've created. There are some modules that cater just to the AKS deployment, and others that I only need for the GitHub Actions related stuff.

## Step 2: Creating our resources for GitHub Actions.

For our Terraform deployments, we'll need to do a couple of things before we can start writing our GitHub Actions workflow file:

1. Create a User Assigned Managed Identity for OIDC authentication.
2. Create federated credentials for the managed identity.
3. Create a Azure Storage account and container to store our state file.
4. Deploy the resources via the CLI

### Using User Assigned Managed Identities instead of Service Principals for federated credentials

In the past, I've used Service Principals with Federation to use OIDC for my GitHub Actions deployments (I've blogged on how to do this [here](https://www.willvelida.com/posts/using-workload-identities-bicep-deployments-powershell/)).

Using User Assigned Managed Identities instead doesn't require us to have elevated permissions in Microsoft Entra ID, and it has a longer token timeout. Both are relatively straightforward to implement in Terraform, but the difference is in the Terraform provider. User Assigned Identities use the the AzureRM provider, while App Registrations use the Azure AD provider.

You can create a User Assigned Identity with the following Terraform resource:

```terraform
resource "azurerm_user_assigned_identity" "msi" {
  location = var.location
  name = var.name
  resource_group_name = var.rg_name
  tags = var.tags
}
```

Nothing too complex here. Since I want to modularize this, I'll also create a ```variables.tf``` file with the following variables that I can provide values to when using the module:

```terraform
variable "location" {
  description = "The location where the user-assigned managed identity will be created."
  type = string
}

variable "name" {
  description = "The name of the user-assigned managed identity."
  type  = string
}

variable "rg_name" {
  description = "The name of the resource group in which the user-assigned managed identity will be created."
  type = string
}

variable "tags" {
  description = "value of tags to assign to the user-assigned managed identity."
  type = map(string)
}
```

We'll also need to provide a couple of outputs in a ```outputs.tf``` file so we can use them as inputs to other modules. We can do this like so:

```terraform
output "user_assinged_identity_id" {
  value = azurerm_user_assigned_identity.msi.id
  description = "ID of the user-assigned managed identity."
}

output "user_assinged_identity_principal_id" {
  value = azurerm_user_assigned_identity.msi.principal_id
  description = "Principal ID of the user-assigned managed identity"
}
```

We can then use the module like so. I've also created a new resource group to store the User Assigned Identity (personal choice on my part):

```terraform
module "identity-resource-group" {
  source   = "../modules/resource-group"
  name     = var.identity_rg_name
  location = var.location
  tags     = var.tags
}

module "gh_usi" {
  source   = "../modules/user-assigned-identity"
  name     = "${var.gh_uai_name}-${var.environment}"
  location = var.location
  rg_name  = module.identity-resource-group.name
  tags     = var.tags
}
```

For those of you who are new to Terraform, let's cover this code in more detail.

**Resources** are the most important element in Terraform. Each resource block that we use describes one or more infrastructure objects. In my case, I've used a resource block to describe the User Assigned Managed Identity that I want to create.

**Modules** are containers for multiple resources that are used together. A module consists of a collection of ```.tf``` files that are kept together in the same directory. So in our ```../modules/user-assigned-identity``` directory, I have the following structure in place:

```
├── modules
│   ├── user-assigned-identity
|       |── main.tf
|       |── variables.tf
|       |── outputs.tf
```

So all those Terraform files are now part of the module.

Finally, **Input variables** serve as parameters for a Terraform module, so that we can customize the implementation of the resource without editing the source, while **Output Values** are like return values for a Terraform module.

To learn more about these resources, check out the following resources on the Terraform docs:

- [Resource Blocks](https://developer.hashicorp.com/terraform/language/resources).
- [Variables and Outputs](https://developer.hashicorp.com/terraform/language/values)
- [Modules](https://developer.hashicorp.com/terraform/language/modules)

As part of my deployment, I'll need to assign the **Owner** role to the User Assigned Identity. This is so we can create the role assignments that I'll be using for both my GitHub and AKS cluster deployments, so I've created another module for that:

```terraform
resource "azurerm_role_assignment" "role" {
  principal_id = var.principal_id
  principal_type = var.principal_type
  role_definition_name = var.role_name
  scope = var.scope_id
}
```

Which we implement in our ```main.tf``` file like so:

```terraform
module "sub_owner_role_assignment" {
  source       = "../modules/role-assignment"
  principal_id = module.gh_usi.user_assinged_identity_principal_id
  role_name    = var.owner_role_name
  scope_id     = data.azurerm_subscription.sub.id
}
```

To be able to grant my managed identity the owner role over the subscription as the scope, we need to be able to retrieve the Id of the Subscription. To use the Subscription Id in my ```main.tf``` file, we can import our subscription as a data resource like so:

```terraform
data "azurerm_subscription" "sub" {
}
```

**Data sources** allow Terraform to use information defined outside of Terraform, defined by another separate Terraform configuration, or modified by functions. The data blok requests that Terraform reads from a given data source (in our case, ```azurerm_subscription```) and export the result under the given local name ```sub```. The name is then used to refer to this resource from elsewhere in the same Terraform file.

So in our role assignment module where we need the Subscription Id, we can use provide it from the data resource using ```data.azurerm_subscription.sub.id```.

To learn more about data sources in Terraform, check out the [following documentation](https://developer.hashicorp.com/terraform/language/data-sources).

### Creating the Federated Credentials

To authenticate to Azure Services securely from GitHub Actions, we'll need to login using OpenID Connect, or OIDC.

OpenID Connect is an identity authentication protocol that's an extension of OAuth 2.0 that standardizes the process for authenticating and authorizing users when they sign in to access services. To learn more about OIDC, check out this [article](https://www.microsoft.com/en-au/security/business/security-101/what-is-openid-connect-oidc).

GitHub Actions can use OIDC in workflows by requesting a short-lived access token directly from Azure. This is better than using credentials, as we won't need to create credentials for GitHub to use and then duplicate them as secrets in GitHub. We can also have more granular control over how workflows can use credentials, and since access tokens are only valid for a single job, we get the benefits of rotating credentials.

When we create a federated credential for a deployment workflow, we are telling Microsoft Entra ID and GitHub to trust each other. When our GitHub Actions workflow attempts to sign in, GitHub will provide information about the workflow run so that Microsoft Entra ID can decide whether or not to allow the sign in.

For our federated credentials, we need to provide information about the *issuer*, *subject*, and *audiences*. The *issuer* is the URL of the external identity provider and must match the *issuer* claim of the external token. The *subject* is the identifier of the external software workload and must match the *sub* or *subject* claim of the external token being shared. Each IdP uses their own *subject*. Finally, *audiences* lists the audiences that can appear in the external token.

To store the *audience* and *issuer* values, I've added these to a ```local.tf``` file like so:

```terraform
locals {
  default_audience_name = "api://AzureADTokenExchange"
  github_issuer_url     = "https://token.actions.githubusercontent.com"
}
```

In my GitHub Actions workflow, I want to be able to trigger the workflow off a pull request, and I want to use an environment to stage the deployment. In order to do this, I need to create 2 federated identity credentials. So I'll create a module for a federated identity credential like so:

```terraform
resource "azurerm_federated_identity_credential" "cred" {
  name = var.federated_identity_credential_name
  resource_group_name = var.rg_name
  audience = [ var.audience_name ]
  issuer = var.issuer_url
  parent_id = var.user_assigned_identity_id
  subject = var.subject
}
```

To create the federated credential identities, I can do so by implementing the modules in my ```main.tf``` file like so:

```terraform
module "gh_federated_credential" {
  source                             = "../modules/federated-identity-credential"
  federated_identity_credential_name = "${var.github_organization_target}-${var.github_repository}-${var.environment}"
  rg_name                            = module.identity-resource-group.name
  user_assigned_identity_id          = module.gh_usi.user_assinged_identity_id
  subject                            = "repo:${var.github_organization_target}/${var.github_repository}:environment:${var.environment}"
  audience_name                      = local.default_audience_name
  issuer_url                         = local.github_issuer_url
}

module "gh_federated_credential-pr" {
  source                             = "../modules/federated-identity-credential"
  federated_identity_credential_name = "${var.github_organization_target}-${var.github_repository}-pr"
  rg_name                            = module.identity-resource-group.name
  user_assigned_identity_id          = module.gh_usi.user_assinged_identity_id
  subject                            = "repo:${var.github_organization_target}/${var.github_repository}:pull_request"
  audience_name                      = local.default_audience_name
  issuer_url                         = local.github_issuer_url
}
```

Notice the different between the *subject* values in the two federated identity credentials. The first one with the subject ```repo:${var.github_organization_target}/${var.github_repository}:environment:${var.environment}``` will grant my GitHub Actions workflow the ability to authenticate to Azure when we're deploying in our dev environment on GitHub. The second identity with the subject ```repo:${var.github_organization_target}/${var.github_repository}:pull_request``` will enable our workflow to authenticate to Azure whenever we are trigging our workflow as part of a pull request. 

With GitHub, you can also use a branch name as the subject, for example ```repo:my-github-user/my-repo:ref:refs/heads/main```.

### Creating resources to manage our .tfstate file

Terraform must store state about your managed infrastructure and configuration. Terraform uses this state to map real world resources to your configuration, keep track of metadata, and to improve performance. It then uses the state file to determine which changes to make to your infrastructure. To learn more about state in Terraform, check out the following [documentation](https://developer.hashicorp.com/terraform/language/state).

This state is stored by default in a ```terraform.tfstate``` file. **DO NOT COMMIT THIS FILE TO SOURCE CONTROL!**.  Instead we can store it inside an Azure Storage Account.

To create this, I've created the following module. I only need one container, so there's nothing complex that I need to do here:

```terraform
resource "azurerm_storage_account" "account" {
  name = var.storage_account_name
  location = var.location
  resource_group_name = var.resource_group_name
  account_tier = var.account_tier
  account_replication_type = var.account_replication_type
  tags = var.tags
}

resource "azurerm_storage_container" "container" {
  name = var.container_name
  storage_account_name = azurerm_storage_account.account.name
  container_access_type = "private"
}
```

So back in my ```main.tf``` file, I implement the module like so. Again, I'm creating another resource group for the storage account:

```terraform
module "tf-resource-group" {
  source   = "../modules/resource-group"
  name     = var.tf_state_rg_name
  location = var.location
  tags     = var.tags
}

module "tf-state-storage" {
  source                   = "../modules/tfstate-storage"
  storage_account_name     = var.storage_account_name
  resource_group_name      = module.tf-resource-group.name
  location                 = var.location
  tags                     = var.tags
  account_replication_type = var.account_replication_type
  account_tier             = var.account_tier
  container_name           = var.container_name
}
```

I also need to grant the User Assigned Identity that I'm using for my GitHub Actions deployments the **Storage Blob Data Contributor** role, so that it can write the ```.tfstate``` file to my storage account. Using my module from earlier, I can do this like so:

```terraform
module "tfstate_role_assignment" {
  source       = "../modules/role-assignment"
  principal_id = module.gh_usi.user_assinged_identity_principal_id
  role_name    = "Storage Blob Data Contributor"
  scope_id     = module.tf-state-storage.id
}
```

### Deploying our GitHub resources

For my resources that I'll use for my GitHub Actions deployments, I decided to just run this on my machine rather than use a GitHub Actions workflow just to make things simpler. Let's take this chance to learn what commands we need to run to deploy our Terraform code.

The core Terraform workflow consists of three main steps.

1. **Initialize** - This prepares your workspace so Terraform can apply your configuration.
2. **Plan** - This allows you to preview the changes Terraform will make before you apply them.
3. **Apply** - This makes the changes defined by your plan to create, update, or destroy resources.

Let's start with the **Initialize** stage. Open a terminal, or use a integrated terminal within an IDE like Visual Studio Code and navigate to the ```./github-deployment``` folder. Within that folder, run the following:

```bash
$ terraform init
```

The ```terraform init``` command initializes a working directory containing Terraform configuration files. When we run this command, it prepares the current working directory for use with Terraform. This includes initializing all the modules that you're using within your ```main.tf``` file, and configuring the providers that you need to use.

Once Terraform has been initialized, we need to format and validate our configuration. To format your code, run the following:

```bash
$ terraform fmt
```

This command will automatically update configurations in the current directory for readability and consistency. Terraform will print out the name of any files that it modified during this process.

We also need to make sure that our configuration is syntactically valid and consistent by using the following command:

```bash
$ terraform validate
```

Now let's focus on the **Plan** stage. When we provision infrastructure with Terraform, it creates an execution plan before it applies any changes. It creates the plan by comparing your Terraform configuration to the state of your infrastructure.

To create a plan, we can run the following:

```bash
$ terraform plan -var-file=<FILENAME> -out tfplan 
```

This command creates a plan based on the set of changes that it will make to match your resources with your configuration. In the above command, you can provide a file that contains the values for your variables, and get the plan to produce the plan in a file. We can then apply the plan in our **Apply** stage like so:

```bash
$ terraform apply "tfplan"
```

You don't have to save the plan. If you don't save the plan, Terraform will create a new plan and prompt you for approval before applying the plan.

For our pipelines, I will be saving a plan so that Terraform only makes the changes I expect it to make. Again, **PLEASE DON'T COMMIT THE PLAN TO SOURCE CONTROL!** Once our resources have been deployed, we need to add some secrets to GitHub so that our pipeline can use them.

For our User Assigned managed identity, we need the following secrets:

| GitHub Secret | User Assigned Managed Identity Property |
| ------------- | --------------------------------------- |
| AZURE_CLIENT_ID | Client ID |
| AZURE_SUBSCRIPTION_ID | Subscription ID |
| AZURE_TENANT_ID | Directory (tenant) ID |

For our storage account, we need the following secrets:

| GitHub Secret | Storage Account Property |
| ------------- | ------------------------ |
| BACKEND_AZURE_RESOURCE_GROUP_NAME | Name of the resource group that the storage account has been deployed to. |
| BACKEND_AZURE_STORAGE_ACCOUNT_NAME | Name of the Storage Account used to store the ```.tfstate``` file. |
| BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME | Name of the Blob Container used to store the ```.tfstate``` file. |

To learn more about how to use secrets in GitHub Actions, check out the following [documentation](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions).

## Step 3: Deploying my AKS cluster via GitHub Actions

With all that setup, we can now look at deploying the AKS cluster with a GitHub Actions workflow file. For this step, I won't focus too much on the Terraform code, but pay more attention to the GitHub Actions workflow itself.

There are some slight changes we'll need to make to our Terraform to ensure that it'll use OIDC authentication, since we'll be running our Terraform commands via GitHub Actions rather than our own local machines.

We can also use different jobs within our GitHub Actions workflow file to separate out our **Plan** and **Apply** stages.

### Configuring our Azure provider to use OIDC

To configure OIDC authentication for our Terraform deployments, I've created a new file called ```providers.tf``` and written the following:

```terraform
terraform {
  required_version = ">=1.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
    azapi = {
      source  = "azure/azapi"
      version = "~>1.5"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "2.30.0"
    }
  }

  backend "azurerm" {
    key      = "terraform.tfstate"
    use_oidc = true
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

provider "azapi" {
  use_oidc = true
}
```

I'll be using both the AzureRM provider and AzAPI providers in my Terraform code.

For the AzureRM provider, we need to configure the backend correctly so that we can store our state file in Azure Storage. Within my Terraform code, I've provided values for both enabling OIDC authentication, and also the name of the state file that the AzureRM provider should use.

When we initialize Terraform using ```terraform init```, we can pass it some backend configuration in the command by using the ```-backend-config``` flag. For OIDC authentication, this includes the details of our Storage account (the resource group, account name, and container name), and the details of our User Assigned Managed Identity (client ID, Subscription ID, and the Tenant ID). We also pass in a flag to tell configure our provider to use Microsoft Entra ID authentication.

We also need to set the ```use_oidc``` flag to true for both our AzureRM and AzAPI provider.

### Setting up our GitHub Actions workflow

Now we can start to look at our GitHub Actions workflow (finally!). After we have created our workflow identity and assign it access to our Azure environment, we can use it within our workflow.

To allow our deployment workflow to request tokens, we need to add the ```permissions``` property like so:

```yaml
name: Deploy AKS Cluster Infra
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  id-token: write
  pull-requests: write

env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_USE_AZUREAD: true
```

Let's describe these permissions one by one:

- ```contents: read``` - This works with the contents of a repository. This specific permission permits an action within our workflow to list the commits.
- ```id-token: write``` - This fetches an OpenID Connect (OIDC) token.
- ```pull-requests: write``` - This permits an action within your workflow to add a label to a pull request. More on this later.

To learn more about how you can control permissions for GitHub tokens within your workflow, check out the following [documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token).

### Generating our Terraform Plan

The first stage of our GitHub Actions workflow will be to generate a Terraform plan that we can manually verify before we apply it. I'll break our workflow file into sections and explain each step.

Let's take a look at the defaults:

```yaml
jobs:
  terraform-plan:
    if: github.event_name == 'pull_request'
    defaults:
      run:
        working-directory: ./cluster-deployment
```

Essentially all this means is that for every pull request, run this job. All my Terraform code for my AKS cluster sits in the ```./cluster-deployment``` folder. So for my GitHub Actions workflow, I'm setting that directory as my working directory.

The next step is to run our Terraform workflow:

```yaml
name: Terraform Plan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Fmt
        id: fmt
        run: terraform fmt -check

      - name: Terraform Init
        id: init
        run: terraform init -backend-config="resource_group_name=${{secrets.BACKEND_AZURE_RESOURCE_GROUP_NAME}}" -backend-config="storage_account_name=${{secrets.BACKEND_AZURE_STORAGE_ACCOUNT_NAME}}" -backend-config="container_name=${{secrets.BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME}}"

      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
```

The only thing I want to call out here is that rather than just running ```terraform init``` by itself, we're providing some arguments to our flags to configure our Azure Storage account as our backend for the state file. These arguments are coming from our secrets in GitHub, as well as our environment variables that we've defined earlier in our GitHub Actions workflow.

The next step in our pipeline is to do some static code analysis of our Terraform code to spot any potential misconfigurations. I'm using tfsec for this, but [tfsec is migrating over to Trivy](https://github.com/aquasecurity/tfsec) - I'll cover that in a future blog post, but for now we can add this using the following:

```yaml
- name: tfsec
  uses: aquasecurity/tfsec-pr-commenter-action@v1.2.0
  with:
    tfsec_args: --soft-fail
    github_token: ${{ github.token }}
```

Now we can generate our Terraform plan, publish the plan as an artifact, and then update our PR to show the status of the plan, as well as the details of our plan.

```yaml

      - name: Terraform Plan
        id: plan
        run: |
          export exitcode=0
          terraform plan -no-color -var-file="./tfvars/terraform.tfvars" -var="azure_object_id=${{ secrets.AZURE_OBJECT_ID }}" -out main.tfplan || export exitcode=$?

          echo "exitcode=$exitcode" >> $GITHUB_OUTPUT

          if [ $exitcode -eq 1 ]; then
            echo "Error: Terraform plan failed"
            exit 1
          else
            echo "Terraform plan was successful"
            exit 0
          fi
        
      - name: Publish Terraform Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ./cluster-deployment/main.tfplan

      - name: Update Pull Request
        uses: actions/github-script@v6
        env:
          PLAN: "terraform\n${{ steps.plan.outputs.stdout }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
  
            <details><summary>Show Plan</summary>
  
            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`
  
            </details>
  
            *Pushed by: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
  
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
      
      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Create String Output
        id: tf-plan-string
        run: |
            TERRAFORM_PLAN=$(terraform show -no-color main.tfplan)
            
            delimiter="$(openssl rand -hex 8)"
            echo "summary<<${delimiter}" >> $GITHUB_OUTPUT
            echo "## Terraform Plan Output" >> $GITHUB_OUTPUT
            echo "<details><summary>Click to expand</summary>" >> $GITHUB_OUTPUT
            echo "" >> $GITHUB_OUTPUT
            echo '```terraform' >> $GITHUB_OUTPUT
            echo "$TERRAFORM_PLAN" >> $GITHUB_OUTPUT
            echo '```' >> $GITHUB_OUTPUT
            echo "</details>" >> $GITHUB_OUTPUT
            echo "${delimiter}" >> $GITHUB_OUTPUT

      - name: Publish Terraform Plan to Task Summary
        env:
          SUMMARY: ${{ steps.tf-plan-string.outputs.summary }}
        run: |
          echo "$SUMMARY" >> $GITHUB_STEP_SUMMARY

      - name: Push Terraform Output to PR
        if: github.ref != 'refs/heads/main'
        uses: actions/github-script@v7
        env:
          SUMMARY: "${{ steps.tf-plan-string.outputs.summary }}"
        with:
            github-token: ${{ secrets.GITHUB_TOKEN }}
            script: |
                const body = `${process.env.SUMMARY}`;
                github.rest.issues.createComment({
                    issue_number: context.issue.number,
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    body: body
                })
```

If we navigate to our PR, the github-actions bot will comment on our PR the outcomes of our various Terraform commands like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t60s3zvdg1j8fo29guap.png)

And it will add a comment showing the Terraform Plan itself.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qbw4l20fs44mp7gz2ufl.png)

The plan will also be uploaded as an artifact to GitHub, which we can see in the Actions UI:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/phbdoroest574kf91gcv.png)

### Applying our Terraform Plan

With our plan generated and saved as an artifact within GitHub, we can now attempt to apply the plan in another GitHub Actions job, which will deploy our infrastructure. Again, I'll break this file down into the various moving parts.

First up, like before I'm setting the ```working-directory``` to the folder where my Terraform code sits. I then initialize my Terraform environment using the same backend configuration that I did for the **Plan** job:

```yaml
 terraform-apply:
    needs: terraform-plan
    name: Terraform Apply
    runs-on: ubuntu-latest
    environment: dev
    defaults:
      run:
        working-directory: ./cluster-deployment

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        id: init
        run: terraform init -backend-config="resource_group_name=${{secrets.BACKEND_AZURE_RESOURCE_GROUP_NAME}}" -backend-config="storage_account_name=${{secrets.BACKEND_AZURE_STORAGE_ACCOUNT_NAME}}" -backend-config="container_name=${{secrets.BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME}}"
```

I now want to retrieve the plan that Terraform generated earlier and apply it. In my workflow file, we can use the ```actions/download-artifact``` task to achieve this.

```yaml
- name: Download Terraform Plan
  uses: actions/download-artifact@v4
  with:
    name: tfplan
    path: ./cluster-deployment
```

**This was a PAIN for me to figure out!** - Maybe because I'm getting forgetful as I get older 😁

The thing that tripped me up here was that since I have set the ```working-directory``` to another folder, the artifact was being saved there, but the task was looking for it in the root directory. So when I attempted to download the artifact, it would fail because it couldn't find the file.

To resolve this, you need to set the ```path``` property in the task to the directory of your working directory, like I have done above.

Once we're able to download our Terraform plan, we can then apply it using ```terraform apply```. In our GitHub Action workflow file, we can add the ```-auto-approve``` flag to automatically approve the apply.

```yaml
- name: Terraform Apply
  run: terraform apply -auto-approve "./main.tfplan"
```

## Conclusion

We've covered a lot of things in the post. Hopefully you've found it useful. There's a lot more that we can do here, especially around testing and security, but I'll leave that to another day.

If you're learning about the cloud (whether that be Azure, AWS, or GCP), and you're figuring out how you can use Infrastructure-as-code to deploy resources to the cloud, I highly recommend that you learn how to do so through CI/CD, such as GitHub Actions. You'll learn a lot about how authentication works, how the CI/CD tool works, how Terraform (or any other IaC tool) works. It's a lot of effort, but it's worth it in the long run.

If you have any questions about this, please feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️