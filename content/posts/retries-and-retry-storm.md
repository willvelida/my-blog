---
title: "The Retry Pattern and Retry Storm Anti-pattern"
date: 2024-01-15
draft: false
tags: ["Azure","Architecture", "Compute", "Cloud Design Patterns", "Cloud Design anti-patterns"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/98vrh84qnj0bh7yyi7xo.png
    alt: "An epic and dramatic 3D visualization of the Retry Storm antipattern in a vast digital landscape. The image showcases a colossal cybernetic storm with intense lightning strikes and powerful whirlwinds, dramatically engulfing a network of futuristic, interconnected systems. These systems are intricately designed with sophisticated geometric shapes and are overwhelmed by an extraordinary number of retry requests, far exceeding the number of stable connections, in a 100:42 ratio. The scene is set against a dark, ominous sky with vibrant electrical effects, emphasizing the catastrophic impact of the antipattern in a high-tech, cyberpunk aesthetic."
    caption: 'Transient errors that occur within cloud environments can be mitigated by using the Retry pattern. However, if we use frequent retries that overwhelm our systems, we can cause Retry Storms that harm the stability of our applications.'
---

When developing cloud applications, we'll be interacting with remote services that can suffer from transient faults. Our applications need to be able to handle transient failures when trying to connect to remote services, and retry a failed operation to improve the stability of your applications. This is where the **Retry pattern** comes into play.

*N.B Transient faults are faults that include the momentary loss of network connectivity to components and services, the temporary unavailability of a service, and timeouts that occur when a service is busy.*

However, we also need to bear in mind that when remote services are unavailable or busy, retrying endlessly is going to make the problem worse, and may not make sense to retry forever since requests are usually only valid for a limited amount of time. If we configure retries to happen endlessly, this is referred to as a **Retry Storm**. This is an anti-pattern that can lead to a waste of resources and cause more problems that it tries to solve.

In this article, I'll talk about the Retry pattern, why and when we would want to implement retries in our application, what Retry Storms are and what we need to consider in order to avoid Retry Storms taking place.

## What is the Retry pattern?

Applications running in the cloud need to be mindful of transient faults that can happen. These can include loss of network connectivity to external services, unavailability of external services, timeouts when external services are busy etc.

we need to design our applications to handle faults transparently without having an impact on our functionality. When our application detects a failure when calling a remote service, we can handle that failure using the following strategies:

1. **Cancel the request** - If our request fails due to an error that isn't caused by a transient issue, or if we know that a subsequent attempt is going to fail as well, we should just cancel the request and throw an exception.
2. **Retry the operation** - If the failure reported is a rare, it might have been caused by something unusual, so in this case we can retry the request immediately since the failure is unlikely to happen again.
3. **Retry the operation after a delay** - If the fault is caused by a transient issue, such as an external service being too busy, that service might need a period of time to recover, so our application should wait a short time before retrying the request again. This delay can be increased incrementally or exponentially, depending on the type of failure.

Let's take a look at an example:

![A diagram illustrating the communication between a calling application and a hosted service. The calling application, represented by a logo with a central hexagon and surrounding cubes and circles, sends multiple requests to the hosted service, depicted as a blue circle with three white dots. Three horizontal arrows point from the calling application to the hosted service, with two arrows returning, indicating responses. The responses are labeled with the HTTP status codes '500' for the first two indicating server errors, and '200' for the last one indicating success.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5mwldis1ehze5pgtcruj.png)

In this diagram, we have an application that invokes an operation on a hosted service. The first request fails, so the host responds with a HTTP 500 response. After a short time, the calling application tries again, but the call fails with a 500 response. After a longer time, the calling application retries the call, and finally succeeds with a HTTP response of 200.

In our applications, we should wrap all requests to remote services in code that implements a retry policy that follows one of the strategies I listed earlier. If you are a .NET developer like myself, you may be familiar with the [Polly library](https://www.thepollyproject.org/). Golang has a library called [Retry](https://pkg.go.dev/github.com/sethvargo/go-retry), and there are numerous third-party libraries for Python and Java.

Depending on the external service, you may want to tailor your retry policy to fit the nature of that service. For example, some vendors may provide a default retry policy out of the box, specify the maximum number of retry attempts, the time between retry attempts, and so on.

When attempting retries, you should to log why the operation is failing. This will be useful in situations when you want to know *why* a request is failing. Bear in mind that you'll need to consider how logs will be processed to avoid overloading your operations team.

If you find that you're making a call to a service that you own, and it's frequently failing, you may want to consider scaling out the service to prevent it failing.

## What should we consider when using the Retry pattern?

Think about your business requirements when considering your retry policy. You may have operations that aren't as critical as others, so in this case you may want to just fail the operation rather than retrying it.

If an external service continues to fail after numerous retries, it'll be better for the application to prevent any further requests going to the overloaded resource and fail the request immediately. You can then wait for a period of time, and allow one request through after that time to see if the request is successful. This is referred to as the **Circuit Breaker** pattern.

If your operations are idempotent, that it will be safe to retry. If it's not, then retried operations could cause unintended side effects. For transactional operations, you'll need to configure your retry policy to ensure that you don't need to undo all the transaction steps that have been successful.

*N.B Idempotent - processing the same event multiple times should have the same effect as processing it once.*

Think about the type of error that's thrown from the external service. In our scenario, we received a ```HTTP 500``` code, but what if we received a ```HTTP 400``` response instead? This indicates that our client application made an invalid request, rather than our external service being offline. Retrying this request is not going to help, since our request is invalid.

If your retry policy is too aggressive, you could cause further problems in your application, affecting the user experience. If you configure your retry policy to run forever, and the external service remains unresponsive, this can make the problem worse and decrease the validity of your request the longer you attempt to retry.

This is an anti-pattern called **Retry Storm**.

## What do we mean by a Retry Storm?

When we retry requests within a short period of time, it's likely those requests will continue to fail as the external systems may not have recovered. If you have clients making multiple connection requests while external systems attempt to recover, you can overwhelm or *storm* the service, making the problem worse.

Take the following example:

![A diagram showing multiple failed attempts by a calling application to communicate with an offline hosted service. The calling application is represented by a purple hexagon with additional geometric shapes in a circular formation. Several horizontal lines labeled 'Request 1' to 'Request n' indicate attempts to connect to the hosted service, which is represented by a blue circle with three white dots and labeled 'Offline hosted service.' Each line connecting the application to the service has an arrow at both ends, but adjacent to the service, there's a '500' error code, signifying server errors for each request.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0hrxybmkiajtacv20g4z.png)

In this scenario, our calling application has a retry policy that's configured to last forever making requests to a hosted service that's offline. As we can see, every time the client application makes a request, we get a HTTP response of 500 (Service Unavailable). With these constant requests, the calling application is overwhelming the service, and it could have an effect on the performance of our calling application.

## How can we avoid creating Retry Storms?

There's a couple of things we can do to prevent Retry Storms from occurring.

We can set a limit of the number of retries our applications attempt. This will help prevent your client application from overloading the external service. You should also set a time limit on how long your application attempts to retry a request, and pause between the retry attempts. Retrying immediately is unlikely to succeed, so by setting a time limit on retries and increasing the amount of time between retries is a great way to exponentially backoff from making requests to external services.

If external services are experiencing issues over a period of time, or if the service is experiencing a specific type of error that can't be solved immediately, you will want to consider implementing a Circuit Breaker pattern instead. Some services may provide a ``retry-after`` header in the response, so you'll want to only retry the request after that specific time period.

Azure SDKs have built-in retry policies that help protect your applications from creating Retry Storms. Earlier, we talked about third-party libraries like Polly (.NET) and retry (Golang). These are great for when the service you're making requests to doesn't have a built-in retry policy. Environments that supports service meshes for outbound calls can also be configured to use retry policies, meaning that you don't have to write the retry code yourself.

Services themselves should be protected against retry storms. Adding gateway layers can protect services from Retry Storms by shutting off connections to it during incidents. It also allows you to throttle requests at your gateway.

## Conclusion

In this article, we discussed the **Retry pattern**, why and when we would want to implement retries in our application, what **Retry Storms** are and what we need to consider in order to avoid Retry Storms taking place.

If you want to read more about these patterns, check out the following resources:

- [Azure Architecture doc on the Retry pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [Azure Architecture doc on the Retry Storm anti-pattern](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/retry-storm/)
- [Transient fault handling](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transient-faults)

If you have any questions, feel free to reach out to me on X/Twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️