# Helpful Links

## Kubernetes

<https://kubernetes.io/>

Kubernetes (k8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

### What is k8s?

<https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/>

### Learn k8s basics

#### Master Components

Master components provide the cluster’s control plane. Master components make global decisions about the cluster (for example, scheduling), and detecting and responding to cluster events (starting up a new pod when a replication controller’s ‘replicas’ field is unsatisfied).

AKS provides the k8s master components as a service

####Node Components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

In Azure, a node is a VM

Your applications run on nodes

The Node has a container runtime (usually Docker, but could be rkt)

### Kubernetes object management

<https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/

For this walkthrough, we will be using declaritive object configuration only


### AKS Docs

<https://docs.microsoft.com/en-us/azure/aks/>


### AKS walkthrough on docs

<https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough>
