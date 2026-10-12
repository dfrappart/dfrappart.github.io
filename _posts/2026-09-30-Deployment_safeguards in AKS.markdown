---
layout: post
title:  "Deployment Safeguards in AKS"
date:   2026-09-30 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Security
---

Hello again,

Today a short article about Deployment Safeguards in AKS.
It's something I really came accross by happenstance, while looking for something on the Terraform doc (Yes I still parse documentation manually, I've always been fond of reading &#x1F937;&#x1F3FB;&#x200D;&#x2642;&#xFE0F;)

I found it interesting so I'll talk about it today ^^.


The agenda:


1. Deployment Safeguards overview
2. Using deployment safeguards
3. AKS Deployment safeguards vs Native PSA management

## 1. Deployment Safeguards overview

Deployment safeguards is a feature available in AKS to enforce best practices. As the [documentation](https://learn.microsoft.com/en-us/azure/aks/deployment-safeguards) states it, it works with Azure policy controls. This is important to consider and we'll see a bit more of this along this article.

In terms of those best practices, it allows to enforce controls equivalent to the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)/[Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/), but not only, as we can see in the below table.

| Deployment safeguard policy | Mutation outcome if available |
|-|-|
| Cannot Edit Individual Nodes| N/A |
| Kubernetes cluster containers CPU and memory resource requests must be defined | Sets default CPU and memory requests and enforces minimums. For more information, see Resource requests mutator. |
| Must Have Anti Affinity Rules or topologySpreadConstraintsSet | Adds pod anti-affinity rules and topology spread constraints to improve workload distribution. For more information, see Anti-affinity and topology spread mutator. |
| No AKS Specific Labels | N/A |
| Kubernetes cluster containers should only use allowed images | N/A |
| Reserved System Pool Taints | Removes the CriticalAddonsOnly taint from a user node pool if not set. AKS uses the CriticalAddonsOnly taint to keep customer pods away from the system pool. This configuration ensures a clear separation between AKS components and customer pods and prevents eviction of customer pods that don't tolerate the CriticalAddonsOnly taint. |
| Ensure cluster containers have readiness or liveness probes configured | N/A |
| Kubernetes clusters should use Container Storage Interface (CSI) driver StorageClass | N/A |
| Kubernetes cluster services should use unique selectors | N/A |
| Kubernetes cluster container images should not include latest image tag | N/A |





## 2. Using deployment safeguards





## 3. AKS Deployment safeguards vs Native PSA management





## 4. Summary

