---
layout: post
title:  "Back to basics - About Network Security Groups"
date:   2021-12-29 17:28:00 +0200
categories: AKS
---

Hi!

I recently discussed with some co-workers and I realized that Network Security Group remains a confusing topic.

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
- An NSG can be applied on either a VM or a subnet
- An NSG works with NSG rules, in both directions, ingress and egress
- An NSG comes with default rules that cannot be removed. Those rules are displayed below.  
  
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

- Service tags are, as the name implies, tags used to identify known IP addresses range, such as **Internet**, **AzureCloud**, **LoadBalancer**, **VirtualNetwork**...

  By using those tags, it is possible to avoid IP addresses range in rules.
  
  The IP ranges behind the tags are updated on Microsoft side

- Application Security Groups (ASG), similarly to Service tags, are custom tags, user managed and associated to VMs.

  Through the use of ASG, it becomes possible to target VMs by tags instead of IP

Ok, that's a lot, and there's more... So let's put some basis with a few recommandations

## 2. Recommandations for using NSG

A few recommandations, to ensure correct filtering and limit brain damage in the NSG rules evaluation:

- Read the Azure doc about the services you are using, to be sure that you do not miss any requirements
- Apply NSG only at the subnet level. And use either Application Security Group or service tags to define your rules.
- Consider the filtering for each object in the VNet
- Focus first on Ingress rules. Egress rules can be limited to add filtering to destination out of Azure. As long as you filter traffic inside a VNet, you should be able to have the same result with another NSG and Ingress rules only.
- Define a standard in the rules priority. For example:
  
| Priority range | Rule type | Description |
|-|-|-|
| 1000 - 1500 | Project specific allow rule | **Allow** rules that should be defined with project team, at design stage, and that may override default infrastructure rules|
| 1500 - 2000 | Project specific deny rule | **Deny** rules that should be defined with project team, at design stage |
| 2000 - 3000 | Infrastructure default allow| **Allow** rules defined to ensure that Azure environment and services are working correctly |
| 3000 - 4000 | Infrastructure default deny| **Deny** rules defined to provide default network flows configuration and comply with Security policies |

Note that some Azure services, such as Azure Bastion, or Application Gateway, which live in dedicated subnets, require specific setof rules. Those rules should be included in the default design, so that those can be configured through an IaC tool and versionned with a Git repo. It could be summarized by the motto **Read the Azure Doc**.
  
- Define default rules. For example:

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 4000 | DefaultDenyAllToOnPremise | Inbound | Deny | * |  * |  * |  * |  **On-Premise Ranges** |
| 3000 | DefaultAllowAlltoAzureCloud | Outbound | Allow | * |  * |  * |  * |  AzureCloud |
| 4000 | DefaultDenyAllToOnPremise | Outbound | Deny | * |  * |  * |  * |  **On-Premise Ranges** |
| 4010 | DefaultDenyAllInternet | Outbound | Deny | * |  * |  * |  * |  Internet |

- Read the Azure doc about the services you are using, to be sure that you do not miss any requirements
- Last but not least, **read the Azure doc**
  
## 3. Scenario of Network flows with NSG  
  
Ok, now that we put some ground rules and that everyone has read the docs, what about a few examples?
  
### 3.1. Managing Network flows for 1 VM in an isolated Virtual Network

This one is easy, but, well, we do need to start with something.

So we have 1 VM, in a subnet in a VNet.
By default, we should have an NSG and since, i follow my own recommandation, it is applied at the Subnet level

![Schema 1](/assets/aboutnsgschema001.png)

At first, we have only the following rules, because those are the default NSG rules that come with **all** NSG:

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 65000 | AllowVnetInBound | Inbound | Allow | * |  * |  * |  VirtualNetwork |  VirtualNetwork |
| 65001 | AllowAzureLoadBalancerInBound | Inbound | Allow | * |  * |  * |  AzureLoadBalancer |  * |
| 65500 | DenyAllInBound | Inbound | Deny | * |  * |  * |  * |  * |
| 65000 | AllowVnetOutBound | Outbound | Allow | * |  * |  * |  VirtualNetwork |  VirtualNetwork |
| 65001 | AllowInternetOutBound | Outbound | Allow | * |  * |  * |  * |  Internet |
| 65500 | DenyAllOutBound | Outbound | Deny | * |  * |  * |  * |  * |

So, in this case, our little VM is indeed isolated, because it's alone in the VNet, and we did not specify any rule.

If we follow the evaluation logic:  

- There is nothing inside the VNet except the VM, so nothing that match the rule 65000
- There is no Azure Load Balancer at this point either, so no match for rule 65001
- There are indeed things outside the VNet, which is the Internet, and the rule 65000 match all those cases, thus preventing access to this VM.

For the egress traffic, if we still follow the same logic, we get this:  

- Rule 65000 would allow the VM, because it match the service tag VirtualNetwork, to reach something corresponding to the service tag VirtualNetwork. But it's a lonley VM, so nothing math here
- Rule 65001 is where it start to get interesting. It states that **"*"**, which means anything this NSG is applied to, so the VM is part of it, can go to the Internet. So the VM is able to reach, let's say [github](https://github.com)
- Last, rule 65500 is here to block all remaining traffic. But we already matched a lot of traffic with the previous one...

Ok, that was easy enough, let's move on and add some traffic.

Let's say that the VM is a web server, so it should be reachable from the Internet. What we need to add then is an ingress rule that would allow Internet to reach the VM.

But, remember, we want to avoid managing IPs, because it's a pain, so we add an application security group and attach it to the VM, and also, it means that i follow my own advice which is kind of the point right?

 Let's call this ASG `WebServer`.
Also, it's kind of obvious, but the VM needs a public IP before being reachable.

![Schema 2](/assets/aboutnsgschema002.png)

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 1010 | AllowWebServerInbound | Inbound | Allow | TCP |  * | 443 | Internet |  WebServer |
  
Still not too hard. Let's add new things

### 3.2. Managing Network flows for 2 VM in an isolated Virtual Network

This time, we have an additional VM in the same VNet.
To get started, let's take the hypothesis that the new VM is in another subnet.

![Schema 3](/assets/aboutnsgschema003.png)

Let's put an NSG on the subnet where this VM lives.
Without adding anything, this NSG will come with the default rules.

So, is the new VM reachable from the Internet?

Following the logic of evalluation, no rule on this NSG match traffic coming from Internet, so that's a no.

Is the new VM reachable from the VM we had at first?

Again, following the logic of evaluation, we can the initial VM, which lives in the Virtual Network, match default ingress rule 65000

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 65000 | AllowVnetInBound | Inbound | Allow | * | * | * |  VirtualNetwork |  VirtualNetwork |
  
It could be ok, but we would like to be a little more selective right?

So let's do this:

- First, let's add a new ASG on the new VM, something like BackendServer
- Second, let's add a new rule which allows specifically the ASG WebServer to reach the ASG BackendServer on, let's say, TCP 1433
- Last, to ensure that only this port is available, we could add another rule that deny everything on ASG BackendServer

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 1010 | AllowWebServerToBackEndServer | Inbound | Allow | TCP | * | 1433 |  WebServer | BackendServer |
| 3010 | DenyAllToBackendServer | Inbound | Deny | * | * | * | * | BackendServer |
  
![Schema 4](/assets/aboutnsgschema004.png)
  
Still not too hard right?

Note that we did not add an egress rule to limit traffic between the `WebServer` VM and the `BackendServer` VM.  It's really not necessary at this point.

Ok, let's dig further then...

### 3.2. Managing Network flows for 2 VM in an Virtual Network peered to another Virtual Network

So now we add another VM, but in another Virtual Network.

If we consider jsut our 2 isolated Virtual Network like that, well it is the first section scenario:

The WebServer is accessible from the Internet. The new VM in the new VNet can reach it through it's Internet access.

Not really interesting.

Now if we add the peering concept, we can allow communication between the 2 virtual networks.

![Schema 5](/assets/aboutnsgschema005.png)

There are options available for the Peering that must be taken into account.

First, it is possible to configure the traffic flows between the peered VNet.

By default, the traffic is allowed between the peered VNet.
  
![Illustration 2](/assets/aboutnsg002.png)  
  
![Illustration 3](/assets/aboutnsg003.png)  
  
![Illustration 4](/assets/aboutnsg004.png)  
  
It has also another impact, which is to change the value of the `VirtualNetwork` Service tag
  
Remember, VNet peering is connecting 2 VNets, so it works both ways, which means that we have this configuration on both VNets.

![Illustration 5](/assets/aboutnsg005.png)  
  
![Illustration 6](/assets/aboutnsg006.png)  
  
So back to our scenario. By default, the VM in the peered VNet, that we call `vnethub` here, will be able match rules that include the `VirtualNetwork` service tags.

Considering the `WebServer`, it means that the rule 65000 will allow access

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 65000 | AllowVnetInBound | Inbound | Allow | * | * | * |  VirtualNetwork |  VirtualNetwork |

Considering the `BackendServer`, well, it's a little different, because we added another rule that match also its address so we have this: 



### 3.3. Managing Network flows for 2 VM in a Spoke Virtual Network


### 3.4. Scenarios that do not work with NSG

## 4. What you should remember


