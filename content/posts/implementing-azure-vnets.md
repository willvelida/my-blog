---
title: "Implementing a basic Azure Virtual Network with Bicep"
date: 2025-01-13
draft: false
tags: ["Azure", "Networking", "Bicep", "", "AZ-700"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7dpeywmzy6mv1ssfk9zq.png
    alt: "Implementing a basic Azure Virtual Network with Bicep"
    caption: "Using Bicep, we can create Azure Virtual Networks (and subnets) for our private networking needs in Azure"
---

Azure Virtual Networks (or VNETs) are the fundamental building block for private networks in Azure. We can built Azure VNETs that are similar to on-prem networks, with the benefit of Azure infrastructure.

We can create VNETs with their own CIDR block, and link them to other Azure VNETs and on-prem networks (providing that there's no overlap with CIDR blocks). We can also control DNS server settings, segmentation of VNETs into subnets, and more.

In this article, I'll talk about what Azure VNETs can do, how we can design our Azure VNETs, and implement VNETs using Bicep.

## What can Azure VNETs do?

Azure VNETs essentially enable Azure resources to securely communicate with each other, with the internet, or our on-prem networks.

All resources integrated with our VNET can communicate outbound to the internet by default. We can communicate with resources inbound by assigning public IP addresses or using a public Load balancer. We can also use these for outbound connections.

Virtual Machines, AKS, App Services etc. can connect to VNETs. We can use service endpoints to connect to other Azure resources such as storage accounts, and when we create a VNET, we provide a mechanism for our services within our VNET to communicate with each other in a direct and secure manner.

With connections to on-prem resources, we can use either Point-to-site virtual networks (VPNs), site-to-site VPNs, or ExpressRoute.

We can also use Azure VNETs to filter and route our network traffic. Using Network Security Groups (NSGs) and Network Virtual Appliances (NVAs), we can filter network traffic between subnets. By default, Azure routes traffic between subnets, connected VNETs, and the internet by default. To override this, we can implement route tables or border gateway protocol (BGP) routes.

## How can we design our Azure VNETs?

With VNETs, we can create multiple VNETs per region per subscription, and create multiple subnets within each virtual network.

When we create a VNet, we need to use address ranges that are enumerated in [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918), which are for private, nonroutable address spaces. The following addresses are:

- `10.0.0.0` - `10.255.255.255` (10/8 prefix)
- `172.16.0.0` - `172.31.255.255` (172.16/12 prefix)
- `192.168.0.0` - `192.168.255.255` (192.168/16 prefix)

In addition, we can't add the following address ranges:

- `224.0.0.0/4` (Multicast)
- `255.255.255.255/32` (Broadcast)
- `127.0.0.0/8` (Loopback)
- `169.254.0.0/16` (Link-local)
- `168.63.129.16/32` (Internal DNS)

Azure will assign a resource in a VNET a private IP address space from the address space that we provision. So say for example we deploy a VM into our VNet with a subnet address space `192.168.1.0/24`, the VM will be assigned a private IP like `192.168.1.4`.

> [!NOTE]
> Azure will reserve the first four and last IP address for a total of five IP addresses within each subnet. These addresses are x.x.x.0-x.x.x.3 and the last address of the subnet.

For example, if we take our address range of `192.168.1.0/24` from before, the following addresses will be reserved:

- `192.168.1.0`
- `192.168.1.1` (Reserved by Azure for the default gateway.)
- `192.168.1.2`, `192.168.1.3` (Reserved by Azure to map the Azure DNS IPs to the VNet space.)
- `192.168.1.255` (Network broadcast address.)

Subnets are a range of IP addresses in the Azure VNET. We can segment our VNETs into different subnet sizes, creating as many as we require.

We can then deploy Azure resources into a specific subnet. We can deploy subnets with a IPv4 subnet definition of `/29` up to `/2`. IPv6 subnets must be exactly `/64` in size. Each subnet that we deploy must have a unique address range that's specified using CIDR format. We must also bear in mind that certain Azure services require their own subnet.

We can use subnets for traffic management, and we can limit access to Azure resources to specific subnets using a VNET service endpoint. We can enable service endpoints for some subnets, and not for others.

When it comes to naming our VNETs, VNETs have a resource group scope. So say we have a network called vnet-prod-australiaeast-001 in one resource group, we can have another network with the same name in another. Subnets are scope to a single virtual network, so each subnet within our VNET must have a distinct name.

An Azure resource (such as a web app, SQL database etc), can only be created in a VNET that exists in the same region and subscription as the resource. However, if we need to connect VNETs that exist in different subscriptions and regions, we can do so (I'll cover VNET peering in more detail in a future blog post).

## How can we implementing our VNETs using Bicep?

Let's get our hands dirty and create a VNET with some subnets. We won't deploy any resources to integrate with our VNET just to keep things simple. All we'll do here is create a **single vnet with three subnets**. 

To get started, we'll need a resource group to deploy our VNET to. We can do that using the following AZ CLI command:

```bash
az group create --name <rg-name> --location <rg-location>
```

To define a VNET, we can do so with the following Bicep code:

```bicep
@description('The name of the virtual network')
param vnetName string

@description('The region where our virtual network will be deployed. Default is resource group location')
param location string = resourceGroup().location

@description('The tags that will be applied to the virtual network resource')
param tags object = {}

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
  }
}
```

This template defines a virtual network that will be deployed to the location of the resource group, and it has an address prefix of `10.0.0.0/16`. I also have the following `bicepparam` file that I'm using to define some parameters for the template:

```bicep
using 'main.bicep'

param vnetName = 'vnet-prod-australiaeast-001'
param tags = {
  Application: 'Sample'
  Type: 'Networking'
}
```

We can deploy the template using the following CLI command:

```bash
az deployment group create --resource-group <rg-name> --template-file .\main.bicep --parameters .\main.prod.bicepparam
```

Give it a couple of minutes, and you should see your new virtual network deployed to Azure!

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s02dnw84e5c8hvgumu7r.png)

Now let's create our three subnets within our virtual network. In Bicep, we can create by defining our subnets within the virtual network definition in our template.

> [!WARNING]
> Avoid creating subnets as child resources. This can cause downtime for your resources in subsequent deployments, or even failed deployments!

Wait, why does this happen? Well when we define subnets by using child resources, in our first deployment the virtual network is deployed, then each subnet is deployed. This is expected behavior, because the Azure Resource Manager will deploy each individual resource separately.

But what happens when we deploy the same Bicep file. The same deployment sequence will occur, but the virtual network will be deployed without any subnets configured on it, because the `subnets` property is empty. Then after the virtual network is deployed, the subnets are redeployed.

So in some situations, we'll get downtime during our deployment. In others, our deployment fails.

However, there may also be some situations where we need to access the resource ID of our subnet, so how can we do this **without** using child resources? As we define our subnets in our virtual network, we can use the `existing` keyword in Bicep to obtain a strongly typed reference to the subnet, and then retrieve its Id.

With all that in mind, here's our updated Bicep template:

```bicep
@description('The name of the virtual network')
param vnetName string

@description('The region where our virtual network will be deployed. Default is resource group location')
param location string = resourceGroup().location

@description('The tags that will be applied to the virtual network resource')
param tags object = {}

var subnet1Name = 'GatewaySubnet'
var subnet2Name = 'DatabaseSubnet'
var subnet3Name = 'CoreServicesSubnet'

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnet1Name
        properties: {
          addressPrefix: '10.0.0.0/24'
        }
      }
      {
        name: subnet2Name
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: subnet3Name
        properties: {
          addressPrefix: '10.0.2.0/24'
        }
      }
    ]
  }

  resource subnet1 'subnets' existing = {
    name: subnet1Name
  }

  resource subnet2 'subnets' existing = {
    name: subnet2Name
  }

  resource subnet3 'subnets' existing = {
    name: subnet3Name
  }
}

@description('The resource ID of subnet 1')
output subnet1Id string = vnet::subnet1.id

@description('The resource ID of subnet 2')
output subnet2Id string = vnet::subnet2.id

@description('The resource ID of subnet 3')
output subnet3Id string = vnet::subnet3.id
```

So in our virtual network, we are creating three subnets with different address prefixes. We then use the `existing` keyword so that we can retrieve the resource id for each subnet, and produce it as an output for our Bicep template, so that we can deploy Azure resources to a subnet using that Id if we want to.

To deploy the template, run the same command as before and you should see the three subnets created in your virtual network by navigating to **Settings** > **Subnets** in your virtual network resource:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rwbdcxeqgsdzmjs5iak4.png)

Bear in mind that this is a basic example. There's a whole lot more we can do with virtual networks and subnets in Bicep, which I'll go into more depth in future blog posts!

## Wrapping up

In this post, I talked about what Azure Virtual Networks are, what we can do with them, how we can design virtual networks, and how we can define and deploy them in Bicep. This was a very basic example designed to keep it simple.

As you get more familiar with networking resources in Azure, there are more complex deployment scenarios that companies of all sizes use to structure their networking resources, such as [hub and spoke](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke?tabs=cli).

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/azure-networking-samples)

Until next time, Happy coding! 🤓🖥️