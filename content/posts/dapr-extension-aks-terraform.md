---
title: "Installing the Dapr extension for Azure Kubernetes Service with Terraform"
date: 2024-10-01
draft: false
tags: ["Terraform", "Azure Kubernetes Service", "Dapr", "DevOps", "IaC"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7isk422123vn6t0plhbe.png
    alt: "Installing the Dapr extension for Azure Kubernetes Service with Terraform"
    caption: "Installing extensions on AKS with Terraform allows us to install different services on our cluster with ARM driven experiences"
---

As part of the [AKS cluster that I'm building for my personal development](https://www.willvelida.com/posts/github-actions-terraform-azure/), I decided it be worthwhile installing the Dapr extension on my cluster. AKS extensions provide an Azure Resource Manager driven experience for installing and managing different services like Dapr on your cluster.

Since I built my cluster using Terraform, I decided to configure the Dapr extension using Terraform as well. In this article, I'll talk about how we can configure our AKS cluster so that we can install extensions on it, How the Dapr cluster extension works, and then I'll explain how we can configure our Dapr extension in Terraform.

# Configuring our AKS Cluster to install extensions.

The first thing that we'll need to to do is ensure that our AKS cluster has a managed identity. Cluster extensions won't work with service-principal based clusters.

In Terraform, we can create our AKS cluster with a managed identity like so:

```terraform
resource "azurerm_kubernetes_cluster" "aks" {
  name = var.aks_name
  location = var.location
  resource_group_name = var.rg_name
  dns_prefix = var.aks_name
  role_based_access_control_enabled = true
  tags = var.tags

  default_node_pool {
    name = "default"
    node_count = var.node_count
    vm_size = var.vm_size
  }

  identity {
    type = "UserAssigned"
    identity_ids = [ var.identity_ids ]
  }

  network_profile {
    network_plugin = "kubenet"
    load_balancer_sku = "standard"
  }

  monitor_metrics {
    annotations_allowed = null
    labels_allowed = null
  }

  linux_profile {
    admin_username = var.username
    ssh_key {
      key_data = var.ssh_public_key
    }
  }
}
```

I've implemented this as a local module, so in my ```main.tf``` file I pass through the ```identity_ids``` as the Id of my managed identity:

```terraform
module "user_assigned_identity" {
  source   = "../modules/user-assigned-identity"
  name     = var.user_assigned_identity_name
  location = var.location
  rg_name  = module.resource-group.name
  tags     = var.tags
}

module "aks" {
  source         = "../modules/aks-cluster"
  rg_name        = module.resource-group.name
  location       = module.resource-group.location
  tags           = var.tags
  username       = var.aks_username
  ssh_public_key = module.ssh-key.key_data
  vm_size        = var.vm_size
  node_count     = var.node_count
  identity_ids   = module.user_assigned_identity.user_assinged_identity_id
  aks_name       = var.aks_name
}
```

Another thing we need to do is register a couple of resource providers on our Azure subscription. This is fairly straightforward as it's just a couple of AZ CLI commands:

```bash
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait
```

Once that's all setup, we can look at configuring the Dapr extension on our AKS cluster. But first, let's first discuss how the Dapr extension works.

# How the Dapr extension works.

The Dapr extension will provision the Dapr control plan on our AKS cluster and create the following services:

- `dapr-operator` = This manages the component updates and Kubernetes services endpoints for Dapr (state store, pub/subs etc.)
- `dapr-sidecar-injector` = This injects Dapr into annotated deployment pods and adds environment variables to enable apps to communicate with Dapr without hard coding the Dapr port values.
- `dapr-placement` = This is used for actors only, and it creates mapping tables that map actor instances to pods.
- `dapr-sentry` = This manages mTLS between services and acts as a cert authority.

Once Dapr is installed on our cluster, we can develop apps using Dapr's building block APIs by adding annotations to our deployments.

Now let's take a look at how we can create our extension using Terraform.

# Installing the Dapr extension on AKS with Terraform

AKS cluster extensions are available in the AzureRm provider. I've also implemented this as module (just in case there are other extensions I want to install on the cluster):

```terraform
resource "azurerm_kubernetes_cluster_extension" "ext" {
  name = var.ext_name
  cluster_id = var.cluster_id
  extension_type = var.extension_type
  release_train = "Stable"
  configuration_settings = var.configuration_settings
}
```

For my module, I just want to pass the following variables in my ```variables.tf``` file:

```terraform
variable "ext_name" {
  type = string
  description = "The name of the AKS extension"
}

variable "cluster_id" {
  type = string
  description = "The ID of the AKS cluster to install the extension"
}

variable "extension_type" {
  type = string
  description = "The type of the extension"
}

variable "configuration_settings" {
  type = map
  description = "The configuration settings for the extension"
  default = {}
}
```

Let's break this down:

- `ext_name` is the name of our Kubernetes Cluster Extension.
- `cluster_id` specifies the Id of our Kubernetes cluster that our extension will be installed on.
- `extension_type` specifies the type of extension that's going to be installed.
- `configuration_settings` are an optional configuration settings property that we can use to configure name-value pairs for the extension.

In our module, I'm hardcoding the `release_train` value to be `Stable`. This can also be `Preview`, which can be good for experimenting with new features, but not suitable for production use cases.

Once our module is defined, it's just a case of implementing it in our `main.tf` file like so:

```terraform
module "dapr-extension" {
  source         = "../modules/aks-extension"
  extension_type = var.dapr_extension_type
  cluster_id     = module.aks.aks_id
  ext_name       = var.dapr_extension_name
}
```

And then setting some values in our `variables.tf` file.

```terraform
variable "dapr_extension_name" {
  type        = string
  description = "The name of the Dapr extension"
  default     = "dapr"
}

variable "dapr_extension_type" {
  default     = "Microsoft.Dapr"
  type        = string
  description = "The type of the Dapr extension"
}
```

Once our AKS cluster has been deployed, we should see our extension configured in the portal.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eernbsjui34hhpdcrbuy.png)

A new namespace called `dapr-system` will also be created within our AKS cluster.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vbo4o7opgogcqq9upm9d.png)

And within our namespace, we should see the `dapr-sentry`, `dapr-operator`, `dapr-sidecar-injector`, and the `dapr-monitoring-metrics` deployments.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6si72etpjrnmq9z559wf.png)

# Conclusion

Installing AKS extensions with Terraform is fairly straightforward. Doing this via Terraform gives us the advantage of installing our extensions for our AKS cluster in a declarative way instead of configuring it imperatively through tools like the AZ CLI. Even though I installed the Dapr extension, we can install extensions for Flux, Azure Machine Learning, and more. 

If you want to learn more about them, you can read more [here](https://learn.microsoft.com/en-us/azure/aks/cluster-extensions#currently-available-extensions)

If you have any questions about this, please feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️