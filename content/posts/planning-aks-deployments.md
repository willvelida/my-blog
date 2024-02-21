---
title: "Planning AKS Deployments"
date: 2024-02-21
draft: false
tags: ["Azure","Cloud Native", "Platform Engineering", "Kubernetes"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fqzdygu59jfsmpzc3oq8.png
    alt: "Planning AKS Deployments"
    caption: "Understanding the core components of AKS can help you plan successful container workload deployments."
---

Kubernetes provides reliable scheduling of fault-tolerant application workloads, and is ideal for container orchestration. AKS is a managed Kubernetes platform on Azure, which makes this simpler for us manage our container workloads.

In this blog post, I'll talk about the main components of Azure Kubernetes, such as control plane nodes, node pools, pods, deployments etc. I'll also talk about how we can configure network access in AKS, and how we can monitor our AKS clusters.

Let's dive in!

## Azure Kubernetes Service

Azure Kubernetes Services is a managed Kubernetes service that reduces the complexity of deployment and coordination of container-based applications. Using AKS, we can build and run microservices to orchestrate and manage the availability of app components.

Azure manages the AKS control plane (which you get for free), and you pay for the nodes that run your applications.

The basic components of AKS include:

- **AKS** - The managed cluster hosted in Azure cloud. Azure will manage the Kubernetes API service, and you only need to manage the agent nodes.
- **Virtual network** - AKS creates a virtual network which agent nodes are connected. You also have the option of creating the VNet first, which gives you more control over on-prem connectivity, IP addressing, subnet configuration.
- **Ingress** - The ingress server will expose HTTP(S) routes to services inside the cluster.
- **Azure Load Balancer** - Once your cluster is created, the cluster can then use load balancer. Once the NGINX service is deployed, the load balancer is configured with a new public IP that will sit in front of your ingress controller. The load balancer routes internet traffic to the ingress.
- **Azure Container Registry** - This is used to store your private Docker images that are deployed to the cluster. AKS can authenticate with ACR using Azure AD identity. You don't need ACR, you can use Docker Hub or other container registry.
- **Azure Monitor** - This collects and store metrics and log, telemetry, and platform metrics. You can set up alerts, monitor your apps, and discover failures. Azure Monitor integrates with AKS to collect metrics from controllers, nodes, and containers.

You might have an architecture that looks like this:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fqzdygu59jfsmpzc3oq8.png)

## Cluster architecture

Kubernetes is essentially a cluster of virtual or on-prem machines. These are called nodes, and they share compute, network, and storage resources.

Each cluster has one master node connected to one or more worker nodes. The worker nodes are responsible for running groups of container workloads called pods. The master node manages which pods run on which worker nodes.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/je3n0w3z27p9dapbk660.png)

Kubernetes clusters are divided into two components:

1. The Control Plane - This provides the core Kubernetes services and orchestration of application workloads.
2. The Nodes - These run your application workloads.

A cluster is a group of computers that you configure to work together as a single system. The cluster contains at least one control plane, and one or more nodes.

Both the control plane and nodes can be physical devices, VMs, or cloud instances. The default OS in Kubernetes is Linux.

Clusters use software that's responsible for scheduling and controlling these tasks. The computers in the cluster that run the tasks are called nodes, and computers that run scheduling software are called control planes. The Kubernetes control plane in the cluster runs a collection of services that manage the orchestration functionality in Kubernetes.

Nodes communicate with the control plane via the API server to inform it about state changes on the node. In the control plane, Kubernetes will include many objects to help the master node communicate with the worker node.

We can communicate with the master node by using kubectl. Kubectl commands are issued to the cluster via the kube-apiserver. This is the Kubernetes API that lives in the master node.

The kube-apiserver then sends requests to the kube-controller-manager in the master node, which then handles worker node operations. Commands from the master node are sent to the kubelet on worker nodes.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fe3zaum4j6sszqun9ufm.png)

The master node will maintain the current state of the cluster in etcd, a key value store database. To run our containerized apps and workloads as pods, we describe the desired state to the cluster using YAML files.

The kube-controller-manager then takes the YAML file and tasks the kube-scheduler with deciding which worker nodes the app should run on.

Working with the kubelet on each node, the kube-scheduler starts the pods, watches the state of the machines, and is responsible for managing the resources.

In Kubernetes deployments, the desired state becomes the current state in the etcd, but we can use rollbacks, rolling updates, and pausing rollouts to manage the state of our application.

In the background, deployments use ReplicaSets to ensure that the specified number of configured pods are running so that if any fail, the ReplicaSet replaces them. This is what makes Kubernetes self-healing.

## AKS pods

Pods are used in Kubernetes to run an instance of your application. A single pod represents a single instance of your app.

They typically have 1:1 mapping with a container, but you can use multiple containers for a single pod, schedule them together and allow them to share related resources.

When we create a pod, we can define resource requests to request a certain amount of CPU or memory for the pod. The Kubernetes Scheduler tries to meet the request by scheduling pods to run on a node with available resources. We can also specify maximum resource limits to prevent pods from consuming too much compute from the underlying node.

Implementing resource limits for all pods is best practice so that we can help the Kubernetes Scheduler identify the necessary resources for that Pod.

A pod is a logical resource, but our workloads run on the containers. Pods are ephemeral resources, and are deployed and managed using Kubernetes Controllers.

## AKS Nodes and Node Pools

When we create an AKS cluster, the control plane is automatically created and configured for us. This provides the core Kubernetes services, and the workload orchestration. The control plane and its resources exist only in the region where you created the cluster.

Nodes (*also referred to as agent nodes or worker nodes*) host the workloads. To run apps and supporting services, an AKS cluster needs at least one node: An Azure VN to run the Kubernetes node components and container runtime. Every cluster must have at least one system node pool with at least one node.

AKS groups nodes that have the same configuration into **node pools** of VMs that run workloads. You can have one node pool in your cluster, or multiple node pools to segregate different workloads on different nodes.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jb49zjkc0boyyqvsbbke.png)

## Namespaces in AKS

Resources in Kubernetes live in a **namespace** to divide an AKS cluster and create, view, or manage access to resources. You can create namespaces to separate groups, and then users belonging to those groups can only work with resources within their assigned namespaces.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/glyrpbyziy3syva5eaiu.png)

When we create an AKS cluster, the following namespaces are available

| **Namespace** | **Description** |
| ------------- | --------------- |
| *default* | Where pods and deployments live. The default namespace you interact with the Kubernetes API |
| *kube-system* | Where core resources live, like network features such as DNS and proxy, or the Kubernetes dashboard. You don't deploy your own apps into this namespace |
| *kube-public* | Typically not used. Used for resources to be visible across the whole cluster, and viewed by any user. |

## Network Access in AKS

Kubernetes provides an abstraction layer for virtual networking. Nodes connect to a virtual network, which provide inbound and outbound connectivity for pods. The *kube-proxy* component runs on each node to provide these network features.

*Services* logically group pods to allow for direct access on a specific port via an IP address or DNS name. *ServiceTypes* allow you to specify what kind of *Service* you want. You can use a *load balancer* to distribute traffic, and for more complex routing you can use *ingress controllers*.

The following *ServiceTypes* are available:

**ClusterIP** creates an internal IP address for use within the AKS cluster. This is good for **internal-only apps** that support other workloads in the cluster.

**NodePort** creates a port mapping on the underlying node that allows the app to be accessed directly with the node IP address and port.

**LoadBalancer** creates an Azure load balancer resources, configures an external IP address, and connects to the requested pods to the load balancer backend pool. To allow traffic to reach the app, load balancing rules are created on the desired pods. You can also use an Ingress controller for extra control and routing of the inbound traffic.

**ExternalName** - This creates a specific DNS entry for easier app access.

Either the load balancers and service IP addresses can be dynamically assigned, or you can use an existing static IP address. You can use both internal or external static IP addresses.

You can also create both internal (with a private IP address, therefore not accessible from the public internet) and external load balancers.

### Azure Virtual Networks

In AKS, we can deploy a cluster using either the *Kubenet networking* model or the *Azure Container Networking Interface (CNI)* model.

The *Kubenet* model is the default configuration when creating a cluster. Nodes receive an IP address from the VNet subnet, and pods receive an IP address from a logically different address space than the nodes VNet subnet.

NAT (Network address translation) is then configured so that the pods can reach resources in the VNet. The source IP address of the traffic is translated to the node's primary IP address. Nodes will use the kubenet Kubernetes plugin, and you have the choice of letting Azure create the VNet for you, or bringing your own VNet.

Only the nodes receive a routable IP address. Pods use NAT to communicate with resources outside the cluster.

The *Azure CNI* approach is a little more advanced. With this approach, every pod gets an IP address from the subnet and can be accessed directly. This means we have to plan these IP addresses in advance and be unique across our network space. The equivalent number of IP addresses per node are then reserved up front.

If you don't plan this properly, you can experience IP address exhaustion or even have to rebuild clusters into a larger subnet as your application grows.

Traffic to endpoints in the same Vnet isn't NAT related to the primary IP of the node. The source address for traffic inside the Vnet is the pod IP. Traffic external to the VNet still NATs to the node's primary IP.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/01vjzf7e6pgvzg1hxmk9.png)

### Ingress Controllers

When we create a LoadBalancer-type Service, we also create the underlying Azure Load balancer resource. This is configured to distribute traffic to the pods in your Service on a given port. The *LoadBalancer* only works at layer 4. At layer 4, your Service isn't aware of the applications and can't make any more routing considerations.

*Ingress controllers* work at layer 7 and can use intelligent rules to distribute app traffic. These typically route HTTP traffic to different apps based on the inbound URL.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q6ou0mv6o1b35imc0s3o.png)


## Monitoring AKS

AKS generates platform metrics and resource logs that we can use to monitor the health and performance of our cluster.

AKS has native integration with Azure Monitor. This stores metrics and logs in Log Analytics and we can process this data to give us insights on our cluster, and create alerts.

Container Insights are a feature of Azure Monitor that collects and stores data that is generated from our cluster. Container Insights can monitor health and the performance of our cluster, and we can send that data to Log Analytics (*Fun fact*, enabling Container Insights for your AKS cluster deploys a containerized version of the Log Analytics agent).

There are different components within an AKS deployment that we can monitor. Each level has different monitoring requirements:

| **Level** | **What is this?** | **How can we monitor it?** |
| --------- | ----------------- | -------------------------- |
| Cluster | This is just VM scale sets that are abstracted as AKS nodes and node pools | We can monitor the status of the node, and resource utilization including CPU, memory, disk, and network |
| Managed AKS components | The control plane components of AKS, including API servers, cloud controller, and kubelet | Control plane logs and metrics from the kube-system namespace |
| Kubernetes objects and workloads | Kubernetes objects such as deployments, containers,and replica sets | Resource utilization and failures |
| Applications | workloads that are running on the AKS cluster | The monitoring here depends on your architecture, but it includes app logs, service transactions etc.
| External components | Components that aren't part of your AKS cluster | Again, specific to your components |

## Conclusion

In this article, we talked about the core components of AKS, how control plane nodes, node pools and workload resources work. How we can configure network access for our AKS clusters, and how we can monitor our AKS clusters at different component levels.

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️