---
title: "How to create availability tests for App Service using Bicep"
date: 2023-01-26
draft: false
tags: ["Azure","App Service","Bicep", "App Insights", "Azure Monitor"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/68qwdfvjgx7001m2vkjl.png
    alt: "Availability tests in Application Insights"
    caption: 'Automate your availability tests in App Service with Bicep'
---

One important aspect of any web application is its availability. Azure provides many tools that ensures your application is up and running, one of them being availability tests, which allow you to periodically check the availability of your application. You can also generate alerts should your web application become unavailable.

You can configure availability tests and alerts in your Bicep templates, automating the entire process!

In this article, we'll create a Bicep template that provisions all the resources that we'll need (the web app, Application Insights instance, action groups etc.) and then disable our web application to trigger the alerts when our application is unavailable.

## Creating our resources in Bicep

### Application Insights

First, let's create our application insights workspace. Application Insights sends web requests to your applications at regular intervals from points around the world. App Insights will alert you if your application isn't responding, or if your application is responding too slowly.

To set up our app insights workspace, we create the following resource in Bicep:

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}
```

### Azure App Service

To create our web app, we'll need an app service plan and the web app itself. We'll include the App Insights connection string and instrumentation key to connect the web app to App Insights:

```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: skuName
    capacity: skuCapacity
  }
}

resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: webSiteName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      minTlsVersion: '1.2'
      appSettings: [
        {
          name: 'APPINSIGHTS_CONNECTIONSTRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: appInsights.properties.InstrumentationKey
        }
      ]
    }
  }
}
```

### Action Groups

Azure Monitor uses action groups to notify users about the alert and take an action. Action groups allow you to group together different actions that are taken in response to an alert.

This allows us to configure an manage a set of actions that are taken when alerts are triggered. This could include sending emails or SMS messages to on-call teams, calling webhooks or running a Logic App. This enables automated responses to critical events, and helps you troubleshoot and resolve issues quickly.

We can create an action group in Bicep like this:

```bicep
resource onCallActionGroup 'Microsoft.Insights/actionGroups@2022-06-01' = {
  name: actionGroupName
  location: 'global'
  properties: {
    enabled: true
    groupShortName: actionGroupName
    emailReceivers: [
      {
        name: actionGroupName
        emailAddress: actionGroupEmail
        useCommonAlertSchema: true
      }
    ] 
  }
}
```

This action group will send an email every time a alert is triggered using the [common alert schema](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema). This schema is a standardized format for alerts in Azure. Activity logs, metrics and log alerts each have their own specific email templates and webhook schema, but the common alert schema provides a standardized format for all alerts

Action groups are **global** services, so you don't deploy them to a specific region. This is great, because if one region goes down, the traffic is routed and processed by other regions.

For more information:

- [Create and manage action groups in the Azure portal](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)

### Availability Tests

Let's create our availability test. You can set these up for any HTTP or HTTPS endpoint that's accessible from the public internet. There are four types of availability tests that we can create:

- **URL Ping Tests**. We can create these tests through the portal to validate whether an endpoint is responding and measure the performance.
- **Standard Tests**. This is similar to the URL ping test, but we can include things like ensuring the TLS/SSL certificate is valid and specify the HTTP request verb.
- **Multi-step web tests**. You can use this to play back a sequence of web requests for more complex scenarios.
- **Custom tests**. This uses the [TrackAvailability()](https://learn.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.telemetryclient.trackavailability?view=azure-dotnet) method to send results back to App Insights.

For the purposes of this test, we're just going to create a Standard test:

```bicep
resource availabilityTest 'Microsoft.Insights/webtests@2022-06-15' = {
  name: webTestName
  location: location
  kind: 'standard'
  tags: {
    'hidden-link:${appInsights.id}': 'Resource'
  }
  properties: {
    Enabled: true
    Frequency: 300
    Timeout: 120
    Kind: 'standard'
    RetryEnabled: true
    Locations: [
      {
        Id: 'emea-ch-zrh-edge'
      }
      {
        Id: 'emea-se-sto-edge'
      }
      {
        Id: 'emea-au-syd-edge'
      }
      {
        Id: 'us-ca-sjc-azr'
      }
      {
        Id: 'apac-hk-hkn-azr'
      }
    ]
    Request: {
      RequestUrl: 'https://${appService.properties.defaultHostName}'
      HttpVerb: 'GET'
      ParseDependentRequests: false
    }
    ValidationRules: {
      ExpectedHttpStatusCode: 200
      SSLCheck: true
      SSLCertRemainingLifetimeCheck: 7
    }
    Name: webTestName
    SyntheticMonitorId: webTestName 
  } 
}
```

In this availability test, we're making a GET request to our App Service URL and expecting a 200 (HTTP OK) response. We're running our test in the following locations:

- Australia East
- UK West
- East Asia
- West US
- France South

From each location, we're running this test every 5 minutes, with a timeout value of two minutes (both expressed in seconds). If the response from our site hasn't been received within the timeout period, it counts as a failure. In this test, we're just calling an endpoint. You do have the option to parse dependent requests (such as calling images, scripts, style files etc.). If they haven't been received within the timeout period either, this will count as a failure.

You may be wondering what the tag is doing. When I first attempted to deploy the template without it, I got the following error:

```bash
"(BadRequest) A single 'hidden-link' tag pointing to an existing AI component is required. Found none."
```

This is used to associate the web availability test with your Application Insights resource.

I recommend that you review the following documentation to get a better understanding on how to configure standard availability tests (particularly the locations, that tripped me up a bit):

- [Standard test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-standard-tests)
- [Bicep documentation for Web Tests](https://learn.microsoft.com/en-us/azure/templates/microsoft.insights/webtests?pivots=deployment-language-bicep)

### Metric Rules

We have our availability test and action group configured, so we can now create an metric alert rule that will generate an alert when our availability test fails.

Metric alerts provides a way for us to get notified when metrics associated with our Azure resources cross a defined threshold.

Let's create a metric alert that will trigger every time our availability test fails in two locations.

```bicep
resource pingAlertRule 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: pingAlertRuleName
  location: 'global'
  properties: {
    actions: [
      {
        actionGroupId: onCallActionGroup.id
      }
    ]
    description: 'Alert for a web test'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.WebtestLocationAvailabilityCriteria'
      webTestId: availabilityTest.id
      componentId: appInsights.id
      failedLocationCount: 2
    }
    enabled: true
    evaluationFrequency: 'PT1M' 
    scopes: [
      availabilityTest.id
      appInsights.id
    ]
    severity: 1
    windowSize: 'PT5M'
  }
}
```

Like action groups, these are global resources. In this alert, we set our action group id as the action group to send this alert to. Since we've configured alerts for this action group to send emails every time an alert is generated, we will get an email.

In our criteria object, we're specifying that this will be a web test availability alert. Depeding on the type of test that you're running, the OData value will change. Check out the Bicep documentation for the differences. We then link our App insights resource id and the web test id to the test, along with the number of failed locations.

So if our availability test fails in two of the location we're running it in, the alert will be generated.

We then specify how often the metric alert is evaluated, what severity this alert should be, the resource ids that this alert is scoped to and the period of time that's used to monitor alert activity based on the threshold.

For more information, check out the following:

- [Create a metric alert for an Azure resource](https://learn.microsoft.com/en-US/Azure/azure-monitor/alerts/tutorial-metric-alert)
- [Bicep reference for metric alerts](https://learn.microsoft.com/en-us/azure/templates/microsoft.insights/metricalerts?pivots=deployment-language-bicep)

### Deploying our template

The complete Bicep template should look like this:

```bicep
@description('Which Pricing tier our App Service Plan to')
param skuName string = 'S1'

@description('How many instances of our app service will be scaled out to')
param skuCapacity int = 1

@description('Location for all resources.')
param location string = resourceGroup().location

@description('Name that will be used to build associated artifacts')
param appName string = uniqueString(resourceGroup().id)

var appServicePlanName = toLower('asp-${appName}')
var webSiteName = toLower('wapp-${appName}')
var appInsightName = toLower('appi-${appName}')
var actionGroupName = 'On-Call team'
var actionGroupEmail = '<YOUR-EMAIL-HERE>'
var webTestName = 'web-test'
var pingAlertRuleName = 'PingAlert-${appService.name}-${subscription().subscriptionId}'

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: skuName
    capacity: skuCapacity
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: webSiteName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      minTlsVersion: '1.2'
      appSettings: [
        {
          name: 'APPINSIGHTS_CONNECTIONSTRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: appInsights.properties.InstrumentationKey
        }
      ]
    }
  }
}

resource onCallActionGroup 'Microsoft.Insights/actionGroups@2022-06-01' = {
  name: actionGroupName
  location: 'global'
  properties: {
    enabled: true
    groupShortName: actionGroupName
    emailReceivers: [
      {
        name: actionGroupName
        emailAddress: actionGroupEmail
        useCommonAlertSchema: true
      }
    ] 
  }
}

resource availabilityTest 'Microsoft.Insights/webtests@2022-06-15' = {
  name: webTestName
  location: location
  kind: 'standard'
  tags: {
    'hidden-link:${appInsights.id}': 'Resource'
  }
  properties: {
    Enabled: true
    Frequency: 300
    Timeout: 120
    Kind: 'standard'
    RetryEnabled: true
    Locations: [
      {
        Id: 'emea-ch-zrh-edge'
      }
      {
        Id: 'emea-se-sto-edge'
      }
      {
        Id: 'emea-au-syd-edge'
      }
      {
        Id: 'us-ca-sjc-azr'
      }
      {
        Id: 'apac-hk-hkn-azr'
      }
    ]
    Request: {
      RequestUrl: 'https://${appService.properties.defaultHostName}'
      HttpVerb: 'GET'
      ParseDependentRequests: false
    }
    ValidationRules: {
      ExpectedHttpStatusCode: 200
      SSLCheck: true
      SSLCertRemainingLifetimeCheck: 7
    }
    Name: webTestName
    SyntheticMonitorId: webTestName 
  } 
}

resource pingAlertRule 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: pingAlertRuleName
  location: 'global'
  properties: {
    actions: [
      {
        actionGroupId: onCallActionGroup.id
      }
    ]
    description: 'Alert for a web test'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.WebtestLocationAvailabilityCriteria'
      webTestId: availabilityTest.id
      componentId: appInsights.id
      failedLocationCount: 2
    }
    enabled: true
    evaluationFrequency: 'PT1M' 
    scopes: [
      availabilityTest.id
      appInsights.id
    ]
    severity: 1
    windowSize: 'PT5M'
  }
}
```

To deploy our template, we can do so using the following AZ CLI commands:

```bash
az group create --name <resource-group-name> --location <location>
az deployment group create --resource-group-name <resource-group-name> --template-file main.bicep
```

Once that's completed, go into the Azure Portal and verify that this resources have been deployed. You should see something similar to the below:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8tjfoamnvkwryhe7zaan.png)

## Running our availability test

Navigate to the availability test resource and you should see some successful tests have been run since our app service is up and running. You can even view which regions are returning successful responses, how long the responses took and percentage of availability per region.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/68qwdfvjgx7001m2vkjl.png)

### Disabling our application

Let's generate an alert by stopping our app service. To do this, just go to your app service and stop it:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9cmkxg8gc9pdtb1regg4.png)

Navigate to the URL of your app service and you should see the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jh4e8vud3k66381u22y1.png)

Since our web app has stopped, we'll start to see some regions return unsuccessful responses in the web availability tests:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ldja4fipipapy8iii3nw.png)

Since every region has reported that our app is unavailable, this will trigger our alert because it's exceeded criteria of two locations.

### Viewing our email alert

The alert email look something like this:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p08im6rb1a7hdvaks0gg.png)

This email's using the common alert schema, so it'll include essential information, such as resource group, subscription, severity etc. As well as some context about the alert, which will depend on the type of alert that was generated.

## Conclusion

Hopefully this article has helped you understand how availability tests work and how we can create them in Bicep. This was a very basic scenario, but I'd encourage you to create all your alerts and action groups within your infrastructure code (you can do this in Terraform as well!). This makes life easier when standing up new environments and managing all your alerts, rather than doing through the portal.

On the Azure docs, I wrote an article on [creating monitoring resources by using Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/scenarios-monitoring), which talks about the various types of monitoring resources you can create in Bicep. Take a look at that if you need a reference on creating monitoring resources using Bicep!

If you have any questions about this article, or have any feedback, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️