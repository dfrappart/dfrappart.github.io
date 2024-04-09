---
layout: post
title:  "AKS Networking considerations - part 1"
date:   2023-10-31 18:00:00 +0200
year: 2023
categories: AKS Network
---

Hi!

Recently, I was invited to talk about AKS networking options.

It's not easy to get started with AKS if you"re not familliar with Azure Network in general and AKS (and hence Kubernetes) in particular.
While preparing this talk, I was reminded of that and well, I thought it would be a good idea to write that down.
So that's why you'll get another AKS article, dedicated to network.

Hope this help to make thins more clear.

Our agenda will be as follow for the part 1: 

- High level overview of AKS architecture
- Network considerations for the control plane

In part 2 we'll look at:

- Network considerations for the worker plane
- Subnets utilizations

Let's get start with this part 1.


## 1. High level overview of AKS architecture

On its most simple description, we could summarize Kubernetes architecture to 2 blocks: 

- The control plane, which controls everything and with which Kubernetes admins interact for everything
- The worker plane which hosts workloads on its nodes through command issued by the control plane


![illustration1](/assets/aksntwconsiderations/k8sarchitecture.png)  

But we're talking about **Azure** Kubernetes service right, so it becomes more or less something like this:

![illustration2](/assets/aksntwconsiderations/AKSarchitecture.png)

The control plane, and specifically, the API server, is in the Azure PaaS network. Which means it's publicly exposed and reachable through fqdns that are Microsoft managed.
The worker plane is akeen to IaaS resources, in the form of scale sets for nodes and NSGs. There are also managed identities (but not representd here, because we talk about network), and load balancers. 
There is also a Virtual network which is either managed by AKS, with a lifecycle that would be linked to the cluster in this case, or self-managed and prepared before the creation of the cluster.

There's much, but for what we want to talk about, that's more than enough. So let's dive in our topic now.

## 2. Network considerations for the control plane

### 2.1. Network concepts for Azure PaaS

As we've juste said, AKS control plane is a PaaS instance. So, similarly to most of PaaS instance, we get some basic network configuration through the portal or the API to managed a minimum level of Network filtering.
Typically, PaaS instance can be protected with some firewall rules for Public IP accept list, or in some case, Virtual network rules.

![illustration3](/assets/aksntwconsiderations/PaaSNetwork.png)

Which means that we can control the access of an Internet client from the public IP accept list, or an Azure hosted client (in a Virtual Network) from the Virtual Network rules. This ones relies on the [Azure Service Endpoints(https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)], which we won't dive in today. 

Interestingly, it could be possible to have a public IP to allow the access from the Virtual network hosted client but it would require to take into account the nature of egress traffic from a virtual network.
Remember, without any need for additional configuration, an Azure virtual machine can go on the Internet. But in this case, it uses a public IP taken in a regional pool. 
Let's say it's not really ideal to add a full region pool of IPs to allow traffic. 
So the other option is be to ensure that the vm get on the Internet with a know public IP. For that we have options such as Azure NAT Gateway, or Azure Firewall. But again,not our topic today. Finally, service endpoints allow, when they are available, to keep the path more private than through a public IP so why bother?

Moving on, usually, accept lists are not considered secure enough. And service endpoints bring other limitations, including that the PaaS instance is still connected to the Azure Public namespace. To answer the need for private PaaS, Microsoft designed the Private link solution and specifically for PaaS, Private endpoint.

Private Endpoint allows to change the DNS management of a PaaS instance. Instead of being managed in Microsoft public DNS, it's registered in a private DNS zone. 
From a Network perspective, the connection to the public Network is deactivated and a new Network Interface is created inside a virtual network. 
This Nic gets a private IP which is registed on the privated DNS zone. 
The solution is quite elegant, but introduce some complexities with the DNS configuration, even more if we talk about hybrid DNS. Please refer to the [Private endpoint documentation](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) and the [integration with DNS](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) for more details, because we want to focus on AKS here ^^

![illustration4](/assets/aksntwconsiderations/PrivateLink.png)

Now that we've tackled the basics, let's look at how it translate in AKS configuration.

### 2.2. API Server accept list

First thing first, AKS control plane does not allow us to configure vnet rules. We only get accept list.

![illustration5](/assets/aksntwconsiderations/aksapiacceptlist.png)

Let's illustrate this with a cluster.

We can start by checking the fqdn of the cluster and that we do get a public IP when we try to resolve it: 

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo1 -g aksntwdemo -o table

Name          Location    ResourceGroup    KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
------------  ----------  ---------------  -------------------  --------------------------  -------------------  ----------------------------------------------------------
aks-ntwdemo1  eastus      aksntwdemo       1.26.6               1.26.6                      Succeeded            aks-ntwdem-aksntwdemo-16e85b-6t70z79x.hcp.eastus.azmk8s.io

yumemaru@azure:~$ nslookup aks-ntwdem-aksntwdemo-16e85b-6t70z79x.hcp.eastus.azmk8s.io
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
Name:	aks-ntwdem-aksntwdemo-16e85b-6t70z79x.hcp.eastus.azmk8s.io
Address: 52.226.4.150


```

This cluster is configured with a publicly accessible API server as shown on the printscreen:

![illustration6](/assets/aksntwconsiderations/apipublic001.png)

With kubectl configured, we can access the API and get information: 

```bash

yumemaru@azure:~$ k config use-context aks-ntwdemo1 
Switched to context "aks-ntwdemo1".

yumemaru@azure:~$ k get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-13348451-vmss000004   Ready    agent   12m   v1.26.6


```

We can easily configure the accept list from the portal in the networking configuration

![illustration6](/assets/aksntwconsiderations/apipublic002.png)

After setting an IP, we can test to access again. If the remote IP is not in the range, we'll get a timeout.

```bash

yumemaru@azure:~$ k get nodes
Unable to connect to the server: dial tcp 52.226.4.150:443: i/o timeout


```

Note that it's also possible to check the API Server profile from the az cli, once it's configured: 

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo1 -g aksntwdemo | jq .apiServerAccessProfile

{
  "authorizedIpRanges": [
    "1.1.1.1/32"
  ],
  "disableRunCommand": null,
  "enablePrivateCluster": null,
  "enablePrivateClusterPublicFqdn": null,
  "enableVnetIntegration": null,
  "privateDnsZone": null,
  "subnetId": null
}


```

If the remote IP is in the accept list, obviously, we can reach the API:

```bash

yumemaru@azure:~$ curl ifconfig.me
81.xxx.yyy.zzz

yumemaru@azure:~$ az aks update -n aks-ntwdemo1 -g aksntwdemo --api-server-authorized-ip-ranges "81.xxx.yyy.zzz"

yumemaru@azure:~$ az aks show -n aks-ntwdemo1 -g aksntwdemo | jq .apiServerAccessProfile

{
  "authorizedIpRanges": [
    "81.xxx.yyy.zzz/32"
  ],
  "disableRunCommand": null,
  "enablePrivateCluster": null,
  "enablePrivateClusterPublicFqdn": null,
  "enableVnetIntegration": null,
  "privateDnsZone": null,
  "subnetId": null
}
yumemaru@azure:~$ k get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-13348451-vmss000004   Ready    agent   70m   v1.26.6


```

Ok that works, but there's a chance that it won't be considered as enough, so let's go further with the AKS private cluster.

### 2.3. AKS private cluster

![illustration7](/assets/aksntwconsiderations/aksprivateendpoint.png)

Remember, we described, very rapidly, the private endpoint concepts earlier.
If you get the impression that it's a mess, well, that's not entirely wrong.
Again, the main principle for private endpoint is the DNS aspect. 
Instead of a public fqdn with a public IP, we get a private fqdn in a private DNS zone, managed in our tenant.
Logically, the routing changes and we secure the access to the API cluster because it's not publicly reachable at all.

In AKS, we have no less than 3 differents way to tackle the private cluster.
In the first option, we bring our own private DNS zone and we tell the cluster, at the creation time, that we want it to be private and to use the DNS zone that we specify.
Because the DNS needs to be created beforehand, there is more planning to do, but it also ensure that we can resolve our clusters names in hybrid environment, because all requests can be forwarded to this zone.
In the second option, we let the cluste rcreate its own Private DNS zone. The private endpoint is registered on this zone in the same way as the first option. However, hybrid scenario are much more complicated (if not impossible) because we have 1 zone per cluster and no way to forward requests to all those zones. It ideal for environments in which the cluster access can be local to the virtual network only.

Ok that makes 2, what about the 3rd?  Well, it's a kind of hybrid private cluster (toally personal naming here, don't use it ^^).
The idea, with this third proposal, i that it's the DNS that is a pain to managed.
So instead of managing DNS zone, we configure our private cluster but let the DNS resolution be managed in Microsoft managed DNS zone.

Last but not least, we don't manage private endpoint the same way as for others PaaS. AKS control plane manages the creation on its own (with some consequences if we consider the BYO DNS...)

Let's have a look at a sample cluster:

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo2 -g aksntwdemo -o table

Name          Location    ResourceGroup    KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
------------  ----------  ---------------  -------------------  --------------------------  -------------------  ----------------------------------------------------------
aks-ntwdemo2  eastus      aksntwdemo       1.26.6               1.26.6                      Succeeded            aks-ntwdem-aksntwdemo-16e85b-t0xkxtuj.hcp.eastus.azmk8s.io

```

We can see from the network section that the cluster is private, but the fqdn is still public:

![illustration8](/assets/aksntwconsiderations/privatecluster001.png)

A query through the az cli with an nslookup confirm this:

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo2 -g aksntwdemo | jq .apiServerAccessProfile

{
  "authorizedIpRanges": null,
  "disableRunCommand": null,
  "enablePrivateCluster": true,
  "enablePrivateClusterPublicFqdn": true,
  "enableVnetIntegration": null,
  "privateDnsZone": "none",
  "subnetId": null
}

yumemaru@azure:~$ nslookup aks-ntwdem-aksntwdemo-16e85b-t0xkxtuj.hcp.eastus.azmk8s.io
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
Name:	aks-ntwdem-aksntwdemo-16e85b-t0xkxtuj.hcp.eastus.azmk8s.io
Address: 10.224.0.4

```

We cannot interact with the cluster except if we try from a network connected to the private endpoint which is the aim.

That was an example of private cluster with public fqdn, let's have a look at a cluster with a manged DNS zone. The BYODNS implies additional planning so we'll forget it for this time.

To create a private cluster with managed DNS, ti's quite easy, we jusge have to let the cluster create it at the provisioning time. The az cli command would look like this: 

```bash

yumemaru@azure:~$ az aks create -n aks-ntwdemo11 -g aksntwdemo  --enable-private-cluster --node-count 1

```

To detect if the DNS zone ismanaged by the cluster, look in the associated resource group that is shown in the properties section:

![illustration9](/assets/aksntwconsiderations/privatecluster002.png)

If there's a DNS zone inside, it's a private cluster with managed DNS zone.
We will note that the cluster has an fqdn and a private fqdn with different values:

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo11 -g aksntwdemo | jq '.fqdn,.privateFqdn'
"aks-ntwdem-aksntwdemo-16e85b-1ulw5vjo.hcp.eastus.azmk8s.io"
"aks-ntwdem-aksntwdemo-16e85b-zkdf7afu.3b7ca984-03df-44f2-bfb3-e2234ff2c58f.privatelink.eastus.azmk8s.io"

```

Which is not the case for a public cluster:

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo1 -g aksntwdemo | jq '.fqdn,.privateFqdn'

"aks-ntwdem-aksntwdemo-16e85b-6t70z79x.hcp.eastus.azmk8s.io"

```

Or a private cluster with public fqdn, for which the values are the same:

```bash

yumemaru@azure:~$ az aks show -n aks-ntwdemo2 -g aksntwdemo | jq '.fqdn,.privateFqdn'

"aks-ntwdem-aksntwdemo-16e85b-t0xkxtuj.hcp.eastus.azmk8s.io"
"aks-ntwdem-aksntwdemo-16e85b-t0xkxtuj.hcp.eastus.azmk8s.io"

```

Looking at the DNS zone, we can see the record for the cluster's API server:

![illustration10](/assets/aksntwconsiderations/privatecluster003.png)

To managed that through terraform, have a look at this [older article]() on my blog ^^

Ok, that's it for the private cluster. Let's look at the last network configuration option for the API server.

### 2.4. API Server Vnet integration

As the name implies, for this last option, it's the API server that is directly integrated in a subnet. This subnet has to be [delegated](https://learn.microsoft.com/en-us/azure/virtual-network/subnet-delegation-overview).
It's important to notice that the feature is still currently in preview.

![illustration11](/assets/aksntwconsiderations/aksvnetintegratedapiserver.png)

Now the strange thing with this option is that the cluster is not necessarily private because the API server is inside a subnet. 
Which is strang imho, but looking at the documentation, we can find in the [documentation](https://learn.microsoft.com/en-us/azure/aks/api-server-vnet-integration) that the API server remains accessible except if the cluster is specified as private.
So what do we get if we still need to managed the private cluster configuration with the same DNS options?
Well as displayed on the schema, the communication between the control plane and the worker plane remains inside the same virtual network. Which means that there is indeed a better security. On the other hand, it does not mean that the communication is unsecure in the other API server configuration. In this case, the communication use konnectivity as a proxy between the API server and the worker plane. Also, for private cluster, the endpoind of the API server is a private endpoint also private. But that's a Nic inside the Vnet connecting the API server whic is still... somewhere else.

At any rate, API server vnet integration is not the same as a private API server. Creating a cluster with this configuration would look like this: 

```bash

yumemaru@azure:~$ az aks create -n aks-ntwdemo05 -g aksntwdemo  --enable-private-cluster --node-count 1 --enable-apiserver-vnet-integration

```

And we would get the API server dedicated subnet looking like that: 

![illustration12](/assets/aksntwconsiderations/vnetintegration01.png)

![illustration13](/assets/aksntwconsiderations/vnetintegration02.png)

Note: It's possible to specify a subnet beforehand with the parameter `--apiserver-subnet-id`

And that will be all for the Control plane and thus part 1 of this article

See you soon for part 2 of this series for AKS networking considerations ^^
