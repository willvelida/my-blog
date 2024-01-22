---
title: "The Queue-Based Load Leveling Pattern"
date: 2024-01-22
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rkoxyv5i0jkre7jmui74.png
    alt: "An elaborate 3D rendering showcasing the Queue-Based Load Leveling pattern in a futuristic setting. The image, in a 100:42 aspect ratio, features a colossal, radiant tube at its center, symbolizing the queue. This tube is vibrant with colorful, digitized requests. Scattered throughout the scene are various computers and digital envelopes, representing data processing and message handling. These computers appear integrated into the high-tech landscape, some floating around the queue, highlighting their role in data analysis and management. The envelopes, depicted as digital holograms, seamlessly merge into the data streams, illustrating the tasks being processed. On the right, a grand, fortress-like structure represents the service, with gates meticulously processing each request. The background is a dramatic skyline, illuminated by neon lights and holographic displays, enhancing the scene's grandeur. The overall atmosphere is intense and awe-inspiring, emphasizing the system's efficiency in managing high demand with added layers of technology and communication."
    caption: 'To prevent external services from being overloaded by requests, we can use queues to act as a buffer between tasks and invoked services to level the incoming load to external services that our applications use.'
---

Applications in the cloud can be subjected to heavy peaks of traffic in intermittent phases. If our applications can't handle these peaks, this can lead to performance, availability and reliability issues. For example, imagine we have an application that stores state temporarily in a cache. We could have a single task within our application that performs this for us, and we could have a good idea of how many times a single instance of our application would perform this task.

However, if the same service has multiple instances running concurrently, the volume of requests made to the cache becomes difficult to predict. Peaks in demand could cause our application to overload the cache and become unresponsive due to the amount of requests flooding in.

This is where **Queue-Based Load Leveling** patterns can help us out. Instead of a service invoking another service, we use a queue that acts as a buffer between our application and the service that it invokes to prevent heavy traffic from overloading services and causing failures or timeouts.

In this article, I'll talk about how we can use Queue-based load leveling to prevent traffic overloading external services, what other benefits this pattern provides, and some things we need to keep in mind when using queue-based load leveling.

## Implementing Queue-Based Load Leveling in Azure

To prevent our applications from directly overloading services, we can introduce a queue between our application and service so that they run asynchronously. Our application posts a message to the queue that's required by the service. Our queue then acts as a buffer between the application and service. The message containing the data that the service needs stays on the queue until it's retrieved by the service.

Say we have an API hosted on Container Apps that interacts with a Cosmos DB database. Without going too much into the internal mechanics of Cosmos DB, if we don't have enough compute resource for our database, we're going to see 429 errors returned to our API if we try process too many requests to our database.

![Graphic showing a simplified system architecture with four container app replicas, represented by purple hexagonal icons with blue circles, on the left. Each replica is connected by lines to a 'Data Store' symbol on the right, depicted as a blue cloud-like icon with orbiting rings. One of the connections is marked with a '429' error, indicating a failed request due to too many requests, typically representing rate limiting or throttling in the system.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5ms7bk8qkl6guis1b83h.png)

To solve this, we can use queue-based load leveling to resolve this by putting a Service Bus queue between replicas of our API and our Cosmos DB database. We can have another Container App that reads messages from the queue that reads/writes to our Cosmos DB account instead. 

![Diagram illustrating the architecture of a queue-based load leveling system using Azure Service Bus. On the left, there are three container app replicas depicted as purple cubes with blue circles, indicating multiple instances sending messages. These messages are directed towards a central horizontal Azure Service Bus Queue, represented as a rectangle filled with envelope icons, signifying queued messages. On the right, a single cube labeled 'Consuming Service Data Store' with a surrounding blue cloud-like design receives messages from the queue, illustrating the processing of queued tasks.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/myayplm65r40gapy6dlw.png)

Within our Container App that reads messages from the queue, we can implement logic that controls the rate at which messages are read from the queue to prevent our Cosmos DB datastore from being overloaded (otherwise we've just moved the problem, rather than actually solving it 😅).

Our Service Bus queue decouples our application from the database, and the container app reading messages from the queue can do so at its own pace, regardless of how many messages are being sent to the queue concurrently. 

This helps increase the availability because any delays in our consuming service won't have an impact on our producer application, which can continue to post messages to the queue. This can also be helpful when we're trying to scale, as the number of queues and consumer services can be scaled to meet the demand.

Queue-based load leveling can also help us control our cloud costs. If you have an idea of what the average load of your application is, within your consumer application you can configure it to meet the average load, rather than accommodate for the peak load. 

Depending on what external service you use (whether that is Cosmos DB, or any other type of datastore), it's likely that the datastore will have throttling implemented when demand reaches a certain threshold. You can use this to configure your consumer application to load level to ensure that whatever throttling threshold your external service has implemented isn't reached.

## What should we keep in mind before implementing this pattern?

You want to avoid moving the problem from the producer side, to the consumer side. WIthin your application logic that receives messages from the queue, you'll want to control the rate at which messages are consumed to avoid overloading external services. This will need to be tested under load to ensure that your incoming load is actually leveled. From those tests, you'll be able to determine how many queues and instances of your consumer you'll need to achieve the load leveling required.

If you expect a reply from the service that you send a request to, you will need to implement a mechanism to do this, since message queues are a one-way communication mechanism. If you need your application to receive a response from the consuming service with low latency, than queue-based load leveling may not be the right pattern for your use-case.

You may also find yourself running behind requests, since they are being queued up by the queue rather than being processed right away. Simply autoscaling the number of instances that are consuming messages from the queue may also cause resource contention on the queue, decreasing the effectiveness of using the queue to level the incoming traffic load.

The persistence mechanism of your chosen queue technology is also important here. You have the potential for losing messages, or the queue itself crashing. Choose a message broker that meets the need for your desired load-leveling behavior, and keep in mind the limitations of that message broker.

## Conclusion

In this article, we discussed the **Queue-Based Load Leveling** pattern is, how we can use it to prevent traffic overloading external services, what other benefits this pattern provides, and some things we need to keep in mind when using queue-based load leveling.

If you want to read more about this pattern, check out the following resources:

- [Azure Architecture doc on the Queue-Based Load Leveling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️