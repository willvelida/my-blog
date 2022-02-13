---
title: "How do bindings work in Isolated Process .NET Azure Functions?"
date: 2022-02-13T21:16:05+13:00
draft: false
tags: ["Azure","C#","Serverless","Azure Functions"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kbkpfcw43p59n1nl3wrj.png
    alt: "Azure Functions Logo"
    caption: 'Using Isolated Process Functions, we can decouple the .NET version from the Functions runtime'
---

In this post, I'll explain what Bindings are in Azure Functions, How they currently work with in-process Functions and how for isolated functions, they work a little differently.

## What are Bindings?

In Azure Functions, we use Bindings as a way of connecting resources to our functions. We can use input and output bindings and the data from our bindings is provided to our Functions as parameters.

We can be flexible in the way that we use Bindings! We can use a combination of Input and Output bindings or none at all (using Dependency Injection instead). 

Input binding pass data to our function. When we execute our function, the Functions runtime will get the data specified in the binding.

Output bindings are resources that we write our output of our function to. To use the, we define an output binding attribute to our Function method.

## How do they work in Class Library Functions

In class library functions, we can configure our Bindings by decorating the Function methods and parameters using C# attributes like so:

```csharp
[FunctionName(nameof("FunctionName"))]
public async Task Run([EventGridTrigger] EventGridEvent[] eventGridEvents,
[EventGrid(TopicEndpointUri = "TopicEndpoint", TopicKeySetting = "TopicKey")] IAsyncCollector<EventGridEvent> outputEvents,
ILogger logger)
{
    // Function code
}
```

In the above code, we've got a function that's triggered by event grid and uses a Event Grid output binding to publish events to. The incoming event is bound to an array of EventGridEvents and the output is published to a IAsyncCollector of EventGridEvent types.

The parameter type is defining the data type for our input data into this function.

## How are they different in Isolated Process Functions

Like class library functions, binding are defined by using attributes in parameters, methods and return types.

Bindings in .NET Isolated Project are different because they can't use the binding classes like IAsyncCollector<T>. We also can't use the types inherited from SDKs like DocumentClient. Instead we have to rely on strings, arrays and POCOs (Plain Old Class Objects).

Let's illustrate this with an example. I've created a function that receives a HTTP POST request and persists a document to Cosmos DB.

Here's our function code:

```csharp
using System;
using System.IO;
using System.Net;
using System.Threading.Tasks;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace IsolatedBindings
{
    public static class InsertTodo
    {
        [Function("InsertTodo")]
        [CosmosDBOutput("%DatabaseName%", "%ContainerName%", ConnectionStringSetting = "CosmosDBConnectionString")]
        public static async Task<object> Run([HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,            
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

                return todo;
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

This function, uses a Azure Cosmos DB output binding. We're applying the binding attribute to our function method which defines how we write our POST request to the Cosmos DB service.

In order to use the CosmosDBOutput binding we need to install the following package:

```csharp
Microsoft.Azure.Functions.Worker.Extensions.CosmosDB
```

Because functions that run in a isolate process use different binding types, we need to use a different set of binding extension packages. We can find the full list of these here: https://www.nuget.org/packages?q=Microsoft.Azure.Functions.Worker.Extensions

In class library functions, we would define our output binding in the method attribute where we would define the output type and write our Function output to that binding like so.

In Isolated Functions, the value returned by our method is the value that will be written to our Cosmos DB Output binding. So in our function, we make a POST request containing the Todo item that we want to write to Cosmos DB and when it gets returned from our Function, it will be written to Cosmos DB.

Let's test this out! I'm using Postman to make the following request:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t6yht9c9c9tp7gvk5xdg.png)

When our function starts up, we should get a local endpoint to send out POST request to like so:

```
http://localhost:7071/api/InsertTodo
```

When I make a POST request to that endpoint, I receive the following response:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uw83fht5cdhug6imooer.png)
 
Let's head into our Cosmos DB account and we should see that our document has been successfully written to:
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u38m2ryrxnjczczo9xkc.png)

## Final thoughts

Personally, I usually opt for using Dependency Injection in Functions over using output bindings when connecting to other resources. 

But if we want to do something quick and easy, output bindings are really useful for getting the job done. Hopefully in this article, you can see even though there are small differences when using Bindings in isolated process functions compared to class library functions, they essentially work the same way.

If you want to read more about how the .NET isolated process works, check out this article: https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide

Happy coding! ☕💻

