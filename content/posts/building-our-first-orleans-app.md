---
title: "Building our first Microsoft Orleans App"
date: 2022-07-10
draft: false
tags: ["dotnet", "Microsoft Orleans", "ASP.NET", "microservices", "Azure Container Apps", "Visual Studio"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/v1mlbdz7wudey0zxitdc.png
    alt: "Microsoft Orleans Logo"
    caption: 'Microsoft Orleans is a cross-platform framework for building distributed applications using .NET'
---

Continuing my [Microsoft Orleans learning journey](https://www.willvelida.com/posts/introduction-to-microsoft-orleans/), I decided to have a bit of a play around with [one of their tutorials](https://docs.microsoft.com/en-us/dotnet/orleans/tutorials-and-samples/tutorial-1). When it comes to complex concepts or new frameworks, I like to get my hands dirty with the code to help me understand how the moving parts work together in action.

In this blog post, I'll be building my own version of the minimal Orleans app that tutorial builds and dissecting each part as I go along. Bear in mind that there are many ways to build an app, and this is just my preferred way of doing it. The idea is to break down the original tutorial and provide a bit more context as to what is going on.

For this tutorial, I used the following tools:

- Visual Studio 2022 Community.
- .NET 6.0.

If you want to refer to the code as we go along, please check out [this repository](https://github.com/willvelida/HelloOrleans) on my GitHub. This tutorial assumes that you have experience creating projects in Visual Studio, but I'll try my best to explain how to do things as we go along :)

Let's dive in!

## Creating our projects

For this tutorial, we'll be creating three projects:

1. **Grains** - This project will be a .NET standard project that will contain our Grains that we'll use in the project.
1. **Silo** - This will be a .NET Console application. Remember that Silos in Orleans are a host that can host one or more Grains. We can even have a group of Silos in our application that form a Cluster.
1. **Client** - This will also be a .NET Console application. This client will be used for communicating with our Grain, connecting to our Cluster and invoke our grain.

In my code, I created a single solution to store my projects in. We'll also need to set up references for our projects like so:

- Silos will reference our Grains project.
- Client will also reference our Grains project.

If you didn't know already, to add a reference from one project to another in Visual Studio, **right-click** the project you want to add a reference to and then in the menu, select **Add** > **Project Reference**. You can then choose which project you want to reference to in the menu.

## Adding our packages

We'll need to add some NuGet packages to our projects to get access to the relevant Orleans APIs. Here are the packages we need to add:

| **Project** | **NuGet Package** |
| ----------- | ----------------- |
| Silo | Microsoft.Orleans.Server |
| Silo | Microsoft.Extensions.Hosting |
| Silo | Microsoft.Extensions.Logging.Console |
| Client | Microsoft.Orleans.Client |
| Client | Microsoft.Extensions.Logging.Console |
| Grains | Microsoft.Orleans.Core.Abstractions |
| Grains | Microsoft.Orleans.CodeGenerator.MSBuild |
| Grains | Microsoft.Extensions.Logging.Abstractions |

To add a NuGet package to a project in Visual Studio, right click on the project file can click on **Manage NuGet Packages**. Click on the **Browse** tab and enter the package name in the text box. I've just the latest package version for each project (Not the prerelease versions though).

### Wait, what did we just add?

Before going any further, let's explain the Microsoft.Orleans packages that I'm using in this tutorial.

- *Microsoft.Orleans.Server* and *Microsoft.Orleans.Client* are meta-packages that will bring in dependencies that you will need on both the client and silo side.
- *Microsoft.Orleans.Core.Abstractions* is a core abstractions library of Microsoft Orleans.
- *Microsoft.Orleans.CodeGenerator.MSBuild* automatically generates the code that is needed to make calls to our grains across machine boundaries.

Now that everything's been set up, let's start building our Orleans app!

## Building our Grain

Let's start off by creating our Grain.

Grains are the most basic primitive in Orleans and they represent actors that define the state data and behavior of an entity. In our project, we're going to be creating a simple Grain that says Hello. We'll need to implement an interface for our Grain like so:

```csharp
using Orleans;

namespace Grains.Interfaces
{
    public interface IHello : IGrainWithIntegerKey
    {
        Task<string> SayHello(string greeting);
    }
}
```

The Grain base class manages various internal behaviors and integrations with the Orleans framework. In our interface, we have implemented the ```IGrainWithIntegerKey``` interface which means that Orleans will track our grain using a Integer identifier.

We just have one method defined, which we will implement in our Grain like so:

```csharp
using Grains.Interfaces;
using Microsoft.Extensions.Logging;
using Orleans;

namespace Grains
{
    public class HelloGrain : Grain, IHello
    {
        private readonly ILogger _logger;

        public HelloGrain(ILogger<HelloGrain> logger)
        {
            _logger = logger;
        }

        public Task<string> SayHello(string greeting)
        {
            _logger.LogInformation("SayHello message received: greeting = {Greeting}", greeting);
            return Task.FromResult($"\n Client said: '{greeting}', so HelloGrain says: Hello!");
        }
    }
}
```

In my ```HelloGrain``` class, I'm inheriting both the Grain base class and my ```IHello``` interface.

Grains interact with each other and get called from outside by invoking the methods declared as part of the Grain interfaces. All methods in the Grain interface must either return a Task, Task<TResult> or a ValueTask<TResult>. In our ```SayHello()``` method, we are return a Task<string> that just prints out a message.

## Creating our Silo

Now that our Grain has been set up, we can create our Silo. Our Silo wil host our ```HelloGrain```. In this tutorial, we're just using a local cluster for development which doesn't require any dependencies on external storage systems (Once I get my head around how Orleans works with external storage systems, I'll write up a blog post on that). 

We can set up our Silo code like so:

```csharp
using Grains;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Orleans;
using Orleans.Configuration;
using Orleans.Hosting;

try
{
    var host = await StartSiloAsync();
    Console.WriteLine("\n\n Press Enter to terminate...\n\n");
    Console.ReadLine();

    await host.StopAsync();
    return 0;
}
catch (Exception ex)
{
    Console.WriteLine(ex);
    return 1;
}

static async Task<IHost> StartSiloAsync()
{
    var builder = new HostBuilder()
        .UseOrleans(c =>
        {
            c.UseLocalhostClustering()
            .Configure<ClusterOptions>(options =>
            {
                options.ClusterId = "dev";
                options.ServiceId = "OrleansBasics";
            })
            .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(HelloGrain).Assembly).WithReferences())
            .ConfigureLogging(logging => logging.AddConsole());
        });

    var host = builder.Build();
    await host.StartAsync();

    return host;
}
```

Let's break this down a bit:

- In our ```StartSiloAsync`` method, we create a [].NET Generic Host](https://docs.microsoft.com/en-us/dotnet/core/extensions/generic-host) that we will use to encapsulate our app's resources and lifetime functionality. 
- We use the ```.UseOrleans()``` method to start configuring our Orleans cluster for local development.
- In our first ```Configure``` method, we configure our Orleans ```ClusterOptions``` to give our Cluster an ID of `dev` and our Service and Id of `OrleansBasics`. 
- We then use ```ConfigureApplicationParts``` to explicitly add the assembly with grain classes to the application setup and add any referenced assembly with the ```WithReferences``` extension. Here, we are just adding our ```HelloGrain``` class.
- We then configure logging for our Silo using the ```ConfigureLogging``` method to add console logging to our Silo.
- Once this is done, our Silo host is built and the silo will start.

## Creating the Client

The last piece of the puzzle will be to create our client to communicate with our HelloGrain, connect it to the cluster and then invoke our grain. Clients allows non-grain code to interact with a Orleans cluster.

In Orleans, we can obtain a client either in the same process as our silo, or in a separate process. According to [the documentation](https://docs.microsoft.com/en-us/dotnet/orleans/host/client#:~:text=This%20article%20will%20discuss%20both%20options%2C%20starting%20with%20the%20recommended%20option%3A%20co%2Dhosting%20the%20client%20code%20in%20the%20same%20process%20as%20the%20grain%20code.), the recommended approach to this is co-hosting the client code in the same process as the grain code. In this tutorial, we will take this approach by using the .NET Generic Host.

We can do this like so:

```csharp
using Grains.Interfaces;
using Microsoft.Extensions.Logging;
using Orleans;
using Orleans.Configuration;

try
{
    using (var client = await ConnectClientAsync())
    {
        await DoClientWorkAsync(client);
        Console.ReadKey();
    }

    return 0;
}
catch (Exception e)
{
    Console.WriteLine($"\nException while trying to run client: {e.Message}");
    Console.WriteLine("Make sure the silo the client is trying to connect to is running.");
    Console.WriteLine("\nPress any key to exit.");
    Console.ReadKey();
    return 1;
}

static async Task<IClusterClient> ConnectClientAsync()
{
    var client = new ClientBuilder()
        .UseLocalhostClustering()
        .Configure<ClusterOptions>(options =>
        {
            options.ClusterId = "dev";
            options.ServiceId = "OrleansBasics";
        })
        .ConfigureLogging(logging => logging.AddConsole())
        .Build();

    await client.Connect();
    Console.WriteLine("Client successfully connected to silo host! \n");

    return client;
}

static async Task DoClientWorkAsync(IClusterClient client)
{
    var friend = client.GetGrain<IHello>(0);
    var response = await friend.SayHello("Good morning HelloGrain!");

    Console.WriteLine($"\n\n{response}\n\n");
}
```

Again, let's break this down:

- In our ```ConnectClientAsync``` method, we configure the client that will connect with out Cluster. It's important to note here that the ```ClusterOptions``` configuration we have defined here is the same that we used for our Silo. We have to do this to ensure that our Client will connect with our Silo.
- In the ```DoClientWorkAsync``` method, we first get a Grain Reference to our ```IHello``` interface. A Grain Reference is a proxy object that implements the same grain interface as the corresponding grain class. It's used to make calls to the target grain, which we do by using ```await friend.SayHello()```. Here, we are just getting a response from our Grain and writing it to our console, but we can use grain references like any other .NET object, passing it to other methods or saving it to persistent storage.

## Seeing it in Action

Let's run our app and see it in action! We'll need to run both the client and the silo at the same time since they're in the same solution. To do this in Visual Studio, **right-click** your solution file and click on **properties**. Under **Startup Project**, select the **Multiple startup projects** radio button and configure the startup projects like so:

![Startup projects configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tkmo4yfebazvcfm3q47o.png)

Click on **Apply** then **Ok** to apply the settings. Press F5 to start both the Silo and Client projects.

First up, our Silo project will startup up and be available for our Client to connect to. Once our client has been successfully connected to the Silo, it will send a message to our Grain 'Good morning HelloGrain!'. Our ```HelloGrain``` Grain will then reply with the message 'Hello!'. You should see two different console outputs like so:

![Projects starting up](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qkjkgvu9nhz1u9zf2sqp.png)

## Wrapping up

In this post, we took the basic minimal Orleans tutorial and explained the moving parts in a bit more detail. Hopefully you can understand the basics of how we can set up a Grain in Orleans, connect it to a Silo and then configure a client to interact with our Grain.

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️