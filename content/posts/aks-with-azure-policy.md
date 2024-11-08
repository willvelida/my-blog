---
title: "Integrating Azure Policy in your AKS cluster using Terraform"
date: 2024-11-08
draft: false
tags: ["Azure", "Azure Kubernetes Service", "Terraform", "DevOps", "IaC", "Azure Policy"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t4h1vypr4rctr0n2h6c7.png
    alt: "Integrating Azure Policy in your AKS cluster"
    caption: "Installing extensions on AKS with Terraform allows us to install different services on our cluster with ARM driven experiences"
---

As part of my AKS lab, I wanted to see how I can enforce compliance on workloads and my AKS control plane so I get into the habit of creating secure and compliant sandbox environments. We can achieve this with Azure Policy! So in this article, I'll cover:

- What Azure Policy is.
- How it integrates with Azure Kubernetes Service.
- How we can enable Azure Policy for our AKS cluster with Terraform
- How we can apply and test policies to our AKS Cluster
- How we can check the overall compliance state for our cluster in the Azure Portal.

## What is Azure Policy?

[Azure Policy](https://learn.microsoft.com/en-us/azure/governance/policy/overview?WT.mc_id=MVP_400037) allows us to manage the state of compliance of our Azure services. It compares the state of your resources to the business rules that you define in Azure Policy. This can include enforcing resource tags on your resources, limiting the types of services you can use in Azure and where you can deploy them.

We define these business rules in Azure Policy using policy definitions. As you can imagine, there are numerous scenarios that are common across all types of businesses, so Azure Policy provides a number of [built-in policies](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies) to cover these scenarios. If you have a special case that applies to your organization, or the built-in policies don't meet a requirement, you can create custom policies using JSON.

In policy definitions, we define a resource compliance condition along with the effect that should be taken if that condition is met.

Azure Policy works by assigning policy definitions or [initiatives](https://learn.microsoft.com/azure/governance/policy/policy-glossary?WT.mc_id=MVP_400037#initiative) (Groups of multiple policy definitions) to a specific scope. This could be a management group, subscription, or a resource group. All policy assignments are automatically inherited to all scopes underneath the assignment (we can make exclusions if we want to).

The policy assignments are evaluated when **we create or update** our Azure resources, or when we change the policy definition or scope.

## How does Azure Policy work with AKS?

Azure Policy works with AKS in two ways. We can enforce compliance with policies on the AKS control plane, and we can enforce compliance on workloads that are running in our cluster.

So at the control plane level, we can use Azure Policy to enforce the use of private clusters, and at the workload level, we can use Azure Policy to to enforce the use of allowed images. At the control plane level, policies work against the Azure API, while at the workload level, policies interact with the Kubernetes API.

To enforce policies on top of the Kubernetes API, Azure Policy for Kubernetes makes use of:

- **Admission webhooks**. These are built-in to the Kubernetes API, and allow the Kubernetes API to make calls to external webhooks to validate if a request to create, delete, modify, or connect to resources should be allowed.
- **Open Policy Agent (OPA)** is an open-source policy engine that provides a high-level language to define policies. We can use OPA to enforce policies in Kubernetes, our CI/CD pipeline, or in our applications. Azure Policy in Kubernetes will translate the policies in Azure into the OPA language that's deployed on our cluster 
- **OPA Gatekeeper**. This is a Kubernetes specific OPA implementation that integrates with the Kubernetes API by using the admission webhooks. Rather than deploying our own webhook handles, we can just use OPA gatekeeper to service the webhook responses.

## Enabling Azure Policy on our AKS cluster with Terraform

The first thing we need to do is register the `Microsoft.PolicyInsights` resource provider for our subscription. We can do this by running the following AZ CLI command:

```bash
az provider register --namespace Microsoft.PolicyInsights
```

To enable Azure Policy, [your cluster must use a supported version of Kubernetes](https://learn.microsoft.com/azure/aks/supported-kubernetes-versions?tabs=azure-cli&WT.mc_id=MVP_400037). You may have to open port 433 on your cluster so that the following domains can fetch policy definitions and assignments, as well as report compliance of the cluster back to Azure Policy:

- `data.policy.core.windows.net`
- `store.policy.core.windows.net`
- `login.windows.net`
- `dc.services.visualstudio.com`

Using my AKS Cluster that I've provisioned with Terraform, configuring Azure Policy for my AKS cluster is just a simple matter of setting the following value:

```terraform
azure_policy_enabled = true
```

Once you've enabled Azure Policy onto your cluster, you can verify that it was installed correctly by running the following commands:

```bash
kubectl get pods -n kube-system

# Produces the following output. Your pod names will be different
azure-policy-669f9bf787-lfl77                          1/1     Running   0             9m3s
azure-policy-webhook-6f96b659b4-kclvd                  1/1     Running   0             9m3s

kubectl get pods -n gatekeeper-system

gatekeeper-audit-777cb4649d-zmxm4        1/1     Running   0          9m56s
gatekeeper-controller-5dd5d8ccf4-56zmq   1/1     Running   0          9m56s
gatekeeper-controller-5dd5d8ccf4-szjvh   1/1     Running   0          9m56s
```

We can also verify that the add-on is installed by running the following AZ CLI command:

```bash
az aks show --query addonProfiles.azurepolicy -g <rg-name> -n <cluster-name>
```

This produces the following output (providing that you're using a managed identity. Clusters using service principles won't output the client ID, object ID, or resource ID):

```json
{
  "config": {
    "version": "v2"
  },
  "enabled": true,
  "identity": {
    "clientId": "<client-id>",
    "objectId": "<object-id>",
    "resourceId": "<cluster-resource-id>"
  }
}
```

## Assigning a policy to our AKS cluster

Now let's apply some policies to our AKS cluster! Again, I'll use Terraform to apply the built-in Policy initiative **Kubernetes cluster pod security baseline standards for Linux-based workloads**. This includes policies for the Kubernetes cluster pod security standards.

To apply this initiative using Terraform, we can do the following:

```bash
# Azure Policies to apply to our AKS cluster
resource "azurerm_resource_policy_assignment" "linuxbaseline" {
  name                 = "enforce-aks-pod-baseline-linux"
  resource_id          = module.aks.aks_id
  policy_definition_id = "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d"

  parameters = jsonencode({
    effect = {
      value = "Deny"
    }
  })
}
```

To see which initiatives are built-in for AKS clusters, check out the following [documentation](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-initiatives?source=recommendations&WT.mc_id=MVP_400037#kubernetes). Each initiative and policy will have a definition id that you can pass to your `azurerm_resource_policy_assignment` resource block. 

In this Terraform resource, I've explicitly added the effect value as `Deny`. I have found that if you don't do this, the default effect value will be `Audit`, which won't stop you from deploying non-compliant workloads or changes to your cluster, but it will show up as non-compliant regardless.

Once you've applied the Terraform, policy assignments can take up to [20 minutes to sync](https://learn.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes?WT.mc_id=MVP_400037#assign-a-policy-definition) in your cluster. We can confirm if the policy assignments are applied to our cluster by running the following:

```bash
kubectl get constrainttemplates

# Produces the following output
NAME                            AGE
k8sazurev2blockhostnamespace    26m
k8sazurev2noprivilege           26m
k8sazurev3allowedcapabilities   26m
k8sazurev3hostfilesystem        26m
k8sazurev3hostnetworkingports   26m
```

As you can see, one of the constraints we've now implemented in our cluster is that we've denied the ability for users to run privileged containers in our cluster. Let's test this out by creating a Pod manifest called `web-app-privileged.yml` that defines a privileged container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app-privileged
spec:
  containers:
  - name: web-app-privileged
    image: willvelida/hello-web-app:1.0.0
    securityContext:
      privileged: true
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 3000
```

To apply this manifest, we can run the following:

```bash
kubectl apply -f .\web-app-privileged.yml
```

If we attempt to apply the privileged pod manifest, we should get the following error message:

```bash
Error from server (Forbidden): error when creating ".\\web-app-privileged.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [azurepolicy-k8sazurev2noprivilege-cfaed4a04d3ae7b0fb6c] Privileged container is not allowed: web-app-privileged, securityContext: {"privileged": true}
```

We've got an error message saying that privileged containers are not allowed. This is a pretty good error message, as not only does it tell us what the problem is, but also provides us with the property within our pod YAML manifest that's causing the issue.

Let's remove the securityContext property in our manifest and redeploy the Pod. I've created a new `.yml` file called `web-app.yml` and defined it like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: web-app
    image: willvelida/hello-web-app:1.0.0
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 3000
```

This time, the pod has been successfully created!

![Image showing a command line output stating that our non-privileged pod has been successfully created](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pm16cysiblrezx5xa437.png)

## Checking our compliance status

We can also view the state of our compliance in the Azure Portal. In the portal, we can see what initiatives and policies we have applied to a particular scope and see a percentage score representing our overall compliance level, which resources are compliant or not, and how many policies are non-compliant.

![Image showing the overall compliance state of our AKS cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dr74urlzp1tj3zn3q1kk.png)

We can drill down into initiatives to see the exact policies that make our resources non-compliant.

![Image showing a detailed view of the compliance state of our AKS cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mpj76o2xne4q49tlzorh.png)

## Conclusion

In this article, I discussed what Azure Policy is and how it integrates with Azure Kubernetes Service. I then talked about how we can apply Azure Policy to our AKS cluster and how we can add Initiatives and Policies to our cluster using Terraform. We then tested the policies by attempting to apply a privileged container, which we couldn't do due to Azure Policy.

If you want to learn more about Azure Policy and how it applies to Azure Kubernetes Service, check out the following articles:

- [Secure your Azure Kubernetes Service (AKS) clusters with Azure Policy](https://learn.microsoft.com/azure/aks/use-azure-policy?WT.mc_id=MVP_400037)
- [What is Azure Policy?](https://learn.microsoft.com/azure/governance/policy/overview?WT.mc_id=MVP_400037)

If you want to see the code for my AKS cluster, please check out my [GitHub](https://github.com/willvelida/my-aks-cluster)

If you have any questions about this, please feel free to reach out to me on [BlueSky](https://bsky.app/profile/willvelida.com)! I'm loving BlueSky at the moment. It has a much better UX and performance than Twitter these days.

Until next time, Happy coding! 🤓🖥️


