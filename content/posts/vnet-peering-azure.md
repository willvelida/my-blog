---
title: "Configuring Virtual Network Peering in Azure"
date: 2025-01-20
draft: false
tags: ["Azure", "Networking", "Bicep", "AZ-700"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qx7eyq04o2e7p8sg9dz0.png
    alt: "Configuring Virtual Network Peering in Azure"
    caption: "In distributed Azure architectures, we need to be able to communicate between virtual networks that are located in different regions, subscriptions, tenants etc. in a secure manner. We can achieve this with virtual network peering!"
---

In distributed Azure architectures, it's necessary to split up your virtual network infrastructure into different parts. This may happen over different Azure regions, or different subscriptions. Even in networks that are distributed, we'll need a mechanism to communicate between these different networks. For this, we can use **virtual network peering**.

Virtual network peering enables us to connect two or more [virtual networks in Azure](https://www.willvelida.com/posts/implementing-azure-vnets/), whether they are in the same Azure region or not. The traffic between peered virtual networks is private, and they appear as one for connectivity purposes. Traffic between virtual machines in peered networks uses the Microsoft backbone infrastructure.

In this article, we'll take a deeper look at cross-virtual network connectivity with virtual network peering, what types of peering we can do, what the benefits of virtual network are, and how we can configure it using Bicep.

## Types of Peering

Azure supports two types of virtual network peering:

1. **Virtual Network Peering** - This is where you connect virtual networks within the same Azure region.
2. **Global virtual network peering** - This is where you connect virtual networks across Azure regions. A caveat to this is that you can peer virtual networks in any public cloud region or China cloud region, but not government ones. You *can* peer virtual networks between Azure government cloud regions.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ld9fv9o43wbd786f1a91.png)

## Benefits of virtual network peering

Virtual network peering gives us a low-latency, high-bandwidth connection between Azure resources that are deployed between different virtual networks.

Because we can peer virtual networks between different regions and subscriptions, this gives us the ability to securely transfer data between between virtual networks across Azure subscriptions, regions, even Microsoft Entra tenants.

There's also no downtime to resources in either virtual network when we create the peering, and afterwards. Network traffic between peered virtual networks is private, and kept on the Microsoft backbone network. No public internet, gateways, or encryption is required to communicate between the virtual networks.

## Connectivity in peered networks

The latency between virtual machines in peered regions in the same network is the same as the latency within a single network. Peering doesn't inflict any extract restrictions on bandwidth within the peering.

We can apply network security groups (NSGs) in either virtual network to block access to other virtual networks or subnets. When we configure the virtual network peering, we can open or close the network security group rules between virtual networks. By default, full connectivity is open.

We can configure a VPN gateway in the peered virtual network as a transit point. A peered virtual network will use the remote gateway to gain access to other resources. This is supported for both local and global VNet peering, but a virtual network can only have one gateway.

We can also use Gateway transits to communicate outside the peering. In this situation, we can use a subnet gateway to use a site-to-site VPN connection to connect to resources in a on-prem network, use a VNet-to-VNet connection to another virtual network, or use a point-to-site VPN to connect to a client.

Gateway transits allow peered virtual networks to share the gateway and get access to resources, without having to deploy a VPN gateway in the peer virtual network.

## Resizing peered virtual network address space

Once we've peered our virtual networks, we can resize the address space without incurring any downtime as our workloads scale. This works for both IPv4 and IPv6 address spaces.

After we resize the address space, we must sync the peer with the new address space. One thing we have to consider is that if one virtual network is peered with a classic virtual network, this isn't supported.

## Service Chaining

Taking this a step further, we can use **service chaining** to direct traffic from one virtual network to a gateway or virtual appliance in a peered network through user-defined routes (UDRs).

We configure UDRs that point to resources in peered networks as the *next hop* IP address. We can also use UDRs to point to virtual network gateways. 

This is useful when deploying hub-and-spoke networks where the hub virtual network hosts components like VPN gateways or network virtual appliances. All spoke virtual networks can then peer with the hub virtual network, and traffic will flow through VPN gateways or network virtual appliances that are hosted in the hub virtual network.

Peering between virtual networks allows the next hop in a UDR to be the IP address of a virtual machine in the peered virtual network or a VPN gateway. The gotcha here is that we can't do this if the UDR specifies that the next hop is an Azure ExpressRoute gateway.

## Globally Peered restrictions

If our virtual networks are peered globally, we need to bear these two constraints in mind.

First, resources in one virtual network won't be able to communicate with the frontend IP address of a basic load balancer (whether it's public or internal) in a globally peered virtual network.

Second, [services that use a basic load balancer won't work over global virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq?WT.mc_id=MVP_400037#what-are-the-constraints-related-to-global-virtual-network-peering-and-load-balancers).

## Configuring VNet peering in Bicep

Let's take a practical look at how virtual network peering works. For this tutorial, we're going to create two virtual networks within the same region, deploy two virtual machines (one to each virtual network), and then configure regional peering between the two VNets. We'll then test the peering by logging into our primary VM and attempt to communicate with our secondary VM from it.

This diagram explains what we're going to build:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fdn236z523uyjarioj2a.png)

If you want to follow along, you'll need the following:

- [An Azure Subscription](https://azure.microsoft.com/en-au/pricing/purchase-options/azure-account/search?ef_id=_k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&OCID=AIDcmmxbrcqs76_SEM__k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&gad_source=1&gclid=CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE&WT.mc_id=MVP_400037)
- [The AZ CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli?WT.mc_id=MVP_400037)
- [VS Code with the Bicep extension installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install?WT.mc_id=MVP_400037#visual-studio-code-and-bicep-extension)

We'll be deploying all our resources to the same resource group, so we can do this using the following AZ CLI command:

```bash
az group create --name <rg-name> --location <rg-location>
```

Let's focus on our primary resources. Here we want to create a virtual network with a virtual machine deployed to it that we can use to log into. Here's the Bicep template that we can define to achieve this:

```bicep
param location string = resourceGroup().location

// VNET 1 RESOURCES
param primaryVnetName string = 'vnet-prod-ae-wv-001'
param primaryVnetAddressPrefix string = '10.30.0.0/16'
param primarySubnetName string = 'PrimarySubnet'
param primarySubnetPrefix string = '10.30.10.0/24'
param primaryVMName string = 'vmprodaewv001'
param primaryNicName string = 'nic-prod-ae-wv-001'
param primaryNsgName string = 'nsg-prod-ae-wv-001'

var vmSize = 'Standard_DS1_v2'
var publicIpName = 'pip-prod-ae-wv-001'
var adminUsername = ''
var adminPassword = ''

resource primaryVnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: primaryVnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        primaryVnetAddressPrefix
      ]
    }
    subnets: [
      { 
        name: primarySubnetName
        properties: {
          addressPrefix: primarySubnetPrefix
        }
      }
    ]
  }

  resource primarySubnet 'subnets' existing = {
    name: primarySubnetName
  }
}

resource publicIP 'Microsoft.Network/publicIPAddresses@2024-05-01' = {
  name: publicIpName
  location: location
  sku: {
      name: 'Standard'
  } 
  properties: {
      publicIPAllocationMethod: 'Static'
  }
}

resource primaryNic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: primaryNicName
  location: location
  properties: {
      ipConfigurations: [
          {
              name: 'ipconfig1'
              properties: {
                  subnet: {
                      id: primaryVnet.properties.subnets[0].id
                  }
                  privateIPAllocationMethod: 'Dynamic'
                  publicIPAddress: {
                    id: publicIP.id
                  }
              }
          }
      ]
      networkSecurityGroup: {
        id: nsg.id
      }
  }
}

resource primaryVM 'Microsoft.Compute/virtualMachines@2024-07-01' = {
  name: primaryVMName
  location: location
  properties: {
      hardwareProfile: {
          vmSize: vmSize
      }
      osProfile: {
          computerName: primaryVMName
          adminUsername: adminUsername
          adminPassword: adminPassword
      }
      networkProfile: {
          networkInterfaces: [
              {
                  id: primaryNic.id
              }
          ]
      }
      storageProfile: {
          imageReference: {
              publisher: 'MicrosoftWindowsServer'
              offer: 'WindowsServer'
              sku: '2019-Datacenter'
              version: 'latest'
          }
          osDisk: {
              createOption: 'FromImage'
          }
      }
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2021-05-01' = {
  name: primaryNsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'default-allow-rdp'
        properties: {
          priority: 1000
          sourceAddressPrefix: '*'
          protocol: 'Tcp'
          destinationPortRange: '3389'
          access: 'Allow'
          direction: 'Inbound'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}
```

This creates a virtual machine with a NSG that enables us to log into the VM using RDP. **This is for demo purposes only as port 3389 is not secure!** - For production scenarios, please use [Azure Bastion](https://learn.microsoft.com/azure/bastion/bastion-overview?WT.mc_id=MVP_400037).

I have an example how you can configure Azure Bastion for virtual machines in Bicep [here](https://www.willvelida.com/posts/configure-public-ip-azure/#configuring-a-public-ip-address-for-azure-bastion-using-bicep).

Now let's create the resources for our secondary virtual network. This time, we don't want to create an NSG for this VM, as we'll be accessing the VM through our primary VM instead of directly:

```bicep
// VNET 2 Resources
param secondaryVnetName string = 'vnet-prod-ae-wv-002'
param secondaryVnetAddressPrefix string = '10.20.0.0/16'
param secondarySubnetName string = 'SecondarySubnet'
param secondarySubnetPrefix string = '10.20.20.0/24'
param secondaryVMName string = 'vmprodasewv001'
param secondaryNicName string = 'nic-prod-ae-wv-002'

// SECONDARY VNET RESOURCES
resource secondaryVnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: secondaryVnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        secondaryVnetAddressPrefix
      ]
    }
    subnets: [
      { 
        name: secondarySubnetName
        properties: {
          addressPrefix: secondarySubnetPrefix
        }
      }
    ]
  }

  resource secondarySubnet 'subnets' existing = {
    name: secondarySubnetName
  }
}

resource secondaryNic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: secondaryNicName
  location: location
  properties: {
      ipConfigurations: [
          {
              name: 'ipconfig1'
              properties: {
                  subnet: {
                      id: secondaryVnet.properties.subnets[0].id
                  }
                  privateIPAllocationMethod: 'Dynamic'
              }
          }
      ]
  }
}

resource secondaryVM 'Microsoft.Compute/virtualMachines@2024-07-01' = {
  name: secondaryVMName
  location: location
  properties: {
      hardwareProfile: {
          vmSize: vmSize
      }
      osProfile: {
          computerName: secondaryVMName
          adminUsername: adminUsername
          adminPassword: adminPassword
      }
      networkProfile: {
          networkInterfaces: [
              {
                  id: secondaryNic.id
              }
          ]
      }
      storageProfile: {
          imageReference: {
              publisher: 'MicrosoftWindowsServer'
              offer: 'WindowsServer'
              sku: '2019-Datacenter'
              version: 'latest'
          }
          osDisk: {
              createOption: 'FromImage'
          }
      }
  }
}
```

Now that we've defined our secondary resources, let's configure the peering between our primary and secondary virtual networks. To do this, we'll need to define the following resources in our Bicep template:

```bicep
resource peerToSecondary 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2024-05-01' = {
  name: 'peer-to-secondary'
  parent: primaryVnet
  properties: {
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: true
    allowGatewayTransit: false
    useRemoteGateways: false
    remoteVirtualNetwork: {
      id: secondaryVnet.id
    }
  }
  dependsOn: [
    primaryVM
  ]
}

resource peerToPrimary 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2024-05-01' = {
  name: 'peer-to-primary'
  parent: secondaryVnet
  properties: {
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: true
    allowGatewayTransit: false
    useRemoteGateways: false
    remoteVirtualNetwork: {
      id: primaryVnet.id
    }
  }
  dependsOn: [
    primaryVM
  ]
}
```

To define virtual network peerings in Bicep, we use the `Microsoft.Network/virtualNetworks/virtualNetworkPeerings` resource. Here, we provide the resource Id of our virtual networks to each peering, and enable virtual network access and virtual networking forwarding.

This will allow traffic from the VM in our primary virtual network to reach the VM that we've deployed in our secondary region. If we were using a gateway link, we would allow gateway transit as part of our peering, but since we're not doing that for this example, we can set this to `false`.

Now we can deploy our Bicep template using the following AZ CLI command:

```bash
az deployment group create --template-file .\main.bicep --resource-group rg-vnetpeering
```

Once our template has been successfully deployed, we can verify that the peering has been applied by looking at the **peerings** configuration for our virtual network. We should see all the virtual networks that have been successfully peered with this virtual network here:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zg6781b3029tsklxn595.png)

We can also verify that the peering has been applied by looking at the **address space** of our virtual network. We should see the *peered virtual network address space* appear in our address space for both our virtual networks:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gysmw3hfal28ramvmnkn.png)

While everything appears to be set up correctly, it's best if we test this. Log on to your primary virtual machine, and open up a **Administrator PowerShell terminal** and enter the following:

```powershell
Test-NetConnection <ip-of-your-secondary-vm> -port 3389
```

This command enables us to test the connectivity for the IP address of our secondary VM, and see if the peering has been successful and we can communicate with it from our primary VM.

If the connection is successful, you should see the following output:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/adc61x9a8a03d4ctojpe.png)

Congratulations! With just a few lines of Bicep, we've created two virtual networks and configured regional VNet peering between them so that resources in one virtual network can communicate with resources in the other without doing so over the public internet!

Once you're done exploring the resources, you can delete them by deleting the resource group with this AZ CLI command:

```bash
$ az group delete --name <rg-name>
```

## Wrapping up

If you've made it this far, well done! Understanding how peering works in Azure virtual networks is important, especially when it comes to designing landing zones and figuring out how various resources across different networks, subscriptions, regions etc. will communicate with each other in a secure manner.

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/azure-networking-samples)

Until next time, Happy coding! 🤓🖥️