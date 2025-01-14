---
title: "Configuring Public IP addresses in Azure"
date: 2025-01-14
draft: false
tags: ["Azure", "Networking", "Bicep", "AZ-700"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wd1144jbeqca6br8r3dd.png
    alt: "Configuring Public IP addresses in Azure"
    caption: "Public IP address allow resources on the internet to communicate with Azure resources, and enable outbound communication for Azure resources to the internet."
---

[Azure Virtual Networks](https://www.willvelida.com/posts/implementing-azure-vnets/) use private IP addresses which aren't routable on public networks. To enable support networks that exist both in Azure and on-prem environments, we need to configure IP addressing for both networks.

Public IP addresses allow resources on the internet to communicate with Azure, and also enable outbound communication for Azure resources to public-facing services on the internet. In Azure, we can create public IP addresses and assign them to specific resources.

In this article, I'll talk about what Public IP addresses are, how we can allocate both dynamic and static IP addresses (and when we'd use one over the other), and then finish off with an example of configuring a Public IP address for an Azure Bastion that we can use to connect to a VM.

> [!NOTE]
> It's better to connect to a Azure VM using Azure Bastion over enabling port 3389 for RDP connections. I won't go into the specifics of Azure Bastion for this blog post, but we'll use a public IP address for it so we can connect to our VM without having to open up port 3389. [Learn more about Azure Bastion here](https://learn.microsoft.com/azure/bastion/bastion-overview?WT.mc_id=MVP_400037).

## What are Public IP addresses in Azure

Public IP addresses are used by internet resources to communicate inbound to resources in Azure. We can create Public IP addresses with either a IPv4 or IPv6 address. They can be either static or dynamically assigned, and are available in either *Basic* or *Standard* SKUs.

> [!NOTE]
> Basic SKU public IPs will be retired on September 30, 2025. I'm deliberately creating a Standard SKU public IP address to avoid the sample we'll create later going stale. But if you're reading this and are using the Basic SKU for Public IPs, check out the [official announcement](https://azure.microsoft.com/updates?id=upgrade-to-standard-sku-public-ip-addresses-in-azure-by-30-september-2025-basic-sku-will-be-retired&WT.mc_id=MVP_400037), and check out the [guidance on upgrading a basic public IP address to a Standard one](https://learn.microsoft.com/azure/virtual-network/ip-services/public-ip-basic-upgrade-guidance?WT.mc_id=MVP_400037).

We can associate public ip address resources with a variety of different Azure resources, such as Virtual Machines, Public Load Balancers, Azure Application Gateways, and more.

## Dynamic vs Static public IP addresses

Public IP addresses have two type of assignment, either static or dynamic.

**Static IP Addresses** - This is an assigned address that doesn't change over the lifespan of the Public IP resource. When we create a static IP, we can't specify the actual IP address that will be assigned to the public IP address resource, as Azure assigns the IP address from a pool of available IP addresses in the Azure region that the IP resource is created in.

**Dynamic IP Addresses** - These are addresses that change over the lifespan of the Public IP resource. The dynamic IP address is assigned when you associate it with a resource, and is released when you stop, or delete that resource. For example, if you assigned the dynamic IP address to a VM, it would be released when you stop that VM. In each Azure region, the public IP addresses are assigned from a unique pool of addresses.

## Public IP address prefixes

If we want to assign a static public IP address from a known range, we can use [Public IP address prefixes](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-address-prefix?WT.mc_id=MVP_400037) to do so. Any address that we create from the prefix can be assigned to any Azure resource that we want to use a standard public IP address for.

Once we delete the individual public IP address, it's returned to the reversed range that we can use later. Any IP addresses in our public IP address prefix are reserved for our use until we delete the prefix.

We can assign the following prefix sizes:

- /28 (IPv4) or /124 (IPv6) = 16 addresses
- /29 (IPv4) or /125 (IPv6) = 8 addresses
- /30 (IPv4) or /126 (IPv6) = 4 addresses
- /31 (IPv4) or /127 (IPv6) = 2 addresses

Say you have Azure virtual machines that you want to associate a public IP with. You can add the entire prefix with a single firewall rule, so as you scale the amount of VMs in Azure, you can associate IPs from the same prefix. This helps with the management overhead when adding IP addresses!

That being said, while you can specify which IP you want from the prefix, you can't actually specify the set of IP addresses for the prefix. You just provide the size of the prefix, and Azure will give the IP addresses for the prefixes based on that.

We also have to create Public IP addresses from the prefix in the same Azure region and subscription as the prefix, and assigned to resources in the same region and subscription.

## Configuring a Public IP address for Azure Bastion using Bicep

Let's get our hands dirty and create a Public Static IP address that we can use for an Azure Bastion host that we can use to connect to a virtual machine.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jw8ibor4kvg9iffaq539.png)

If you want to follow along, you'll need the following:

- [An Azure Subscription](https://azure.microsoft.com/en-au/pricing/purchase-options/azure-account/search?ef_id=_k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&OCID=AIDcmmxbrcqs76_SEM__k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&gad_source=1&gclid=CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE&WT.mc_id=MVP_400037)
- [The AZ CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli?WT.mc_id=MVP_400037)
- [VS Code with the Bicep extension installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install?WT.mc_id=MVP_400037#visual-studio-code-and-bicep-extension)

To get started, we'll need a resource group to deploy our VNET to. We can do that using the following AZ CLI command:

```bash
az group create --name <rg-name> --location <rg-location>
```

First thing we'll need is to create a virtual network with two subnets. One for the virtual machine, and another for the Bastion host:

```bicep
param vmName string = 'vmprodaewv001'
param location string = resourceGroup().location

var addressPrefix = '10.0.0.0/16'
var vmSubnetPrefix = '10.0.0.0/24'
var bastionSubnetPrefix = '10.0.1.0/24'
var vnetName = 'vnet-prod-ae-wv-001'
var vmSubnetName = 'VMSubnet'
var bastionSubnetName = 'AzureBastionSubnet'

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
                name: vmSubnetName
                properties: {
                    addressPrefix: vmSubnetPrefix
                }
            }
            {
                name: bastionSubnetName
                properties: {
                    addressPrefix: bastionSubnetPrefix
                }
            }
        ]
    }
}
```

> [!NOTE]
> When naming the subnet for your Azure Bastion host, you have to call it `AzureBastionSubnet`, otherwise the deployment will fail.

Now we can create the VM and NIC (Network Interface):

```bicep
param vmName string = 'vmprodaewv001'
param adminUsername string = '<your-username>'
param adminPassword string = '<your-password>'

var nicName = 'nic-prod-ae-wv-001'
var vmSize = 'Standard_DS1_v2'

resource nic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
    name: nicName
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

resource vm 'Microsoft.Compute/virtualMachines@2024-07-01' = {
    name: vmName
    location: location
    properties: {
        hardwareProfile: {
            vmSize: vmSize
        }
        osProfile: {
            computerName: vmName
            adminUsername: adminUsername
            adminPassword: adminPassword
        }
        networkProfile: {
            networkInterfaces: [
                {
                    id: nic.id
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

Here, we create a network interface and then attach it to the VM. For our network interface, we reference the virtual machine subnet within our virtual network and then use dynamic private IP allocation for it. 

We don't want public access for the VM by itself, we'll enable that through Bastion like so:

```bicep
var publicIpName = 'pip-prod-ae-wv-001'
var bastionHostName = 'bastion-prod-ae-wv-001'

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
                        id: publicIP.id
                    }
                }
            }
        ]
    }
}
```

In this code block, we define a Public IP address resource with a static allocation method, and a Standard SKU. This means that the IP address will remain the same over the lifespan of the Public IP resource. We then create our Bastion Host and configure it to be deployed to our `AzureBastionSubnet` with our Public IP address enabled for it.

We can deploy the template using the following AZ CLI command:

```bash
az deployment group create --resource-group <rg-name> --template-file .\main.bicep
```

Once our template has been deployed, we can navigate to our Bastion Host, and see that our Public IP address resource has been allocated to it:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0b3azfvulxw98krun0mz.png)

Navigating to our Public IP resource, we can see that it's associated with our Bastion resource:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g0de8ayeiqjltof4zr6d.png)

All we have to do now is head over to our VM and connect to it via Bastion. Under **Connect** in your VM resource, click on **Bastion**. You should see that your Bastion instance has been configured for this VM, and you can connect to the VM using its username and password.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b9aw3m5gqgjumjn5j3rp.png)

Because the Public IP address is configured for the Bastion host, you are making it available for outbound connections rather than the VM. If our login attempt was successful, we should be able to access our virtual machine:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xvcvl25zn54t3ckazy0f.png)

If you want to clean up your resources after you're done, just delete the resource group using this AZ CLI command:

```bash
az group delete --name <rg-name>
```

## Wrapping up

Hopefully you have a better understanding of Public IP addresses in Azure and how they work. If you want to take a look at the code we went through in this post, please check it out on my [GitHub](https://github.com/willvelida/azure-networking-samples).

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/azure-networking-samples)

Until next time, Happy coding! 🤓🖥️