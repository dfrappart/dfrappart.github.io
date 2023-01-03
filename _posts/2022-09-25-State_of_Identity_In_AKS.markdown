---
layout: post
title:  "The state of Identity Management in AKS"
date:   2022-09-25 09:00:00 +0200
year: 2022
categories: AKS Security
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
In a simplified way, the Server app had access to the AAD tenant, to get information on users, and to authenticate on behalf of the user once the authorization were validated.  
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


resource "azurerm_kubernetes_cluster" "AKS" {

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

As a matter of fact, if we check the Cluster role bindings on the cluster, we can see a clusterrolebinding that bind the cluster admin role and that is called `aks-cluster-admin-binding-aad`:
  
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

In the previous section, we were able to demonstrate that, in a AKS managed Azure AD, as a member of the Azure Active Directory Group assigned to the cluster, we can get cluster admin access (in the Kubernetes sense).

But we did take a shortcut.

To begin, let's consider this: the AKS cluster is an objet living in Azure

So the first step is to have access in the Azure plane.
For that, we have 2 roles in the list of Azure Built-in roles:

- **Azure Kubernetes Service Cluster User**

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

- **Azure Kubernetes Service Cluster Admin Role**  

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

faye [ ~ ]$ az aks get-credentials -n aks-1 -g rsg-aksIdentityState1

Merged "aks-1" as current context in /home/penny/.kube/config

faye [ ~ ]$ k config get-contexts

CURRENT   NAME    CLUSTER   AUTHINFO                                  NAMESPACE
*         aks-1   aks-1     clusterUser_rsg-aksIdentityState1_aks-1  

faye [ ~ ]$ k get ns

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code D825HAGUL to authenticate.

Error from server (Forbidden): namespaces is forbidden: User "faye@teknews.cloud" cannot list resource "namespaces" in API group "" at the cluster scope: User does not have access to the resource in Azure. Update role assignment to allow access.

```

And this is perfectly logical, because the actions in the role only grant access to get the credentials, nothing more.  

![Illustration 7](/assets/aksidentity/aksidentity007.png)  

But is it possible to do more only from the Azure plane?

If we have a look on the cluster configuration, we can see that there is one option mentionning Azure RBAC:

![Illustration 8](/assets/aksidentity/aksidentity008.png)  

We can check the status of the cluster through the cli:

```bash

az aks show -n aks-1 -g rsg-aksIdentityState1 | jq .aadProfile.enableAzureRbac
true

```

And this allows to use other Azure roles, such as for example the **Azure Kubernetes Service RBAC Reader**

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

Granting this role to a user on a cluster with the the Azure RBAC set as true will us access to the Kubernetes plane:

![Illustration 9](/assets/aksidentity/aksidentity009.png)  
  
```bash

jet [ ~ ]$ k config current-context
aks-1

jet [ ~ ]$ k get ns
NAME                STATUS   AGE
calico-system       Active   11d
default             Active   11d
gatekeeper-system   Active   11d
kube-node-lease     Active   11d
kube-public         Active   11d
kube-system         Active   11d
tigera-operator     Active   11d

```

However, this same role on a cluster with only Kubernetes Native RBAC:

```bash

az aks show -n aks-2 -g rsg-aksIdentityState2 | jq .aadProfile.enableAzureRbac
false

```

![Illustration 10](/assets/aksidentity/aksidentity010.png)  

Does not grant access to the cluster, because it does not accept roles from the Azure plane to interact with Kubernetes object:

```bash

jet [ ~ ]$ k config use-context aks-2
Switched to context "aks-2".

jet [ ~ ]$ k get ns
Error from server (Forbidden): namespaces is forbidden: User "jet@teknews.cloud" cannot list resource "namespaces" in API group "" at the cluster scope

```

While the access granted on the Azure plane is still the same, because the cluster `aks-2` is configured to accept only Kubernetes native role, the assignment is not working anymore. In this case, we would need to configure RBAC in the Kubernetes plane.

#### 2.2.2 RBAC from the Kubernetes plan

To illustrate this purpose, we will take another user called Leonard.
Leonard is only assigned the **Azure Kubernetes Service Cluster User Role** on both cluster `aks-1` and `aks-2`.
However we grant him a Kubernetes role on `aks-1`:

```bash

k config current-context
aks-1

k get rolebinding -o wide
NAME             ROLE                AGE   USERS                   GROUPS   SERVICEACCOUNTS
defaultnsadmin   ClusterRole/admin   11d   leonard@teknews.cloud

k describe rolebinding defaultnsadmin
Name:         defaultnsadmin
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  admin
Subjects:
  Kind  Name                   Namespace
  ----  ----                   ---------
  User  leonard@teknews.cloud  

```

Thus, on `aks-1`, Leonard can do things in the default namespace:

```bash

leonard [ ~ ]$ k config use-context aks-1
Switched to context "aks-1".

leonard [ ~ ]$ k run nginxtest --image=nginx
pod/nginxtest created

leonard [ ~ ]$ k get pod
NAME        READY   STATUS    RESTARTS   AGE
nginxtest   1/1     Running   0          5s

```

But not on the cluster `aks-2`:

```bash

leonard [ ~ ]$ k config use-context aks-2
Switched to context "aks-2".

leonard [ ~ ]$ k run pod testnginx --image=nginx
Error from server (Forbidden): pods is forbidden: User "leonard@teknews.cloud" cannot create resource "pods" in API group "" in the namespace "default"

```

#### 2.2.3 The weak link of inheritance and how to mitigate it

![Illustration 11](/assets/aksidentity/aksidentity011.png)  

Let's look back to the role **Azure Kubernetes Service Cluster Admin Role**.
This role includes the action `Microsoft.ContainerService/managedClusters/listClusterAdminCredential/action`. In its own it is not a problem, but if we consider the **Contributor** role:

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "Grants full access to manage all resources, but does not allow you to assign roles in Azure RBAC, manage assignments in Azure Blueprints, or share image galleries.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c",
  "name": "b24988ac-6180-42a0-ab88-20f7382dd24c",
  "permissions": [
    {
      "actions": [
        "*"
      ],
      "notActions": [
        "Microsoft.Authorization/*/Delete",
        "Microsoft.Authorization/*/Write",
        "Microsoft.Authorization/elevateAccess/Action",
        "Microsoft.Blueprint/blueprintAssignments/write",
        "Microsoft.Blueprint/blueprintAssignments/delete",
        "Microsoft.Compute/galleries/share/action"
      ],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "Contributor",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

The trouble here is the actions section with `"*"`, because it does includes `Microsoft.ContainerService/managedClusters/listClusterAdminCredential/action`.
Automatically, users with this role on resource groups or subscriptions containing can get credential admin.
This can be done through the `az aks get-credentials` with the `--admin` parameter.

To illustrate, let's take a user called Penny.
Penny is **Azure Kubernetes Service Cluster User Role**, but also **Contributor**:

![Illustration 12](/assets/aksidentity/aksidentity012.png)  
  
```bash
penny [ ~ ]$ az aks get-credentials -n aks-2 -g rsg-aksIdentityState2 --admin
Merged "aks-2-admin" as current context in /home/penny/.kube/config

penny [ ~ ]$ k config get-contexts
CURRENT   NAME          CLUSTER   AUTHINFO                                   NAMESPACE
*         aks-2-admin   aks-2     clusterAdmin_rsg-aksIdentityState2_aks-2 

penny [ ~ ]$ k get nodes
NAME                              STATUS   ROLES   AGE   VERSION
aks-aksnp02-13614584-vmss000005   Ready    agent   69m   v1.23.8
aks-aksnp02-13614584-vmss000006   Ready    agent   69m   v1.23.8

penny [ ~ ]$ k create namespace testns --dry-run=client
namespace/testns created (dry run)

```
  
This is a problem, caused by the inheritance of RBAC on Azure. But there's a way to avoid that.
Actually, this `--admin` parameter leverage the local admin that is enabled by default on AKS.
Luckily, there's an option to disable this:

![Illustration 13](/assets/aksidentity/aksidentity013.png)  
  
If we test the command on a cluster with this account deactivated, we get the following:

```bash

penny [ ~ ]$ az aks show -n aks-3 -g rsg-aksIdentityState3 | jq .disableLocalAccounts
true

penny [ ~ ]$ az aks get-credentials -n aks-3 -g rsg-aksIdentityState3 --admin
(BadRequest) Getting static credential is not allowed because this cluster is set to disable local accounts.
Code: BadRequest
Message: Getting static credential is not allowed because this cluster is set to disable local accounts.

```

And we can mitigate the inheritance issue.
With that we conclude the RBAC section for the control plane. Let's now have a look to the Managed Identity topic.

### 2.3. Managed Identities in the control plane
  
This is a quite short section when compared to the previous one. An AKS cluster leverages managed identity.
However, while it is something that is often configurable after the build, it is activated by default in AKS case:
There is always a managed identity associated to the control plane.

What we can do, on the other hand, is specify a user assigned identity rather than a system assigned one.

![Illustration 14](/assets/aksidentity/aksidentity014.png)  
  
```bash

az aks show -n aks-1 -g rsg-aksIdentityState1 | jq .identity

{
  "principalId": null,
  "tenantId": null,
  "type": "UserAssigned",
  "userAssignedIdentities": {
    "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksIdentityState1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/uaiaks1": {
      "clientId": "00000000-0000-0000-0000-000000000000",
      "principalId": "00000000-0000-0000-0000-000000000000"
    }
  }
}

```

One use case for the user assigned identity is when the assignment is required for the creation of the cluster.
For example, if a cluster is configured to be private, and the DNS zone is already created, then the cluster will need access on this DNS zone before the cluster can be created, to be registered. For further information, have a look at my article on private cluster [here](https://blog.teknews.cloud/aks/2022/07/07/AKS_private_cluster_through_the_terraform_lens.html).  

And with that we are at last finished with the control plane.
  
## 3. IAM for the worker plane

In the worker plane, there is not so much IAM things as for the control plane. All in all, we just have managed identities which are associated to the worker plane.
From the Azure perspective, those identities are always located in the resource group created at the cluster creation.

![Illustration 15](/assets/aksidentity/aksidentity015.png)  
  
![Illustration 16](/assets/aksidentity/aksidentity016.png)  

And they are associated with the VMSS created for Kubernetes node pools.

![Illustration 17](/assets/aksidentity/aksidentity017.png)  

There's a hint in the name of the identities. One is used the Kubelet agent on the node. This identity is always ended with a suffix `agent-pool`, and we can check it in the cluster properties:

```bash

az aks show -n aks-1 -g rsg-aksIdentityState1 | jq .identityProfile

{
  "kubeletidentity": {
    "clientId": "00000000-0000-0000-0000-000000000000",
    "objectId": "00000000-0000-0000-0000-000000000000",
    "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-dfitcfr-dev-tfmodule-aksobjects1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks-1-agentpool"
  }
}

```

As for the control plane, if it is not specified, a system assigned identity is used rather than a user assigned one. If role assignments are required for Kubelet, it may be easier to use a user assigned identity and grant access before or at the same time than the cluster creation.

Now, for the others identities, again, there's a hint in the names. For example, if an identity contains *policy* in its name, it's because the Azure Policy agent is enabled:

```bash

az aks show -n aks-1 -g rsg-aksIdentityState1 | jq .addonProfiles.azurepolicy

{
  "config": null,
  "enabled": true,
  "identity": {
    "clientId": "00000000-0000-0000-0000-000000000000",
    "objectId": "00000000-0000-0000-0000-000000000000",
    "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-dfitcfr-dev-tfmodule-aksobjects1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azurepolicy-aks-1"
  }
}

```

And we have a pattern similar for the Key Vault CSI Secret provider:

```bash

{
  "config": null,
  "enabled": true,
  "identity": {
    "clientId": "00000000-0000-0000-0000-000000000000",
    "objectId": "00000000-0000-0000-0000-000000000000",
    "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-dfitcfr-dev-tfmodule-aksobjects1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azurekeyvaultsecretsprovider-aks-1"
  }
}

```

As for the use of this identity, a `SecretProviderClass` object would be created with the `useVMManagedIdentity` set to `true` and the `userAssignedIdentityID` set to the principal id of the forementioned user assigned identity:
  
```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: $kvname
spec:
  provider: azure
  parameters:
    useVMManagedIdentity: "true"               
    userAssignedIdentityID: $uaiaddon 
    keyvaultName: $kvname
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: $secrename
          objectAlias: $secretname            
          objectType: secret                    
          objectVersion:    
    tenantId: $aadtenantid  


```

There are details on [AKS documentation](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) or on my [blog](https://blog.teknews.cloud/aks/2022/05/19/Hands_on_Vault_Provider_for_Secrets_Store_part2.html) ^^

And that's kind of all for the worker plane.
  
## 4. IAM for workloads hosted on Kubernetes

Now in the case of applications hosted on AKS, one of the issue is to try to benefit from the managed identities inside pods, which by nature, isolate the workloads from the nodes, and at the same time from the Cloud platform.

So is always the possibility of using secret and configmap to set credentials inside containers, but this is not as modern as relying on managed identities.
This lack has been on the mind of AKS teams because we have seen the emergence of not only one but two projects to access Azure services from pods without injecting secret inside the containers that needed it.
Well, rather than two projects, let's say two version of a project, because it has been branded has such:

- Pod Identity v1
- Workload Identity a.k.a Pod Identity v2

Let's start with the v1, even if, as it s name implies, it is kind of deprecated.

### 4.1. Pod Identity

The aim of Pod Identity was to grant the managed identity feature inside Kubernetes.

![Illustration 18](/assets/aksidentity/aksidentity018.png)  

Its architecture, as explained on the [documentation](https://azure.github.io/aad-pod-identity/docs/) relies on CRDs objects, a Kubernetes deployment and a Kubernetes daemonset which manage somehow the interaction with Azure AD for the authentication of the workloads.

![Illustration 19](/assets/aksidentity/aksidentity019.png)  

This project had its AKS add-on, althought it never went out of preview, and it was implemented in a few community solutions such as:

- Velero backup
- Azure Key Vault CSI Secret store
- Application Gateway as Ingress Controller
- Azure Service Operator

And maybe some others that I am not aware of.

If we take the example of the Azure Key Vault CSI Secret store, which is clearly the most known example, we have a yaml manifest for the secret store slightly different than with the managed identity:

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: $kvname
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"               
    userAssignedIdentityID: 00000000-0000-0000-0000-000000000000
    keyvaultName: $kvname
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-csisecret1
          objectAlias: kvs-csisecret1            
          objectType: secret                    
          objectVersion:         
    tenantId: 00000000-0000-0000-0000-000000000000

```

And this time, the principal id would refer to the id of a managed identity that is declared in AKS as a CRDs which act as a kind of digital twin to the managed identity in Azure.

I also happen to have writen an [article](https://blog.teknews.cloud/aks/2021/02/10/podidentityjourney.html) on the topic in the past.

While providing the feature expected, it did also had some limitations and for that, a new version was created and was called...

### 4.2. Workload Identity

So this is the latest project to leverage authentication based on Azure Active Directory in Kubernetes workload.  
While already announced as the replacement for Pod Identity (v1), it is completely different in the architecture and the way it approach its interaction with Azure Active Directory.  
  
![Illustration 20](/assets/aksidentity/aksidentity020.png)  
  
Instead of relying on CRDs and the interception of IMDS traffic, it leverage the [workload identity federation concepts](https://azure.github.io/azure-workload-identity/docs/concepts.html) and in the Kubernetes plane, service account rather than CRDs.

One big advantage versus Pod Identity is the capability to be used on Kubernetes Cluster on any platform.

This excellent [article](https://blog.baeke.info/2022/01/31/kubernetes-workload-identity-with-aks/) from Geert Baeke explains the concepts quite deeply.

In my opinion, there is still one big lack with workload identity whic is the non support for managed identities.
This make the solution very efficient for all Kubernetes cluster except AKS which would probably be better with managed identity support ^^.
Also currently, I've seen only Key Vault CSI Secret store refering to an implementation in tis documentation.  
So let's be patient...

## 5. to summarize

That was a long ride right?

Let's summarize everything before leaving.

There is IAM all over the place in Kubernetes and by extension AKS.
The good news is that AAD and AKS like each other so the integration is quite smooth and powerfull.
A few reco that I developped over the years:

- For the control plane
  - Use Managed AAD for authentication, it's easy to set and allow to leverage AAD security features
  - Disable Local account to avoid uncontrolled privilege escalation
  - Use rather **Azure Kubernetes Service User** and Kubernetes native RBAC for granular access management
  - Leverage user assigned identity for the control plane
- For the worker plane
  - Identify managed identity for Kubelent and add-on and configure activity log alerts
- For workloads in Kubernetes
  - Adopt workload identity for new applications development
  - When possible, leverage either Pod Identity or workload Identity, with a preference for the later

And with that said, see you next time ^^
