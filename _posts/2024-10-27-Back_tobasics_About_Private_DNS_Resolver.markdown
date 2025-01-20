---
layout: post
title:  "Back to basics - About Private DNS Resolver"
date:   2024-10-27 18:00:00 +0200
year: 2024
categories: Network
---

Hi all!

I had to solve some issue quite recently, related to DNS in Azure.
Guess what?
The solution that we found involved Azure Private DNS Resolver, so I'll take the opportunity to write a bit about DNS in general in Azure and use case that are solved with this managed service.

The Agenda:


1. Managing DNS in Azure
2. DNS in hybrid infrastruture
3. Sample DNS configuration in a VWAN based topology

Let's get started!

## 1. Managing DNS in Azure

### 1.1. Basic DNS considerations

As anyone involved in IT probably knows, we use DNS all the time in our day to day connected life.
Indeed, not many people are very good at remembering series of numbers and that's what it's all about in Network (and we won't even consider IPv6 &#128552;	).

That being said, because Azure is first and foremost a public cloud, it does use a lot of (mainly v4) IPs.
And since, we did not become magically proficient with numbers since the previous sentence, we do rely on DNS resolution to transfert those IP addresses in human readable name.

Microsoft manages most of the DNS domain for us, as long as we don't need to change anything.
Microsoft also offers a way to manage other public DNS domain if we come to need it. But, importantly enough to note, there is no registrar service inlcuded in Azure.
So names that should be reachable on the Internet are either managed by Microsoft, or by us as usual. 
In the latter case, we can use a 3rd party registrar and a DNS zone in Azure, with some configuration of the `ns records` on the registrar side.

![illustration1](/assets/dnsresolver/dns001.png)

Now what happens when we want to manage private DNS, meaning DNS not publicly reachable?
That would mean we want to have DNS for workloads inside a virtual network, making those isolated from the outside.
Again, as long as we don't try to change anything, it does work on its own.

Virtual machines in an Azure virtual network inherit their DNS configuration from said virtual network.

![illustration2](/assets/dnsresolver/dns002.png)

It's also possible to go to the network interface level to be more granular, even if it would probably get more complex to maintain as time pass.

![illustration3](/assets/dnsresolver/dns003.png)

If we want to further integrate DNS for our Virtual machines, we then leverage a **private** DNS zone, which can be used to register the VMs IP addresses.
In this case, we would configure the private DNS zone with `autoregister` along with a DNS link to the viurtal network hosting workload that should be able to resolve names.
Because, since the DNS zone is private, if we did not specified a virtual network link, then there would be no way that we could have resolution. 
With the link, the default Azure DNS configuration is able to find the private DNS zone and thus get DNS resolution.

![illustration4](/assets/dnsresolver/dns004.png)

![illustration5](/assets/dnsresolver/dns005.png)

Now let's consider Azure managed services.

### 1.2. PaaS, private endpoint and DNS

By default, as we mentionned, everything is public in Azure, so it makes sense that a PaaS instance would have a public IP.
Except... we don't want that right?
We would much prefer to have; let's say, an Azure Keyvault not accessible with a public IP.
To do so, we can rely on the [private link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview) technology, applied to PaaS which then become the [private endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) technology.
Simply said, we change the public IP of our Keyvault to a private IP which is the private endpoint IP: a network interface living in a virtual network in Azure. We talked about that previously when we discussed about [AKS Networking](/_posts/2023-10-31-AKS_Networking_considerationspart1.markdown)

![illustration6](/assets/aksntwconsiderations/PrivateLink.png)

Notice that we change the IP to be private, so it would not make sense that we keep the public DNS domain with a private record set (there is at least one exception to this, but let's not get confused today &#129299;).
Refering to the [Private Endpoint documentation](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns), we can see that the technolgy leverage Azure private DNS zones that are, in this case, subdomain of the public DNS domain used for the public PaaS instance.
Taking our keyvault example, which public name was `<keyvault_name>.vault.azure.net`, it would become `<keyvault_name>.privatelink.vaultcore.azure.net`.

Obviously, once the public access is disabled in profit of the private endpoint, we need to be in a connected network to resolve and access the key vault. That's the point.

```bash

yumemaru@azure~$ nslookup kvnysvsi.vault.azure.net
Server:         168.63.129.16
Address:        168.63.129.16#53

Non-authoritative answer:
kvnysvsi.vault.azure.net        canonical name = <keyvault_name>.privatelink.vaultcore.azure.net.
Name:   kvnysvsi.privatelink.vaultcore.azure.net
Address: 172.21.12.132

```

So, as long as we remain in a full Azure environment, it's quite simple. 

- A properly named private DNS zone
- A private endpoint registered in the DNS zone
- A virtual network link to the virtual network in which the private endpoint lives (or a virtual network connected to this virtual network)

What about in hybrid environment?

## 2. DNS in hybrid infrastruture

With hybrid environment, one of the main challenge is the network part. 
Leaving the old question about puting DNS in the system or the network perimeter &#129325;, we can include hybrid DNS in the challenge of the hybrid network configuration. LEt's start with the resolution from On Premise to Azure private DNS

### 2.1. Getting DNS resolution from On-premise to Azure

Taking the hypothesis that we have some workload in an On-Premise network, we can easily assume that there is a DNS infrastructure already in place, to allow DNS resolution in this network. Chances are the DNS infrastructure relies on an AD DS integrated DNS.

![illustration6](/assets/dnsresolver/onpremdns.png)

Once Azure comes in the picture, we need some kind of connection between the on premise network and the Azure network. 

This is an post about DNS, so let's not discuss the connection mean and consider that it's already existing.
With that we have:

- the on-premise network, connected to the Azure network containing the private endpoint.
- A DNS system on premise, currently able to resolve on premise name.
- Azure services, such as VM, living in a virtual network, or an Azure PaaS, also connected to a virtual network by the mean of a private endpoint
- Private DNS zones, which register the Azure workloads, being VMs or PaaS, and link to the virtual network where DNS resolution must occur.

To allow the resolution from on-premise, we need to tell the to on-premise DNS where to find the DNS authority for the Azure workload. And that's done by configuring DNS fowrwarding.
Specifically, we use conditional forwarding and specify Azure DNS IP `168.63.129.16`, which is the same IP that answered in our sample DNS resolution.
Again, we can rely on the Azure documentation which specifies which DNS domain to configure in the forwarding, depending on the target Azure service. Taking our Keyvault example, and an AKS cluster in East Us, we get the following configuration: 

| Azure service| Private DNS zone for private Endpoint| Forwarding configuration |
|-|-|-|
| Azure Key Vault | `privatelink.vaultcore.azure.net` | `vault.azure.net` <br>`vaultcore.azure.net` |
|Azure Kubernetes Service | `privatelink.eastus.azmk8s.io` | `eastus.azmk8s.io` |

![illustration8](/assets/dnsresolver/onpremdnsconnectedconditionalfwd.png)

For an Azure virtual machine registered in a private dns zone, we add a conditional forwarder to the DNS domain of the private DNS zone.
On a Windows DNS server, it would look like this:

![illustration9](/assets/dnsresolver/conditionalfwd001.png) 

![illustration10](/assets/dnsresolver/conditionalfwd002.png)

From our client Vm, we get DNS resolution by the mean of the conditional forwarding: 

```bash

PS C:\Users\vmadmin> ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : client1
   Primary Dns Suffix  . . . . . . . : dcdemo.infra
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : dcdemo.infra
                                       reddog.microsoft.com

Ethernet adapter Ethernet 2:

   Connection-specific DNS Suffix  . : reddog.microsoft.com
   Description . . . . . . . . . . . : Microsoft Hyper-V Network Adapter #2
   Physical Address. . . . . . . . . : 60-45-BD-FE-A7-5A
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::bc84:3604:caec:675c%13(Preferred) 
   IPv4 Address. . . . . . . . . . . : 172.21.11.68(Preferred) 
   Subnet Mask . . . . . . . . . . . : 255.255.255.192client1
   Lease Obtained. . . . . . . . . . : Sunday, October 27, 2024 8:04:23 PM
   Lease Expires . . . . . . . . . . : Thursday, December 4, 2160 4:08:35 AM
   Default Gateway . . . . . . . . . : 172.21.11.65
   DHCP Server . . . . . . . . . . . : 168.63.129.16
   DHCPv6 IAID . . . . . . . . . . . : 123749821
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2E-79-B9-6F-60-45-BD-D7-80-C2
   DNS Servers . . . . . . . . . . . : 172.21.11.4
   NetBIOS over Tcpip. . . . . . . . : Enabled



```

![illustration11](/assets/dnsresolver/nslookup001.png)

![illustration12](/assets/dnsresolver/nslookup002.png)

And we have it!
DNS resolution  from on-premise to Azure.
Let's see how we could achieve the reverse now.

### 2.2 Getting DNS resolution from Azure to On-premise

With conditional forwarding, it's possible to tell an on-premise workload to find the proper DNS information through conditional forwarding.
But how could we do this in reverse?
We have 2 possible use case here:

- The Azure VM need to access DNS resolution for on-premise service
- An Azure PaaS instance need to resolve DNS name for on-premise servide

If we think about it, it's theoritically simple. As long as the service lives in a virtual Network, it get its DNS configuration from the DNS configuration on the vnet or the Nic, as we've seen earlier.
So if we change the DNS configuration and point the DNS config to an on-premise DNS server (in our case, that would be DC1), then we get DNS resolution, end of story &#128526;

But well, it's not ideal right?
As I said, it's a DNS focused post, so no consideration for network connnectivity between Azure and on-premise.
But let's make a little incurion in the network, just a bit.

If all the DNS request go to an on-premise server, we go through the network connection each time. This is definitely not considered a best practice in the well architected framework.
Indeed, the [recommandation](https://learn.microsoft.com/en-us/azure/architecture/hybrid/hybrid-dns-infra#availability) is to keep the DNS servers close to theirs clients.

So what do we do?
Well, the first option is to add a DNS infrastructure, meaning at least a pair of servers on 2 availability zones, in Azure, and configure the virtual network with a custom DNS configuration pointing to this DNS infrastructure.

![illustration13](/assets/dnsresolver/azuretoonprem.png)

That work also. 

Note by the way that we change the on-premise DNS configuration which forward to the Azure IaaS DNS now, instead of the Azure DNS address.
All in all, it's a bit more to manage in terms of IaaS, so some people may not like the idea.
Fortunately, there is a more managed option which is called the DNS private resolver

## 3. Sample DNS configuration in a VWAN based topology and DNS Private resolver

For this last part, we will put ourselves in a more real life scenario.
We will take a Hub & spoke topoloigy with Azure Virtual WAN.

We have a spoke hosting an Aks cluster and some VMs, and another one that we'll use for the DNS private resolver.
Simulating an on-premise enetwork, we have a spoke which contains a domain controller called `dc1`, and a client called `client1`.
Those 2 Vms are configured with `dc1` as a DNS server, and with the conditional forwarding that we discussed in the previous part, directly to the Azure IP `168.63.129.16`.

At this point, the other spokes use the default Azure configuration and can only get public DNS resolution, or Azure private DNS zone resolution, if the private zones are linked to the spokes.

![illustration14](/assets/dnsresolver/vwantopology001.png)

Now let's add a DNS private resolver.

### 3.1. DNS private resolver overview

Behind the name, it's a managed service that solves differents DNS scenario.
The first one: we want to have hybrid DNS, from Azure to on-premise. Except, we don't wand to manage a IaaS DNS server.

Here come the DNS private resolver.

From the related [documentation](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview), this service enables querying Azure DNS private zones from an on-premises environment and vice versa **without deploying VM based DNS servers**.

Technically speaking, a DNS private resolver, once deployed, will leverage either an outbound endpoint, and inbound enpoint, or both.

#### 3.1.1 Oubount Endpoint

Outbound endpoints manage egress traffic from Azure to on-premise network. It uses a dedicated subnet, even if there is no IP visibly consumed by the service.

![illustration15](/assets/dnsresolver/privateresolver001.png)

```json

{
    "name": "sub2-vnet-sbx-dnsfwd",
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub2-vnet-sbx-dnsfwd",
    "etag": "W/\"c4a6b65e-df44-4b86-9cd2-62e60ee4c933\"",
    "properties": {
        "provisioningState": "Succeeded",
        "addressPrefix": "172.21.13.64/26",
        "networkSecurityGroup": {
            "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/networkSecurityGroups/nsg-sub2-vnet-sbx-dnsfwd"
        },
        "serviceAssociationLinks": [
            {
                "name": "dnsResolverLink",
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub2-vnet-sbx-dnsfwd/serviceAssociationLinks/dnsResolverLink",
                "etag": "W/\"c4a6b65e-df44-4b86-9cd2-62e60ee4c933\"",
                "type": "Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks",
                "properties": {
                    "provisioningState": "Succeeded",
                    "linkedResourceType": "Microsoft.Network/dnsResolvers",
                    "link": "/OutboundEndpoint/bac4d5bd-07fc-4a63-9ecb-1ddde6be017d",
                    "enabledForArmDeployments": false,
                    "allowDelete": false,
                    "subnetId": "00000000-0000-0000-0000-000000000000",
                    "locations": []
                }
            }
        ],
        "delegations": [
            {
                "name": "Microsoft.Network.dnsResolvers",
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub2-vnet-sbx-dnsfwd/delegations/Microsoft.Network.dnsResolvers",
                "etag": "W/\"c4a6b65e-df44-4b86-9cd2-62e60ee4c933\"",
                "properties": {
                    "provisioningState": "Succeeded",
                    "serviceName": "Microsoft.Network/dnsResolvers",
                    "actions": [
                        "Microsoft.Network/virtualNetworks/subnets/join/action"
                    ]
                },
                "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
            }
        ],
        "privateEndpointNetworkPolicies": "Disabled",
        "privateLinkServiceNetworkPolicies": "Enabled"
    },
    "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

![illustration16](/assets/dnsresolver/privateresolver002.png)

To manage the forwarding to the on-premise dns servers, the outbount endpoint works with a dns forwarding ruleset. It's with this element that we can access to the forwarding rules. In our case, we add the AD DS DNS domaine `dcdemo.infra.` and specify `dc1` IP as the target for the forwarding.

![illustration17](/assets/dnsresolver/privateresolver003.png)

In this state, the on-premise workload are still not available. We need to add a link to the virtual network that need to be able to resolve.
Because it's a link, we do not have to change the virtual network DNS configuration

![illustration18](/assets/dnsresolver/privateresolver004.png)

![illustration19](/assets/dnsresolver/vwantopology002.png)

Testing resolution on vm `client2`, we get DNS resolution for `dc1` and `client1`

![illustration20](/assets/dnsresolver/privateresolver005.png)


And that gives us resolution for on-premise DNS domains, on any spoke with a virtual network link. This architecture pattern is called the [distributed DNS architecture](https://learn.microsoft.com/en-us/azure/dns/private-resolver-architecture#distributed-dns-architecture).

It's interesting to note that the DNS query goes to *Internet*, in this case Azure Public DNS IP `168.63.129.16`, as we can see on the nslookup output.

Let's have a look at what brings us the Inbound endpoint now.

#### 3.1.2. Inbound Endpoint

Since the Outbound endpoint allows us to manage Egress Traffic from Azure to On-premise, it make sense that the Inbound endpoint allows us to do the opposite.
As for the Outbound endpoint, the Inbound enpoint uses a dedicated subnet. However, it does consumes IP, with a managed load balancer that can be seen in the connected device.

![illustration21](/assets/dnsresolver/privateresolver006.png)

```json

{
    "name": "sub1-vnet-sbx-dnsfwd",
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub1-vnet-sbx-dnsfwd",
    "etag": "W/\"db0ec902-a1b3-497a-ba4a-ee07c65e269a\"",
    "properties": {
        "provisioningState": "Succeeded",
        "addressPrefix": "172.21.13.0/26",
        "networkSecurityGroup": {
            "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/networkSecurityGroups/nsg-sub1-vnet-sbx-dnsfwd"
        },
        "ipConfigurations": [
            {
                "id": "/subscriptions/cbef980d-e065-4675-b6f2-2d9674bf9fbd/resourceGroups/rg-prod-eastus-main-endpoints/providers/Microsoft.Network/loadBalancers/lbi-prod-eastus-main-ept-59b465e9fb2d4cc1899fc5f4f27ad6e2/frontendIPConfigurations/frontend-ipconfig"
            }
        ],
        "serviceAssociationLinks": [
            {
                "name": "dnsResolverLink",
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub1-vnet-sbx-dnsfwd/serviceAssociationLinks/dnsResolverLink",
                "etag": "W/\"db0ec902-a1b3-497a-ba4a-ee07c65e269a\"",
                "type": "Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks",
                "properties": {
                    "provisioningState": "Succeeded",
                    "linkedResourceType": "Microsoft.Network/dnsResolvers",
                    "link": "/InboundEndpoint/59b465e9-fb2d-4cc1-899f-c5f4f27ad6e2",
                    "enabledForArmDeployments": false,
                    "allowDelete": false,
                    "locations": []
                }
            }
        ],
        "delegations": [
            {
                "name": "Microsoft.Network.dnsResolvers",
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-dns-forwarder/providers/Microsoft.Network/virtualNetworks/vnet-sbx-dnsfwd1/subnets/sub1-vnet-sbx-dnsfwd/delegations/Microsoft.Network.dnsResolvers",
                "etag": "W/\"db0ec902-a1b3-497a-ba4a-ee07c65e269a\"",
                "properties": {
                    "provisioningState": "Succeeded",
                    "serviceName": "Microsoft.Network/dnsResolvers",
                    "actions": [
                        "Microsoft.Network/virtualNetworks/subnets/join/action"
                    ]
                },
                "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
            }
        ],
        "privateEndpointNetworkPolicies": "Disabled",
        "privateLinkServiceNetworkPolicies": "Enabled"
    },
    "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

![illustration22](/assets/dnsresolver/privateresolver010.png)



![illustration23](/assets/dnsresolver/privateresolver007.png)

Instead of pointing to an Azure IaaS DNS server, we can configure the condition forwarder on the `dc1` to point to the Inbound endpoint

![illustration24](/assets/dnsresolver/privateresolver008.png)

We can check that DNS resolution is still working on `client1`

![illustration25](/assets/dnsresolver/privateresolver009.png)

And with that, we have a fully fonctional Hybrid DNS architecture.
We can also, instead of using the distributed model in Azure, switch to the centralized model.
In this case, we just need to change the DNS configuration for all spokes so that it point to the Inbound Endpoint.
On the DNS private resolver vnet, however, we keep the DNS configuration to Azure default, and we add all the required link:

- To the DNS private resolver outbound endpoint
- To the Private DNS zones

With this centralized model, we manage the Virtual Network link centrally on the DNS private resolver Vnet.
Also, instead of forwarding queries to the Internet, Azure client forward the queries to the Inbound Endpoint first.

![illustration26](/assets/dnsresolver/vwantopology003.png)

And we've done it ^^

Last, if we put ourselves in a Secure Hub configuration, then we need to consider the Firewall rules. Because this is only DNS, it's UDP 53 to and from the DNS resolver Vnet.

Ok let's wrap it!

## 4. Summary

In this article we reviewed the basics of DNS:

- In Azure, DNS is provided by Azure DNS on virtual Network, without any configuration
- Managing DNS for object living in Vnet implies using Private DNS zones and Virtual Network link on said zones.
- Troubles start with hybrid resolution but can be resolved with Conditional forwarding on the On-premise DNS systems, and something in Azure, preferably managed such as DNS Private resolver, in Azure to forward traffic to On-premise.
- DNS private resolver can be use with either a cnetralized or a distributed model

And that's all.

I hope it's useful, see you soon!




