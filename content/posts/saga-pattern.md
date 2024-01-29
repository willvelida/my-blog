---
title: "The Saga Pattern"
date: 2024-01-29
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jew8yuocdunnexdu7icp.png
    alt: "3D rendering of the Saga design pattern in a vast, futuristic digital universe. The image shows a network of microservices visualized as advanced, intricate nodes, resembling miniaturized, glowing cities with complex pathways. These nodes are connected by robust, multi-layered streams of light in vivid colors and intricate patterns, representing the flow of transactions in a saga. Surrounding the nodes are floating holographic interfaces displaying complex algorithms, code snippets, and real-time transaction data. Above this network, a colossal hologram of a digital globe is central, with beams of light emanating from it, signifying global data consistency management. The scene is set against a cosmic background filled with stars, emphasizing the grand scale and high-tech nature of the Saga design pattern."
    caption: 'Transactions in distributed architectures are hard. Using the Saga pattern, we can manage data consistency across microservices using a sequence of transactions in each service to trigger distributed transactions.'
---

In a microservices architecture, we may adopt a [database-per-microservice](https://learn.microsoft.com/en-us/dotnet/architecture/cloud-native/distributed-data#database-per-microservice-why) approach to let each domain service us a data store that best serves the type of data that microservices uses. With a database-per-microservice approach, we can scale out our data stores independently, and should our data store fail, that failure will be isolated from other services.

However, this approach gets complicated when we need to perform operations in a transactional manner. Transactions must be ACID (atomic, consistent, isolated and durable). Now within a single service, this isn't too challenging. Across multiple microservices, this becomes difficult to manage. Distributed transactions require all services in a transaction to commit or roll back before the transaction can proceed. Not all data stores support this model.

[Interprocess communication](https://en.wikipedia.org/wiki/Inter-process_communication) does allow separate processes to share data, but all microservices would have to be available for transactions to commit. A better approach would be to implement the **Saga pattern**, which can help us manage data consistency across microservices in distributed transaction scenarios.

A saga is a sequence of transactions that updates each service and publishes a message or event to trigger the next step in the transaction. Should a step fail, the sage will perform a compensating transaction that rollback the preceding transactions that succeeded.

In this article, I'll do a quick refresh on what ACID transactions mean, before stepping into the weeds of the Saga pattern, what approaches we can take to implement the Saga pattern, things we need to keep in mind when implementing the Saga pattern, and when we should use the Saga pattern.

## Remind me, what are ACID transactions again?

Transactions are a single unit of work that can be made up of multiple operations. Within a transaction, events change state on entities, and commands capture all the information required to perform an action on an entity.

Transactions must be ACID. In a microservices architect, ACID means:

- **Atomicity** is a set of operations that must occur together or none at all.
- **Consistency** means that the transaction takes data from one valid state to another.
- **Isolation** guarantees that concurrent transactions would produce the same data state that transactions executed sequentially would have produced.
- **Durability** ensures that transactions that are committed remain that way when our systems fails.

## What is the Saga Pattern?

The Saga pattern uses a sequence of local transactions (transactions within a service). Each of these local transactions will update that service data store, and then sends a message or event to trigger the next local transaction in the saga.

If a local transaction should fail, the service within a saga will perform compensating transactions that undo changes that were made in the previous service.

![A diagram depicting the flow of a Saga design pattern in a microservice architecture. The top of the diagram has 'Saga' written in bold with a dashed outline indicating the scope of the pattern. Three vertical service containers are shown in sequence from left to right, labeled 'Service A', 'Service B', and 'Service C'. Each service container has a lightning bolt icon at the top and an 'SQL' cylinder icon at the bottom, labeled 'Local Transaction', indicating individual databases. Arrows labeled 'Message/Event' connect these services horizontally, demonstrating the asynchronous communication between them. The message flow starts from Service A, moves to Service B, and then to Service C, outlining a chain of local transactions linked by messages or events as part of the Saga pattern.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/140tb44uz12d3bmkh2kd.png)

**Compensating transactions** are transactions that can be potentially reversed by processing another transaction with the opposite effect. We can also implement **pivot transactions**, which if they commit, the saga will run until the process is complete. This transaction cannot be retried or be compensated, or it can be the LAST compensating transaction or the first retryable one in the saga. **Retryable transactions** follow pivot transactions, and are guaranteed to succeed.

There are two ways that we can implement the Saga pattern: Choreography or Orchestration.

## Approach 1: Choreography

In this approach, we coordinate our sagas where they exchange events without a single centralized point of control. Each local transaction publishes domain events that trigger local transactions in other services.

![A diagram illustrating a message-driven microservice architecture. On the left side is an icon labeled 'Order API', composed of three interconnected purple hexagons, symbolizing the initiation of an order. An arrow labeled 'Order Created' points from the Order API to a central rectangular element labeled 'Service Bus Queue', depicted with three envelopes within a box, representing a messaging system that holds the order information. From the Service Bus Queue, three arrows extend to the right, each pointing to a distinct service icon labeled 'Service A', 'Service B', and 'Service C', respectively. These services are represented by individual sets of interconnected hexagons similar to the Order API, indicating separate microservices that handle different aspects of the order. The diagram communicates the flow of data from the initial API through a service bus and then to various microservices.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d0cxmynz9r3ciu1y8q7m.png)

This approach is great for simple workflows that only have a few services in the Saga and they don't need to be coordinated. There's no single point of failure, since responsibilities are distributed across the saga, and this implementation doesn't need additional service implementation or maintenance.

However, as you grow your architecture, the complexity increases as it will be difficult to track with Saga participants need to listen to which commands. Each service in the saga may become dependent on each other cyclically since they need to consumer each other's commands. Integration testing becomes a nightmare, as all services will need to be running to simulate a transaction.

## Approach 2: Orchestration

The alternative approach is to coordinate your sagas, so that a central controller tells all services in the sage what local transactions they need to execute.

The orchestrator handles all transactions and tells every service in the sage which operation they need to perform based on events, while interprets the state of each task, as well as handling failures with compensating transactions.

![A simple diagram representing a microservice orchestration for order processing. On the left, there is an icon labeled 'Order API' with three connected hexagons, indicating the initial point of the process where an order is created. An arrow labeled 'Order Created' points from the Order API to a central cube labeled 'Orchestrator', which represents the control center managing the workflow. From the Orchestrator, three arrows point outwards to three other icons, each labeled 'Service A', 'Service B', and 'Service C' respectively. These services are depicted with similar hexagon motifs, symbolizing different microservices involved in the order process. The diagram is designed to represent the flow of data and control from the initial API call through the orchestrator to the various services in a microservices architecture.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xk3wub7lz4h6lmgm94ty.png)

This approach is great for when our workflows have numerous services in the saga, or we know ahead of time that more services will be added. You remove the cyclical dependencies that you can potentially suffer in Choreography, since the orchestrator depends on all participants in the saga. Services in the saga also don't need to know about commands for other services, providing you with clear separation of concerns.

However, this does introduce complexity, since this implementation requires coordination, and there's an additional point of failure, since the orchestrator manages the workflow.

## What do we need to keep in mind when implementing the Saga pattern?

Initially, the Saga pattern is a bit of a challenge to implement. Transactions are not local, they're distributed, which can be a pain to coordinate and manage (Coincidentally, this was the first pattern I was introduced to coming out of university. It was a nightmare!)

This pattern is also a pain to debug and test, as the more services you introduce, the greater the complexity. Data can't be rolled back in this pattern either, since services commit changes to their local database.

As with all microservice architectures, you need to be able to handle transient failures, and idempotency is important to handle data consistency.

Your saga can potentially be made up of several services, for please implement a method to observe all your services and the ability to track the workflow of the saga!

This pattern could also introduces challenges around data durability. Lost updates, dirty reads, and non-repeatable reads could all occur within a Saga. You may need to implement semantic locks, pessimistic concurrency, versioning, commutative updates etc. to reduce the effect that anomalies may introduce.

## When should we use the Saga pattern?

If you need to ensure data consistency in a microservice architecture, or if you need to roll back or compensate with your microservices, the Saga pattern can provide you with the ability to do both.

However, if you have cyclic dependencies between your microservices, tightly coupled transactions or compensating transactions that occur in earlier services within your Saga workflow, you should think of alternatives.

## Conclusion

In this article, we discussed the **Saga** pattern, the two ways that we can implement the Saga pattern, what we need to keep in mind when implementing the Saga pattern, and when we should and shouldn't use the Saga pattern.

If you want to read more about this pattern, check out the following resources:

- [The Saga Pattern by Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Compensating Action on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/conversation/CompensatingAction.html)
- [Azure Architecture doc on the Saga distributed transactions pattern](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️