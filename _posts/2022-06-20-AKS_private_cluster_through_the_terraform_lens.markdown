---
layout: post
title:  "AKS private cluster through the terraform lens"
date:   2022-06-20 08:51:00 +0200
categories: AKS
---

Hi everyboy!

Today's topic is AKS in private mode.

In this article, we will take an interest to this deployment model, but rather than through a classic architecture concepts, we'll take it from the Infrastructure as Code point of view.

At the end of those lines, we will have gone through the differents requirements for having a private AKS cluster.

## Table of content

1. Kubernetes Seurity principles in 2 minutes
2. Translating Kubernetes Security in Azure
3. From the Infrastructure as code point of view
4. Before leaving


## 1. Kubernetes Seurity principles in 2 minutes

Let's start by a very high level view of kubernetes architecture. Just to be on the same page.

![Illustration 1](/assets/pvaks/pvaks001.png)

If we summarize, we have to consider the control plane, which as the name implies, is reposnible for controlling whatever happens in the worker plane, where live the applications.

It is really very simlified but that'a about it.

Now let's take from [kubernetes documentation](https://kubernetes.io/docs/concepts/security/overview/
) a description of what need to be secured for a kubernetes based architecture:  

Security topics for kubernetes are divided in 4 categories, called the 4 C: 

- The Cloud
- The cluster
- The container
- The code

![Illustration 2](/assets/pvaks/pvaks002.png)  
  
TYpically, we could say that the first 2 Cs are more infrastructure oriented. Considering actual topics for those we get the following:  

| Cloud area security topic |
|-|
| Network access to API Server |
| Network accerss to nodes |
| Kubernetes access to Cloud provider API|
| Access to ETCD |
| ETCD Encryption |

| Cluster area security topic |
|-|
| RBAC Authorization |
| Authentication |
| Application secrets management |
| Pod Security Standards |
| Network Policies |
| TLS for Kubernetes Ingress |

Which does make a lot of topics. But today, we are talking about AKS private cluster which is the Azure answer to the first topic **Network access to API Server**.

That's right, only this tiny little topic today.

Ok, now that we have define the foundations, let's focus on what we have considering Azure and AKS.

## 2. Translating K8S Security in Azure

Starting from our last kubernetes representation, we get something like below in Azure:  

![Illustration 3](/assets/pvaks/pvaks003.png)  

As we can see on this (yet again) very simplified schema, AKS is PaaS. As a majority of PaaS, it is by design exposed in a public DNS namespace. In AKS case `<something>.<azureregion>.azmk8s.io`.

Now the main discussion point is this *public* access. For the fluidity of the discussion heren, let's just say that we want a way to avoid public exposition of the control plane.
We have a few option. The simplest being to use an accept list on the API server. Thus, only known and defined public IP can access to the API server, which is the part that we access on the control plane.

But in some case, a simple accept list is not acceptable because, well, it does not follow a regulation.

So this time, if we want to still use a PaaS but in a private network **only**, we will have to rely on [Azure Private Endpoints](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview). The private endpoint is Microsoft answer for the need of private PaaS and similar to a kind of NAT.

Technically, a Network Interface Card is connected in a Virtual Network and NATed to the PaaS instance through [Private Link](https://docs.microsoft.com/en-us/azure/private-link/private-link-service-overview). This NIC thus gets a private IP from the VNet range. To be able to resolve the PaaS instance name, there is  the need for an Azure Private DNS zone which will register the PaaS instance with its private IP. From an Azure Network standpoint, all virtual network that need to resolve the DNS record for the PaaS instance need to be linked to the Azure Private DNS zone containing this record.

Also, in the meantime, the public fqdn is disabled and makes it impossible to reach the PaaS instance from anywhere on the Internet. We have to be **inside** the private network to reach the PaaS instance.

Just to be clear, Private Endpoint is not limited to AKS, even if we will only look at how to put tha tin place in the following part.

So to summarize all that, we get a schema looking like that: 

![Illustration 4](/assets/pvaks/pvaks004.png)  
