---
title: "Creating a Azure Kubernetes cluster with an attached Azure Container Registry with Bicep"
date: 2022-09-11T20:47:30+13:00
draft: false
tags: ["Azure Kubernetes Service","Azure","Bicep","Kubernetes","Azure Container Registry","Containers", "Infrastructure as Code"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c2py048l6jaq0u3ubm8v.png
    alt: "AKS, ACR and Bicep Lang logs"
    caption: 'You can use Bicep to provision AKS clusters, ACR registries and role assignments to pull images from ACR into AKS'
---

Azure Kubernetes Service (AKS) simplifies deploying a managed Kubernetes cluster into Azure by offloading the operational overhead to Azure. Azure Container Registry allows you to build, store and manage container images in a private registry, from which you can pull your images into your AKS cluster.

Azure Container Registry integrates with AKS. We can attach container registries to our AKS clusters using an Azure Active Directory managed identity. We can then assign that managed identity will the AcrPull role assignment that allows our AKS cluster to pull images from our Azure Container Registry.

In this article, we'll create our AKS cluster, Azure Container Registry and role assignments in Bicep, deploy it to Azure, build a container image that we'll push into our private registry and then create the manifest file containing the Kubernetes objects that we'll need to run the application we packaged into our container image.

If you want to deploy this sample, you can do so by clicking the 'Deploy to Azure' button below:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fwillvelida%2Fazure-samples%2Fmain%2Faks-bicep%2Fazure-deploy.json)

## Creating our resources

Let's start by creating our AKS cluster. We'll keep it simple and create a cluster with 3 nodes that runs Linux workloads:

```json
@description('Random name for our application')
param applicationName string = uniqueString(resourceGroup().id)

@description('The location to deploy our resources')
param location string = resourceGroup().location

@description('The name of the cluster')
param clusterName string = 'aks${applicationName}'

@description('Optional DNS Prefix to use with hosted Kubernetes API server FQDN')
param dnsPrefix string = 'aks${applicationName}'

@description('Disk size (in GB) to provision for each of the agent pool nodes. This value ranges from 0 to 1023. Specifying 0 will apply the default disk size for that agentVMSize.')
@minValue(0)
@maxValue(1023)
param osDiskSizeGB int = 0

@description('The number of nodes for the cluster.')
@minValue(1)
@maxValue(50)
param agentCount int = 3

@description('The size of the Virtual Machine.')
param agentVMSize string = 'standard_d2s_v3'

@description('User name for the Linux Virtual Machines.')
param linuxAdminUsername string = ''

@description('Configure all linux machines with the SSH RSA public key string. Your key should include three parts, for example \'ssh-rsa AAAAB...snip...UcyupgH azureuser@linuxvm\'')
param sshRSAPublicKey string = ''

resource aks 'Microsoft.ContainerService/managedClusters@2022-05-02-preview' = {
  name: clusterName
  location: location
  identity: {
   type: 'SystemAssigned' 
  }
  properties: {
    dnsPrefix: dnsPrefix
    agentPoolProfiles: [
      {
        name: 'agentpool'
        osDiskSizeGB: osDiskSizeGB
        count: agentCount
        vmSize: agentVMSize
        osType: 'Linux'
        mode: 'System'
      }
    ]
    linuxProfile: {
      adminUsername: linuxAdminUsername
      ssh: {
        publicKeys: [
          {
            keyData: sshRSAPublicKey
          }
        ]
      }
    }
  }
}
```

To access the nodes in our AKS clusters, we'll need to connect using an SSH key pair. We can generate this using the Azure Cloud Shell and running the ```ssh-keygen``` command. Go to your Cloud Shell in the portal and run the following:

```bash
ssh-keygen -t rsa -b 4096
```

You'll get a response that tells you where the SSH key pair has been written to. In the Cloud Shell, run the ```code``` command to open up the file, and copy and paste the contents. This is the value that you'll need to provide your ```sshRSAPublicKey`` parameter.

## Creating our container registry

We can also create our Container Registry in Bicep. For this demo, we'll create a system-assigned identity and enable the admin user so we can autheticate with our container registry on our local machine. For production scenarios, I recommend that you disable the admin user on your container registry:

```json
@description('The name of our container registry')
param containerRegistryName string = 'acr${applicationName}'

resource acr 'Microsoft.ContainerRegistry/registries@2022-02-01-preview' = {
  name: containerRegistryName
  location: location
  sku: {
    name: 'Standard'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    adminUserEnabled: true
  }
}
```

## Granting our role assignment

AKS clusters need identities to access Azure resources, which can be either a managed identity or a service principal. What we want to do here is to create a role assignment that allows our cluster to pull container images from our container registry.

We've already created a system-assigned identity for both our AKS cluster and our Azure Container Registry, so all we need to do is to create a role assignment in our Bicep code using the ```Microsoft.Authorization/roleDefinitions``` resource.

The roleAssignments resource type is an [**extension resource**](https://docs.microsoft.com/en-us/azure/templates/microsoft.authorization/roleassignments?pivots=deployment-language-bicep). This means that we can apply this resource to another resource by using the *scope* property.

We create our role assignment using the following Bicep code:

```json
var acrPullRoleDefinitionId = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')

resource acrPullRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, aks.id, acrPullRoleDefinitionId)
  scope: acr
  properties: {
    principalId: aks.properties.identityProfile.kubeletidentity.objectId
    roleDefinitionId: acrPullRoleDefinitionId
    principalType: 'ServicePrincipal'
  }
}
```

In the above code, we're defining a variable that contains the role definition id for the [AcrPull role](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpull). We then create our role assignment resource using a guid with the ```guid()``` function that Bicep provides. Having a guid as the name for our role assignment resource is a requirement. We then scope our role assignment to our Container Registry.

The important thing to note here is that we are granting the role assignment to the Kubelet identity of our cluster, since that's what authenticates to our Azure Container Registry.

Read [this guidance](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity) to understand how to use managed identities in AKS.

## Deploying our cluster

Now that we've created our Bicep template, we can start deploying our resources. First off, let's create our resource group by running the following AZ CLI command:

```bash
az group create --name <resource-group-name> --location <location>
```

Once that's been created, we can deploy our template:

```bash
az deployment group create --resource-group <resource-group-name> --template-file main.bicep
```

Give it a few minutes and your cluster, container registry and role assignment will be created. Navigate to the portal to verify that everything has been deployed correctly.

## Pushing images to Azure Container Registry

With our resources created, we can push images into our container registry. We'll need to log into our Container Registry by using the following AZ CLI command:

```bash
az acr login <acr-login-server>
```

Your ```<acr-login-server>``` name should look like the following: *mycontainerregistryname.azurecr.io*.

We enabled admin user on the registry, so we can use the username and password to log into our container registry when prompted. You can find this under **Settings** > **Access Keys** in the portal.

For the purposes of this demo, I'm going to use the Azure Vote application. You can pull that image from Microsoft's container registry by running the following:

```bash
docker pull mcr.microsoft.com/azuredocs/azure-vote-front:v1
```

Before we push this image to our Azure Container Registry, we need to tag the image:

```bash
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acr-login-server>/azure-vote-front:v1
```

Once tagged, we can push it to ACR:

```bash
docker push <acr-login-server>/azure-vote-front:v1
```

You can verify that your image was successfully pushed by navigating to your Container Registry and navigating to **Services** > **Repositories**.

![Container Image that has been pushed to ACR](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5t0efr8yburzhn54ugcs.png)

## Deploying to Kubernetes

In order to work with our cluster, we'll need to get the credentials. We can do so using the following:

```bash
az aks get-credential --resource-group <resource-group-name> --name <cluster-name>
```

This will merge your AKS credentials into your .kube config. When we run kubectl commands, we'll be running it against this cluster.

We can now create the required Kubernetes Objects in a manifest file to deploy our application to AKS. For this demo, I'm just going to use one file:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-back
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-back
  template:
    metadata:
      labels:
        app: azure-vote-back
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: azure-vote-back
        image: mcr.microsoft.com/oss/bitnami/redis:6.0.8
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "yes"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 6379
          name: redis
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-back
spec:
  ports:
  - port: 6379
  selector:
    app: azure-vote-back
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-front
  template:
    metadata:
      labels:
        app: azure-vote-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: azure-vote-front
        image: <acr-login-server>/azure-vote-front:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
        env:
        - name: REDIS
          value: "azure-vote-back"
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-front
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: azure-vote-front
```

Let's break this manifest file down:

- Our first object is a **Deployment** for our backend app. Deployments are described as a desired state, and the Deployment Controller changes the actual state to the desired state. In this deployment, we'll pulling a Redis image that we'll use for our backend and we're exposing it on port 6379. We can create Deployments in Kubernetes using the *Deployment* kind. For this deployment, we create a single Pod.
- We then create a **Service** object for our backend. Services are an abstraction in Kubernetes that defines a logical set of Pods and how to access them. We target the pods used by a service using selectors.
- We then create another Deployment for our front end app. Under *spec.containers.image*, we pass in the image name that we pushed up to our Azure Container Registry earlier.
- We then expose our frontend Deployment with another Service. We expose our service using a **LoadBalancer**. This is a public load balancer that helps distribute inbound flows to our app. When we create the service, Azure Load Balancer will be configured with a new public IP that will front out service.

We can apply this manifest file to our cluster by running the following kubectl command:

```bash
kubectl apply -f azure-vote.yml
```

We can get the external IP for our frontend service by running the following:

```bash
kubectl get service azure-vote-front --watch
```

This will give you an **EXTERNAL-IP** which you can use to navigate to the front-end.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dddk3wah6mcfswsxgpb5.png)

Well done for making it this far! To save on costs, you can simply delete the resource group:

```bash
az group delete --name <resource-group-name>
```

## Conclusion

In this post, we created a AKS cluster and a Azure Container Registry, granted a role assignment to our cluster so it can pull images from our registry. We then created a simple Kubernetes manifest that deployed our application using an image from our Azure Container Registry.

If you want to learn more about some of the concepts we've covered in this article, please check out the following:
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kuberenetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Use a public Standard Load Balancer in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/load-balancer-standard)
- [Authenticate with Azure Container Registry from Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli)

If you have any questions, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️