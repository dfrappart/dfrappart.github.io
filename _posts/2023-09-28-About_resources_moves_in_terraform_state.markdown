---
layout: post
title:  "About resources move in terraform state"
date:   2023-09-28 18:00:00 +0200
year: 2023
categories: Azure Terraform
---

Hi!

In previous articles, we explored different ways to import existing (Azure) resources in the state, and how to generate the terraform configuration associated.
While quite useful, we came to the conclusion that the need to modify the configuration generated.

In this article, we'll look at just that and detail different scenarios and methods to manage resources.

Our agenda will be as below: 

- Terraform resource moves basics
- When Terraform is smart enough to understand moves
- Using a declarative approach for moves
- Using the terraform cli for moves

And then we'll conclude ^^

## 1. Terraform resource moves basics

Before looking at how to move resources, let's redefine some basics.

When a resource is defined in an hcl configuration, it's something like this: 

```bash

resource "azurerm_resource_group" "RgDemo" {}

```
Then we can find it in the state after the resource is created, something like this: 

```bash

yumemaru@azure:~/Documents/myrepo/inframgmt$ terraform state list

azurerm_resource_group.RgDemo

```

Until now, nothing new or complicated.

However, changing the name `RgDemo` to something else, like `OtherRg`, well terraform will understand it as a new resource, try to destroy the resource `azurerm_resource_group.RgDemo` and create the resource `azurerm_resource_group.OtherRg`.
Let's just say that it may cause probleme for many reasons.

If we want the move to occur correctly, meaning without destruction of resources, then we have to to it differently.
There is a way by manipulating the state, through terraform cli. We'll see that in details in the last part of the article. 
There is also another way more on the declarative side, with terraform move block, which are very similar to import blocks that we discuss in a [previous article](https://blog.teknews.cloud/azure/terraform/2023/07/02/NativeImporttf.html).
Hoever, sometime, terraform can detect and understand on his own that the resource just move. And we'll start with that.

Note that we do not mention the way of managing the state directly from the file, which is something that we should NEVER do! That being said, let's move on.

## 2. When terraform is smart enough

To illustrate this scenario, we'll take the case where we have only 1 resource group in our configuration:

```bash

resource "azurerm_resource_group" "RGMonitor" {
  location   = var.AzureRegion
  managed_by = null
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

