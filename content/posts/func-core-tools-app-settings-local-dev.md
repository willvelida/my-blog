---
title: "Using Azure Functions Core Tools to fetch app settings for local development"
date: 2022-02-13T20:58:14+13:00
draft: false
tags: ["Azure","C#","Serverless","Azure Functions"]
ShowToc: true
TocOpen: true
---

*Update: If you prefer watching videos to reading, I made a video on my [YouTube channel](https://www.youtube.com/channel/UCgV0RWjqmZ-Zl6vfoH2Wqjg) that covers this content*

Before deploying our Azure Functions, it's good to debug our Functions locally to ensure that it works as expected. Running our Functions locally requires a *local.settings.json* file to store our application settings.

When we first create a Function in Visual Studio, a local.settings.json file is generated for us. However, if we clone a Function app from a repository, this file won't be cloned! (*Hopefully* it won't be cloned. It's good practice not to commit this file to your repo since it has application secrets!).

Thankfully, we can use Azure Function Core Tools to create a local.settings.json file and import our Function settings to that file so we can run our Functions locally as if we were running it against that environment!

## Why would we do this?

Say if we have multiple environments (DEV, Test, UAT etc) and we wanted to locally debug a Function using that environments settings, we can use Azure Function Core Tools to simplify the retrieval of those settings. Especially for Functions that have a lot of settings. We wouldn't want to waste our time copying and pasting all those settings!

## Before we start

You'll need to make sure that the [Azure Function Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cportal%2Cbash%2Ckeda) have been installed on your machine. Azure Functions Core Tools include a version of the same runtime that powers Azure Functions Runtime that we can run on our machine.

While we'll just be focusing on fetching our app settings, Functions Core Tools come with commands that allow us to create Functions, deploy our Functions and more!

If you haven't installed the tools yet, check out [this documentation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cportal%2Cbash%2Ckeda#install-the-azure-functions-core-tools) to get started.

## Let's begin

For this tutorial, I've cloned one of my existing projects from GitHub. I've deployed this Function to Azure already and my .gitignore file excludes the local.settings.json file that was generated when I created the project.

To help me with local debugging, I'm going to use the Function Core tools to help fetch my app settings from Azure. Just to verify that the local settings file isn't in my project directory, here's a screen shot:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ni7v0v1a8r9e0gsm7iue.png)

We'll need to create a JSON file to write our App settings to. To do this, right-click your Functions Project and create a new file. Create a *JavaScript JSON Configuration File* and give it the name *local.settings.json*

### Step 1: Set our Azure Subscription

If you work with multiple Azure Subscriptions, we'll need to set it using AZ CLI to ensure that when we run our Function Core Tools, it's looking in the right place.

Open up the command line or PowerShell on your machine and log into your Azure Account by running the following command:

```bash
az login
```

Once you're logged in, you should see a list of subscriptions to which you have access to. To set the subscription that your Function is hosted in, run this command:

```bash
az account set -s "<subscription-name-or-id>"
```

-s being the shorthand parameter for subscription.

### Step 2: Fetch our App Settings

Now we can fetch our app settings from our Function App in Azure. Make sure you are in your Project's directory and run the following command in your terminal:

```bash
func azure functionapp fetch-app-settings '<function-name>' --output-file local.settings.json
```

You should see an output in your terminal similar to this:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lnoy25jc3mclpucl91xq.png)

Go back into your project in Visual Studio and check out your *local.settings.json* file. We can see that our Function settings have been retrieved and written to our local settings file!

However, there's just one problem....

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/plc1tzk08kt49ykzwmd1.png)

Our settings are encrypted! That's not going to help us when running our Function locally! Function app settings are encrypted when stored and are only decrypted before being injected into your Function's process memory when it starts. 

To learn more about application settings in Azure Functions, check out this [article](https://docs.microsoft.com/en-us/azure/azure-functions/security-concepts#application-settings).

### Step 3: Decrypting our settings.

Thankfully, we can decrypt our settings using Azure Function Core Tools! In your terminal, run the following command:

```bash
func settings decrypt
```

Head back into Visual Studio and you'll see that your settings have been decrypted and ready to use for your local debugging needs!

(*Of course I'm not going to give you a screenshot of that! It's secret!* 😂😉).

We can now start up our Function locally, as if we were running it against our Azure environment!

One thing to note here! If you are storing some of your application settings in Key Vault (as you should!), then when you retrieve the setting, you may only get the Key Vault URL of which the secret is stored in. To use the actual value, you'll need to retrieve this from your Key Vault.

## Wrapping up

As you can see, using Azure Function Core Tools can help speed up our development process by allowing us to quickly retrieve our Function App settings.

If you want to learn more, check out the following resources:

- [Work with Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cportal%2Cbash%2Ckeda)
- [Azure Functions Core Tools GitHub](https://github.com/Azure/azure-functions-core-tools)