---
layout: post
title:  "Terraform Import blocks"
date:   2023-07-02 18:00:00 +0200
year: 2023
categories: Azure Terraform
---

Hi!

Last time we talked about importing resources in the state with `aztfexport`.
Since I was late in writing this part, Hashicorp had the time to release its v1.5.0 which bring the import blocks, a native capability for generating terraform configuration. Let's have a look

## Prerequisites

Before using the import block feature, we need to have terraform version 1.5.x. That's a mandatory prerquisite, so I would probably forced that through a required_version statement:

```go

terraform {

  required_version = ">=1.5.0"
}

```

The other prerequisite is to know which resources are planned to be imported. Specifically, the Id of the resources is required in the import process. Same as before with the manual import finally, so no big surprise here.
Let's move to look at the concepts now

## Concepts

This is a brand new feature of terraform version `1.5.0`. It brings a new kind of block called the import block:

```go

import {
    to = <resource_address_in_tf_config>
    id = <resource_id_in_the_cloud_provider>
}


```

To use this block, we need the id of the resource to be imported an the resource name in the terraform config.

It is usable basically, meaning without code generation. In this case, we would need to write the resource configuration in a `.tf` file.
This is very similar to the *classic* import, in the sense that there is some manual steps involved.
Still, in some case, it could be useful and hence easier to add just the import block near an existing terraform conifugration. Also, through this block, we can keep a history of the import in the source control.
From my point of view, that would be a win ^^.

The less basic use is with the `-generate-config-out`. As the name implies, it does allow the generation of the terraform config, inside a file that is specified in the terraform plan command: 

```bash

yumemaru@azure/$ terraform plan -generate-config-out=generatedresources.tf

```

The plan should generate the terraform configuration in the specified file, and specify in its output the number of resources to be imported.
Afterward, executing an apply will write the resources into the state, and that's it.
To summarize, the workflow of import looks like that: 

![illustration1](/assets/tfimportblock/importingresourcewithimportblock.png)

Which is, in the end, way simpler than before ^^
Ok let's try this for real now.

## Generating Terraform configuration

To illustrate the configuration generation, let's take resources in an Azure resource group: 

![illustration2](/assets/tfimportblock/rgtoimport.png)

As discussed, we need the resources ids. Let's run a bit of az cli:

```bash

yumemaru@azure$ az resource list -g rg-dns | jq .[].id
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/aks.teknews.cloud"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/lab.teknews.cloud"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io"
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"

```

Now we need to write our import blocks. We'll put everything in a `import.tf` file:

```go


import {
    to = azurerm_dns_zone.AKSTeknews
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud"
}

import {
    to = azurerm_dns_zone.LabTeknews
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud"

}

import {
    to = azurerm_private_dns_zone.LabPsql
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com"
}

import {
    to = azurerm_private_dns_zone_virtual_network_link.LabPsqlLink
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink"
}

import {
    to = azurerm_private_dns_zone.AKSEastUs
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io"
}

import {
    to = azurerm_private_dns_zone.KV
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net"

}

import {
    to = azurerm_private_dns_zone.AKSWestEu
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io"
}

import {
    to = azurerm_private_dns_zone.AKSWestUs
    id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"
}


```

A word of caution here, be sure that the case is as expected by terraform. Sometimes, the output from the az cli does not follow the proper case and a lower case appears instead of an upper case. In this sample the DNS zone Id that we got in the az cli output is `"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/lab.teknews.cloud"` while terraform expect the resource to be `dnsZones`. Just be careful with that.

At this point we have prepared the resources to be imported. We want to generate the configuration, so wewill use the argument `-generate-config-out` with `terraform plan`

```bash

yumemaru@azure$ terraform plan -generate-config-out=generatedresources.tf
azurerm_dns_zone.AKSTeknews: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_private_dns_zone.AKSWestUs: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
azurerm_private_dns_zone.AKSEastUs: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.KV: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_dns_zone.LabTeknews: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
azurerm_private_dns_zone.LabPsql: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_private_dns_zone.AKSWestEu: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_private_dns_zone.KV: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_dns_zone.AKSTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_private_dns_zone.AKSEastUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_private_dns_zone.AKSWestEu: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_private_dns_zone.LabPsql: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_dns_zone.LabTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
data.azurerm_resource_group.rgdns: Reading...
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.AKSWestUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
data.azurerm_resource_group.rgdns: Read complete after 0s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns]

Terraform will perform the following actions:

  # azurerm_dns_zone.AKSTeknews will be imported
  # (config will be generated)
    resource "azurerm_dns_zone" "AKSTeknews" {
        id                        = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud"
        max_number_of_record_sets = 10000
        name                      = "aks.teknews.cloud"
        name_servers              = [
            "ns1-05.azure-dns.com.",
            "ns2-05.azure-dns.net.",
            "ns3-05.azure-dns.org.",
            "ns4-05.azure-dns.info.",
        ]
        number_of_record_sets     = 4
        resource_group_name       = "rg-dns"
        tags                      = {}

        soa_record {
            email         = "azuredns-hostmaster.microsoft.com"
            expire_time   = 2419200
            fqdn          = "aks.teknews.cloud."
            host_name     = "ns1-05.azure-dns.com."
            minimum_ttl   = 300
            refresh_time  = 3600
            retry_time    = 300
            serial_number = 1
            tags          = {}
            ttl           = 3600
        }
    }

======================================================================truncated======================================================================

  # azurerm_private_dns_zone_virtual_network_link.LabPsqlLink will be imported
  # (config will be generated)
    resource "azurerm_private_dns_zone_virtual_network_link" "LabPsqlLink" {
        id                    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink"
        name                  = "psqldnslink"
        private_dns_zone_name = "lab.postgres.database.azure.com"
        registration_enabled  = false
        resource_group_name   = "rg-dns"
        tags                  = {}
        virtual_network_id    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-agicpeered/providers/Microsoft.Network/virtualNetworks/vnet-aksagicpeered"
    }

Plan: 8 to import, 0 to add, 0 to change, 0 to destroy.
╷
│ Warning: Config generation is experimental
│ 
│ Generating configuration during import is currently experimental, and the generated configuration format may change in future versions.
╵

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Terraform has generated configuration and written it to generatedresources.tf. Please review the configuration and edit it as necessary
before adding it to version control.

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform
apply" now.

```

Without the `--generated-config-out`, wre would have had an error stating that the config did not exist for our resources to be imported. With the switch we get the file `generatedresources.tf`. We did so we had no error, and we have a new config file

```go

# __generated__ by Terraform
# Please review these resources and move them into your main configuration files.

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink"
resource "azurerm_private_dns_zone_virtual_network_link" "LabPsqlLink" {
  name                  = "psqldnslink"
  private_dns_zone_name = "lab.postgres.database.azure.com"
  registration_enabled  = false
  resource_group_name   = "rg-dns"
  tags                  = {}
  virtual_network_id    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-agicpeered/providers/Microsoft.Network/virtualNetworks/vnet-aksagicpeered"
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com"
resource "azurerm_private_dns_zone" "LabPsql" {
  name                = "lab.postgres.database.azure.com"
  resource_group_name = "rg-dns"
  tags                = {}
  soa_record {
    email        = "azureprivatedns-host.microsoft.com"
    expire_time  = 2419200
    minimum_ttl  = 10
    refresh_time = 3600
    retry_time   = 300
    tags         = {}
    ttl          = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io"
resource "azurerm_private_dns_zone" "AKSWestUs" {
  name                = "privatelink.westus.azmk8s.io"
  resource_group_name = "rg-dns"
  tags                = {}
  soa_record {
    email        = "azureprivatedns-host.microsoft.com"
    expire_time  = 2419200
    minimum_ttl  = 10
    refresh_time = 3600
    retry_time   = 300
    tags         = {}
    ttl          = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io"
resource "azurerm_private_dns_zone" "AKSWestEu" {
  name                = "privatelink.westeurope.azmk8s.io"
  resource_group_name = "rg-dns"
  tags = {
    ManagedBy = "The portal T_T"
  }
  soa_record {
    email        = "azureprivatedns-host.microsoft.com"
    expire_time  = 2419200
    minimum_ttl  = 10
    refresh_time = 3600
    retry_time   = 300
    tags         = {}
    ttl          = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io"
resource "azurerm_private_dns_zone" "AKSEastUs" {
  name                = "privatelink.eastus.azmk8s.io"
  resource_group_name = "rg-dns"
  tags = {
    ManagedBy = "The portal T_T"
    Usage     = "pocPVLink"
  }
  soa_record {
    email        = "azureprivatedns-host.microsoft.com"
    expire_time  = 2419200
    minimum_ttl  = 10
    refresh_time = 3600
    retry_time   = 300
    tags         = {}
    ttl          = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net"
resource "azurerm_private_dns_zone" "KV" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = "rg-dns"
  tags = {
    ManagedBy = "The portal T_T"
    Usage     = "pocPVLink"
  }
  soa_record {
    email        = "azureprivatedns-host.microsoft.com"
    expire_time  = 2419200
    minimum_ttl  = 10
    refresh_time = 3600
    retry_time   = 300
    tags         = {}
    ttl          = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud"
resource "azurerm_dns_zone" "AKSTeknews" {
  name                = "aks.teknews.cloud"
  resource_group_name = "rg-dns"
  tags                = {}
  soa_record {
    email         = "azuredns-hostmaster.microsoft.com"
    expire_time   = 2419200
    host_name     = "ns1-05.azure-dns.com."
    minimum_ttl   = 300
    refresh_time  = 3600
    retry_time    = 300
    serial_number = 1
    tags          = {}
    ttl           = 3600
  }
}

# __generated__ by Terraform from "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud"
resource "azurerm_dns_zone" "LabTeknews" {
  name                = "lab.teknews.cloud"
  resource_group_name = "rg-dns"
  tags = {
    Country       = "fr"
    Environment   = "Lab"
    ManagedBy     = "The portal T_T"
    Project       = "tflab"
    ResourceOwner = "That would be me"
  }
  soa_record {
    email         = "azuredns-hostmaster.microsoft.com"
    expire_time   = 2419200
    host_name     = "ns1-01.azure-dns.com."
    minimum_ttl   = 300
    refresh_time  = 3600
    retry_time    = 300
    serial_number = 1
    tags          = {}
    ttl           = 3600
  }
}


```

We could now move the generated config in other part of the folder, and for example, pyut the new generated resources next to the import block.
But htat's not mandatory. What is mandatory though, is to pass an apply step so that the resources are effectivemy imported in our state.
If we were to pass a `terraform state list` now, we would have no output.



```bash

df@df2204lts:~/Documents/myrepo/terrafy/tfimport$ terraform apply
azurerm_private_dns_zone.KV: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.AKSWestEu: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_dns_zone.LabTeknews: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
azurerm_private_dns_zone.LabPsql: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.KV: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_private_dns_zone.AKSWestUs: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
azurerm_private_dns_zone.AKSEastUs: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_dns_zone.AKSTeknews: Preparing import... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_private_dns_zone.AKSWestUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
data.azurerm_resource_group.rgdns: Reading...
azurerm_private_dns_zone.AKSWestEu: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_dns_zone.LabTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
azurerm_dns_zone.AKSTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_private_dns_zone.AKSEastUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_private_dns_zone.LabPsql: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
data.azurerm_resource_group.rgdns: Read complete after 0s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns]

Terraform will perform the following actions:

  # azurerm_dns_zone.AKSTeknews will be imported
    resource "azurerm_dns_zone" "AKSTeknews" {
        id                        = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud"
        max_number_of_record_sets = 10000
        name                      = "aks.teknews.cloud"
        name_servers              = [
            "ns1-05.azure-dns.com.",
            "ns2-05.azure-dns.net.",
            "ns3-05.azure-dns.org.",
            "ns4-05.azure-dns.info.",
        ]
        number_of_record_sets     = 4
        resource_group_name       = "rg-dns"
        tags                      = {}

        soa_record {
            email         = "azuredns-hostmaster.microsoft.com"
            expire_time   = 2419200
            fqdn          = "aks.teknews.cloud."
            host_name     = "ns1-05.azure-dns.com."
            minimum_ttl   = 300
            refresh_time  = 3600
            retry_time    = 300
            serial_number = 1
            tags          = {}
            ttl           = 3600
        }
    }

======================================================================truncated======================================================================

  # azurerm_private_dns_zone_virtual_network_link.LabPsqlLink will be imported
    resource "azurerm_private_dns_zone_virtual_network_link" "LabPsqlLink" {
        id                    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink"
        name                  = "psqldnslink"
        private_dns_zone_name = "lab.postgres.database.azure.com"
        registration_enabled  = false
        resource_group_name   = "rg-dns"
        tags                  = {}
        virtual_network_id    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-agicpeered/providers/Microsoft.Network/virtualNetworks/vnet-aksagicpeered"
    }

Plan: 8 to import, 0 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 

  azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.LabPsql: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_private_dns_zone.LabPsql: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_private_dns_zone.AKSEastUs: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_private_dns_zone.AKSEastUs: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_private_dns_zone.KV: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_private_dns_zone.KV: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_private_dns_zone.AKSWestEu: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_private_dns_zone.AKSWestEu: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_private_dns_zone.AKSWestUs: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
azurerm_private_dns_zone.AKSWestUs: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
azurerm_dns_zone.AKSTeknews: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_dns_zone.AKSTeknews: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
azurerm_dns_zone.LabTeknews: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
azurerm_dns_zone.LabTeknews: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]

Apply complete! Resources: 8 imported, 0 added, 0 changed, 0 destroyed.

```

Now that the apply has been completed, we do have resources in our state:


```bash

df@df2204lts:~/Documents/myrepo/terrafy/tfimport$ terraform state list
data.azurerm_resource_group.rgdns
azurerm_dns_zone.AKSTeknews
azurerm_dns_zone.LabTeknews
azurerm_private_dns_zone.AKSEastUs
azurerm_private_dns_zone.AKSWestEu
azurerm_private_dns_zone.AKSWestUs
azurerm_private_dns_zone.KV
azurerm_private_dns_zone.LabPsql
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink
df@df2204lts:~/Documents/myrepo/terrafy/tfimport$ terraform plan
data.azurerm_resource_group.rgdns: Reading...
azurerm_private_dns_zone_virtual_network_link.LabPsqlLink: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/virtualNetworkLinks/psqldnslink]
azurerm_private_dns_zone.AKSWestUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io]
azurerm_private_dns_zone.LabPsql: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com]
azurerm_dns_zone.LabTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud]
azurerm_private_dns_zone.KV: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net]
azurerm_private_dns_zone.AKSWestEu: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io]
azurerm_private_dns_zone.AKSEastUs: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io]
azurerm_dns_zone.AKSTeknews: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud]
data.azurerm_resource_group.rgdns: Read complete after 0s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-dns]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

```

And that's about all. Let's wrap it now ^^

## To conclude

Hashicorp has come a long way since the begining of Terraform. The now long awaited feature to generate automatically the resource configuration to be imported is available, and even the import process is hugely simpler than before.
However, while the feature was not there, the landscape of available product to fill this gap have developped and we have other tool that can do something similar.
In the case for Azure, we have for example Aztfexport which can also help to ease the only remaining pain point of the native import feature fo Terraform which is to identify the resources Ids to be imported.
I would also like to add that AzTfexport can see more from a resource group than Az cli, as I did in this article, and potentially be more thorough for resources that are not visible in a reource group (because they are nested in another resource, for example subnets in vnet, or DNS record in DNS zones...).
If I had to chose, I'm not sure I would directly go to the native tool yet. Maybe a mix of more than one tool.
On the other hand, while it's still early, I don't know the future of 3rd party tool with the integration of the feature natively so we'll have to wait and see.

Lastly, As for Aztfexport, by no mean is the generated configuration completely valid for a target IaC management. There are revamp of the configuration to be planned and thus probably some state mv stuff. 
I'll probably talk about that in a future article.

For now that's all.

See you people
