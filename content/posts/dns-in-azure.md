---
title: "Understanding Private and Public DNS in Azure"
date: 2025-01-16
draft: true
tags: ["Azure", "Networking", "Bicep", "AZ-700"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ijfc1iuovtd3t31emx1f.png
    alt: "Understanding Private and Public DNS in Azure"
    caption: "DNS is split into two areas: Public, and Private DNS for resources accessible from your own internal networks. In Azure, we can deploy both public and private DNS zones to resolve resources exposed to the Internet, or services within our virtual networks."
---

To facilitate communication between resources in [Azure deployed in virtual networks](https://www.willvelida.com/posts/implementing-azure-vnets/), we can use domain name resolution over relying on IP address, making the communication process simpler. In Azure, DNS is split into two areas: [Public DNS](https://learn.microsoft.com/azure/dns/public-dns-overview?WT.mc_id=MVP_400037), and [Private DNS](https://learn.microsoft.com/azure/dns/private-dns-overview?WT.mc_id=MVP_400037).

Domain Name System, otherwise known as DNS, is responsible for resolving a service name to an IP address. Azure DNS provides DNS hosting, resolution, and load balancing for your Azure applications.

In this article, I'll talk about the differences between Public DNS Domains and how we can delegate DNS domains. Then i'll talk about how Private DNS works in Azure, and how we can set up Private DNS Zones in Azure.

## Azure Public DNS

Public DNS services resolve names and IP addresses for services over that are accessible over the internet. [Azure Public DNS](https://learn.microsoft.com/azure/dns/public-dns-overview?WT.mc_id=MVP_400037) is a hosting service for DNS domains that provides name resolution in Azure. Using Public DNS, we can manage our DNS records in the same way as we would manage our other Azure infrastructure.

Public DNS isn't used to buy domain names (you can use [App Service Domains](https://learn.microsoft.com/azure/app-service/manage-custom-dns-buy-domain?WT.mc_id=MVP_400037#buy-and-map-an-app-service-domain) or a 3rd party to do that), but your domains can be hosted in Azure Public DNS to manage the records.

DNS domains in Azure are hosted on Azure's global network of DNS name services using [anycast networking](https://en.wikipedia.org/wiki/Anycast). Using Azure DNS, we can manage and resolve domain names securely and reliably in our virtual networks without having to add custom DNS solutions.

A DNS Zone is used to host the DNS records for a particular domain. We first need to create a DNS Zone to host our domain in Azure DNS. Each DNS record for your domain is then created inside the DNS zone.

When you create a DNS zone in Azure DNS, you need to keep in mind:

- The name of the zone has to be unique within the resource group, and it can't exist already.
- You can reuse the same name in a different resource group or subscription, and;
- Where multiple zones share the same name, each instance will be assigned a different name server address. Only one set of those can be configured with the domain name register that you use.

For more on how Zones and records work in Public DNS, check out the [documentation](https://learn.microsoft.com/azure/dns/dns-zones-records?WT.mc_id=MVP_400037).

### Delegating DNS Domains and Child Domains

We can use Azure DNS to host a DNS zone and manage DNS records for that domain in Azure. We have to delegate the domain to Azure DNS from the parent domain in order for DNS queries for a domain to reach Azure DNS.

We'll need to know the name server names for our zone. Every time a DNS zone is created, Azure DNS will allocate name servers from a pool. Once these are assigned, Azure DNS will automatically create the authoritative NS records in your zone.

After the zone is created and we know the name servers, we'll need to update the parent domain by editing the NS records with the ones that Azure DNS creates.

If we want to setup separate **child zones**, we can delegate a subdomain in Azure DNS. So if I've configured willvelida.com, I can go ahead an configure a separate child zone for shop.willvelida.com (I don't know what I'd actually sell though...anyone want a hat? I could etsy some 😂).

Setting up subdomains follows the same process as delegations, except the NS records must be created in the parent zone (so willvelida.com) in Azure DNS, rather than the domain register.

## Creating a Public DNS Zone in Azure with Bicep

Let's take a simple example of setting up a custom domain for a static web app. Here, we'll set up the public dns zone, and then create a CNAME record that maps the default host name of the static web app (which would look something like: https://lively-water-089522f1e.4.azurestaticapps.net) and map it to custom domain. 

If you want to follow along, you'll need the following:

- [An Azure Subscription](https://azure.microsoft.com/en-au/pricing/purchase-options/azure-account/search?ef_id=_k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&OCID=AIDcmmxbrcqs76_SEM__k_CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE_k_&gad_source=1&gclid=CjwKCAiA7Y28BhAnEiwAAdOJUJwBlIJ1_o1ejVanQ8tdp9MJ_lo23M6UBNtBrctZsb8jceEhP3LmQRoCXPwQAvD_BwE&WT.mc_id=MVP_400037)
- [The AZ CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli?WT.mc_id=MVP_400037)
- [VS Code with the Bicep extension installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install?WT.mc_id=MVP_400037#visual-studio-code-and-bicep-extension)

First up, we can create a App Service Domain to use a custom domain:

```bicep
param customDomainName string = 'example.com'
param email string = ''
param firstName string = ''
param lastName string = ''
param phone string = '<needs to be in the following format +00.000000000>'
param agreedAt string = utcNow()
param agreedBy string = '<your-client-ip>'

resource customDomain 'Microsoft.DomainRegistration/domains@2024-04-01' = {
  name: customDomainName
  location: 'global'
  properties: {
    autoRenew: false
    privacy: true
    consent: {
      agreedAt: agreedAt
      agreedBy: agreedBy
      agreementKeys: [
        'DNRA'
        'DNPA'
      ]
    }
    contactAdmin: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactBilling: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactRegistrant: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactTech: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
  }
}
```

Then we can create our static web app and custom domain:

```bicep
param location string = 'westus2'
param webAppName string = 'swa-prod-ae-wv-001'

resource swa 'Microsoft.Web/staticSites@2021-01-15' = {
  name: webAppName
  location: location
  sku: {
    tier: 'Free'
    name: 'Free'
  }
  properties: {}
}

resource swaCustomDomain 'Microsoft.Web/staticSites/customDomains@2021-01-15' = {
  name: customDomainName
  parent: swa
  properties: {
    validationMethod: 'dns-txt-token'
  }
}
```

And finally, we can define our Public DNS zone and CNAME record for our static web app:

```bicep
resource publicDnsZone 'Microsoft.Network/dnsZones@2018-05-01' = {
  name: customDomainName
  location: 'global'
}

resource cname 'Microsoft.Network/dnsZones/CNAME@2018-05-01' = {
  name: 'www'
  parent: publicDnsZone
  properties: {
    TTL: 3600
    CNAMERecord: {
      cname: swa.properties.defaultHostname
    }
  }
}
```

Our complete template should look something like this (you'll need to enter your own value for the parameters):

```bicep
param customDomainName string = 'example.com'
param email string = ''
param firstName string = ''
param lastName string = ''
param phone string = '<needs to be in the following format +00.000000000>'
param agreedAt string = utcNow()
param agreedBy string = '<your-client-ip>'
param location string = 'westus2'
param address1 string = ''
param city string = ''
param country string = ''
param postalCode string = ''
param state string = ''
param webAppName string = 'swa-prod-ae-wv-001'

resource customDomain 'Microsoft.DomainRegistration/domains@2024-04-01' = {
  name: customDomainName
  location: 'global'
  properties: {
    autoRenew: false
    privacy: true
    consent: {
      agreedAt: agreedAt
      agreedBy: agreedBy
      agreementKeys: [
        'DNRA'
        'DNPA'
      ]
    }
    contactAdmin: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactBilling: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactRegistrant: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
    contactTech: {
      email: email
      nameFirst: firstName
      nameLast: lastName
      phone: phone
      addressMailing: {
        address1: address1
        city: city
        country: country
        postalCode: postalCode
        state: state
      }
    }
  }
}

resource swa 'Microsoft.Web/staticSites@2021-01-15' = {
  name: webAppName
  location: location
  sku: {
    tier: 'Free'
    name: 'Free'
  }
  properties: {}
}

resource swaCustomDomain 'Microsoft.Web/staticSites/customDomains@2021-01-15' = {
  name: customDomainName
  parent: swa
  properties: {
    validationMethod: 'dns-txt-token'
  }
}

resource publicDnsZone 'Microsoft.Network/dnsZones@2018-05-01' = {
  name: customDomainName
  location: 'global'
}

resource cname 'Microsoft.Network/dnsZones/CNAME@2018-05-01' = {
  name: 'www'
  parent: publicDnsZone
  properties: {
    TTL: 3600
    CNAMERecord: {
      cname: swa.properties.defaultHostname
    }
  }
}
```

You can deploy your template using the following AZ CLI command:

```bash
az deployment group create --resource-group <rg-name> --template-file .\main.bicep
```

Navigate to your Public DNS zone, and you should see a CNAME record that maps your custom domain to the default host name of your Static Web App.

## Private DNS Services in Azure

Private DNS services resolve names and IP addresses for services. When we deploy resources in virtual networks and we need to resolve domain names to internal IP addresses, we can use one of the following:

1. Azure Private DNS Zones
2. Azure-provided name resolution
3. Name resolution that uses our own DNS server.

### Azure Private DNS Zones

Private DNS zones in Azure are only available to internal resources. We can access them from any region, subscription, vnet, and tenant because they are global in scope. If we can read the zone, we can use it for name resolution.

When the DNS zone is deployed, we can autoregister the resource resource based on the resource name in Azure. We can also manually create the resource records. Private DNS zones support the full range of records.

We can link our virtual networks to private DNS zones. This is possible in two ways:

1. *Register* - Each virtual network can link to one private DNS zone for registration. If you want, you can link up to 100 virtual networks to the same private DNS zone for registration.
2. *Resolution* - There may be many other private DNS zones for different namespaces. You can link a virtual network to each zone for name resolution, and each virtual network can link up to 1000 private DNS zones for name resolution.

### Azure provided DNS

Azure provides a free default internal DNS which provides basic authoritative DNS capabilities. All zone names and records are managed by Azure, and give you no control over the name and life cycle of DNS records.

Any VM created in the virtual network is registered in the internal DNS zone and gets a DNS domain name. However, like all free things in life, there are some limitations. You can resolve across different virtual networks, Azure provided DNS registers resource names, not guest OS name, and you can't create manual records.

### Integrating Azure virtual networks with your on-prem DNS

You can use custom DNS configuration on your virtual network to integrate your on-prem DNS. One common pattern is to use an internal Azure private DNS zone for auto registration, then use a custom configuration to forward queries from an external DNS server to external zones.

## Creating a Azure Private DNS Zone with Bicep

Let's take a look at setting up a Private DNS Zone using Bicep. Remember that unlike Public DNS zones, private DNS zones provide records that aren't visible on the public internet, so in order for us to create our private domain in Azure Private DNS zones, we'll need to create one.

To do this, we can use the following Bicep code:

```bicep
param domain string = 'private.willvelida.com'

resource dnsZone 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: domain
  location: 'global'
}
```

Private DNS Zones are global resources (Hence the `global` value used for location). This means that the private zone isn't dependant on a single virtual network or region, and can be linked to multiple virtual networks in different regions.

This is great from a reliability perspective, as if service is interrupted in a virtual network, the private zone will still be available.

Let's create a virtual network for our Private DNS Zone to be linked to. We can do this in Bicep with the following:

```bicep
param vnetName string = 'vnet-prod-ae-wv-001'
param subnetName string = 'mySubnet'

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.2.0.0/16'
      ]
    }
    subnets: [
      { 
        name: subnetName
        properties: {
          addressPrefix: '10.2.0.0/24'
        }
      }
    ]
  }

  resource subnet 'subnets' existing = {
    name: subnetName
  }
}
```

With our virtual network defined, we can define a link to that virtual network.

To resolve DNS records in a private DNS zone, resources must be linked to the private zone. The virtual network link associates the virtual network to the private zone.

When we create a virtual network link, we can enable autoregistration for DNS records for devices in the virtual network. If this is enabled, Azure Private DNS updates DNS records whenever a VM inside the virtual network that's linked is created, changes its IP address, or is deleted.

To define a virtual network link in Bicep, we can define the following resource:

```bicep
param vnetLinkName string = 'myVNetLink'

resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2024-06-01' = {
  name: vnetLinkName
  parent: dnsZone
  location: 'global'
  properties: {
    virtualNetwork: {
      id: virtualNetwork.id
    }
    registrationEnabled: true
  }
}
```

Again, the virtual network link is a global resource, and we enable autoregistration by setting the `registrationEnabled` property to `true`.

Now let's create a VM that we can use to test autoregistration. I'll just use a simple Windows Azure VM for this:

```bicep
param vmName string = 'vmprodaewv001'
param adminUsername string = ''
param adminPassword string = ''
param nsgName string = 'nsg-prod-ae-wv-001'
param nicName string = 'nic-prod-ae-wv-001'
param vmSize string = 'Standard_DS1_v2'

// Omitted code
// Network Security Group
resource nsg 'Microsoft.Network/networkSecurityGroups@2024-05-01' = {
  name: nsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowRDP'
        properties: {
          priority: 1000
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '3389'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

resource nic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: nicName
  location: location
  properties: {
      ipConfigurations: [
          {
              name: 'ipconfig1'
              properties: {
                  subnet: {
                      id: virtualNetwork.properties.subnets[0].id
                  }
                  privateIPAllocationMethod: 'Dynamic'
              }
          }
      ]
      networkSecurityGroup: {
        id: nsg.id
      }
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

Our complete Bicep template should look like this:

```bicep
param domain string = 'private.willvelida.com'
param location string = resourceGroup().location
param vnetName string = 'vnet-prod-ae-wv-001'
param vnetLinkName string = 'myVNetLink'
param subnetName string = 'mySubnet'
param vmName string = 'vmprodaewv001'
param adminUsername string = ''
param adminPassword string = ''
param nsgName string = 'nsg-prod-ae-wv-001'
param nicName string = 'nic-prod-ae-wv-001'
param vmSize string = 'Standard_DS1_v2'

resource dnsZone 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: domain
  location: 'global'
}

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.2.0.0/16'
      ]
    }
    subnets: [
      { 
        name: subnetName
        properties: {
          addressPrefix: '10.2.0.0/24'
        }
      }
    ]
  }

  resource subnet 'subnets' existing = {
    name: subnetName
  }
}

resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2024-06-01' = {
  name: vnetLinkName
  parent: dnsZone
  location: 'global'
  properties: {
    virtualNetwork: {
      id: virtualNetwork.id
    }
    registrationEnabled: true
  }
}

// Network Security Group
resource nsg 'Microsoft.Network/networkSecurityGroups@2024-05-01' = {
  name: nsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowRDP'
        properties: {
          priority: 1000
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '3389'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

resource nic 'Microsoft.Network/networkInterfaces@2024-05-01' = {
  name: nicName
  location: location
  properties: {
      ipConfigurations: [
          {
              name: 'ipconfig1'
              properties: {
                  subnet: {
                      id: virtualNetwork.properties.subnets[0].id
                  }
                  privateIPAllocationMethod: 'Dynamic'
              }
          }
      ]
      networkSecurityGroup: {
        id: nsg.id
      }
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

To deploy this template, we can use the following AZ CLI command:

```bash
az deployment group create --resource-group <rg-name> --template-file .\main.bicep
```

Once your template has been deployed, we should be able to see that the autoregistration worked. Navigate to your Private DNS Zone  in the Azure Portal and look at **Recordsets** under **DNS Management**:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6slycywkw8ldjp2wkyxl.png)

As you can see, a DNS record of Type A has been created with the value of our VM's IP address. Now we can test the name resolution for our Private Zone.

Open up your Windows VM in the Azure Portal and click **Run Command** under the **Operations** menu. Select **RunPowerShellScript** and run the following command:

```powershell
ping <name-of-your-vm>.private.willvelida.com
```

You should see the following response:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tc3ywtlchjqv2urtnt5x.png)

If you want to clean up your resources after you're done, just delete the resource group using this AZ CLI command:

```bash
az group delete --name <rg-name>
```

## Wrapping Up

Hopefully you now have a better understanding of how both Public and Private DNS works in Azure, when you'd use one over the other, and how you can set them up using Bicep.

If you want to take a look at the code we went through in this post, please check it out on my [GitHub](https://github.com/willvelida/azure-networking-samples).

If you have any questions about this, please feel free to either **comment below** or reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! If you just want to look at the code for this blog post, [it's on my GitHub!](https://github.com/willvelida/azure-networking-samples)

Until next time, Happy coding! 🤓🖥️