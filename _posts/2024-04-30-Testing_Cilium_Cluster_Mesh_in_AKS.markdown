---
layout: post
title:  "AKS and Cilium 101"
date:   2024-04-30 18:00:00 +0200
year: 2024
categories: AKS Network
---

Hi there!

Last time, we discussed a bit about Cilium in an AKS environment.
But we let it at a point on which, well, we are far from having a complete picture. So in this article, I propose that we have a look at Cilium cluste rMesh featuer and see how it works and what it brings.

Our agenda will be as follow

1. Cluster Mesh concepts
2. Preparing the lab
3. Testing cluster mesh

Let's get started!

## 1. Cluster Mesh concepts

So what do we talk about here?
AS the name implies, we want to achieve a mesh between clusters so that we can get a global view of the hosted workload on our meshed clusters. Note by the way that there is an excellent blog article written on [Cilium blog](https://cilium.io/blog/2019/03/12/clustermesh/) that explain all of this.
Use cases  could be:

- High availability
- Shared services
- Splitting services by kind

That being said, how does it works?

Well first we have a bunch of prerequisites: 

- Cluster worker nodes should have 


## 2. Preparing the lab



## 3. Testing cluster mesh


## 4. Summary



