---
layout: post
title:  "Keep control of k8s namespaces from the Azure plane"
date:   2026-09-10 18:00:00 +0200
year: 2026
categories: Kubernetes AKS
---

Hello!

Summer being almost done, it's time fto go back to tech stuff.
In this article, we'll have a look at a not so new feature of AKS called the managed namespace.
The fact that it's been around for some time does not diminish its value, so we'll take some time on the topic ^^.


Our agenda:


1. Overview of the managed namespace in AKS
2. Creating and managing a managed namespace
3. Managed namespaces vs native namespaces

## 1. Overview of the managed namespace in AKS

Before diving in Azure managed namespaces, a small reminder of what a namespace is.

If we take the [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), we get the following: 

*In Kubernetes, namespaces provide a mechanism for isolating groups of resources within a single cluster. Names of resources need to be unique within a namespace, but not across namespaces. Namespace-based scoping is applicable only for namespaced objects (e.g. Deployments, Services, etc.) and not for cluster-wide objects (e.g. StorageClass, Nodes, PersistentVolumes, etc.).*

To put it simply, it's a logical container allowing organization, but also, with the addition of configurations and other objects, some measure of security segmentation. 
Let's insist on this: **without additional configuration, namespace is not natively an isolation mechanism regarding security.**

If I was an Azure guy, I would compare a namespace to a resource group &#128527;.

Also, even in a cluster intended for a single team, we'll find some default namespaces.

```zsh

➜  ~ k $cil1 get ns
NAME                   STATUS   AGE
default                Active   18d
kube-node-lease        Active   18d
kube-public            Active   18d
kube-system            Active   18d

```

Ok that's fine, but why would I need a managed namespace then?

Well, first thing first, a namespace, while simple in its basic definition, is a kubernetes object.

```zsh

➜  ~ k $cil1 get namespaces kube-system -o yaml

```

```yaml

apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2026-08-19T21:43:07Z"
  labels:
    kubernetes.io/metadata.name: kube-system
  name: kube-system
  resourceVersion: "4"
  uid: b3374276-249e-4853-a99c-0f96d2d6f362
spec:
  finalizers:
  - kubernetes
status:
  phase: Active

```

So, putting ourselves in an AKS context, we may be more proficient in the Azure plane rather than the k8s one.
Still considering the AKS context, managing a native kubernetes object implies access to the kubernetes plane. It's easily done witn an az aks command, but still...

```zsh

➜  ~ az aks list -o table
Name      Location       ResourceGroup    KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
--------  -------------  ---------------  -------------------  --------------------------  -------------------  --------------------------------------------
aks-lab1  francecentral  rsg-cluster-aks  1.36.0               1.36.0                      Succeeded            akslab1-48svf6dv.hcp.francecentral.azmk8s.io

➜  ~ az aks get-credentials -n aks-lab1 -g rsg-cluster-aks
A different object named aks-lab1 already exists in your kubeconfig file.
Overwrite? (y/n): y
A different object named clusterUser_rsg-cluster-aks_aks-lab1 already exists in your kubeconfig file.
Overwrite? (y/n): y
Merged "aks-lab1" as current context in /Users/df/.kube/config
Converted kubeconfig to use Azure CLI authentication.

```

Last, remember, we mention that securing the namespace requires additional kubernetes objects, that the Azure person may not know. Typically:

- securing network inside andf outside of the namespace with networkpolicies.
- managing access with kubernetes roles and rolebindings.
- controlling resources consuption


## 2. Creating and managing a managed namespace



## 3. Managed namespaces vs native namespaces



## 4. Summary







