---
layout: post
title:  "Hands on Azure Key Vault Provider for Secrets Store CSI Driver Part 1"
date:   2022-05-18 11:00:00 +0200
year: 2022
categories: AKS Security
---

Hi!

This article is about the Azure Key Vault Provider for Secret Store CSI Driver.
Behind the long name, we get a way to consume secrets from a Key Vault inside of Kubernetes, and in our case, Azure Kubernetes Service.
Ok, let's look at the agenda for this part 1.
  
## Table of content

1. About Secret Store CSI Driver
2. Azure Key Vault Provider for Secrets Store CSI Driver
3. How to install, the easy way
4. How to install with Helm
5. Summary before next part
  
## 1. About Secret Store CSI Driver  
  
Let's start from the beginning.

First, there is the [Kubernetes Container Storage Interface](https://kubernetes-csi.github.io/docs/), which we will refer to as the CSI.
As for other *CxI*, this is a standardized interface for storage in Kubernetes.
First we had the [Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/) or *CRI*, which provided a standardized way for different container runtime to be used by kubelet.

With the CSI we get something similar for storage: a standardized interface for storage provider to develop from.
There are gains from this standardized approcah, obviously, but that's not topic of this article, so let's move on.

In the CSI family, we find the [Secret Store CSI driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction.html).

As the name hints, it's a way to interface secrets store solutions as volume in a Kubernetes environment.

Which is fine because it's a way to externalise secrets and thus secrets management from Kubernetes.

It has the following features, which are stables:  

- Multiple external secrets store providers
- Pod portability with the SecretProviderClass CustomResourceDefinition
- Mounts secrets/keys/certs to pod using a CSI Inline volume
- Mount multiple secrets store objects as a single volume
- Linux and Windows containers

And those fatures, which are currently alpha:

- Auto rotation of mounted contents and synced Kubernetes secret
- Sync with Kubernetes Secrets

In the main lines, it works like this:  
  
On pod start and restart, the CSI secret store driver:

- Communicate with the provider using gRPC
- retrieve the secret content from the external secret store

Once a Secret is mounted as a volume, its data is mounted in the file system of the container.

![Illustration 1](/assets/csisecret/csisecret001.png)

It works with a daemonset that facilitates communication with every instance of Kubelet.

Each driver pod has the following containers:  

- node-driver-registrar: Responsible for registering the CSI driver with Kubelet
- secrets-store: Implements the CSI Node service gRPC services described in the CSI specification. It’s responsible for mount/unmount the volumes during pod creation/deletion.
- liveness-probe: Responsible for monitoring the health of the CSI driver and reports to Kubernetes. This enables Kubernetes to automatically detect issues with the driver and restart the pod to try and fix the issue.

Also, it comes with some CRDs:

| CRD name | CRD function |
|----------|--------------|
| SecretProviderClass | A namespaced resource in Secrets Store CSI Driver that is used to provide driver configurations and provider-specific parameters to the CSI driver. |
| SecretProviderClassPodStatus| A namespaced resource in Secrets Store CSI Driver that is created by the CSI driver to track the binding between a pod and SecretProviderClass. |

Our main focus will be the CRD SecretProviderClass, and specifically how it is configured for an Azure Key Vault as a provider.

## 2. Azure Key Vault Provider for Secrets Store CSI Driver
  
Now that we've discussed Secret Store CSI driver in general, let's have a look at the Secret Store CSI driver for Key Vault.
Azure Key Vault is currently one of the supported secret provider, and it relies, as the name implies, on an Azure Key Vault under the hood.

For the curious minds, the supported providers are listed below. Note that the features may vary between the different providers.

- AWS Provider
- Azure Provider
- GCP Provider
- Vault Provider

Focusing on the Azure provider:

- It relies on an Azure Key Vault
- Kubernetes 1.18 is recommanded, but that's not an issue, since we are currently in the 1.23.x and well, 1.18 is certainly not a supported version anymore ^^

Now regarding the available features, it can do pretty much all of the Secret store CSI driver featrues that we already discussed, so that means:

- Mounts secrets/keys/certs to pod using a CSI volume
- Mount multiple secrets store objects as a single volume
- Support CSI Inline volumes
- Supports pod portability with the SecretProviderClass CRD
- Supports Linux & Windows containers
- Sync with Kubernetes Secrets (Secrets Store CSI Driver v0.0.10+)
- Supports auto rotation of mounted contents and synced Kubernetes secrets (Secrets Store CSI Driver v0.0.15+)

Before diving in the details of the secret store kubernetes object, let's have a look at what it brings:  

![Illustration 2](/assets/csisecret/csisecret002.png)  
  
On the left, we have the Azure control plane, on the right the Kubernetes control plane.
Using the driver, we are using inside the Kubernetes control plane an object which makes a kind of digital twin of the Azure Key Vault which lives in the Azure control plane.

What is nice in this case is that the secret are managed outside of Kubernetes, so not necessarily stored in etcd. I'm saying not necessarily, because, well, in some case it will still be the case, but more on that later.

Now, since the picture is introducing the structure of the object, let's focus on this.
A secret store using the Azure provider is looking like this:  

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: ${AzureKVName}
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"               
    userAssignedIdentityID: ${UAIClientId}
    keyvaultName: ${AzureKVName}
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: ${SecretName}
          objectAlias: ${SecretName}            
          objectType: secret                    
          objectVersion: ${SecretVersion}       
    tenantId: ${TenantId}} 


```

For the purpose of being as thorough as possible, we could put all the available properties in a table. However, since it is available on the [documentation page](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/getting-started/usage/), let's not do that ^^ but only focus on the main properties:  

| Property name | Required | Description | Default value |
|-|-|-|-|
| `apiVersion` | Yes | Define the api version used. should be `secrets-store.csi.x-k8s.io/v1` | `secrets-store.csi.x-k8s.io/v1` |
| `kind` | Yes | Define the kind of Kubernetes object, which is here the CRD `SecretProviderClass` | `SecretProviderClass` |
| `provider` | Yes | The name of the provider, in our case, it will be obviously azure | "" |
| `usePodIdentity` | No | Define if we use Pod Identity for accessing the key vault instance behind the secret store provider class object | "false" |
| `useVMManagedIdentity` | No | Define if we use an Azure managed identity for accessing the key vault instance behind the secret store provider class object | "false" |
| `userAssignedIdentityId` | No | Define the client id of the managed identity if useVMManagedIdentity is set to true | "" |
| `clientID` |  |  |  |
| `keyvaultname` | Yes | Define the name of the Key Vault instance used to bakc this secrets store provider | "" |
| `cloudName` | No | Define the name og the Azure Cloud (`AzurePublicCloud`, `AzureUSGovernement`, `AzureChinaCloud`, `AzureGermanCloud`, `AzureStackCloud`)  | "" |
| `objects` | Yes | An array containing the objects from the Key Vault used in the secret store | "" |
| `objectName` | Yes | Name of a Key Vault object| "" |
| `objectType` | Yes | Type of Key Vault object: `secret`, `key` or `cert` |  |
| `objectVersion` | No | Version of a Key Vault object. If not provided, the last version is used | "" |
| `tenantID` | Yes | The Azure Active Diretory Tenant Id to which the Key Vault is attached to |  |

Now, what's important to look at, first, is how we can authenticate on the Key Vault from Kubernetes.
We have 3 ways, which are specified with the properties :  

- With a service principal, which means using an Azure AD application registration. This way is still currently the only way if we use the Azure provider outside of Azure.
- With an Azure managed identity. In this case, we leverage, in an Azure environment, a managed identity associated with the scale sets corresponding to the node pools
- With Pod Identity, which is using also Azure managed identities, but bound on pods rather than nodes

Ok, there's also a fourth way, still in preview, which is by using Azure Workload Identity.

In the future, this will be the v2 of Pod Identity, but currently, it's not providing as much integration as Pod Identity v1. We won't look at this in this article because, well it probably should be addressed in a dedicated serie ^^

## 3. How to install, the easy way  
  
As the title hints, there is mpre than one way to install this Key Vault CSI driver. In this part we will look at the easiest way, which is through the AKS add-on.

### 3.1. Installing the add-on through az cli

Microsoft works a lot on the AKS product so that is as easy as possible to deploy. The same statement applies when it comes to add-on.

Installing the CSI driver is as simple as the following command:  
  
```bash

az aks enable-addons -n ${AKSName} -g ${module.ResourceGroup.RGName} -a azure-keyvault-secrets-provider --enable-secret-rotation


```
  
In this case we only need to know the AKS cluster name and the resource group it is located in, which we can get onthe portal or through the cli with `az aks list` command.
Note that at this point we do not need the resource Id of an Azure Key Vault.
And thats because, this add-on only install the requirement so that we can create secret store in Kubernetes afterward.

Meaning, afterward, there is still work to do, but in the Kubernetes control plane. We'll have a look at what happened in this control plane later. For now, let's look at just another way to install this add-on.

### 3.2. Installing the add-on with terraform

Yup, that's also possible, because this add-on is in GA, and also because there is a lot of effort on the provider team when it comes to AKS.

As a matter of fact, there is a statement on the terraform rgistry page for AKS object, indicating that one should use the last version of the provider as much as possible to ensure the latest feature are available.

The being said, how to install?

Well again this is a very simple snippet of code, as follow:  
  
```hcl

key_vault_secrets_provider {

  secret_rotation_enabled               = var.CSIKVSecretRotationEnabled
  secret_rotation_interval              = var.CSIKVSecretRotationInterval

  }

```

Note that there were some change in the provider version 3.x.x, and the add-on objects moved from being in a block called add-on to a serie of different blocks directly in the resource.

It was also stated before the new release of the provider ^^.

For more flexibility on the activation of this add-on, we can use a dynamic block which is activated following the value of a boolean:  

```hcl

  dynamic "key_vault_secrets_provider" {

    for_each = var.IsCSIKVAddonEnabled ? ["fake"] : []

    content {

      secret_rotation_enabled               = var.CSIKVSecretRotationEnabled
      secret_rotation_interval              = var.CSIKVSecretRotationInterval

    }

  }

```

### 3.3. Installing through the portal

Ok, there's a 3rd possibility to install, which is through the portal. But eh, where's the fun in that?

It's as simple as checking the option:  

![Illustration 3](/assets/csisecret/csisecret003.png)
  
### 3.4. Checking the install

Now that the installation is done, let's have a look at what we have.

If you have a brand new cluster, looking for new namespace related directly to the CSI Key Vault Secret store will give you nothing.

That's because everything gets installed in the kube-system when using the add-on.  

As we can see below, we have the daemonset for the CSI Secret store driver, and additional pods for the Key Vault provider:

```bash

kubectl get ds -n kube-system

NAME                                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
aks-secrets-store-csi-driver               5         5         5       5            5           <none>          4m6s
aks-secrets-store-csi-driver-windows       0         0         0       0            0           <none>          4m6s
aks-secrets-store-provider-azure           5         5         5       5            5           <none>          4m6s
aks-secrets-store-provider-azure-windows   0         0         0       0            0           <none>          4m6s
===================================truncated===================================

```

Note that we have objects for windows workload, but the desired state being 0, we have 0 pod.

While it is possible to enable secret rotation with `--enable-secret-rotation --rotation-poll-interval 5m`, I did not find a way to modify the windows related pod desired state. Nut it might be because i do not have Windows based node pools.

Now what about the authentication mechanism?

We did not install Pod Identity nor Workload Identity, and we did not had an Azure AD App registration created behind the scene. However, if we look in the AKS managed resource group, we can see that we have a new User Assigned Identity with a name that hint something:  
  
![Illustration 4](/assets/csisecret/csisecret004.png)
  
So with the add-on installed, we have a brand new managed identity associated to the scaleset(s) of our cluster.

In this case then we would set the argument `useVMManagedIdentity` in our SecretClassProvider to **true** and specify the client id of the managed identity with the argument `userAssignedIdentityId`.
We will look at how to use this managed identity within the Kubernetes environment in part 2 of this article.

Note also that if we want to use Pod identity,we need to have either the feature installed as an add-on, or through a helm chart.

And in this case, instead of the argument `useVMManagedIdentity`, we would set the argument `usePodIdentity` to **true**

Now if we want to have the self managed installation, we can use helm.
  
## 4. How to install with Helm

Until now, we had a look at the concepts onf CSI secret store and how to install as an add-on.
But chances are that you may want to do it yourself, or even that you don't necessariliry run Kubernetes as AKS.
As such, the manual install through Helm makes more sense.

Also you might be a control freak as I am and just want to get the mechanism under the hood. Let's just do that shall we ?

So we mentionned that there were two parts to this driver and thus two helm chart:  

- The [secrets store CSI driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver/tree/main/charts/secrets-store-csi-driver) which is the standardized interface
- The [Azure Key Vault provider for Secrets store CSI driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure/tree/master/charts/csi-secrets-store-provider-azure) which is the junction with Azure Key Vault instances

Note that installing the secrets store CSI driver is a prerequisite for using any of the secret store CSI drivers that are available.

Note also that to make things simpler, there is an option to install the secrets store CSI driver automatically when installing the chart for the Key Vault part.

There is no need to copy paste the documentation so let's just focus on the parameters that are of interest ofr us here:  

| Helm chart parameter | Description | Configured value |
|-|-|-|
| `secrets-store-csi-driver.install` | Value to define if the CSI driver is installed ad part of this helm release. | `true` |
| `secrets-store-csi-driver.enableSecretRotation` | Value to define if the secret rotation from the key vault is activated. `true` means that the new value of the secret in the key vault will be reflected | `true` |
| `secrets-store-csi-driver.rotationPollInterval` | Value to define at which insterval the driver poll the Key Vault to get the secret value | `1m` |
| `syncSecret.enabled` | Allows to sync with a Kubernetes secret | `true` |

Another interesting parameter could be the `linux.tolerations`. In an AKS environment, if we use the one taint `CriticalAddonsOnly=true:NoSchedule` available for the default node pool, we could use this parameter.
Also, if you do check the helm chart documentation, you will notice Windows related parameters. By default, the secret store is not enabled for Windows nodes.

For those who wonder, I use the terraform helm provider just like that:

```bash

######################################################################
# installing csi secret store key vault provider from helm

resource "helm_release" "csisecretstorekvprovider" {
  name                                = "csisecretstorekvprovider"
  repository                          = "https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts"
  chart                               = "csi-secrets-store-provider-azure"
  version                             = var.CSISecretStoreKvPRoviderChartVer
  namespace                           = "csisecretstorekvprovider"
  create_namespace                    = true

  dynamic "set" {
    for_each                          = var.HelmCSISecretStoreKVProviderParam
    iterator                          = each
    content {
      name                            = each.value.ParamName
      value                           = each.value.ParamValue
    }

  }


  depends_on = [
    #helm_release.csisecretstore
  ]

}

```

Withe following variables to feed my chart:

```bash


variable "CSISecretStoreKvPRoviderChartVer" {
  type                          = string
  description                   = "The version of the chart"
}

variable "HelmCSISecretStoreKVProviderParam" {
  type                  = map
  description            = "A map used to feed the dynamic blocks of the pod identity helm chart"
  default                = {

      "set1" = {
        ParamName             = "secrets-store-csi-driver.install"
        ParamValue            = "true"

    }
      "set2" = {
        ParamName             = "secrets-store-csi-driver.enableSecretRotation"
        ParamValue            = "true"

    }

      "set3" = {
        ParamName             = "secrets-store-csi-driver.rotationPollInterval"
        ParamValue            = "1m"

    }

      "set4" = {
        ParamName             = "syncSecret.enabled"
        ParamValue            = "true"

    }
  }

}


```

Again, with the helm chart, there is only the resources for the driver. No secret provider class yet. Since we requested the installation to use a namespace called `csisecretstorekvprovider`, that's what we should have:

```bash

k get all -n csisecretstorekvprovider

NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/csisecretstorekvprovider-csi-secrets-store-provider-azure-7qxfz   1/1     Running   0          67m
pod/csisecretstorekvprovider-csi-secrets-store-provider-azure-8c9cd   1/1     Running   0          67m
pod/csisecretstorekvprovider-csi-secrets-store-provider-azure-x7f6t   1/1     Running   0          67m
pod/secrets-store-csi-driver-7s2xb                                    3/3     Running   0          64m
pod/secrets-store-csi-driver-lxcbz                                    3/3     Running   0          64m
pod/secrets-store-csi-driver-xsbt8                                    3/3     Running   0          64m

NAME                                                                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/csisecretstorekvprovider-csi-secrets-store-provider-azure   3         3         3       3            3           kubernetes.io/os=linux   13h
daemonset.apps/secrets-store-csi-driver                                    3         3         2       3            2           kubernetes.io/os=linux   13h

```

Because we included the installation of both the secret store csi driveer and the key vault provider, we have everything under the same namespace.

Also, this time, this is a manual install, so no Managed Identity under the hood, and neither any reference to the add-on on the cluster.

```bash

{
  "azurepolicy": {
    "config": null,
    "enabled": true,
    "identity": {...}
  },
  "omsagent": {
    "config": {...},
    "enabled": true,
    "identity": {...}
  }
}

```

And the portal is only displaying the identity for those addons, plus the one for kubelet, but none for the CSI Secret Store extension:

![Illustration 5](/assets/csisecret/csisecret005.png)

And that's about it for the installation.

## 5. Summary before next part

In this part we tried to have a complete view of the installation and how the Key Vault secret store is working.
We did not go into details on the usage but the follow up is coming soon.

Until next time then!
