---
layout: post
title:  "What if I want to see Network Logs on Azure?"
date:   2024-02-05 18:00:00 +0200
year: 2024
categories: Azure Network Security
---


Hello everyone!

in this article, I propose a journey in the Azure Network observability.
After choosing what to do in terms of Azure Network topology, at one point, we have to cnsider operations, which means we need to see which flows go through which firewalls.
As most people using Azure probably know, those firewalls can take different forms and the associated visibility will depends of the firewall nature.
So, we will start by a rapid review of the possiblility for the Network filtering in Azure and  we will have a look at the different logs (because visibility is almost always related to logs) available for the Azure Network service through different scenarios.

Let's get started!
  
## 1. Review of Azure Network filtering options

Depending on the objective, we may use differents solutions for Network filtering in Azure:
- locally to a virtual network, we usually rely on Network Security Groups. If you're familiar with this service, you probably know that it's perfect to secure flows between workload inside Azure subnet in a granular way, as long as the requirement is on the layer 4.
- In a Hub & Spoke scenario, with virtual Network or Virtual WAN and Virtual Hub, we rely on a central Firewall that can be used to filter interspoke traffic, egress traffic to Internet, and in some scenario (but with additional services) Internet exposure. This Hub Firewall can be an Azure Firewall, or a 3rd party appliance. However, We'll focus on Azure Firewall here, because we are interested in Azure Native observability options

I mentionned additional services for Internet exposure, including WAF related solutions, but we will focus mainly on NSGs and Azure Firewall. IMHO, WAF is a full topic on its own so I'd prefer addressing that in a dedicated article.

Last, we also have some Security control available with Azure Network Manager, but, as for WAF, I think it deserves its own article ^^.

Ok let's get to the heart of the topic.

## 2. Getting visibility for flows going through NSGs

NSGs are L4 statefull Firewall, as we said already, perfect for VNet local filtering. We can use Network watcher to evaluate the flows but it does not show information on real packet going through the NSG.
For that we need to rely on logs, which are **NOT** configured by default.

As for most of Aure managed services, we have [Azure Resources Logs](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs) available for NSG, that we can set in the Diagnostic Settings section of the resource

![illustration1](/assets/fwobs/nsgdiagsettings.png)

As displayed on the picture, there are 2 available categories: 
- `Network Security Group Event`
- `Network Security Group Rule Counter`

To be able to look at those logs, we need to send the logs in a sink that allows querying. There are many options, but the native one (and unfortunately quite expensive also) is to use a log analytics workspace. It quite easy to do it from the portal, and doing it through terraform would look like this:

```go

resource "azurerm_monitor_diagnostic_setting" "NsgDiagSettings" {
  for_each                   = local.Subnets
  name                       = local.Subnets[each.key].Nsg.DiagSettingsName #format("%s-%s", "diag", azurerm_network_security_group.Nsgs[each.key].name)
  storage_account_id         = local.StaLogId
  log_analytics_workspace_id = local.LawLogId
  target_resource_id         = azurerm_network_security_group.Nsgs[each.key].id

  dynamic "enabled_log" {
    for_each = data.azurerm_monitor_diagnostic_categories.Nsg[each.key].log_category_types
    content {
      category = enabled_log.value

    }
  }

  dynamic "metric" {
    for_each = data.azurerm_monitor_diagnostic_categories.Nsg[each.key].metrics
    content {
      category = metric.value

    }
  }
}

```

Notice the reference to the data source for the log categories, which is instrumented this way, and allows us to automatically get all the log categories. On the other hand, it implies a re-evaluation at each plan / apply.

```go

data "azurerm_monitor_diagnostic_categories" "Nsg" {

  for_each    = local.Subnets
  resource_id = azurerm_network_security_group.Nsgs[each.key].id
}

```

## 3. Getting visibility for flows going through an Azure Firewall

## 4. Wrapping it up

- 