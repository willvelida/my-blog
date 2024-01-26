---
title: "The Sequential Convoy Pattern"
date: "2024-01-27T08:30:00"
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/47xijksqiow1twbg5ptp.png
    alt: "The image shows a complex 3D rendering of a Sequential Convoy pattern, which is designed to represent a data processing system. Multiple conveyor belts are intricately laid out in parallel and perpendicular formations, each carrying a series of cubes and rectangular blocks in various colors such as blue, green, orange, and pink. These blocks symbolize data packets or messages. The conveyor belts are interlinked but operate independently, showcasing the non-blocking processing of different sequences of data. Each belt runs without interference from the others, indicating the system's ability to handle multiple data streams simultaneously. The overall scene conveys a bustling, efficient data processing facility, with each color-coded or labeled block moving in a defined, orderly sequence, illustrating the concept of processing related messages in order without delaying other processes."
    caption: 'Using the Sequential Convoy pattern, we can process related messages in a defined order without blocking the processing of other groups of messages.'
---

In most cases, we will our applications to process a sequence of messages in the order that they arrive. This is especially the case if and when we need to scale out our message processors to handle any increased load in messages.

This isn't always an easy task, because as our application scales out, instances can often pull messages from the queue independently, similar to how this happens when implementing the [**Competing Consumers pattern**](https://www.willvelida.com/posts/competing-consumers-pattern/).

For example, you may have an e-commerce application that processes customer orders. Each order goes through several steps, such as validating the order, processing payment from the customer and shipping the order. Each step must be processed in this specific order, and the services performing each step needs to keep track of which order it is processing. It must be able to do this without blocking the processing of other orders that are made.

This is where the **Sequential Convoy** pattern comes into play. With this pattern, we can process a set of related messages in a defined order, without blocking the processing of other groups of messages that our application needs to process.

## How does the Sequential Convoy pattern work?

Using our e-commerce application example from before, we use the Sequential Convoy pattern to push messages related to our categories within the queuing system, and pull only from one category, one message at a time. 

![The image is a flow diagram depicting a producer-consumer model with a queue. It consists of three rectangles and one square, representing different components, connected by arrows that show the flow of data. On the left side, there is a rectangle labeled "Producer" which has an arrow pointing to a square in the center labeled "Queue". From the "Queue", there are three arrows pointing to the right, each leading to a rectangle. These three rectangles are labeled "Consumer A", "Consumer B", and "Consumer C", respectively. This diagram illustrates a typical producer-consumer scenario where a single producer sends messages to a queue, and multiple consumers retrieve messages from the queue](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wh5176a03lidlu924x7g.png)

In our e-commerce application, orders made in our application would be our categories, and messages representing actions made on our orders would be pulled from the queue within that group.

In Azure Service Bus, we can use Sessions and give our messages a SessionId to correlate our messages. For example, order 1 is made, and the order gets validated, payment is processed, and then shipped to the customer. Each message needs to be processed in order in a First-In-First-Out (FIFO) manner, but only at the order level. 

When we send each message, we can define a SessionId property to the message to indicate that these messages are correlated to a specific order. Using Azure Functions or Azure Logic Apps, we consume messages correlated with the same SessionId to process each message in the order that it's received.

![Diagram illustrating message sequencing in Azure Service Bus. The top of the diagram shows the Azure Service Bus logo followed by a queue with messages labeled numerically to indicate order. Below are three horizontal lines representing different sessions, each with a 'Session ID'. Messages with matching session IDs are aligned vertically under the queue. Arrows point from these messages to three different receivers, each marked by a lightning bolt symbol. Receiver 1 gets messages from 'Session ID 1', Receiver 2 from 'Session ID 2', and Receiver 3 from 'Session ID 3', demonstrating how messages are directed to specific receivers based on session IDs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hpcrywet0e3rgjk4ydct.png)

Orders are processed in a transaction which will never span multiple orders. Each consumer can processes orders in parallel, but in a FIFO manner within a order.

Let's take a look at how this might look in Azure:

![The image is a flowchart representing a transaction processing system with multiple steps and actors. It begins on the left with a symbol labeled "Producer," which resembles a lightning bolt, indicating an action or event generation. An arrow leads from the Producer to a symbol labeled "Ledger Queue," depicted as a database icon. Another arrow extends from the Ledger Queue to a "Ledger Processor," which is represented by the same lightning bolt icon as the Producer.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/av74vow4hr2ohx2ukona.png)

We have a ledge processor that fans out messages by de-batching the content of each message in the first queue. The ledger processor Function takes care of processing the ledger one transaction at a time. It then sets the SessionId of the message to match the order ID, so the broker knows that these messages need to be processed within the same batch. It then sends each transaction to our transactions queue with the SessionId set to the order ID.

Each consumer Function will listen to the Transaction queue where each matching order ID message will be processed in the queue using [peek-lock mode](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-transfers-locks-settlement#peeklock).

## What should we keep in mind when using the Sequential Convoy pattern?

Scaling can be an issue here, since the ledger queue will be the primary bottleneck. Use a property of your incoming message to scale out on. In the e-commerce example, this could be the order ID. Your required throughput should also be taken into account.

As you scale, how you add categories to the convoy will also be a factor to consider. You may need to consider adding ledger processors to accommodate for new categories that you add to the system.

Finally, consider the message broker that you'll be using. If it allows for one-at-a-time processing of messages within a category, great! If not, you may need to consider alternatives. Regardless of message broker, you may receive messages that are out of order due to network latency. You may have to implement a flag within your last message to indicate that it is indeed the last message to process. Sequence numbers may also need to be considered if you have multiple messages as part of a transaction.

## When should use the Sequential Convoy pattern and when shouldn't we?

If you have messages that arrive in an order and MUST be processed within that order, or your incoming messages can be categorized in a way that it becomes a unit of scale for the system, then the Sequential Convoy pattern is an effective pattern for handling those messages.

This pattern is not suitable for scenarios where you have extremely high throughput scenarios (such as processing millions of messages per second or minute). This is because the FIFO requirements of Sequential Convoys limits the scaling of your system.

## Conclusion

In this article, we discussed the **Sequential Convoy pattern**, how it works, what we should keep in mind when using this pattern, and when we should and shouln't use it.

If you want to read more about this pattern, check out the following resources:

- [Azure Architecture doc on the Sequential Convoy pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sequential-convoy)
- [Send related messages in order by using a sequential convoy in Azure Logic Apps with Azure Service Bus](https://learn.microsoft.com/en-us/azure/logic-apps/send-related-messages-sequential-convoy)
- [Message Sessions in Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions)
- [Peek-Lock Message (Non-Destructive Read)](https://learn.microsoft.com/en-us/rest/api/servicebus/peek-lock-message-non-destructive-read)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️