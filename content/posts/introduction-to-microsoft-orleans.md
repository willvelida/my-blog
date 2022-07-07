---
title: "Introduction to Microsoft Orleans"
date: 2022-07-07T20:47:30+13:00
draft: false
tags: ["dotnet", "Microsoft Orleans", "ASP.NET", "microservices", "Azure Container Apps"]
ShowToc: true
TocOpen: true
cover:
    image: https://raw.githubusercontent.com/dotnet/orleans/gh-pages/assets/logo_full.png
    alt: "Microsoft Orleans Logo"
    caption: 'Microsoft Orleans is a cross-platform framework for building distributed applications using .NET'
---

I had a bit of spare time yesterday between customer meetings and wanted to do some coding. I've been hearing about [Microsoft Orleans](https://docs.microsoft.com/en-us/dotnet/orleans/overview) a lot on twitter and even found this [Microsoft Learn module](https://docs.microsoft.com/en-us/learn/modules/orleans-build-your-first-app/) that shows you how to build your first Orleans app with ASP.NET Core 6.0. I'm a fan of distributed systems and frameworks, so I've wanted to get my head around Orleans for a while.

For those of you who may not know, Orleans is a cross-platform framework that you can use to build distributed applications in .NET. These apps span more that just a single process using peer-to-peer communication. Orleans can scale from a single lonely on-prem server to globally distributed applications running in the cloud. It can run anywhere .NET can, including Linux, Windows and macOS.

In this article, I'll be discussing:

- What Microsoft Orleans is.
- What the Actor Model is and how it works with Orleans.
- The essential concepts of Orleans.
- What we can do with Orleans.

Let's dive in!

## What is Microsoft Orleans?

As I mentioned above, Microsoft Orleans is a cross-platform framework for building distributed applications using .NET.. We can scale Orleans elastically, from a single on-prem server to multiple, globally distributed applications running in the cloud.

For .NET developers, Orleans extends the familiar concepts that we are used to working with in a single server environment to multi-server environments. This is great for developers who are transitioning from building applications on single machines to multi-machine environments.

Orleans can run on Linux, Windows and macOS, and we can deploy Orleans Apps to Kubernetes, VMs or PaaS services such as Azure Container Apps.

## What is the Actor Model and how does it work with Orleans?

Orleans is based on the actor model. This is a programming model where each actor is a lightweight, concurrent, immutable object that encapsulates a piece of state and corresponding behavior.

Actors will communicate with each other exclusively using asynchronous messages. Orleans invented the idea of *Virtual Actors*, which are logical entities that always exist virtually, meaning they cannot be created explicitly or destroyed.

Microsoft Research wrote a paper back in 2010 on Virtual Actors which you can read [here](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/) if you want to learn more.

## What are the essential concepts of Orleans?

There are a couple of primitives that Orleans uses that we need to know about. Following on from Virtual Actors, the first primitive that we will discuss are *grains*.

Grains represent actors in the Actor model and define the state data and behavior of an entity. An example of this could be a shopping cart or a product. Grains are identified through user-defined keys, which can be used to access the grains by other grains and clients. Here's a visual representation of what a grain looks like:

![visual representation of a Grain in Orleans](https://docs.microsoft.com/en-us/dotnet/orleans/media/grain-formulation.svg)

Now when we implement Grains in dotnet, we essentially create a normal class that inherits from the Grain base class like so:

```csharp
public interface IUserGrain : IGrainWithStringKey
{
    Task<string> GetUsername(string email);
}

public class UserGrain : Grain, IUserGrain
{    
    public Task<string> GetUsername(string email)
    {
        return Task.FromResult(email);
    }
}
```

The Grain base class manages various internal behaviors and integrations with the Orleans framework. With Grain Classes, we need to implement at least one of the following interfaces:
- IGrainWithGuidKey
- IGrainWithIntegerKey
- IGrainWithStringKey
- IGrainWithGuidCompoundKey
- IGrainWithIntegerCompoundKey

These interfaces are used to mark your class with a different data type for the identifier that Orleans will use to track the grain.

Grains are stored in something called *Silos*. Grains that are active in your application will remain in memory, while those grains that are inactive can be persisted to storage.

Grains become active when they are needed by our applications. They have a managed lifecycle which is handled by the Orleans runtime. We don't need to write code that manages the lifecycle, we can just write code that assumes that grains are available to us in memory. Here is a visual representation of the Grain lifecycle in Orleans:

![Grain lifecycle in Microsoft Orleans](https://docs.microsoft.com/en-us/dotnet/orleans/media/grain-lifecycle.svg)

Let's turn our attention to *Silos*. A Silo in Orleans is a host that hosts one or more grains. We may have a group of *Silos* that we use in our application, which is known as a cluster.

The cluster will coordinate work between silos, which will allow communications with all our grains as if they were available in a single process. We can organize our grains by storing different types of grains in different silos, and our app can retrieve individual grains without worrying about how they are managed within the cluster. We can visualize the relationship between clusters, grains and silos using the following diagram:

![Visual representation of the relationship between clusters, silos and grains](https://docs.microsoft.com/en-us/dotnet/orleans/media/cluster-silo-grain-relationship.svg)

Silos provide a set of services such as timers, reminders, persistence, transactions, streams and more alongside the core programming model.

## What can we do with Orleans?

If we know that our .NET applications will eventually have to scale, then Orleans is a great framework for this. Orleans has been used successfully in a variety of different applications (such as Banking apps, Chat apps etc.).

My favorite use of Orleans is in [Halo!](https://www.microsoft.com/en-us/research/blog/orleans-simplifies-development-scalable-apps-cloud/) In this situation, game sessions are treated as Actors. The assumption is that game sessions are always there, so this simplifies the process of activating and creating the game sessions.

Orleans has the following features that make it easy to use it in a variety of different applications:

- Persistence (Ensures that the state is available before processing a request and ensuring that consistency is maintained)
- Timers and reminders (Provides both durable and non-durable scheduling mechanisms for Grains)
- Grain placement (The runtime decides which Silo to activate the grain on, which is configurable)
- Grain versioning
- Stateless workers
- ACID Transactions
- Streams (processing data in real-time)

As I learn more about Microsoft Orleans, I'll write up some deeper dive articles on each of these features. So stay tuned 😁

## Wrapping up

To be honest, I initially thought that Orleans would be super complicated to get my head around due to the unusual terminology, so hopefully I've simplified here for you. Once you spend a bit of time and get your head around, the concepts make a lot more sense.

In future articles, I'll do some hands-on tutorials on creating applications using Microsoft Orleans, as well as diving into the different features I mentioned earlier.

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️