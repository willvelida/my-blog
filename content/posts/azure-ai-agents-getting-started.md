---
title: "Building AI Agents with Azure AI Agent Service"
date: 2025-05-17
draft: false
tags: ["Azure", "AI Agents", "AI", "GenAI", "LLMs"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7mfav6a8b7qexni47yfr.png
    alt: "How to build Azure AI Agents with Azure AI Agent Service"
    caption: "Azure AI Agent Service is a fully managed service designed to empower developers to securely build, deploy, and scale high-quality, and extensible AI agents without needing to manage the underlying compute and storage resources."
---

As the technology advances, Generative AI models are becoming powerful enough to operate autonomously to automate tasks. This is improvement on being able to perform simple tasks in "chat" like applications. This allows us to build AI Agents, which are applications that can use generative AI models with contextual data to automate tasks based on user input and the context that they can perceive.

In this article, I'll talk about how we can build AI Agents using Azure AI Agent service.

## What are AI Agents?

An AI Agent is an application that uses generative AI to understand and perform tasks on behalf of a user or another program. Agents use AI models to understand context, make decisions, use grounding data, and take actions to achieve a specific goal.

AI Agents can operate independently to execute complex workflows and automate processes without the need of human intervention.

AI Agents are great for a number of reasons:

1. **Automating routine tasks** - Rather than doing reptitive and mundane tasks yourself, AI Agents can handle them for you, giving you time to focus on more important work.
2. **Enhanced Decision-making** - AI Agents can be more useful than ordinary GenAI applications because they can handle complex scenarios (like data analysis, outcome prediction etc.)
3. **Scalability** - AI Agents can scale without increasing the need for human resources.
4. **High Availability** - AI Agents can operate continuously without breaks.

## What is Azure AI Agent Service?

Azure AI Agent Service is a service within Azure AI Foundary that we can use to create, test, and manage AI Agents. We can create agents either directly within the Azure AI Foundry portal, or through the Azure AI Foundry SDK.

With Azure AI Agent Service, we can create AI Agents that answer questions, perform actions, or automate workflows. Agents developed using Azure AI Agent Service have the following components:

- **Models** - For any GenAI application, we're going to need a GenAI model. This enables the agent to reason and generate natural language responses to prompts. Azure AI Foundry provides numerous amounts of models that we can choose from in it's model catalog.
- **Knowledge** - This is used to ground prompts that we send the agent with contextual data. This could include internet search results from Google, or an Azure AI Search Index that hosts your own data or documents.
- **Tools** - These are programmatic functions that enable the agent to automate actions. You can use built-in tools to access knowledge in Azure AI Search and Bing, as well as a code interpreter tool that you can use to generate and run Python code. You can also create custom tools using your own code or Azure Functions.

Conversations between uses and agents take place on a thread, which retains a history of messages exchanged in the conversation, as well as any data assets that are generated.

## Creating an Azure AI Foundy instance with Bicep

Before we can build Agents using Azure AI Foundry, we'll need to create some backing infrastructure for it. We can start with a simple Azure AI Foundry using API key authentication. For this, we'll create the following resources:

- Azure Storage Account
- Azure AI Services (along with the deploying the model that we'll use)
- Azure AI Hub
- Azure AI Project

I'll explain why we need these resources as we're going through the Bicep code.

### Storage Account

Storage accounts are used to store artifacts for your projects like flows and evaluations. Data isolations is ensured through storage containers, and are secured through Azure RBAC for the project identity.

We can create our Storage Account using the following Bicep:

```bicep
@description('The name of the Application')
param applicationName string

@description('The environment that this storage account will be deployed to')
param environmentName string

@description('The Azure region that this storage account will be deployed to. Default is "eastus2"')
@allowed([
  'eastus'
  'eastus2'
  'swedencentral'
  'westus'
  'westus3'
])
param location string = 'eastus2'

@description('The SKU applied to this storage account. Default value is "Standard_LRS"')
@allowed([
  'Standard_LRS'
  'Standard_ZRS'
  'Standard_GRS'
  'Standard_GZRS'
  'Standard_RAGRS'
  'Standard_RAGZRS'
  'Premium_LRS'
  'Premium_ZRS'
])
param storageSku string = 'Standard_LRS'

@description('The tags that will be applied to this storage account')
param tags object = {}

resource storage 'Microsoft.Storage/storageAccounts@2024-01-01' = {
  name: replace('${applicationName}${environmentName}stor','-','')
  location: location
  tags: tags
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    allowCrossTenantReplication: false
    allowSharedKeyAccess: true
    encryption: {
      keySource: 'Microsoft.Storage'
      requireInfrastructureEncryption: false
      services: {
        blob: {
          enabled: true
          keyType: 'Account'
        }
        file: {
          enabled: true
          keyType: 'Account'
        }
        queue: {
          enabled: true
          keyType: 'Service'
        }
        table: {
          enabled: true
          keyType: 'Service'
        }
      }
    }
    isHnsEnabled: false
    isNfsV3Enabled: false
    keyPolicy: {
      keyExpirationPeriodInDays: 7
    }
    largeFileSharesState: 'Disabled'
    minimumTlsVersion: 'TLS1_2'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
    supportsHttpsTrafficOnly: true
  }
}

@description('The resource Id of the deployed Storage Account')
output storageId string = storage.id
```

The two important things to note here is that we have a small allowed list for our location parameter. That's because at the time of writing, Agents are only available in those regions, so for simplicity, I'm going to deploy all the resources needed within those locations.

We're also providing the resource Id of the storage account as an output, as we'll need it for our AI Hub.

### Azure AI Services and Model Deployment

To use a LLM like OpenAI models, we'll need to create an Azure AI Service. 

Here you have different options. You can create an Azure OpenAI resource, which will give you access to OpenAI models, allow you to create deployments etc. or you can create an Azure AI Service, which gives you access to Azure Open AI, as well as other Azure AI Services.

We'll go for the later option, and then create a model deplooyment that uses the `gpt-4o-mini` Open AI model:

```bicep
@description('The name of the Application')
param applicationName string

@description('The environment that this AI Services account will be deployed to')
param environmentName string

@description('The Azure region that this AI Services account will be deployed to. Default is "eastus2"')
@allowed([
  'eastus'
  'eastus2'
  'swedencentral'
  'westus'
  'westus3'
])
param location string = 'eastus2'

@description('The name of the model that we will deploy. Default is "gpt-4o-mini"')
param modelName string = 'gpt-4o-mini'

@description('The tags that will be applied to this AI Services account')
param tags object = {}

var aiServicesName = '${applicationName}-${environmentName}-aiservice'

resource aiServices 'Microsoft.CognitiveServices/accounts@2025-04-01-preview' = {
  name: aiServicesName
  location: location
  tags: tags
  sku: {
    name: 'S0'
  }
  kind: 'AIServices'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    customSubDomainName: toLower(aiServicesName)
    publicNetworkAccess: 'Enabled'
  }
}

resource modelDeployment 'Microsoft.CognitiveServices/accounts/deployments@2025-04-01-preview' = {
  name: modelName
  parent: aiServices
  sku: {
    name: 'GlobalStandard'
    capacity: 50
  }
  properties: {
    model: {
      name: modelName
      version: '2024-07-18'
      format: 'OpenAI'
    }
  }
}

@description('The resource Id of the deployed Azure AI Service')
output aiServiceId string = aiServices.id

@description('The endpoint of the deployed AI Service')
output aiServiceEndpoint string = aiServices.properties.endpoint
```

Breaking this down, the AI Services resource consists of two main components:

1. **Azure AI Service Account**
   - Creates the base service using the S0 SKU tier
   - Enables system-assigned managed identity for secure authentication
   - Creates a custom subdomain for API access
   - Public network access is enabled (can be restricted for production)

2. **Model Deployment**
   - Deploys the default `gpt-4o-mini` model
   - Uses GlobalStandard SKU with 50 capacity units
   - Configures the model version (2024-07-18)
   - Sets OpenAI as the model format
   - Links to the parent AI Services account through the `parent` property

The code exports two critical values:
- `aiServiceId`: Resource ID needed by other Azure services
- `aiServiceEndpoint`: The endpoint URL for API calls

Again, we are limiting the location to AI Agent-supported regions: `eastus`, `eastus2`, `swedencentral`, `westus`, `westus3`.

This infrastructure provides the foundation for our AI Agent's language processing capabilities and will be referenced by our AI Hub in the next section.

### Azure AI Hub

Now let's define our template for our Azure AI Hub. Hubs are the top-level Azure resource for Azure AI Foundry and provide a centralized way for teams to govern AI resources across playgrounds and projects. Once a hub is created, developers can create projects from it and access resources.

We can define our Bicep template for our Azure AI Hub like so:

```bicep
@description('The name of the Application')
param applicationName string

@description('The environment that this AI Hub account will be deployed to')
param environmentName string

@description('The Azure region that this AI Hub account will be deployed to. Default is "eastus2"')
@allowed([
  'eastus'
  'eastus2'
  'swedencentral'
  'westus'
  'westus3'
])
param location string = 'eastus2'

@description('The tags that will be applied to this AI Hub account')
param tags object = {}

@description('Resource ID of the storage account resource for storing experimentation outputs')
param storageAccountId string

@description('Resource ID of the AI Services resource')
param aiServicesId string

@description('Resource ID of the AI Services endpoint')
param aiServicesTarget string

var aiHubName = '${applicationName}-${environmentName}-hub'

resource aiHub 'Microsoft.MachineLearningServices/workspaces@2025-01-01-preview' = {
  name: aiHubName
  location: location
  tags: tags
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: aiHubName
    description: 'This is a basic AI Foundry Hub'
    storageAccount: storageAccountId
  }
  kind: 'hub'

  resource aiServicesConnection 'connections@2025-01-01-preview' = {
    name: '${aiHubName}-connection-AIServices'
    properties: {
      category: 'AIServices'
      target: aiServicesTarget
      authType: 'ApiKey'
      isSharedToAll: true
      credentials: {
        key: '${listKeys(aiServicesId, '2022-10-01').key1}'
      }
      metadata: {
        ApiType: 'Azure'
        ResourceId: aiServicesId
        Location: location
      }
    }
  }
}

@description('The resource Id of the deployed AI Hub')
output aiHubId string = aiHub.id
```

Breaking this down, the AI Hub template has three important functions:

1. **Central Management** - The hub acts as the central point for all AI resources:
   - Creates a Machine Learning workspace with `kind: 'hub'` designation
   - Uses system-assigned managed identity for secure access
   - Links to our storage account for artifact storage

2. **AI Services Connection** - The hub creates a direct link to our AI models:
   - Creates a nested connection using `aiServicesConnection` 
   - Uses API key authentication for simplicity
   - Enables sharing models across all projects with `isSharedToAll: true`
   - Securely retrieves the API key using `listKeys()` function
   - Sets up proper metadata for Azure integration

3. **Resource References** - The hub links to our previously created resources:
   - References the storage account for storing agent artifacts
   - Links to our AI Services for model access
   - Points to the AI Services endpoint for API communication

As with our other resources, we're limiting deployment to regions that support AI Agents: `eastus`, `eastus2`, `swedencentral`, `westus`, `westus3`.

The final output provides the AI Hub's resource ID, which we'll need when creating our AI Project in the next section.

### Azure AI Project

Project workspaces created in Azure AI Hubs inherit the same security settings and shared resource access. Teams can create workspaces as needed to organize work, isolate data, and/or restrict access.

We can create projects using the following Bicep template:

```bicep
@description('The name of the Application')
param applicationName string

@description('The environment that this AI Project account will be deployed to')
param environmentName string

@description('The Azure region that this AI Project account will be deployed to. Default is "eastus2"')
@allowed([
  'eastus'
  'eastus2'
  'swedencentral'
  'westus'
  'westus3'
])
param location string = 'eastus2'

@description('Resource ID of the AI Hub resource')
param aiHubId string

@description('The tags that will be applied to this AI Project account')
param tags object = {}

var projectName = '${applicationName}-${environmentName}-project'
var subscriptionId = subscription().subscriptionId
var resourceGroupName = resourceGroup().name
var projectConnectionString = '${location}.api.azureml.ms;${subscriptionId};${resourceGroupName};${projectName}'

resource aiProject 'Microsoft.MachineLearningServices/workspaces@2025-01-01-preview' = {
  name: projectName
  tags: union(tags, {
    ProjectConnectionString: projectConnectionString
  })
  identity: {
    type: 'SystemAssigned'
  }
  location: location
  properties: {
    friendlyName: projectName
    description: 'This is a sample project.'
    hubResourceId: aiHubId
  }
  kind: 'Project'
}

@description('The resource Id of the deployed AI Project')
output aiProjectId string = aiProject.id

@description('The connection string of the deployed AI Project')
output aiProjectConnectionString string = aiProject.tags.ProjectConnectionString
```

The AI Project template has several important elements:

1. **Project Configuration**
   - Creates a Machine Learning workspace with `kind: 'Project'` designation
   - Links to our AI Hub through the `hubResourceId` property
   - Uses system-assigned managed identity for secure access
   - Follows our naming convention with `{applicationName}-{environmentName}-project`

2. **Connection String**
   - Builds a connection string that applications will use to access this project
   - Embedded in the resource tags for easy retrieval
   - Includes subscription, resource group, project name, and region
   - Format: `{location}.api.azureml.ms;{subscriptionId};{resourceGroupName};{projectName}`

3. **Outputs**
   - Provides the project's resource ID for reference in other templates
   - Exports the connection string which is crucial for our Python applications
   - This connection string will be used in our agent code with `AZURE_AI_AGENT_PROJECT_CONNECTION_STRING`

Like all our other resources, we're restricting deployment to the regions that support AI Agents.

### Final Bicep file

The AI Project is the final piece that completes our infrastructure setup for developing Azure AI Agents, so let's implement all our modules into our `main.bicep` file:

```bicep
@description('The name of the Application')
param applicationName string

@description('The environment that this application will be deployed to')
param environmentName string

@description('The tags that will be applied to all resources')
param tags object

module storageAccount 'modules/storage-account.bicep' = {
  name: 'storage'
  params: {
    applicationName: applicationName
    environmentName: environmentName
    tags: tags
  }
}

module aiServices 'modules/ai-services.bicep' = {
  name: 'aiServices'
  params: {
    applicationName: applicationName 
    environmentName: environmentName
    tags: tags
  }
}

module aiHub 'modules/ai-hub.bicep' = {
  name: 'aiHub'
  params: {
    aiServicesId: aiServices.outputs.aiServiceId
    aiServicesTarget: aiServices.outputs.aiServiceEndpoint
    applicationName: applicationName
    environmentName: environmentName
    storageAccountId: storageAccount.outputs.storageId
    tags: tags
  }
}

module aiProject 'modules/ai-project.bicep' = {
  name: 'aiProject'
  params: {
    aiHubId: aiHub.outputs.aiHubId
    applicationName: applicationName 
    environmentName: environmentName
    tags: tags
  }
}
```

To supply our parameters with some required values, we'll use a JSON parameters file. At the time of writing, I was having a lot of trouble getting `bicepparam` files to work, so for now, JSON will do:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "applicationName": {
        "value": "<app-name>"
      },
      "environmentName": {
        "value": "<env-name>"
      }, 
      "tags": {
        "value": {
            "Owner": "<your-name>",
            "Application": "<app-name>"
        }
      }
    }
  }
```

To deploy our Bicep template, we can run the following command:

```bash
az deployment group create --resource-group <your-rg-name> --template-file main.bicep --parameters main.parameters.json
```

Once your Bicep template has been deployed, we can now start to build our agent.

## Building our Agent

Now that we have our Azure AI Foundry infrastructure set up, let's create an Agent with Python that processes expense data. We'll need to create an `.env` file that loads up some configuration that we need to connect to our Azure AI Foundry:

```text
AZURE_AI_AGENT_PROJECT_CONNECTION_STRING="<your-ai-project-connection-string>"
AZURE_AI_AGENT_MODEL_DEPLOYMENT_NAME="gpt-4o-mini"
```

To get our connection string, we can navigate to our Azure AI Foundry Project and under **Project details**, copy the value for **Project connection string**:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gcwj0p4vm6dvi05jdd8q.png)

For our expenses data, we'll just use a simple `.txt` file to store some expenses, expense amount, and date.

```text
Category,Cost
Accommodation, 674.56
Transportation, 2301.00
Meals, 267.89
Misc., 34.50
```

We'll also need install some pip packages so that we can work with the Azure AI Agent SDK to build our agents. In a `requirements.txt` file, add the following packages:

```text
azure-identity>=1.13.0
azure-ai-projects>=1.0.0
python-dotenv>=1.0.0
```

We can then create a virtual environment and install the packages using the following commands:

```bash
# Create a virtual environment
python -m venv venv

# Activate the virtual environment
# For Windows:
venv\Scripts\activate

# Install packages from requirements.txt
pip install -r requirements.txt
```

This will create a new virtual environment, activate it, and then install all the packages listed in our `requirements.txt` file into the virtual environment.

Now we can create a new `main.py` file and define the imports we need to start building our agents. Let's import the libraries that we need by writing the following:

```python
import os
from dotenv import load_dotenv
from typing import Any
from pathlib import Path
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FilePurpose, CodeInterpreterTool
```

Now let's create our Agent in our `main` function:

```python
def main():
    # clear the console
    os.system('cls' if os.name =='nt' else 'clear')

    # load environment variables from .env file
    load_dotenv()
    PROJECT_CONNECTION_STRING=os.getenv("AZURE_AI_AGENT_PROJECT_CONNECTION_STRING")
    MODEL_DEPLOYMENT=os.getenv("AZURE_AI_AGENT_MODEL_DEPLOYMENT_NAME")

    # Display the data to be analyzed
    script_dir = Path(__file__).parent
    file_path = script_dir / 'data.txt'

    with file_path.open('r') as file:
        data = file.read() + "\n"
        print(data)

    # Connect to the Azure AI Foundry project
    project_client = AIProjectClient.from_connection_string(
        credential=DefaultAzureCredential(exclude_environment_credential=True,exclude_managed_identity_credential=True),
        conn_str=PROJECT_CONNECTION_STRING
    )

    with project_client:
        # Upload the data file and create a CodeInterpreterTool
        file=project_client.agents.upload_file_and_poll(file_path=file_path, purpose=FilePurpose.AGENTS)
        print(f"Uploaded {file.filename}")

        code_interpreter = CodeInterpreterTool(file_ids=[file.id])

        # Define an agent that uses the CodeInterpreterTool
        agent = project_client.agents.create_agent(
            model=MODEL_DEPLOYMENT,
            name="data-agent",
            instructions="You are an AI Agent that analyses the data in the file that has been uploaded. If the user requests a chart, create it and save it as a .png file.",
            tools=code_interpreter.definitions,
            tool_resources=code_interpreter.resources,
        )
        print(f"Using agent: {agent.name}")

        # Create a thread for the conversation
        thread = project_client.agents.create_thread()

        # Loop until the user types 'quit'
        while True:
            # Get input text
            user_prompt = input("Enter a prompt (or type 'quit' to exit):")
            if user_prompt.lower() == "quit":
                break
            if len(user_prompt) == 0:
                print("Please enter a prompt")
                continue

            # Send a prompt to the agent
            message = project_client.agents.create_message(
                thread_id=thread.id,
                role="user",
                content=user_prompt
            )

            run = project_client.agents.create_and_process_run(thread_id=thread.id, agent_id=agent.id)

            # Check the run status for failures
            if run.status == "failed":
                print(f"Run failed: {run.last_error}")

            # Show the lastest response from the agent
            messages = project_client.agents.list_messages(thread_id=thread.id)
            last_msg = messages.get_last_text_message_by_role("assistant")
            if last_msg:
                print(f"Last Message: {last_msg.text.value}")

        # Get the conversation history
        print("\nConversation Log:\n")
        messages = project_client.agents.list_messages(thread_id=thread.id)
        for message_data in reversed(messages.data):
            last_message_content = message_data.content[-1]
            print(f"{message_data.role}: {last_message_content.text.value}\n")
        
        # Get any generated files
        for file_path_annotation in messages.file_path_annotations:
            project_client.agents.save_file(file_id=file_path_annotation.file_path.file_id, file_name=Path(file_path_annotation.text).name)
            print(f"File saved as {Path(file_path_annotation.text).name}")

        # Clean up
        project_client.agents.delete_agent(agent.id)
        project_client.agents.delete_thread(thread.id)

# Define the entry point for our application
if __name__ == '__main__':
    main()
```

This code implements an interactive AI agent application using Azure AI Services, specifically the Azure AI Foundry project (part of Azure AI Studio). The agent is designed to analyze data from a text file and generate visualizations when requested.

The main function begins with environment setup - clearing the console and loading environment variables that contain the Azure connection information. It uses a best practice of storing sensitive configuration in environment variables rather than hardcoding them in the source code.

Next, it reads data from a file named `data.txt` located in the same directory as the script. This data will be analyzed by the AI agent based on user prompts.

The code then connects to Azure AI Foundry using the `AIProjectClient` with Azure DefaultAzureCredential, which follows the security best practice of using managed identities where possible. It explicitly excludes certain credential types to control the authentication flow more precisely.

Within the client session context, the code uploads the data file to Azure and configures a `CodeInterpreterTool` that gives the AI agent the ability to execute code against the data. It then creates an AI agent with specific instructions to analyze the data and generate charts when requested.

The interaction loop allows users to have a conversation with the agent, sending prompts and receiving responses. Each message is added to a conversation thread to maintain context. The agent processes each prompt using Azure's AI capabilities and returns responses, which are displayed to the user.

After the conversation concludes, the code displays the full conversation history and saves any files generated by the agent (such as charts or visualizations) to the local system. Finally, it cleans up resources by deleting the agent and thread to prevent unnecessary resource consumption.

Now that we have defined our agent, let's run it and see how it works. Run the following command:

```bash
python main.py
```

Now ask our agent the following:

```bash
Create a pie chart showing cost by category
```

Our Agent will go ahead and create a pie chart for our expenses by category, we should get the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6gv6pkb5ggvtnaihyw8f.png)

## Conclusion

Wow, this was a long one! If you're still with me, hopefully you have a better understanding of how AI Agents work, and how we can use Azure AI Foundry to build agents.

I'm working on a variety of samples for building agents with Azure AI Agent service on GitHub. The sample code for this specific article can be found [here](https://github.com/willvelida/azure-ai-agent-samples/tree/main/code-samples/Python/basic-agent)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! I'm loving BlueSky at the moment. It has a much better UX and performance than Twitter these days.

Until next time, Happy coding! 🤓🖥️