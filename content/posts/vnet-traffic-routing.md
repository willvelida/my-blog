---
title: "Custom Routing in Azure Virtual Networks"
date: 2025-01-22
draft: false
tags: ["Azure", "Networking", "Bicep", "AZ-700"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i2q4fafk9l3qbcnuxoih.png
    alt: "Custom Routing in Azure Virtual Networks"
    caption: "To control traffic flow within Azure virtual networks, we can use custom routes to configure the routes that network traffic will flow through network virtual appliances"
---

In order to control traffic flow within our Azure virtual networks, we can use custom routes, and configure the routes to direct traffic through a network virtual appliance.

Azure automatically creates a route table for each subnet in our virtual networks, and adds system default routes to the table. We can override these default routes with custom routes and more custom routes to route tables.

In this article, we'll learn how routing in Azure works, how we can use custom routes to override the default routes, before implementing an example of custom routing using Bicep.

## Routing in Azure

Azure will automatically create a route table for each subnet within our Azure virtual network. Within this table, Azure will add *system default routes* which we can override some of those routes with *custom routes*, and add more custom routes to the route table. Azure will route outbound traffic from a subnet based on the routes in a subnet's route table.

Let's take a deeper look at the different between system routes and custom routes in Azure.

## System Routes

Azure will automatically create system routes and assign those routes to each subnet in our virtual network. We can't create these or remove them ourselves. What we can do is **override** some system routes with custom routes.

Azure will create default system routes for each subnet, and adds optional default routes to subnets when we use specific capabilities in Azure.

Each route contains an **address prefix** and a **next hop type**. When traffic leaving a subnet is sent to an IP address that's within the address prefix of a route, Azure will use the route that contains the prefix.

When we create our virtual network, Azure will create the following default system routes for each subnet within the virtual network:

| Source | Address Prefixes | Next hop type |
| ------ | ---------------- | ------------- |
| Default | Unique to the VNET | Virtual Network |
| Default | 0.0.0.0/0 | Internet |
| Default | 10.0.0.0/8 | None |
| Default | 192.168.0.0/16 | None |
| Default | 100.64.0.0/10 | None|

Let's take a look at these next hop types:

- **Virtual Network** - This routes traffic between address ranges within the address space of a virtual network. Azure will create a route with an address prefix which corresponds to each address range defined within the same address space of our virtual network. Azure will automatically route traffic between subnets using the routes created for each address range.

- **Internet** - This routes traffic specified by the address prefix to the Internet. Any address not specified by an address range within a virtual network will be routed by Azure to the internet (unless the destination is for an Azure service. In this case, traffic is routed over the backbone network). We can override this system route for the 0.0.0.0/0 address prefix with a custom route.

- **None** - Traffic routed to the None next hop type will be dropped instead of routing outside the subnet.

Azure also adds default system routes for any Azure capabilities that we enable. Depending on which capability we enable, optional default routes are added to either all subnets within a our virtual network, or in specific subnets. These include:

| Source | Address Prefixes | Next Hop Type | All Subnets or Some? |
| ------ | ---------------- | ------------- | -------------------- |
| Default | Unique to the VNet | VNet peering | All |
| VNet gateway | Prefixes advertised from on-perm via BGP, or configured in local network gateway | Virtual network gateway | All |
| Default | Multiple | `VirtualNetworkServiceEndpoint` | Only the subnet a service endpoint is enabled for |

New hop types?! Let's take a look at what these mean:

- **VNET peering** - Routes are added for each address range within the address space of each virtual network when we create a [VNet peering between two virtual networks](https://www.willvelida.com/posts/vnet-peering-azure/).

- **VNET gateway** - Azure will add one or more routes with Virtual network gateway as the next hop type when we add a virtual network gateway to a virtual network. This happens because the gateway adds the routes to the subnet. [There are limits to the number of routes that we can propagate to an Azure virtual network gateway](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits?toc=%2Fazure%2Fvirtual-network%2Ftoc.json&WT.mc_id=MVP_400037#networking-limits), so we should summarize on-prem routes to the largest address ranges possible when adding the gateway.

- **VirtualNetworkServiceEndpoint** - When we enable a service endpoint to a service, Azure will add the public IP address of that service to the route table. Service endpoints are enabled for individual subnets within a virtual network, so the route is only added to the table of a subnet that the service endpoint is enabled for. Now, the public IP addresses of services in Azure do change from time-to-time, but Azure will also manage the updates to the routing tables when this occurs.

## Custom routes

If we require greater control over the way network traffic is routed, we can override the default routes by using our own **user-defined-routes (UDR)**. This is useful when we need to ensure traffic between two subnets passes through a firewall appliance.

Each subnet can have 0 or one associated route table. When we create a route table and associate it to a subnet, the routes within the route table are combined with, or override, the default routes Azure adds to a subnet. If there are any conflicts, UDRs override the default routes.

When we create a UDR, we can specify the following next hop types:

- **Virtual appliance** - This is a VM that runs as a network application, such as a firewall. When we create a route with the virtual appliance hop type, we also specify a next hop IP address. This can be either the private IP address of a network interface that's attached to a VM, or the private IP of an Azure internal load balancer.

- **Virtual network gateway** - When we use a virtual network gateway, we use this hop type when we want traffic destined for specific address prefixes routed to the virtual network gateway. The virtual gateway itself must be of type **VPN**

- **None** - If you want to drop traffic to an address prefix, we specify the None next hop type rather than forwarding the traffic to a destination.

- **Virtual network** - Used when we want to override the default routing within a virtual network.

- **Internet** - Used when we want to explicitly route traffic destined to an address prefix to the Internet.

## Service Tags for user-defined routes

Instead of using an explicit IP range, we can specify a service tag as the address prefix for a User Defined Route. Service Tags represent a group of IP address prefixes from specific Azure services which Microsoft manages.

### Border Gateway Protocol

If we have a network gateway in our on-prem environment, we can exchange routes with a virtual network gateway in Azure using Border Gateway Protocol (BGP). BGP is a standard routing protocol that's used to exchange routing information among two or more networks.

BGP is used to transfer data between systems on the internet, such as different host gateways. We use this in Azure typically when we connect to an Azure datacenter using Azure ExpressRoute. We can also use BGP if we're connecting to an Azure virtual network using a VPN site-to-site connection.

## Routing network traffic in Bicep

Let's build a sample of creating our own routes to override the default routing in Azure. Custom routes are helpful when we want to route traffic between subnets through an NVA, so let's build the following:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fb8u4rdwy3dvgcls0ebq.png)

In this demo, we'll create a virtual network and all the subnets we need for our various components, 3 virtual machines, one public, one private, and a VM that we'll use as an NVA, an Azure Bastion host that we'll use to connect to the virtual machines, and a route table that we'll associate with a subnet, and then create a custom route to route traffic from one subnet to another through an NVA.

If you want to follow along, you'll need the following:

- [An Azure Subscription](https://azure.microsoft.com/en-au/pricing/purchase-options/azure-account/search?ef_id=_k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&OCID=AIDcmmxbrcqs76_SEM__k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&gad_source=1&gclid=CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE&WT.mc_id=MVP_400037)
- [The AZ CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli?WT.mc_id=MVP_400037)
- [VS Code with the Bicep extension installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install?WT.mc_id=MVP_400037#visual-studio-code-and-bicep-extension)

We'll be deploying all our resources to the same resource group, so we can do this using the following AZ CLI command:

```bash
az group create --name <rg-name> --location <rg-location>
```

The first thing we'll need to do is create a virtual network, Azure Bastion host, and subnets that will be part of our virtual network:

```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: publicVmSubnetName
        properties: {
          addressPrefix: publicVmSubnetPrefix
        }
      }
      {
        name: bastionSubnetName
        properties: {
          addressPrefix: bastionSubnetPrefix
        }
      }
      {
        name: privateVmSubnetName
        properties: {
          addressPrefix: privateVmSubnetPrefix
        }
      }
      {
        name: dmzSubnetName
        properties: {
          addressPrefix: dmzSubnetPrefix
        }
      }
    ]
  }
}

resource bastionPip 'Microsoft.Network/publicIPAddresses@2024-05-01' = {
  name: bastionIpName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource bastionHost 'Microsoft.Network/bastionHosts@2024-05-01' = {
  name: bastionHostName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'bastionIpConfig'
        properties: {
          subnet: {
            id: vnet.properties.subnets[1].id
          }
          publicIPAddress: {
            id: bastionPip.id
          }
        }
      }
    ]
  }
}
```

In our virtual network, we'll create 4 subnets. One for our Bastion Host, a DMZ subnet where we will deploy our NVA VM, and a Private subnet where we will deploy the virtual machines that we want to route traffic to. We also have a subnet for our Public VM.

We also create a public IP address for our Bastion host, as well as the Bastion host itself.

Now let's create our NVA Virtual Machine that will help with network functions. We'll be creating a NVA using a Ubuntu VM:

```bicep
// NVA Resources
resource nvaNic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: nvaNicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: vnet.properties.subnets[3].id
          }
          privateIPAllocationMethod: 'Dynamic'
        }
      }
    ]
    networkSecurityGroup: {
      id: nvaNsg.id
    }
    enableIPForwarding: true
  }
}

resource nvaVm 'Microsoft.Compute/virtualMachines@2024-07-01' = {
  name: nvaVmName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: nvaVmName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nvaNic.id
        }
      ]
    }
    storageProfile: {
      imageReference: {
        publisher: 'canonical'
        offer: 'ubuntu-24_04-lts'
        sku: 'server'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
      }
    }
  }
}

resource nvaNsg 'Microsoft.Network/networkSecurityGroups@2024-05-01' = {
  name: nvaNsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'default-allow-ssh'
        properties: {
          priority: 1000
          sourceAddressPrefix: '*'
          protocol: 'Tcp'
          destinationPortRange: '22'
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

We're also enabling IP forwarding in our NIC to route traffic through the NVA. We need to do this in Azure and the operating system of our VM (we'll do the latter part in a bit). For the Azure side, we enable IP forwarding in our NIC resource by setting the `enableIPForwarding` property to `true`. 

When IP forwarding is enabled, any traffic that's received by our NVA that's destined for a different IP address, won't be dropped and instead forwarded to the correct destination.

Now we're going to create our public and private virtual machines. One virtual machine will be integrated into the `PublicSubnet` and the other will be integrated into the `PrivateSubnet`. We'll use the same Ubuntu image for both virtual machines.

Let's start with our public machine. This VM will be used to simulate a machine that can be reached on the public internet.

```bicep
resource publicVm 'Microsoft.Compute/virtualMachines@2024-07-01' = {
  name: publicVMName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: publicVMName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: publicNic.id
        }
      ]
    }
    storageProfile: {
      imageReference: {
        publisher: 'canonical'
        offer: 'ubuntu-24_04-lts'
        sku: 'server'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
      }
    }
  }
}

resource publicNic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: publicVMNicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: vnet.properties.subnets[0].id
          }
          privateIPAllocationMethod: 'Dynamic'
        }
      }
    ]
  }
}
```

For our Private VM, the Bicep code is similar, but this time we'll deploy the VM to our private subnet.

```bicep
resource privateVm 'Microsoft.Compute/virtualMachines@2024-07-01' = {
  name: privateVMName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: privateVMName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: privateNic.id
        }
      ]
    }
    storageProfile: {
      imageReference: {
        publisher: 'canonical'
        offer: 'ubuntu-24_04-lts'
        sku: 'server'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
      }
    }
  }
}

resource privateNic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: privateVMNicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: vnet.properties.subnets[2].id
          }
          privateIPAllocationMethod: 'Dynamic'
        }
      }
    ]
  }
}
```

Let's deploy our resources to create the virtual network, subnets, bastion hosts, public ip, and the virtual machines. To do this, use the following AZ CLI command:

```bash
$ az deployment group create --resource-group <rg-name> --template-file .\main.bicep
```

Before we create our route table, we'll need to enable IP forwarding in our NVA Virtual Machine's operating system. To do this, navigate to your NVA VM in the Azure Portal. Once you're there, select **Connect** and then **Connect via Bastion**. Enter your username and password, and once you're logged in, enter the following bash command:

```bash
$ sudo vim /etc/sysctl.conf
```

This will open up a Vim editor on your `/etc/sysctl.conf` file. In the Vim editor, remove the `#` from the line `net.ipv4.ip_forward=1`. (You can do this by pressing the **Insert** key, uncommenting the line, then press **Esc**. To save and quit the file, enter **:wq** and press **Enter**). Close the Bastion session, and then restart your virtual machine.

Once your VM has restarted successfully, we can create the route table. In this route table, we want to define the route of the traffic through the NVA VM, associate the route table to the `PublicSubnet` where the public VM has been deployed.

```bicep
// ROUTE TABLE
resource routeTable 'Microsoft.Network/routeTables@2024-05-01' = {
  name: routeTableName
  location: location
  properties: {
    routes: [
      {
        name: 'to-private-subnet'
        properties: {
          nextHopType: 'VirtualAppliance'
          addressPrefix: vnet.properties.subnets[2].properties.addressPrefix
          nextHopIpAddress: nvaNic.properties.ipConfigurations[0].properties.privateIPAddress
        }
      }
    ]
  }
}

resource privateSubnet 'Microsoft.Network/virtualNetworks/subnets@2024-05-01' existing = {
  name: '${vnet.name}/${privateVmSubnetName}'
}

resource subnetRouteAssociation 'Microsoft.Network/virtualNetworks/subnets@2024-05-01' = {
  name: privateSubnet.name
  properties: {
    addressPrefix: privateVmSubnetPrefix
    routeTable: {
      id: routeTable.id
    }
  }
}
```

> [!NOTE]
> This isn't perfect Bicep code. I'm just doing this for demonstration purposes. I plan to make updates to these samples in the future and clean this up!

Deploy the template again using the following command:

```bash
$ az deployment group create --resource-group <rg-name> --template-file .\main.bicep
```

Now it's time to test the routing of network traffic! Let's start by testing from our public VM to our private VM.

Navigate to your public VM and connect to it via Bastion. Once you've logged on and see the terminal, enter the following command:

```bash
$ tracepath private-vm
```

You should see that there are two hops from our public vm to our private one, because the traffic is routed via our NVA.

Start a new Bastion session for our private VM, login, and run the following:

```bash
$ tracepath <name-of-public-vm>
```

This time, there's only one hop, which is our public VM. There's no custom route for this path, so Azure routes traffic directly between subnets

Once you're done with your resources, run the following AZ CLI command to delete the resource group, which will delete the resources:

```bash
$ az group delete --name <rg-name>
```

## Wrapping up

We've covered a lot in this article, but hopefully you now have a better understanding on how custom routing works in Azure, how system and custom routes work in Azure, and how we can define custom routes for greater network traffic control for our Azure environment.

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/azure-networking-samples)

Until next time, Happy coding! 🤓🖥️