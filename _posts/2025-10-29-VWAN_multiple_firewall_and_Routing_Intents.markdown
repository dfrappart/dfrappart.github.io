---
layout: post
title:  "VWAN, multiple firewalls, and routing intent"
date:   2025-10-29 18:00:00 +0200
year: 2025
categories: Network
---

Hello people!

In this article, to change a bit from the kubernetes stuff, I propose to have a look at VWAN and the routing intent feature.

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

There is one importnat point to remember, which is that because the virtual hub is a managed hub, we cannot deploy whatever we want inside.

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

### 2.1. Multiple hubs, multiple firewalls, and connectivity issue between spokes 

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
|-|-|-|-|-|-|
| spoke1 | 172.22.0.0/24 | vwan-lab-001-vhub-lab-001 | defaultRouteTable in vhub1 | defaultRouteTable in vhub1 | no |
| spoke2 | 172.22.1.0/24 | vwan-lab-001-vhub-lab-001 | rt-lab-vwan-lab-001-vhub-lab-001 | noneRouteTable | yes |
| spoke3 | 172.22.2.0/24 | vwan-lab-001-vhub-lab-001 | rt-lab-vwan-lab-001-vhub-lab-001 | noneRouteTable | yes
| spoke4 | 172.22.3.0/24 | vwan-lab-001-vhub-lab-002 | defaultRouteTable in vhub2 | defaultRouteTable in vhub2 | no |
| spoke5 | 172.22.4.0/24 | vwan-lab-001-vhub-lab-002 | rt-lab-vwan-lab-001-vhub-lab-002 | noneRouteTable | yes | 
| spoke6 | 172.22.5.0/24 | vwan-lab-001-vhub-lab-002 | rt-lab-vwan-lab-001-vhub-lab-002 | noneRouteTable | yes |

And the following vms.

| Vm | Spoke | Ip | Connected to vhub |
|-|-|-|-|
| vm1 | spoke1 | 172.22.0.68 | vwan-lab-001-vhub-lab-001 |
| vm2 | spoke2 | 172.22.1.4 | vwan-lab-001-vhub-lab-001 |
| vm3 | spoke3 | 172.22.2.4 | vwan-lab-001-vhub-lab-001 |
| vm4 | spoke4 | 172.22.3.4 | vwan-lab-001-vhub-lab-002 |
| vm5 | spoke5 | 172.22.4.4 | vwan-lab-001-vhub-lab-002 |
| vm6 | spoke6 | 172.22.5.4 | vwan-lab-001-vhub-lab-002 |

The portal may be slow to render the vhub network connections and the routing configuration, so It may be better to rely on the cli to get information.

We can see that the default route table is associated and propagate the spoke1. 
We can also see that the noneRouteTable is used to propagate the connections that are associated to the custom route table.
With this configuration, we have the firewall in vhub 1 between spoke1, spoke2 and spoke3.
We have the same configuration respectively for spoke 4, spoke 5 and spoke 6 in vhub2.

However, spoke1 and spoke4, because of their association to the default route table, are not secured with a firewall in between.

```bash

df@df-2404lts:~$ az network vhub route-table list --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 -o table
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

vmadmin@spoke1vm:~$ ping -c 4 172.22.1.4
PING 172.22.1.4 (172.22.1.4) 56(84) bytes of data.
64 bytes from 172.22.1.4: icmp_seq=1 ttl=63 time=23.0 ms
64 bytes from 172.22.1.4: icmp_seq=2 ttl=63 time=4.55 ms
64 bytes from 172.22.1.4: icmp_seq=3 ttl=63 time=4.58 ms
64 bytes from 172.22.1.4: icmp_seq=4 ttl=63 time=4.39 ms

--- 172.22.1.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 4.386/9.140/23.043/8.027 ms
vmadmin@spoke1vm:~$ ping -c 4 172.22.2.4
PING 172.22.2.4 (172.22.2.4) 56(84) bytes of data.
64 bytes from 172.22.2.4: icmp_seq=1 ttl=63 time=5.36 ms
64 bytes from 172.22.2.4: icmp_seq=2 ttl=63 time=4.39 ms
64 bytes from 172.22.2.4: icmp_seq=3 ttl=63 time=3.72 ms
64 bytes from 172.22.2.4: icmp_seq=4 ttl=63 time=18.1 ms

--- 172.22.2.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 3.722/7.886/18.074/5.910 ms



```

Same for vm1 with vm4.

```bash
vmadmin@spoke1vm:~$ ping -c 4 172.22.3.4
PING 172.22.3.4 (172.22.3.4) 56(84) bytes of data.
64 bytes from 172.22.3.4: icmp_seq=1 ttl=62 time=5.04 ms
64 bytes from 172.22.3.4: icmp_seq=2 ttl=62 time=2.53 ms
64 bytes from 172.22.3.4: icmp_seq=3 ttl=62 time=2.65 ms
64 bytes from 172.22.3.4: icmp_seq=4 ttl=62 time=2.29 ms

--- 172.22.3.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 2.290/3.125/5.041/1.113 ms


```

Connectivity is ok between vm2 and vm3.

```bash

vmadmin@spoke2vm:~$ ping -c 4 172.22.2.4
PING 172.22.2.4 (172.22.2.4) 56(84) bytes of data.
64 bytes from 172.22.2.4: icmp_seq=1 ttl=63 time=5.46 ms
64 bytes from 172.22.2.4: icmp_seq=2 ttl=63 time=4.65 ms
64 bytes from 172.22.2.4: icmp_seq=3 ttl=63 time=4.71 ms
64 bytes from 172.22.2.4: icmp_seq=4 ttl=63 time=4.64 ms

```


Because it's similar to vm1 with vm2 and vm3, we have no issue between vm4 and vm5 and vm6

```bash

vmadmin@spoke4vm:~$ ping -c 4 172.22.4.4
PING 172.22.4.4 (172.22.4.4) 56(84) bytes of data.
64 bytes from 172.22.4.4: icmp_seq=1 ttl=63 time=6.50 ms
64 bytes from 172.22.4.4: icmp_seq=2 ttl=63 time=4.79 ms
64 bytes from 172.22.4.4: icmp_seq=3 ttl=63 time=4.56 ms
64 bytes from 172.22.4.4: icmp_seq=4 ttl=63 time=4.73 ms

--- 172.22.4.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 4.559/5.143/6.496/0.785 ms
vmadmin@spoke4vm:~$ ping -c 4 172.22.5.4
PING 172.22.5.4 (172.22.5.4) 56(84) bytes of data.
64 bytes from 172.22.5.4: icmp_seq=1 ttl=63 time=5.59 ms
64 bytes from 172.22.5.4: icmp_seq=2 ttl=63 time=3.49 ms
64 bytes from 172.22.5.4: icmp_seq=3 ttl=63 time=4.27 ms
64 bytes from 172.22.5.4: icmp_seq=4 ttl=63 time=11.0 ms

--- 172.22.5.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 3.485/6.093/11.028/2.946 ms

```

But problems appear when we try to reach vm5 or vm6 from vm1, vm2 or vm3, so typically any vm connected to vhu1 trying to reach vm connected to vhub2.

```bash

vmadmin@spoke2vm:~$ ping -c 4 172.22.3.4
PING 172.22.3.4 (172.22.3.4) 56(84) bytes of data.

--- 172.22.3.4 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3102ms

vmadmin@spoke2vm:~$ ping -c 4 172.22.4.4
PING 172.22.4.4 (172.22.4.4) 56(84) bytes of data.

--- 172.22.4.4 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3095ms

```

Which is a shame because it did work without a firewall.

The table summarize the possible vm connectivities

| Connect with| vm1 (hub1, no fw) | vm2 (hub1 with fW1) | vm3 (hub1 with fW1) | vm4 (hub2, no fw) | vm5 (hub2 with fW2)| vm6 (hub2 with fW2)|
|-|-|-|-|-|-|-|
| **vm1** | &#9640; | &#9989; | &#9989;| &#9989;| &#10060; | &#10060; |
| **vm2** | &#9989;| &#9640;| &#9989;| &#10060;| &#10060; | &#10060; |
| **vm3** | &#9989;| &#9989;| &#9640;| &#10060;| &#10060; | &#10060; |
| **vm4** | &#9989; | &#10060; | &#10060; | &#9640;| &#9989; | &#9989;|
| **vm5** | &#10060; | &#10060; | &#10060; | &#9989;| &#9640; | &#9989;|
| **vm6** | &#10060; | &#10060; | &#10060; | &#9989;| &#9989; | &#9640;|

so what can we try?

### 2.2. Playing with static routes

The first idea that come to mind would be to rely on static routes. That's what we did on the default route table so that vm1 could get access to vm2 and vm3. 

Static routes are added to tell the default route table to route spoke behind the firewall to the ip of this firewall. We can note that it's not done with the ip of the firewall but rather its reosurce id.

```bash

df@df-2404lts:~$ az network vhub route-table show -n defaultRouteTable --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 | jq .routes
[
  {
    "destinationType": "CIDR",
    "destinations": [
      "172.22.2.0/24"
    ],
    "name": "route-to-spoke-spoke3",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001",
    "nextHopType": "ResourceId"
  },
  {
    "destinationType": "CIDR",
    "destinations": [
      "172.22.1.0/24"
    ],
    "name": "route-to-spoke-spoke2",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001",
    "nextHopType": "ResourceId"
  }
]

```

We have the same on the default route table for vhub2.

```bash

df@df-2404lts:~$ az network vhub route-table show -n defaultRouteTable --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 | jq .routes
[
  {
    "destinationType": "CIDR",
    "destinations": [
      "172.22.5.0/24"
    ],
    "name": "route-to-spoke-spoke6",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002",
    "nextHopType": "ResourceId"
  },
  {
    "destinationType": "CIDR",
    "destinations": [
      "172.22.4.0/24"
    ],
    "name": "route-to-spoke-spoke5",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002",
    "nextHopType": "ResourceId"
  }
]

```

Let's have a look at network watcher and the next hop tool.

![illustration](/assets/routingintent/vm1tovm3.png)

To reach vm3, vm1 goes through `10.100.254.132`. And we have the same with vm2 to vm3.

![illustration](/assets/routingintent/vm2tovm3.png)

Without surprise, we can see that this ip is the hub firewall ip.

```bash

df@df-2404lts:~$ az network firewall list | jq .[].hubIPAddresses.privateIPAddress 

"10.100.254.132"
"10.100.252.132"

```

Between vms connected to the same hub, we get the firewall as the next hop, which is what we set with the custom route table or by adding a static route on the default route table.

Also, to reach vm4, vm1 is going thorugh `10.100.254.68` which is the vhub router ip.

![illustration](/assets/routingintent/vm1tovm4.png)

```bash

df@df-2404lts:~$ az network vhub list | jq .[].virtualRouterIps
[
  "10.100.254.70",
  "10.100.254.69"
]
[
  "10.100.252.70",
  "10.100.252.69"
]

```

So maybe we could try adding a static route with the neighbour firewall as a next hop?
Let's try this.
Note that we need a more specific route than the ones already defined on the custom route table.

![illustration](/assets/routingintent/statifroutetofwvhu2.png)

No issue or warning to apply the firewall from vhub2 as a next hom in a static route.

**Note that we only have the possibility to set resource id as next hop.**

**Note also that the only available resource ids are hub firewall or virtual network connections.**

But after applying, we get the following error.

![illustration](/assets/routingintent/statifroutetofwvhu2error.png)


```bash

Failed to update Route Table
Failed to update Route Table 'rt-lab-vwan-lab-001-vhub-lab-001'. Error: The next hop rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002'>afw-vwan-lab-001-vhub-lab-002 in route RouteToSpoke5 in hubRouteTable /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/virtualHubs/vwan-lab-001-vhub-lab-001/hubRouteTables/rt-lab-vwan-lab-001-vhub-lab-001 is not allowed. The next hop can only be the local hub's firewall.

```
So it does not seem to be working.

Worse, there is no control before the application of the configuration, and the route table is now in a failed state.

```bash

df@df-2404lts:~$ az network vhub route-table show -n rt-lab-vwan-lab-001-vhub-lab-001 --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 | jq .provisioningState
"Failed"
df@df-2404lts:~$ az network vhub route-table show -n rt-lab-vwan-lab-001-vhub-lab-001 --vhub-name vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 | jq .routes
[
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
  },
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
      "172.22.4.0/24"
    ],
    "name": "RouteToSpoke5",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002",
    "nextHopType": "ResourceId"
  }
]

```

**Note: To fix the route tabel failed status, we need to change the next hop to the local hub firewall, or to delete the route. The latter cannot be done from the portal**

Well, it seems that we are stuck.
But no, we have the routing intent option. 

### 2.3 Using routing intent for cross secure vhub traffic

Finally, We're here.

It was kind of a long intro just to come to this point, but It was, IMHO, necessary 	&#128517;

Ok first let's have a look at the routing intent on the portal.

![illustration](/assets/routingintent/routingintentportal.png)

So the idea here is to switch from custom route table to a more dynamic approach.
We can also see that we need to remove the custom route table, which means that we may impact all the existing network conections between spokes and vhub.

So if we have to migrate from custom route table to routing intent, we need to fllow those steps:

- Change spoke network connections to use the default route table instead of the custom route
- Remove the custom route table
- Create the routing intent and its associated policies.
- Re-create network connection for all spokes

As seen on the portal screenshot, we have 2 kindof policies. One for the Internet traffic and one for the private traffic.

For the purpose of illustration, the code snippet showing the 2 policies in a routing intent resource.

```go

# Create Routing Intent to force traffic via the firewall. Should not be used if custom RT are used

resource "azurerm_virtual_hub_routing_intent" "VhubRoutingIntent" {
  for_each       = { for k, v in var.VwanConfig.VhubsConfig : k => v if v.EnableSecureHubwithRoutingIntents && v.EnableHubFW }
  name           = "routing-intent-${azurerm_virtual_hub.Vhub[each.key].name}"
  virtual_hub_id = azurerm_virtual_hub.Vhub[each.key].id

  routing_policy {
    name         = "InternetTrafficPolicy"
    destinations = ["Internet"]
    next_hop     = azurerm_firewall.fw_hub[each.key].id
  }

  routing_policy {
    name         = "PrivateTrafficPolicy"
    destinations = ["PrivateTraffic"]
    next_hop     = azurerm_firewall.fw_hub[each.key].id
  }
}

```
We should be able to display the routing intent policies.

```bash

df@df-2404lts:~$ az network vhub routing-intent list --vhub vwan-lab-001-vhub-lab-001 -g rg-lab-vwan-001 |jq .[].routingPolicies
[
  {
    "destinations": [
      "Internet"
    ],
    "name": "InternetTrafficPolicy",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001"
  },
  {
    "destinations": [
      "PrivateTraffic"
    ],
    "name": "PrivateTrafficPolicy",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-001"
  }
]
df@df-2404lts:~$ az network vhub routing-intent list --vhub vwan-lab-001-vhub-lab-002 -g rg-lab-vwan-001 |jq .[].routingPolicies

[
  {
    "destinations": [
      "Internet"
    ],
    "name": "InternetTrafficPolicy",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002"
  },
  {
    "destinations": [
      "PrivateTraffic"
    ],
    "name": "PrivateTrafficPolicy",
    "nextHop": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-lab-vwan-001/providers/Microsoft.Network/azureFirewalls/afw-vwan-lab-001-vhub-lab-002"
  }
]

```

Once the routing intents exist on both vhub, we can check again the connectivity accross the vhubs.

```bash

vmadmin@spoke1vm:~$ ping -c 1 172.22.1.4
PING 172.22.1.4 (172.22.1.4) 56(84) bytes of data.
64 bytes from 172.22.1.4: icmp_seq=1 ttl=63 time=10.1 ms

--- 172.22.1.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 10.073/10.073/10.073/0.000 ms
vmadmin@spoke1vm:~$ ping -c 1 172.22.4.4
PING 172.22.4.4 (172.22.4.4) 56(84) bytes of data.
64 bytes from 172.22.4.4: icmp_seq=1 ttl=62 time=30.9 ms

--- 172.22.4.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.872/30.872/30.872/0.000 ms
vmadmin@spoke1vm:~$ ping -c 1 172.22.5.4
PING 172.22.5.4 (172.22.5.4) 56(84) bytes of data.
64 bytes from 172.22.5.4: icmp_seq=1 ttl=62 time=8.25 ms

--- 172.22.5.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 8.250/8.250/8.250/0.000 ms

```

```bash

vmadmin@spoke6vm:~$ ping -c 1 172.22.1.4
PING 172.22.1.4 (172.22.1.4) 56(84) bytes of data.
64 bytes from 172.22.1.4: icmp_seq=1 ttl=62 time=18.9 ms

--- 172.22.1.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 18.906/18.906/18.906/0.000 ms
vmadmin@spoke6vm:~$ ping -c 1 172.22.4.4
PING 172.22.4.4 (172.22.4.4) 56(84) bytes of data.
64 bytes from 172.22.4.4: icmp_seq=1 ttl=63 time=6.58 ms

--- 172.22.4.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 6.578/6.578/6.578/0.000 ms
vmadmin@spoke6vm:~$ ping -c 1 172.22.3.4
PING 172.22.3.4 (172.22.3.4) 56(84) bytes of data.
64 bytes from 172.22.3.4: icmp_seq=1 ttl=63 time=5.39 ms

--- 172.22.3.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 5.386/5.386/5.386/0.000 ms
vmadmin@spoke6vm:~$ ping -c 1 172.22.2.4
PING 172.22.2.4 (172.22.2.4) 56(84) bytes of data.
64 bytes from 172.22.2.4: icmp_seq=1 ttl=62 time=6.26 ms

--- 172.22.2.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 6.257/6.257/6.257/0.000 ms

```

A rapid look at the firewall logs shows that we have to go through both firewall.

![illustration](/assets/routingintent/flowfw1-table.png)

![illustration](/assets/routingintent/flowfw1-chart.png)

![illustration](/assets/routingintent/flowfw2-table.png)

![illustration](/assets/routingintent/flowfw2-chart.png)

An interesting point is that flows cross-hubs do go through both firewall, which is a desired effect, so all is good.


Ok time to wrap up!

## 3. Summary

hum, well, Vwan is powerfull, but definitly not so simple.

In this article, after reviewing a few basics on Vwan/vhub, we went through a scenario of multiple secure hubs.
The summary could be like this:

- In single vhub envirtonment, managing the route for the secure hub is doeable with custom route table and not so difficult once understood.
- Managing routing between spoke associated with a route without the firewall is also doeable, under the condition of adding a static route to the firewall proitected spokes, which may be cumbersome, specifically if no automation is in place
- Managing routing for multiple secure hub envirtonment IS NOT possible with custom route.
- The only alternative to this last scenario si with routing intent. The good part is that routing intent tend to make thing more dynamic and we do not need to manage anything else once the routing intent policies are in place.
- When routing intent is enabled, all traffic specified by the routing policies go through the firewalls. If the traffic originates from a spoke connected with vhub1, it goes through the firewall in vhub1 and then to the firewall in vhub2 before reaching its target in the spoke connected to the vhub2.

And, that's all for today.

I hope you found it useful &#128526;

