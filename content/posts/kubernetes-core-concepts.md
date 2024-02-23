---
title: "CKAD Reference - Kubernetes: Core Concepts"
date: 2024-02-23
draft: false
tags: ["Azure","Cloud Native", "Platform Engineering", "Kubernetes", "CKAD"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3hwgv1he508nqm0c3tqe.png
    alt: "Core Concepts in Kubernetes"
    caption: "Understanding the core concepts of Kubernetes, and how to use the kubectl to create primitives in Kubernetes, is an important first step in your Kubernetes journey"
---

I'm studying for my CKAD exam, so this article is serving as a reference for me to cover the core concepts in that exam. This article will cover Pods, Namespaces, the `kubectl` CLI tool, and how we can use it to create objects in Kubernetes.

## Kubernetes Primitives

These are the basic building blocks in Kubernetes for building and operating applications that you host on it. These could include Pod, Deployments, Services etc

Every Kubernetes primitive follows a structure which is reflected in the manifest of the object (This is usually a YAML file). These sections are usually found in each Kubernetes primitive:

- **API Version** - The K8s API version that defines the structure of the primitive. You may see different prefixes for each version, and some API versions may be alpha, or beta versions. We can list API versions that are compatible for the version of our cluster by running `kubectl api -versions`
- **Kind** - This defines the primitive type (Pod, Service, etc.).
- **Metadata** - This describes high-level information about the object. This could include the name of the object, which namespace it lives in etc.
- **Spec** - Or specification which declares the desired state. This includes what image should run the container, environment variables are needed etc.
- **Status** - This describes the *actual* state of the object. The K8s controllers will try to always reflect the desired state into the actual state.

## Interacting with the Kubernetes Cluster

`kubectl` is the primary tool that we use to interact with our Kubernetes cluster from the command line. A simple command will consist of the following:

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

So a real `kubectl` command would look something like this:

```bash
kubectl get pod app -o app.yaml
```

## Working with objects in Kubernetes

We can create objects in Kubernetes either imperatively or declaratively. 

Imperatively, we don't need a manifest definition. We can just use the `kubectl run` or `kubectl create` commands to create objects, If we need to provide configuration, we can do use via the command-line options, like so:

```bash
kubectl run webui --image=mywebuiiamge --port=80
# pod/webui created
```

Using the declarative approach, we define our Kubernetes objects using a manifest file. We then create the object using `kubectl create` or `kubectl apply`. This gives us the benefit of improved maintenance of our object.

```bash
kubectl create -f webui.yaml
# pod/webui created
```

You may have seen that instead of the `create` command, we can use the `apply` command instead. The `create` command will create a new object, and if you try to use `create` for an existing object, this will result in an error.

The `apply` command allows us to update an existing object in full, or incrementally. If you use `apply` for an object that doesn't exist in your cluster, it'll create the object:

```bash
kubectl apply -f webui.yaml
# pod/webui configured
```

We can delete Kubernetes objects using the `kubectl delete` command. This is useful when we want to delete objects that we don't need anymore. We can delete an object either:

- by providing the name of the object, or
- by deleting the object by pointing to the manifest file that created it.

```bash
kubectl delete pod webui
# or
kubectl delete -f webui.yaml
```

We can edit the state of live objects in our cluster like so:

```bash
kubectl edit pod webui
```

Or we can replace the object. We can use the `replace` command to overwrite the configuration of our live objects with one that we've defined in our manifest file:

```bash
kubectl replace -f webui.yaml
```

## Pods in Kubernetes

Pods are the primary primitive in Kubernetes. It lets you run containerized applications. In most cases, Pods and containers have a one-to-one mapping between them. There are some cases where you can have more than one container in a single Pod. Pods can also consume persistent storage, configuration, etc. 

A container will package an application that includes its environment and configuration. It usually contains the operating system, source code, and dependencies. 

We turn applications into containers by containerizing them (sounds obvious right?). We do this by defining instructions in a Dockerfile, which defines the build process for that application. From that Dockerfile, we build an image, and then publish that image to a registry for others to consume.

### Creating Pods

Once our image is ready to be consumed, we can use it in our Pod definition. When we create our Pod, the container runtime engine (CRI) will check if the container images already exists locally, and if not, it will download it from a container image registry.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4ai2a0dblhtwgfxcyd4f.png)

We can create pods using the `run` command. This creates a Pod imperatively. For example:

```bash
kubectl run webui --image=myacr/webui --restart=Never --port=8080
```

You can also create Pods from a YAML manifest file. For example, we could have the following Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webui
  labels:
    name: webui
    env: dev
spec:
  containers:
  - name: webui
    image: myacr/webui:latest
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
```

And then create the Pod by using either `kubectl create` or `kubectl apply`:

```bash
kubectl create -f webui.yaml
```

### Retrieving Pod details

Once our Pod is created, we can inspect it using the `kubectl get` command. To list all pods in our cluster, we can run the following:

```bash
kubectl get pods
NAME READY STATUS RESTARTS AGE
webui 1/1 Running 0 69s
```

If we want to query a specific pod, all we need to do is pass in the name like so:

```bash
kubectl get pods webui
NAME READY STATUS RESTARTS AGE
webui 1/1 Running 0 69s
```

When we list pods, we may not see it running right away. This is because Kubernetes has asynchronous control loops, so it takes a couple of seconds to retrieve the image and start the container. Pods have several phases in its lifecycle:

| **Status** | **Description** |
| ---------- | --------------- |
| Pending | Pod has been accepted by Kubernetes system, but the container images have not been created |
| Running | At least one container is running, or is starting |
| Succeeded | All containers in the Pod have terminated successfully |
| Failed | Containers in Pod have terminated, and at least one failed with an error |
| Unknown | The state of the Pod is unknown |

We can get more details from our Pod by running the `kubectl describe` command, for example:

```bash
kubectl describe pods webui
```

The output for this command will contain the metadata for the Pod, containers that it runs and the event log. This can be quite long, so if we want to retrieve specific parts of information, we can use the following:

```bash
kubectl describe pods webui | grep Image:
# produces Image: myacr/webui
```

We can also retrieve the log output of a container using `kubectl logs` like so:

```bash
kubectl logs webui
```

We can also use the `-f` flag to stream the logs in real time. One thing to keep in mind that if a container is restarted for whatever reason, we will lose the logs. The `logs` command only produces logs for the current container.

You can get the logs of the previous container by using the `-p` flag. this is handy when we want to identify the cause of the restart.

### Commands in containers

To run commands in our containers, we can use the `kubectl exec` command to open a shell in our container, like so:

```bash
kubectl exec -it webui -- /bin/sh
```

We don't have to provide a resource type, as this command only works for a Pod, and the two dashes (--) separate the `exec` command and what we want to run inside of the container.

### Deleting Pods

Deleting pods can be done using the `kubectl delete` command:

```bash
kubectl delete pod webui
```

Kubernetes will try to delete the Pod gracefully, so it will finish active requests to the Pod so that end users aren't disrupted. This can take between 5-30 seconds.

We can also delete our Pods by pointing the delete command to the YAML manifest that created it:

```bash
kubectl delete -f webui.yaml
```

## Namespaces

Namespaces in Kubernetes are used to represent the scope for object names, and help isolate objects in Kubernetes by team/responsibility/applications etc.

We can use the `kubectl` to perform some basic operations in our cluster. To list them, we can run the following:

```bash
kubectl get namespaces

# Produces
NAME STATUS AGE
default Active 14d
kube-node-lease Active 14d
kube-public Active 14d
kube-system Active 14d
```

The `default` namespace hosts objects that aren't assigned to a specific namespace. Namespaces that begin with a `kube-` prefix are not end-user namepsaces (As an app developer, you don't need to work with these).

If we want to create a new namespace, we can use the `create namespace` command. For example:

```bash
kubectl create namespace my-new-namespace
```

We can also represent a namespace in a YAML manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: my-new-namespace
```

Once our namespace has been created, we can create objects in it. We can specify the namespace to create the object by using the `--namespace` or `-n` flag:

```bash
kubectl get pods -n my-new-namespace
```

Finally, we can delete a namespace using `kubectl delete`. Deleting a namespace will also delete the objects in that namespace automatically:

```bash
kubectl delete namespace my-new-namespace
```

## Conclusion

In this article, we covered Pods, Namespaces, the `kubectl` CLI tool, and how we can use it to create objects in Kubernetes.

If you want to learn more about the concepts we talked about here, check out the following articles:

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Containers](https://kubernetes.io/docs/concepts/containers/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

If you have any questions on the above, feel free to reach out to me on twitter [@willvelida](https://twitter.com/willvelida)

Until next time, Happy coding! 🤓🖥️