---
title: "Working with Queues and Topics in Azure Service Bus"
date: 2022-04-18T10:15:15+12:00
draft: true
tags: ["Azure","Azure Service Bus","C#", "Bicep", "Messaging"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/brunyulntyeh5xg3md5m.png
    alt: "Queues and Topics in Azure Service Bus"
    caption: 'In Azure Service, we can send messages to queues and topics, depending on how many consumers of the message we need.'
---

Azure Service Bus is a message broker that we can use to send messages to queues or publish messagees to topics so that consumers can subscribe to those topics to receive those messages. In the article, I'll explain what the differences are between queues and topics in Azure Service Bus, how we can provision Service Bus namespaces with either queues or topics using Bicep and then I'll show you how we can send and receives messages from our queue or topic.

I've created a couple of samples that you can refer to as you read through this post:

| Sample | GitHub Link | Deploy me! |
| ------ |-------- | -------------- |
| **Queues** | [Code]((https://github.com/willvelida/azure-samples/tree/main/service-bus-queues)) | [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fwillvelida%2Fazure-samples%2Fmain%2Fservice-bus-queues%2Fdeploy%2Fazuredeploy.json) |
| **Topics** | [Code](https://github.com/willvelida/azure-samples/tree/main/service-bus-topics) | [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fwillvelida%2Fazure-samples%2Fmain%2Fservice-bus-topics%2Fdeploy%2Fazuredeploy.json) |

## What are Queues?

Queues work on a **First In, First Out** (FIFO) basis. This means that clients that receive messages from the queue and then process that message in the order in which they were added to the queue, and they will be the only consumer that processes this message. The queue will store this message until our client is able to process them. To process the message, the client will pull the message off the queue.

One of the benefits of using queues is that the producers and consumer of the queue don't have to send and receive the message at the same time. The message will be stored in the queue and will only be processed once the consumer pulls the message off the queue. Producers can keep sending messages to the queue.

This functionality allows us to decouple components within our architecutre, since producers and consumers don't need to be concerned with what the other one is doing. If we have a high throughput of messages coming into the queue, we can scale our consumers without having to scale our producers.

Queues aren't unique to Azure Service Bus. In Azure, we can also use Storage Queues, but there is a difference between the two options. Check out this [article](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted) to see when you should use Service Bus or Storage Queues.

### Creating a Service Bus Namespace with a Queue in Bicep

Creating a basic queue in Azure Service Bus only needs a few lines of code:

```bicep
@description('The location where we will deploy our resources to. Default is the location of the resource group')
param location string = resourceGroup().location

@description('Name of our application.')
param applicationName string = uniqueString(resourceGroup().id)

var serviceBusName = 'sb${applicationName}'
var queueName = 'messages'

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2021-11-01' = {
  name: serviceBusName
  location: location
  sku: {
    name: 'Basic'
  }
}

resource serviceBusQueue 'Microsoft.ServiceBus/namespaces/queues@2021-11-01' = {
  name: queueName
  parent: serviceBusNamespace
}
```

A Service Bus Namespace is a container for all messaging components. Within our namespace, we can have both queues ad topics (depending on the SKU that we use for our Service Bus). You can think of a Service Bus Namespace as a server with its own capacity of a large cluster of all-active VMs. This makes Service Bus an available and reliable service at scale, without us needing to manage the service.

We can provision a Service Bus namespace by declaring a resource of type ```Microsoft.ServiceBus/namespaces``` and give it a name, location and SKU. For this queue snippet, we use the Basic SKU.

To create our queue, we declare a resource block of type ```Microsoft.ServiceBus/namespaces/queues```, give it a name and use our Service Bus namespace as the parent, which will let Bicep know that this queue belongs in this namespace.

Check out this [reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.servicebus/namespaces/queues?tabs=bicep) for the full ARM reference API for Azure Service Bus Queues.

### Sending and Retrieving Messages from the Queue in C#

Now that we have our namespace and our queue, we can start to send messages to the queue. We can achieve this using the following code:

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Configuration;

IConfiguration configuration = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json")
    .Build();

ServiceBusClient serviceBusClient = new ServiceBusClient(configuration["connectionString"]);
ServiceBusSender sender = serviceBusClient.CreateSender(configuration["queueName"]);

int numberOfMessages = 3;

using (ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync())
{
    for (int i = 1; i < numberOfMessages; i++)
    {
        if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
        {
            throw new Exception($"The message {i} is too large to fit in the queue");
        }
    }

    try
    {
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of {numberOfMessages} messages have been sent to the queue");
    }
    finally
    {
        await sender.DisposeAsync();
        await serviceBusClient.DisposeAsync();
    }
}

Console.WriteLine("Press any key to end the application");
Console.ReadKey();
```

Let's go through the important peices of this code:

- To work with the Service Bus SDK in C#, we can install the ```Azure.Messaging.ServiceBus``` NuGet package. I've also installed ```Microsoft.Extensions.Configuration``` so I can use an ```appsettings.json``` file for my application settings, which I load in my ```IConfiguration``` object.
- We can then create a ```ServiceBusClient``` object passing in the connection string of our Service Bus, then we create a ```ServiceBusSender``` object for our queue.
- We then create a ```ServiceBusMessageBatch``` object and add messages to that batch using the ```TryAddMessage()``` method. Once we've added all our messages to the batch, we send them to our queue using the ```SendMessagesAsync()``` method.
- Once we've sent our messages, we clean up our resources using ```DisposeAsync()``` on both our ```ServiceBusSender``` and ```ServiceBusClient``` objects.

Receiving a message from the queue:

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Configuration;

IConfiguration configuration = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json")
    .Build();

ServiceBusClient serviceBusClient = new ServiceBusClient(configuration["connectionString"]);
ServiceBusProcessor processor = serviceBusClient.CreateProcessor(configuration["queueName"], new ServiceBusProcessorOptions());

try
{
    processor.ProcessMessageAsync += MessageHandler;
    processor.ProcessErrorAsync += ErrorHandler;

    await processor.StartProcessingAsync();

    Console.WriteLine("Wait for a minute and then press any key to end the processing");
    Console.ReadKey();

    Console.WriteLine("\nStopping the receiver");
    await processor.StopProcessingAsync();
    Console.WriteLine("Stopped receiving messages");
}
finally
{
    await processor.DisposeAsync();
    await serviceBusClient.DisposeAsync();
}

async Task MessageHandler(ProcessMessageEventArgs args)
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body}");

    await args.CompleteMessageAsync(args.Message);
}

Task ErrorHandler(ProcessErrorEventArgs args)
{
    Console.WriteLine(args.Exception.ToString());
    return Task.CompletedTask;
}
```

## What are Topics?

Topics are different to Queues since instead of working with a single consumer, we can have multiple **subscribers** to our topic, who will receive their own copy of the message from the topic. This works in a pub/sub pattern, where we will have messages being published to the topic and have multiple clients subscribe to that topic. 

In Topics, our consumers don't directly consume the message from our Topic. Instead, we create subscriptions that subscribe to the topic and our consumers receive a copy of a message from the topic. In Azure Service Bus, we can define filters on these subscriptions that determine conditions for messages to be published to a subscription and actions that modifies the message metadata.

### Create a namespace with a Topic and Subscriber in Bicep

Again, creating our basic topic and subscriber only needs a few lines of Bicep code:

```bicep
@description('The location where we will deploy our resources to. Default is the location of the resource group')
param location string = resourceGroup().location

@description('Name of our application.')
param applicationName string = uniqueString(resourceGroup().id)

var serviceBusName = 'sb${applicationName}'
var topicName = 'messages'
var subscriptionName = 'messagesub'

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2021-11-01' = {
  name: serviceBusName
  location: location
  sku: {
    name: 'Standard'
  }
}

resource topic 'Microsoft.ServiceBus/namespaces/topics@2021-11-01' = {
  name: topicName
  parent: serviceBusNamespace
}

resource subscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2021-11-01' = {
  name: subscriptionName
  parent: topic
}
```

We create our Service Bus Namespace, but this time we provision our namespace using the Standard SKU. Topics aren't supported at the Basic pricing tier, so if you're using Topics in your application, you'll need to use either Standard or Premium pricing.

Much like our queue from before, we create a Topic resource using the type ```Microsoft.ServiceBus/namespaces/topics@2021-11-01``` using our Service Bus namespace as the parent resource. We then also create a Subscription resource using the type ```Microsoft.ServiceBus/namespaces/topics/subscriptions@2021-11-01``` using our Topic resource as the parent.

To see the full ARM Reference API for Azure Service Bus Topics, check out this [reference guide](https://docs.microsoft.com/en-us/azure/templates/microsoft.servicebus/namespaces/topics?tabs=bicep).

### Sending and Subscribing to Topics in C#

```csharp
ServiceBusSender sender = serviceBusClient.CreateSender(configuration["topicName"]);
```

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Configuration;

IConfiguration configuration = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json")
    .Build();

ServiceBusClient serviceBusClient = new ServiceBusClient(configuration["connectionString"]);
ServiceBusProcessor serviceBusProcessor = serviceBusClient.CreateProcessor(configuration["topicName"], configuration["subscriptionName"], new ServiceBusProcessorOptions());

try
{
    serviceBusProcessor.ProcessMessageAsync += MessageHandler;
    serviceBusProcessor.ProcessErrorAsync += ErrorHandler;

    await serviceBusProcessor.StartProcessingAsync();

    Console.WriteLine("Wait for a minute and then press any key to end the processor");
    Console.ReadKey();

    Console.WriteLine("\nStopping the receiver...");
    await serviceBusProcessor.StopProcessingAsync();
    Console.WriteLine("Stopped receiving messages");
}
finally
{
    await serviceBusProcessor.DisposeAsync();
    await serviceBusClient.DisposeAsync();
}

async Task MessageHandler(ProcessMessageEventArgs args)
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body} from the subscription {configuration["subscriptionName"]}");
    await args.CompleteMessageAsync(args.Message);
}

Task ErrorHandler(ProcessErrorEventArgs args)
{
    Console.WriteLine(args.Exception.ToString());
    return Task.CompletedTask;
}
```

## Conclusion

In this post, we went through the difference between Queues and Topics in Azure Service Bus, how we can provision Azure Service Bus namespaces and create Queues and Topics using Bicep code and how we can send and receive messages from both Queues and Topics in Service Bus using C#.

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️