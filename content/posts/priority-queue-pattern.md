---
title: "The Priority Queue Pattern"
date: 2024-01-17
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pggl0gtkd67oimqqajlr.png
    alt: "A dynamic 3D illustration of a Priority Queue system in computing. At the bottom, a vast array of futuristic computers forms a grid, with a central towering structure that represents the sending application. From this, digital envelopes adorned with glowing edges, indicative of their priority status, burst forth in an array. High-priority envelopes emit brighter light and are surrounded by visible energy fields, while lower-priority ones have a softer glow. They ascend towards an elaborate, radiant queue structure above, which sorts them before they continue towards their destinations. The background resembles a star-filled night sky, signifying the vastness and complexity of the data processing universe. The overall image conveys a sense of grandeur and high-stakes data management."
    caption: 'Using Priority Queues, we can assign requests and messages with a priority level so that we can receive and process higher priority requests.'
---

The **Priority Queue** pattern is a pattern that allows us to prioritize requests sent to services so that higher priority requests are received and processed faster than lower priority requests.

This is useful for when we are building applications where we need to prioritize some clients over others, or if some requests need to be processed faster than usual for business/compliance reasons.

In this article, I'll talk about what the Priority Queue pattern is, what advantages priority queues can provide, what we need to consider when implementing the Priority Queue pattern and how can implement it priority queues in Azure (*Spoiler alert, it's a little hacky!*).

## What is the Priority Queue pattern?

In our applications, we usually have services that will send requests to other services. When it comes to the cloud, we may send these requests to other services via a message broker. In many situations, the order that the requests are received by the consuming service isn't that important. However, there are some scenarios where we will need to process higher priority requests over lower priority requests.

This is where the **Priority Queue** pattern comes into play. This is where we will assign a priority to a message that we send to the message queue. Our publisher application can assign the priority, and then messages in the queue are automatically reordered so higher priority messages can be received and processed before those that have a lower priority.

![A simple black-and-white flowchart illustrating a Priority Queue messaging system. On the left, a rectangle labeled 'Sending Application' points to a series of envelopes indicating a message flow. One envelope is marked 'High Priority' as it enters a queue labeled 'Priority Queue.' Inside the queue, envelopes are ordered, showing 'Low Priority' and 'High Priority' labels. An arrow indicates that messages are ordered by priority in the queue. On the right, arrows lead from the Priority Queue to three circles representing 'Consumers,' with arrows looping back, suggesting a continuous process.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lygairn58t7sqg1zvfgp.png)

However, in the Azure world, there isn't a service that natively supports automatic prioritization of messages. So to implement a priority queue pattern in Azure, we need an alternative.

Azure Service Bus topics support subscriptions that provide message filtering. So you could create a Service Bus topic that your producers can send messages to, and in the message metadata you can assign a priority to the message that can be filtered on. The message is then sent to the subscription for the consumer to read it.

![A flowchart representing a message prioritization system with color accents. A 'Producer Application' on the left sends messages marked 'High' and 'Low' priority to a central 'Service Bus Topic' box. The Service Bus Topic is divided into 'High Priority Subscription' and 'Low Priority Subscription' sections. Four lightning bolt symbols, two labeled 'High Priority Consumer' and two labeled 'Low Priority Consumer,' are connected to their respective subscriptions, indicating the flow of messages based on priority. The color scheme of blue and yellow lightning bolts adds visual distinction between different priorities.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5n2faukb464hxqc7a6e2.png)

Another option you could implement is to maintain separate queues for each message priority. This shifts the responsibility to your application to send the message to the right queue.

![A black-and-white flowchart depicting a message prioritization system. A 'Sending Application' is shown at the bottom left, from which two envelopes are sent; one is labeled 'High Priority Message' and the other 'Low Priority Message.' Each envelope flows into its respective message queue, with one queue labeled for high priority messages and the other for low priority messages. From each queue, multiple arrows point to a series of circles labeled 'Consumer,' indicating the distribution of messages to various consumers based on their priority.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u0fzd9i06jcej1ptxnrs.png)

## What advantages do priority queues give us?

Using the priority queue pattern can provide us with a couple of benefits. If you have business operations that have a higher priority over others, such as providing higher levels of performance to larger customers, than having a priority queue for those customers can help provide that.

If you use a single queue and assign priority to the message itself, you can reduce your operational costs by scaling down the number of consumers if you need to. If you have multiple queues, you can scale the number of consumers for lower priority queues down, and even stop processing low priority messages should your architecture allow that.

Having multiple message queues can also improve the performance of your application. You can partition your messages based on the priority that you assign. High priority messages can be processed immediately, and lower priority messages can be handled later.

## What should we consider before implementing priority queues?

The *priority* of the message has a significant business impact, so priority should be framed in terms of business requirements. If a message has a high priority assigned to it, what does that mean to the business? Does that mean the message has to be processed in 10 seconds, 30 seconds, a minute?

Once you have an idea of what priority means from a business perspective, you can design how you're going to implement priority queues to support that.

Another thing you should consider when determining the priority levels of your messages is to determine how that affects the way your messages are processed. For example, should all high priority messages in the queue be processed before lower priority messages are processed? If so, you'll need to implement the priority queue pattern in such a way so that low priority messages will remain on the queue until all high priority messages are processed.

If you are required to process all messages (regardless of priority), then implementing separate queues for different priorities could be a good solution for this. This may introduce additional overhead from an operational standpoint, and shift the responsibility to your application to ensure that the message is sent to the right queue.

## Conclusion

In this article, we discussed the **Priority Queue** pattern is, what advantages priority queues can provide, what we need to consider when implementing the Priority Queue pattern and how can implement it priority queues in Azure.

If you want to read more about this pattern, check out the following resources:

- [Azure Architecture doc on the Priority Queue pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/priority-queue)
- [What is Priority Queue | Introduction to Priority Queue on Geeks for Geeks](https://www.geeksforgeeks.org/priority-queue-set-1-introduction/)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️