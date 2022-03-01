---
layout: post
title:  "Back to basics - About Network Security Groups"
date:   2022-03-01 22:45:00 +0200
categories: Network
---

Hi!

I recently discussed with some co-workers and I realized that Network Security Group remains a confusing topic.

How is it working, when does it filter traffic, is it enough in a hub and spoke topology?

Many questions that are not that complex but still require to take some time to answer to.

This article aims to help fill those gaps.
  
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

- Service tags are, as the name implies, tags used to identify known IP addresses ranges, such as **Internet**, **AzureCloud**, **LoadBalancer**, **VirtualNetwork**...

  By using those tags, it is possible to avoid IP addresses ranges in rules.
  
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
  
## 3. Scenarios of Network flows with NSG  
  
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

### 3.2. Managing Network flows between VMs in in peered Virtual Network

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
  
| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 1010 | AllowWebServerToBackEndServer | Inbound | Allow | TCP | * | 1433 |  WebServer | BackendServer |
| 3010 | DenyAllToBackendServer | Inbound | Deny | * | * | * | * | BackendServer |
  
Prior to rule 65000, rule 3010 here would match VirtualNetwork traffic, because `*` includes it.

So the VM is not allowed access in this specific case.

How could we proceed to allow access?

An attentive reader would probably propose to use an ASG. But, because there is a but, ASGs work only for the local virtual network.

It means that it is not possible to reference virtual machines as source / target with ASGs when the VMs are in different VNet.
The error message is pretty explicit: 

```json

"statusMessage": "{\"error\":
                    {
                        \"code\":\"AllApplicationSecurityGroupsInSecurityRuleMustBeFromSameVnet\",
                        \"message\":\"Security Rule /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rghub/providers/Microsoft.Network/networkSecurityGroups/nsg-spoke-subnet1/securityRules/DenyallfromVMHubToWebServer has Application Security Groups (ASGs) from multiple virtual networks (/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/RGHUB/providers/Microsoft.Network/virtualNetworks/VNETHUB, /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/RGHUB/providers/Microsoft.Network/virtualNetworks/VNETSPOKE). All ASGs on the Security Rule must be from the same virtual network.\",
                        \"details\":[]
                    }
                  }"

``` 

So then, alowing traffic for the VM from the Hub Network can only be done through IP based rule. That's a shame.

We would need to implemen something like this: 

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 1010 | AllowWebServerToBackEndServer | Inbound | Allow | TCP | * | 1433 |  WebServer | BackendServer |
| 1020 | AllowHubServerToBackEndServer | Inbound | Allow | TCP | * | 1433 |  **IP_Of_HubVM** | BackendServer |
| 3010 | DenyAllToBackendServer | Inbound | Deny | * | * | * | * | BackendServer |

Ok it's time for our last scenario.
  
### 3.3. Managing Network flows in a Hub & Spokes topology

A usual network topology in Azure is the Hub & Spoke.

In this approach, similarly to the last scenario, we have a Hub Virtual Network which is peered to all the different Spokes.

Since this is a model relying on Virtual Network peering, what we discussed in the previous section is valid.
However, there's more.
The peering, by design, is not transitive. Which mean that Spoke-to-spoke traffic is not routed. So in this case, there is no real need to define rules on NSG right?

Well, there is. Because, with something playing the role of a router, we can define a route from one Spoke to another through the Hub.

This article is about NSG, so we won't go into detail in routing scenarios in Hub & Spoke. However, it means that we have the same scenario as previously which summarize in 2 points: 

- Virtual Network Service Tag in a Spoke is extended by default with the range of the direct peer, in this case the Hub Virtual Network
- ASG are not usable to filter traffic originating from the Hub, or another Spoke

Let's illustrate: 

![Schema 6](/assets/aboutnsgschema006.png)

Considering VM1 in the left Spoke, the default peering configuration and rule 65000 allow traffic from VM2 to VM1

| priority | name | direction | access | protocol | source Port Range(s) | destination Port Range(s) | source Address Prefix(es) | destination Address Prefix(es) |
|-|-|-|-|-|-|-|-|-|
| 65000 | AllowVnetInBound | Inbound | Allow | * | * | * |  VirtualNetwork |  VirtualNetwork |
  
And that's the same for VM3 regarding traffic coming from VM2.

For traffic coming from VM3 to VM1 (and vice versa), there is no rule that is allowing traffic, because the VirtualNetwork Service Tag only extend to get the range from the Hub Network.
Also, there is **no route** from Spoke to Spoke. So even if we add a rule on one of the spoke NSG, it won't be routed without adding something (that we won't talk in this article ^^).

Now, if we wanted to add granular rule to specifically allow or deny VM2 to VM1 (or VM3), we **cannot** use an ASG to identify the source from VM2 to the NSG in the spoke VM, because **ASG can only be used for rules in the VNet in which the ASG is applying to**.

Ok, not easy, but hopefully detailed enough.

Last, if we take the hypothesis of using a NSG in the Hub Network to filter traffic from VM3 to VM2, **It will not work**, because the NSG filter traffic on 

what it is applied to. So, in this case, the NSG would be applied in a subnet  in the Hub which does not apply for spoke-to-spoke traffic. 

And also there's no route in our case anyway ^^  
  
Below schema summarize everything we just detailled: 
  
  
![Schema 7](/assets/aboutnsgschema007.png)
  
That's it for now so let's wrap up.
## 4. What you should remember

So, in a few sentences:
  
NSG can manage network flows on a distributed approach, by applying an NSG to a subnet preferably, or a VM if really needed.

NSG is easier to manage with rules leveraging ASGs and Service Tags

NSG is ideal for Intra-Virtual Network filtering, but less ideal for cross Virtual Networ filtering, because ASGs won't work for cross VNet NSG rules

Also, peering has an impact on the VirtualNetwork Service Tag, and thus on default NSGrules, except if you are aware of it and configure your peering with custom parameters.

Without additional routing, spoke-to-spoke communication in Hub & Spoke is not possible, but not limited by NSG, which can be used for filtering, even if we need to rely on IP based rules in this case.

Ok, that's all.
I hope this help for the use of NSG.
There's probably room for other network topics such as routing or waf, or even other firewall solutions, but well not here so next time ^^.

