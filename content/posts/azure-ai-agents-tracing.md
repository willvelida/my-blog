---
title: "How Tracing works in Azure AI Foundry Agents"
date: 2025-06-04
draft: false
tags: ["Azure", "AI Agents", "AI", "GenAI", "LLMs", "Observability"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ekfb9hlyw60najn2vu1e.png
    alt: "How Tracing works in Azure AI Foundry Agents"
    caption: "Tracing in Azure AI Foundry Agents allow us to see how agents proccess inputs and outputs within agents runs, as well as the tools they may have invoked, and the order which they were invoked, helping us understand the reasoning behind an agent's execution."
---

Determining how Azure AI Foundry Agents makes decisions is important for troubleshooting and debugging purposes. However, it can get a little complicated when our agents perform complex workflows. Our agents could perform numerous executions, making it difficult to track decisions made by all or them, or some agents may invoke tools, that invoke other tools, which invoke more tools! (And so on and so forth).

Tracing our agents helps us see the inputs and outputs involved in a particular agent run, as well as the order in which those agents were invoked. In this blog post, I'll talk about how tracing agents works, how we can do some simple tracing using the Azure AI Foundry Agents playground, and how we can implement tracing in our pro-code agents using OpenTelemetry

## Tracing in the Agents Playground

We can do some simple tracing in the AI Foundry portal that lets us trace threads and runs that our agents produce. Take a simple weather agent, that has the following instructions:

```text
You are a helpful weather agent that provides information abut the weather. Ensure you ask what units of measurement the user wants the weather in (C or F) and the location where they want the weather. If someone asks you a non related weather question, respond "I don't know"
```

In the playground, we can ask it some simple questions about the weather, and it'll respond with some weather information (nothing too interesting here). What we're interested in is viewing the traces and metrics that our agent has produced, so we can see how our agent has performed in terms of **AI quality** and **Risk and safety**.

We can dive into this information by clicking either **Metrics** or **Thread info**. Let's take a look at **Thread info** first:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g1877080eoftft8x0n6m.png)

When we drill down into **Thread info**, we can see some infromation about the thread, it's run and the steps it took, any tool calls that were made etc. We can also see the our inputs and the agents output. Here's an example:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aswzz4909nwp16avn0ej.png)

We can also take a look at any associated metadata from our thread. Highlighted is some information about the number of tokens our underlying LLM used during this run:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4hbv6g0edx7tv4ghpf8r.png)

Finally, we can view information on any evaluators that we have selected for our agent. For example, in this particular run, we can see that the agent didn't do that well when responding to our question on asking about the weather, due to the limitations of the underlying model:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kpysbngve71uj4q1razv.png)

## Tracing agents with OpenTelemetry

This is good for starters, but we can also trace our agents performace and behavior using OpenTelemetry and connecting an Application Insights resources to Azure AI Foundry.

We can do this in the AI Foundry portal by selecting **Observability** from the left pane, select **Tracing** and then connecting an existing App Insights resource, or creating a new one:

![](https://learn.microsoft.com/en-us/azure/ai-services/agents/media/ai-foundry-observability.png#lightbox)

We can also do this in Bicep by providing the resource ID of an Application Insights resource to our Azure AI Foundry Hub resource:

```bicep
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
    applicationInsights: appInsightsId
    systemDatastoresAuthMode: 'Identity'
  }
  kind: 'hub'

  resource aiServicesConnection 'connections@2025-01-01-preview' = {
    name: '${aiHubName}-connection-AIServices'
    properties: {
      category: 'AIServices'
      target: aiServicesTarget
      authType: 'AAD'
      isSharedToAll: true
      metadata: {
        ApiType: 'Azure'
        ResourceId: aiServicesId
        Location: location
      }
    }
  }
}
```

Once we've connected an App Insights resource to our AI Foundry, we can install the packages we need to start tracing our agents. Use ```pip install``` to install the following packages:

```bash
pip install opentelemetry-sdk
pip install azure-core-tracing-opentelemetry
pip install opentelemetry-exporter-otlp
```

The ```opentelemetry-exporter-otlp``` package will provide you an exporter to send results to our observability backend.

Now let's take a simple example that sends traces to Azure Monitor. **If you haven't used pip install for the above packages**, list the following packages in a ```requirements.txt``` file:

```text
azure-ai-projects
azure-identity
dotenv
opentelemetry-sdk
azure-core-tracing-opentelemetry
opentelemetry-exporter-otlp
azure-monitor-opentelemetry
```

Now create a virtual environment for your Python Agent using the following commands (Powershell):

```powershell
# Create a virtual environment
python -m venv .venv

# Activate the virtual environment (Windows)
.venv\Scripts\activate
```

Once our virtual environment has been created, we can create a basic agent that uses a tool to retrieve weather information. Here's the full code (we'll break this down):

```python
from typing import Any, Callable, Set

import os, time, json
from azure.ai.agents import AgentsClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import (
    FunctionTool,
    ToolSet,
    ListSortOrder,
)
from opentelemetry import trace
from azure.monitor.opentelemetry import configure_azure_monitor
from azure.ai.agents.telemetry import trace_function
from dotenv import load_dotenv

load_dotenv()

agents_client = AgentsClient(
    endpoint=os.getenv("PROJECT_ENDPOINT"),
    credential=DefaultAzureCredential(),
)

# Enable Azure Monitor tracing
application_insights_connection_string = os.getenv("APPLICATIONINSIGHTS_CONNECTION_STRING")
configure_azure_monitor(connection_string=application_insights_connection_string)

scenario = os.path.basename(__file__)
tracer = trace.get_tracer(__name__)


# The trace_func decorator will trace the function call and enable adding additional attributes
# to the span in the function implementation. Note that this will trace the function parameters and their values.
@trace_function()
def fetch_weather(location: str) -> str:
    """
    Fetches the weather information for the specified location.

    :param location (str): The location to fetch weather for.
    :return: Weather information as a JSON string.
    :rtype: str
    """
    # In a real-world scenario, you'd integrate with a weather API.
    # Here, we'll mock the response.
    mock_weather_data = {"New York": "Sunny, 25°C", "London": "Cloudy, 18°C", "Tokyo": "Rainy, 22°C"}

    # Adding attributes to the current span
    span = trace.get_current_span()
    span.set_attribute("requested_location", location)

    weather = mock_weather_data.get(location, "Weather data not available for this location.")
    weather_json = json.dumps({"weather": weather})
    return weather_json


# Statically defined user functions for fast reference
user_functions: Set[Callable[..., Any]] = {
    fetch_weather,
}

# Initialize function tool with user function
functions = FunctionTool(functions=user_functions)
toolset = ToolSet()
toolset.add(functions)

# To enable tool calls executed automatically
agents_client.enable_auto_function_calls(toolset)

with tracer.start_as_current_span(scenario):
    with agents_client:
        # Create an agent and run user's request with function calls
        agent = agents_client.create_agent(
            model=os.getenv("MODEL_DEPLOYMENT_NAME"),
            name="my-agent",
            instructions="You are a helpful agent",
            toolset=toolset,
        )
        print(f"Created agent, ID: {agent.id}")

        thread = agents_client.threads.create()
        print(f"Created thread, ID: {thread.id}")

        message = agents_client.messages.create(
            thread_id=thread.id,
            role="user",
            content="Hello, what is the weather in New York?",
        )
        print(f"Created message, ID: {message.id}")

        run = agents_client.runs.create_and_process(thread_id=thread.id, agent_id=agent.id, toolset=toolset)
        print(f"Run completed with status: {run.status}")

        # Delete the agent when done
        agents_client.delete_agent(agent.id)
        print("Deleted agent")

        # Fetch and log all messages
        messages = agents_client.messages.list(thread_id=thread.id, order=ListSortOrder.ASCENDING)
        for msg in messages:
            if msg.text_messages:
                last_text = msg.text_messages[-1]
                print(f"{msg.role}: {last_text.text.value}")
```

Let's break this down:

```python
load_dotenv()

agents_client = AgentsClient(
    endpoint=os.getenv("PROJECT_ENDPOINT"),
    credential=DefaultAzureCredential(),
)

# Enable Azure Monitor tracing
application_insights_connection_string = os.getenv("APPLICATIONINSIGHTS_CONNECTION_STRING")
configure_azure_monitor(connection_string=application_insights_connection_string)

scenario = os.path.basename(__file__)
tracer = trace.get_tracer(__name__)
```

After loading our environment variables with ```load_dotenv()```, we initialize an ```AgentsClient``` which serves as the main interface to the Azure AI Agent service. We supply the AI Foundry project endpoint, which should look something like this:

```text
https://<AIFoundryResourceName>.services.ai.azure.com/api/projects/<ProjectName>
```

We also provide a ```DefaultAzureCredential``` so that we can use managed identities to authenticate our ```AgentsClient```. Make sure you have the ```Azure AI User``` RBAC role within Azure AI Foundry and then login using the ```az login``` command.

We then configure Azure Monitor by calling the ```configure_azure_monitor``` method with the connection string from our Application Insights resource. This will send telemetry data to App Insights from our Agent.

Let's move onto the tool that our Agent will use:

```python
@trace_function()
def fetch_weather(location: str) -> str:
    """
    Fetches the weather information for the specified location.

    :param location (str): The location to fetch weather for.
    :return: Weather information as a JSON string.
    :rtype: str
    """
    # In a real-world scenario, you'd integrate with a weather API.
    # Here, we'll mock the response.
    mock_weather_data = {"New York": "Sunny, 25°C", "London": "Cloudy, 18°C", "Tokyo": "Rainy, 22°C"}

    # Adding attributes to the current span
    span = trace.get_current_span()
    span.set_attribute("requested_location", location)

    weather = mock_weather_data.get(location, "Weather data not available for this location.")
    weather_json = json.dumps({"weather": weather})
    return weather_json
```

This is just a basic tool that returns hardcoded weather data, but *within* the tool, we access the current trace span using OpenTelemetry's ```trace.get_current_span()``` method, and then add a custom attribute to the span using the ```set_attribute()``` method.

We then configure our Agent to use the custom tools in the following code:

```python
# Statically defined user functions for fast reference
user_functions: Set[Callable[..., Any]] = {
    fetch_weather,
}

# Initialize function tool with user function
functions = FunctionTool(functions=user_functions)
toolset = ToolSet()
toolset.add(functions)

# To enable tool calls executed automatically
agents_client.enable_auto_function_calls(toolset)
```

Now let's dive into the meat of our Agent code (along with the tracing capabilities):

```python
with tracer.start_as_current_span(scenario):
    with agents_client:
        # Create an agent and run user's request with function calls
        agent = agents_client.create_agent(
            model=os.getenv("MODEL_DEPLOYMENT_NAME"),
            name="my-agent",
            instructions="You are a helpful agent",
            toolset=toolset,
        )
        print(f"Created agent, ID: {agent.id}")

        thread = agents_client.threads.create()
        print(f"Created thread, ID: {thread.id}")

        message = agents_client.messages.create(
            thread_id=thread.id,
            role="user",
            content="Hello, what is the weather in New York?",
        )
        print(f"Created message, ID: {message.id}")

        run = agents_client.runs.create_and_process(thread_id=thread.id, agent_id=agent.id, toolset=toolset)
        print(f"Run completed with status: {run.status}")

        # Delete the agent when done
        agents_client.delete_agent(agent.id)
        print("Deleted agent")

        # Fetch and log all messages
        messages = agents_client.messages.list(thread_id=thread.id, order=ListSortOrder.ASCENDING)
        for msg in messages:
            if msg.text_messages:
                last_text = msg.text_messages[-1]
                print(f"{msg.role}: {last_text.text.value}")
```

We wrap our Agent code within a tracing context using ```tracer.start_as_current_span(scenario)```, which creates a span in Azure Monitor for the entire lifecycle of the agent. Any operations that are performed as part of the agent workflow are captured within a single trace.

Within this context, we create our agent using the ```agents_client.create_agent()``` passing through our LLM (in my case, using gpt-4.1), the name of our agent, instructions, and the weather toolset that we configured earlier.

We then create a conversation thread for our Agent using ```agents_client.threads.create()``` and ask it a question about the weather in New York in the ```agents_client.messages.create()``` passing through the thread ID. We then process the thread using ```agents_client.runs.create_and_process()```, which triggers the agent to analyze the message and then use the weather tool that we define should it need to. We then delete the agent using the ``agents_client.delete_agent(agent.id)``. We then retrieve the entire conversation thread using ``agents_client.messages.list()``.

Throughout the lifetime of the agent, we track each operation within the OpenTelemetry trace context, and use the console output to confirm whether or not those operations were successful.

Running the agent a couple of times, we can verify that output is being sent to the console like so:

```bash
Created agent, ID: asst_i2rlh7jxtXk55vlom9Gh7bID
Created thread, ID: thread_hpBAF1Dg6TgKIhaUl9WMZt6O
Created message, ID: msg_Fl6Cn84OT4g6ZOGsJSnHHL3M
Run completed with status: RunStatus.COMPLETED
Deleted agent
MessageRole.USER: Hello, what is the weather in New York?
MessageRole.AGENT: The current weather in New York is sunny with a temperature of 25°C.
```

We can also start to see traces being sent to the Azure Monitor in the Azure AI Foundry portal under **Tracing**. Here we can see the operations that our Agent took, along with the weather tool that it invoked as part of the operation:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/88yt7wx8wzj5c4chn1um.png)

We can drill down into each operation of a particular run to see additional metadata:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/boooz93f1wo42vf5q8ct.png)

# Conclusion

Hopefully I've given you a bit of insight into how we can use tracing within your Azure AI Agents.

One thing to note here is that at the time of writing, there is a bug in the tracing functionality that will cause the agent's function tool to call related information (which may include PII information) to be included in the traces even when content recording is not enabled.

If you want to reac more about Observability in Azure AI Foundry Agent Service, check out the following documentation:

- [Trace and observe agents](https://learn.microsoft.com/en-us/azure/ai-services/agents/concepts/tracing)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! I'm loving BlueSky at the moment. It has a much better UX and performance than Twitter these days.

Until next time, Happy coding! 🤓🖥️