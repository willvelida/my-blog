---
title: "Decentralized workflow coordination using the Choreography pattern"
date: 2024-01-30
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2wa7lckpyx7fod34kvyr.png
    alt: "An expansive 3D rendering of a futuristic and sophisticated business workflow, depicted in a 100:42 ratio. The scene is filled with numerous high-tech computers, servers, and data centers, all bustling with activity. Each piece of technology is represented by advanced, futuristic machines and holographic interfaces, interconnected by a complex web of glowing data streams and electrical connections. The network operates in a decentralized fashion, emphasizing a collaborative decision-making process without a central hub. Overhead, dynamic beams of light and energy swirl in orchestrated patterns, adding to the sense of an awe-inspiring, orchestrated complexity that characterizes advanced technological systems."
    caption: 'Instead of relying on a centralized orchestrator, you can choreograph your microservices to participate in the workflow of your business transactions.'
---

In a microservices architecture, services are divided into smaller ones to work together to process business transactions. This gives us the advantage of being able to develop services quicker and scale them independently.

However, designing a distributed workflow is challenging and communication between the services is complex.

One approach we could take is to implement a centralized orchestrator that acknowledges all incoming requests and delegates them to the relevant services. The orchestrator manages the entire workflow of the transaction.

The orchestrator helps to reduce point-to-point communication between services, but it does result in tight coupling between the orchestrator and other services. If you want to add or remove services, this will result in you having to rewrite parts of your application.

If you want to learn more about orchestration in a Saga pattern, check out [this blog post](https://www.willvelida.com/posts/saga-pattern/) that I wrote on the subject.

Instead of relying on a central point of control, we can implement the **Choreography pattern**, where each component in the system participates in the workflow of a business transaction.

In this article, I'll talk about how we can use the Choreography pattern to perform tasks in a workflow transaction, when we should use the Choreography pattern, and some things we need to keep in mind when using the Choreography pattern.

## What is the Choreography pattern?

Instead of relying on a centralized service, let each service within your architecture decide when and how a business operation is processed. We could implement asynchronous messaging to coordinate business operations between our different services:

![A diagram illustrating a message-driven microservice architecture. On the left side is an icon labeled 'Order API', composed of three interconnected purple hexagons, symbolizing the initiation of an order. An arrow labeled 'Order Created' points from the Order API to a central rectangular element labeled 'Service Bus Queue', depicted with three envelopes within a box, representing a messaging system that holds the order information. From the Service Bus Queue, three arrows extend to the right, each pointing to a distinct service icon labeled 'Service A', 'Service B', and 'Service C', respectively. These services are represented by individual sets of interconnected hexagons similar to the Order API, indicating separate microservices that handle different aspects of the order. The diagram communicates the flow of data from the initial API through a service bus and then to various microservices.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d0cxmynz9r3ciu1y8q7m.png)

Using asynchronous messaging, clients will publish messages to a message queue. These messages are then pushed to subscribers who are interested in that particular message.

Each subscriber performs the operation that the message prescribes. The subscriber then replies to the message queue with the outcome of that operation. If it's successful, the service can send a message to initiate the next step in the workflow. If it fails, the message broker can retry the operation.

Using this approach, services choreograph the transaction workflow among themselves without needing an orchestrator, or needing direct communication between them. There's also no bottleneck caused by the orchestrator, since it doesn't exist!

I previously worked as a Software Engineer for a bank where we implemented the Choreography pattern as part of a payment workflow. I've simplified the process in the diagram below.

![A flowchart representing a file processing system. It starts with 'File A Storage' represented by a folder icon, leading to 'FileValidator', symbolized by a lightning bolt, which flows into 'FileAdaptor', another lightning bolt symbol. From 'FileAdaptor', the process leads to 'FileCreator', also depicted by a lightning bolt, and ends at 'File B Storage', shown with a folder icon. Two vertical arrows labeled 'DLQ' point downwards from 'FileAdaptor' and 'FileCreator', indicating a Dead Letter Queue where files might go if an issue occurs during processing. The flow is linear, with arrows between each component showing the direction of the file movement.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qp7ilgqc63u14rcpcnpx.png)

Every week we would pick up a file from Blob Storage, which would kick off the transaction. We had an Azure Function that would perform some validation work against a reference table to ensure that the File was in a valid format. If it was valid, we would send the payload in a message to an Adaptor function, where we would transform the file into a format that another transaction would pick up. This payload would be sent to a FileCreator Function, which would produce the file in a format that could be used by other banking systems.

Should there be any failures in the transaction, we would send compensating events to undo successful operations in the transaction, as well as send the original payload to the dead-letter queue, so we can investigate what went wrong.

## What should we keep in mind when using the Choreography pattern?

While removing the orchestrator removes a centralized performance bottleneck, decentralizing our workflow can create some issues. Rather than a single service being the single point of failure, the failure is distributed between all services. Now, you may think that this is better, but each service becomes responsible for the entire workflow, which needs to be considered within the logic of each service. Handling retries, terminating requests gracefully, communicating results etc. all adds complexity to each of your applications.

If an operation fails, it can be difficult to recover from that failure. We can solve this by implementing compensating transactions to undo steps in the workflow that were successful. We could also fire events to retry operations, or time them out.

If your workflow needs to occur in a sequence, you'll need to consider implementing multiple message brokers to send messages in the required order.

As the number of services grow within your architecture, the complexity of your choreography will increase. Tracing all of this is also something you should keep in mind!

## When should I use the Choreography pattern?

If you're using a centralized orchestrator within your architecture and your experiencing performance bottlenecks with it, you may want to consider implementing the choreography pattern instead. 

This also makes it easier if you're adding or removing services from your architecture, as this will cause minimal disruption to your communication paths.

## Conclusion

In this article, we discussed the **Choreography** pattern, how we can implement it, what we need to keep in mind when implementing it, and when we should use this pattern.

If you want to read more about this pattern, check out the following resources:

- [Choreography on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/conversation/Choreography.htmll)
- [Azure Architecture doc on the Choreography pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/choreography)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️ 