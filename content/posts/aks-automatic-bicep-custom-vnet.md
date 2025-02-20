---
title: "Creating an AKS Automatic cluster with your OWN custom VNET in Bicep"
date: 2025-02-20
draft: false
tags: ["Azure", "AKS", "Bicep", "Kubernetes", "Azure Kubernetes Service", "AKS Automatic", "AKS Networking"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y5ui4m70d78jlmm316r4.png
    alt: "Creating an AKS Automatic cluster with your OWN custom VNET in Bicep"
    caption: "Bringing your own virtual network with AKS Automatic clusters is now supported!"
---

In this article, I'm going to show you how to deploy an [AKS Automatic Cluster](https://learn.microsoft.com/azure/aks/intro-aks-automatic?WT.mc_id=MVP_400037) within your own custom virtual network using Bicep.

- An Azure Subscription
- A code editor - I'm using Visual Studio Code!
- The AZ CLI
- kubectl
- A bash shell (VS Code has an integrated terminal, Windows terminal is also pretty neat)

If you don't know what AKS Automatic is, we'll cover that before we start. We'll then work through the Bicep code that we need to provision a cluster with our own virtual network.

> [!NOTE]
> I've also covered this content in a video on my YouTube Channel, so check it out here: https://youtu.be/NY28RELKVCs 

## What is AKS Automatic?

AKS Automatic gives Kubernetes administrators an streamlined mechanism to set up Azure Kubernetes Service with common configurations as default. Azure takes care of setting up the cluster, including looking after node management, scaling, security etc. that follows the AKS well-architected recommendations.

In terms of security, at the cluster level, AKS Automatic uses Azure Linux OS along with Automatic upgrades. Local access and SSH access is disabled, with Azure RBAC being enabled to access the Kubernetes API. Workload identity is enabled so that our developers can build applications that authenticate to Azure resources using Entra ID for passwordless experiences. Image cleaner add-on are already configured for you and the node resource group is locked down to prevent users from accidentally or intentionally making changes to the cluster.

There are also deployment safeguards configured for you, as well as both the Azure Policy and Azure Key Vault provider add-ons so that you can work with secrets and policies from day 1.

When it comes to networking, we can think of how Pods communicate with each other and how traffic ingresses and egresses from the cluster. For pod networking, AKS Automatic implements the Azure CNI overlay networking with Cilium. For the data plane, ASK App Routing add Is configured for ingress which is essentially a managed nginx Ingress controller which can integrate with Azure DNS. For Egress, AKS NAT Gateway is installed for scalable outbound connection flows. Service meshes aren’t enabled by default, but you can use either your own service mesh or Azure service mesh.

AKS Automatic autoscaling is enabled by AKS Node Autoprovisioning which is based on the open-source Karpenter project that automatically provisions and deprovision nodes based on workload demands. The cluster autoscaler routinely checks for underutilized nodes and scale accordingly to save you from wasting resources. AKS Automatic also adds KEDA and VPA add-ons. So for KEDA, we can scale our clusters based on events and metrics, and the VPA (Vertical Pod Autoscaler) will essentially help us automatically adjust resource requests based on actual usage.

Finally for observability (starting to get into the day 2 stuff), AKS Automatic integrates with Azure Managed Prometheus for metric collection, Container Insights for log collection, and Managed Grafana for visualization.

With Managed Grafana, it’ll come with several Kubernetes and Azure related dashboards already installed so you can see how the cluster is operating just by going into your Grafana dashboards.

> [!NOTE]
> At the time of writing, when provisioning AKS Automatic through Infrastructure-as-code, not all of these things are included. AKS Networking, Azure Policy, Auto-provisioning and Key Vault providers are configured, but observability features like Azure Managed Prometheus and Grafana are not. Watch this space!

## Creating our resources

Before we do anything, use the AZ CLI to login to your Azure Subscription. To do this, open up a `bash` terminal and run the following:

```bash
$ az login --use-device-code
```

For this tutorial, we'll be creating the following resources:

- An AKS Automatic Cluster
- A user-assigned managed identity
- A virtual network with 2 subnets
    - One subnet will be for our API Server.
    - The other will be for our cluster


### Before we start writing code....

To work with our AKS Automatic cluster with the AZ CLI, we'll need to install the `aks-preview` extension:

```bash
$ az extension add --name aks-preview
```

Once that's installed, we can go ahead and register the following flag using the AZ CLI:

```bash
$ az feature register --namespace Microsoft.ContainerService --name AutomaticSKUPreview
```

Once this is registered, refresh the registration of the feature by running the following:

```bash
$ az provider register --namespace Microsoft.ContainerService
```

### Creating our resource group

We'll also need a resource group to deploy our resources to. For this, we'll use the AZ CLI:

```bash
$ export RG_NAME='<RG-NAME>'
$ export LOCATION='<LOCATION>'
```

For `RG_NAME`, give your resource group a name, and for `LOCATION` choose an Azure Region that supports availability zones. Because I live in a land Down Under, I've chosen `australiaeast`. [Choose a region](https://learn.microsoft.com/azure/aks/availability-zones-overview?WT.mc_id=MVP_400037) that's close to you.

Once you've set those variable, use the AZ CLI to create your resource group:

```bash
$ az group create --name $RG_NAME --location $LOCATION
```

Your output should look like this:

```bash
{
  "id": "/subscriptions/<subscription-id>/resourceGroups/<RG-NAME>",
  "location": "<LOCATION>",
  "managedBy": null,
  "name": "<RG-NAME>",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

I've omitted my details for clarity.

With our resource group created, let's start to define our resources using Bicep. To keep things simple for now, we'll have a `main.bicep` file (where we will define our template) and a `main.bicepparam` file (where we will define our parameters that we pass to our template). 

### Creating a virtual network

Let's start by creating our custom virtual network! Write the following Bicep:

```bicep
@description('The location where all resources will be deployed. Default is the location of the resource group')
param location string = resourceGroup().location

@description('The name given to the virtual network')
param vnetName string

@description('The name given to the API server subnet')
param apiServerSubnetName string

@description('The name given to the cluster subnet')
param clusterSubnetName string

var addressPrefix = '172.19.0.0/16'
var apiSeverSubnetPrefix = '172.19.0.0/28'
var clusterSubnetPrefix = '172.19.1.0/24'

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
    subnets: [
      {
        name: apiServerSubnetName
        properties: {
          addressPrefix: apiSeverSubnetPrefix
          delegations: [
            {
              name: 'aks-delegation'
              properties: {
                serviceName: 'Microsoft.ContainerService/managedClusters'
              }
            }
          ]
        }
      }
      {
        name: clusterSubnetName
        properties: {
          addressPrefix: clusterSubnetPrefix
        }
      }
    ]
  }

  resource apiSubnet 'subnets' existing = {
    name: apiServerSubnetName
  }

  resource clusterSubnet 'subnets' existing = {
    name: clusterSubnetName
  }
}
```

When using a custom virtual network with AKS Automatic, we need to create and delegate our API server subnet to `Microsoft.ContainerService/managedClusters`. This grants our AKS cluster the permissions to inject the API server pods and internal load balancer into that subnet.

### Network Security Group

All traffic within the virtual network will be allowed by default. If we want to restrict traffic between subnets, we can create a Network Security Group (NSG) to do so:

```bicep
@description('The name given to the NSG for the API server subnet')
param apiServerNsgName string

resource apiServerNsg 'Microsoft.Network/networkSecurityGroups@2024-05-01' = {
  name: apiServerNsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowClusterToApiServer'
        properties: {
          access: 'Allow'
          direction: 'Inbound'
          priority: 100
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRanges: [
            '433'
            '4443'
          ]
          sourceAddressPrefix: vnet::clusterSubnet.properties.addressPrefix
          destinationAddressPrefix: apiSeverSubnetPrefix
        }
      }
      {
        name: 'AllowAzureLoadBalancerToApiServer'
        properties: {
          access: 'Allow'
          direction: 'Inbound'
          priority: 200
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '9988'
          sourceAddressPrefix: 'AzureLoadBalancer'
          destinationAddressPrefix: apiSeverSubnetPrefix
        }
      }
    ]
  }
}
```

These NSG rules define the following for the API Server Subnet:

- From our **API Server Subnet** to the **Cluster Subnet**, we define a inbound rule to enable communication between Nodes and the API server using the TCP protocol on port 443 and 4443.
- From our **API Server Subnet** to **Azure Load Balancer**, we define a inbound rule to enable communication between Azure Load Balancer and the API Server.

### Managed Identity

For password-less authentication to Azure Services, we'll need to create a user-assigned identity that we can assign permissions to:

```bicep
@description('The name given to the user-assigned identity')
param uaiName string

resource uai 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: uaiName
  location: location
}
```

### Network Contributor Role

Our managed identity needs the `Network Contributor` roles on our virtual network. If we didn't have this, our cluster would start throwing failures when auto provisioning our nodes, as well cause provisioning failures if our API server subnet lacked permissions.

```bicep
var networkContributorRoleId = resourceId('Microsoft.Authorization/roleDefinitions', '4d97b98b-1d4f-4787-a291-c67834d212e7')

resource networkContributorRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, vnet.id, networkContributorRoleId)
  scope: vnet
  properties: {
    principalId: uai.properties.principalId 
    roleDefinitionId: networkContributorRoleId
    principalType: 'ServicePrincipal'
  }
}
```

We define the `networkContributorRoleId` variable using the guid representing the **built-in Network Contributor Role** and then assign it to our managed identity. This will be a Service Principal.

### Creating our AKS Automatic

Now we can go ahead an create our AKS Automatic cluster:

```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2024-09-02-preview' = {
  name: aksClusterName
  location: location
  sku: {
    name: 'Automatic'
  }
  properties: {
    agentPoolProfiles: [
      {
        name: 'systempool'
        mode: 'System'
        count: 3
        vnetSubnetID: vnet::clusterSubnet.id
      }
    ]
    apiServerAccessProfile: {
      subnetId: vnet::apiSubnet.id
    }
    networkProfile: {
      outboundType: 'loadBalancer'
    }
  }
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${uai.id}': {}
    }
  }
}
```

There isn't much to our Bicep code here, lets' break it down:

- We set this cluster to be an *AKS Automatic* cluster by setting the value of our `sku` to `Automatic`.
- We create a single *System Node Pool* with 3 nodes and integrate it with our cluster subnet via its resource ID.
- We integrate our API Server with our subnet that we've created for our API Server.
- We attach our managed identity to the cluster via the `identity` resource block.

### Giving us admin rights over the cluster

Before we can connect to our cluster, we need to assign ourselves a role to be able to manage it. For this tutorial, I'm going to grant myself the `Azure Kubernetes Service RBAC Cluster Admin` role, but take a look at the [built-in roles for AKS](https://learn.microsoft.com/azure/aks/manage-azure-rbac?tabs=azure-cli&WT.mc_id=MVP_400037#aks-built-in-roles) and see which one works for you:

```bicep
@description('The user object Id')
@secure()
param userObjectId string

var aksClusterAdminRoleId = resourceId('Microsoft.Authorization/roleDefinitions', 'b1ff04bb-8a4e-4dc4-8eb5-8693973ce19b')

resource aksClusterAdminRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(subscription().id, resourceGroup().id, userObjectId, 'Azure Kubernetes Service RBAC Cluster Admin')
  scope: aks
  properties: {
    principalId: userObjectId
    principalType: 'User'
    roleDefinitionId: aksClusterAdminRoleId
  }
}
```

For this role assignment, we assign it to our `userObjectId`, which will be our principal Id. I'll cover how we can retrieve that in just a bit.

### Our Bicepparam file

For our various parameters, we need to provide values to them within our `main.bicepparam` file. Here's what I used for mine, but make sure you change them for your deployment!

```bicep
using 'main.bicep'

param aksClusterName = 'prod-wv-aks-auto-001'
param vnetName = 'vnet-wv-001'
param apiServerSubnetName = 'apiServerSubnet'
param clusterSubnetName = 'clusterSubnet'
param uaiName = 'uai-wv-001'
param apiServerNsgName = 'nsg-wv-apiserver-001'
param userObjectId = ''
```

To get the value of your `userObjectId`, we can run the following AZ CLI command:

```bash
$ export USER_ID="$(az ad signed-in-user show --query id -o tsv)"
```

We're saving this to a variable in our bash terminal so that we can use it when deploying the template, which we'll do now.

## Deploying our Bicep template

To deploy our resources to our resource group, run the following AZ CLI command:

```bash
$ az deployment group create --resource-group $RG_NAME --template-file main.bicep --parameters main.bicepparam userObjectId=$USER_ID
```

## Verifying everything's up and running

To manage our AKS Automatic cluster, we can use `kubectl` and AZ CLI commands. First we need to configure `kubectl` to connect to our cluster. We can do so by running the following command:

```bash
$ az aks get-credentials --resource-group <resource-group> --name <cluster-name>
```

We can verify the connection to our cluster by using the following `kubectl` command to return a list of cluster nodes:

```bash
kubectl get nodes
```

You should get the following output asking you to login:

```bash
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code AAAAAAAAA to authenticate.
```

Because we assigned ourselves the `Azure Kubernetes Service RBAC Cluster Admin` role, we should be able to access our cluster and get the following output:

```bash
NAME                                STATUS   ROLES   AGE     VERSION
aks-nodepool1-13213685-vmss000000   Ready    agent   2m26s   v1.28.5
aks-nodepool1-13213685-vmss000001   Ready    agent   2m26s   v1.28.5
aks-nodepool1-13213685-vmss000002   Ready    agent   2m26s   v1.28.5
```

## Deploying the AKS Store Demo App to our AKS Automatic Cluster

Now that we have our lab environment set up, let's deploy an application to it. For this, I'm going to use (or steal, depending on your viewpoint) the [AKS Store Demo application](https://github.com/Azure-Samples/aks-store-demo) provided by the AKS team.

First, let's use `kubectl` to create a namespace for the store application:

```bash
$ kubectl create ns aks-store-demo
```

Within the sample repository, there's a yaml file that'll create all the Kubernetes resources we need for our application. We can apply this using `kubectl apply`

```bash
$ kubectl apply -n aks-store-demo -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-ingress-quickstart.yaml
```

Give it a minute! The `kubectl apply` command will provision all the resources, and will create a node pool using [node auto provisioning](https://learn.microsoft.com/azure/aks/node-autoprovision?tabs=azure-cli&WT.mc_id=MVP_400037). We can run the following command to wait for a public IP address for us to access our application:

```bash
kubectl get ingress store-front -n aks-store-demo --watch
```

After a short while, we should see a valid IP address that has been assigned to the service:

```bash
NAME          CLASS                                HOSTS   ADDRESS        PORTS   AGE
store-front   webapprouting.kubernetes.azure.com   *       4.254.104.11   80      18m
```

Open up a browser and navigate to that IP address, and you should be able to access the application:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t29u9v8iwy7rj72sfw68.png)

## Wrapping up

Congratulations! You've successfully deployed an AKS Automatic cluster in your own custom network using Bicep and have managed to deploy an application to it.

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/aks-demos/tree/main/lab-env-bicep)

Until next time, Happy coding! 🤓🖥️