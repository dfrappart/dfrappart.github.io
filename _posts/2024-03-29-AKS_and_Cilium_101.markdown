---
layout: post
title:  "AKS and Cilium 101"
date:   2024-03-29 18:00:00 +0200
year: 2024
categories: AKS Network
---

Hi there!

If you're in the K8S world, you've probably come across eBPF and Cilium lately.
I did and started to look at what it means when considering AKS, which is my main Kubernetes platform.
In this article, I propose a first look at AKS and Cilium together and what we can do with it. 
Spoiler, I'll have to do more than 1 article to be able to deep dive in all the features that Cilium brings to the table.
For now, it will be just an introduction.

So below is our agenda:

1. (Re-)Introducing Cilium
2. Cilium & AKS a love story?
3. Deploying AKS with Cilium as a CNI
4. Looking at the features of Cilium

Let's get started, with a review of what Cilium is.

## 1. (Re-)Introducing Cilium

So, at its heart, Cilium is a CNI.

CNI stands for Container Network Interface.

As for most of the things in the container landscape, with the broadening footprint of its adoption, standards had to be created. First, there was the [Open Container Initiative](https://opencontainers.org/about/overview/), which, as the name implies, was the start of a standard for container description.
You may also have come accross [Container Storage Initiative](https://github.com/container-storage-interface/spec) which is kind of similar, but focused on the way storage providers can interface their solutions in container based environment.

Which brings us CNI now, the same for Network considerations in container based environments (and in our case, it definitely means Kubernetes environment).

So, as described on the dedicated [github](https://github.com/containernetworking/cni/blob/main/SPEC.md​), the aims of the CNI specifications is to define:

- A format for administrators to define network configuration.​
- A protocol for container runtimes to make requests to network plugins.​
- A procedure for executing plugins based on a supplied configuration.​
- A procedure for plugins to delegate functionality to other plugins.​
- Data types for plugins to return their results to the runtime.

Now, the idea for today is not to go deep into the CNI specs, because, it would be way out of my reach :laughing:.
For now let's just list a few well known CNI such as:

- [Calico](https://docs.tigera.io/calico/latest/about/)
- [Azure](https://github.com/Azure/azure-container-networking) (well we do talk about AKS after all &#x1F606;	)
- and of course [Cilium](https://docs.cilium.io/en/stable/overview/intro/)

Now that we've prepared the basics, what about Cilium?

So Cilium is bringing a new approach to network in Kubernetes because it relies on [eBPF](https://ebpf.io/what-is-ebpf/#what-is-ebpf), which stands for Extended Berkley Packet Filter.

eBPF in itself is the game changer, because it allows to run sandboxed programs within the operating system. 
With the aid of a Just-In-Time compiler and verification engine, the OS is able to guarantees safety and execution, as if those programs were natively compiled.
eBPF is used for many use case, including networking, observability and security functionnality, which means, most of Cilium topics ^^

![illustration0](/assets/akscilium/ebpfschema.png)

Now that we've talked a little bit about Cilium, let's focus on what we can do on Azure with this CNI.


## 2. Cilium & AKS, a love story?

Let's start by reviewing rapidly what we have for CNI options on AKS.
We talked about that in a previous article about [AKS Network considerations]().
Initially, we had a choice between kubenet and Azure CNI. 
While Azure CNI was often the de-facto choice for AKS aimed to production, It brough a serious probleme of IP consumptions and was sometimes difficult to put in place in hybrid environement.
On the other hand, kubenet, while not subject to this IP consumption problem, was (and still is) crippled by an additional latency due to the additional hop inducted by the pod network overlay.

![illustration1](/assets/aksntwconsiderations/kubenet001.png)

![illustration2](/assets/aksntwconsiderations/cni002.png)

To solve that, fortunately, the AKS teams worked a lot and gave us additional solutions. 
Two of those mark an interest because of the potential additional feature:

- Azure CNI powered by Cilium
- Bring your own CNI

The first option, as the name imply, is an Azure CNI that bring hypothetically the power of Cilium.
Unfortunately, in its current state, we do not get some of the interesting features such as L7 network policies or Hubble, as describe in the [Azure documentation](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium#limitations). We'll get more to those feature in the last part of this article, and probably in other article, because, there's a lot to see and test.

The second option is not specifically dedicated to Cilium, but since we can choose whatever CNI we want, Cilium becomes a possible choice. We do have to install it ourselve though, and thus become responsible of its state and update, and all the nice stuff that you have to do when you managed your own kubernetes. This will be our choice for the remaining parts of this article.

Last, because there's always another option, is the Cilium Enterprise proposed by Isovalent. It's quite well documented on  a [Isovalent blog post](https://isovalent.com/blog/post/isovalent-aks/) and remove the issue of managing Cilium without support. There is of course a cost to consider, and to balance between the skill set of the platform engineering team, and the benefit of an available support.

![illustration3](/assets/akscilium/akscilium001.png)

To illustrate the interest of Cilium community or enterprise version versus the current Azure CNI powered by Cilium, we have the following table:

| Features | Azure CNI Powered by Cilium | Cilium  Open Source | Isovalent Enterprise for Cilium |
|-|:-:|:-:|:-:|
| Container Networking (CNI) |&#x2705;|&#x2705;|&#x2705;|
|Kubernetes Network Policy & Services |&#x2705;|&#x2705;|&#x2705;|
| Collaborative Support Agreement |&#x2705;|&#x2705;|&#x2705;|
| Advanced Network Policy & Encryption (DNS, L7, TLS/SNI, …) |&#x274C;|&#x2705;|&#x2705;|
| Ingress, Gateway API, & Service Mesh |&#x274C;|&#x2705;|&#x2705;|
| Multi-Cluster, Egress Gateway, BGP, Non-Kubernetes Workloads |&#x274C;|&#x2705;|&#x2705;|
| Hubble Network Observability (Metrics, Logs, Prometheus, Grafana, OpenTelemetry) |&#x274C;|&#x2705;|&#x2705;|
| SIEM Integration & Timescape Observability Storage |&#x274C;|&#x274C;|&#x2705;|
|Tetragon Runtime Security |&#x274C;|&#x2705;|&#x2705;|
|Enterprise-hardened Cilium Distribution, Training, 24×7 Enterprise Grade Support |&#x274C;|&#x274C;|&#x2705;|

For further details, we can check the [Isovalent page](https://isovalent.com/product/) comparing Enterprise vs Open Source.

So, many options for Cilium, as expected. Let's try to deploy an AKS cluster with this CNI now.

## 3. Deploying AKS with Cilium as a CNI

As we discussed in the previous section, we can get a flavor of Cilium, either with Azure CNI powered by Cilium, or through the BYO CNI option.

Considering the az cli arguments, we need to use with the `az aks create` command the following switch

| AKS byo cni parameters​ | AKS Azure CNI powered by cilium parameters |
|-|-|
| `--network-plugin none` | `--network-plugin azure` <br> `--network-plugin-mode overlay` <br> `--network-dataplane cilium` <br> |

Creating an Azure CNI powered by Cilium cluster is done with a command looking like this:

```bash

yumemaru@azure:~$ az aks create -n aks-cnicilium -g rsg-ciliumlab -l eastus --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --load-balancer-sku standard --vnet-subnet-id "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-ciliumlabntw/providers/Microsoft.Network/virtualNetworks/vnet-sbx-spokecilium1/subnets/sub2-vnet-sbx-spokecilium1" --enable-oidc-issuer --enable-encryption-at-host --enable-workload-identity 

```

Once the cluster is deployed, we can get the cluster credentials and have a look at Cilium status. We need cilium cli btw, which binary are available [here](https://github.com/cilium/cilium-cli#helm-installation-mode). 

```bash

yumemaru@azure:~$ az aks get-credentials -n aks-cnicilium -g rsg-ciliumlab
The behavior of this command has been altered by the following extension: aks-preview
Merged "aks-cnicilium" as current context in /home/df/.kube/config
yumemaru@azure:~$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Containers:            cilium             Running: 3
                       cilium-operator    Running: 2
Cluster Pods:          9/9 managed by Cilium
Helm chart version:    0.1.0-3127877da2a229fb171bf4c5d84fcaf9aa8584c3
Image versions         cilium             mcr.microsoft.com/oss/cilium/cilium:1.13.10-1: 3
                       cilium-operator    mcr.microsoft.com/oss/cilium/operator-generic:1.13.10: 2


```

We'll note the hubble state, which is disabled, as defined in the Azure documentation. We could try to enable the feature through the cilium cli, but spoiler: It does not work.

So that's about all for the CNI powered by Cilium, because, we don't get the first feature that we want to test which is Network observability through Hubble.

Now, for a cluster created with the byocni, because we do bring our own CNI, the cluster comes with nodes in `NotReady` State : 

```bash

yumemaru@azure:~$ k get no
NAME                                STATUS     ROLES   AGE   VERSION
aks-nodepool1-30325416-vmss000000   NotReady   agent   20m   v1.28.5
aks-nodepool1-30325416-vmss000001   NotReady   agent   20m   v1.28.5
aks-nodepool1-30325416-vmss000002   NotReady   agent   19m   v1.28.5

```

We do need to install cilium, with the following paramaters for the helm chart: 

```go

{

    "set1" = {
      ParamName  = "hubble.relay.enabled"
      ParamValue = "true"
    },
    "set2" = {
      ParamName  = "hubble.ui.enabled"
      ParamValue = "true"
    },
    "set3" = {
      ParamName  = "aksbyocni.enabled"
      ParamValue = "true"
    },
    "set4" = {
      ParamName  = "nodeinit.enabled"
      ParamValue = "true"
    },
    "set5" = {
      ParamName  = "kubeProxyReplacement"
      ParamValue = "true"
    },
    "set6" = {
      ParamName  = "k8sServiceHost"
      ParamValue = "aksciliumlab-p68lm1x0.hcp.eastus.azmk8s.io"
    },
    "set7" = {
      ParamName  = "k8sServicePort"
      ParamValue = "443"
    }
  }

```

And then the cilium status command shows us the following display: 

```bash

yumemaru@azure:~$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            hubble-ui          Running: 1
                       cilium-operator    Running: 2
                       hubble-relay       Running: 1
                       cilium             Running: 3
Cluster Pods:          29/29 managed by Cilium
Helm chart version:    1.14.0
Image versions         cilium             quay.io/cilium/cilium:v1.14.0@sha256:5a94b561f4651fcfd85970a50bc78b201cfbd6e2ab1a03848eab25a82832653a: 3
                       hubble-ui          quay.io/cilium/hubble-ui:v0.12.0@sha256:1c876cfa1d5e35bc91e1025c9314f922041592a88b03313c22c1f97a5d2ba88f: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.12.0@sha256:8a79a1aad4fc9c2aa2b3e4379af0af872a89fcec9d99e117188190671c66fc2e: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.0@sha256:3014d4bcb8352f0ddef90fa3b5eb1bbf179b91024813a90a0066eb4517ba93c9: 2
                       hubble-relay       quay.io/cilium/hubble-relay:v1.14.0@sha256:bfe6ef86a1c0f1c3e8b105735aa31db64bcea97dd4732db6d0448c55a3c8e70c: 1


```


This time, we do have hubble, among our enabled features for Cilium.
Let's start playing a bit then.


## 4. Looking at the features of Cilium

So the first thing we'll look at is indeed Hubble, which is a network observability tool included with Cilium, that is, if we enabled it for the installation.

To perform some test, we'll add some resources:

```yaml

apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: basics
spec: {}
status: {}


```

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: basicdeploy
    org: cilium101
  name: basicdeploy
  namespace: basics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: basicdeploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: basicdeploy
        org: cilium101
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}


```

```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: basicdeploy
  name: basicdeploy
  namespace: basics
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: basicdeploy
status:
  loadBalancer: {}

```

Once our deployement is ready and exposed with its service,

```bash

yumemaru@azure:~$ k get all -n basics 
NAME                               READY   STATUS    RESTARTS   AGE
pod/basicdeploy-65ff8695df-p58zr   1/1     Running   0          19h

NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
service/basicdeploy   LoadBalancer   10.0.189.143   172.171.45.48   80:30316/TCP   20h

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/basicdeploy   1/1     1            1           20h

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/basicdeploy-65ff8695df   1         1         1       20h

```

we can start with the Hubble UI. It's done with the `cilium hubble ui` command, and we get, mapped to a local port, the UI in our navigator. As for other Kubernetes dashboard, we can select the namespace.

![illustration4](/assets/akscilium/akscilium002.png)

The UI display a graphical view of the flows in the chosen namespace.

![illustration5](/assets/akscilium/akscilium003.png)

And a list of the flows:

![illustration6](/assets/akscilium/akscilium004.png)

If we add a new pod, and use it to test access on the deployment, we see the traffic rendered in the hubble UI.

```bash

yumemaru@azure:~$ k exec testpod -- curl -i -X GET http://172.171.45.48

yumemaru@azure:~$ k exec testpod -- curl -i -X GET http://10.0.189.143

```

![illustration7](/assets/akscilium/akscilium005.png)

![illustration8](/assets/akscilium/akscilium006.png)

Ok that's nice, let's try the cli now.

As for the UI, we enable access to the cli with a cilium command:

```bash

yumemaru@azure:~$ cilium hubble port-forward&
yumemaru@azure:~$ hubble observe

```

The result is a bit too much, so we want to fine tune a little the result. Looking at the help, we can see some interesting switch:

```bash
--from-label      filter            Show only flows originating in an endpoint with the given labels (e.g. "key1=value1", "reserved:world")
--from-namespace  filter            Show all flows originating in the given Kubernetes namespace.
--from-pod        filter            Show all flows originating in the given pod name prefix([namespace/]<pod-name>). If namespace is not provided, 'default' is used
--to-label        filter            Show only flows terminating in an endpoint with given labels (e.g. "key1=value1", "reserved:world")
--to-namespace    filter            Show all flows terminating in the given Kubernetes namespace.
--to-pod          filter            Show all flows terminating in the given pod name prefix([namespace/]<pod-name>). If namespace is not provided, 'default' is used

```
We can thus filter the traffic between the test pod and the deployment and get something like this:
```bash

yumemaru@azure:~$ hubble observe --to-pod basics/basicdeploy-65ff8695df-p58zr --from-pod default/testpod
Mar 29 20:13:27.359: default/testpod (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) post-xlate-fwd TRANSLATED (TCP)
Mar 29 20:13:27.359: default/testpod:43414 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 20:13:27.360: default/testpod:43414 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: ACK)
Mar 29 20:13:27.360: default/testpod:43414 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Mar 29 20:13:27.362: default/testpod:43414 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: ACK)
Mar 29 20:13:27.362: default/testpod:43414 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: ACK, FIN)

```

Now let's see if we can identify traffic depending on the verdict. For that, we'll add network policies to filter traffic, and since Cilium also gives us some interesting options regzrding this component, we'll have a look at our second feature of the CNI.

A native kubernetes network policy to block all traffic on a namespace would look like this:

```yaml

kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny-all
  namespace: basics
spec:
  podSelector: {}
  ingress: []

```

The `podSelector` configured with `{}` means that we select all pods in the namespace, and the `ingress` with the `[]` means that we allow no Ingress traffic.

We could use thenative policy with Cilium, but let's have a look at the Cilium-specific one:

```yaml

apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cilium101-default-deny"
  namespace: basics
spec:
  description: "Default-deny ingress policy for cilium101"
  endpointSelector:
    matchLabels:
      org: cilium101
  ingress:
  - {}

```

We can notice some differences here. While the `ingress` parameter remains, we select for the target an endpoint with the `endpointSelector` rather than a `podSelector`. 

Let's apply this policy and see what hubble tells us:

```bash

yumemaru@azure:~$ hubble observe --to-pod basics/basicdeploy-65ff8695df-p58zr --from-pod default/testpod

Mar 29 21:06:33.831: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 21:06:33.831: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 21:06:35.847: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 21:06:35.847: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 21:06:40.099: default/testpod:48740 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 21:06:40.105: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 21:06:40.105: default/testpod:48740 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)


```

This time, we can see some dropped packet, mentioning the `Policy denied DROPPED`.

Let's add a new pod in a new namespace called client: 

```bash

yumemaru@azure:~$ k create ns client
namespace/client created
yumemaru@azure:~$ k run client --image=nginx -n client
pod/client created

```

And now we add a Cilium Network policy to allow traffic from this pod

```yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allowngfromclientnstobasicapp
  namespace: basics
spec:
  description: L3-L4 policy to restrict cilium101 deployment to client pod only
  endpointSelector:
    matchLabels:
      org: cilium101
  ingress:
  - fromEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: client
        run: client
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP

```

Again, we can have a look at our traffic through hubble and see the allowed traffic: 

```bash
yumemaru@azure:~$ hubble observe --to-pod basics/basicdeploy-65ff8695df-p58zr --from-pod default/testpod
Mar 29 22:11:59.762: default/testpod (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) post-xlate-fwd TRANSLATED (TCP)
Mar 29 22:11:59.762: default/testpod:55722 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:12:06.887: default/testpod:55722 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:12:15.075: default/testpod:55722 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:12:31.203: default/testpod:55722 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:13:04.995: default/testpod:55722 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:23:01.583: default/testpod (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) post-xlate-fwd TRANSLATED (TCP)
Mar 29 22:23:01.584: default/testpod:47484 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:23:01.588: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 22:23:01.588: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:23:02.599: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 22:23:02.599: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:23:04.615: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 22:23:04.615: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:23:08.647: default/testpod:47484 (ID:2615) -> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) to-overlay FORWARDED (TCP Flags: SYN)
Mar 29 22:23:08.647: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Mar 29 22:23:08.647: default/testpod:47484 (ID:2615) <> basics/basicdeploy-65ff8695df-p58zr:80 (ID:9727) Policy denied DROPPED (TCP Flags: SYN)

```

Ok, but at this point, this is just another way of writing a network policy, without any addiotion. Let's gear up and add some L7 in our network policy.
For this we'll want to manage egress traffic on our deployment, and allow only a subset of Egress Traffic.
Let's start with a deny Egress:


```yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress
  namespace: basics
spec:
  podSelector:
    matchLabels:
      org: cilium101
  policyTypes:
  - Egress
  egress: []

```

We can definitely see the dropped traffic to kube-dns

![illustration9](/assets/akscilium/akscilium007.png)

![illustration10](/assets/akscilium/akscilium008.png)

![illustration11](/assets/akscilium/akscilium009.png)

Let's first fix access to kube-dns:

```yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cilium101todns
  namespace: basics
spec:
  podSelector:
    matchLabels:
      org: cilium101
  policyTypes:
  - Egress
  egress:
  # allow DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP

```

We can see the traffic is now forwarded to kube-dns, even if the egress traffic is still globaly blocked:

![illustration12](/assets/akscilium/akscilium010.png)

![illustration14](/assets/akscilium/akscilium012.png)

Let's fix this, only for selected hostname with the following policy:

```yaml

kind: CiliumNetworkPolicy
metadata:
  name: "to-google"
  namespace: basics
spec:
  endpointSelector:
    matchLabels:
      org: cilium101
  egress:
    - toEndpoints:
      - matchLabels:
          "k8s:io.kubernetes.pod.namespace": kube-system
          "k8s:k8s-app": kube-dns
      toPorts:
        - ports:
           - port: "53"
             protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchName: "google.com"


```

From the pod, Google can now be reached, which is not the case for, for example, github:

```bash

yumemaru@azure:~$ k exec -n basics basicdeploy-65ff8695df-p58zr -- curl -i -X GET http://google.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   219  100   219    0     0   5456      0 --:--:-- --:--:-- --:--:--  5475
HTTP/1.1 301 Moved Permanently
Location: http://www.google.com/
Content-Type: text/html; charset=UTF-8
Content-Security-Policy-Report-Only: object-src 'none';base-uri 'self';script-src 'nonce-sV5c5Sjw7ddtkiSm6Ojijg' 'strict-dynamic' 'report-sample' 'unsafe-eval' 'unsafe-inline' https: http:;report-uri https://csp.withgoogle.com/csp/gws/other-hp
Date: Fri, 29 Mar 2024 22:50:49 GMT
Expires: Sun, 28 Apr 2024 22:50:49 GMT
Cache-Control: public, max-age=2592000
Server: gws
Content-Length: 219
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
yumemaru@azure:~$ k exec -n basics basicdeploy-65ff8695df-p58zr -- curl -i -X GET http://github.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:08 --:--:--     0

```

A last look at hubble gives us confirmation of flows 

```bash

yumemaru@azure:~$ hubble observe --from-pod basics/basicdeploy-65ff8695df-p58zr
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) -> kube-system/coredns-789789675-79xnk:53 (ID:41216) policy-verdict:L3-L4 EGRESS ALLOWED (UDP)
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) -> kube-system/coredns-789789675-79xnk:53 (ID:41216) to-proxy FORWARDED (UDP)
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) <> 172.21.0.4 (host) pre-xlate-rev TRACED (UDP)
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) -> kube-system/coredns-789789675-79xnk:53 (ID:41216) dns-request proxy FORWARDED (DNS Query github.com. A)
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) <> 172.21.0.4 (host) pre-xlate-rev TRACED (UDP)
Mar 29 22:50:56.889: basics/basicdeploy-65ff8695df-p58zr:58834 (ID:9727) -> kube-system/coredns-789789675-79xnk:53 (ID:41216) dns-request proxy FORWARDED (DNS Query github.com. AAAA)
Mar 29 22:50:56.892: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:50:56.892: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:50:57.907: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:50:57.907: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:50:59.923: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:50:59.923: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:51:03.955: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:51:03.955: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:51:12.152: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:51:12.152: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:51:28.275: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:51:28.275: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> github.com:80 (world) Policy denied DROPPED (TCP Flags: SYN)
Mar 29 22:52:01.812: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> 140.82.114.3:80 (world) policy-verdict:none EGRESS DENIED (TCP Flags: SYN)
Mar 29 22:52:01.812: basics/basicdeploy-65ff8695df-p58zr:33316 (ID:9727) <> 140.82.114.3:80 (world) Policy denied DROPPED (TCP Flags: SYN)

```

Ok time to wrap this up!

## 5. Summary

We've seen quite a lot of stuff here:

- What and why Cilium is such a hot topic with its eBPF based archiecture
- cilium in an Azure landscape

And a few features:

- Cilium Network policies and most interstingly the L7 capabilities
- and most of all, the Network observability tool Hubble that gives us live insight of what happens on our Kubernetes Network

Just those makes Cilium (IMHO) a game changer, but there's more:

- Cluster Mesh
- Authentication
- Encryption

And also a Gateway API without any other installation.

Well, let's get back on all that in another article ^^

Until then, have a good time!



