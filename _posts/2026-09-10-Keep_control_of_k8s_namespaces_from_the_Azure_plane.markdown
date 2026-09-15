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
Considering the AKS context, managing a native kubernetes object implies access to the kubernetes plane. It's easily done witn an az aks command, but still...

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

At this point, we can create a namespace with `kubectl`, if we do know a bit of kubernetes cli.

```zsh

➜  ~ k $cil create ns k8s-ns-demo -o yaml

```

```yaml

apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2026-09-11T17:38:08Z"
  labels:
    kubernetes.io/metadata.name: k8s-ns-demo
  name: k8s-ns-demo
  resourceVersion: "65062"
  uid: 320dd9e7-5c98-4683-b40f-1f3b91197150
spec:
  finalizers:
  - kubernetes
status:
  phase: Active


```

Also, we mention that securing the namespace requires additional kubernetes objects, that the Azure person may not know. Typically:

- securing network inside andf outside of the namespace with networkpolicies.
- managing access with kubernetes roles and rolebindings.
- controlling resources consuption but that's the exception because requests and limit can be configured directly in the namespace ^^

It's starting to make a lot of kubernetes resources, And that's where the managed namespace comes in the equation.

Refering to the [documentation](https://learn.microsoft.com/en-us/azure/aks/managed-namespaces?pivots=azure-portal), the Azure managed namespace provides a way to isolate teams and workloads. But that's typically just what the kubernetes native namespace does.
The important part here is the "Azure managed" part. 

Think of the Azure managed namespace as an Azure plane abstraction to:

- a namespace
- a networkpolicy to manage ingress or egress traffic
- requests ad limit at theamespace level
- labels and annotations that we would like added to the namespace.

Additionaly, it comes with some interesting rbac option. Again from the documentation, we have the following roles that can be assigned to managed the managed namespaces from the Azue plane.

| Role 	| Description|
|-|-|
| [Azure Kubernetes Service Namespace Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-kubernetes-service-namespace-contributor) 	| Allows access to create, update, and delete managed namespaces on a cluster.|
| [Azure Kubernetes Service Namespace User](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-kubernetes-service-namespace-user) | Allows read-only access to a managed namespace on a cluster. Allows access to list credentials on the namespace.|


And those to access resources in the kubernetes plane.

| Role | Description |
|-|-|
| Azure Kubernetes Service RBAC Reader | Allows read-only access to see most objects in a namespace. It doesn't allow viewing roles or role bindings. This role doesn't allow viewing Secrets, since reading the contents of Secrets enables access to ServiceAccount credentials in the namespace, which would allow API access as any ServiceAccount  |in the namespace (a form of privilege escalation).
| Azure Kubernetes Service RBAC Writer | Allows read/write access to most objects in a namespace. This role doesn't allow viewing or modifying roles or role bindings. However, this role allows accessing Secrets and running Pods as any ServiceAccount in the namespace, so it can be used to gain the API access levels of any ServiceAccount in the  |namespace.
| Azure Kubernetes Service RBAC Admin | Allows read/write access to most resources in a namespace, including the ability to create roles and role bindings within the namespace. This role doesn't allow write access to resource quota or to the namespace itself. |

It definitely brings a huge change to the table: granualar authorization to the namespace scope from the Azure plane, which was more difficult before.

Ok let's play 	&#128513;


## 2. Creating and managing a managed namespace

We'll start be creating a single managed namespace. 

It's doeable with the az cli, or the portal, or `bicep`. So we'll pick the 4th option a.k.a terraform.

Obviously, i would have liste terraform if it was an available resource in the `azurerm` provider, but it'snot, so we'll use `azapi`, because if something is available on `bicep`, it is also on `azapi`.

A short research on the [documentation](https://learn.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters/managednamespaces?pivots=deployment-language-terraform) (or you could probably vibecode it, but where's the fun in that I'll ask you &#128541;)

```go

resource "azapi_resource" "ManagedNs" {


  type = "Microsoft.ContainerService/managedClusters/managedNamespaces@2026-04-02-preview"
  name = "managedns"
  parent_id = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourcegroups/<rg_name>/providers/Microsoft.ContainerService/managedClusters/<luster_name>"
  location = "<aks_location>"
  tags = merge(var.MandatoryTags, var.OptionalTags, var.ExtraTags)
  body = {
    properties = {
      adoptionPolicy = "Always"
      annotations = {
        "someannotation" = "somevalue"
      }
      defaultNetworkPolicy = {
        egress = "AllowAll"
        ingress = "DenyAll"
      }
      defaultResourceQuota = {
        cpuLimit = "500m"
        cpuRequest = "750m"
        memoryLimit = "256Mi"
        memoryRequest = "128Mi"
      }
      deletePolicy = "Delete"
      labels = {
        "pod-security.kubernetes.io/audit" = "restricted"
        "pod-security.kubernetes.io/audit-version" = "latest"
      }
    }
  }
}

```

If everyting goes well, you should have a managed namespace, that you can see through the az cli, or on the portal

```zsh

➜  ~ az aks namespace list -o table

Name                ResourceGroup    Location
------------------  ---------------  -------------
aks-lab1/managedns  rsg-cluster-aks  francecentral

```

![illustration1](/assets/managedns/managedns001.png)

![illustration2](/assets/managedns/managedns002.png)

![illustration3](/assets/managedns/managedns003.png)


Leveraging the rbac roles we introduced earlier, we can grant access to the namespace level, without touching the kubernetes cli, for now. 

Fun fact though, when we list the managed namespace from the cli or from the portal, we do have a resource group property which hint tha twe should be able to choose the target rg of the managed namespace.
However, we can only refer to the cluster own resource group in the cli, and there is no resourceGroup or anything similar in the arm api. So we are stuck with creating the managed namespace in the cluster's rg.



## 3. Managed namespaces vs native namespaces



## 4. Summary







