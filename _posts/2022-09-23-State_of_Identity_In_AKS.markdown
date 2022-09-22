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

### 2.1. Authentication with Azure Active Directory Integration

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

Or in a terraform configuration:
  
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
![Illustration 4](/assets/aksidentity/aksidentity004.png)  
![Illustration 5](/assets/aksidentity/aksidentity005.png)  
![Illustration 6](/assets/aksidentity/aksidentity006.png)  



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
  name: 00000000-0000-0000-0000-000000000000

```

If we look in the subjects section, we can find a kind set to `group` with a name defined as an object id, corresponding to the object id of the group that was specified in the aad profile.

Ok the admin access seems pretty clear, let's see the way we manage access for users.

### 2.2. Authorization for non admin user

#### 2.2.1. RBAC from the Azure plan
  
First let's step back a little.

In the previous section, we were able to demonstrate that, in a AKS managed Azure AD, as a member of the Azure Active Directory Group assigned to the cluster, we can get cluster admin access (in the kubernetes sense).

But we did take a shortcut.

To begin, let's consider this: the AKS cluster is an objet living in Azure

So the first step is to have access in the Azure plane.
For that, we have 2 roles in the list of Azure Built-in roles:

- Azure Kubernetes Service User

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "List cluster user credential action.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/4abbcc35-e782-43d8-92c5-2d3f1bd2253f",
  "name": "4abbcc35-e782-43d8-92c5-2d3f1bd2253f",
  "permissions": [
    {
      "actions": [
        "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action",
        "Microsoft.ContainerService/managedClusters/read"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "Azure Kubernetes Service Cluster User Role",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

- Azure Kubernetes Service Cluster Admin Role 

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "List cluster admin credential action.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/0ab0b1a8-8aac-4efd-b8c2-3ee1fb270be8",
  "name": "0ab0b1a8-8aac-4efd-b8c2-3ee1fb270be8",
  "permissions": [
    {
      "actions": [
        "Microsoft.ContainerService/managedClusters/listClusterAdminCredential/action",
        "Microsoft.ContainerService/managedClusters/accessProfiles/listCredential/action",
        "Microsoft.ContainerService/managedClusters/read",
        "Microsoft.ContainerService/managedClusters/runcommand/action"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "Azure Kubernetes Service Cluster Admin Role",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

The important part, specifically for the **Azure Kubernetes Service Cluster User Role** is the action `Microsoft.ContainerService/managedClusters/listClusterUserCredential/action`.

With this, a user is able the get AKS credentials through `az aks get-credentials` as we did previously. And that's all.
Following the process of logon, we would get a result like below:

```bash

az aks get-credentials -n aks-1 -g rsg-aksIdentityState1
Merged "aks-1" as current context in /home/penny/.kube/config

k get ns

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code D825HAGUL to authenticate.

Error from server (Forbidden): namespaces is forbidden: User "penny@teknews.cloud" cannot list resource "namespaces" in API group "" at the cluster scope: User does not have access to the resource in Azure. Update role assignment to allow access.

```

And this is perfectly logical, because the actions in the role only grant access to get the credentials, nothing more. But is it possible to do more only from the Azure plane?

If we have a look on the cluster configuration, we can see that there is one option mentionning Azure RBAC:

![Illustration 7](/assets/aksidentity/aksidentity007.png)  

We can check the status of the cluster through the cli:

```bash

az aks show -n aks-1 -g rsg-aksIdentityState1 | jq .aadProfile.enableAzureRbac
true

```

And this allows to use other Azure roles, such as for example the `Azure Kubernetes Service RBAC Reader`

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "Allows read-only access to see most objects in a namespace. It does not allow viewing roles or role bindings. This role does not allow viewing Secrets, since reading the contents of Secrets enables access to ServiceAccount credentials in the namespace, which would allow API access as any ServiceAccount in the namespace (a form of privilege escalation). Applying this role at cluster scope will give access across all namespaces.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/7f6c6a51-bcf8-42ba-9220-52d62157d7db",
  "name": "7f6c6a51-bcf8-42ba-9220-52d62157d7db",
  "permissions": [
    {
      "actions": [
        "Microsoft.Authorization/*/read",
        "Microsoft.Insights/alertRules/*",
        "Microsoft.Resources/deployments/write",
        "Microsoft.Resources/subscriptions/operationresults/read",
        "Microsoft.Resources/subscriptions/read",
        "Microsoft.Resources/subscriptions/resourceGroups/read",
        "Microsoft.Support/*"
      ],
      "notActions": [],
      "dataActions": [
        "Microsoft.ContainerService/managedClusters/apps/controllerrevisions/read",
        "Microsoft.ContainerService/managedClusters/apps/daemonsets/read",
        "Microsoft.ContainerService/managedClusters/apps/deployments/read",
        "Microsoft.ContainerService/managedClusters/apps/replicasets/read",
        "Microsoft.ContainerService/managedClusters/apps/statefulsets/read",
        "Microsoft.ContainerService/managedClusters/autoscaling/horizontalpodautoscalers/read",
        "Microsoft.ContainerService/managedClusters/batch/cronjobs/read",
        "Microsoft.ContainerService/managedClusters/batch/jobs/read",
        "Microsoft.ContainerService/managedClusters/configmaps/read",
        "Microsoft.ContainerService/managedClusters/endpoints/read",
        "Microsoft.ContainerService/managedClusters/events.k8s.io/events/read",
        "Microsoft.ContainerService/managedClusters/events/read",
        "Microsoft.ContainerService/managedClusters/extensions/daemonsets/read",
        "Microsoft.ContainerService/managedClusters/extensions/deployments/read",
        "Microsoft.ContainerService/managedClusters/extensions/ingresses/read",
        "Microsoft.ContainerService/managedClusters/extensions/networkpolicies/read",
        "Microsoft.ContainerService/managedClusters/extensions/replicasets/read",
        "Microsoft.ContainerService/managedClusters/limitranges/read",
        "Microsoft.ContainerService/managedClusters/namespaces/read",
        "Microsoft.ContainerService/managedClusters/networking.k8s.io/ingresses/read",
        "Microsoft.ContainerService/managedClusters/networking.k8s.io/networkpolicies/read",
        "Microsoft.ContainerService/managedClusters/persistentvolumeclaims/read",
        "Microsoft.ContainerService/managedClusters/pods/read",
        "Microsoft.ContainerService/managedClusters/policy/poddisruptionbudgets/read",
        "Microsoft.ContainerService/managedClusters/replicationcontrollers/read",
        "Microsoft.ContainerService/managedClusters/replicationcontrollers/read",
        "Microsoft.ContainerService/managedClusters/resourcequotas/read",
        "Microsoft.ContainerService/managedClusters/serviceaccounts/read",
        "Microsoft.ContainerService/managedClusters/services/read"
      ],
      "notDataActions": []
    }
  ],
  "roleName": "Azure Kubernetes Service RBAC Reader",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

This time, the role comes with **dataActions** which relate to the Kubernetes plane.

Granting this role to a user on a cluster withe the Azure RBAC set as true will us access to the Kubernetes plane:

```bash

k get ns in cluster 1

```

However, this same role on a cluster with only Kubernetes Native RBAC:

```bash

az aks show -n aks-2 -g rsg-aksIdentityState2 | jq .aadProfile.enableAzureRbac
false

```

Does not grant access to the cluster, because it does not accept roles from the Azure plane to interact with Kubernetes object:

```bash

k get ns in cluster 2

```

So here comes the limit in terms of authorization management for kubernetes in the Azure plane. To be more grnaular on the kubernetes side, we need to look at the kubernetes native rbac.

#### 2.2. RBAC from the Kubernetes plan

#### 2.3 The weak link of inheritance and how to mitigate it

#### 2.4. MAnaged Identities in the control plane


### 3. IAM fo rthe worker plane

### 4. IAM for workloads hosted on kubernetes