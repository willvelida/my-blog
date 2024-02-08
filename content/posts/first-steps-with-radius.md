---
title: "Building our first Radius application on Azure Kubernetes Service"
date: 2024-02-07
draft: false
tags: ["Azure","Radius","Cloud Native", "CNCF", "Bicep", "Platform Engineering", "Kubernetes"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g7fd1s4qhv26hbevyfaj.png
    alt: "Building our first Radius application on Azure Kubernetes Service"
    caption: "Radius is a open-source cloud-native platform that bridges the gap between application developers and platform engineers to build and operate cloud-native applications."
---

Earlier on this week, I gave a presentation to two user groups on the Radius project. T

If you want to check them out you can view them here:

{{< youtube 2TogDyJHHJs >}}

{{< youtube vV9R3owCdcQ >}} 

Following these two talks, I wanted to write a blog post on setting up a basic Radius application on a Azure Kubernetes Service cluster, which is what we'll go through now 🙂

I hope after NDC Sydney, I'll have some time to do some YouTube content on Radius. But for now, let's get stuck in and build a Radius application from scratch. I'll start by giving a brief explanation of what Radius is, then we'll jump into creating an AKS cluster in Azure, setting up our local machine to use Radius, then build our first Radius application.

## What is Radius?

Radius is an open-source cloud-native application platform that enables collaboration between developers and operators to build cloud-native applications across private and public clouds.

Using Radius, developers will define their applications, and the dependencies that their application relies on. Operators define the environments that those applications will run on. Radius brings both developers and operators together to help them build and deploy apps and infrastructure that meets the requirements of both the application, and the environment that those applications need to run on.

Lots of enterprises (perhaps maybe the one you work for) attempting to build custom internal developer platforms to help standardize the way that they deploy and manage cloud-native applications. Radius attempts to provide an open-source solution that helps lower the barrier of entry to standardize the deployment and development of cloud-native applications.

With Platform Engineering still emerging from our experiences with DevOps and DevSecOps, we need tools, frameworks and platforms that can help enable collaboration between application developers, platform engineers and IT operators.

Radius provides concepts like Environments and Recipes that support collaboration between these roles. Platform Engineers can create templates to start development, IT operations can deploy Radius Environments and provide Recipes that conform to their organization's policies, and developers can focus on the architecture of their application.

Let's start developing our Radius application, and along the way I'll explain the core concepts behind Radius.

## Building our Radius application

To build our Radius application, we need to do the following:

- Create our AKS cluster
- Set up the tools we need to run our Radius application
- Create and run a new Radius environment and application on our AKS cluster
- Add components to our Radius application.

Let's get started!

### Creating our AKS cluster

First thing we'll need to do is create a new AKS cluster. We can do this easily by using the AZ CLI. Let's start by creating a new resource group and then deploying the AKS cluster in that resource group:

```bash
# Create the Resource Group
az group create --name <RG_NAME> --location <RG_LOCATION>

# Create the AKS Cluster
az aks create --resource-group <RG_NAME> --name <AKS_CLUSTER_NAME> --location <RG_LOCATION> --node-count 3
```

Just replace the values with your own, and provision the cluster in a Azure region closest to you.

Once your AKS cluster has been deployed, you'll need to grab the credentials for your cluster. We can get those by running this AZ CLI command:

```bash
az aks get-credentials --resource-group <RG_NAME> --name <AKS_CLUSTER_NAME>
```

Now that we have our AKS cluster, we'll also need to set up our machine with the tools needed to work with Radius.

### Setting up our local machine to use Radius

To work with Radius, we'll need to install two tools.

First up is the rad CLI. The rad CLI manages your applications, resources, and environments. To install it on your local machine in PowerShell, you can run this command:

```powershell
iwr -useb "https://raw.githubusercontent.com/radius-project/radius/main/deploy/install.ps1" | iex
```

You may need to refresh your $PATH environment variable to access the rad CLI:

```powershell
$Env:Path = [System.Environment]::GetEnvironmentVariable("Path","User")
```

On Linux/WSL, you can install the CLI with this:

```bash
wget -q "https://raw.githubusercontent.com/radius-project/radius/main/deploy/install.sh" -O - | /bin/bash
```

Once the installation has been completed, you can verify by running:

```bash
rad version

## Example output
RELEASE   VERSION   BICEP     COMMIT
0.30.0    v0.30.0   0.30.0    f5a4a551b95168fbc2f33358bc8551c2cedece54
```

The second tool we'll need is the VS Code extension. There's a Radius Bicep extension available in VS code that provides support for writing Radius resources in Bicep.

At the time of writing, this is a temporary fork of the official Bicep extension to support Radius. In the future (or present, depending on when you read this), this will be merged into the official Bicep repository.

That being said, you can only have one VS Code Bicep extension installed at a time, so you'll either have to disable the official extension on your existing VS Code, or you can do what I did and install VS Code Insiders, and just have the Radius Bicep extension running on that.

Whichever option you choose, to install the Radius Bicep extension, simply just search for Radius Bicep in the Extension tab in VS Code or the [Visual Studio marketplace](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.rad-vscode-bicep&ssr=false#overview).

![](https://docs.radapp.io/installation/vscode-bicep/images/radius-bicep.png)

Now that you have the tools installed, we can start to build our first Radius app!

### Creating a new Radius Environment and Application

The first thing we'll need to do is initialize a Radius Environment. Environments are where are applications are deployed, and they determine how an application runs on a particular platform. They act as *landing zones* for Radius Applications, and application deployed to environments inherit the container runtime, configuration, Recipes and other settings from the environment.

To initialize a new environment, we'll create a new directory, navigate into it, and then initialize a new environment. Using the command line, run the following:

```bash
mkdir radius-demo
cd radius-demo
```

With our directory created, we can initialize a new environment:

```bash
rad init
```

When asked if you want to create a new application, select *Yes*. ```rad int``` will set up a local development environment where the environment configuration is handled for you. This will also create a new file in your directory called ```app.bicep```, where your application will be defined.

We can view all our app's resources and relationships using the ```rad``` CLI. To view the full application definition, run the following:

```bash
rad app show myapp -o json
```

You should get output similar to the below:

```json
{
  "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/applications/radius-demo",
  "location": "global",
  "name": "radius-demo",
  "properties": {
    "environment": "/planes/radius/local/resourceGroups/default/providers/Applications.Core/environments/default",
    "provisioningState": "Succeeded",
    "status": {
      "compute": {
        "kind": "kubernetes",
        "namespace": "default-radius-demo"
      }
    }
  },
  "systemData": {
    "createdAt": "0001-01-01T00:00:00Z",
    "createdBy": "",
    "createdByType": "",
    "lastModifiedAt": "0001-01-01T00:00:00Z",
    "lastModifiedBy": "",
    "lastModifiedByType": ""
  },
  "tags": {},
  "type": "Applications.Core/applications"
}
```

In this JSON format, there's a couple of things we should note:

- The ``id`` property is what we refer to as the **Universal Control Plane** ID of the application. The Universal Control Plane is the service that performs the central functionality in Radius. It receives inbound HTTP traffic to the Radius API, and either serves the response itself, or routes the request to a cloud resource provider.
- The ``location`` is where your Application resides. At the time of writing, all application are deployed to the ``global`` location, but it'll live in the location where you deployed your cluster.
- The ``environment`` specifies the Radius Environment that the Applications will bind to at deployment. This is where the containers will run, and which namespace they will be deployed to.
- Finally, ``compute`` specifies the hosting platform where running services in the Application will run. Currently, Kubernetes is the only compute platform for now. When I was at Microsoft, when you created a new Radius app, it was exclusive to Azure Container Apps, so it's possible that we'll be able to provision Radius apps on Container Apps soon.

With our application deployed, we can use the CLI to see what's deployed. Run the following:

```bash
rad app connections
```

Since we haven't deployed anything, we should see the following:

```bash
Displaying application: radius-demo

(empty)
```

When we ran ```rad init```, a Bicep file was generated for us. We'll use this file to define all the resources we need for our application. Let's start to work with this file, add some resources, and deploy our application.

### Deploying and Running our basic Radius Application

In your directory, you should see the following scaffolded Bicep file:

```bicep
import radius as radius

@description('The Radius Application ID. Injected automatically by the rad CLI.')
param application string

resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      ports: {
        web: {
          containerPort: 3000
        }
      }
    }
  }
}
```

Let's go ahead and deploy this file using the rad CLI! Using the command line, and in the directory that your Bicep file is in, run the following:

```bash
rad deploy app.bicep
```

You should see your container being deployed:

```bash
Building .\app.bicep...
Deploying template '.\app.bicep' for application 'radius-demo' and environment 'default' from workspace 'default'...

Deployment In Progress...

..                   demo            Applications.Core/containers

Deployment Complete

Resources:
    demo            Applications.Core/containers
```

Now, if we run ``rad app connections`` again, we'll see the container that we just deployed, along with the Kubernetes resources that were created to run it:

```bash
Displaying application: radius-demo

Name: demo (Applications.Core/containers)
Connections: (none)
Resources:
  demo (apps/Deployment)
  demo (core/Service)
  demo (core/ServiceAccount)
  demo (rbac.authorization.k8s.io/Role)
  demo (rbac.authorization.k8s.io/RoleBinding)
```

With our application deployed, we can run the application by running the following command:

```bash
rad run app.bicep
```

This command will deploy our container, port-forward the application to 3000, and start a log stream:

```bash
Building .\app.bicep...
Deploying template '.\app.bicep' for application 'radius-demo' and environment 'default' from workspace 'default'...

Deployment In Progress...

..                   demo            Applications.Core/containers

Deployment Complete

Resources:
    demo            Applications.Core/containers

Starting log stream...

+ demo-5876f78f84-rkqz8 › demo
demo-5876f78f84-rkqz8 demo No APPLICATIONINSIGHTS_CONNECTION_STRING found, skipping Azure Monitor setup
demo-5876f78f84-rkqz8 demo Using in-memory store: no connection string found
demo-5876f78f84-rkqz8 demo Server is running at http://localhost:3000
demo-5876f78f84-rkqz8 demo [port-forward] connected from localhost:3000 -> ::3000
```

Once the log stream has started, we can navigate to http://localhost:3000 to view the container:

![](https://docs.radapp.io/tutorials/new-app/demo-landing.png)

From here we can see the Radius connections that have configured, the metadata for the container, and a page for a Todo List 😅

Navigate to the Todo list page, and you should see the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yoyewsagbvhp417wjrzk.png)

As you can see from the above, we haven't configured a database yet, so any Todo items that we persist will only be saved to memory. Let's create a database connections to we can save our items to that, rather than in-memory.

### Adding a database and backend container

First, let's add a MongoDB database to store our Todo items in. Using Radius Bicep, we can add dependencies like Dapr State Stores, MongoDB databases, Redis Caches and more. To add a Mongo Database, we can add the following in our ``app.bicep`` file:

```bicep
@description('The ID of your Radius Environment. Set automatically by the rad CLI.')
param environment string

resource mongodb 'Applications.Datastores/mongoDatabases@2023-10-01-preview' = {
  name: 'mongodb'
  properties: {
    environment: environment
    application: application
  }
}
```

All we've done here is add a MongoDB database using a Radius resource block. We haven't specified *where* or *how* to run the MongoDB database. This is where Recipes come into play.

Recipes enable a seperation of concerns between developers and IT operators by automating infrastructure deployment. As developers, we can select the resource we want (such as a MongoDB database) and IT operators can codify how these resources should be deployed and configured:

![](https://docs.radapp.io/guides/recipes/overview/recipes.png)

So in this example, a developer could simply say that they just want a Cache as part of their application. Platform Engineers define how that infrastructure is deployed and configured within their environment. Here, our Recipe states that this is a Redis Cache that's going to be deployed on Azure, with a Private Endpoint. It'll have a particular SKU and we'll also enable some diagnostic logging.

The advantage here is that developers who may not have a complete understanding of the cloud environment can simply state what it is their application requires, and Platform Engineers with a greater understanding of how the cloud works within their environment can provide Recipes for developers to consume.

To add a connection from our container to the MongoDB database we just defined, all we need to do is this to our container resource block:

```bicep
connections: {
      mongodb: {
        source: mongodb.id
      }
    }
```

So our complete ``demo`` container should look like this:

```bicep
resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      ports: {
        web: {
          containerPort: 3000
        }
      }
    }
    connections: {
      mongodb: {
        source: mongodb.id
      }
    }
  }
}
```

Our MongoDB database is now a defined connection in our application. To re-run the app, we simply run the ``rad run`` command. We should see the following output:

```bash
Building .\app.bicep...
Deploying template '.\app.bicep' for application 'radius-demo' and environment 'default' from workspace 'default'...

Deployment In Progress...

Completed            mongodb         Applications.Datastores/mongoDatabases
.                    demo            Applications.Core/containers

Deployment Complete

Resources:
    demo            Applications.Core/containers
    mongodb         Applications.Datastores/mongoDatabases

Starting log stream...
```

Navigate to localhost:3000, and we'll see that our MongoDB container is listed as a connection:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o6qjpzngppugo0jljqsr.png)

When we run ``rad app connections``, we should see MongoDB added as a new dependency, along with the underlying Kubernetes resources used to create them:

```bash
Displaying application: radius-demo

Name: demo (Applications.Core/containers)
Connections:
  demo -> mongodb (Applications.Datastores/mongoDatabases)
Resources:
  demo (apps/Deployment)
  demo (core/Secret)
  demo (core/Service)
  demo (core/ServiceAccount)
  demo (rbac.authorization.k8s.io/Role)
  demo (rbac.authorization.k8s.io/RoleBinding)

Name: mongodb (Applications.Datastores/mongoDatabases)
Connections:
  demo (Applications.Core/containers) -> mongodb
Resources:
  mongo-bzmp2btdgzez6 (apps/Deployment)
  mongo-bzmp2btdgzez6 (core/Service)
```

### Adding a gateway

Now every time we connect to localhost:3000, we are connecting directly with the container. Adding a gateway is a much better alternative for exposing our applications to the internet, and Radius provides a default recipe for a gateway. To add the gateway, we add it to our Bicep file with this resource:

```bicep
resource gateway 'Applications.Core/gateways@2023-10-01-preview' = {
  name: 'gateway'
  properties: {
    application: application
    routes: [
      {
        path: '/'
        destination: 'http://demo:3000'
      }
    ]
  }
}
```

Once this is added, we can use the ``rad deploy`` command to deploy our application.

We should see a public endpoint that has been created as part of our deployment:

```bash
Building .\app.bicep...
Deploying template '.\app.bicep' for application 'radius-demo' and environment 'default' from workspace 'default'...

Deployment In Progress...

Completed            mongodb         Applications.Datastores/mongoDatabases
Completed            gateway         Applications.Core/gateways
.                    demo            Applications.Core/containers

Deployment Complete

Resources:
    demo            Applications.Core/containers
    gateway         Applications.Core/gateways
    mongodb         Applications.Datastores/mongoDatabases

Public Endpoints:
    gateway         Applications.Core/gateways http://gateway.radius-demo.20.227.63.98.nip.io
```

When we navigate to it, we should be redirected to our application:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8gcj36btpvn1n6h2m7vr.png)

### Cleanup

We've successfully created our first Radius application!! Once you're done, you can delete the Radius environment from your AKS cluster:

```bash
rad env delete default --yes
```

If you want to delete the resource group that you created as part of this tutorial, run the following:

```bash
az group delete --name <RG_NAME>
```

## Conclusion

In this tutorial, we built our first Radius application on Azure Kubernetes Services! Hopefully you had a lot of fun, and learnt the basic concepts of Radius along the way!

I'll be doing more Radius content in the future, so keep your eyes peeled for that. In the meantime, I recommend that you take a look at the following resources:

- [Getting started with Radius](https://docs.radapp.io/getting-started/)
- [Radius Environments](https://docs.radapp.io/guides/deploy-apps/environments/overview/)
- [Running applications on Radius](https://docs.radapp.io/guides/deploy-apps/howto-deploy/howto-run-app/)
- [Radius Recipes](https://docs.radapp.io/guides/recipes/overview/)
- [Radius on GitHub](https://github.com/radius-project)
- [Radius Roadmap](https://github.com/orgs/radius-project/projects/8/views/1)

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️