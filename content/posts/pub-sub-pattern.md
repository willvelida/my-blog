---
title: "The Publisher-Subscriber pattern (pub/sub messaging)"
date: 2024-01-26
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ayyeydn6ztq2zoo4w4m2.png
    alt: "The image is a 3D rendering of an asynchronous messaging subsystem, designed to be visually striking and epic. In the center, a colossal, intensely glowing structure represents the 'Input Messaging Channel'. Attached to this are numerous dynamic extensions, each signifying an 'Output Messaging Channel' leading to futuristic nodes or terminals, symbolizing 'Subscribers'. These nodes are uniquely designed, illuminated, and receiving energy-packed data packets. At the heart of the structure is a spectacular, central hub, resembling a powerful energy core, which orchestrates the flow of messages from the input to the output channels. The background is a vibrant, digital cosmos filled with neon lights, digital patterns, and conveying a sense of vast, technological space. The overall scene emphasizes the scale and intricacy of a high-tech communication network, rendered in a 100:42 aspect ratio."
    caption: 'Pub/Sub messaging enables applications to send events to multiple interested consumers asynchronously without coupling them to subscribers.'
---

In distributed architectures, different parts of our system will need to provide information to other parts as events happens. One way we could solve this is to provision dedicated queues for each consumer and send messages asynchronously to decouple them from message senders.

However, this wouldn't scale if we had to do this for large amounts of consumers, and what if some consumers where only interested in parts of the information that producers send. 

The answer to this problem can be found in the **Publisher-Subscriber pattern**, also known as **pub/sub messaging**, where we can enable our producers to send events to subscribers asynchronously, without coupling them to the senders.

Pub/Sub messaging is a fundamental pattern to use in distributed architectures, so in this blog post I'll talk about what pub/sub messaging is, what benefits it provides and some things to think about when using pub/sub messaging.

## What is Pub/Sub Messaging?

Pub/Sub messaging enables applications and producers to send events to subscribers asynchronously without those producers having to be coupled to senders.

The sender (*publisher*) uses an input channel to package events into messages and then send the message. These messages are then sent to output messaging channels for each consumer (*subscriber*) who is interested in the message. Each subscriber will get its own copy of the message from their output channel.

![Flow diagram representing a messaging system with a publisher-subscriber model. Starting from the left, a rectangle labeled 'Publisher' sends a message to a rectangle labeled 'Input Channel'. This channel then forwards the message to a central rectangle labeled 'Message Broker'. From the Message Broker, a message is distributed to three rectangles labeled 'Subscriber', each connected to an 'Output Channel' stemming from the Message Broker. The diagram visually represents the process of messages being published, brokered, and then subscribed to by different consumers.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9yim4um40b27zneqdp3i.png)

Using an example in Azure, we could have an Azure Function that sends a message to a Service Bus topic. A topic could have multiple subscribers, and each subscriber can be registered to the topic. If we have multiple Azure Functions acting as subscribers to the topic, each subscriber would receive their own copy of the message from the topic.

![Schematic diagram of a message publishing system using Azure Service Bus Topic. On the left, an icon labeled 'Publisher' with a lightning bolt symbol sends a message towards a rectangular box labeled 'Service Bus Topic', depicted at the center with the Azure Service Bus logo above it. Inside the rectangle are envelopes, representing messages. Three arrows emanate from the Service Bus Topic to three icons on the right, each labeled 'Subscriber' with a lightning bolt symbol, indicating the distribution of messages to multiple subscribers.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w0gkp295vrngdv40zhvj.png)

Event-driven architectures work with event producers generating a stream of events that event consumers listen to:

![Flowchart depicting an event processing system. The chart begins with a rectangle labeled 'Event Producer' on the left, which sends an arrow to a central rectangle labeled 'Event Ingestion'. From the Event Ingestion rectangle, three arrows branch out to three separate rectangles on the right, each labeled 'Event Consumer', indicating the distribution of the event to multiple consumers.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jr2ud6tmtw5cdjacpz5h.png)

Event-driven architectures can use pub/sub messaging to keep track of subscribers to events. When an event is published, it's sent to each subscriber. After the event is received, it can't be replayed, and new subscribers don't see the event.

This is a little different to another pattern called **event streaming**, where events are written to a log, and instead of consumers subscribing to a stream, they can read any part of the stream. This means that they can join the stream at any time, and replay events they want. However, in event streaming, clients are responsible for advancing its position within the stream.

## What are the benefits of pub/sub messaging?

Pub/Sub messaging helps us decouple producers and consumers that need to communicate. This helps us to manage them independently, whether that means we scale the producer out to handle produce more messages, or add consumers as we need to. It also gives us the ability to manage messages if consumers are offline through the message broker itself. 

This separation of concerns allows each application to focus on its core functionality while offloading the responsibility of routing messages to the right consumer to the message broker.

The overall reliability of our architecture increases. Asynchronous messaging helps our application run smoothly under increased traffic and handle intermittent failures better. Testability also increases, as we can inspect our messages as part of our testing strategy.

Subscribers don't have to pick up the messages off the message broker right away. Depending on your business requirements, you have the option to pick up the message on a specific schedule.

Since components of our architecture are decoupled by a message broker, integration between these components becomes easier since all they need to do is have the ability to receive the message from the message broker. This means that systems using different platforms, running in the cloud or on-prem, or different programming protocols can now communicate with each other more effectively.

## What do we need to keep in mind when implementing pub/sub messaging?

If at all possible, avoid building your own service that supports pub/sub messaging. There's numerous products out there that you can use for pub/sub messaging. In the Azure world, Azure Service Bus, Event Hubs and Event Grid can all be used for pub/sub architectures. Outside of Azure, Kafka, Redis and RabbitMQ all support pub/sub capabilities.

Communication in pub/sub messaging is unidirectional (moving in a single direction). If you require your subscribers to send acknowledgment or communicate the status back to the publisher, you're better off using the [**Request-Reply Pattern**](https://www.willvelida.com/posts/async-request-reply/).

How you handle your subscribers will be a big aspect of your design. Your message broker must be able to provide the ability for consumers to subscribe and unsubscribe as and when they need to. Security for your subscribers must also be taken into account. Connecting subscribers to a messaging channel securely is critical to prevent unauthorized applications from accessing messages from the message broker.

You'll also want to consider how subscribers handle messages. The order that messages are received aren't guaranteed, as well as the order messages were created. Messages may also be sent more than once (particularly if a sender fails after posting the message, then a new instance of the sender starts up and repeats the message). 

One method to prevent ordering and duplication from causing adverse affects in your system is to design your message processing logic to be idempotent.

*N.B A method is considered to be idempotent if the intended effect on the server of the same request being performed multiple times is the same as the effect for a single request.*

Azure Service Bus has a feature called [Sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions), which can force messages that are tagged with the same SessionId to be grouped together and delivered in a First-In-First-Out (FIFO) manner. In this context, FIFO means the order in which messages **arrived** to Service Bus, not the internal order that you want to follow when processing the messages. 

![Diagram illustrating message sequencing in Azure Service Bus. The top of the diagram shows the Azure Service Bus logo followed by a queue with messages labeled numerically to indicate order. Below are three horizontal lines representing different sessions, each with a 'Session ID'. Messages with matching session IDs are aligned vertically under the queue. Arrows point from these messages to three different receivers, each marked by a lightning bolt symbol. Receiver 1 gets messages from 'Session ID 1', Receiver 2 from 'Session ID 2', and Receiver 3 from 'Session ID 3', demonstrating how messages are directed to specific receivers based on session IDs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hpcrywet0e3rgjk4ydct.png)

Another feature of Service Bus is [Message Deferral](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-deferral), which allows you to process messages in a order that you define by deferring messages through its SequenceId.

Other message considerations include the lifetime of our message (should they expire after a certain time?), when our message should be processed, and whether or not different messages should be processed faster than others. If you do need to assign a priority level to specific messages, you may want to consider implementing the [**Priority Queue pattern**](https://www.willvelida.com/posts/priority-queue-pattern/) to ensure that higher priority messages are processed before others.

Finally, if the content of your message is malformed (known as a *Poison Message*), you'll want to prevent those messages being returned to the queue, and capture the payload somewhere so you can analyze the content and fix the issue if needed. Azure Service Bus supports this by using [dead-letter queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues).

## In what situations would we use this pattern?

If you have an application that needs to send messages to numerous applications, regardless of where they are deployed or what communication protocol they use, the pub/sub pattern is effective for broadcasting that information. Pub/sub communication is ideal if you don't need responses from consumers in real-time, and your architecture can support eventual consistency.

However, if you only have a few consumers in your architecture, and they all require different pieces of information from your producer, pub/sub won't help you out here. If you require real-time responses and interactions between your producer and consumers, you'll want to consider an alternative.

## Conclusion

In this article, we discussed the **Publisher-Subscriber pattern** what benefits it provides and what we need to consider when using pub/sub messaging.

If you want to read more about this pattern, check out the following resources:

- [Publish-Subscribe Channel pattern on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html)
- [Azure Architecture doc on the Publisher-Subscriber pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber)
- [Idempotent message processing](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-mission-critical/mission-critical-data-platform#idempotent-message-processing)
- [Event-driven architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️


