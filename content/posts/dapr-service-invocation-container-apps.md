---
title: "Dapr Service Invocation with Azure Container Apps"
date: 2022-11-06T20:47:30+13:00
draft: false
tags: ["Azure Container Apps","Dapr","Bicep","Serverless","Azure","Containers", "Infrastructure as Code"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gvnxpn4jzpegh1ki8b31.png
    alt: "Azure Container Apps code with Dapr and ACA logo"
    caption: 'With integration with Azure Container Apps, you can invoke services using Dapr'
---

I had a bit of time last week to do some Dapr learning, so I started to read the [Dapr for .NET Developers e-book](https://learn.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/) that's available on our documentation (completely free by the way!). In one of the early chapters, the book outlines a tutorial that you can run locally to use service invocation to communicate between to two applications.

Running locally is fairly straightforward, so I wanted to deploy the two applications as Container Apps, since it has support for Dapr in the platform.

So this article's going to talk a little bit about Service Invocation in Dapr, how Container Apps supports Dapr, and how we can setup our Container Apps environments to support our Dapr applications.

## Service Invocation in Dapr

Dapr has several building blocks that we can use to build distributed microservices. Building blocks are HTTP or gRPC APIs that can be called from your application code. They help address the challenges we face when building microservices and they codify best practices and patterns.

One of these building blocks supports service invocation, which allows your application to reliably communicate with other applications using HTTP or gRPC. Service invocation acts like a reverse proxy that comes with service discovery, access control, metrics, retries and more.

For more information about Service Invocation in Dapr, [check out the documentation](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview/).

## Dapr in Azure Container Apps

You can build Dapr applications on Container Apps. You can enable Container Apps within your environment to use Dapr, and you can configure Dapr components at the environment level, which can be shared across multiple container apps.

We supply our Dapr-enabled container apps with an identifier that is used for service discovery, state encapsulation and pub/sub consumption.

For this example, we won't be configuring any Dapr components. With service invocation, we'll just be communicating between our applications using Dapr. If we were to use components for our container apps, we can scope our components to the specific Dapr applications that need to use those components.

For more on Dapr integration with Azure Container Apps, [check out the following](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview?tabs=bicep1%2Cyaml).

## Using Service Invocation in our app.

In my [sample code](https://github.com/willvelida/aca-dapr-service-invocation), I've got two C# projects: One is a ASP.NET core web app (acting as our front-end application) that will communicate to a ASP.NET Core Web API (which will be our back-end application) via Dapr. All it's doing is retrieving weather forecasts from the API using the Service Invocation building block.

To make this work, we'll need to make the following changes in our front end application. First we'll need to install the Dapr .NET SDK:

```powershell
Install-Package Dapr.AspNetCore
```

We'll then need to add the DaprClient in our Program.cs file:

```csharp
// Add services to the container.
builder.Services.AddDaprClient();
builder.Services.AddRazorPages();

// REST OF THE FILE
```

This will register the DaprClient with the ASP.NET Core dependency injection system. We can now inject our DaprClient instance into our code when we need to communicate with the service invocation building block.

We'll be presenting the weather forecast data on our home page, so the changes will need to be made in our Index.cshtml.cs file. In our OnGet() method, we'll need to make the following changes:

```csharp
public async Task OnGet()
        {
            var forecasts = await _daprClient.InvokeMethodAsync<IEnumerable<WeatherForecast>>(
                HttpMethod.Get,
                "mybackend",
                "weatherforecast");

            ViewData["WeatherForecastData"] = forecasts;
        }
```

The InvokeMethodAsync is making the service invocation call. Looking at the paramters:

- HttpMethod.Get = This is the HTTP Method that we'll be using against the service. In this case, we're making a GET request against our Container App.
- "mybackend" = This is the app Id of our Dapr application that we're calling, which we will set in our container app configuration. **Be mindful** that at the time of writing, container apps has a limitation that application names need to be all lowercase, so make sure you use the right name in this method.
- "weatherforecast" = This is the method name within our backend that we'll be invoking. This will be our "GetWeatherForecast" method within our backend api.

## Configuring our Bicep code to support Dapr

As I mentioned earlier, Dapr is enabled at the container app level. The Dapr APIs are exposed to each container app through a Dapr sidecar, which will be invoked from our container app via HTTP.

We'll enable Dapr for both our container apps in our Bicep template:

```javascript
var frontendName = 'myfrontend'
var backendName = 'mybackend'

// Omitted Bicep code

resource env 'Microsoft.App/managedEnvironments@2022-06-01-preview' = {
  name: containerEnvironmentName
  location: location
  tags: tags
  properties: {
   daprAIConnectionString: appInsights.properties.ConnectionString
   appLogsConfiguration: {
    destination: 'log-analytics'
    logAnalyticsConfiguration: {
      customerId: logAnalytics.properties.customerId
      sharedKey: logAnalytics.listKeys().primarySharedKey
    }
   } 
  }
}

resource frontend 'Microsoft.App/containerApps@2022-06-01-preview' = {
  name: frontendName
  location: location
  tags: tags
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      activeRevisionsMode: 'Multiple'
      ingress: {
        external: true
        transport: 'http'
        targetPort: 80
        allowInsecure: false
      }
      dapr: {
        enabled: true
        appPort: 80
        appId: frontendName
      }
      secrets: [
        {
          name: 'container-registry-password'
          value: containerRegistry.listCredentials().passwords[0].value
        }
      ]
      registries: [
        {
          server: '${containerRegistry.name}.azurecr.io'
          username: containerRegistry.listCredentials().username
          passwordSecretRef: 'container-registry-password'
        }
      ]
    }
    template: {
      containers: [
        {
          image: frontendImage
          name: frontendName
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Development'
            }
          ]
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: 5
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource backend 'Microsoft.App/containerApps@2022-06-01-preview' = {
  name: backendName
  location: location
  tags: tags
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      activeRevisionsMode: 'Multiple'
      ingress: {
        external: false
        transport: 'http'
        targetPort: 80
        allowInsecure: false
      }
      dapr: {
        enabled: true
        appPort: 80
        appId: backendName
      }
      secrets: [
        {
          name: 'container-registry-password'
          value: containerRegistry.listCredentials().passwords[0].value
        }
      ]
      registries: [
        {
          server: '${containerRegistry.name}.azurecr.io'
          username: containerRegistry.listCredentials().username
          passwordSecretRef: 'container-registry-password'
        }
      ]
    }
    template: {
      containers: [
        {
          image: backendImage
          name: backendName
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Development'
            }
          ]
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: 5
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

In our Container App environment, we're configuring our Application Insights instance to collect the telemetry that's going to be generated by Dapr when there's communication occurring between our different services.

In this example, I've used the connection string to connect to my Application Insights workspace. On March 31 2025, [support for instrumentation key ingestion in Application Insights will end](https://learn.microsoft.com/en-us/azure/azure-monitor/app/separate-resources#about-resources-and-instrumentation-keys), so get into the practice now of using the connection string. (Not Container App specific either).

Within our container apps, we configure Dapr like so:

```javascript
var frontendName = 'myfrontend'
var backendName = 'mybackend'

// frontend dapr config
dapr: {
  enabled: true
  appPort: 80
  appId: frontendName
  enableApiLogging: true
}

// backend dapr config
dapr: {
  enabled: true
  appPort: 80
  appId: backendName
  enableApiLogging: true
}
```

In both our container apps, we're setting the appId (Dapr application identifier) to the name of our container app. Recall that in our frontend code, we're making a service invocation call to our backend using the appId of 'mybackend', so this will need to be the appId of our backend application.

We're also enabling API logging for the Dapr sidecar. In Bicep, we have the option for setting the log level for the Dapr sidecar. I've kept it to the default of 'info' by not setting an explicit value, but you can define a level that suits your requirements. Bear in mind that this telemetry will be collected by Application Insights, and there might be extra associated costs with this.

To learn more about how to configure Dapr for your container apps in Bicep, check out the [Container Apps Environment](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/managedenvironments?pivots=deployment-language-bicep) and [Container Apps](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/containerapps?pivots=deployment-language-bicep) reference documentation.

## Monitoring our container apps

Taking a look at Application Insights, we can see the following telemetry being generated by Dapr:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mzngeo53m5qenr7e8fz5.png)

Here, we see that Dapr is making a GET request to our backend service to retrieve our weather forecast information. In our frontend code, we defined this GET call whenever we navigate to the home page. 'mybackend' is the Dapr Id for our Container App, and 'weatherforecast' the method that we want to invoke. We can see in the 'rpc.service' property that this call has been identified as a ServiceInvocation call to our 'service.name' myfrontend.

In Log Analytics, we can see the logs being emitted from our container app by running the following KQL query:

```kusto
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == 'myfrontend'
| project Time=TimeGenerated, AppName=ContainerAppName_s, Revision=RevisionName_s, Container=ContainerName_s, Message=Log_s
| take 100
```

There are two types of logs in Container Apps. Console logs, which are emitted by your app and System logs, which are emitted by the Container Apps service. Looking at our console logs for our frontend app (which in this case will be generated by our Dapr sidecar), we can see that our frontend is making a service invocation call to our backend:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i8uwzz6w9zaac3jo5mur.png)

For more information on logging in Azure Container Apps, take a look at the [following documentation](https://learn.microsoft.com/en-us/azure/container-apps/logging).

## Conclusion

In this article, we talked about how Service Invocation works in Dapr. We then talked about how Dapr integrates with Azure Container Apps, how we can configure Dapr in our container apps so we can invoke services in our code. Finally, we talked about how we can monitor our container apps, and see the logs generated by both our Container App and the Dapr sidecar.

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️