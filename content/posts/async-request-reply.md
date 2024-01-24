---
title: "The Asynchronous Request-Reply pattern"
date: 2024-01-24
draft: false
tags: ["Azure","Architecture", "Messaging", "Cloud Design Patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pycgxlg726p9cz2tysqa.png
    alt: "A 3D digital illustration depicting a futuristic concept of the Asynchronous Request-Reply pattern. On the left, there's a streamlined, high-tech frontend interface, characterized by sleek, simple, and user-friendly design elements, symbolizing the user's request. This interface is glowing subtly, emphasizing its advanced technology. On the right, an intricate and complex backend system is illustrated, representing asynchronous processing. It features detailed, sophisticated machinery and network structures, with a more industrial and robust appearance. Connecting these two elements are glowing, dynamic data streams, creating a visual flow of communication and response between the frontend and backend. The overall composition highlights the extended communication path in a digital, futuristic environment, captured in a wide format with a 100:42 aspect ratio."
    caption: 'Some APIs will need to initiate long-running background tasks. Clients will still need a clear response, so using the Asynchronous Request-Reply pattern can help decouple the background processing from the client application'
---

Client-side applications often rely on APIs to provide data and functionality, which are either in your control, or provided by a 3rd party. We usually communicate with these APIs using HTTP(S) protocols and REST. Usually, these APIs are designed to respond quickly to requests.

However, there can be a variety of things that affect the latency response from these APIs. These include network infrastructure, how many clients are attempting to call the API, the difference in location between the caller and API itself etc.

If these issues occur within our control, we can solve them relatively easily by scaling out the backend API. However, there are going to be problems that are out of our control, such as network problems. Now most APIs can respond over the same connection, but other requests may execute long-running or background process work that takes longer than it's reasonable for a client to receive a response. 

In a previous article, I talked about how we can use [**Queue-based load leveling** patterns](https://www.willvelida.com/posts/queue-based-load-leveling/) to use a message queue to separate the client and backend process so they can scale independently. While that's a viable solution, one potential downside of that pattern is that it does bring in additional complexity when our clients need to be notified of a successful operation.

Another solution to this problem is using the **Asynchronous Request-Reply pattern** by implementing HTTP polling. This helps client side receive an acknowledgment that work is either in progress, or accepted.

In this article, I'll describe what the Asynchronous Request-Reply pattern does, some issues and considerations we need to keep in mind when implementing this pattern, as well as when we should (and shouldn't) use this pattern.

## Implementing the Asynchronous Request-Reply pattern

Polling can be useful to client applications, as it's challenging to use long running connections or provide call-back endpoints. With polling, the client application will make a call to the API, which triggers long running operations in the backend.

The API will return a response as quickly as possible, acknowledging the request has been received. This is usually in the form of a HTTP `202 (Accepted)` status code. The response should also contain an endpoint that we can poll to check the status of the long running operation. The API will process the background work, which could be in the form of sending a message to a queue.

When we make a successful call to the status endpoint, it should return HTTP `200 (Ok)` as well as an update on the status of the background process work. Once the work is complete, the status endpoint can either return a resource that shows the work is complete, or redirect to another resource url.

Let's visualize this a little better:

![Sequence diagram for an HTTP request-response flow involving a Client app, API endpoint, Status endpoint, and Resource URL. The Client app initiates the process by sending a POST request to the API endpoint, which responds with 'HTTP 202' indicating that the request has been accepted for processing but not completed. The Client app then repeatedly sends GET requests to the Status endpoint, which initially returns 'HTTP 200', signifying OK status but the resource is not yet ready. Eventually, the Status endpoint responds with 'HTTP 302', a redirect, indicating the resource is ready and providing the Resource URL. Finally, the Client app sends a GET request to the Resource URL, which responds with 'HTTP 200', indicating that the requested resource is successfully retrieved. The diagram uses solid lines for requests and dashed lines for responses, with arrows indicating the direction of the communication.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t0lkd5tb84l84e0urdp3.png)

In this diagram, the client will send a request and receive a 202 (Accepted) response. The client can then send requests to the status endpoint. If the work is still in process, a 200 response is received. When the work is completed, the status endpoint will return a HTTP `302 (Found)` response with a resource url that the client can be redirected to.

We could implement this pattern in Azure like so:

![Flowchart illustrating the Asynchronous Request-Reply pattern in software architecture. The diagram includes the Client App, Web API, Queue, Background Processor, Storage Blob, and Status Endpoint as interconnected components. The Client App calls the API, which sends a message to the Queue. This message is processed by the Background Processor, which writes the result to the Storage Blob. The Status Endpoint checks for the result, and the Client App calls the Status Endpoint to get the processing status. Each component is represented by a symbol with purple cubes and blue connecting dots, except for the Queue and Storage Blob, which are depicted with blue and grey box icons, respectively.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0a92wsezmkeng2sub310.png)

So in this example, we have a client app hosted on Azure Container Apps that makes a call to an API. The API responds with the status endpoint, and it sends a message for a worker app to process, and write the result to Blob storage. While this work is going on in the background, the client app can call the status endpoint to see the status of the background work.

## What do we need to consider when implementing this pattern?

It's important to keep in mind that not every service will return 202. Sometimes, you may get a HTTP `404 (Not Found)` response since the resource URL doesn't exist yet.

If you receive a 202 response, you should check the header to see the URL that you need to call for checking the status of the request and to see how frequently the client can poll for the response (These should be the `Location` and `Retry-After` headers). You may need to manipulate the headers or payload in order to do this.

Bear in mind that you should return the appropriate HTTP response depending on the result of the background work. If you want to redirect on completion, either HTTP 302 or HTTP 303 are appropriate. When the background has successfully completed, HTTP 200, 201 or 204 should be returned.

If an error occurs while processing, you should store the error at the URL location set in the header along with an appropriate response that you can give to the client (HTTP 4xx codes). You may also want to give clients the option of cancelling a long-running operation. In this scenario, you should support some form of cancellation in your backend API.

## When should and when shouldn't you use this pattern?

As I mentioned earlier, this pattern is great for when you have client applications making requests to APIs that initiate background work and they need a response immediately instead of using long-running connections. It's also useful in architectures where you need to integrate new services with legacy applications that don't support modern callback technologies, such as WebSockets.

However, if you're required to have a response streamed to your client application in real-time, or you can use a service that uses asynchronous notifications instead, then you should consider alternatives. Server-side persistent network connections such as WebSockets can be used to notify the caller of the result instead of polling the URL yourself.

## Conclusion

In this article, we discussed the Asynchronous Request-Reply pattern does, some issues and considerations we need to keep in mind when implementing this pattern, as well as when we should (and shouldn't) use this pattern.

If you want to read more about this pattern, check out the following resources:

- [Asynchronous Request-Response pattern on Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/conversation/RequestResponse.html)
- [Azure Architecture doc on the Asynchronous Request-Reply pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/async-request-reply)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️
