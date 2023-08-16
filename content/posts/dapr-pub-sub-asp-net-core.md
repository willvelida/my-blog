---
title: "Implementing Dapr Pub/Sub functionality to ASP.NET Core Web APIs"
date: 2023-08-15
draft: false
tags: ["Dapr","Dotnet","ASP.NET Core", "Azure Service Bus", "Messaging"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ihtfh2rwbezmo90zri4w.png
    alt: "Implementing Dapr Pub/Sub in ASP.NET Core Web APIs"
    caption: "Dapr's Pub/Sub API simplifies the implementation of pub/sub functionality in distributed architectures."
---

In event-driven architectures, communicating between different microservices via messages can be enabled using Publish and Subscribe functionality (Pub/Sub for short).

The **publisher** (or producer) writes messages to a message topic (or queue), while a **subscriber** will subscribe to that particular topic and receives the message. Both the publisher and subscriber are unaware of which application either produced or subscribed to the topic. This pattern is useful when we want to send messages to different services, while ensuring that they are decoupled from each other.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m1ctwnymytf08aloa1kh.png)

*image taken from https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/*

In this article, we'll discuss how Pub/Sub works in Dapr, and how we can implement Pub/Sub functionality in a ASP.NET Core Web API. We'll then configure our Pub/Sub component and wire it up to our API to see it in action.

## What is Pub/Sub in Dapr?

Dapr has a Pub/Sub API that you can use to implement Pub/Sub capabilities to your applications. It's platform agnostic, meaning that you can a variety of message brokers to send and receive messages from. We can configure our message broker as a Dapr Pub/Sub component at runtime, removing the dependency from our service and making it more flexible to changes (should you need, or even want to, change your message broker).

The API offers at-least-once message guarantees, meaning that Dapr ensures that the message will be delivered *at least once* to every subscriber. In the event that a message fails to deliver, or your application fails, Dapr will attempt to redeliver the message until it's successful.

When we use Pub/Sub in Dapr, our service (In our case, our publisher service), will make a network call to a Pub/Sub building block API. This building block will make a call to our Pub/Sub component (our configured message broker). To receive messages on a topic, Dapr subscribes to the Pub/Sub component with a topic and then delivers messages to a particular endpoint when messages arrive.

In this article, we'll build two APIs. One that will act as our publisher which will send messages to our message broker (In this example, I'll be using Azure Service Bus), and the other will act as our subscriber, which will receive messages from our Service Bus topic:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/np1ot4j861fyrxhnd4hf.png)

Dapr Pub/Sub has a variety of features available that simplifies Pub/Sub capabilities in your application, so I'd recommend taking a look at the [Dapr documentation on Pub/Sub](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/) if you want to gain a deeper understanding.

## Configuring Dapr in our Web API

To see Dapr Pub/Sub in action, we'll build two ASP.NET Core Web APIs: One to handle the publishing of an Bookshop order message, another to subscribe to the topic that our orders will be sent to so they can receive the message.

To see the full code for this project, please check out [this repository](https://github.com/willvelida/dapr-resources/tree/main/PubSub) on my GitHub.

To work with Dapr in an ASP.NET Core Web, we'll need to install the ```Dapr.AspNetCore``` package. For this tutorial, we'll need to install this package in both our API's. To do this, we can run the following NET CLI command in our Web API projects:

```bash
dotnet add package Dapr.AspNetCore
```

Alternatively, you can use the NuGet Package Manager in Visual Studio to install it.

This package will allow you to interact with Dapr applications through the Dapr Client and build routes and controllers in your ASP.NET applications using Dapr.

## Implementing our Pub/Sub logic

Now that the Dapr package has been installed in both our APIs, we can start to build our publisher API. For this application, we're going to keep our Order model simple like so:

```csharp
namespace Bookshop.Common
{
    public class Order
    {
        public string OrderId { get; set; }
        public string Title { get; set; }
        public string Author { get; set; }
        public decimal Price { get; set; }
    }
}
```

For our publisher, we can create the following controller:

```csharp
using Bookshop.Common;
using Dapr.Client;
using Microsoft.AspNetCore.Mvc;

namespace Bookshop.Publisher.Controllers
{
    [Route("api/orders")]
    [ApiController]
    public class OrderPublisherController : ControllerBase
    {
        private readonly DaprClient _daprClient;
        private readonly ILogger<OrderPublisherController> _logger;

        public OrderPublisherController(DaprClient daprClient, ILogger<OrderPublisherController> logger)
        {
            _daprClient = daprClient;
            _logger = logger;
        }

        [HttpPost]
        public async Task<IActionResult> CreateOrder([FromBody] Order order)
        {
            if (order is not null)
            {
                _logger.LogInformation($"Publishing order ID {order.OrderId}");

                await _daprClient.PublishEventAsync("dapr-pubsub", "orderstopic", order);

                return Ok(order);
            }

            return BadRequest();
        }
    }
}
```

Let's break this down.

We inject our ```DaprClient``` into the controller since we'll be using it to publish our event. In this controller, we have a single method called ```CreateOrder``` which will be invoked when we make a POST request. We'll need to pass an Order request payload to the endpoint, which will invoke the ```PublishEventAsync``` method.

In this method, we pass three parameters:

- ```dapr-pubsub``` - This will be the name of the Pub/Sub component that we will define later.
- ```orderstopic``` - This is the name of our topic that we will send Order messages to.
- ```order``` - This is the payload that we'll send to our ```orderstopic```.

In our Bookshop.Publisher ```Program.cs``` file, we need to add the DaprClient service to our service container like so:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddDaprClient();
builder.Services.AddControllers();

// Rest of file
```

To receive messages from our ```orderstopic```, we can create a controller that will act as our subscriber:

```csharp
using Bookshop.Common;
using Microsoft.AspNetCore.Mvc;

namespace Bookshop.Subscriber.Controllers
{
    [Route("api/orders")]
    [ApiController]
    public class OrderSubscriberController : ControllerBase
    {
        private readonly ILogger<OrderSubscriberController> _logger;

        public OrderSubscriberController(ILogger<OrderSubscriberController> logger)
        {
            _logger = logger;
        }

        [Dapr.Topic("dapr-pubsub", "orderstopic")]
        [HttpPost("orderreceived")]
        public async Task<IActionResult> OrderReceived([FromBody] Order order)
        {
            if (order is not null)
            {
                _logger.LogInformation($"Received Order ID: {order.OrderId}: {order.Title} - {order.Author} - {order.Price}");
                return Ok();
            }

            return BadRequest();
        }
    }
}
```

Again, let's break this down.

We don't need to inject a ```DaprClient``` into this controller. Instead, we annotate our ```OrderReceived``` method with the ```Dapr.Topic``` attribute. This attribute accepts two arguments:

1. The name of our Pub/Sub component that this subscriber will target. In this case, our component will be called ```dapr-pub```, which we will define later.
2. The name of the topic that to subscribe to, which is called ```orderstopic```.

The action method will expect to receive an ```Order``` object and once the message is received, we will log out the order that has been received and return a ```200``` response.

In our subscriber ```Program.cs``` file, we need to add the following:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers().AddDapr();

var app = builder.Build();

// Configure the HTTP request pipeline.
app.UseAuthorization();

app.UseCloudEvents();

app.MapControllers();

app.MapSubscribeHandler();

app.Run();
```

Let's break this down,

- We integrate Dapr into the MVC pipeline in the line ```builder.Services.AddControllers().AddDapr()```.
- On line ```app.UseCloudEvents()```, this will add CloudEvents middleware into the ASP.NET Core middleware pipeline. This unwraps requests that use the CloudEvents structured format, so the receiving method can read the event payload that is sent directly.
- On line ```app.MapSubscribeHandler()```, this will make the endpoint ```http://localhost:<your-app-port>/dapr/subscribe``` available for the consumer so it responses and returns the available subscriptions. When the endpoint is called, it finds all the WebAPI actions decorated with the ```Dapr.Topic``` attribute and will tell Dapr to create subscriptions for them.

## Working with Dapr Pub/Sub components

Now that our application code is finished, we'll need to set up our Pub/Sub component. For my message broker, I'm going to be using Azure Service Bus. To define our Pub/Sub component to use Azure Service Bus, we can write the following YAML:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: dapr-pubsub
spec:
  type: pubsub.azure.servicebus
  version: v1
  metadata:
    - name: connectionString
      value: "<YOUR-SERVICE-BUS-CONNECTION-STRING>"
    - name:  consumerID
      value: "order-processor-subscription"
```

To define which message broker we will use for Pub/Sub capabilities, we define this using the ```type``` field. So for Azure Service Bus, this will be of type ```pubsub.azure.servicebus```. In the ```metadata``` section, we just need to define our ```connectionString``` and the ```consumerID```. Now for this example, I'm using connection strings purely for local development. In production scenarios, *please* use Azure AD to authenticate to your Service Bus namespace for more granular control. You can also use Dapr [secret stores](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/) to store sensitive values.

For our ```consumerID```, I've created a *subscription* for my Service Bus *topic* called ```order-processor-subscription```. The value that we give our ```consumerID``` in our Pub/Sub component should match the name of your subscription.

The power of the Dapr framework lies in its portability, meaning that you can plug in different message brokers without having to change your application code. Check out [this guide](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-publish-subscribe/#set-up-the-pubsub-component) on how to configure Pub/Sub components in Dapr and [this reference](https://docs.dapr.io/reference/components-reference/supported-pubsub/) to see all the supported message brokers available to you for your Dapr applications. 

## Testing our API

With our application logic and Pub/Sub component defined, we can now test sending and receiving messages via Dapr. To do this, we'll need to open two separate command lines (one for our publisher, one for our subscriber). In each command line, run the following commands:

```bash
# Run our Publisher API
dapr run --app-id bookshop-publisher --app-port <app-port-defined-in-launchSettings.json-file> --dapr-http-port 3500 --resources-path ..\..\..\components\ dotnet run

# Run our Subscriber API
dapr run --app-id bookshop-subscriber --app-port <app-port-defined-in-launchSettings.json-file> --dapr-http-port 3502 --resources-path ..\..\..\components\ -- dotnet run
```

Again, let's break these commands down:

- The ```--app-id``` parameter sets the id for our applications.
- For our ```--app-port```, we will use the ports defined in our APIs ```launchSettings.json``` file.
- We need to give both our APIs different ```--dapr-http-ports```, so I've used ```3500``` for our publisher and ```3502``` for our subscriber.
- I've added my Pub/Sub component into a ```components``` folder, so I've used the ```--resources-path``` to point to that folder when running our application.
- We then run both our publisher and subscriber with ```dotnet run```.

To test our APIs, we start by sending a ```POST``` request to our publisher endpoint so it will send a message to our topic. For this, I'm going to use Postman. In Postman, we can do this like so:

![Image of the Postman UI, with a URL defined in the URL text box, along with the message payload and the successful response below the payload](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kggwl7l8d5x8snga98xl.png)

Here, we make a ```POST``` request to our ```http://localhost:5105/api/orders``` endpoint, and send a JSON payload for our order. In our controller, we 

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i7ifwiysvqvknztvvcsp.png)

Since we're running our Subscriber API at the same time, we should see in our Subscriber terminal that the message has been received.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rhukmaj32a86yldky8ol.png)

## Conclusion

In this article, we talked about Pub/Sub in Dapr and how we can use it to enable Pub/Sub capabilities in our distributed applications. We then learnt how to configure a publisher and subscriber in ASP.NET Core Web API projects, how to configure our pub/sub component and how we can test our publisher and subscriber projects locally.



If you have any questions on the above, feel free to reach out to me on twitter (or X, whichever suits you) [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️