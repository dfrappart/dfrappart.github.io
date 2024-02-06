---
layout: post
title:  "What if I want to see Network Logs on Azure?"
date:   2024-02-20 18:00:00 +0200
year: 2024
categories: Azure Network Security
---


Hello everyone!

This is part 2 of my serie on getting visibility on Azure Network logs.
In part 1, we focused on the NSGs and we saw a bunch of interesting things.
Now for this second part, I would like to discuss about visibility on Azure Firewall.
Indeed, the most frequent Network topology being Hub & Spokes, We usually find a central Firewall in the Hub. 
That this firewall is the only point of filtering or is an additional layer for the NSGs is clearly out of the scope of this article.
We'll focus on how to get visibility on the associated logs, the same way we did with NSG.

The agenda: 

- The Azure Firewall and its log categories
- Parsing logs of the Firewall
- What if I don't want to do kql every single morning ?

Let's get started!
  
## 1. The Azure Firewall and its log categories



## 2. Parsing logs of the Firewall

```bash

AzureDiagnostics
| where TimeGenerated >= ago(24h)
| where ResourceId contains "/SUBSCRIPTIONS/16E85B36-5C9D-48CC-A45D-C672A4393C36/RESOURCEGROUPS/RG-VWAN-LAB-001/PROVIDERS/MICROSOFT.NETWORK/AZUREFIREWALLS/AFW-VHUB-VWAN-LAB-001"
| summarize count() by Category

```


```bash

AzureDiagnostics
| where TimeGenerated >= ago(24h)
| where ResourceId contains "/SUBSCRIPTIONS/16E85B36-5C9D-48CC-A45D-C672A4393C36/RESOURCEGROUPS/RG-VWAN-LAB-001/PROVIDERS/MICROSOFT.NETWORK/AZUREFIREWALLS/AFW-VHUB-VWAN-LAB-001"
| where Category == "AZFWNetworkRule" //"AzureFirewallNetworkRule"


```


```bash

AzureDiagnostics
| where TimeGenerated >= ago(24h)
| where ResourceId contains "/SUBSCRIPTIONS/16E85B36-5C9D-48CC-A45D-C672A4393C36/RESOURCEGROUPS/RG-VWAN-LAB-001/PROVIDERS/MICROSOFT.NETWORK/AZUREFIREWALLS/AFW-VHUB-VWAN-LAB-001"
| where Category == "AZFWNetworkRule" //"AzureFirewallNetworkRule"
| summarize count() by SourceIP, DestinationIp_s, Action_s, DestinationPort_d, RuleCollection_s, Rule_s, ActionReason_s, Protocol_s

```

## 4. Wrapping it up

- 

## 3. What if I don't want to do kql every single morning ?

