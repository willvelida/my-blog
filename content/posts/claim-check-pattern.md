---
title: "The Claim Check Pattern (reference-based messaging)"
date: 2024-01-13
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patters"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ffa6wtm2n2al0dz6sn2.png
    alt: "A 3D digital illustration in wide format depicting a data management concept. In the center, a large envelope labeled 'CLAIM CHECK' radiates with a blue glow and hovers above a futuristic messaging platform, symbolizing its delivery. To the right, a cube marked 'PAYLOAD' is connected to the platform via a glowing orange data stream, representing its storage in an external service. On the left, an abstract device represents the splitting of the message into the claim check and payload. The background features a network of lines and nodes, suggestive of a high-tech data transfer system."
    caption: 'Using the Claim-Check pattern, we can process large message without impacting our service bus and throttling our clients.'
---

The **Claim-Check** pattern, or Reference-Based Messaging, is a pattern where we split a large message into a claim check and a payload. The claim check is sent to our message broker and the payload will be stored in an external data store.

This pattern enables us to process large messages, while preventing our message bus from being overloaded and prevent our client applications slowing down. We can also reduce costs with this pattern, but utilizing cheap storage alternatives rather than increasing the resource units used by our message broker.

In this article, I'll talk about what problems this pattern solves, how we can implement it and some considerations we need to keep in mind when using the claim-check pattern.

## Why would we need this pattern?

In messaging architectures, we need to be able to send and receive a large number of messages. The data in these messages could contain any kind of binary data of any size.

When we start to send large messages to our message brokers, this is where we run into problems. The larger the message content, the more resources we'll need to use in order to process it. A message large enough, and your clients will start to experience throttling.

Message brokers are designed to handle a large amount of small messages, and in Azure, different message brokers will have limits on the size of the message.

| Broker | Message size limit |
| ------ | ------------------ |
| Azure Service Bus | 256KB for standard tier, 100 MB for Premium tier. |
| Azure Event Hubs | 256KB for basic tier, 1MB for Standard, Premium and Dedicated. |
| Azure Event Grid | 512KB for MQTT, 1MB for events, custom topics, system topic and partner topic resources. |

For more information on limits in various Azure messaging/event services, check out the following resources:

- [Service Bus quotas](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas)
- [Event Hubs quotas](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)
- [Event Grid quotas](https://learn.microsoft.com/en-us/azure/event-grid/quotas-limits)

## Implementing this pattern

If we find that our messages exceed the limits imposed on us by our message broker, we can use the claim-check pattern to work around this.

Instead of including the entire payload within the message, we can store the payload into an external data store such as a database, or into blob storage. We then use the reference of the stored payload and send just that reference to the message bus.

This reference is the claim check used to retrieve our payload from our external storage. Our clients that need to process that specific message can do so by obtaining the reference to that payload.

Let's take a look at what that might look like:

![A flowchart illustrating the claim check pattern in message processing. It starts with 'Message with Data' leading to 'SendMessage', represented by a lightning bolt symbol. This then branches off to 'Message with Claim Check', symbolized by a checkmark inside an envelope, which continues to 'ReceiveMessage', marked by a corresponding lightning bolt. The flow diverts to 'Payload stored in Data Store', indicated by a database symbol with a recycling arrow. The process concludes with the 'Message with Data' at the end of the cycle.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/maioryq6gjcnk4eaad7b.png)

In the diagram, we start with our *SendMessage* function that sends will send the message to Azure Service Bus. In this case, the payload of the message exceeds the message size limit of Service Bus, so we split the message into two parts.

The payload of our function will be stored by our *SendMessage* into an external data store. I've used Cosmos DB in the diagram, but it could be any data store that's appropriate for your needs.

The reference of the message (The *claim-check*) is then sent to the Service Bus. This reference is then read by our *Message Receiver* function, who uses it to retrieve the payload from our external data store and process the message.

## What should we keep in mind when using this pattern?

If you don't need the data payload after you've processed the message, you should think about deleting it. The more data you store over time, the more money it's going to cost you.

Another factor to consider is that since you'll be storing the message payload in an external data store, you'll need to factor in additional latency. If the size of your messages vary in size, you may only want to implement this pattern when a message exceeds the size limit of your message broker.

## Conclusion

In this article, we discussed the **Claim-Check** pattern and how we can use it to process messages that are larger in size than our message broker can handle.

If you want to read more about this patters, check out the following resources:

- [Claim-Check pattern on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html)
- [Azure Architecture doc on the Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️

