---
layout: post
title:  "The state of Identity Management in AKS"
date:   2022-09-23 09:00:00 +0200
categories: AKS
---

Hello people!

As the title's saying, this article is about Identity management in Azure Kubernetes Services.
This service evolved quite a lot, as expected from a managed Kubernetes in the Cloud.
With this fast pace of evolution comes the hard time following the changes. Specificially for the Identity management, it is critical to be clear on the configurations and options available.

This article aims to do just that, with the following plan:

1. A rapid review of the Kubernetes mapped in Azure context
2. IAM for the control plane
3. IAM for the worker plane
4. IAM for hosted workloads.

That being said, let's get started.  
  
## 1. A rapid review of the Kubernetes mapped in Azure context

This part is a short one, just to summarize what we are looking at in Kubernetes and which parts are subject to IAM. 
In a very simplified view, Kubernetes is composed of the control plane, which, as the name implies, control everything and the worker plane which, being controlled by the control plane, host Kubernetes workloads.
Kubernetes workloads are composed of differents Kubernetes API objects, which are used by applications.

![Illustration 1](/assets/aksidentity/aksidentity001.png)  

Now, what we should ask ourselves here is which part may interact with IAM topics. And the answer is, let's be blunt: All of them.

Because the control plane is so important in the integrity of Kubernetes, there are indeed a lot of topics around IAM, and we'll see that in the following part.
But there is also things to consider regarding the worker plane, and specifically Kubelet and how it interact with others things. 
Because we are talking about AKS, obviously, we will have a look at Kubelet interactions with the Azure platform.
Last, but not least, and unfortunately, not too deep, are the questions regarding how to managed workloads interaction with the Azure platform. We will keep on a higl level view for this, because it would require a dedicated article.

Let's move on with the Azure mapping then.

![Illustration 2](/assets/aksidentity/aksidentity002.png)  
  
AKS is a PaaS service, and as such is largely managed by the Azure platform.
But also, it has a lot af building blocks that are seen in the IaaS category:

- Virtual Network and subnet
- Virtual Machine Scale sets
- Network Security Groups
- Load Balancer

Those we get once the service is built. But before building it, we need:

- A resource group, obviously
- Usually, a Virtual Network, because we do create it before hand
- an SSH key
- An Azure Active Directory Group, more on that later
- Managed Identities, System Assigned or User assigned, also more on that in the following part.

Enough with this architecture review, we can now move on and see what are those AAD group and managed identities used for.

## 2. IAM for the control plane

In the heart of the topic, at last ^^

This part will probably be the most rich in terms of concepts, because, as we said, the control plane... control everything!

### 2.1. Azure Active Directory Integration

We mentioned the requirement of an Azure Active Directory Group. And that's because AKS can integrate with Azure AD for authentication.
That means that we don't need to manage from the `kubectl` actions around csr, because our Identity provider is the same as the one for the Azure platform: Azure Active Directory.

The integration has evolved for the best and we can now benefit from the Managed Integration, as opposite to the now called legacy integration.
In the previous version, we would have need to managed Service principals in Azure Active Directory, in the form of 2 application registrations.
One was called the Server App, and the other the Client App. 
In a simplified way, the Server app had access to the AAD tenant, te get information on users, and to authenticate on behlf of the user once the authorization were validated. 
The Client app was the front App on which user interacted, and it had authorization on the Server app.
The Server app came with a secret that need to be managed, and all in all, the configuration was not trivial.

But that was before, and now, while the process is clearly similar under the hood, we only care about specifying an Azure AD group which is granted cluster admin access on the cluster at creation.
In terms of configuration, at build time, this group's object Id and the Azure AD tenant Id are specified in the configuration.

```json

"aadProfile": {
      "adminGroupObjectIDs": [
        "00000000-0000-0000-0000-00000000000"
      ],
      "adminUsers": null,
      "clientAppId": null,
      "enableAzureRbac": true,
      "managed": true,
      "serverAppId": null,
      "serverAppSecret": null,
      "tenantId": "00000000-0000-0000-0000-000000000000"
}

```

This setting is accessible through the API, with any tool talking to the API, such as az cli or terraform. 

With az cli

```bash

az aks create -g MyResourceGroup -n MyManagedCluster --enable-aad --aad-admin-group-object-ids <id-1,id-2> --aad-tenant-id <id>

```

```bash


resource "azurerm_kubernetes_cluster" "AKSRBACCNI" {

================================truncated==================================
  
  role_based_access_control_enabled       = true

  azure_active_directory_role_based_access_control {
      managed                             = true
      azure_rbac_enabled                  = true
      admin_group_object_ids              = var.AKSClusterAdminsIds

  }
================================truncated==================================


}

```

As a member of the group that is assigned to the cluster as an admin, it's quite easy to authenticate and interact with the cluster:

```bash

az aks get-credentials -n aks-1 -g rsg-aksIdentityState1

Merged "aks-1" as current context in /home/df/.kube/config

kubelogin convert-kubeconfig

kubectl get namespaces 

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code DXJJZWZMT to authenticate.

```

![Illustration 3](/assets/aksidentity/aksidentity003.png) 

```bash

NAME                STATUS   AGE
calico-system       Active   7d11h
default             Active   7d11h
gatekeeper-system   Active   7d11h
kube-node-lease     Active   7d11h
kube-public         Active   7d11h
kube-system         Active   7d11h
tigera-operator     Active   7d11h

```

As a matter of fact, if we check the Cluster role bindings on the cluster, we can see a clusterrolebinding that bind the cluster admin role and that is called ``:

```bash

k get clusterrolebindings.rbac.authorization.k8s.io | grep aad
aks-cluster-admin-binding-aad                          ClusterRole/cluster-admin                                                          7d11h

k get clusterrolebindings.rbac.authorization.k8s.io aks-cluster-admin-binding-aad -o yaml

```

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:

========truncated==========

  name: aks-cluster-admin-binding-aad
  resourceVersion: "505"
  uid: 9e9d78b5-924a-4ac2-84fc-456bcd71a386
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: 546e2d3b-450e-4049-8f9c-423e1da3444c

```

In the subject section, the name of the group is actually the object id of the group that is assigned 

Now, let's see how we can manage access to users.

### 2.2. Azure Active Directory Integration