---
layout: post
title:  "Sandbox containers and AKS"
date:   2023-03-18 18:00:00 +0200
year: 2023
categories: AKS Security
---

Hello All!

In this article, I would like to talk about sandbox container, in Azure Kubernetes Service context.

We will start by introducing the concepts behind sandbox container, why it may be needed. 
Then we'll look at the currently available solution onr sandbox container in Kubernetes.
And last, we'll go pragmatics and demonstrate how the sandbox container tehnologies are deployed and used in an AKS cluster.

## 1. Container sandboxing 101

Before understanding the need of sandbox containers, we need to step back a little and look at the basics of container architecture.

Remember, the idea behind container is to have a lighter abastraction layer btween the app and its hosts.
In the virtualization world, we have the host, the hypervisor, and the virtual machine which then host the app binary.

![illustration1](/assets/katacontainer/sbxcontainer001.png)

In the container world, the hypervisor disappears, along with the Virtual Machine layer and its OS.

![illustration2](/assets/katacontainer/sbxcontainer002.png)

The result is what everyone knows, it's way more fast and resource efficient to have those layers removed and depends only on the container engine and the kernel capabilities to isolate workloads.

However, because there is so much less layer, there is also less isolation. The kernel is shared, even if there is isolation by the mean of cgroups and namespace. 

Because there are some case where a better isolation is needed, technical solutions were developped. 
Those solutions are usually based on one of the 2:

- Rule-based execution
- Machine level virtualization

Rule-based execution is used by identifying which syscall are made by the app, and by allowing only those calls from the container to the shared host kernel. Solutions using Rule-based execution are seccomp, SELinux or AppArmor.

Machine level virtualization is probably easier because it provides an isolation through a light-weight virtual machine.

![illustration3](/assets/katacontainer/sbxcontainer003.png)

While Machine level virtualization relies on light-weight virtual machine, it does impact the performance of the application that is hosted in this sandboxed container.

In the Machine level virtualization category, we find [katacontainer](https://katacontainers.io/docs/), on which we will have a look at in the following part.

To conclude with the sandbox container solutions, we should have a look at [gVisor](https://gvisor.dev/docs/).
gVisor acts as an intermediary between the application and the host kernel. It intercepts the syscall from the application and pass only limited calls to the host kernel.

![illustration4](/assets/katacontainer/sbxcontainer004.png)

Ok that's all for the intro on sandbox containers. In the following sections, we will get deeper in the Rule-based execution and focus more specifically on katacontainer and gVisor, in an AKS context.

## 2. Sandbox containers in AKS

Let's go back to our cloud managed Kubernetes now.
If we look at either gVisor or katacontainer, there are things to install on the kubernetes nodes.
Except that, well, we usually don't install anything on the nodes.
Indeed, those nodes are actually instances from vm scale sets. By design, the scale sets are managed by the control plane of AKS, and the instances lifecycle is not managed by a human admin. Because of this, it is not really practicle to install any binary on a node each time it is provisioned.

There's a way, though, to have a pod running on each nodes. For use case like that, we can rely on the daemonset controller, which will ensure that we have the pods described in the controller always deployed on each nodes of our cluster. 

![illustration6](/assets/katacontainer/sbxcontainer006.png)

If we consider an AKS cluster, we can use daemon sets along with taints on node pools to have pods running on each sets of nodes in a node pools.

![illustration5](/assets/katacontainer/sbxcontainer005.png)

If we want to install gVisor on all the nodes of a specific node pool, we could rely on a daemonset which would deploy a pod with an elevated container. If this said container was configured to execute the installation of gVisor, then we could achieve our goal.
Note that there is a very nice article from [Daniel Neumann](https://www.danielstechblog.io/about-me/) on [his blog explaining this approach](https://www.danielstechblog.io/running-gvisor-on-azure-kubernetes-service-for-sandboxing-containers/).

**schema pod installing gvisor**

there are a few watch points here however.

First, we need to have a pod with elevated access, in this case access to the node local storage at least, so that we could deploy sandboxed container.
There may be a contradiction here, don't you think? 
Second, we need to modify the configuration of nodes managed by an Azure service. So while the approach wit the daemonset is typical of kubernetes environment and thus respect the best practice of kubernetes node management, it is kind of a grey zone in terms of Azure support. Meaning it should work but if it does not anymore, we're on our own because there won't be any support from Microsoft, or at least, not that much.

Another way for sandbox container, this time supported by Microsoft, is to rely on a sandbox technology available on the underlying OS of the node image.
Luckily for us, this is the case for katacontainer and the node image based on [Mariner](https://microsoft.github.io/CBL-Mariner/docs/#cbl-mariner-linux).
In this specific case, no installation on the nodes is required, and we can only focus on the kubernetes part of creating our sandbox container.