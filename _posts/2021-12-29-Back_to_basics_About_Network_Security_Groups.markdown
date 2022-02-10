---
layout: post
title:  "Back to basics - About Network Security Groups"
date:   2021-12-29 17:28:00 +0200
categories: AKS
---

Hi!

I recently discussed with some co-workers of mine and I realized that Network Security Group remains a confusing topic.

How is it working, when does it filter traffic, is it enough in a hub and spoke topology?

Many questions that are not that complex but still require to take some time to answer to.

This article aims to fill those gaps...
  
## Table of content

1. The Network Security Group 101
2. Recommandations for using NSG
3. Scenario of Network flows with NSG
4. What you should remember

## 1. The Network Security Group 101  

In itself, the network security group, or NSG, is a distributed statefull firewall.

What it does:

- An NSG filters traffic on objects living in the Virtual Network only, specifically, on the Network Interface card
- An NSG can be apply for filtering management, on either a VM or a subnet
- An NSG works with NSG rules, in both direction, ingress and egress
- An NSG comes with default rules that cannot be removed  
  
| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 65000 | AllowVnetInBound | Inbound | Allow | * |  * |  * |  VirtualNetwork |  VirtualNetwork |
| 65001 | AllowAzureLoadBalancerInBound | Inbound | Allow | * |  * |  * |  AzureLoadBalancer |  * |
| 65500 | DenyAllInBound | Inbound | Deny | * |  * |  * |  * |  * |
| 65000 | AllowVnetOutBound | Outbound | Allow | * |  * |  * |  VirtualNetwork |  VirtualNetwork |
| 65001 | AllowInternetOutBound | Outbound | Allow | * |  * |  * |  * |  Internet |
| 65500 | DenyAllOutBound | Outbound | Deny | * |  * |  * |  * |  * |
  
- NSG rules have a priority defined by a priority number, **between 100 and 4096**. THe lower the number, the higher the priority
- Evaluation of the NSG rules follows the schema below:  
  
![Illustration 1](/assets/aboutnsg001.png)
  
- An NSG rules can allow or deny traffic
- NSG rules can be enriched with [Service tags](https://docs.microsoft.com/en-us/azure/virtual-network/service-tags-overview) and [Application Security Groups](https://docs.microsoft.com/en-us/azure/virtual-network/application-security-groups)

Before moving forward, a word on the Service Tags and the Application Security Groups:

- Service tags are, as the name imply, tags used to identify known IP addresses range, such as **Internet**, **AzureCloud**, **LoadBalancer**, **VirtualNetwork**...

  By using those tags, it is possible to avoid IP addresses range in rules.
  
  The IP ranges behind the tags are updated on Microsoft side

- Application Security Groups (ASG), similarly to Service tags, are custom tags, user managed and associated to VMs.

  Through the use of ASG, it becomes possible to target VMs by tags instead of IP

Ok, that's a lot, and there's more... So let's put some basis with a few recommandations

## 2. Recommandations for using NSG

A few recommandations, to ensure correct filtering and limit brain damage in the NSG rules evaluation:

- Apply NSG only at the subnet level. And use either Application Security Group or service tags to define your rules.
- Consider the filtering for each object in the VNet
- Focus first on Ingress rules. Egress rules can be limited to add filtering to destination out of Azure. As long as you filter traffic inside a VNet, you should be able to have the same result with another NSG
- Define a standard in the rules priority. For example:

- Define default rules. For example:

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 4000 | DefaultDenyAllToOnPremise | Inbound | Deny | * |  * |  * |  * |  **On-Premise Ranges** |
| 3000 | DefaultAllowAlltoAzureCloud | Outbound | Allow | * |  * |  * |  * |  AzureCloud |
| 4000 | DefaultDenyAllToOnPremise | Outbound | Deny | * |  * |  * |  * |  **On-Premise Ranges** |
| 4010 | DefaultDenyAllInternet | Outbound | Deny | * |  * |  * |  * |  Internet |
  
## 3. Scenario of Network flows with NSG  



## 4. What you should remember


