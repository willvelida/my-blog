---
title: "Implementing Dapr Pub/Sub in ASP.NET Core Web APIs"
date: 2023-08-14
draft: true
tags: ["Dapr","Dotnet","ASP.NET Core", "Azure Cosmos DB"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ybagwwedpcsrokvzlczp.png
    alt: "Implementing Dapr State Management in ASP.NET Core Web APIs"
    caption: "Dapr's State Management API simplifies handling state in distributed architectures."
---

In event-driven architectures, communicating between different microservices via messages can be enabled using Publish and Subscribe functionality (Pub/Sub for short).

The **publisher** (or producer) writes messages to a message topic (or queue), while a **subscriber** will subscribe to that particular topic and receives the message. Both the publisher and subscriber are unaware of which application either produced or subscribed to the topic. This pattern is useful when we want to send messages to different services, while ensuring that they are decoupled from each other.

In this article, we'll discuss how Pub/Sub works in Dapr, and how we can implement Pub/Sub functionality in a ASP.NET Core Web API. We'll then configure our Pub/Sub component and wire it up to our API to see it in action.

## What is Pub/Sub in Dapr?

Dapr has a Pub/Sub API that you can use to implement Pub/Sub capabilities to your applications. It's platform agnostic, meaning that you can a variety of message brokers to send and receive messages from. We can configure our message broker as a Dapr Pub/Sub component at runtime, removin the dependency from our service and making it more flexible to changes (should you need, or even want to, change your message broker).

The API offers at-least-once message guarantees, meaing that Dapr ensures that the message will be delivered *at least once* to every subscriber. In the event that a message fails to deliver, or your application fails, Dapr will attempt to redeliver the message until it's successful.

When we use Pub/Sub in Dapr, our service (In our case, our publisher service), will make a network call to a Pub/Sub building block API. This building block will make a call to our Pub/Sub component (our configured message broker). To receive messages on a topic, Dapr subscribes to the Pub/Sub component with a topic and then delivers messages to a particular endpoint when messages arrive.

In this article, we'll be building the following:

<INSERT-DIAGRAM-HERE>

Dapr Pub/Sub has a variety of features available that simplifies Pub/Sub capabilities in your application, so I'd recommend taking a look at the [Dapr documentation on Pub/Sub](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/) if you want to gain a deeper understanding.


## Configuring Dapr in our Web API

## Implementing our Pub/Sub logic

## Working with Dapr Pub/Sub components

## Testing our API

## Conclusion