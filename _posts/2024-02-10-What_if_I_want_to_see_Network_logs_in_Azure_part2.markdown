---
layout: post
title:  "What if I want to see Network Logs on Azure? - Part2"
date:   2024-02-15 18:00:00 +0200
year: 2024
categories: Azure Network Security
---


Hello everyone!

This is part 2 of our journey to Network observability on Azure.
In [part1](/_posts/2024-02-10-What_if_I_want_to_see_Network_logs_in_Azure_part1.markdown), we review some basics and start playing with NSG diagnostics logs.
While giving us some informations, it was far from enough to get specific information, mainly due to the log structure, which will refers the source or destination IP depending on the rule condition.
Fortunately, we do have flow logs to get much more visibility.

Let's get started!
  
## 1. Azure Network Watcher, Traffic Analytics and flow logs

As the title suggest, Flow logs are a feature of [Azure Network watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview) and are actually the foundations behind [Traffic analytic](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics). 

Traffic Analytics provides, as the name implies, some basics analytics on the Network flows directly in the portal.

Before anything, we would need to configure flow logs on NSGs. As opposite to diagnostic settings, which are related resources to the NSG, a flow log is created as an independant resource, related to Azure Network Watcher.

![Illustration 1](/assets/fwobs/flowlog01.png)

![Illustration 2](/assets/fwobs/flowlog02.png)

Creating flow log resources can be done through the portal, or through some terraform configuration as displayed below.

```go

resource "azurerm_network_watcher_flow_log" "Flowlogs" {

  for_each = local.Subnets

  network_watcher_name      = local.NetworkWatcherName
  name                      = local.Subnets[each.key].Nsg.FlowLogName
  location                  = var.Location
  resource_group_name       = local.NetworkWatcherRGName
  network_security_group_id = azurerm_network_security_group.Nsgs[each.key].id
  storage_account_id        = local.StaLogId
  enabled                   = true
  version                   = 2

  retention_policy {
    enabled = true
    days    = 365
  }

  traffic_analytics {
    enabled               = var.IsTrafficAnalyticsEnabled #true
    workspace_id          = data.azurerm_log_analytics_workspace.LawLog.workspace_id
    workspace_region      = data.azurerm_log_analytics_workspace.LawLog.location
    workspace_resource_id = data.azurerm_log_analytics_workspace.LawLog.id
    interval_in_minutes   = 10
  }
}

```

It can also be configured through an [Azure Policy](https://www.azadvertizer.net/azpolicyadvertizer/e920df7f-9a64-4066-9b58-52684c02a091.html), which allow to remediate automatically NSGs without flow logs.

It's quite easy to get interesting informations directly from the portal, once flow logs are configured with a Log Analytics Workspace.

![Illustration 3](/assets/fwobs/ta01.png)

![Illustration 4](/assets/fwobs/ta02.png)

![Illustration 5](/assets/fwobs/ta03.png)

![Illustration 6](/assets/fwobs/ta04.png)

![Illustration 7](/assets/fwobs/ta05.png)

![Illustration 8](/assets/fwobs/ta06.png)

Someone searching a little should find at one point a kusto query behind GUI, as below.

![Illustration 9](/assets/fwobs/ta07.png)

that one is a bit more advanced than what we've seen up now, so let's start from the beginning.

## 2. Querying flow logs

For flow logs, the log category that we are interested in is `AzureNetworkAnalytics_CL`, and in all the examples, we'll also specify `SubType_s == 'FlowLog'`

A basic query would then be like this:

```bash

AzureNetworkAnalytics_CL
| where SubType_s == 'FlowLog'
| where TimeGenerated >= ago(24h)

```
![Illustration 10](/assets/fwobs/flowlog03.png)

If we expand one of the result, we can see taht we have much more details thant with the diagnostic logs.

![Illustration 10](/assets/fwobs/flowlog04.png)

![Illustration 10](/assets/fwobs/flowlog05.png)

![Illustration 10](/assets/fwobs/flowlog06.png)


The query below can be used to get a summary of the flow logs:

```bash

AzureNetworkAnalytics_CL
| where SubType_s == 'FlowLog'
| project DestIP_s, SrcIP_s, SrcPublicIPs_s, NIC_s, VM_s, Subnet_s,DestPort_d, L4Protocol_s, L7Protocol_s, FlowStatus_s, FlowDirection_s,FlowType_s,NSGList_s, NSGRules_s, NSGRuleType_s, FlowStartTime_t, FlowEndTime_t

```

![Illustration 11](./Img/kql056.png)

## 3. What about some dashboard?



## 4. Wrapping it up

- 