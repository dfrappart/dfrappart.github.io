---
layout: post
title:  "About resources moves in terraform state"
date:   2023-09-28 18:00:00 +0200
year: 2023
categories: Azure Terraform
---

Hi!

In previous articles, we explored different ways to import existing (Azure) resources in the state, and how to generate the terraform configuration associated.
While quite useful, we came to the conclusion that we needed to modify the configuration generated.

In this article, we'll look at just that and detail different scenarios and methods to manage resources.

Our agenda will be as below: 

- Terraform resource moves basics
- When Terraform is smart enough to understand moves
- Using a declarative approach for moves
- Using the terraform cli for moves

And then we'll conclude ^^

## 1. Terraform resources moves basics

Before looking at how to move resources, let's redefine some basics.

When a resource is defined in an hcl configuration, it's something like this: 

```bash

resource "azurerm_resource_group" "RgDemo" {}

```
Then we can find it in the state after the resource is created, something like this: 

```bash

yumemaru@azure:~$ terraform state list

azurerm_resource_group.RgDemo

```

Until now, nothing new or complicated.

However, changing the name `RgDemo` to something else, like `OtherRg`, well terraform will understand it as a new resource, try to destroy the resource `azurerm_resource_group.RgDemo` and create the resource `azurerm_resource_group.OtherRg`.
Let's just say that it may cause probleme for many reasons.

If we want the move to occur correctly, meaning without destruction of resources, then we have to do it differently.
There is a way, by manipulating the state, through terraform cli. We'll see that in details in the last part of the article. 
There is also another way, more on the declarative side, with terraform moved block, which are very similar to import blocks that we discuss in a [previous article](https://blog.teknews.cloud/azure/terraform/2023/07/02/NativeImporttf.html).
However, sometimes, terraform can detect and understand on his own that the resource just move. And we'll start with that.

Note that we do not mention the way of managing the state directly from the file, which is something that we should NEVER do! That being said, let's move on.

## 2. When terraform is smart enough

To illustrate this scenario, we'll take the case where we have only 1 resource group in our configuration:

```go

resource "azurerm_resource_group" "RGMonitor" {
  location   = var.AzureRegion

  name       = "rsg-monitor"
  tags       = {}
}

```

We may want to change the way the resource is declared, and include it in a configuration with `count`, for example, because we know that we will have more than 1 rg to manipulate in the configuration.

```bash


Terraform will perform the following actions:

  # azurerm_resource_group.RGMonitor has moved to azurerm_resource_group.RGMonitor[0]
    resource "azurerm_resource_group" "RGMonitor" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-monitor"
        name     = "rsg-monitor"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────

```

We can note that terraform mention a move.

Remember that a resource in Terraform changes when we have a count (or for_each) statement. Instead of being a simple object, it becomes a list of previous object.
In this present case, terraform is able to move automatically a single resource to a list of one resource. 

However, that's the only case that I know of, where it's that simple. For more complex refactoring, we'll need another way. One of which is the moved block.

## 3. The moved block, a declarative approach to move resources

This time, we will go a little further. Let's take a scenario in which we have differents resources groups.
Those resources groups, for a reason or another, are configured as repeated resources blocks:

```go

resource "azurerm_resource_group" "RGSecurity" {
  location   = var.AzureRegion

  name       = "rsg-security"
  tags       = {}
}

resource "azurerm_resource_group" "RGKeyVault" {
  location   = "westeurope"

  name       = "rg-kv"
  tags       = {}
}

resource "azurerm_resource_group" "RgDns" {
  name     = "rg-dns"
  location = var.AzureRegion
}

```

We still have the previous configuration for the resource group and its variable list: 

```go

variable "RgList" {
    type = list(string)
    description = "The list of Resource Groups"
    default = ["rsg-monitor"]
}


resource "azurerm_resource_group" "RGMonitor" {
  count      = length(var.RgList)
  location   = var.AzureRegion

  name       = var.RgList[count.index]
  tags       = {}
}

```

We would like to include in the resource block using the count the other resources groups.
For that we can count on the [moved block](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring).

Let's start with one resource group first.

We'll define the moved block as below: 

```go

moved {
  from = azurerm_resource_group.RGSecurity
  to   = azurerm_resource_group.RGMonitor[1]
}

```

And change the `RGList` variable:

```go

variable "RgList" {
    type = list(string)
    description = "The list of Resource Groups"
    default = ["rsg-monitor","rsg-security"]
}

```

If we try a plan now, we'll get the following error:

```bash

yumemaru@azure:~$ terraform plan
╷
│ Error: Moved object still exists
│ 
│   on rg.tf line 35:
│   35: moved {
│ 
│ This statement declares a move from azurerm_resource_group.RGSecurity, but that resource instance is still declared at rg.tf:16,1.
│ 
│ Change your configuration so that this instance will be declared as azurerm_resource_group.RGMonitor[1] instead.
╵

```

It makes sense, the resource is declared twice, so we can comment it and try again:

```bash

yumemaru@azure:~$ terraform plan
╷
│ Error: Reference to undeclared resource
│ 
│   on security.tf line 13, in resource "azurerm_log_analytics_workspace" "LawSecurity":
│   13:   location                           = azurerm_resource_group.RGSecurity.location
│ 
│ A managed resource "azurerm_resource_group" "RGSecurity" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on security.tf line 16, in resource "azurerm_log_analytics_workspace" "LawSecurity":
│   16:   resource_group_name                = azurerm_resource_group.RGSecurity.name
│ 
│ A managed resource "azurerm_resource_group" "RGSecurity" has not been declared in the root module.
╵

```

Again, it makes sense, we may have dependencies that need to be adressed. Let's fix this:

```bash

yumemaru@azure:~$ terraform plan

Terraform will perform the following actions:

  # azurerm_resource_group.RGSecurity has moved to azurerm_resource_group.RGMonitor[1]
    resource "azurerm_resource_group" "RGMonitor" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-security"
        name     = "rsg-security"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

And now we have what we wanted. With the moved block, we just move the resource from one place to another in the state, and we can even keep track of the change in the configuration, similarly to the import block, so everything seems nice. Let's apply those changes.

Now, if we refer at the documentation, we can read that the moved block can also be used to rename resources in the terraform configuration (meaning not in the cloud provider, where the name is usually synonym of new resource).
An attentive reader will have noticed that the resource group block is called `RGMonitor` but is supposed to contain a list of our resource groups. So it does not work like that. We should rename it. 
Let's use another moved block: 

```go

moved {
  from = azurerm_resource_group.RGMonitor
  to   = azurerm_resource_group.RG
}

```

We will also change the current resource group block to reflect the new target:

```go

resource "azurerm_resource_group" "RG" {
  count      = length(var.RgList)
  location   = var.AzureRegion

  name       = var.RgList[count.index]
  tags       = {}
}

```

And let's run a plan: 

```bash

yumemaru@azure:~$ terraform plan
╷
│ Error: Reference to undeclared resource
│ 
│   on monitor.tf line 12, in resource "azurerm_log_analytics_workspace" "LawMonitor":
│   12:   location                           = azurerm_resource_group.RGMonitor[0].location
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on monitor.tf line 15, in resource "azurerm_log_analytics_workspace" "LawMonitor":
│   15:   resource_group_name                = azurerm_resource_group.RGMonitor[0].name
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on monitor.tf line 31, in resource "azurerm_log_analytics_workspace" "LawMonitor2":
│   31:   location                           = azurerm_resource_group.RGMonitor[0].location
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on monitor.tf line 34, in resource "azurerm_log_analytics_workspace" "LawMonitor2":
│   34:   resource_group_name                = azurerm_resource_group.RGMonitor[0].name
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on security.tf line 13, in resource "azurerm_log_analytics_workspace" "LawSecurity":
│   13:   location                           = azurerm_resource_group.RGMonitor[1].location
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.
╵
╷
│ Error: Reference to undeclared resource
│ 
│   on security.tf line 16, in resource "azurerm_log_analytics_workspace" "LawSecurity":
│   16:   resource_group_name                = azurerm_resource_group.RGMonitor[1].name
│ 
│ A managed resource "azurerm_resource_group" "RGMonitor" has not been declared in the root module.

```

We have some error related to dependencies, but we can fix that easily and re-run the plan:

```bash

yumemaru@azure:~$ terraform plan

Terraform will perform the following actions:

  # azurerm_resource_group.RGMonitor[0] has moved to azurerm_resource_group.RG[0]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-monitor"
        name     = "rsg-monitor"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

  # azurerm_resource_group.RGMonitor[1] has moved to azurerm_resource_group.RG[1]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-security"
        name     = "rsg-security"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

If error similar to this remains, re-check the configuration to be sure the resource is not declared elswhere.

```bash

yumemaru@azure:~$ terraform plan
╷
│ Error: Moved object still exists
│ 
│   on rg.tf line 44:
│   44: moved {
│ 
│ This statement declares a move from azurerm_resource_group.RGSecurity, but that resource instance is still declared.
│ 
│ Change your configuration so that this instance will be declared as azurerm_resource_group.RGMonitor[1] instead.
╵
╷
│ Error: Moved object still exists
│ 
│   on rg.tf line 50:
│   50: moved {
│ 
│ This statement declares a move from azurerm_resource_group.RGMonitor, but that resource is still declared.
│ 
│ Change your configuration so that this resource will be declared as azurerm_resource_group.RG instead.

```

In my own case, I got this error due to an existing import block that was still refering to the moved resources. I corrected this with a comment:

```go

/*
import {
  to = azurerm_resource_group.RGMonitor
  id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-monitor"
}
*/

/*
import {
  to = azurerm_resource_group.RGSecurity
  id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-security"
}
*/

```

At this point we can check the state with the command `terraform state list` and `terraform state show`:

```bash

yumemaru@azure:~$ terraform state list

azurerm_resource_group.RG[0]
azurerm_resource_group.RG[1]

yumemaru@azure:~$ terraform state show azurerm_resource_group.RG[0]

# azurerm_resource_group.RG[0]:
resource "azurerm_resource_group" "RG" {
    id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-monitor"
    location = "eastus"
    name     = "rsg-monitor"
    tags     = {}
}
yumemaru@azure:~$ terraform state show azurerm_resource_group.RG[1]

# azurerm_resource_group.RG[1]:
resource "azurerm_resource_group" "RG" {
    id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-security"
    location = "eastus"
    name     = "rsg-security"
    tags     = {}
}

```

Now let's say I want to move the remaining resource groups:

```go

resource "azurerm_resource_group" "RGKeyVault" {
  location   = "westeurope"

  name       = "rg-kv"
  tags       = {}
}

resource "azurerm_resource_group" "RgDns" {
  name     = "rg-dns"
  location = var.AzureRegion
}

```

The only trouble that I'll get is not related to the moved block, but to the way I defined my configuration. 

Indeed, with the count, I only iterate the same configuration, but all parameters should be the sames (except the name).
But with the resource group `RGKeyVault`, I come accross an issue because it's in another region.
I need to refactor my configuration to be able to set more than one parameter if needed. 
Which mean I would be better serve with a for_each rather than a count.

Let's try this.

First we add a new variable for the resource groups configuration:

```go

variable "RgConfig" {
  type = map(object({
    RgLocation               = string
    

  }))

  default = {
    "rsg-monitor" = {
      RgLocation = "eastus"
    }
    "rsg-security" = {
      RgLocation = "eastus"      
    }
  }
}

```

Then the moved block, for each instance: 

```go

moved {
  from = azurerm_resource_group.RG[0]
  to   = azurerm_resource_group.RG["rsg-monitor"]
}

moved {
  from = azurerm_resource_group.RG[1]
  to   = azurerm_resource_group.RG["rsg-security"]
}

```

And the configuration change to the one below:

```go

resource "azurerm_resource_group" "RG" {
  for_each = var.RgConfig
  
  location   = each.value.RgLocation

  name       = each.key
}

```
Instead of a count, I have now a for_each and I'm able to specify 
There are potential dependencies to modify, which, once solved, allow us to get the below plan:

```bash

yumemaru@azure:~$ terraform plan
Terraform will perform the following actions:

  # azurerm_resource_group.RG[0] has moved to azurerm_resource_group.RG["rsg-monitor"]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-monitor"
        name     = "rsg-monitor"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

  # azurerm_resource_group.RG[1] has moved to azurerm_resource_group.RG["rsg-security"]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-security"
        name     = "rsg-security"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.

```

ANd now, I'm able to add the remaining resource group through my configuration variable:

```go

variable "RgConfig" {
  type = map(object({
    RgLocation               = string
    

  }))

  default = {
    "rsg-monitor" = {
      RgLocation = "eastus"
    }
    "rsg-security" = {
      RgLocation = "eastus"      
    }
    "rg-kv" = {
      RgLocation = "westeurope"
    }
    "rg-dns" = {
      RgLocation = "eastus"      
    }
  }
}

```

And the following moved blocks:

```go

moved {
  from = azurerm_resource_group.RGKeyVault
  to   = azurerm_resource_group.RG["rg-kv"]
}

moved {
  from = azurerm_resource_group.RgDns
  to   = azurerm_resource_group.RG["rg-dns"]
}

```

```bash

yumemaru@azure:~$ terraform plan

Terraform will perform the following actions:

  # azurerm_resource_group.RgDns has moved to azurerm_resource_group.RG["rg-dns"]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/16e85b36-5c9d-48cc-a45d-c672a4393c36/resourceGroups/rg-dns"
        name     = "rg-dns"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

  # azurerm_resource_group.RGKeyVault has moved to azurerm_resource_group.RG["rg-kv"]
    resource "azurerm_resource_group" "RG" {
        id       = "/subscriptions/16e85b36-5c9d-48cc-a45d-c672a4393c36/resourceGroups/rg-kv"
        name     = "rg-kv"
        tags     = {}
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.

```

And with that we have seen a lot about moved block. We could probably do other things but I would prefer to explore the last scenario, with terraform cli.

## 4. Using Terraform cli for resources moves

The moved block can cover many refactoring scenario, as we've seen earlier. It also has the benefit to allow keeping a visual track of the move in the configuration which is good.
However, in some cases, it may be difficult to perform all the refactoring through only those blocks. Fortunately, there are command in the terraform cli that are useful.
I used 2 of those previously:

- `terraform state list` which list all the content of the state
- `terraform state show` which show a specific terraform item in the state.

Notice that I used item, rather than resource. That's because, we may find either real resources, or data, or resource created through module.
Anyway, let's take the follwoing scenario to illustrate our purpose. This is extracted from a [previous talk](https://github.com/dfrappart/tfstatemanip/tree/main/Terraformconfig/02_Move) that I made a few years back. 

In this scenario, we manage an Azure Database for MySQL server, with database in a single module

![illustration1](/assets/move/move001.png)

![illustrationé](/assets/move/move002.png)

![illustration2](/assets/move/move003.png)

The objective here is typivally to delegate and grant the managemnt of the database to someone else. This someone else will use a new module for the databases, and propose update through git commit. So we have a simple module like that:


```go

# MySQL databases
resource "azurerm_mysql_database" "MySQLDB" {
  count                                       = length(var.MySQLDbList)
  name                                        = "${element(var.MySQLDbList,count.index)}" 
  resource_group_name                         = var.RGName
  server_name                                 = Var.MySQLServerName
  charset                                     = var.MySQLDbCharset
  collation                                   = var.MySQLDbCollation
}

```

```go

module "MySQLDBs" {

  #Module Location
  source                                  = "../../Modules/MySQLDB"

  #Module variable     
  MySQLDbList                             = var.MySQLDbList
  RGName                                  = module.ResourceGroup.RGName
  MySQLServerName                         = module.MySQL.ServerName



}

```

If we add the new module to the configuration and run a plan we get something like this:

```bash

yumemaru@azure:~$ terraform plan
=================================Truncated=================================
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.MySQLDBs.azurerm_mysql_database.MySQLDB[0] will be created
  + resource "azurerm_mysql_database" "MySQLDB" {
      + charset             = "latin2"
      + collation           = "latin2_general_ci"
      + id                  = (known after apply)
      + name                = "db1"
      + resource_group_name = "rsgcloudouest2"
      + server_name         = "msqlcloudouest2"
    }

  # module.MySQLDBs.azurerm_mysql_database.MySQLDB[1] will be created
  + resource "azurerm_mysql_database" "MySQLDB" {
      + charset             = "latin2"
      + collation           = "latin2_general_ci"
      + id                  = (known after apply)
      + name                = "db2"
      + resource_group_name = "rsgcloudouest2"
      + server_name         = "msqlcloudouest2"
    }

  # module.MySQLDBs.azurerm_mysql_database.MySQLDB[2] will be created
  + resource "azurerm_mysql_database" "MySQLDB" {
      + charset             = "latin2"
      + collation           = "latin2_general_ci"
      + id                  = (known after apply)
      + name                = "db3"
      + resource_group_name = "rsgcloudouest2"
      + server_name         = "msqlcloudouest2"
    }

Plan: 3 to add, 0 to change, 0 to destroy.

```

Which is what we would like to have, but wait, the database already exist right ? And are in the state. LEt's check this with the `terraform state list` command:


```bash

yumemaru@azure:~$ terraform state list
=================================Truncated=================================
module.MySQL.azurerm_mysql_database.MySQLDB[0]
module.MySQL.azurerm_mysql_database.MySQLDB[1]
module.MySQL.azurerm_mysql_database.MySQLDB[2]
module.MySQL.azurerm_mysql_server.MySQLServer
=================================Truncated=================================

```

And get more details with `terraform state show`:

```bash

yumemaru@azure:~$ terraform state show module.MySQL.azurerm_mysql_database.MySQLDB[0]
# module.MySQL.azurerm_mysql_database.MySQLDB[0]:
resource "azurerm_mysql_database" "MySQLDB" {
    charset             = "latin2"
    collation           = "latin2_general_ci"
    id                  = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgcloudouest2/providers/Microsoft.DBforMySQL/servers/msqlcloudouest2/databases/mysql-dbcloudouest2-db1"
    name                = "mysql-dbcloudouest2-db1"
    resource_group_name = "rsgcloudouest2"
    server_name         = "msqlcloudouest2"
}

```

This one is the same that we had in the plan so we will get an error if applied this way. That's a perfect use case for the `terraform state mv` command:

```bash


yumemaru@azure:~$ terraform state mv -dry-run module.MySQL.azurerm_mysql_database.MySQLDB[0] module.MySQLDBs.azurerm_mysql_database.MySQLDB[0]
Would move "module.MySQL.azurerm_mysql_database.MySQLDB[0]" to "module.MySQLDBs.azurerm_mysql_database.MySQLDB[0]"


```

After oving the resources (without the `-dry-run`), we should get a proper plan where no resources are added or destroyed. Before that there is most certainly some tweaking to do, such as variables to change, or parameters in the module to modify.
But that's all.

Now, just to be clear, while before moving resources between module was not supported, last versions of terraform allow such moves with block similar to this

```go

moved {
  from = module.MySQL.azurerm_mysql_database.MySQLDB[0]
  to   = module.MySQLDBs.azurerm_mysql_database.MySQLDB[0]
}

```

The last command that should be know is `terraform state rm` which allow to remove an existing resource from the state.
I will not detail the scenario here, but imagine that the resource would be managed in a totally different terraform lifecycle. So we don't need to destroy it but we don't want to manage it anymore. 
A couple of `terraform state rm` and correctly placed `/*` `*/` would allow this. Then the existing resources would have to be imported with any method, but preferably the import block.

Ok time to wrap this up

## 5. Before living

So we reviewed what possibilities we had to managed ressource moves and thus allow either configuration refactoring, or resources lifecycle changes.
Hashicorp has a clear objective which can be summarize as `everything as code`. Meaning that the move are also written in the configuration, as for the import. There are still imperative way to do it but since they diverge from the declarative approach proned by terraform, it just seem logical to see always new features for the declarative approach.

That's all for today. I hope you had fun reading this.

See you!