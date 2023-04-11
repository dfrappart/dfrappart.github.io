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

![illustration1](/assets/workloadid/workloadidentityauthsvc.png)

Now let's consider Azure. 
In the Azure plane, we have a way to authenticate and authorize a service to do things in Azure, or in an Azure AD integrated Azure service with managed Identity.

![illustration2](/assets/workloadid/workloadidentityuai.png)

And it works fine. No creds to manage, so in theory no risk of finding a connection string or a secret hardcoded in the application code.

But what happen if we consider containers and as such AKS?

Well, first, we must remember that container are, by design, isolated processes, or services running on a shared kernel. Even if the isolation is less complete than a full virtual OS (but also les heavy and thus way more fast), there is still an isolation, and the Azure platform cannot easily see what's happenind inside a container, or in AKS case a pod.

The first answer to this property of the container / pod which was in this case an issue was something called pod identity.
With pod identity, we introduced in AKS a corresponding objet in the Kubernetes plane to an Azure managed identity in the Azure plane.

![illustration3](/assets/workloadid/workloadidentitypodid.png)

A custom resource definition allowed us to  create a managed identity object in the kubernetes plane and was associated to a pod through a managed identity binding. You know, kind of the same way as for services and labels, but for identities.
A daemonset ensured that a pod was running at all time on each node to intercept the Authentication request, and talked with another set of pods organized in a deployement that itself was checking Azure AD for authentication, and authorization.

While looking elegant in terms of Kubernetes approach (you know, because everybody does CRDs ^^), the underlying architecture was limited in many aspects, and it was not developed to a v2 version.

Instead, pod identity v2 was redevelopped, and called workload identity. 

## 2. Workload Identity concepts

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
As described roughmly on the schema below, an application, called a workload in this case, check with a local idp to get a token.
This token is known only from the local idp though, and the workload then send this token to the little part of AAD that is federated which is the secrity principal, an app reg or a managed identity. However, since this identity is federated with the local idp, it checks this local idp for verification of the token. 
If said token is true, then we can move on the authorization part, in which we rely on the role assignment associated to the AAD security principal. That means the security principal in AAD should have the corresponding authorization of the said workload. If not, when sending back the request access token to the workload, it wouldn't grant any permissions. The nice part in that as anyone can see is the granularity of this federation, scope on one security principal only.

![illustration5](/assets/workloadid/workloadidentityfederation.png)

That's it for the concepts onf workload identity. Now let's translate that in an AKS (and really a kubernetes) point of view.

## 3. Implementing Workload Identity in AKS

If we take the previous schema in a Kubernetes context, we roughly get this:

![illustration6](/assets/workloadid/workloadidentityfederationk8s.png)

Kubernetes comes with an OIDC url since version `1.20`. Which means that kubetnetes is the de-facto oidc provider acting kind of like the we are refering to in our schema.
Then we have the workload in itself, which is composed of a pod (or rather pods in any kind of desired controller, beut let's keep it simple for now ^^) and a service account associated to this pod.
The service account is the part that get the token and that will allow the request token in Azure AD to be used for access required by the pod.
On the overview, it's way more simple (and way more elegant, even for CRDs lovers) than the previous pod identity. Indeed, only basic kubernetes objects are required.