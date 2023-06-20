---
title: "Building Simple Dapr Cron Jobs with ASP.NET Core Web APIs"
date: 2023-06-19
draft: false
tags: ["Dapr","Dotnet","ASP.NET Core"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nl0d9s1mrhs782o88n59.png
    alt: "Building Simple Dapr Cron Jobs with ASP.NET Core Web APIs"
    caption: "Using Cron job bindings in Dapr, we can run background workers and tasks on a schedule"
---

I've been thinking about migrating one of my side projects from Azure Functions to the Dapr framework to make it more portable. One of the most important components of this architecture is the ability to [refresh an authentication token that enables each Function app to retrieve data from an external API](https://medium.com/geekculture/building-a-token-refresh-service-for-the-fitbit-api-with-azure-functions-and-c-55027bf9d267). In an Azure Functions architecture, we can use timer triggers to perform scheduled jobs simply. 

The issue here is that Functions have an opinionated programming model. While this does make integrations between components simple, this can lock us into the Azure Functions hosting model, limiting the portability that I'd like my architecture to achieve.

The Dapr framework provides bindings that support cron jobs. Cron jobs uses a configurable interval schedule to trigger our application. This is great for performing scheduled tasks, such as background jobs or cleanup work that needs to be performed at a regular interval.

This article is going to cover how to configure a basic Dapr cron job in a ASP.NET Web API. I'll cover how to set up a cron binding component, configure the Web API to subscribe to the cron binding and then finally, run it to see it in action.

This sample is going to be super simple, so don't expect it to break the internet (or whatever the kids are saying this days). We're just here to see how Dapr cron jobs work at a basic level.

Let's dive in! 🚀

## Setting up our Cron Job

To set up our Cron Job, we create a YAML file with the following content:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: SayHello
spec:
  type: bindings.cron
  version: v1
  metadata:
  - name: schedule
    value: "@every 10s"
scopes:
  - say-hello
```

Let's break down the important parts here:

- The ```bindings.cron``` type specifies that we are defining a Dapr cron job.
- In the top level ```metadata``` property, we give our component a name of ```SayHello```. We need the endpoint of our ASP.NET Web API controller to match the name of this component, so we'll remember that for later.
- In the ```spec``` block, we're defining a schedule to run this Cron Job every 10 seconds.

Dapr Cron binding supports the following formats:

| Character | Descriptor | Acceptable Values |
| --------- | ---------- | ----------------- |
| 1 | Second | 0 to 59, or * |
| 2	| Minute | 0 to 59, or * |
| 3	| Hour | 0 to 23, or * (UTC) |
| 4	| Day of the month | 1 to 31, or * |
| 5	| Month | 1 to 12, or * |
| 6	| Day of the week | 0 to 7 (where 0 and 7 represent Sunday), or * |

So you could have the following cron jobs defined like so:

* ```30 * * * * *``` - every 30 seconds
* ```CRON_TZ=America/New_York 0 30 04 * * *``` - every day at 4:30am New York time

Dapr also supports shortcuts. So in my components, I've used ```@every 10s``` to run our Cron Job every 10 seconds. You can also use ```@daily``` or ```@hourly``` for daily and hourly jobs.

To learn more about the Cron binding spec in Dapr, check out the [documentation](https://docs.dapr.io/reference/components-reference/supported-bindings/cron/).

## Configuring our Web API as a Cron Job

Lets set up our Web API that will run on a schedule:

```csharp
using Dapr.Client;
using Microsoft.AspNetCore.Mvc;

namespace SayHello.Api.Controllers
{
    [Route("SayHello")]
    [ApiController]
    public class HelloController : ControllerBase
    {
        private readonly ILogger<HelloController> _logger;
        private readonly DaprClient _daprClient;

        public HelloController(ILogger<HelloController> logger, DaprClient daprClient)
        {
            _logger = logger;
            _daprClient = daprClient;
        }

        [HttpPost]
        public IActionResult Post()
        {
            _logger.LogInformation($"Hello Dapr Cron Job! The time is {DateTime.Now.ToString("dd-MM-yyyy HH:mm:ss")}");

            return Ok();
        }
    }
}
```

This is just a basic API controller that we've defined here. The important point here is that we've given our ```Route``` the name *SayHello*. This matches the name of our component of our Cron job that we defined earlier.

With that defined, our endpoint that we will listen to matches the name of the component of the Cron job. We then set up a POST request that logs a message to the console every 10 seconds by the Dapr sidecar.

Even though we aren't using the DaprClient in our code, we'll need to install it in our Web API project and then register the service in our ```Program.cs``` file like so:

```csharp
// Program.cs - Rest of file left out for brevity.
// Add services to the container.
builder.Services.AddDaprClient();
builder.Services.AddControllers();
```

If you don't install the ```Dapr.AspNetCore``` package, the API won't be able to subscribe to the Cron binding that we've defined earlier.

## Running our Cron Job

To run our cron job, we can run it using the Dapr CLI:

```bash
dapr run --app-id say-hello --app-port 7042 --dapr-http-port 3500 --app-ssl --resources-path ../../../components/ -- dotnet run --launch-profile https
```

I've given my Dapr application an id of ```say-hello```. The ```--app-port``` is the HTTPS port of my ASP.NET Web API found in my ```launchSettings.json``` file and the ```--resources-path``` parameter is pointing to the directory that my cron binding YAML component file is stored.

Once we've run our Dapr CLI command, we should see our message being logged to the console every 10 seconds like so:

![Our Web API logging a message every 10 seconds to the console](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/erlp1sv91b46ft76fry3.png)

## Conclusion

In this article, I talked about how we can use cron jobs in Dapr to run applications on a schedule, such as background workers or cleanup jobs. I showed you how you can configure your Cron job as a YAML component, wire it up to a ASP.NET Core Web API and then we ran our cron job to see it in action.

In the future, I'll be looking to build something a little more complex using Dapr cron jobs. Once I've got something cool to share, I'll let you y'all know! 😀

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️