---
title: "Developing .NET Isolated Process Azure Functions"
date: 2022-02-13T21:14:05+13:00
draft: false
tags: ["Azure","C#","Serverless","Azure Functions"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kbkpfcw43p59n1nl3wrj.png
    alt: "Azure Functions Logo"
    caption: 'Using Isolated Process Functions, we can decouple the .NET version from the Functions runtime'
---

We can run our C# Azure Functions in an isolated process, decoupling the version of .NET that we use in our Functions from the version of the runtime that our Functions have been developed on ⚡

Before this, we would have to develop Functions that had a class library and host that were tightly integrated with each other. That meant that we had to run our .NET in-process on the same version as the Azure Functions Runtime (.NET Core 3.x on Azure Functions Runtime v3). With out-of-process Functions, we are able to use .NET 5 with v3 of the Azure Functions runtime.

With the ability to run our Functions out-of-process, we can benefit in the following ways.
- We can full control over the process, from how we start the app, to controlling the configuration of our Function.
- Using this control, we can use current .NET behaviors for DI and middleware.
- We will also benefit by having fewer conflicts, so our assemblies won't conflict with different versions of the assemblies that are used by the host process.

## Creating our Function

In order to create a Function that runs out-of-process from the Functions runtime, we'll need the following:

- .NET 5.0 installed
- Visual Studio 2019 version 16.10 or later installed (Make sure that you have either the Azure Development or ASP.NET and web development workload installed)

Once you have those installed, create a new Azure Function project in Visual Studio. When you create your project, make sure that you select .NET 5 (Isolated) for your project, like so:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sb1kwz97jccpywqmveql.png)

For this demo, I'm going to create a simple function that triggers on a HTTP POST request and inserts an item into an Azure Cosmos DB container. Not exactly changing the world I know, but the purpose here is to show you how Isolated Functions work compared to Azure Functions built as a [C# Class Library Function](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-your-first-function-visual-studio).

## What comes out of the box?

Essentially a .NET isolated function project is Console App that targets .NET 5.0. Your solution explorer should look something like this:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hgqk9vmnp0ik0qhe416g.png)

I've added a couple of files here, but these files get generated for you:

-  **.csproj file**. This file will define the project and its dependencies.
- **local.settings.json**. This stores app settings, connection string and settings used for local development.
- **host.json file**. This file contains the global config options for all functions within a Function app.
- **Program.cs file**. This will be the entry point for our application.


## Startup and Configuration

Through the Program.cs file, we have access to the start-up of our Function application. Instead of having to create a separate Startup class to do this, we now have direct access to the host instance, enabling us to set any configurations and dependencies on the host directly.

Here's an example:

```csharp
using Microsoft.Azure.Cosmos;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.IO;

namespace DemoIsolatedFunction
{
    public class Program
    {
        public static void Main()
        {
            var host = new HostBuilder()
                .ConfigureFunctionsWorkerDefaults()
                .ConfigureAppConfiguration(config => config
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("local.settings.json")
                    .AddEnvironmentVariables())
                .ConfigureServices(services =>
                {
                    services.AddSingleton(sp =>
                    {
                        IConfiguration configuration = sp.GetService<IConfiguration>();
                        return new CosmosClient(configuration["CosmosDBConnectionString"]);
                    });
                })
                .Build();

            host.Run();
        }
    }
}
```

Here, we are creating our host instance using a new HostBuilder object that will return a IHost instance that runs asynchronously to start our Function.

The **[ConfigureFunctionsWorkerDefaults](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.workerhostbuilderextensions.configurefunctionsworkerdefaults?view=azure-dotnet&preserve-view=true#Microsoft_Extensions_Hosting_WorkerHostBuilderExtensions_ConfigureFunctionsWorkerDefaults_Microsoft_Extensions_Hosting_IHostBuilder_)** method is used to add the settings required to run our Function app out-of-process. This does a couple of things such as providing integration with Azure Functions logging and providing default gRPC support.

The **[ConfigureAppConfiguration](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostbuilder.configureappconfiguration?view=dotnet-plat-ext-5.0&preserve-view=true)** method is being used here to add the configuration we need for our Function App. Here, I'm using it to use my **local.settings.json** file for local debugging.

The **[ConfigureServices](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostbuilder.configureservices?view=dotnet-plat-ext-5.0&preserve-view=true)** method allows us to inject the services we need in our application. Here, I'm using it to inject a Singleton instance of my Cosmos Client.

## Our Function Application

We're now ready to start writing our Function code. Here, I'm simply injecting the services I need in my function and on a POST request, inserting a Todo item into my Cosmos DB container:

```csharp
using DemoIsolatedFunction.Models;
using Microsoft.Azure.Cosmos;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System;
using System.IO;
using System.Net;
using System.Threading.Tasks;

namespace DemoIsolatedFunction.Functions
{
    public class InsertTodo
    {
        private readonly IConfiguration _configuration;
        private readonly CosmosClient _cosmosClient;
        private readonly Container _todoContainer;

        public InsertTodo(
            IConfiguration configuration,
            CosmosClient cosmosClient)
        {
            _configuration = configuration;
            _cosmosClient = cosmosClient;
            _todoContainer = _cosmosClient.GetContainer(_configuration["DatabaseName"], _configuration["ContainerName"]);
        }

        [Function("InsertTodo")]
        public async Task<HttpResponseData> Run([HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "Todo")] HttpRequestData req,
            FunctionContext executionContext)
        {
            HttpResponseData response;
            var logger = executionContext.GetLogger("InsertTodo");
            logger.LogInformation("C# HTTP trigger function processed a request.");

            try
            {
                var request = await new StreamReader(req.Body).ReadToEndAsync();

                var todo = JsonConvert.DeserializeObject<TodoItem>(request);
                todo.Id = Guid.NewGuid().ToString();

                await _todoContainer.CreateItemAsync(
                    todo,
                    new PartitionKey(todo.Id));

                response = req.CreateResponse(HttpStatusCode.OK);
            }
            catch (Exception ex)
            {
                logger.LogError($"Exception thrown: {ex.Message}");
                response = req.CreateResponse(HttpStatusCode.InternalServerError);
            }

            return response;
        }
    }
}
```

This function uses an HTTP Trigger to write a record to Cosmos DB. HTTP Triggers in isolated functions are different to older versions of the runtime as we must use [HttpRequestData](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.functions.worker.http.httprequestdata?view=azure-dotnet&preserve-view=true) and [HttpResponseData](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.functions.worker.http.httpresponsedata?view=azure-dotnet&preserve-view=true) to access the request and response data. 

In out-of-process functions, we don't have access to the original HTTP request and response objects. What happens in out-of-process functions is that the incoming HTTP Request Message gets translated to a HttpRequestData object. From here, the data is provided from the request.

Here's our post request:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xbizhk19v3zyvx6mlxdf.png)

This request provides data to our Body attribute in the HttpRequestData object. 

## Logging and Execution Context

We can write to logs in .NET isolated functions by using a ILogger instance. Isolated functions pass through a FunctionContext object that provides information about a function execution. Here, we can call the GetLogger() method passing in our function name like so:

```csharp
var logger = executionContext.GetLogger("InsertTodo");
logger.LogInformation("C# HTTP trigger function processed a request.");
```

## Want to learn more?

There's still a bit of work to do on .NET Isolated Functions. With .NET 6 dropping sometime in November, that version will probably be the LTS version and [you can start working with .NET 6 with Azure Functions v4](https://github.com/Azure/Azure-Functions/wiki/V4-early-preview) (It is in early preview, so expect bugs).

Personally, I'm super excited about .NET Isolated Functions! 🙌 With the increase in .NET version cadence, having the .NET version decoupled from the Azure Functions runtime version will provide .NET devs 👩‍💻👨‍💻 far more flexibility when it comes to using the latest features in .NET, rather than being constrained by limitations imposed by the runtime.

If you want to read more about how the .NET isolated process works, check out this article: https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide#logging

If you prefer to get your hands dirty with the code, follow this tutorial: https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-developer-howtos?tabs=browser&pivots=development-environment-vs

Happy coding! 💻☕