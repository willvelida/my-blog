---
title: "How to purge stale images from Azure Container Registry with ACR Tasks and GitHub Actions"
date: 2024-12-06
draft: false
tags: ["Azure", "Azure Container Registry", "GitHub Actions", "DevOps", "IaC"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hjo5p2um6lw5f9cgwa1e.png
    alt: "How to purge stale images from Azure Container Registry with ACR Tasks and GitHub Actions"
    caption: "Using Azure Policy, we can enforce compliance on our AKS clusters at the workload and Azure control plane levels"
---

Cleaning up stale images (images that you're not using) from your Azure Container Registry is important for a couple of reasons. First off, storing images in ACR isn't free. Even with the Basic SKU, once you go past your 10GB limit, you end up paying [$0.00516 AUD (at the time of writing) per GB of additional storage](https://azure.microsoft.com/en-au/pricing/details/container-registry/?WT.mc_id=MVP_400037). If you have container images that are just sitting there not being used, it'll waste money and second, it'll be difficult to manage your images.

Depending on your build pipeline, you might be building multiple versions of the same image. In my case, I'm using an Azure Subscription with MVP benefits, which gives me around $250 AUD a month to spend on Azure. I don't want to waste that money, so cleaning up stale images is an important task.

To do this, we can use [ACR tasks](https://learn.microsoft.com/azure/container-registry/container-registry-tasks-overview?WT.mc_id=MVP_400037) to clean up stale images. We can also use GitHub Actions to run scripts on a schedule, so this blog post outlines a solution that I've created that cleans up all the stale images within all the repositories in my Azure Container Registry.

## What are ACR Tasks?

ACR tasks provide the ability to perform compute execution workflows within ACR. You can use cloud-based container image building for platforms such as Linux, Windows, and ARM, enable automated builds triggered by source code updates, timer triggers, and updates to a container's base image, and perform on-demand container image builds.

## So how can we use it to clean up stale images?

We can use the AZ CLI to interact with our ACR, and use the Azure Container Registry CLI to create and run ACR tasks.

In my scenario, I have a single Azure Container Registry with multiple repositories inside it for each application. This is in my DEV environment, so I want to be able to run a nightly cron job that cleans up stale images, and keep at least the current version, plus 2 previous versions (total of 3).

To achieve this, we need to complete the following steps:

- Creating our ACR Task that cleans up our stale images.
- Creating a bash script that can run the ACR Task over multiple repositories in our container registry.
- Run a GitHub Actions workflow on a nightly schedule that logs into our ACR, retrieves all the repositories within our ACR, and cleans up all the stale images.

### Performing a 'dry run' of your task

ACR tasks allow us to perform a 'dry run' of our tasks before we actually commit them (similar to `BEGIN TRANSACTION` AND `ROLLBACK TRANSACTION` in T-SQL). It's a good idea to do this 'locally' in your terminal to see what would happen as a result of your task. So in my case, my ACR Task would look like this:

```bash
PURGE_CMD="acr purge --filter 'biotrackr-auth-svc:.*' --ago 0d --keep 3 --untagged --dry-run"
az acr run --cmd "$PURGE_CMD" --registry acrbiotrackrdev /dev/null
```

Did you notice the `--dry-run` flag that we passed through? Yep, you guessed it - That means that this is just a dry run of our task, and nothing will actually happen. You should also see the following output in the terminal:

```bash
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
```

### Removing stale images from multiple repositories with a bash script

Now that we have an idea of what our ACR task will do, we can now run it within a bash script to clean up all the stale images within each repository. Let's break down the specific ACR task first before creating our script.

The ACR task that we want to run will be as follows:

```bash
az acr run --registry $ACR_NAME --cmd "acr purge --filter '$repository:.*' --ago 0d --keep 3 --untagged" /dev/null
```

Let's break this down:

- `az acr run` will execute our command in the context of our Azure Container Registry.
- `--registry $ACR_NAME` specifies the name of the Azure Container Registry to run the command against.
- `--cmd "acr purge --filter '$repository:.*' --ago 0d --keep 3 --untagged"` will run the `acr purge` command with the following options.
    - `--filter '$repository:.*'` will target all images in the specified repository.
    - `--ago 0d` specifies the age of images to consider for purging. 0 days means all images within our repository.
    - `--keep 3` means that we will keep the 3 latest images.
    - `--untagged` includes untagged images in the purged operation. **If you don't do this, you'll just delete the tagged images within your repository**.
- `/dev/null` discards the output of the `az acr run`.

With this ACR Task, we need to be able to run it against all repositories within our Azure Container Registry. This can be achieved using a basic bash script like so:

```bash
#!/bin/bash

# Variables
ACR_NAME=${ACR_NAME}
RESOURCE_GROUP=${RESOURCE_GROUP}

# Get the list of repositories
repositories=$(az acr repository list --name $ACR_NAME --resource-group $RESOURCE_GROUP --output tsv)

# Loop through each repository and purge old images
for repository in $repositories; do
    echo "Purging old images in repository: $repository"
    az acr run --registry $ACR_NAME --cmd "acr purge --filter '$repository:.*' --ago 0d --keep 3 --untagged" /dev/null
done
```

Within this script, we:

- Set variables for both our Container Registry name and resource group.
- Use the AZ CLI to list all repositories within our Container Registry using the `az acr repository list` command and save that to tsv output.
- Use a `for` loop to loop over all repositories within our Container Registry and run our ACR Task to clean up our stale images.

### Running our bash script on a schedule via GitHub Actions

Now that we have our bash script, let's create a GitHub Actions workflow file that runs nightly (and also on demand just so we don't have to wait to midnight if we need to clean up our images).

We can run workflows in GitHub Actions by adding the `schedule` [event trigger](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule). We can pass through a CRON expression that will determine when our GitHub Action workflow. CRON jobs take the following format:

```bash
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
│ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
│ │ │ │ │
│ │ │ │ │
│ │ │ │ │
* * * * *
```

I use a tool like [crontab.guru](https://crontab.guru/) to help write a proper CRON expression.

To run jobs at midnight every day, we can use the following CRON job expression: `0 0 * * *`. If we want to run our GitHub Action workflow manually, we can use the `workflow_dispatch` trigger.

As part of our workflow, we need to log into Azure, retrieve the name of our Azure Container Registry, log into the registry, and pass that value along with the resource group name into our script so we can clean up our images.

With that in mind, let's create our workflow.

```yaml
name: Purge Old Container Images from ACR

on:
    schedule:
        - cron: '0 0 * * *' # Run daily at midnight
    workflow_dispatch:

permissions:
    contents: read
    id-token: write
    pull-requests: write

jobs:
    purge:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout repository code
              uses: actions/checkout@v4

            - name: Azure login
              uses: azure/login@v2
              with:
                client-id: ${{ secrets.AZURE_CLIENT_ID }}
                tenant-id: ${{ secrets.AZURE_TENANT_ID }}
                subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

            - name: Get ACR name
              id: getacrname
              run: |
                  acrName=$(az acr list --resource-group ${{ secrets.AZURE_RG_NAME_DEV }} --query "[0].name" -o tsv)
                  echo "::set-output name=acrName::$acrName"    

            - name: Login to Azure Container Registry
              run: az acr login --name ${{ steps.getacrname.outputs.acrName }}

            - name: Run Purge Script
              env:
                ACR_NAME: ${{ steps.getacrname.outputs.acrName }}
                RESOURCE_GROUP: ${{ secrets.AZURE_RG_NAME_DEV }}
              run: |
                chmod +x infra/scripts/purge-old-images.sh
                ./infra/scripts/purge-old-images.sh
          
```

In this workflow, we log into Azure using federated credentials (I've written a blog post on how to set that up [here](https://www.willvelida.com/posts/using-workload-identities-bicep-deployments-powershell/)). 

We then run a couple of AZ CLI commands. First we retrieve the name of our Container Registry, and then use that to log into it so we can perform our ACR task in our bash script.

With scripts, we can pass environment variables to them within GitHub Actions using the `env` parameter. Here, we pass through the name of our Container Registry that we retrieved in our `getacrname` step, along with the name of our resource group.

**One thing to note about workflows that run on a schedule** - These won't run unless they've been [merged into your default branch](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule). This also includes workflows that are manually triggered (without a branch trigger).

### Bringing it all together

Running our workflow in GitHub Actions, we can see our ACR Task produce the following output:

```bash
Purging old images in repository: biotrackr-auth-svc
WARNING: Queued a run with ID: cr4
WARNING: Waiting for an agent...
2024/12/06 05:51:26 Alias support enabled for version >= 1.1.0, please see https://aka.ms/acr/tasks/task-aliases for more information.
2024/12/06 05:51:26 Creating Docker network: acb_default_network, driver: 'bridge'
2024/12/06 05:51:26 Successfully set up Docker network: acb_default_network
2024/12/06 05:51:26 Setting up Docker configuration...
2024/12/06 05:51:26 Successfully set up Docker configuration
2024/12/06 05:51:26 Logging in to registry: acrbiotrackrdev.azurecr.io
2024/12/06 05:51:27 Successfully logged into acrbiotrackrdev.azurecr.io
2024/12/06 05:51:27 Executing step ID: acb_step_0. Timeout(sec): 600, Working directory: '', Network: 'acb_default_network'
2024/12/06 05:51:27 Launching container with name: acb_step_0
Deleting tags for repository: biotrackr-auth-svc
Deleting manifests for repository: biotrackr-auth-svc
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:32f79f40dabc09bf0581e10b93f693779f1867a4e2e0d6f4d3e5db324a40ac7f
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:33d69b862072d01e215e447cdc86b41d59f6eb0187666be4a86044bae5ecdc50
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:38a16a84f3c5319f7044211e29e5be961b144764ada2dde899de90fd9e107f6f
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:38dd20434603d6fbfba1bb682f5626c36a7a31875410e0208b5b64482933dfb1
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:47b78e5e0e79292615c3096874a0053d675fbe79e01bf8cdb374fcf533e5a979
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:659443a520da26a763adb223621f05cb683c8d163f157a4d72557b5321c4f297
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:6a0db37fcd9a4ebcd78532bee41d312583fc53becfd90638685100617e86d56d
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:73cc361920f56b08d3d9bb4c3c59744297fa635801e9a3b30997056c339d5f81
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:7614d7da1a49a871b2e1597309820ac81eab570613d1de0dbac25c16d4b32950
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:78eeafdac515edc5cbab7b35c1c62c93f9ed7b555e5bd23d7dbc8e0a2b4f7b2c
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:7b82d9271181455c8d8309dddd2905f94df0dfd998d9539ba667fb31a203707b

Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:86e2fa4f7df872e2c2c266fd919d13d8736590fd2b35a2b4dd64a439459b550d
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:a8ff90db87ff8f30185e0336d6d6055840a25823277c867669b039382204886b
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:ac8d0f7746d36179b542c60182b8513969086a2cd6f091548373258143bba99a
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:ee76214fee3e10fb3bfe3e5d62f0bc39255e3370ff550934d0d7ad065dcc37ea
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:f678340dbd4902b33dea78e9eae4aa1a0e69eba04536da99b68e1e5b43f38255
Deleted acrbiotrackrdev.azurecr.io/biotrackr-auth-svc@sha256:fbed0090dc34ea4874787184d7c4f5f041cd88a8f6a8ddcb750b709973976ed6

Number of deleted tags: 0
Number of deleted manifests: 17
2024/12/06 05:52:00 Successfully executed container: acb_step_0
2024/12/06 05:52:00 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 33.162147)
```

There was quite a bit to clean up!

With ACR tasks, we can deploy them as part of our [infrastructure as code](https://learn.microsoft.com/azure/templates/microsoft.containerregistry/registries/tasks?pivots=deployment-language-bicep&WT.mc_id=MVP_400037) or we can perform quick ACR tasks as runs. When we run ACR Tasks as scripts, these are run as quick tasts, so we can view past runs in the *Runs* tab in **Tasks** of our Container Registry. 

![Image of an Azure Container Registry in the Azure Portal, with a view of ACR Task logs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bk68jk60wtujk598czqf.png)

We can also view the log output of a specific run. Here we can see that all the images that we consider stale have been removed (both the tagged images and untagged).

![Image of a log output of a specific ACR Task](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8ldp7qxqpazludgock7e.png)

Since we wanted to keep the latest 3 versions of our image in our repository, we can navigate to one of our repositories and verify that there are 3 images remaining.

![Image of a Azure Container Registry repository, with three container images left](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h6q9hwoyy1n8gfsftfpb.png)

## Conclusion

With ACR tasks, we can clean up stale images that we aren't using anymore, and save ourselves a bit of money on storage costs. In this article, I've integrated the AZ CLI, Bash and GitHub Actions to clean up our images in all of our repositories in our Container Registry.

We can use the AZ CLI to run ACR Tasks on our Container Registry directly, we can create ACR tasks in Bicep. So no matter how you want to create and deploy ACR tasks, hopefully this article has explained how you can use them to perform container related actions on your Azure Container Registry.

If you want to see the code that I talked about, please check out my [GitHub](https://github.com/willvelida/biotrackr)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! I'm loving BlueSky at the moment. It has a much better UX and performance than Twitter these days.

Until next time, Happy coding! 🤓🖥️