---
layout: post
title:  "Workload Identity in AKS"
date:   2023-04-15 23:00:00 +0200
year: 2023
categories: AKS Security
---

Hey people!

It's been long overdue. Here is an article about Workload Identity in an AKS context.
The agenda:

- Before Workload Identity, the need and the solution
- Workload Identity concepts
- Implementing Workload Identity in AKS

## 1. Before Workload Identity

So, workload Identity have been around for quite some time now, but is still currently in preview. It was planned to go on GA in early april, but was finally pushed away to later in the month.
That being said, let's step back a little and try to understand the need behind the solution.

If we consider microservices, there are usually interaction, even if in a loose coupled configuration, between the differents part of an application (a.k.a microservices). At some point, to have a secured application, we would like to have each service proving who it said it is, which is the process of authentication. And to have a working environment, the authenticated service should have authorizations defined.

**schemaauthenticatedservices**

Now let's consider Azure. 
In the Azure plane, we have a way to authenticate and authorize a service to do things in Azure, or in an Azure AD integrated Azure service with managed Identity.

**managed identity shema**

And it works fine. No creds to manage, so in theory no risk of finding a connection string or a secret hardcoded in the application code.

But what happen if we consider containers and as such AKS?

Well, first, we must remember that container are, by design, isolated processes, or services running on a shared kernel. Even if the isolation is less complete than a full virtual OS (but also les heavy and thus way more fast), there is still an isolation, and the Azure platform cannot easily see what's happenind inside a container, or in AKS case a pod.

The first answer to this property of the container / pod which was in this case an issue was something called pod identity.
With pod identity, we introduced in AKS a corresponding objet in the Kubernetes plane to an Azure managed identity in the Azure plane.

**schema juxta managed identityazure/k8s**

A custom resource definition allowed us to  create a managed identity object in the kubernetes plane and was associated to a pod through a managed identity binding. You know, kind of the same way as for services and labels, but for identities.
A daemonset ensured that a pod was running at all time on each node to intercept the Authentication request, and talked with another set of ppods organized in a deployement that itself was checking Azure AD for authentication, and authorization.

**AADPodIdentity schema**

While looking elegant in terms of Kubernetes approach (you know, because everybody does CRDs ^^), the underlying architecture was limited in many aspect, and it was not developed to a v2 version.

Instead, pod identity v2 was redevelopped, and called workload identity. 

## 2. Workload IDentity concepts

Now that the little history is reminded, let's focus on how workload identity works.
A better explanation should be to look at the scope of workload identity.
Outside of a pure Kubernetes scope (Yup, not only AKS, which was one of the limit of pod identity v1 btw), workload identity is defined at the Azure Active Directory level.
There is a dedicated section on the [Microsoft Entra documentation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/) about it.
It describes the different kind of eligible principals in Azure AD that can be used:

- Managed Identities
- Application registrations

And also the concept of workload identity federation. 
That's right, this is really about federating and Identity provider and Azure AD through either an application registration or a managed identity.
Obviously managed identities only work in an Azure context, and non-Azure environment must relies on Application registration.
But here's the interesting part.
By establishing a workload identity federation, we configure the Azure AD security principals to trust token issued from an external IdP.
The beauty of it? 
No need to share a secret or a certificate for external system & application registration scenario.
And definitely not for Managed identity, as it should be.

**schema federated worklod identity**
