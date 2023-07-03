---
layout: post
title:  "Aztfexport - A tool to export existing Azure resources to terraform configuration"
date:   2023-06-22 18:00:00 +0200
year: 2023
categories: Azure Terraform
---

Hello people!

If you remember, I published a post about the AzAPI provider for terraform, which gives the same level of support to deploy resources in Azure as ARM template or Bicep.
At the time the provider was first announced by Microsoft team, another tool was also released which proposed an long awaited feature: generate automatically terraform configuration for existing resources.
Note that with terraform version 1.5.x, there's a native capability to generate terraform config also from imported resources. We'll see that in a coming soon article also ^^

Now, about Microsoft tool. In the early days it was called terrafy and was recently renamed aztfexport. It does offer option for helping in importing Azure resources and generating the associated terraform configuration.

Let's have a look at this in this article.

## 1. Managing in terraform existing Azure resources

Before diving into the tool, let's get some context.

When we begin the journey of IaC adoption, being with terraform or any other tool, there's a need for a strong governance.
Infrastructure created, in Azure for example, should not be created with many different tools.
If the choice is to use something like terraform, then, at least on the defined scope, it should remain the only tool.

But, because, you know there's always a but ^^, It also happen that some resources may be created manually, or through another tool, such as PowerShell or Az Cli.

In this case, as it has been since the begining of terraform, one must rely on manipulating the state, and specifically, importing resource in the state.
If you look at it, it's not that difficult. However it is quite long and if not painful, at least error prone and time consuming.

![illustration1](/assets/aztfexport/importingresource.png)

The terraform documentation provides a sample to import all resources, as in the case of a virtual network described below:

```go

terraform import azurerm_virtual_network.exampleNetwork /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup1/providers/Microsoft.Network/virtualNetworks/myvnet1

```

But that means we keep a one by one resource approach.

Again, time consuming and error prone.

Fortunately, we have the `aztfexport` tool

## 2. Aztfexport concepts

So, the `aztfexport` tool is here to provide us the automated generation  of the configuration. In our schema before, we can get rid of the steps previously in blue ^^

As stated in the documentation, we can import in terraform (or export from Azure, hence the name of the tool) indivudual resource, resources under a resource group or even a custom set of resources.

Under the hood, the tool relies on `terraform` binaries, and on 2 other go tools, namely `aztft` and `tfadd`.
As one can read on aztfexport [readme](https://github.com/Azure/aztfexport), first, `aztft` identitfy a resource type from the azurerm resource id. Then, `terraform` import is invoked, and last, `tfadd` generates the HCL configuraztionfor each imported resources.

![illustration2](/assets/aztfexport/aztfexportworkflow.png)

I guess that's all for the concepts. Nothing big, on paper, but potentially really helpful because of the automated approach of the import and hcl configuration creation. Let's see the installation now.

## 3. Installation

Installing aztfexport can be done through differents options. I personnaly prefer to use, a packet manager, in my case apt. This is documented quite well on the Github repo of aztfexport.

In this case, there actually 2 steps, first, adding the Microsoft gpg, then install the package.

```bash

# Add ms gpg key

curl -sSL https://packages.microsoft.com/keys/microsoft.asc > /etc/apt/trusted.gpg.d/microsoft.asc

# Add repository for aztfexoprt installation

apt-add-repository https://packages.microsoft.com/ubuntu/22.04/prod

# aztfexport installation

apt-get install aztfexport

```

When the installation is complete, it sshould be possible to query the version as follow

```bash

yumemaru@azure:~$ aztfexport 
NAME:
   aztfexport - A tool to bring existing Azure resources under Terraform's management

USAGE:
   aztfexport <command> [option] <scope>

VERSION:
   v0.12.0(fc6e1ff)

COMMANDS:
   config              Configuring the tool
   resource, res       Exporting a single resource
   resource-group, rg  Exporting a resource group and the nested resources resides within it
   query               Exporting a customized scope of resources determined by an Azure Resource Graph where predicate
   mapping-file, map   Exporting a customized scope of resources determined by the resource mapping file
   help, h             Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version


```

We are now ready to handle some resource import. Let's see that

## 4. Testing on some resources


To illustrate our purpose, we'll take a very simple use case first. 
I have private dns zones that I created a long time ago, and I would like to move the lifecycle of those resources in terraform.

![illustration2](/assets/aztfexport/sourcersc001.png)

The attentive readers will have noticed that we have an option to import a set of resources from a resource group. But first we need to be logged in on Azure, and have the appropriate subscription selected.

```bash

yumemaru@azure$ az account show
{
  "environmentName": "AzureCloud",
  "homeTenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "isDefault": true,
  "managedByTenants": [],
  "name": "DFR_MGMT",
  "state": "Enabled",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "user": {
    "name": "",
    "type": "user"
  }
}

```

Afterward, we can try an import of resources contained in a specific resource group. In our case, the resource group is named `rg-dns` 

```bash

yumemaru@azure$ aztfexport resource-group rg-dns

```

The following screen should be displayed at first:

![illustration3](/assets/aztfexport/initializingtfexport.png)

And this one when the initialization is completed: 

```bash

   Microsoft Azure Export for Terraform 
  
     rg-dns                                                                                                                                 
                                                                                                                                            
    24 items                                                                                                                                
                                                                                                                                            
  │ 💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns                                                             
  │ azurerm_resource_group.res-0                                                                                                            
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud      
    azurerm_dns_zone.res-1                                                                                                                  
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud/CNAME
    azurerm_dns_cname_record.res-2                                                                                                          
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud/CNAME
    azurerm_dns_cname_record.res-3                                                                                                          
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud/NS/@ 
    azurerm_dns_ns_record.res-4                                                                                                             
                                                                                                                                            
    /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/aks.teknews.cloud/SOA/@  
    (Skip)                                                                                                                                  
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud      
    azurerm_dns_zone.res-6                                                                                                                  
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud/CNAME
    azurerm_dns_cname_record.res-7                                                                                                          
                                                                                                                                            
    💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud/CNAME
    azurerm_dns_cname_record.res-8                                                                                                          
                                                                                                                                            
    •••                                                                                                                                     
                                                                                                                                            
    ↑/k up • ↓/j down • / filter • delete skip • e show error • r show recommendation • w import • s save • q quit • ? more                 



```

We have the possibility to scroll on each resource listed and use the options displayed on the bottom of the screen. Using the `r` option, we can see that the proposed resource is in this case an `azurerm_dns_zone`

```bash   

Microsoft Azure Export for Terraform 
  
     rg-dns   Possible resource type(s): azurerm_dns_zone                                                                                   
                                                                                                                                            
    24 items                                                                                     
                                                                                    
  │ 💡/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud      
  │ azurerm_dns_zone.res-1

```

It may be possible that a target resource in terraform is not proposed, because there is no equivalence in the azurerm provider:

```bash

  Microsoft Azure Export for Terraform 
  
     rg-dns   No resource type recommendation is available...                                                                               
                                                                                                                                            
    24 items                                                                                                                                                                                                                                                            
  │ /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/aks.teknews.cloud/SOA/@  
  │ (Skip) 

```

Notice the proposed resource name belo the resource id. It can be customized by selecting the resource through the terminal.

Once ready, we can start the import with the `w` command

![illustration4](/assets/aztfexport/startimport.png)


```bash

 Microsoft Azure Export for Terraform 
  
  ⣻   Skipping /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/private
  
  🌽 /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastu
  🍒 /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastu
  🧲 /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vault
  🍥 /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vault
  🐯 /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.weste
  
  
  █████████████████████████████████████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░  83%


```
At the end of the import, press any key as requested:

![illustration5](/assets/aztfexport/importcompleted.png)

We get the following files afterward:

```bash

yumemaru@azure$ tree
.
├── aztfexportResourceMapping.json
├── aztfexportSkippedResources.txt
├── main.tf
├── provider.tf
├── terraform.tf
└── terraform.tfstate

```

In the `main.tf`, we find all the imported (in the state, but exported from Azure ^^) resources. We'll come back to this a little bit later

We have the bare minimum in the `provider.tf`, because, as expected, the file is based on our authentication method which is, in this case, az cli:

```bash

provider "azurerm" {
  features {
  }
}

```

Same for the `terraform.tf` file which selected automatically a provider version:

```bash

terraform {
  backend "local" {}
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.56.0"
    }
  }
}

```

We can also notice the `terraform.tfstate` file which is the locally saved state.

Last, but not least, we have 2 additionals files with which we are less familiar.
`aztfexportSkippedResources.txt` contains all the Azure object that could not be matched to azurerm provider object

```js

Following resources are marked to be skipped:

- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/aks.teknews.cloud/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/lab.teknews.cloud/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/lab.postgres.database.azure.com/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.eastus.azmk8s.io/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westeurope.azmk8s.io/SOA/@
- /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io/SOA/@


```

`aztfexportResourceMapping.json`, as the name implies, shows the mapping between the discovered resources (exported from Azure) and the terraform resources (imported).

```json

{
	"/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns": {
		"resource_id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns",
		"resource_type": "azurerm_resource_group",
		"resource_name": "res-0"
	},
	"/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnszones/aks.teknews.cloud": {
		"resource_id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/aks.teknews.cloud",
		"resource_type": "azurerm_dns_zone",
		"resource_name": "res-1"
	},
=============================================truncated=============================================
	"/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io/A/aks4": {
		"resource_id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.westus.azmk8s.io/A/aks4",
		"resource_type": "azurerm_private_dns_a_record",
		"resource_name": "res-22"
	}
}

```

So the resources are now available in terraform, which means that we should be able to run a terraform plan, and potentially, add or remove resources. Let's try that.

```bash

yumemaru@azure:~/Documents/myrepo/terrafy/rg-dns$ terraform init

Initializing the backend...

Successfully configured the backend "local"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "3.46.0"...
- Installing hashicorp/azurerm v3.46.0...
- Installed hashicorp/azurerm v3.46.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

```

And then a plan

```bash

df@df2204lts:~/Documents/myrepo/terrafy/rg-dns$ terraform plan

============================truncated==========================
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

```

As expected. 

Because there are old cname, I can now remove this from the configuration, by commenting or removing the configuration

```h

/*
resource "azurerm_dns_cname_record" "res-7" {
  name                = "guestbook"
  record              = "pubip-agwagicmeetup2.westeurope.cloudapp.azure.com"
  resource_group_name = "rg-dns"
  ttl                 = 1800
  zone_name           = "lab.teknews.cloud"
  depends_on = [
    azurerm_dns_zone.res-6,
  ]
}
resource "azurerm_dns_cname_record" "res-8" {
  name                = "votingapp"
  record              = "pubip-agwagicmeetup2.westeurope.cloudapp.azure.com"
  resource_group_name = "rg-dns"
  ttl                 = 900
  zone_name           = "lab.teknews.cloud"
  depends_on = [
    azurerm_dns_zone.res-6,
  ]
}
*/

```

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # azurerm_dns_cname_record.res-7 will be destroyed
  # (because azurerm_dns_cname_record.res-7 is not in configuration)
  - resource "azurerm_dns_cname_record" "res-7" {
      - fqdn                = "guestbook.lab.teknews.cloud." -> null
      - id                  = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud/CNAME/guestbook" -> null
      - name                = "guestbook" -> null
      - record              = "pubip-agwagicmeetup2.westeurope.cloudapp.azure.com" -> null
      - resource_group_name = "rg-dns" -> null
      - tags                = {} -> null
      - ttl                 = 1800 -> null
      - zone_name           = "lab.teknews.cloud" -> null

      - timeouts {}
    }

  # azurerm_dns_cname_record.res-8 will be destroyed
  # (because azurerm_dns_cname_record.res-8 is not in configuration)
  - resource "azurerm_dns_cname_record" "res-8" {
      - fqdn                = "votingapp.lab.teknews.cloud." -> null
      - id                  = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-dns/providers/Microsoft.Network/dnsZones/lab.teknews.cloud/CNAME/votingapp" -> null
      - name                = "votingapp" -> null
      - record              = "pubip-agwagicmeetup2.westeurope.cloudapp.azure.com" -> null
      - resource_group_name = "rg-dns" -> null
      - tags                = {} -> null
      - ttl                 = 900 -> null
      - zone_name           = "lab.teknews.cloud" -> null

      - timeouts {}
    }

Plan: 0 to add, 0 to change, 2 to destroy.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────



```

Let's look a little more to the configuration from `main.tf`.


```bash

resource "azurerm_resource_group" "res-0" {
  location = "westeurope"
  name     = "rg-dns"
}
resource "azurerm_dns_zone" "res-1" {
  name                = "aks.teknews.cloud"
  resource_group_name = "rg-dns"
  depends_on = [
    azurerm_resource_group.res-0,
  ]
}
resource "azurerm_dns_cname_record" "res-2" {
  name                = "guestbook"
  record              = "pubip-agw-1.westeurope.cloudapp.azure.com"
  resource_group_name = "rg-dns"
  ttl                 = 1800
  zone_name           = "aks.teknews.cloud"
  depends_on = [
    azurerm_dns_zone.res-1,
  ]
}
resource "azurerm_dns_cname_record" "res-3" {
  name                = "votingapp"
  record              = "pubip-agw-1.westeurope.cloudapp.azure.com"
  resource_group_name = "rg-dns"
  ttl                 = 900
  zone_name           = "aks.teknews.cloud"
  depends_on = [
    azurerm_dns_zone.res-1,
  ]
}

```

As expected, only hard-coded values for the resources are included.

Let's conclude this overview now.

## Conclusion

First, if we can consider that the resources are now managed through terraform, on the other hand, the code should be refactored. Because hardcoded value is not something that should be kept, specifically in a version control system.

Also, as mentioned on the [documentation](https://pkg.go.dev/github.com/Azure/aztfexport#readme-non-goal), there's no guarantee that the generated code can be used as a template to reproduce the same set of resource.

Third, the generated state is local and should be migrated to an appropriate backend.

And that's all for this topic. Very soon I'll try the terraform native import available in release 1.5.0.

See you ^^

