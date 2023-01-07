---
layout: post
title:  "Overview of the AzAPI Terraform provider"
date:   2023-01-08 00:00:00 +0200
year: 2023
categories: Azure Terraform AKS
---

Hello!

This post is a long-time coming article on the fantastic AzAPI provider for Terraform.
I wanted to test this tool since so long, but by lack of time...
You know...

We will talk about:

- What is the AzAPI Provider 
- What needs It answers to
- How It works 
- and we'll finish with things to take into consideration in the IaC lifecycle

Let's get started!

## 1. Getting to know the AzAPI provider

Before diving in AzAPI provider, let's do a little history of Terraform.

### 1.1. Going back to Terraform older versions

As a 3rd pary / cloud agnostic product, Terraform can work with many different Clouds, or infrastructure providers or whatever.

Because of this horizontal development, at one point, the Terraform binaries were splitted in 2 : 

- The Terraform binaries
- The providers binaries 

Note the *s* at the end of providers. It does mean that there are more than one. That's what you find on the [Terraform registry providers page](https://registry.terraform.io/browse/providers).
Now obviously, there are dependancies on the up-to-date status of the providers and the availability of the related Cloud provider dev teams.
Knowing that, since the begining of Terraform there were ways to work around the limited available resources.

Globally available for any provider, we can use the null resource which is used by a specific [provider](https://registry.terraform.io/providers/hashicorp/null/3.2.1).
While It does, as the name implies, nothing, It can be coupled with a locally executed cli through another feature of Terraform which is the [local-exec](https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec) provisioner. In Azure scope, we tend to couple this with an az cli or a PowerShell command.

Below is an exemple of this workaround that I used to do before AGIC was in GA and available in the Terraform AzureRM provider: 

```bash

resource "null_resource" "Install_AGIC_Addon" {
  #Use this resource to install AGI on CLI
  provisioner "local-exec" {
    command = "az aks enable-addons -n ${data.azurerm_kubernetes_cluster.AKSCluster.name} -g ${data.azurerm_resource_group.AKSRG.name} -a ingress-appgw --appgw-id ${data.azurerm_application_gateway.AGW.id}"
  }

  depends_on = [

  ]
}

```

Another workaround, this time specific to Azure, is the use of specific resources classified as [Templates](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group_template_deployment).
With time passing, those resources evolved and we have now more than just one:

- `azurerm_management_group_template_deployment`
- `azurerm_resource_deployment_script_azure_cli`
- `azurerm_resource_deployment_script_azure_power_shell`
- `azurerm_resource_group_template_deployment`
- `azurerm_subscription_template_deployment`
- `azurerm_template_deployment`
- `azurerm_tenant_template_deployment`

Lately, I've only used the `azurerm_resource_group_template_deployment` resource.

Now, there is a problem in those workaround, in the fact that well, the resources, created through either az cli in local-exec or in a template resource, are not fully visible in the state.
Which means that we have an impact on the way we manage the resource lifecycle in our Terraform execution cycles.
To be honest though, It did evolved in the good direction, because you can see for exemple that the `azurerm_resource_group_template_deployment` tries to destroy the associated resources when we executed the destroy command.  
  
![illustration1](/assets/azapi/azapi001.png)  
  
But still, we hope for better.

And that's where AzAPI comes into the game

### 1.2. The AzAPI provider

The AzAPI provider is relatively recent in the Azure provider life.
The idea is to provide day 0 resources availability and complete resources available in the AzureRM provider.

But how do we get started?

Well, our first thought should be to check the Terraform registry page and look for the provider documentation.
And we will find there informations related to the provider configuration, as expected, but only a few informations about the available resources:

![illustration2](/assets/azapi/azapi002.png)  

Well, where to look then?

As a matter of facts, it's important to remember that the azapi provider is provided by Microsoft people. So naturally, we should find information on some Microsoft documentation.

If you have ever written an ARM template, you may (should!) have had a look on the [ARM template reference page](https://learn.microsoft.com/en-us/azure/templates/).
In the past, there was only json sample to help us define the template, or the `azurerm_resource_group_template_deployment` that we wanted to deploy.

Then came the ARM evolution a.k.a **bicep**, and the page was augmented with the sample for describing resources in bicep, in a similar way that you can find descrption for resources on the Terraform Azure provider pages.

And last was added the AzAPI resource description. Looking at this ARM Template reference page, we get this:
  
![illustration3](/assets/azapi/azapi003.png)  

![illustration4](/assets/azapi/azapi004.png)  

I guess that's all for the presentation, let's look at how to work with it now.      

## 2. Working with the provider

### 2.1. Configuring the provider

The first thing to do is to configure the provider so that we can use it. The provider block should look like this:

```bash

provider "azapi" {
  subscription_id                          = var.AzureSubscriptionID
  client_id                                = var.AzureClientID
  client_secret                            = var.AzureClientSecret
  tenant_id                                = var.AzureTenantID
    
}

```

Depending on what you are using the provider for, there's chance that the client id and secret are the same as the one for the AzureRM provider.

That's for the authentication. For the provider version we also need the block terraform to look like this:

```bash

terraform {
  required_version = ">~ 1.3.0"

  required_providers {
    azurerm = {...}

    azuread = {...}

    azapi = {
      source = "azure/azapi"
      version = ">~1.1"
    }

  }

  backend "azurerm" {}
}

```

The important part here is the source, for which we need to specify `azure/azapi`
Well, nothing too complex here. Let's have allok to the resources

### 2.2. Understanding the provider's resources

There are only 3 resources, as we have seen earlier: 

- `azapi_resource`  

Typically, a new Azure resource does not exist in the AzureRM provider, and then we could use the `azapi_resource`
For exemple, there is not yet a resource available for Azure Container App, so it would make a good candidate.

- `azapi_resource_action`

This resource can be used to perform action on the resource that we can do in cli or using the API.
For example, starting or stoping an AKS cluster would leverage this `azapi_resource_action`.

- `azapi_update_resource`

This resource is, from my point of view, the most interesting in the Iac workflow point of view.
One of the most frequent case is when there is a new feature on an Azure resource, and the Terraform provider is not yet updated.
This is something that I've met many time when working with AKS for instance.
However, there are limits to the use of this resource.
An update means that the resource can be modified without recreating the resource in the Azure API, which may be different than the Terraform provider.

To identify if the resource's feature can be added, the easiest way is to look the az cli.
Whenever there's an `az some_resource update` with an arg for the feature, then the `azapi_update_resource` should work.
I guess a use case would help

### 2.3. A sample use case in a Terraform configuration lifecycle

To illustrate the use of AzAPI resource, we will take an AKS cluster.
Also we will focus on the `azapi_update_resource`.
Azure Kubernetes Service is typically one of the resource that moves the most frequently.
Even with the huge effort from the AzureRM provider, we are often stuck with feature that are not yet available, hence a need for a workaround.

Let's put the story in place.

### 2.3.1. The sample use case

In our use case, we have an AKS cluster deployed with a module which is versioned and which version is fixed in the module call:

```bash

module "AKS" {
  for_each                              = toset(var.TrainingList)
  #Module Location
  source                                = "github.com/dfrappart/Terra-AZModuletest//Custom_Modules/IaaS_AKS_Cluster?ref=aksv1"

  #Module variable


  AKSLocation                           = azurerm_resource_group.RG[each.value].location
  AKSRGName                             = azurerm_resource_group.RG[each.value].name
  AKSSubnetId                           = azurerm_subnet.subnet[each.value].id
  AKSNetworkPlugin                      = "kubenet"
  AKSNetPolProvider                     = "calico"
  AKSClusSuffix                         = substr(replace(replace(each.value,".",""),"-",""),0,12)
  AKSIdentityType                       = "UserAssigned"
  UAIIds                                = [module.UAI_AKS[each.value].FullUAIOutput.id]
  PublicSSHKey                          = tls_private_key.akssshkey[each.value].public_key_openssh  
  AKSClusterAdminsIds                   = [data.azuread_group.aksadmin.object_id]
  TaintCriticalAddonsEnabled            = false
  LawLogId                              = azurerm_log_analytics_workspace.logaks.id
  EnableHostEncryption                  = true
  LawDefenderId                         = data.azurerm_log_analytics_workspace.defenderlaw.id

}

```

For this cluster, I would like to enable the Blob storage CSI storage classe.
I will take the hypothesis that it is not available in the AzureRM provider (which is not true, but play along).
Also, I may not be the one responsible for the module, so, well, I could always need to modify the cluster **before** the team had the time to add the feature.

Let's go then.

### 2.3.2. Defining the update resource

I know that I can add this feature on an existing cluster from the [Azure documentation](https://learn.microsoft.com/en-us/azure/aks/azure-blob-csi?tabs=NFS), whith the following command

```bash

az aks update --enable-blob-driver -n myAKSCluster -g myResourceGroup

```

So we can start by adding the `azapi_update_resource` :

```bash

resource "azapi_update_resource" "aksEnableBlobCSI" {
  for_each                              = toset(var.TrainingList) 
  
  type                                  = "Microsoft.ContainerService/managedClusters@2022-09-02-preview"
  resource_id                           = module.AKS[each.value].KubeId

  body                                  = jsonencode({
                                                properties = {
                                                  
                                              }
                                            })

}

```

Note the API at the end of the resource type. That's something that we will find directly on the resource definition in the [documentation](https://learn.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters?pivots=deployment-language-terraform).

Searching for the `storage` string, we find the part that we are looking for:

```bash


      storageProfile = {
        blobCSIDriver = {
          enabled = bool
        }
        diskCSIDriver = {
          enabled = bool
          version = "string"
        }
        fileCSIDriver = {
          enabled = bool
        }
        snapshotController = {
          enabled = bool
        }
      }

```

Once we know that we are looking for the `storageProfile` property, we can find for [details](https://learn.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters?pivots=deployment-language-terraform#managedclusterstorageprofile-2): 

**ManagedClusterStorageProfile**

| Name | Description | Value |
|-|-|-|
| blobCSIDriver | AzureBlob CSI Driver settings for the storage profile. | ManagedClusterStorageProfileBlobCSIDriver |
| diskCSIDriver | AzureDisk CSI Driver settings for the storage profile. | ManagedClusterStorageProfileDiskCSIDriver |
| fileCSIDriver | AzureFile CSI Driver settings for the storage profile. | ManagedClusterStorageProfileFileCSIDriver |
| snapshotController | Snapshot Controller settings for the storage profile. | ManagedClusterStorageProfileSnapshotController |

**ManagedClusterStorageProfileBlobCSIDriver**

| Name | Description | Value |
|-|-|-|
| enabled | Whether to enable AzureBlob CSI Driver. The default value is false. | bool |

Also, depending on the IDE add-ons, there's a chance that we can get help from intellisense:

![illustration5](/assets/azapi/azapi005.png)  

![illustration6](/assets/azapi/azapi006.png)  

![illustration7](/assets/azapi/azapi007.png)  

![illustration8](/assets/azapi/azapi008.png)  

Which gives us the configuration of the resource as follow:

```bash

resource "azapi_update_resource" "aksEnableBlobCSI" {
  for_each                              = toset(var.TrainingList) 
  
  type                                  = "Microsoft.ContainerService/managedClusters@2022-09-02-preview"
  resource_id                           = module.AKS[each.value].KubeId

  body                                  = jsonencode({
                                                properties = {
                                                  storageProfile = {
                                                    blobCSIDriver = {
                                                      enabled = true
                                                    }
                                                  }
                                              }
                                            })

}

```

Remember that this is a Terraform config, which means, even if it's not the case here, that we could use interpolation to refer to other resources attributes.

### 2.3.3. Managing the update resource in the Terraform lifecycle

With the definition written, the plan and apply are ready to go:

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  + create
  ~ update in-place

Terraform will perform the following actions:

  # azapi_update_resource.aksEnableBlobCSI["david.frappart"] will be created
  + resource "azapi_update_resource" "aksEnableBlobCSI" {
      + body                    = jsonencode(
            {
              + properties = {
                  + storageProfile = {
                      + blobCSIDriver = {
                          + enabled = true
                        }
                    }
                }
            }
        )
      + id                      = (known after apply)
      + ignore_casing           = false
      + ignore_missing_property = true
      + name                    = (known after apply)
      + output                  = (known after apply)
      + parent_id               = (known after apply)
      + resource_id             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.ContainerService/managedClusters/aks-davidfrappar"
      + type                    = "Microsoft.ContainerService/managedClusters@2022-09-02-preview"
    }


Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  ~ AKS = (sensitive value)

────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly
these actions if you run "terraform apply" now.

```

The terraform apply should modify the parameter as desired.

Just to be sure, we could check the state of our AKS cluster:

```bash

yumemaru@azure:~/azapilab$ az aks list | jq '.[].name, .[].resourceGroup, .[].powerState, .[].storageProfile'
"aks-davidfrappar"
"rsg-david.frappart"
{
  "code": "Running"
}
{
  "blobCsiDriver": null,
  "diskCsiDriver": {
    "enabled": true
  },
  "fileCsiDriver": {
    "enabled": true
  },
  "snapshotController": {
    "enabled": true
  }
}

```

There is currently nothing set for the `blobCsiDriver`.

But after the apply is completed, it's changed, which is what we wanted:

```bash

azapi_update_resource.aksEnableBlobCSI["david.frappart"]: Creation complete after 3m4s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.ContainerService/managedClusters/aks-davidfrappar]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

AKS = <sensitive>
aksssh = <sensitive>
kmskey = <sensitive>
kmskv = <sensitive>
yumemaru@azure:~/azapilab$ az aks show -n aks-davidfrappar -g rsg-david.frappart | jq .storageProfile.blobCsiDriver
{
  "enabled": true
}

```

That was the easy part, adding parameter to existing resources. It's working as expected.
Now the next step, the module as evolved, because the parameter is now configurable in the provider.
In the module that we use, there is an addon on the readme:

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_IsBlobDriverEnabled"></a> [IsBlobDriverEnabled](#input\_IsBlobDriverEnabled) | Is the Blob CSI driver enabled? Defaults to false. | `bool` | `true` | no |

Nice!

But how do we proceed?

Well, one looking for answers should always start by the documentation.
We can read in the Azure documentation that there is a tool to help the migration from AzAPI resources to AzureRM resources called [AzAPI2AzureRM](https://github.com/Azure/azapi2azurerm/releases).
But this time, by documentation, I meant the Terraform registry website:

![illustration8](/assets/azapi/azapi008.png)  

Which means, that the property modified by the update will not be revert back if we delete the resource.

We can start by commenting the `azapi_update_resource`:

```bash

module "AKS" {
  for_each                              = toset(var.TrainingList)
  #Module Location, with ref to last commit 
  source                                = "github.com/dfrappart/Terra-AZModuletest//Custom_Modules/IaaS_AKS_Cluster?ref=aksv1"

  #Module variable

===========================truncated================================



}

# Commented because included in the new module version
#
#resource "azapi_update_resource" "aksEnableBlobCSI" {
#  for_each                              = toset(var.TrainingList) 
#  
#  type                                  = "Microsoft.ContainerService/managedClusters@2022-09-02-preview"
#  resource_id                           = module.AKS[each.value].KubeId
#
#  body                                  = jsonencode({
#                                                properties = {
#                                                  storageProfile = {
#                                                    blobCSIDriver = {
#                                                      enabled = true
#                                                    }
#                                                  }
#                                              }
#                                            })
#
#}
#

```

We can see the destroy of the resource in the plan:

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  ~ update in-place
  - destroy

Terraform will perform the following actions:

  # azapi_update_resource.aksEnableBlobCSI["david.frappart"] will be destroyed
  # (because azapi_update_resource.aksEnableBlobCSI is not in configuration)
  - resource "azapi_update_resource" "aksEnableBlobCSI" {
      - body                    = jsonencode(
            {
              - properties = {
                  - storageProfile = {
                      - blobCSIDriver = {
                          - enabled = true
                        }
                    }
                }
            }
        ) -> null
      - id                      = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.ContainerService/managedClusters/aks-davidfrappar" -> null
      - ignore_casing           = false -> null
      - ignore_missing_property = true -> null
      - name                    = "aks-davidfrappar" -> null
      - output                  = jsonencode({}) -> null
      - parent_id               = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart" -> null
      - resource_id             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.ContainerService/managedClusters/aks-davidfrappar" -> null
      - type                    = "Microsoft.ContainerService/managedClusters@2022-09-02-preview" -> null
    }


Plan: 0 to add, 0 to change, 1 to destroy.

```

After the apply, checking the AKS resource will reveal no change on the `storageProfile`

```bash

yumemaru@azure:~/azapilab$ az aks show -n aks-davidfrappar -g rsg-david.frappart | jq .storageProfile.blobCsiDriver
{
  "enabled": true
}


```

We can now change the version of the module to the appropriate commit:

```bash

module "AKS" {
  for_each                              = toset(var.TrainingList)
  #Module Location, with ref to last commit 
  source                                = "github.com/dfrappart/Terra-AZModuletest//Custom_Modules/IaaS_AKS_Cluster?ref=aksv1"

  #Module variable

===========================truncated================================
  IsBlobDriverEnabled                   = true


}

```

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # module.AKS["david.frappart"].azurerm_kubernetes_cluster.AKSRBACCNI will be updated in-place
  ~ resource "azurerm_kubernetes_cluster" "AKSRBACCNI" {
        id                                  = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.ContainerService/managedClusters/aks-davidfrappar"
        name                                = "aks-davidfrappar"
        tags                                = {
            "Company"      = "dfitc"
            "CostCenter"   = "lab"
            "Country"      = "fr"
            "Environment"  = "dev"
            "ManagedBy"    = "Terraform"
            "Project"      = "tfmodule"
            "ResourceOwne" = "That could be me"
        }
        # (33 unchanged attributes hidden)

      + storage_profile {
          + blob_driver_enabled         = true
          + disk_driver_enabled         = true
          + disk_driver_version         = "v1"
          + file_driver_enabled         = true
          + snapshot_controller_enabled = true
        }

        # (9 unchanged blocks hidden)
    }



Plan: 0 to add, 1 to change, 0 to destroy.

```

After the apply, well, nothing changed on our parameter:

```bash

yumemaru@azure:~/azapilab$ az aks show -n aks-davidfrappar -g rsg-david.frappart | jq .storageProfile
{
  "blobCsiDriver": {
    "enabled": true
  },
  "diskCsiDriver": {
    "enabled": true,
    "version": "v1"
  },
  "fileCsiDriver": {
    "enabled": true
  },
  "snapshotController": {
    "enabled": true
  }
}


```

But It is now included in the AzureRM resource.

## 3. Before leaving

So that's it. 
We saw the way we can update resource with the `azapi_update_resource`.
We also saw that the resource is though to be disposable, so that It can easily be migrated manually.
We did not check the automated tool for migration however. But since It was a hands on kind of article, I hope you'll forvive me ^^
I have some other use case for AKS and the AzAPI provider, so I will probably use It again soon.
But for now, that's all!
