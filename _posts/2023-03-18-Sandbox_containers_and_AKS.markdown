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

Ok that's all for the intro on sandbox containers. In the following sections, we will let the Rule-based execution apart and focus more specifically on katacontainer and gVisor, in an AKS context.

## 2. Preparing for sandbox containers in AKS

Let's go back to our cloud managed Kubernetes now.
If we look at either gVisor or katacontainer, there are things to install on the kubernetes nodes.
Except that, well, we usually don't install anything on the nodes.
Indeed, those nodes are actually instances from vm scale sets. By design, the scale sets are managed by the control plane of AKS, and the instances lifecycle is not managed by a human admin. Because of this, it is not really practicle to install any binary on a node each time it is provisioned.

There's a way, though, to have a pod running on each nodes. For use case like that, we can rely on the daemonset controller, which will ensure that we have the pods described in the controller always deployed on each nodes of our cluster. 

![illustration6](/assets/katacontainer/sbxcontainer006.png)

If we consider an AKS cluster, we can use daemon sets along with taints on node pools to have pods running on each sets of nodes in a node pools.

![illustration5](/assets/katacontainer/sbxcontainer005.png)

If we want to install gVisor (or any binary as a matter of fact) on all the nodes of a specific node pool, we could rely on a daemonset which would deploy a pod with an elevated container. If this said container was configured to execute the installation of gVisor, then we could achieve our goal.

**schema pod installing gvisor**

There are a few watch points here however:

- First, we need to have a pod with elevated access, in this case access to the node local storage at least, so that we could deploy sandboxed container.
There may be a contradiction here, don't you think? 
- Second, we need to modify the configuration of nodes managed by an Azure service. So while the approach wit the daemonset is typical of kubernetes environment and thus respect the best practice of kubernetes node management, it is kind of a grey zone in terms of Azure support. Meaning it should work but if it does not anymore, we're on our own because there won't be any support from Microsoft, or at least, not that much.

There's an excellent article detailing this configuration, written by [Daniel Neumann](https://www.danielstechblog.io/about-me/) on his [blog](https://www.danielstechblog.io/running-gvisor-on-azure-kubernetes-service-for-sandboxing-containers/) which details the configuration of said container and how to deploy It with a daemonset. We'll reviw that a bit in the next section of this article.

Another way for sandbox container, this time supported by Microsoft, is to rely on a sandbox technology available on the underlying OS of the node image.
Luckily for us, this is the case for katacontainer and the node image based on [Mariner](https://microsoft.github.io/CBL-Mariner/docs/#cbl-mariner-linux).
In this specific case, no installation on the nodes is required, and we can only focus on the kubernetes part of creating our sandbox container.

If we dive a little deeper on our AKS architecture, we should remebmer that by default we have 1 node pool (a.k.a the default node pool). This node pool is by default (for now at least) an Ubuntu based node pool so no katacontainer there.
Also, because it's the default system node pool, it hosts all the AKS required pods for AKS to work. It's better to leave it alone, so we'll use additional node pool.

Regarding gvisor, it can be any node pool because this is self-managed sandbox software install, so there is no underlying architecture requirement.
Adding a node pool is not too difficult and can be done from the portal, az cli or some terraform configuration.

```bash

module "Nodepoolgvisor" {

  for_each                              = var.TrainingConfig
  source                                = "github.com/dfrappart/Terra-AZModuletest//Custom_Modules/IaaS_AKS_NodePool?ref=aksnpv1"

  AKSSubnetId                           = azurerm_subnet.subnet[each.key].id
  NPSuffix                              = "gvisor"
  AKSClusterId                          = module.AKS[each.key].FullAKS.id
  AKSNodeTaints                         = ["gvisor=true:NoSchedule"]

}

```

```bash

resource "azurerm_kubernetes_cluster_node_pool" "AKSNodePool" {

    lifecycle {...}

##############################################################
# Basic configuration

  name                                  = "np${var.NPSuffix}"
  kubernetes_cluster_id                 = var.AKSClusterId
  vm_size                               = var.AKSNodeInstanceType
  zones                                 = var.AKSAZ
  mode                                  = var.NPMode
  orchestrator_version                  = var.KubeVersion
  max_pods                              = var.AKSMaxPods
  workload_runtime                      = var.WorkloadRuntimeType
  message_of_the_day                    = var.MessageofTheDay 

##############################################################
# Security Parameters

  custom_ca_trust_enabled               = var.IsCustomCATrustEnabled
  enable_host_encryption                = var.EnableHostEncryption
  fips_enabled                          = var.IsFipsEnabled

##############################################################
# Network configuration

  enable_node_public_ip                 = var.EnableNodePublicIP 
  vnet_subnet_id                        = var.AKSSubnetId
  pod_subnet_id                         = var.PodSubnetId
  node_network_profile {
    node_public_ip_tags                 = var.NodePublicIpTags
  }

##############################################################
# Autoscaling configuration

  enable_auto_scaling                   = var.EnableAKSAutoScale 
  max_count                             = var.MaxAutoScaleCount
  min_count                             = var.MinAutoScaleCount
  node_count                            = var.AKSNodeCount
  scale_down_mode                       = var.ScaleDownMode

##############################################################
# OS related Configuration

  os_sku                                = var.AKSNodeOSSku
  os_type                               = var.AKSNodeOSType
  os_disk_type                          = var.AKSNodeOSDiskType
  os_disk_size_gb                       = var.AKSNodeOSDiskSize

##############################################################
# Nodes taints and Labels management

  node_labels                           = var.AKSNodeLabels
  node_taints                           = var.AKSNodeTaints  

##############################################################
# Node pool spot configuration

  priority                              = var.AKSNPPriority
  eviction_policy                       = var.EvictionPolicy
  spot_max_price                        = var.SpotMaxPrice 

##############################################################
# Node pool reservation configuration  
  
  capacity_reservation_group_id         = var.CapacityReservationGroupId

##############################################################
# Host and prximity configuration

  proximity_placement_group_id          = var.PlacementGroupId
  host_group_id                         = var.HostGroupId 

##############################################################
# Kubelet configuration  

  kubelet_disk_type                     = var.KubeletDiskType
  kubelet_config {...}

##############################################################
# Linux OS configuration 

  linux_os_config {...}

##############################################################
# Upgrade configuration
  upgrade_settings {
    max_surge                           = var.AKSMaxSurge
  }

##############################################################
# Windows OS configuration 

  dynamic "windows_profile" {...}



##############################################################
# Tags 

  tags = merge(var.DefaultTags,var.ExtraTags,{"AKSClusterName"=split("/",var.AKSClusterId)[8]})



}

```

For katacontainer, we do have a requirement to use a Mariner node pool.
It's important to remember that it is currently a preview, and as such it requires to be activate in the provider with the command `az feature register` command.

```bash

az feature register --namespace "Microsoft.ContainerService" --name "KataVMIsolationPreview"

```

Also, we have to specify the `--workload-runtime` to `KataMshvVmIsolation` so that the feature is activated on the node pool. Otherwise, the runtime class will not be available. More on that in the next part.

```bash

az aks nodepool add --cluster-name <AKS_Cluster_Name> --resource-group <AKS_Resource_Group> --name <Node_Pool_Name> --os-sku mariner --workload-runtime KataMshvVmIsolation --node-vm-size <VM_Size>

```

Interestingly enough, while there is a `workload_runtime` parameter in the terraform provider, it currently only support `OCIContainer` or `WasmWasi`. So we are stuck with either az cli or ARM.

That's almost all on the node pool configuration. The last details are more in the kubernetes plane so we will have a look in the next part

## 3. Running Sandbox container in AKS with gvisor

first thing first, let's have a look on our nodes. With a custom columns selection, we can get the taints and labels. Note that we're selecting a specific label here: 

```bash

yumemaru@azure:~$ k get nodes -o custom-columns='NodeName:.metadata.name,LabelAgentPool:.metadata.labels.agentpool,NodeTaintsKey:.spec.taints[].key,NodeTaintsValue:.spec.taints[].key,NodeTaintsEffect:.spec.taints[].effect'
NodeName                               LabelAgentPool   NodeTaintsKey        NodeTaintsValue   NodeTaintsEffect
aks-aksnp0sbxcon-29325118-vmss00000m   aksnp0sbxcon     CriticalAddonsOnly   true              NoSchedule
aks-aksnp0sbxcon-29325118-vmss00000n   aksnp0sbxcon     CriticalAddonsOnly   true              NoSchedule
aks-aksnp0sbxcon-29325118-vmss00000o   aksnp0sbxcon     CriticalAddonsOnly   true              NoSchedule
aks-npgvisor-28185204-vmss000009       npgvisor         gvisor               true              NoSchedule
aks-npgvisor-28185204-vmss00000a       npgvisor         gvisor               true              NoSchedule
aks-npkata-37278511-vmss000003         npkata           KataContainer        true              NoSchedule
aks-npkata-37278511-vmss000004         npkata           KataContainer        true              NoSchedule
aks-npkata-37278511-vmss000005         npkata           KataContainer        true              NoSchedule

```

With that, we can install gvisor with a daemonset: 

```yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gvisor
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: gvisor
  template:
    metadata:
      labels:
        app: gvisor
    spec:
      hostPID: true
      restartPolicy: Always
      containers:
      - image: docker.io/yumemaru1979/gvisor:latest
        imagePullPolicy: Always
        name: gvisor
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          privileged: true
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: k8s-node
          mountPath: /k8s-node
      volumes:
      - name: k8s-node
        hostPath:
          path: /tmp/gvisor
      tolerations:
        - key: gvisor
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        agentpool: npgvisor

```

The interesting part on a scheduling point of view here is the toleration which match the one from our node gvisor `gvisor=true:NoSchedule`.

We are also using a `nodeSelector` to match the `agentpool: npgvisor` so that we are sure that the pods will only execute on this node pool.
For the image and what it does, really, I took the information from [Daniel Neumann blog](https://www.danielstechblog.io/running-gvisor-on-azure-kubernetes-service-for-sandboxing-containers/) as mentionned earlier.

With that we should see that we have as many pod as node in the node pool:

```bash
yumemaru@azure$ k get daemonsets.apps gvisor -n kube-system
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR        AGE
gvisor   2         2         2       2            2           agentpool=npgvisor   34h

```

Checking on the portal, we can also see which pod run on which node:

![illustration7](/assets/katacontainer/sbxcontainer007.png)

With that ready, we need now to add a runtime class: 

```yaml

apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    agentpool: npgvisor

```

And last, we can schedule pods on the node pool. To use the runtime class and thus get a sandboxed container, we need to specify the `runtimeClassName`. We also specify the `NodeSelector` and the `tolerations` to ensure that the pod can run on the node pool that we want:

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: gvisor
  name: gvisortest
  namespace: gvisordemo
spec:
  runtimeClassName: gvisor
  nodeSelector:
    agentpool: npgvisor  
  tolerations:
  - key: gvisor
    operator: Exists
    effect: NoSchedule
  containers:
  - image: nginx
    name: gvisortest
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

for comparison purpose, we can also add a 

### 3.1 sandbox with gvisor

get labels on node and taints
create ds with node selector
create  runtime class
create pod


### 3.2. sanbox with katacontainer

still get labels and taints
get runtime class
create pod

check kernel
