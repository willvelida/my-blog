---
title: "Implementing Dapr State Management in ASP.NET Core Web APIs"
date: 2023-06-15
draft: false
tags: ["Dapr","Dotnet","ASP.NET Core", "Azure Cosmos DB"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j2h5diedl3uyneomby1c.png
    alt: "Implementing Dapr State Management in ASP.NET Core Web APIs"
    caption: 'Dapr's State Management API simplifies handling state in distributed architectures.'
---

In distributed architectures, we'll usually have a number of independent stateless services that serve a purpose (set around a domain, such as ordering items, storing items in a catalog, handling authentication etc.)

While services should strive to be stateless, there will be some services that need to track state in order to fufil a business operation or function. Dapr has a state management build block that simplifies state tracking and enables you to store state across a variety of data stores.

In this article, we'll discuss what State Management is in Dapr, and how we can implement it in an .NET ASP.NET Core Web API. We'll then configure our state management component and test our API to see how it works in action.

## What is State Management in Dapr

Dapr has a State Management API that your application can use to save, query, delete and read key/value pairs in **state stores**. There are a variety of [supported state stores](https://docs.dapr.io/reference/components-reference/supported-state-stores/) that Dapr applications can use, such as MongoDB, Apache Cassandra, Redis, AWS DynamoDB, Azure Cosmos DB etc. 

Thanks to Dapr's pluggable components capability, we can model our data store as components and swap them out to use a different data store, without having to change your code in your application.

State Management also provides you with options on choosing the level of data consistency that your application needs and it provides Optimistic Concurrency Control to handle concurrency. By default, Dapr uses eventual consistency and last-write wins for concurrency, but you can configure these to suit the requirements of your data store.

To interact with your application's state, the State Management building block provides various CRUD operations and the ability to query and filter your key/value state. You can also use bulk and transactional operations in your applications.

There's a lot more to State Management in Dapr, so I recommend that you take a look at the [documentation](https://docs.dapr.io/developing-applications/building-blocks/state-management/state-management-overview/) for the complete picture.

## Configuring Dapr in .NET 7

To see State Management in action, let's build an ASP.NET Web API that handles a catalog of books. We'll keep the model of our Books simple, and just focus on the APIs needed to interact with our state store in our API code.

To see the full code for this project, please check out [this repository](https://github.com/willvelida/dapr-resources/tree/main/StateManagement) on my GitHub.

To work with Dapr in an ASP.NET Core Web API, we need to install the ```Dapr.AspNetCore``` package. To do this, we can run the following .NET CLI command in our Web API project:

```bash
dotnet add package Dapr.AspNetCore
```

Alternatively, you can use the NuGet Package Manager in Visual Studio to install it

This pakcage will allow you to interact with Dapr applications through the Dapr Client and build routes and controllers in your ASP.NET applications using Dapr.

## Implementing logic to work with Dapr State Management

With Dapr installed, we can start to define an interface that we'll use to interact with our state store. This is going to be a simple CRUD application, so we can define a basic interface like so:

```csharp
using Bookstore.Api.Common.Dtos;
using Bookstore.Api.Common.Models;

namespace Bookstore.Api.Services.Interfaces
{
    public interface IBookService
    {
        Task<List<Book>> GetBooksByCategory(string category);
        Task<Book?> GetBookById(string id);
        Task<string> CreateBook(BookDto bookDto);
        Task<bool> DeleteBook(string id);
        Task<bool> UpdateBook(string id, BookDto bookDto);
    }
}
```

With our interface define, we can implement it and define the logic like so:

```csharp
using Bookstore.Api.Common.Dtos;
using Bookstore.Api.Common.Models;
using Bookstore.Api.Services.Interfaces;
using Dapr.Client;
using Microsoft.Extensions.Logging;

namespace Bookstore.Api.Services
{
    public class BookService : IBookService
    {
        private static string STORE_NAME = "statestore";
        private readonly DaprClient _daprClient;
        private readonly ILogger<BookService> _logger;

        public BookService(DaprClient daprClient, ILogger<BookService> logger)
        {
            _daprClient = daprClient;
            _logger = logger;
        }

        public async Task<string> CreateBook(BookDto bookDto)
        {
            try
            {
                var book = new Book
                {
                    Id = Guid.NewGuid().ToString(),
                    Name = bookDto.Name,
                    Price = bookDto.Price,
                    Category = bookDto.Category,
                    Author = bookDto.Author
                };

                _logger.LogInformation($"Saving a new Book with Id: {book.Id} to state store");
                await _daprClient.SaveStateAsync<Book>(STORE_NAME, book.Id, book);
                return book.Id;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(CreateBook)}: {ex.Message}");
                throw;
            }
        }

        public async Task<bool> DeleteBook(string id)
        {
            try
            {
                _logger.LogInformation($"Deleting Book with Id: {id}");
                await _daprClient.DeleteStateAsync(STORE_NAME, id);
                return true;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(DeleteBook)}: {ex.Message}");
                throw;
            }
        }

        public async Task<Book?> GetBookById(string id)
        {
            try
            {
                _logger.LogError($"Getting Book with Id: {id}");
                var book = await _daprClient.GetStateAsync<Book>(STORE_NAME, id);
                return book;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(GetBookById)}: {ex.Message}");
                throw;
            }
        }

        public async Task<List<Book>> GetBooksByCategory(string category)
        {
            try
            {
                var query = "{" +
                    "\"filter\": {" +
                        "\"EQ\": { \"category\": \"" + category + "\" }" +
                    "}}";

                var queryResponse = await _daprClient.QueryStateAsync<Book>(STORE_NAME, query);

                var bookList = queryResponse.Results.Select(q => q.Data).OrderByDescending(q => q.Category).ToList();

                return bookList;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(GetBooksByCategory)}: {ex.Message}");
                throw;
            }
        }

        public async Task<bool> UpdateBook(string id, BookDto bookDto)
        {
            try
            {
                _logger.LogInformation($"Updating book with Id: {id}");
                var currentBook = await _daprClient.GetStateAsync<Book>(STORE_NAME, id);
                if (currentBook is not null)
                {
                    currentBook.Name = bookDto.Name;
                    currentBook.Price = bookDto.Price;
                    currentBook.Category = bookDto.Category;
                    currentBook.Author = bookDto.Author;
                    await _daprClient.SaveStateAsync<Book>(STORE_NAME, currentBook.Id, currentBook);
                    return true;
                }
                return false;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(UpdateBook)}: {ex.Message}");
                throw;
            }
        }
    }
}
```

Let's break this down.

I define a private variable named ```STORE_NAME```, which will be the name of my state store that this application will use. So when I save and retrieve state, Dapr will look for my state store component named *statestore* and use it to hold and retrieve my application state. I then define a private variable for both my ```DaprClient``` and ```ILogger``` that I pass as parameters to my constructor.

In my ```CreateBook``` method, I pass in a new ```BookDto``` object and save it to my state store when I call the ```SaveStateAsync<T>``` method. In this method, I pass in the name of my state store, the id of the Book which I will use as my key, and the value that I want to persist, which will be the book that we want to save.

The ```DeleteBook``` and ```GetBookById``` methods are straightforward. I pass in the id of the book I want to retrieve or delete, and then call either ```DeleteStateAsync``` or ```GetStateAsync<T>``` (where T is Book), passing in the Book Id and the name of my state store.

```GetBooksByCategory``` is a little different. Here, I'm passing in the category of our book (Romance, Fiction, Non-Fiction etc) and searching for all books that have that particular category. I then use a filter query to retrieve all the state objects that have a particular category. The [state query API](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-state-query-api/) is still in **alpha** stage, so it might have changed a bit by the time you read this, but you can use the API to retrieve, filter and sort the key/value data stored in your state store component.

Once we've created our query, we pass it as a parameter to the ```QueryStateAsync<T>``` method (where again, T is our Book class). I then return the results of that query as a list of Book objects.

Finally, the ```UpdateBook``` method is simple enough. Here we pass in the id of an existing book and the ```BookDto``` request object. We use the book id to retrieve our existing book (again, using ```GetStateAsync<T>```) and if the book is retrievable, we do some simple mapping of the new request to the current book object, and save it in our state store using the ```SaveStateAsync<T>``` method.

With our interface and service class defined, we can go head and register the service in our ```Program.cs``` file, along with our ```DaprClient```.

```csharp
// Add services to the container.
builder.Services.AddDaprClient();
builder.Services.AddSingleton<IBookService, BookService>();
```

Now that our service has been defined and registered, we can implement it in our API controller like so:

```csharp
using Bookstore.Api.Common.Dtos;
using Bookstore.Api.Common.Models;
using Bookstore.Api.Services.Interfaces;
using Microsoft.AspNetCore.Mvc;

namespace Bookstore.Api.Controllers
{
    [Route("api/books")]
    [ApiController]
    public class BookController : ControllerBase
    {
        private readonly IBookService _bookService;
        private readonly ILogger<BookController> _logger;

        public BookController(ILogger<BookController> logger, IBookService bookService)
        {
            _logger = logger;
            _bookService = bookService;
        }

        [HttpGet]
        [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(Book))]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> Get(string category)
        {
            try
            {
                var books = await _bookService.GetBooksByCategory(category);
                return Ok(books);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(Get)}: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }

        [HttpGet("{bookId}")]
        [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(Book))]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> GetBook(string bookId)
        {
            try
            {
                var book = await _bookService.GetBookById(bookId);

                if (book is not null)
                {
                    return Ok(book);
                }

                return NotFound();
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(GetBook)}: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }

        [HttpPost]
        [ProducesResponseType(StatusCodes.Status201Created)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> Post([FromBody] BookDto bookDto)
        {
            try
            {
                var bookId = await _bookService.CreateBook(bookDto);

                if (bookId is not null)
                {
                    return Created($"/api/books/{bookId}", null);
                }

                return BadRequest();
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(Post)}: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }

        [HttpPut("{bookId}")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> Put(string bookId, [FromBody] BookDto bookDto)
        {
            try
            {
                var updated = await _bookService.UpdateBook(bookId, bookDto);

                if (updated)
                {
                    return Ok();
                }

                return BadRequest();
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(Put)}: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }

        [HttpDelete("{bookId}")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> Delete(string bookId)
        {
            try
            {
                var deleted = await _bookService.DeleteBook(bookId);

                if (deleted)
                {
                    return Ok();
                }

                return NotFound();
            }
            catch (Exception ex)
            {
                _logger.LogError($"Exception thrown in {nameof(Delete)}: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }
    }
}
```

## Working with Dapr State Management components

Before we can test our API, we'll need to set up our State Management components. To do this, we can define our component using everyone's favorite markup language, YAML! 😁

For my state store, I'll be using Azure Cosmos DB. To define our state store component to use Cosmos DB, we can write the following:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: <cosmos-url>
  - name: masterKey
    value: "<primary-key>"
  - name: database
    value: bookstoredb
  - name: collection
    value: books
```

To define which datastore we will use for state management, we define this using the ```type``` field. So for Cosmos DB, we use the ```state.azure.cosmosdb``` type. In the ```metadata``` section, we configure things like connection strings, keys, table names etc. For Cosmos DB, we need to provide the URL, masterKey, database and collection name that our application will use for storing state.

Different data stores will use different metadata, so check out [this guide](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-get-save-state/#step-1-setup-a-state-store) on how to configure state store components and [this reference](https://docs.dapr.io/reference/components-reference/supported-state-stores/) to see all the different data stores that are supported in Dapr.

Another important point to state is that metadata values can contain secrets, which are viewed in plain text, unless you go with the recommended pattern of using a [secret store](https://docs.dapr.io/operations/components/component-secrets/). I'll cover secret stores in a future article, but *PLEASE* don't store secrets in plain text in your applications!

## Testing our API

With our state store component defined, we can now test the API. We can run our API either by the command line (using the Dapr CLI) or by configuring our ```launchSettings.json``` file to run it in Visual Studio. To run our application via the command line, we can use the following:

```bash
dapr run --app-id bookstore-api --app-port 7264 --dapr-http-port 3500 --app-ssl --resources-path ..\..\..\components\ -- dotnet run --launch-profile https
```

Let's break down the ```dapr run``` command:

The ```--app-id``` parameter sets the id for our application. This is used for service discovery (and for Service Invocation. I wrote [an article](https://www.willvelida.com/posts/dapr-service-invocation-container-apps/) on Dapr Service Invocation which I recommend you take a look at after reading this!)

I define the ```--app-port``` that the application will listen on using the HTTPS port that's defined in my ```launchSettings.json``` file and the ```--dapr-http-port``` for Dapr to listen on (I'm using the default of 3500). I also point the application to use my state store component folder in the ```--resources-path``` parameter so that the API will use Cosmos DB as my state store. I then run the application using ```dotnet run --launch-profile https```.

We can also configure our API to use Dapr in our ```launchSettings.json``` file like so:

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:6276",
      "sslPort": 44354
    }
  },
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5101",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:7264;http://localhost:5101",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "dapr": {
      "commandName": "Executable",
      "workingDirectory": "$(ProjectDir)",
      "executablePath": "dapr.exe",
      "commandLineArgs": "run --app-id bookstore-api --dapr-http-port 3500 --app-ssl --resources-path ../../../components/ --app-port 7264 -- dotnet run ."
    }
  }
}
```

Take a look at the ```dapr``` json object block. It's essentially the same as launching our application through the Dapr CLI, but this way we can launch it in Visual Studio, the same way as we would if we wanted to run an IIS Express server or run a https or http server for our API.

I'm using Swagger in my API to make testing this a little easier, but you can also call the endpoint using tools like Postman or curl.

Let's create a book to store in Cosmos DB. I'll call my ```/api/books``` endpoint with a POST request to store a new book. Should the request be valid, I'll get a 201 response like so:

![Testing our create endpoint](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p0naafjtbpgz6shahsbr.png)

With my new book created, I can see it in my Cosmos DB collection.

![Viewing our book in Cosmos DB](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/co1yfmgkqi2qs3zaze95.png)

Let's test our GET endpoint ```/api/books/category```. Here I'll pass in the *Fiction* category to retrieve all the fiction books in my collection:

![Retreiving our Fiction books](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wto8uuym1ekgzkjrvp4q.png)

I can also retrieve my book by making a GET request to the ```/api/books/{bookId}``` endpoint and pass in the id of the book:

![Getting Book by ID](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r0173ckp8bw7ttyaak4h.png)

To update the book, I can use the id of an existing book, a new request body containing the updated book and make a PUT request to the ```/api/books/{bookId}``` endpoint. In this request, I'm making an update to the price of my existing book, so I can make that request like so:

![Updating our Book](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0wpr3b9n1x56kd44t524.png)

Since the request was successful, I can see that the price has been updated in Cosmos DB.

![Our updated book in Cosmos DB](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o7tawh5ufjhg6zhoeaf2.png)

Finally, we can remove the book from our state store by making a DELETE request to our ```/api/books/{bookId}``` endpoint and pass in the id of the book that we want to delete.

![Deleting our book](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nx7ffqux25e0jlel6x80.png)

With the DELETE request being successful, we can see that the book has been deleted from our Cosmos DB collection.

![Our book has gone](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i6bjzkxam8n0nxx5uqa3.png)

## Conclusion

In this article, I introduced State Management in Dapr and how we can use it to store state in our distributed applications. I then demonstrated how you can use Dapr State Management in an ASP.NET Core Web API, configure our state store component and how we can run and test our API project as a Dapr application.

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️