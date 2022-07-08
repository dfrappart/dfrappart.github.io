---
layout: post
title:  "AKS private cluster through the terraform lens"
date:   2022-07-08 10:00:00 +0200
categories: AKS
---

Hi everyboy!

Today's topic is AKS in private mode.

In this article, we will take an interest to this deployment model, but rather than through a classic architecture concepts, we'll take it from the Infrastructure as Code point of view.

At the end of those lines, we will have gone through the differents requirements for having a private AKS cluster.

## Table of content

1. Kubernetes Seurity principles in 2 minutes
2. Translating Kubernetes Security in Azure
3. From the Infrastructure as code point of view
4. Before leaving


## 1. Kubernetes Seurity principles in 2 minutes

Let's start by a very high level view of kubernetes architecture. Just to be on the same page.

![Illustration 1](/assets/pvaks/pvaks001.png)

If we summarize, we have to consider the control plane, which as the name implies, is reponsible for controlling whatever happens in the worker plane, where live the applications.

It is really very simlified but that's about it.

Now let's take from [kubernetes documentation](https://kubernetes.io/docs/concepts/security/overview/
) a description of what need to be secured for a kubernetes based architecture:  

Security topics for kubernetes are divided in 4 categories, called the 4 C:  

- The Cloud
- The cluster
- The container
- The code

![Illustration 2](/assets/pvaks/pvaks002.png)  
  
TYpically, we could say that the first 2 Cs are more infrastructure oriented. Considering actual topics for those we get the following:  

| Cloud area security topic |
|-|
| Network access to API Server |
| Network accerss to nodes |
| Kubernetes access to Cloud provider API|
| Access to ETCD |
| ETCD Encryption |

| Cluster area security topic |
|-|
| RBAC Authorization |
| Authentication |
| Application secrets management |
| Pod Security Standards |
| Network Policies |
| TLS for Kubernetes Ingress |

Which does make a lot of topics. But today, we are talking about AKS private cluster which is the Azure answer to the first topic **Network access to API Server**.

That's right, only this tiny little topic today.

Ok, now that we have define the foundations, let's focus on what we have considering Azure and AKS.

## 2. Translating K8S Security in Azure

Starting from our last kubernetes representation, we get something like below in Azure:  

![Illustration 3](/assets/pvaks/pvaks003.png)  

As we can see on this (yet again) very simplified schema, AKS is PaaS. As a majority of PaaS, it is by design exposed in a public DNS namespace. In AKS case `<something>.<azureregion>.azmk8s.io`.

Now the main discussion point is this *public* access. For the fluidity of the discussion heren, let's just say that we want a way to avoid public exposition of the control plane, and specifically the API server.
We have a few options. The simplest being to use an accept list on the API server. Thus, only known and defined public IP can access to the API server, which is the part that we access on the control plane.

But in some case, a simple accept list is not acceptable because, well, it does not follow a regulation.

So this time, if we want to still use a PaaS but in a private network **only**, we will have to rely on [Azure Private Endpoints](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview). The private endpoint is Microsoft answer for the need of private PaaS and similar to a kind of NAT.

Technically, a Network Interface Card is connected in a Virtual Network and NATed to the PaaS instance through [Private Link](https://docs.microsoft.com/en-us/azure/private-link/private-link-service-overview). This NIC thus gets a private IP from the VNet range. To be able to resolve the PaaS instance name, there is  the need for an Azure Private DNS zone which will register the PaaS instance with its private IP. From an Azure Network standpoint, all virtual network that need to resolve the DNS record for the PaaS instance need to be linked to the Azure Private DNS zone containing this record.

Also, in the meantime, the public fqdn is disabled and makes it impossible to reach the PaaS instance from anywhere on the Internet. We have to be **inside** the private network to reach the PaaS instance.

Just to be clear, Private Endpoint is not limited to AKS, even if we will only look at how to put tha tin place in the following part.

So to summarize all that, we get a schema looking like that:  

![Illustration 4](/assets/pvaks/pvaks004.png)  

And that's how, on the principles, we render our API server fully private.

However, there are a few catches:

- There is a limitation with with NSG filtering private endpoint. Indeed, this specific NIC NATing the PaaS instance is not filterable by an NSG rules. So, inside the Subnet, no filtering to this NIC. It's on the point of changing though because the filtering is currently in preview and start to be visible from the portal.

- A similar limitation regarding routing with UDR is also impacting the private endpoint. If you ever thought of working around the previous limitation by routing the traffic to an NVA doing the filtering in its place, plan accordingly.

- Because it's changing the network path to the PaaS instance, it also change the public FQDN, managed by Microsoft, to a Private FQDN. In our AKS case, we move to a sub DNS zone prefixed with `privatelink` such as `<something>.privatelink.<azureregion>.azmk8s.io`.

![Illustration 5](/assets/pvaks/pvaks005.png)  

Apart from that it seems good. Now how do we try that? That's coming in the next part

## 3. From the infrastructure as code point of view

### 3.1. What we'll need

From the previous part, we know that first we will need those objects:  

- A virtual Network with at least 1 subnet
- An AKS cluster
- A private DNS zone
- Additional USer assigned identities that we did not discussed yet ^^

Obviously, we want to rely on [terraform documentation about AKS](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster)

Specifically we will look at those arguments in the kubernetes cluster resource:  

![Illustration 6](/assets/pvaks/pvaks006.png)  
  
![Illustration 7](/assets/pvaks/pvaks007.png)  

### 3.2. Creating the basic resources

Let's start by the basics resources then. We could relies on some module, but the purpose being solely to get uder the hood of the private AKS, we'll keep it simple.

So firt, a [resource group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group):

```bash

# Resource Group

resource "azurerm_resource_group" "DemoRG" {
    name                        = "DemoRG"
    location                    = "westus"
  
}

```

Then we need a [Virtual Network](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_network) and a [subnet](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet):  

```bash

resource "azurerm_virtual_network" "DemoVNet" {
    count                       = 3
    name                        = "DemoVNet${count.index+1}"
    address_space               = ["172.24.0.0/24"]
    location                    = azurerm_resource_group.DemoRG.location
    resource_group_name         = azurerm_resource_group.DemoRG.name
  
}

resource "azurerm_subnet" "DemoSubnet" {
    count                       = 3
    name                        = "DemoSubnet${count.index+1}"
    address_prefixes            = ["172.24.0.0/26"]
    resource_group_name         = azurerm_resource_group.DemoRG.name
    virtual_network_name        = azurerm_virtual_network.DemoVNet[count.index].name
  
}

```

Nothing very difficult here. Notice however the `count` meta-argument. That's because we have more than one scenario so we want to have a look at all of those.
Ok that's about it for the basics resources so let's play with the AKS pal. Let's run a plan and a apply to have it built and forget it in the following steps: 

```bash

terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_resource_group.DemoRG will be created
  + resource "azurerm_resource_group" "DemoRG" {
      + id       = (known after apply)
      + location = "westus"
      + name     = "DemoRG"
    }

  # azurerm_subnet.DemoSubnet[0] will be created
  + resource "azurerm_subnet" "DemoSubnet" {
      + address_prefixes                               = [
          + "172.24.0.0/26",
        ]
      + enforce_private_link_endpoint_network_policies = false
      + enforce_private_link_service_network_policies  = false
      + id                                             = (known after apply)
      + name                                           = "DemoSubnet1"
      + resource_group_name                            = "DemoRG"
      + virtual_network_name                           = "DemoVNet1"
    }

  # azurerm_subnet.DemoSubnet[1] will be created
  + resource "azurerm_subnet" "DemoSubnet" {
      + address_prefixes                               = [
          + "172.24.0.0/26",
        ]
      + enforce_private_link_endpoint_network_policies = false
      + enforce_private_link_service_network_policies  = false
      + id                                             = (known after apply)
      + name                                           = "DemoSubnet2"
      + resource_group_name                            = "DemoRG"
      + virtual_network_name                           = "DemoVNet2"
    }

  # azurerm_subnet.DemoSubnet[2] will be created
  + resource "azurerm_subnet" "DemoSubnet" {
      + address_prefixes                               = [
          + "172.24.0.0/26",
        ]
      + enforce_private_link_endpoint_network_policies = false
      + enforce_private_link_service_network_policies  = false
      + id                                             = (known after apply)
      + name                                           = "DemoSubnet3"
      + resource_group_name                            = "DemoRG"
      + virtual_network_name                           = "DemoVNet3"
    }

  # azurerm_virtual_network.DemoVNet[0] will be created
  + resource "azurerm_virtual_network" "DemoVNet" {
      + address_space       = [
          + "172.24.0.0/24",
        ]
      + dns_servers         = (known after apply)
      + guid                = (known after apply)
      + id                  = (known after apply)
      + location            = "westus"
      + name                = "DemoVNet1"
      + resource_group_name = "DemoRG"
      + subnet              = (known after apply)
    }

  # azurerm_virtual_network.DemoVNet[1] will be created
  + resource "azurerm_virtual_network" "DemoVNet" {
      + address_space       = [
          + "172.24.0.0/24",
        ]
      + dns_servers         = (known after apply)
      + guid                = (known after apply)
      + id                  = (known after apply)
      + location            = "westus"
      + name                = "DemoVNet2"
      + resource_group_name = "DemoRG"
      + subnet              = (known after apply)
    }

  # azurerm_virtual_network.DemoVNet[2] will be created
  + resource "azurerm_virtual_network" "DemoVNet" {
      + address_space       = [
          + "172.24.0.0/24",
        ]
      + dns_servers         = (known after apply)
      + guid                = (known after apply)
      + id                  = (known after apply)
      + location            = "westus"
      + name                = "DemoVNet3"
      + resource_group_name = "DemoRG"
      + subnet              = (known after apply)
    }

Plan: 7 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── 

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

![Illustration 8](/assets/pvaks/pvaks008.png)  

### 3.2. adding a kubernetes cluster

Again, we'll rely on the documentation to have a sample of a [kubernetes cluster](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster) in hcl:  

```bash

resource "azurerm_kubernetes_cluster" "DemoAKS" {
    

    name                            = "aks-1"
    resource_group_name             = azurerm_resource_group.DemoRG.name
    dns_prefix                      = "aks1"
    location                        = azurerm_resource_group.DemoRG.location

    default_node_pool {
      name                          = "np0"
      node_count                    = 1
      vm_size                       = "Standard_D2s_v4"
      vnet_subnet_id                = azurerm_subnet.DemoSubnet[0].id
    }

    identity {
      type = "SystemAssigned"
    }

    network_profile {
      network_plugin                = "kubenet"
      network_policy                = "calico"

    }
    

    azure_active_directory_role_based_access_control {
        managed                     = true
        tenant_id                   = var.AzureTenantID
        admin_group_object_ids      = [var.AKSAdmins]
        azure_rbac_enabled          = true
    }
}

```

And that is a very basic kubernetes cluster, with just RBAC and AAD integration as per the `azure_active_directory_role_based_access_control` block.

At this point, if we run a `terraform plan` again, we'll just have a classic aks cluster, not yet private:

```bash

terraform plan
azurerm_resource_group.DemoRG: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG]
azurerm_virtual_network.DemoVNet[1]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet2]
azurerm_virtual_network.DemoVNet[2]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet3]
azurerm_virtual_network.DemoVNet[0]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet1]
azurerm_subnet.DemoSubnet[2]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet3/subnets/DemoSubnet3]
azurerm_subnet.DemoSubnet[0]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet1/subnets/DemoSubnet1]
azurerm_subnet.DemoSubnet[1]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet2/subnets/DemoSubnet2]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.DemoAKS will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks1"
      + fqdn                                = (known after apply)
      + http_application_routing_zone_name  = (known after apply)
      + id                                  = (known after apply)
      + kube_admin_config                   = (sensitive value)
      + kube_admin_config_raw               = (sensitive value)
      + kube_config                         = (sensitive value)
      + kube_config_raw                     = (sensitive value)
      + kubernetes_version                  = (known after apply)
      + location                            = "westus"
      + name                                = "aks-1"
      + node_resource_group                 = (known after apply)
      + oidc_issuer_url                     = (known after apply)
      + portal_fqdn                         = (known after apply)
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = (known after apply)
      + private_fqdn                        = (known after apply)
      + public_network_access_enabled       = true
      + resource_group_name                 = "DemoRG"
      + role_based_access_control_enabled   = true
      + run_command_enabled                 = true
      + sku_tier                            = "Free"

      + auto_scaler_profile {
          + balance_similar_node_groups      = (known after apply)
          + empty_bulk_delete_max            = (known after apply)
          + expander                         = (known after apply)
          + max_graceful_termination_sec     = (known after apply)
          + max_node_provisioning_time       = (known after apply)
          + max_unready_nodes                = (known after apply)
          + max_unready_percentage           = (known after apply)
          + new_pod_scale_up_delay           = (known after apply)
          + scale_down_delay_after_add       = (known after apply)
          + scale_down_delay_after_delete    = (known after apply)
          + scale_down_delay_after_failure   = (known after apply)
          + scale_down_unneeded              = (known after apply)
          + scale_down_unready               = (known after apply)
          + scale_down_utilization_threshold = (known after apply)
          + scan_interval                    = (known after apply)
          + skip_nodes_with_local_storage    = (known after apply)
          + skip_nodes_with_system_pods      = (known after apply)
        }

      + azure_active_directory_role_based_access_control {
          + admin_group_object_ids = [
              + "00000000-0000-0000-0000-000000000000",
            ]
          + azure_rbac_enabled     = true
          + managed                = true
          + tenant_id              = "00000000-0000-0000-0000-000000000000"
        }

      + default_node_pool {
          + kubelet_disk_type    = (known after apply)
          + max_pods             = (known after apply)
          + name                 = "np0"
          + node_count           = 1
          + node_labels          = (known after apply)
          + orchestrator_version = (known after apply)
          + os_disk_size_gb      = (known after apply)
          + os_disk_type         = "Managed"
          + os_sku               = (known after apply)
          + type                 = "VirtualMachineScaleSets"
          + ultra_ssd_enabled    = false
          + vm_size              = "Standard_D2s_v4"
          + vnet_subnet_id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet1/subnets/DemoSubnet1"
        }

      + identity {
          + principal_id = (known after apply)
          + tenant_id    = (known after apply)
          + type         = "SystemAssigned"
        }

      + kubelet_identity {
          + client_id                 = (known after apply)
          + object_id                 = (known after apply)
          + user_assigned_identity_id = (known after apply)
        }

      + network_profile {
          + dns_service_ip     = (known after apply)
          + docker_bridge_cidr = (known after apply)
          + ip_versions        = (known after apply)
          + load_balancer_sku  = "standard"
          + network_mode       = (known after apply)
          + network_plugin     = "kubenet"
          + network_policy     = "calico"
          + outbound_type      = "loadBalancer"
          + pod_cidr           = (known after apply)
          + service_cidr       = (known after apply)

          + load_balancer_profile {
              + effective_outbound_ips    = (known after apply)
              + idle_timeout_in_minutes   = (known after apply)
              + managed_outbound_ip_count = (known after apply)
              + outbound_ip_address_ids   = (known after apply)
              + outbound_ip_prefix_ids    = (known after apply)
              + outbound_ports_allocated  = (known after apply)
            }

          + nat_gateway_profile {
              + effective_outbound_ips    = (known after apply)
              + idle_timeout_in_minutes   = (known after apply)
              + managed_outbound_ip_count = (known after apply)
            }
        }

      + windows_profile {
          + admin_password = (sensitive value)
          + admin_username = (known after apply)
          + license        = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

Because we will look into different scenarios, let's add the `count` meta-argument:

```bash

resource "azurerm_kubernetes_cluster" "DemoAKS" {
    count                                   = 3

    name                                    = "aks${count.index+1}"
    resource_group_name                     = azurerm_resource_group.DemoRG.name
    dns_prefix                              = "aks${count.index+1}"
    location                                = azurerm_resource_group.DemoRG.location

    default_node_pool {
      name                                  = "np0"
      node_count                            = 1
      vm_size                               = "Standard_D2s_v4"
      vnet_subnet_id                        = azurerm_subnet.DemoSubnet[count.index].id
    }

    identity {...}

    network_profile {...}
    
    private_cluster_enabled                 = tobool(var.IsAKSPrivate[count.index]) 
    
    private_dns_zone_id                     = var.AKSPRivDNS[count.index] 

    private_cluster_public_fqdn_enabled     = tobool(var.AKSPriwithpubfqdn[count.index])

    azure_active_directory_role_based_access_control {...}
}

```

And some variables:

```bash

variable "IsAKSPrivate" {
  type                                  = list
  description                           = "Define if AKS Cluster is public or private"
  default                               = ["false","false","false"]

}

variable "AKSPRivDNS" {
  type                                  = list
  description                           = ""
  default                               = [null,null,null]

}

variable "AKSPriwithpubfqdn" {
  type                                  = list
  description                           = ""
  default                               = [null,null,null]

}


```

The plan will not change drastically yet:  

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.DemoAKS[0] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks1"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = false
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = (known after apply)
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

  # azurerm_kubernetes_cluster.DemoAKS[1] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks2"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = false
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = (known after apply)
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

  # azurerm_kubernetes_cluster.DemoAKS[2] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks3"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = false
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = (known after apply)
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

Plan: 3 to add, 0 to change, 0 to destroy.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

Notice the value for `private_cluster_enabled` still on `false` and the `private_dns_zone_id` known after apply.

Now let's make some private cluser

### 3.3. Making it private

Refering to the documentation, we have to configure the argument `private_cluster_enabled` to `true` to have a private cluster.
Also, because, as we mentionned earlier, it relies on a private dns zone, we need to configure also the argument `private_dns_zone_id`. The documentation states the following for this argument:  

```hcl

private_dns_zone_id - (Optional) Either the ID of Private DNS Zone which should be delegated to this Cluster, System to have AKS manage this or None. In case of None you will need to bring your own DNS server and set up resolving, otherwise cluster will have issues after provisioning. Changing this forces a new resource to be created.

```

It is not so clearly explained but we can either provide a private dns zone resource id, or `System` or `None`.
We will start with system and see what happen.
For that we will change the variables as follow:

```bash

variable "IsAKSPrivate" {
  type                                  = list
  description                           = "Define if AKS Cluster is public or private"
  default = ["false","true","false"]

}

variable "AKSPRivDNS" {
  type                                  = list
  description                           = "Set the argument private_dns_zone_id. Accepted values are System, None or a resource id in the form /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/RgName/providers/Microsoft.Network/privateDnsZones/privatelink.RegionName.azmk8s.io"
  default = [null,"System",null]

}

```

Notice that we added an explanation to the usage of the variable `AKSPrivDNS`

Now let's run a plan again. We should have a change for the `azurerm_kubernetes_cluster.DemoAKS[1]`:

```bash
===============================================truncated===============================================
  # azurerm_kubernetes_cluster.DemoAKS[1] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks2"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = true
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = "System"
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

```

If you apply the configuration, you should notice that it's faster to provision a public clsuter than a private cluster:

```bash

azurerm_kubernetes_cluster.DemoAKS[0]: Creation complete after 4m0s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ContainerService/managedClusters/aks1]
===============================================truncated===============================================
azurerm_kubernetes_cluster.DemoAKS[1]: Creation complete after 9m2s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ContainerService/managedClusters/aks2]

```

At the end of the apply we should see a few differences:

- First, one cluster is private and the others are not, as planned

![Illustration 9](/assets/pvaks/pvaks009.png)  
  
![Illustration 10](/assets/pvaks/pvaks010.png)  

- Second, there is an additional resource in the AKS managed resource group:

![Illustration 11](/assets/pvaks/pvaks011.png)  
  
![Illustration 12](/assets/pvaks/pvaks012.png)  
  
![Illustration 13](/assets/pvaks/pvaks013.png)  

In particular, we can see a private DNS zone, with a role assignment of contributor assigned to the AKS managed identity:

![Illustration 14](/assets/pvaks/pvaks014.png)  
  
That's fine. We have a private cluster with a DNS zone managed automatically by the AKS service.

What if we don't really have familiarity with DNS configuration? Is there a way to get a private cluster but without this overhead of managing a private DNS zone?

That's the next part of this article ^^

Before looking into those topics, you may want to add a lifecycle block on the subnets resources.
Indeed, when a private endpoint is configured, there is a change that is made automatically on the `enforce_private_link_endpoint_network_policies` argument.
You may have this result when `terraform plan` is re-run:  
  
```bash

  # azurerm_subnet.DemoSubnet[1] has changed
  ~ resource "azurerm_subnet" "DemoSubnet" {
      ~ enforce_private_link_endpoint_network_policies = false -> true
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet2/subnets/DemoSubnet2"
        name                                           = "DemoSubnet2"
        # (6 unchanged attributes hidden)
    }

```
  
Hence the lifecycle block in the subnets:  

```bash

resource "azurerm_subnet" "DemoSubnet" {

  lifecycle {
    ignore_changes = [
      enforce_private_link_endpoint_network_policies
    ]
  }
    count = 3
    name = "DemoSubnet${count.index+1}"
    address_prefixes = ["172.24.0.0/26"]
    resource_group_name = azurerm_resource_group.DemoRG.name
    virtual_network_name = azurerm_virtual_network.DemoVNet[count.index].name
  
}

```

### 3.4 private but not private

So about a private cluster, but without a private DNS zone, we can see the argument

```hcl

private_cluster_public_fqdn_enabled - (Optional) Specifies whether a Public FQDN for this Private Cluster should be added. Defaults to false.


```
  
Let's modify our variables to take that into acccount:

```bash

variable "IsAKSPrivate" {
  type                                  = list
  description                           = "Define if AKS Cluster is public or private"
  default = ["false","true","true"]

}

variable "AKSPRivDNS" {
  type                                  = list
  description                           = "Set the argument private_dns_zone_id. Accepted values are System, None or a resource id in the form /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/RgName/providers/Microsoft.Network/privateDnsZones/privatelink.RegionName.azmk8s.io"
  default = [null,"System","None"]

}

variable "AKSPriwithpubfqdn" {
  type                                  = list
  description                           = "Set the argument private_cluster_public_fqdn_enabled to ne able to use a private cluster with a public fqdn. Requires the feature Microsoft.ContainerService/EnablePrivateClusterPublicFQDN"
  default = [null,null,"true"]

}

```

```bash

  # azurerm_kubernetes_cluster.DemoAKS[2] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix                          = "aks3"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = true
      + private_cluster_public_fqdn_enabled = true
      + private_dns_zone_id                 = "None"
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

```

Once the apply is completed, we have a new AKS cluster, still private but with a public fqdn.

![Illustration 15](/assets/pvaks/pvaks015.png)  
  
Since we don't have a GUI for the Azure public DNS zone, we can use a `nslookup` command to check the IP registered behind the fqdn and we do get a RFC 1819 IP address, as opposed to the public AKS wth which the `nslookup` gives us a public IP:  
  
```bash

nslookup aks3-c6a31191.hcp.westus.azmk8s.io
Serveur :   dns.google
Address:  8.8.8.8

Réponse ne faisant pas autorité :
Nom :    aks3-c6a31191.hcp.westus.azmk8s.io
Address:  172.24.0.4

```

```bash

nslookup aks1-47e76fe1.hcp.westus.azmk8s.io
Serveur :   dns.google
Address:  8.8.8.8

Réponse ne faisant pas autorité :
Nom :    aks1-47e76fe1.hcp.westus.azmk8s.io
Address:  13.87.224.26

```

So easily enough we found 2 ways to build the private AKS cluster from a terraform stand point.
We now have a private cluster but without the overhead of a DNS zone to manage. Indeed, in this case, the DNS is fully managed on the Azure side, zone included.

Now, what happen if governance decision requires DNS to be managed outside of the Kubernetes scope?
How could we avoid as many private DNS zone as there are AKS clusters?

This is a typical question in hybrid environments leveraging private endpoint for all PaaS instances.
Usually, DNS is managed centrally by a team. So in our AKS case, it would mean bring our own DNS zone.

### 3.5. Bring you own DNS zone

Where are we now?

On the previous steps we noticed that the DNS zone provisionned by AKS is configured with an RBAC assignment to the AKS Identity.

Let's consider a private DNS zone already existing. In this case, the AKS cluster will need to be assigned the appropriate role on the DNS zone. Chances are that the `contributor` role will be too high, so we will have to use a less permissive role. The terraform documentation propose to use the `Private DNS Zone Contributor` role which comes with the following pemrissions:  

Also the documentation is stating the following:

```hcl

If you use BYO DNS Zone, AKS cluster should either use a User Assigned Identity or a service principal (which is deprecated) with the Private DNS Zone Contributor role and access to this Private DNS Zone.

```

Since we do not like deprecated stuff, it's time to consider the User Assigned Identity that we mention earlier.

It will implies to create this resource, or at least to have the corresponding principal id so that we can configure our cluster declaration `Identity` block as below

```bash

    identity {
      type                                  = "UserAssigned"
      identity_ids                          = ["/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aksUAI"]

    }
```

If we take the hypothesis that we have to create the User Assigned Identity, we will add the following resources:

- A User Assigned Identity Terraform resource:

```bash

resource "azurerm_user_assigned_identity" "aksUAI" {
  resource_group_name                       = azurerm_resource_group.DemoRG.name
  location                                  = azurerm_resource_group.DemoRG.location

  name                                      = "aksUAI"
}

```

- A role assignement on the target DNS zone that already exist:

```bash

resource "azurerm_role_assignment" "DNSContributor" {

  provider                                  = azurerm.mgmt

  scope                                     = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns"
  role_definition_name                      = "Private DNS Zone Contributor"
  principal_id                              = azurerm_user_assigned_identity.aksUAI.object_id
}

```

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "Lets you manage DNS zones and record sets in Azure DNS, but does not let you control who has access to them.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/befefa01-2a29-4197-83a8-272ff33ce314",
  "name": "befefa01-2a29-4197-83a8-272ff33ce314",
  "permissions": [
    {
      "actions": [
        "Microsoft.Authorization/*/read",
        "Microsoft.Insights/alertRules/*",
        "Microsoft.Network/dnsZones/*",
        "Microsoft.ResourceHealth/availabilityStatuses/read",
        "Microsoft.Resources/deployments/*",
        "Microsoft.Resources/subscriptions/resourceGroups/read",
        "Microsoft.Support/*"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "DNS Zone Contributor",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

Note the `provider` argument. This is because, in my case, and probably most of the case, the DNS zone is hosted on a different Azure subscription. Hence the provider refering to an alias looking like that:

```bash

provider "azurerm" {
  subscription_id                          = var.AzureSubscriptionIDMgmt
  client_id                                = var.AzureClientID
  client_secret                            = var.AzureClientSecret
  tenant_id                                = var.AzureTenantID

  alias                                    = "mgmt"

  features {...}
  
}

```

Last, the User Assigned Identity needs also to be granted a role on the Virtual Network.
That's because apart from writing in the DNS zone the hostname corresponding to the AKS cluster, it is also necessary to link the DNS zone to the Virtual Network. Failure to do so will result in a time out on the provisioning, with terraform trying to perform the link but being unable to do it.
Note that this time there is no alias provider involved. And that's because the Network for AKS cannot be in a different subscription than the AKS cluster. That may seem obvious, but let's be accurate ^^.

```bash

resource "azurerm_role_assignment" "NetworkContributor" {


  scope                                     = azurerm_resource_group.DemoRG.id
  role_definition_name                      = "Network Contributor"
  principal_id                              = azurerm_user_assigned_identity.aksUAI.object_id
}

```

```json

{
  "assignableScopes": [
    "/"
  ],
  "description": "Lets you manage networks, but not access to them.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/4d97b98b-1d4f-4787-a291-c67834d212e7",
  "name": "4d97b98b-1d4f-4787-a291-c67834d212e7",
  "permissions": [
    {
      "actions": [
        "Microsoft.Authorization/*/read",
        "Microsoft.Insights/alertRules/*",
        "Microsoft.Network/*",
        "Microsoft.ResourceHealth/availabilityStatuses/read",
        "Microsoft.Resources/deployments/*",
        "Microsoft.Resources/subscriptions/resourceGroups/read",
        "Microsoft.Support/*"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "Network Contributor",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}

```

There are a few things to modify in the configuration now. 

First, because having a `count = somenumber` is not ideal, let's change that and refer rather to the lenght of a list with `count = length(var.IsAKSPrivate)`. This is not related directly to the problem at hand, but still, this is a terraform centric article so...

```bash

resource "azurerm_kubernetes_cluster" "DemoAKS" {
    count                                   = length(var.IsAKSPrivate)
===============================================truncated===============================================
}

```

Then let's add a new variable to specify if we bring a DNS zone:

```bash

variable "IsBYODNSPVZone" {
  type                                  = list
  description                           = "Define if Private AKS Cluster use a DNS zone managed elsewhere"
  default = ["false","false","false","true"]

}

```

We also want to modify a few of our variables default value so we will have something like that in tfvars file:

```bash

IsAKSPrivate                        = ["false","true","true","true"]
AKSPRivDNS                          = [null,"System","None","/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"]
AKSPriwithpubfqdn                   = [null,null,"true",null]
IsBYODNSPVZone                      = ["false","false","false","true"]

```

And now a few modification on the aks resource definition:

```bash

resource "azurerm_kubernetes_cluster" "DemoAKS" {
    count                                   = length(var.IsAKSPrivate)

    name                                    = "aks${count.index+1}"
    resource_group_name                     = azurerm_resource_group.DemoRG.name

    # Adding a conditional on the dns_prefix argument because it cannot co-exist with the argument dns_prefix_private_cluster
    dns_prefix                              = tobool(var.IsBYODNSPVZone[count.index]) ? null : "aks${count.index+1}"
    location                                = azurerm_resource_group.DemoRG.location

    # Adding a conditional on the dns_prefix_private_cluster argument. Set to null when the variable IsBYODNSPVZone is null. It is set only for case when a dns is brought.
    dns_prefix_private_cluster              = tobool(var.IsBYODNSPVZone[count.index]) ? "aks${count.index+1}" : null

    default_node_pool {...}

    # Adding a conditional on the type argument for the identity block. Depending on the case, we want to have an UAI instead of a system Assigned Identity, so that we can assign it the role require for AKS provision.
    # Failure to do so will result in a time out in provisioning.
    # When type is set to UserAssigned, it is required to specify the resource ids of the UAI that are assigned. Note the plural meaning that we should provide a set to the argument identity_ids.
    identity {
      type                                  = tobool(var.IsBYODNSPVZone[count.index]) ? "UserAssigned" : "SystemAssigned"
      identity_ids                          = tobool(var.IsBYODNSPVZone[count.index]) ? toset([azurerm_user_assigned_identity.aksUAI.id]) : toset([])
    }

    network_profile {...}
    
    private_cluster_enabled                 = tobool(var.IsAKSPrivate[count.index]) 
    
    private_dns_zone_id                     = var.AKSPRivDNS[count.index] 

    private_cluster_public_fqdn_enabled     = tobool(var.AKSPriwithpubfqdn[count.index])

    azure_active_directory_role_based_access_control {...}
}

```

Last we can re-run `terraform plan`:

```bash

terraform plan
azurerm_resource_group.DemoRG: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG]
azurerm_virtual_network.DemoVNet[0]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet1]
azurerm_virtual_network.DemoVNet[2]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet3]
azurerm_virtual_network.DemoVNet[1]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet2]
azurerm_subnet.DemoSubnet[0]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet1/subnets/DemoSubnet1]
azurerm_subnet.DemoSubnet[2]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet3/subnets/DemoSubnet3]
azurerm_subnet.DemoSubnet[1]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.Network/virtualNetworks/DemoVNet2/subnets/DemoSubnet2]
azurerm_kubernetes_cluster.DemoAKS[2]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ContainerService/managedClusters/aks3]
azurerm_kubernetes_cluster.DemoAKS[1]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ContainerService/managedClusters/aks2]
azurerm_kubernetes_cluster.DemoAKS[0]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG/providers/Microsoft.ContainerService/managedClusters/aks1]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
  ~ update in-place

Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.DemoAKS[3] will be created
  + resource "azurerm_kubernetes_cluster" "DemoAKS" {
      + dns_prefix_private_cluster          = "aks4"
      + fqdn                                = (known after apply)
===============================================truncated===============================================
      + private_cluster_enabled             = true
      + private_cluster_public_fqdn_enabled = false
      + private_dns_zone_id                 = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"
      + private_fqdn                        = (known after apply)
===============================================truncated===============================================
    }

  # azurerm_role_assignment.DNSContributor will be created
  + resource "azurerm_role_assignment" "DNSContributor" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = (known after apply)
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Private DNS Zone Contributor"
      + scope                            = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_role_assignment.NetworkContributor will be created
  + resource "azurerm_role_assignment" "NetworkContributor" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = (known after apply)
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Network Contributor"
      + scope                            = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/DemoRG"
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_subnet.DemoSubnet[3] will be created
  + resource "azurerm_subnet" "DemoSubnet" {
===============================================truncated===============================================
    }

  # azurerm_user_assigned_identity.aksUAI will be created
  + resource "azurerm_user_assigned_identity" "aksUAI" {
===============================================truncated===============================================
    }

  # azurerm_virtual_network.DemoVNet[3] will be created
  + resource "azurerm_virtual_network" "DemoVNet" {
===============================================truncated===============================================
    }

Plan: 6 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── 

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

So then, after apply, we should have a new private cluster, but this time, instead of letting the AKS identity managed its own DNS private zone, we used one already existing and potentially in the scope of some other team.

As long as this team is eager to let us write our own dns record, we're good to go. And the + is that it will allow the centralized management of DNS for hydrid DNS scenario and private network access from connected networks, which was kind of the point.

![Illustration 16](/assets/pvaks/pvaks016.png)  

## 4. Before leaving

So we have seen all of it I guess.  

To summarize:  

- The technical part is not that difficult to get a private cluster
- We can do:
  - private cluster with DNS managed by AKS
  - private cluster with public fqdn a.k.a fully managed from the DNS point of view by Microsoft
  - private cluster with bring your own DNS a.k.a the model where you are not alone in using private endpoint

Apart from the techicalities, we did hint at a few governance decisions, which may be impacted by compliance decisions also.
This is not the place to address that but is still a mandatory topic to address.

Last, there is a [new preview regarding the API server for AKS](https://docs.microsoft.com/en-us/azure/aks/api-server-vnet-integration), which allow to integrate directly in a Virtual Network the API server.
We'll probably give it a ride in a coming article, but for today, that's enough.