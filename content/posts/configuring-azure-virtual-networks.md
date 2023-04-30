---
title: "Building Your Network in the Cloud: A Beginner's Guide to Azure Virtual Networks"
date: 2023-04-29
draft: false
tags: ["Azure","Virtual Networks","Networking", "Bicep"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nonrtzm867lwmz3drol3.png
    alt: "Building Your Network in the Cloud: A Beginner's Guide to Azure Virtual Networks"
    caption: 'Azure virtual networks (VNets) enable cloud engineers to build secure and scalable networks in Azure'
---

## What are virtual networks?

As you look to move workloads to the cloud, it's important to have a secure and scalable network infrastructure that can support your applications. In Azure, this is where virtual networks come in.

Virtual networks (VNet for short) are the fundamental building block for your private network in Azure. They enable many types of Azure resources to communicate with each other, the internet and on-prem networks securely. You can also use VNets to filter and route network traffic. 

In this blog post, I'll start by talking about why we would use virtual networks and how we can plan our virtual networks to ensure they will support our architecture in Azure. I'll then show you how to create a basic virtual network using Bicep, before showing you how we can integrate Azure resources in a virtual network, using Azure Functions as an example.

Let's dive in!

## Why use virtual networks?

As I mentioned earlier, virtual networks enable Azure resources to communicate with other Azure resources, the internet and on-prem networks securely.

By default, all of our resources within a VNet can communicate outbound to the internet. If we assign a public IP address or use a public Load balancer, we can communicate inbound to a resource.

Securely communicating with other Azure resources can be achieved in the following ways:

- Deploying our resources to our virtual network (this can include AKS, VMs etc.)
- Through a service endpoint, which allow us to secure our Azure resources only to a virtual network.
- We can connect virtual networks to each other via VNet peering. This enables resources in different VNets to communicate through the VNet peering.

When it comes to communicating with on-prem networks, we can use Point-to-site VPNs to establish a link between a virtual network and a single computer in our on-prem network; Site-to-site VPN, which establishes links between on-prem VPN devices and an Azure VPN Gateway deployed in the virtual network and Azure ExpressRoute, which establishes private connections (meaning that traffic does not go over the internet) between your on-prem network and Azure through ExpressRoute partners.

Subnets in virtual networks allows you to implement logicial divisions in your VNet, which improves security and makes your virtual network more manageable. Each subnet contains a range of IP address that fall within the address space of your virtual network, and it must be unique within that address space. You can't overlap the range of one subnet with other subnet IP address in the same VNet and the IP address space for a subnet must be specified by using CIDR notation.

Azure will reserve 5 IP addresses for each subnet (the first four addresses and the last address). Taking the IP address range of 192.168.1.0/24 as an example, these addresses will be reserved by Azure:

- 192.168.1.0
- 192.168.1.1
- 192.168.1.2 and 192.168.1.3
- 192.169.1.255

We can filter traffic between subnets by using **Network Security Groups (NSGs)**, which contain multiple inbound and outbound security rules that enable you to filter traffic to and from resources by IP addresses, port and protocol, and **Network Virtual appliances**, which is a VM that performs a network function (such as a firewall).

You can use custom route tables to control where traffic is routed to for each subnet, and border gateway protocols to propagate your on-prem BGP routes to your virtual networks in Azure.

## Planning your virtual network

In the real world, you'll probably deploy various virtual networks to support your application workloads. This will require planning to ensure that your resources can connect with each other effectively and securely.

Virtual networks can be used in various different scenarios:

- **Dedicated private cloud-only VNets** - If you're just building for the cloud, you won't need to worry about communication cross-premises, but having a virtual network for your services can ensure your resources communicate directly and safely with each other in the cloud.
- **Extending your data center with VNets** - You can scale your on-prem datacenter capacity with virtual networks in Azure by using traditional site-to-site VPNs, providing a secure connection between Azure and your VPN gateway.
- **Hybrid cloud scenarios** - Virtual networks can support a range of hybrid cloud scenarios.

You'll also need to consider what topology you'll need for your virtual network. There's a variety of different options which I'll save the details for in another blog post, but some options include:

- **Hub-spoke** - This is where a 'hub' virtual network will host shared Azure services and is the central point for connectivity for cross-premises networks. 'Spoke' virtual networks isolate and manage workloads seperately for each spoke. Workloads hosted in the spoke can use shared services in the hub.
- **Mesh network** - This is where all virtual networks in a network group are connected to each other. They can pass traffic bi-directionally to one another and you can regional meshes (virtual networks in the same region) or global meshes (across all Azure regions).

I've deliberately left out copious amounts of details for the purposes of this article. It's important for me to stress that you'll need to give some careful consideration around planning your virtual networks to ensure that it suits the requirements of your architecture. That said, please take at the following resources when planning your virtual network:

- [Plan virtual networks](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-vnet-plan-design-arm)
- [Hub-spoke network topology in Azure](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke?tabs=cli)
- [Connectivity configuration in Azure Virtual Network Manager](https://learn.microsoft.com/en-us/azure/virtual-network-manager/concept-connectivity-configuration#mesh-network-topology)

## Planning IP addressing

It's also importnant to plan your IP addresses in Azure. This will ensure that you don't overlap IP address space in Azure with on-prem locations, as this will create major contention issues.

You can assign IP addresses to Azure resources to communicate with other resources in Azure, your on-prem network and the internet.

Public IP address allow your resources in Azure to communicate with the internet. Private IP address enabloe communication within a VNet and your on-prem network. IP address can be statically or dynamically assigned and you can seperate these resources into different subnets.

Again, I've kept this brief for simplicity. For more information, check out the following:

- [Plan for IP addressing](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-ip-addressing)

## Creating virtual networks with Bicep

Provisioning a basic virtual network with a subnet in Bicep is pretty straightforward. Take a look at the following Bicep code:

```bicep
@description('Random suffix to apply to our resources')
param appName string = uniqueString(resourceGroup().id)

@description('The location that our resources will be deployed to')
param location string = resourceGroup().location

@description('The name of our virtual network')
param virtualNetworkName string = 'vnet-${appName}'

var subnetName = 'MySubnet'

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2022-09-01' = {
  name: virtualNetworkName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.0.0/24'
        }
      }
    ]
  }
}
```

Here, we have a resource for our virtual network that reserves a list of address blocks with the CIDR range of 10.0.0.0/16. We also have a subnet with an address prefix of 10.0.0.0/24. We can verify the address space of our VNet by looking in the portal:

![Diagram showing virtual network blade](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gff64v14djgmvg0jh6ew.png)

We can also verify that our subnet has been created successfully with the address space that we defined, along with the address range that the subnet has been allocated:

![Diagram showing the subnet created within our virtual network](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cto8pobsqi21e8uf8c5v.png)

## Integrating Azure Functions with Virtual Networks

Creating a basic virtual network is simple enough, but it's better to illustrate an example of integrating a resource with our virtual network.

For this, we'll deploy an Azure Function Premium plan and integrate with out virtual network. Take a look at the following:

```bicep
@description('Random suffix to apply to our resources')
param appName string = uniqueString(resourceGroup().id)

@description('The location that our resources will be deployed to')
param location string = resourceGroup().location

@description('The name of our virtual network')
param virtualNetworkName string = 'vnet-${appName}'

@description('The name of the Azure Function App.')
param functionAppName string = 'func-${appName}'

@description('The name of the Storage Account')
param storageAccountName string = '${appName}azfunc'

@description('The name of the App Service Plan')
param appServicePlanName string = '${appName}asp'

@description('The name of the App Insights workspace')
param appInsightsName string = '${appName}ai'

@description('The language worker runtime to load in the function app.')
@allowed([
  'dotnet'
  'node'
  'python'
  'java'
])
param functionWorkerRuntime string = 'dotnet'

@description('Specifies the OS used for the Azure Function hosting plan.')
@allowed([
  'Windows'
  'Linux'
])
param functionPlanOS string = 'Windows'

var subnetName = 'MySubnet'
var isReserved = ((functionPlanOS == 'Linux') ? true : false)

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2022-09-01' = {
  name: virtualNetworkName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.0.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
        }
      }
    ]
  }

  resource subnet 'subnets' existing = {
    name: subnetName
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'Storage'
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: appServicePlanName
  location: location
  sku: {
    tier: 'ElasticPremium'
    name: 'EP1'
    family: 'EP'
  }
  properties: {
    maximumElasticWorkerCount: 20
    reserved: isReserved
  }
  kind: 'elastic'
}

resource appInsight 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource functionApp 'Microsoft.Web/sites@2022-09-01' = {
  name: functionAppName
  location: location
  kind: (isReserved ? 'functionapp,linux' : 'functionapp')
  properties: {
    reserved: isReserved
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: appInsight.properties.InstrumentationKey
        }
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccountName};EndpointSuffix= ${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccountName};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value};'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: toLower(functionAppName)
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: functionWorkerRuntime
        }
      ]
    }
  }
  dependsOn: [
    virtualNetwork
  ]
}

resource functionAppVNetConfig 'Microsoft.Web/sites/networkConfig@2022-09-01' = {
  name: 'virtualNetwork'
  parent: functionApp
  properties: {
    subnetResourceId: virtualNetwork::subnet.id
    swiftSupported: true
  }
}

```

There's a lot going on here, so let's focus on the important stuff:

```bicep
subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.0.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
        }
      }
    ]
```

For our subnet, we're injecting our Function app into our virtual network by delegating the *Microsoft.Web/serverFarms* service into our subnet. Subnet delegation enables us to designate a specific subnet for an Azure PaaS service that needs to be injected into the virtual network. When we do this, we allow that service to establish basic network configuration rules for that specific subnet which allow the service to work properly.

```bicep
resource functionAppVNetConfig 'Microsoft.Web/sites/networkConfig@2022-09-01' = {
  name: 'virtualNetwork'
  parent: functionApp
  properties: {
    subnetResourceId: virtualNetwork::subnet.id
    swiftSupported: true
  }
}
```

In our network config resource, we use the subnet resource id to indicate that this is the subnet that our function app will join. We have to make sure that the subnet that we join the function app to already has the *Microsoft.Web/serverFarms* delegation defined first.

Our full Bicep template will deploy the following resources shown in the diagram below:

![Diagram showing resources that will be deployed.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6v4n0fgxdkbt1uyoou4j.png)

There's a lot more we can do to secure our Azure Function resources (for example, putting private endpoints on our Storage account). But for now, this is good enough.

Now if we take a look at our subnet, we can see that the *Microsoft.Web/serverFarms* delegation has been applied.

![Subnet showing delegation of web server farms](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w5wrv4asm1arky2cr3uq.png)

Looking at our virtual network, we can see that our new function app is using the virtual network

![Virtual network showins apps that are integrated](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dcgxciawzc9xhfaz0rus.png)

Within our function app, we can also verify that it's been integrated with our subnet, securing our outbound traffic.

![Azure Function networking blade](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/syq8nik9fnu5t7gin967.png)

## Conclusion

In this blog post, we talked about why we would use virtual networks and how we can plan our virtual networks to ensure they'll support our architecture in Azure. We finished by creating a basic virtual network using Bicep, and integrated an Azure function in it.

Hopefully this has given you a gentle introduction into virtual networks in Azure, the different scenarios on how we would use virtual networks to support our applications and how we can plan our virtual networks.

As always, if you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️