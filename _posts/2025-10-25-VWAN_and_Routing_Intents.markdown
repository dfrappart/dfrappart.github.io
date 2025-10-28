---
layout: post
title:  "VWAN and routing intent"
date:   2025-10-25 18:00:00 +0200
year: 2025
categories: Network
---

Hello people!

In this article, to change a bit from the kubernetes stuff, I propose to have a look at VWAN and the routing intent feature.
It should not be a too long article, which will be a change from the last ones &#129325;.

Our Agenda will be:


1. A review of VWAN and some routing stuffs
2. Be more dynamic with routing intent

Let's get started!


## 1. A review of VWAN and some routing stuffs

### 1.1. Hub(s) & spokes with Vwan

So, before anything, we probably need a refresh.

Vwan is Microsoft managed service for... WAN. No surprise here.
There are many things to check when planning for the use of Vwan, but here we want to focus on its managed aspect, and the capabilities it provides in the area of Network backbone, specifically the hub & spoke topology.

One of the main feature of vwan is related to its virtual hub, a child resource that act as a Virtual Network that would be a hub in a hub & spokes topology.

Except that the routing management is centralized in the Vwan/Vhub, instead of distributed to each VirtualNetwork/Subnets.

![illustration1](/assets/vwan/hubandspoke.png)

![illustration2](/assets/routingintent/vwanbasics.png)

The fact that the routing can be centrally managed is thanks to the virtual router that is hiding inside our Vhub, but because it's managed, we don't need to know more &#129323;. 

For those who noticed the 2 Vhubs on the schema, that is also another interesting point of vwan and vhubs, plural.

To have multi-region network topology, we just need to add a vhub, in another region (or not, it works also for same region multi-hubs), and we automatically have network connectivity between the hubs.

Because each hubs have its own routers, there is route propagation between the hubs, and the spokes that are connected to each hub, without much trouble.

there is one importnat point to remember, which is that because the virtual hub is a managed hub, we cannot deploy whatever we want inside. We'll see more on that later.

Now, what about the infamous Secure hub?

### 1.2. Adding hub firewall in vhub

Usually, one of the objective of the hub & spokes with vnets only is to provide an center point of firewall management, by adding an NVA, Azure Firewall or anything else, inside the hub. In this case we have a wide choice of NVA to deploy as IaaS market place offers, or as SaaS offers integrated to Vnet, such as the [Palo Alto NGFW](https://docs.paloaltonetworks.com/cloud-ngfw-azure/deployment/cloud-ngfw-for-azure-deployment-architectures).

Also, because of the non-transitivity of the peering, we need an NVA in the hub to act as a next hop for the routing between spokes. It is usually  A firewall that takes this role, or in some case, a virtual network gateway.

In a Vhub, because of the integrated router, there is no need for additional configuration when connecting spokes to the hub. Keeping the default network connection configuration will automatically propagate routes between connected spokes, so that there is a path between each workloads insides differents Vnet connected.

So we add a firewall only for the purpose of security, hence the **secure hub** name.

Adding the firewall is quite easy, except that we have less choice than with the Vnet hub.
Having the firewall becoming the center point of traffic between our spokes however, is less easy.

This time there is a need for routing configuration. 

Navigating in the vhub, we can find 2 route tables.

![illustration3](/assets/securehub/routeconfig.png)

- The default route table is... the default, and is the one that is used for propagating required routes for spoke connectivity.
- The none route table is a route table that is used to... not propagate routes &#129299;.

At this point, even if there is a firewall in the vhub, no route redirect traffic through it.

One way to achieve this goal is to add a custom route which define the Firewall IP as the next hop for all RFC 1918 ranges.

![illustration4](/assets/securehub/routeconfigsecure.png)

![illustration5](/assets/securehub/routingvmdifferentspokes.png)

Then we just need to specify the custom route table in the network connection configuration when we connect a spoke. We also need to not propagate the range of the spoke in the route table, so we choose the propagate to the none route table. It may feel strange at first but well.

For more details, there is quite a lot of documentation available on Internet, and I wrote a [walkthrough for Secure hub](https://blog.teknews.cloud/network/security/terraform/2024/01/19/Walkthrough_Secure_Hub_in_Virtual_WAN.html) some times ago that may still be of help.

With this we manage more than one route table, but we do have a Secure hub.

Other scenarios can be considered, such as specifying public spokes and private spokes. This way, with 2 route tables, we can manage the network connection depending of the spokes nature.

![illustration6](/assets/securehub/routingdifferenttable.png)

In this last scenario, adding static route on the default route table is required to provide a route for the yellow spoke to any blue spokes.

So we can have a secure hub, have centrally managed routes and, sometimes get lost in all this routing configuration &#128517;

## 2. Be more dynamic with routing intent

Before diving into the nice routing intent feature, we'll look at a specific scenario where the multiple route tables scenario does not work.

### 2.1. Multiple hub, multiple firewall, and connectivity issue between spokes 

Let's consider the following scenario:

We are in a multi-region topology and, because Vwan is made for this, we want to leverage multiple virtual hub in different regions.
Because we need to secure the East-West flow, meaning the flows between spokes, we want ot have firewalls in both virtual hubs.

![illustration7](/assets/routingintent/vwanmultiplefw.png)

Capitalizing on the discussed matter, we know that we can manage routing configuration for spokes with a custom route table stating that all traffic should go to the local hub firewall to which a spoke is connected.

Under the condition that all spokes have a vnet range in the RFC 1918 range, we can assume that a vm in a blue spoke can find a route for a vm in another blue spoke. This statement is also true for yellow vms in differents yellow spokes (not represented on the schema, but you get the idea).

What about the case where a blue vm try to reach a yellow vm?

For the purpose of experimenting, we create a VWAN with 2 secure hubs and a custom route table for each vhub.
The route tables are configured to set the firewall as the next hop for the private IP range, and for Internet Egress.

```json

df@df-2404lts:~/$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 | jq .[2]
{
  "associatedConnections": [],
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubRouteTables/rt-lab-vwan-lab-001-vhub-lab-001",
  "labels": [
    "privatezone-vwan-lab-001-vhub-lab-001"
  ],
  "name": "rt-lab-vwan-lab-001-vhub-lab-001",
  "propagatingConnections": [],
  "provisioningState": "Succeeded",
  "resourceGroup": "rg-lab-vwan-001",
  "routes": [
    {
      "destinationType": "CIDR",
      "destinations": [
        "0.0.0.0/0"
      ],
      "name": "InternetToFw",
      "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001",
      "nextHopType": "ResourceId"
    },
    {
      "destinationType": "CIDR",
      "destinations": [
        "10.0.0.0/8",
        "192.168.0.0/16",
        "172.16.0.0/12"
      ],
      "name": "PrivateCIDRToFW",
      "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001",
      "nextHopType": "ResourceId"
    }
  ],
  "type": "Microsoft.Network/virtualHubs/hubRouteTables"
}

```

Also we have the following spokes with the routing configuration detailed in the below table.

| Spoke | range | Connected to vhub | Network connection associated to route table | Network connection propagate to route table | static route added on default route |
|-|-|-|-|
| spoke1 | 172.22.0.0/24 | vwan-lab-001-vhub-lab-001 | defaultRouteTable in vhub1 | defaultRouteTable in vhub1 | no |
| spoke2 | 172.22.1.0/24 | vwan-lab-001-vhub-lab-001 | rt-lab-vwan-lab-001-vhub-lab-001 | noneRouteTable | yes |
| spoke3 | 172.22.2.0/24 | vwan-lab-001-vhub-lab-001 | rt-lab-vwan-lab-001-vhub-lab-001 | noneRouteTable | yes
| spoke4 | 172.22.3.0/24 | vwan-lab-001-vhub-lab-002 | defaultRouteTable in vhub2 | defaultRouteTable in vhub2 | no |
| spoke5 | 172.22.4.0/24 | vwan-lab-001-vhub-lab-002 | rt-lab-vwan-lab-001-vhub-lab-002 | noneRouteTable | yes | 
| spoke6 | 172.22.5.0/24 | vwan-lab-001-vhub-lab-002 | rt-lab-vwan-lab-001-vhub-lab-002 | noneRouteTable | yes |

The portal may be slow to render the vhub network connections and the routing configurattion, so It may be better to rely on the cli to get information.

We can see that the default route table is associated and propagate the spoke1. 
We can also see that the noneRouteTable is used to propagate the connections that are associated to the custom route table.
With this configuration, we have the firewall in vhub 1 between spoke1, spoke2 and spoke3.
We have the same configuration respectively for spoke 4, spoke 5 and spoke 6 in vhub2.

However, spoke1 and spoke4, because of their association to the default route table, are not secured with a firewall in between.

```bash

df@df-2404lts:~/$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 -o table
Name                              ProvisioningState    ResourceGroup
--------------------------------  -------------------  ---------------
defaultRouteTable                 Succeeded            rg-lab-vwan-001
noneRouteTable                    Succeeded            rg-lab-vwan-001
rt-lab-vwan-lab-001-vhub-lab-001  Succeeded            rg-lab-vwan-001

df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[0].propagatingConnections
[
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke1-to-vwan-lab-001-vhub-lab-001"
]
df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[0].associatedConnections
[
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke1-to-vwan-lab-001-vhub-lab-001"
]

df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[1].propagatingConnections
[
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke3-to-vwan-lab-001-vhub-lab-001",
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke2-to-vwan-lab-001-vhub-lab-001"
]
df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[1].associatedConnections
[]

df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[2].associatedConnections
[
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke3-to-vwan-lab-001-vhub-lab-001",
  "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubVirtualNetworkConnections/peer-vnet-sbx-spoke2-to-vwan-lab-001-vhub-lab-001"
]
df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[2].propagatingConnections
[]

```

So now connection on vm1 in spoke1, we should be able to gain access to vm2 and vm3.

```bash

```

Same for vm1 with vm4.

```bash

```

Connectivity is ok between vm2 and vm3.

```bash

```


Because it's similar to vm1 with vm2 and vm3, we have no issue between vm4 and vm5 and vm6

```bash

```

But problems appear when we try to reach vm5 or vm6 from vm1, vm2 or vm3, so typically any vm connected to vhu1 trying to reach vm connected to vhub2.

```bash

```

Which is a shame because it did work without a firewall.

so what can we try?

### 2.2. Playing with static routes

The first idea that come to mind would be to rely on static routes. That's what we did on the default route table so that vm1 could get access to vm2 and vm3.

But first let's have a look at network watcher and the next hop tool.

**network watcher**

Between vms connected to the same hub, we get the firewall as the next hop, which is what we set with the custom route table or by adding a static route on the default route table.

So maybe we could try adding a static route with the neighbour firewall as a next hop?
Let's try this.
Note that we need a more specific route than the ones already defined on the custom route table.

**addingstaticroutenexthopneightbourfw**

Well, it seems that we are stuck.
But no, we have the routing intent option.

### 2.3 Using routing intent for cross secure vhub traffic

Finally, We're here.

It was kind of a long intro just to come to this point, but It was, IMHO, necessary 	&#128517;

Ok first let's have a look at the routing intent.



Ok time to wrap up!

## 3. Summary

Soooooo!

Another eventful journey right?


To summarize:

Application Gateway works... supposedly.

Currently there are some isue with the chart which does not propagate the client Id of the managed identity where it should.
A not so easy issue to tackle, as mentionned already.

Once the ALB controller is deployed and working, depending on the underlying network, we may have more trouble.

First, Azure CNI Overlay is fine, as long as the deployment of the AGC is in the same Vnet range than the Cluster. Otherwise, we encounter errors, hinting that the multi-vnet deployment is not functionnal yet.

Second, the managed deployment is not totally working. For this, I'm not totaly clear on what's not ok and what is. Supposedly, a CIDR of `/24` fopr the ALB subnet. But we saw that we can get error with a /24 or a /23. So maybe it's just the managed deployment which is not working fantastically. Or maybe it's more related to the Azure CNI Overlay. I did not push it to find out what may not work with an Azure CNI Pod Subnet (I mean for the managed deployment).

Also, I did not try with an Azure CNI Powered by Cilium. I kind of though it may have been too risky &#129325;

Anyway, I hope it was intersting for some of you.

I'll be back soon for more AGC I think, and definitly more Gateway API.

Have fun ^^







