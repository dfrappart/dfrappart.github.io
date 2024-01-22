---
layout: post
title:  "Walkthrough - Secure Hub in Azure Virtual WAN"
date:   2024-01-19 18:00:00 +0200
year: 2024
categories: Azure Network Terraform
---

Hi all!

This is the 2nd article about Azure Virtual WAN. 
This time we're going to look at the Secure Hub concepts, what it is about and what it brings to the table.
As for last time, we'll take a pragmatic approach and build our environment iteratively to illustrate the concepts.

Hope you'll enjoy it!

On the agenda today: 

1. Concepts:
- From Virtual Hub to Secure Hub
- Why Secure Hub. Use case for a centralized Firewall in the Cloud
- a look on the available options
2. Building a Secure Hub from scratch
- Building a hub
- adding a firewall to get a Secure Hub
- Configuring routing & Firewall rules
- Additional routing configuration

With that being said, let's get started.

## 1. From Virtual Hub to Secure Hub

If you read the [Azure documentation about Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/), or my l[ast article on this topic](https://blog.teknews.cloud/azure/network/2023/08/28/virtualwan101.html), you'll know that in a Virtual WAN based topology, we deploy inside (or is it on top 🤔) one or more virtual Hub. You probably also now that a virtual Hub is a good solution to ease the East-West connectivity between Spokes thanks to the managed virtual routers that take care of route propagation in the vHub. All of this without using UDRs for routing, inside of spokes.

However, in this simple, yet efficient scenario, we lack option to filter traffic between those spokes, or from those spokes to other places which are not virtual network in Azure. Indeed, we can leverage Network Security Groups, but limitations do exist such as rules limited to L4, or Application Security Group only valid for intra spoke filtering. I wrote another article about NSGs in the past that you can find [here](https://blog.teknews.cloud/network/2022/03/01/Back_to_basics_About_Network_Security_Groups.html)

![illustration1](/assets/securehub/hubandspokewithnsg.png)

One answer proposed by the Virtual WAN model is the use of a Secure Hub. In this scenario, a Network appliance is deployed inside the Virtual Hub and allows for a centralized Network filtering option. To be clear, **I am NOT** saying that NSGs should be removed in favor of a Hub Firewall only. Firstly because a Zero trust approach will prefer additional security layers (not taking into account the configuration management but that's clearly not the main topic here). Secondly, from a Spoke Vnet point of view, leaving the vnet to go through a firewall in the hub, to go back into the spoke would be strange, to say the least, from an network perspective. Hence local filtering in Spokes.
So, instead of a simple Hub & spokes, we get something similar to this:

![illustration2](/assets/securehub/hubandspokewithfw.png)

There are probably questions regarding the routing configuration, but fear not, we'll tackle that in the last part of the article. Now we have a centralized firewall in our virtual hub, which change our hub to a secure hub
As mentioned, its use cases are to 

- managed network filtering between spokes
- provide additional capabilities such as L7 filtering
- provide an Internet breakout for spokes (Not to be forgotten since [default Internet access is going away from Azure](https://azure.microsoft.com/fr-fr/updates/default-outbound-access-for-vms-in-azure-will-be-retired-transition-to-a-new-method-of-internet-access/))

An interesting aspects is also this centralized property, which can be used to give a central Network team a point of control for everything that comes out of the spoke. Network flows management governance is not the heart of today's topic, but let's keep that in mind.

Otherwise, other capabilities can also be considered depending on the solution used.

Which bring us to the available options for the firewall, on which the capabilities will depend.

About that, it's always a good idea to remind ourselves of the **managed aspect** of the virtual hub. Because it's managed, there are less responsibilities on the customer part, but also less control. It means that the options for NVA are less rich than the Azure market place offer which... offers many dofferent options for NVA to be deployed in a self-managed Virtual NEtwork.

In a Virtual Hub, the first available option is the managed one, with Azure Firewall. 
Azure Firewall is a managed, zone redundant, appliance which can bring the filtering to the next level. Its native integration with Azure ease the automation implementation. Autmating an Azure Firewall deployment and configuration is done with the same tool used for Azure deployment and configuration. On the other hand, there are maybe not all the options that a 3rd party provider can deliver. 
One of those could be the Internet Ingress traffic, which is not a primary objective for Azure Firewall, except if coupled with an Azure WAF solution.

If some configuration are lacking, then it is possible to look at a 3rd party provider, as documented on the [Azure documentation](https://learn.microsoft.com/en-us/azure/virtual-wan/about-nva-hub#partners).
Currently, Security partner includes [CheckPoint](https://www.checkpoint.com/cloudguard/microsoft-azure-security/wan/) and [Fortigate](https://www.fortinet.com/products/next-generation-firewall), and also [Palo Alto(https://azure.microsoft.com/en-us/updates/public-preview-palo-alto-networks-saas-cloud-ngfw-integration-with-virtual-wan/)] as a SaaS offer in public preview. Other partners can be found but more on the SD WAN integration than the Firewalling options.

For now, we'll stop here on the concept and get practical.

## 2. Building a secure hub from scratch

Last time, we built everything through az cli. To put a little more fun in the build experience, this time, we're going to go through a terraform path.
So to get started, we want a virtual hub, in a virtual WAN.
We will also want a few spokes to connect to the vHub with NSGs and all.
Then we'll add the firewall and start playing with it.

### 2.1. Building a Virtual Hub through terraform

Since we did it once already, we know that for a virtual hub, the following resources are needed:

- a virtual WAN
- a virtual hub

A quick check to the terraform documentation will get us the following

```go

resource "azurerm_resource_group" "RgVwanLab" {
  name     = "rg-vwan-lab-001"
  location = "eastus"

}

resource "azurerm_virtual_wan" "Vwan" {
  name                              = "vwan-lab"
  resource_group_name               = azurerm_resource_group.RgVwanLab.name
  location                          = azurerm_resource_group.RgVwanLab.location
  allow_branch_to_branch_traffic    = true
  disable_vpn_encryption            = true
  type                              = "Standard"
  tags                              = {}
  office365_local_breakout_category = "None"
}

resource "azurerm_virtual_hub" "Vhub" {
  name                = "vhub-vwan-lab-001"
  resource_group_name = azurerm_resource_group.RgVwanLab.name
  location            = azurerm_resource_group.RgVwanLab.location
  virtual_wan_id      = azurerm_virtual_wan.Vwan.id
  address_prefix      = "10.100.254.0/23"
  hub_routing_preference = "ExpressRoute"
  sku = "Standard"
  tags = {}
}

```

We want to add some spokes, and once those are created, add a network connection from the virtual hub, which is similar to a peering from the spoke point of view but managed centrally in the virtual hub.

```go

resource "azurerm_virtual_hub_connection" "peering_spoke1" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo11","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet.VNetFullOutput.id

}


resource "azurerm_virtual_hub_connection" "peering_spoke2" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo21","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet2.VNetFullOutput.id

}


resource "azurerm_virtual_hub_connection" "peering_spoke3" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo31","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet3.VNetFullOutput.id

}

resource "azurerm_virtual_hub_connection" "peering_spoke4" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo41","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet4.VNetFullOutput.id

}

```

### 2.2. Adding a Firewall in the pciture

This time we want to create a Secure hub, so we will need the resources regarding our firewall, in this case, an Azure firewall:


```go

resource "azurerm_firewall" "fw_hub" {
  name                = "afw-vhub-vwan-lab-001"
  location            = azurerm_resource_group.RgVwanLab.location
  resource_group_name = azurerm_resource_group.RgVwanLab.name
  sku_name            = "AZFW_Hub"
  sku_tier            = var.FwSkuTier
  firewall_policy_id  = azurerm_firewall_policy.fwpolicy_hub.id
  virtual_hub {
    virtual_hub_id  = azurerm_virtual_hub.Vhub.id
    public_ip_count = var.FWPubIpCount
  }
}

```

Those who managed an Azure Firewall in virtual network will notice that there is less configuration to do to attach a firewall to a virtual hub.
In fact, we just have to specify the `virtual_hub` block. Also, we have to set the `sku_name` to `AZFW_Hub`.
Notice the firewall_policy_id, which, while being optional, allows us to attach a firewall policy to the firewall. We need to add in this case the corresponding resource: 

```go

resource "azurerm_firewall_policy" "fwpolicy_hub" {
  name                = "fwpol-basepolicy"
  resource_group_name = azurerm_resource_group.RgVwanLab.name
  location            = azurerm_resource_group.RgVwanLab.location
}

```

This attach a base policy to the firewall, but no rule at this point.

With that we only have created our firewall and a container of rules, but the routing configuration will still propagate network connection on the default route, and nothing goes throught the firewall.

Checking the topology on the portal we cannot see the firewall yet:


![illustration3](/assets/securehub/Topology.svg)

Having a look at the firewall, we can see some interesting information regarding the IPs : 

![illustration4](/assets/securehub/azfwipconfig.png)

Now if we check the routing configuration between 2 vms in 2 spokes, we can see confirm that the next hop is not the firewall IP : 

![illustration5](/assets/securehub/nexthopnofw.png)

The routing configuration of the VWAN is in the default configuration. We have a default route table,

![illustration6](/assets/securehub/routeconfig.png)

 and the routes for each spoke is propagated at the network connection (peering) creation.

![illustration7](/assets/securehub/ntwconnectiondefaultrouting.png)

### 2.3. Configuring routing to get a Secure Hub


Now let's add a custom route table which we will use to force traffic through the firewall.

```go

resource "azurerm_virtual_hub_route_table" "CustomRouteTable" {
  name           = "rt-lab-001"
  virtual_hub_id = azurerm_virtual_hub.Vhub.id
  labels         = ["privatezone"]


}


resource "azurerm_virtual_hub_route_table_route" "InternetRoute" {
  route_table_id = azurerm_virtual_hub_route_table.CustomRouteTable.id

  name              = "InternetToFw"
  destinations_type = "CIDR"
  destinations      = ["0.0.0.0/0"]
  next_hop_type     = "ResourceId"
  next_hop          = azurerm_firewall.fw_hub.id
}

resource "azurerm_virtual_hub_route_table_route" "PrivateCIDRRoute" {
  route_table_id = azurerm_virtual_hub_route_table.CustomRouteTable.id

  name              = "PrivateCIDRToFW"
  destinations_type = "CIDR"
  destinations      = ["10.0.0.0/8","172.16.0.0/12","192.168.0.0/16"]
  next_hop_type     = "ResourceId"
  next_hop          = azurerm_firewall.fw_hub.id
}



```

Now that we have this new route table, we need to modify the network connections. for testing purpose, we will only change network connections to spoke 2 and 4:

```go

resource "azurerm_virtual_hub_connection" "peering_spoke2" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo21","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet2.VNetFullOutput.id

  internet_security_enabled = false
  routing {
   associated_route_table_id = format("%s%s%s",var.VirtualHubId,"/hubRouteTables/",var.CustomRouteTableName)
   propagated_route_table {
     route_table_ids = [format("%s%s%s",var.VirtualHubId,"/hubRouteTables/","noneRouteTable")]
     labels          = ["none"]
   }
  }

}

resource "azurerm_virtual_hub_connection" "peering_spoke4" {
  name                      = format("peer-%s-to-%s", "vnet-sbx-spokedemo41","vhub-vwan-lab-001")
  virtual_hub_id            = var.VirtualHubId
  remote_virtual_network_id = module.testvnet4.VNetFullOutput.id

  internet_security_enabled = true
  routing {
   associated_route_table_id = format("%s%s%s",var.VirtualHubId,"/hubRouteTables/",var.CustomRouteTableName)
   propagated_route_table {
     route_table_ids = [format("%s%s%s",var.VirtualHubId,"/hubRouteTables/","noneRouteTable")]
     labels          = ["none"]
   }
  }
}

```

We'll notice the parameter `internet_security_enabled` which can be set to false or true. More on that later.
In the `routing` block, we can see the `associated_route_table_id` set to the custom route table that we juste created.
In the `propagated_route_table` block, we set the propagation to the `noneRouteTable` which is a way to not propagate the range from the connected network. The default configuration propagate those ranges and allows a simple routing configuration. 
Since we want to move to a Secure Hub, we said goodbye to the simple configuration ^^.

checking on the portal, we have an additional route table:

![illustration8](/assets/securehub/routeconfigsecure.png)

And we can see the network connections changes regarding the associated route table

![illustration9](/assets/securehub/ntwconnectioncustomrouting.png)

Let's check Network watcher for the Next Hop.

First in a spoke, between 2 VMs in the same subnets: 

![illustration10](/assets/securehub/routingvmsamespoke.png)

![illustration11](/assets/securehub/routingvmsamespokentwwatcher.png)

Adding a public IP to one of the vm, with the proper NSG configuration, we can get access to the VM and check that we can indeed access to the other vm

```bash

yumemaru@azure:~$ ssh -i spoke21-vm2_key.pem azureuser@172.203.210.198
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-1053-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Jan 18 16:19:09 UTC 2024

  System load:  0.0               Processes:             109
  Usage of /:   6.3% of 28.89GB   Users logged in:       0
  Memory usage: 7%                IPv4 address for eth0: 172.21.1.68
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

21 updates can be applied immediately.
17 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Thu Jan 18 16:05:26 2024 from 81.220.211.213
azureuser@spoke21-vm2:~$ ping -c 4 172.21.1.4
PING 172.21.1.4 (172.21.1.4) 56(84) bytes of data.
64 bytes from 172.21.1.4: icmp_seq=1 ttl=64 time=1.65 ms
64 bytes from 172.21.1.4: icmp_seq=2 ttl=64 time=1.35 ms
64 bytes from 172.21.1.4: icmp_seq=3 ttl=64 time=1.45 ms
64 bytes from 172.21.1.4: icmp_seq=4 ttl=64 time=1.66 ms

--- 172.21.1.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.349/1.525/1.659/0.132 ms


```

```bash

azureuser@spoke21-vm2:~/sshkey$ ssh -i spoke21-vm1_key.pem azureuser@172.21.1.4
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-1053-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Jan 18 16:31:48 UTC 2024

  System load:  0.07              Processes:             112
  Usage of /:   5.7% of 28.89GB   Users logged in:       0
  Memory usage: 7%                IPv4 address for eth0: 172.21.1.4
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Thu Jan 18 16:30:17 2024 from 172.21.1.68
azureuser@spoke21-vm1:~$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.21.1.4  netmask 255.255.255.192  broadcast 172.21.1.63
        inet6 fe80::6245:bdff:feff:4a18  prefixlen 64  scopeid 0x20<link>
        ether 60:45:bd:ff:4a:18  txqueuelen 1000  (Ethernet)
        RX packets 87319  bytes 51917010 (51.9 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 79452  bytes 22008661 (22.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 90  bytes 10600 (10.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 90  bytes 10600 (10.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

azureuser@spoke21-vm1:~$ 


```

Now let's check the routing configuration for VM in different spokes. Remember, we configured 2 spokes to use the custom route through the Firewall.

![illustration12](/assets/securehub/routingvmdifferentspokes.png)

![illustration13](/assets/securehub/routingvmdifferentspokesntwwatcher.png)

This time, the next hop is `10.100.254.132`, which is, if you remember, the Firewall private IP.
Wihout surprise, when we try to ping the VM, we dan't get any answer, because we go through the Firewall, and we did not open any flow.

```bash

azureuser@spoke21-vm2:~/sshkey$ ping -c 4 172.21.3.4
PING 172.21.3.4 (172.21.3.4) 56(84) bytes of data.

--- 172.21.3.4 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3053ms

azureuser@spoke21-vm2:~/sshkey$ 


```

Let's add some rules on the firewall. As opposite to the NSGs, the Azure Firewall is indeed a Network appliance through which the traffic goes.
For the purpose of this scenario, we will create one rule collection per spoke, to define the flows that we want to allow in each spoke.
Also in the specific case, we would like validate tha the firewall is doing its job, so we'll add a rule to allow ICMP between the spoke, and another to allow only one subnet to access another through SSH.



```go

resource "azurerm_firewall_policy_rule_collection_group" "FwRulleCollSpoke21" {
  name               = "FwRulleCollSpoke21"
  firewall_policy_id = var.FwPolicyId
  priority           = 10000

}


resource "azurerm_firewall_policy_rule_collection_group" "FwRulleCollSpoke41" {
  name               = "FwRulleCollSpoke41"
  firewall_policy_id = var.FwPolicyId
  priority           = 10010

  network_rule_collection {
    name     = "Spoke41AllowNtw"
    priority = 400
    action   = "Allow"
    rule {
      name                  = "AllowSSHToSpoke41"
      protocols             = ["TCP"]
      source_ip_groups      = [module.testvnet2.subnetIpGroups.Subnet1.id]
      destination_ip_groups = [module.testvnet4.subnetIpGroups.Subnet1.id]
      destination_ports     = ["22"]
    }
    rule {
      name                  = "AllowICMPToSpoke4"
      protocols             = ["ICMP"]
      source_ip_groups      = [module.testvnet2.VnetIpGroup.id]
      destination_ip_groups = [module.testvnet4.VnetIpGroup.id]
      destination_ports     = ["*"]
    }
  }

}

```

Notice in the  source and destination, the use of IP Groups which are Azure objects allowing us to define custom tags for the Azure Firewall.



```go

resource "azurerm_ip_group" "VnetCidr" {
  name                = local.VnetName
  location            = azurerm_virtual_network.Vnet.location
  resource_group_name = azurerm_virtual_network.Vnet.resource_group_name

  cidrs = [var.Vnet.AddressSpace]
}

```

The output of this kind of object would gives us something like that.

```go

{
  "cidrs" = toset([
    "172.21.1.0/24",
  ])
  "firewall_ids" = tolist([])
  "firewall_policy_ids" = tolist([
    "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-vwan-lab-001/providers/Microsoft.Network/firewallPolicies/fwpol-test001",
  ])
  "id" = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-spoke-lab-001/providers/Microsoft.Network/ipGroups/vnet-sbx-spokedemo21"
  "location" = "eastus"
  "name" = "vnet-sbx-spokedemo21"
  "resource_group_name" = "rg-spoke-lab-001"
  "timeouts" = null /* object */
}

```

Reflecting on that we could also write the rules using the `cidrs` properties of the IP Group instead of its id, changing at the same time the rule touse the argument `source_addresses` and `destinations_addresses` instead of `source_ip_groups` and `destination_ip_groups`:

```go

    rule {
      name                  = "AllowICMPToSpoke4"
      protocols             = ["ICMP"]
      source_ip_groups      = module.testvnet2.VnetIpGroup.cidrs
      destination_ip_groups = module.testvnet4.VnetIpGroup.cidrs
      destination_ports     = ["*"]
    }

```
However, the rules created with the IP Groups Ids are displayed with the name of the IP group on the portal, while thos ecreated with the IP Groups cidrs are showing the cidr, whic is less user friendly.

![illustration14](/assets/securehub/ruledisplayed.png)


With that done, we can now ping the VM in the second spoke from all VMs in the first. However, only the VM1 in the first spoke can access the VM in the second spoke throug SSH:

```bash

azureuser@spoke21-vm2:~$ ping -c 4 172.21.3.4
PING 172.21.3.4 (172.21.3.4) 56(84) bytes of data.
64 bytes from 172.21.3.4: icmp_seq=1 ttl=63 time=3.08 ms
64 bytes from 172.21.3.4: icmp_seq=2 ttl=63 time=3.07 ms
64 bytes from 172.21.3.4: icmp_seq=3 ttl=63 time=2.55 ms
64 bytes from 172.21.3.4: icmp_seq=4 ttl=63 time=2.73 ms

--- 172.21.3.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 2.552/2.858/3.078/0.226 ms
azureuser@spoke21-vm2:~/sshkey$ ssh -i spoke41-vm1_key.pem azureuser@172.21.3.4
ssh: connect to host 172.21.3.4 port 22: Connection timed out


```

```bash

azureuser@spoke21-vm2:~/sshkey$ ssh -i spoke21-vm1_key.pem azureuser@172.21.1.4
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-1053-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Jan 19 10:40:54 UTC 2024

  System load:  0.14              Processes:             111
  Usage of /:   5.7% of 28.89GB   Users logged in:       0
  Memory usage: 7%                IPv4 address for eth0: 172.21.1.4
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Fri Jan 19 10:39:51 2024 from 172.21.1.68
azureuser@spoke21-vm1:~$ cd sshkeys/
azureuser@spoke21-vm1:~/sshkeys$ vim spoke41-vm1_key.pem
azureuser@spoke21-vm1:~/sshkeys$ ssh -i spoke41-vm1_key.pem azureuser@172.21.3.4
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.2.0-1018-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Jan 19 10:41:34 UTC 2024

  System load:  0.080078125       Processes:             106
  Usage of /:   6.1% of 28.89GB   Users logged in:       0
  Memory usage: 7%                IPv4 address for eth0: 172.21.3.4
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

2 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Thu Jan 11 14:04:05 2024 from 172.21.1.4
azureuser@spoke41-vm1:~$ 

```


### 2.4. Additional considerations

#### 2.4.1. Egress flows on the Firewall

At this point, it seems that everything is working fine. The firewall is in place, and it allows us to filter between spoke.
But we did not discuss about the parameter `internet_security_enabled`.
Remember, we created one network connection with this parmater set to `false` and the other to `true`.

For the VM in the spoke with the `internet_security_enabled` set to false, I was able to connect from the internet directly (after adding a public IP and configuring the NSG though) and I have no trouble to reach Ubuntu depots, while I did not specify any flow in this sense.

Now on the VM in the second spoke, on which the `internet_security_enabled` is set to true, in the current configuration, I cannot reach those same depots. 

```bash

azureuser@spoke41-vm1:~$ sudo apt update
Err:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
  470  status code 470 [IP: 52.147.219.192 80]
Err:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease
  470  status code 470 [IP: 52.147.219.192 80]
Err:3 http://azure.archive.ubuntu.com/ubuntu jammy-backports InRelease
  470  status code 470 [IP: 52.147.219.192 80]
Err:4 http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease
  470  status code 470 [IP: 52.147.219.192 80]
Reading package lists... Done
N: See apt-secure(8) manpage for repository creation and user configuration details.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
E: The repository 'http://azure.archive.ubuntu.com/ubuntu jammy InRelease' is no longer signed.
E: Failed to fetch http://azure.archive.ubuntu.com/ubuntu/dists/jammy/InRelease  470  status code 470 [IP: 52.147.219.192 80]
N: See apt-secure(8) manpage for repository creation and user configuration details.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
E: The repository 'http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease' is no longer signed.
E: Failed to fetch http://azure.archive.ubuntu.com/ubuntu/dists/jammy-updates/InRelease  470  status code 470 [IP: 52.147.219.192 80]
E: Failed to fetch http://azure.archive.ubuntu.com/ubuntu/dists/jammy-backports/InRelease  470  status code 470 [IP: 52.147.219.192 80]
E: The repository 'http://azure.archive.ubuntu.com/ubuntu jammy-backports InRelease' is no longer signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: Failed to fetch http://azure.archive.ubuntu.com/ubuntu/dists/jammy-security/InRelease  470  status code 470 [IP: 52.147.219.192 80]
E: The repository 'http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease' is no longer signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.


```

I need to specify those, preferably by fqdns, to allow the access.

```go

  application_rule_collection {
    name = "SpokeAllowApps"
    priority = 500
    action = "Allow"
    rule {
      name = "AllowSpoke41toUbuntu"
      source_ip_groups = [module.testvnet4.VnetIpGroup.id]
      destination_fqdns = ["ubuntu.com","*.ubuntu.com"]

    }
  }

```

```bash

azureuser@spoke41-vm1:~$ sudo apt update
Hit:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Get:3 http://azure.archive.ubuntu.com/ubuntu jammy-backports InRelease [109 kB]
Get:4 http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Get:5 http://azure.archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [1282 kB]
Get:6 http://azure.archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [262 kB]
Get:7 http://azure.archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [1276 kB]
Get:8 http://azure.archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [208 kB]
Get:9 http://azure.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1032 kB]
Get:10 http://azure.archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [231 kB]
Get:11 http://azure.archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [42.1 kB]
Get:12 http://azure.archive.ubuntu.com/ubuntu jammy-backports/main amd64 Packages [41.7 kB]
Get:13 http://azure.archive.ubuntu.com/ubuntu jammy-backports/universe amd64 Packages [24.2 kB]
Get:14 http://azure.archive.ubuntu.com/ubuntu jammy-security/main amd64 Packages [1065 kB]
Get:15 http://azure.archive.ubuntu.com/ubuntu jammy-security/main Translation-en [201 kB]
Get:16 http://azure.archive.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [1248 kB]
Get:17 http://azure.archive.ubuntu.com/ubuntu jammy-security/restricted Translation-en [204 kB]
Get:18 http://azure.archive.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [831 kB]
Get:19 http://azure.archive.ubuntu.com/ubuntu jammy-security/universe Translation-en [158 kB]
Fetched 8445 kB in 2s (5306 kB/s)                           
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
11 packages can be upgraded. Run 'apt list --upgradable' to see them.
azureuser@spoke41-vm1:~$ 


```

Checking on Network Watcher, we can see that for the spoke with `internet_security_enabled` set to `false`, the next hop for the well known `8.8.8.8` IP is still using the system route, and thus the default Internet Egress configuration.

![illustration14](/assets/securehub/EgressRouteSecurityDisabled.png)

While for the spoke with `internet_security_enabled` set to `true`, the next hop for the same IP is the Firewall.

![illustration15](/assets/securehub/EgressRouteSecurityEnabled.png)

Interestingly enough, the `internet_security_enabled` parameter is set to true by default when the network connection is created from the portal.


#### 2.4.2. Ingress flows from Internet for SecurityEnabled Spoke

Now what happens if I want to access a VM in a spoke with Internet Security Enabled?
Well, to summarize, it does not work.
All in all, it seems logical, regarding the next hop configuration for Internet. If we try to enter the Vnet from the opened port on the NSG, the traffic comes back through the Firewall, hence, no access.

```bash

df@df2204lts:~/SSHKeys$ ssh -i spoke41-vm2_key.pem azureuser@20.102.118.165
ssh: connect to host 20.102.118.165 port 22: Connection timed out
df@df2204lts:~/SSHKeys$ 

```

There are some possible scenario to be more granular depending on the subnets, but we'll keep that for another time ^^.

#### 2.4.3. Managing different route tables

Ok, our last thing before wrapping it up is the management between different route tables.
That's something that is quite well documented in the Azure documentation in scenarios such as [Isolate Vnets custom scenario](https://learn.microsoft.com/en-us/azure/virtual-wan/scenario-isolate-vnets-custom).
In our case, we have some spokes that are considered as private, and thus ehind the firewall with a specific route table, and others, which are considered public, and use the default route table.

![illustration16](/assets/securehub/routingdifferenttable.png)

So the private spokes are...privates (except maybe the one with Internet Security deisbled, but that was for a demonstration purpose).
The public ones use the default routing and do not go through the firewall between them.
However, while the private spokes have route to those public spokes with the routes configured on the custom route table, 

```go
resource "azurerm_virtual_hub_route_table_route" "PrivateCIDRRoute" {
  route_table_id = azurerm_virtual_hub_route_table.CustomRouteTable.id

  name              = "PrivateCIDRToFW"
  destinations_type = "CIDR"
  destinations      = ["10.0.0.0/8","172.16.0.0/12","192.168.0.0/16"]
  next_hop_type     = "ResourceId"
  next_hop          = azurerm_firewall.fw_hub.id
}

```

![illustration17](/assets/securehub/routefromprivateotopublic.png)

The opposite is not true. Hence, problem of access.

![illustration18](/assets/securehub/routefrompublictoprivatenoconfig.png)


To solve that, we need to add a route to the private spokes on the default route table. 
But we cannot just propagate the route from the network connection, or we would not have the firewall in between.
And anyway, we already have the route in one way. Wejust need the route back.
So we add a route on the default route table for each private spokes.
It can be easily configured through a terraform sample like that : 

```go

resource "azurerm_virtual_hub_route_table_route" "routetospoke2" {
  route_table_id = format("%s%s%s", var.VirtualHubId, "/hubRouteTables/", "defaultRouteTable")

  name              = "routetospoke2"
  destinations_type = "CIDR"
  destinations      = module.testvnet2.VnetIpGroup.cidrs
  next_hop_type     = "ResourceId"
  next_hop          = var.FwId
}


resource "azurerm_virtual_hub_route_table_route" "routetospoke4" {
  route_table_id = format("%s%s%s", var.VirtualHubId, "/hubRouteTables/", "defaultRouteTable")

  name              = "routetospoke4"
  destinations_type = "CIDR"
  destinations      = module.testvnet4.VnetIpGroup.cidrs
  next_hop_type     = "ResourceId"
  next_hop          = var.FwId
}

```

Once applied, the previous network watcher shows us that we have a next hop configured as the firewall from public to private spokes 

![illustration19](/assets/securehub/routefrompublictoprivateconfig.png)


We still need to allow some flows to get things done : 

```go

    rule {
      name                  = "AllowSSHFromSpoke11Subnet1"
      protocols             = ["TCP"]
      source_ip_groups      = [module.testvnet.subnetIpGroups.Subnet1.id]
      destination_ip_groups = [module.testvnet2.subnetIpGroups.Subnet1.id]
      destination_ports     = ["22"]
    }
    rule {
      name                  = "AllowICMPFromSpoke41"
      protocols             = ["ICMP"]
      source_ip_groups      = [module.testvnet4.VnetIpGroup.id]
      destination_ip_groups = [module.testvnet2.VnetIpGroup.id]
      destination_ports     = ["*"]
    }
    rule {
      name                  = "AllowICMPFromSpoke11"
      protocols             = ["ICMP"]
      source_addresses      = module.testvnet.VnetIpGroup.cidrs
      destination_addresses = module.testvnet2.VnetIpGroup.cidrs
      destination_ports     = ["*"]

    }

```

```go

    rule {
      name                  = "AllowSSHFromSpoke11Subnet1"
      protocols             = ["TCP"]
      source_ip_groups      = [module.testvnet.subnetIpGroups.Subnet1.id]
      destination_ip_groups = [module.testvnet4.subnetIpGroups.Subnet1.id]
      destination_ports     = ["22"]
    }

    rule {
      name                  = "AllowICMPFromSpoke11"
      protocols             = ["ICMP"]
      source_addresses      = module.testvnet.VnetIpGroup.cidrs
      destination_addresses = module.testvnet4.VnetIpGroup.cidrs
      destination_ports     = ["*"]

    }

```

Once applied, the rules allow us access from public spoke to private spoke:

```bash

azureuser@spoke11-vm1:~/sshkeys$ ping -c 4 172.21.1.4
PING 172.21.1.4 (172.21.1.4) 56(84) bytes of data.
64 bytes from 172.21.1.4: icmp_seq=1 ttl=63 time=4.11 ms
64 bytes from 172.21.1.4: icmp_seq=2 ttl=63 time=3.63 ms
64 bytes from 172.21.1.4: icmp_seq=3 ttl=63 time=3.22 ms
64 bytes from 172.21.1.4: icmp_seq=4 ttl=63 time=2.29 ms

--- 172.21.1.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 2.291/3.312/4.107/0.667 ms

azureuser@spoke11-vm1:~/sshkeys$ ssh -i spoke2-vm1.pem azureuser@172.21.1.4
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-1053-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Jan 19 17:04:55 UTC 2024

  System load:  0.0               Processes:             113
  Usage of /:   5.7% of 28.89GB   Users logged in:       0
  Memory usage: 9%                IPv4 address for eth0: 172.21.1.4
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Fri Jan 19 17:03:47 2024 from 172.21.0.68
azureuser@spoke21-vm1:~$ 

```


Ok time to wrap up!

## Summary

Transforming a virtual hub to a secure hub implies adding a firewall. In our case, we did it with an Azure Firewall, for simplicity reasons mainly, so that we could create rules through the same terraform configuration.
Apart from the necessity of having a Firewall, we also need to configure the routing in the Virtual Hub so that the spokes to spokes traffic is routed to the Firewall. It's not very difficult once we have identified what to configure:
- The network connection to the hub with its routing parameter
- Additional routes in the default route table if we want to allow routing between public and private spokes

Also, we obviously need to configure rules on the firewall. We could spend more time just on that, but we've seen the basics with NEtwork rules and Applications rules.
We did it leveraging the IP Groups, Azure resources dedicated to Firewall policies.

We also checked the `internet_security_enabled` parameter on the network connection and what it means to set it to true or false:
- true means egress traffic goes through the Firewall, breaking at the same time direct public access from the spokes
- `false` means that egress traffic is still using the default route table of the Vnet, thus **not** going to the Azure Firewall.

There are still things that we should have a look at:
- We did not look **at all** to the logs that we can get on either the Azure Firewall or the NSGs
- Following the evolution of th eVirtual WAN, we did not eithe rlook at the routing intents

Which means that I'll probablyu come back for more Network stuff ^^

That being said, see you next time