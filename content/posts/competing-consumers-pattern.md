---
title: "The Competing Consumers Pattern"
date: 2024-01-15
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/70lpttvdgbod5rl2k18m.png
    alt: "A conceptual 3D rendering depicting the 'Competing Consumers' model in message processing. Multiple futuristic robotic arms, representing consumers, extend towards a central, glowing digital messaging channel. This channel is overflowing with luminous data messages, illustrating a high volume of information. The robotic arms are engaged in grabbing and processing these messages, symbolizing the concepts of optimizing throughput, enhancing scalability, and distributing workload. The scene is set against a sleek, high-tech backdrop, emphasizing advanced technology and digital processing."
    caption: 'Competing Consumers can enable multiple consumers to process messages received on the same message broker. Multiple messages can be processed concurrently to optimize the throughput and scalability of our application.'
---

Applications running in the cloud should expect to handle a large number of requests. One method of dealing with large amounts of requests is to pass them through a message broker to a consumer service that handles these requests asynchronously. This helps ensure that requests are being processed without throttling or degrading the performance of our applications.

The volume of these requests can vary over a period of time. Imagine that you're a developer for an e-commerce website. The traffic you'll receive during events like Black Friday or Boxing Day sales will be significantly than usual, with requests coming from multiple users that is difficult to predict. During peak hours, you may have to process hundred or thousands of requests per second, while other times demand may be significantly smaller.

Depending on the type of requests, the nature of work required to handle these might also vary. If we try to process these requests on a single instance of our consumer, you may throttle the instance, or your message broker might be overloaded with messages that come from the application.

One solution could be to run multiple instances of your consumer service, but you'll need to coordinate the consumers to ensure that each message is only delivered to a single consumers, as well as ensuring that consumers are load balanced correctly to prevent a single instance from becoming a bottleneck.

An alternative solution would be to implement the **Competing Consumers pattern**. This enables multiple concurrent consumers to process messages received on the same messaging channel. Using multiple concurrent consumers, your application can process multiple messages to optimize the throughput, improve scalability and availability.

In this post, I'll talk about the Competing Consumers pattern, the benefits of the pattern, things we need to be mindful of when implementing the pattern and when we should (and shouldn't) use this pattern.

## What does the Competing Consumers pattern do?

As I mentioned earlier, the Computing Consumers pattern enables multiple consumers to process messages concurrently on the same messaging channel. With these concurrent consumers, your application can process multiple messages to optimize the throughput of your application, improve scalability and increase availability.

Using this pattern, we can use a message queue to communicate between the applications and instances of the consumer service. Your application will post messages to the queue, and instances of your consumer service will receive messages from the queue and process them. This allows the pool of consumer services to handle messages from any instance of your application, like so:

![A schematic diagram illustrating a message queuing system in Azure Service Bus. On the left, multiple application instances, represented by lightning bolt icons, are generating messages. These messages are directed towards a central Azure Service Bus Message Queue, depicted as a series of envelopes lined up in a queue. On the right side of the queue, there's a consumer service instance pool, indicated by a vertical rectangle filled with lightning bolt icons, processing the messages from the queue. The diagram represents the flow of messages from producers to a managed message queue and then to the consumers](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lsd0rmp2qsk2g2wphakd.png)

## What benefits can the Competing Consumers pattern provide?

This patterns can provide us with a couple of benefits. It can provide a system that can handle different volumes to traffic sent to the queue by our application instance, by enabling our message queue to act as a buffer between our application and consumer instances. This helps us solve the problem of having high volumes of traffic impacting our overall availability and responsiveness.

The message queue also removes the need to implement a complex orchestrator between the consumer instances. Our message broker ensures that each message that is sent is delivered at least once. As far as our consumer service is concerned, we can apply auto-scaling to our consumers to handle the number of messages that we have to process.

The reliability of our system also improves, since the message producer isn't concerned about sending messages to a specific consumer. This means that if a consumer instance fails, it won't block the work being done by the producer, it will just be processed by another working instance.

## What should we be mindful of when implementing the Competing Consumers pattern?

While this pattern can improve the scalability of your application, you should also consider if you need to scale your message broker. Having a single message queue for numerous producers/consumers could throttle the queue. You may want to consider implementing multiple queues for producers to send to. You'll also want to think about the reliability of your message broker. You'll need to ensure that your message isn't lost.

Poison messages should also be taken into consideration. You'll need to prevent poison messages from returning to the queue or being reprocessed, and capture the details of the poison message so you can determine what to do next.

If your operations are idempotent, you'll need to design the system in such a way to prevent messages from being processed more than once. This is also important to ensure that the order of messages don't affect your business operations. In Azure Service bus queues, you can implement guaranteed first-in-first-out (FIFO) ordering by [using sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions).

## When should we use this pattern? And when shouldn't we?

If your workload is divided into independent tasks that can run in parallel and asynchronously, implementing the Competing Consumers pattern can help with dealing with variable amounts of traffic, and increase the scalability of your workload. This pattern can also help increase the availability of your application, as well as resiliency against failures.

However, if your application is tightly coupled with dependencies between tasks that must be performed, this will be a difficult pattern to implement. This is particularly true if our application must wait for a task to be completed by a consumer before it can continue.

## Conclusion

In this article, we discussed the **Competing Consumers pattern**, the benefits of the pattern, things we need to be mindful of when implementing the pattern and when we should (and shouldn't) use this pattern.

If you want to read more about these patterns, check out the following resources:

- [Azure Architecture doc on the Competing Consumers pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers)
- [Competing Consumers on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️