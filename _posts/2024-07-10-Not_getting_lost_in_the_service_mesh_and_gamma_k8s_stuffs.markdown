---
layout: post
title:  "Not Getting lost in the Service mesh and GAMMA initiative stuff in the k8s landscape"
date:   2024-07-10 18:00:00 +0200
year: 2024
categories: AKS Security Network
---

Hello all!

This article initiative is the result of me, looking at some nice tech stuff in k8s/AKS stuff and getting a little lost between all the differents initiatives, names, stuff... about microsgmentation, mesh and nort-south/east-west traffic management.
It will probably be a little bit theoretical but I'll need that as a basis for carrying on the other tech stuff explorations, so please deal with me ^^

We'll have two part in this session:

1. About service mesh(es)
2. The impact of Kubernetes Gateway API
3. Where we are

Let's get started!

## 1. About service mesh(es)

Working in the k8s landscape, it's very difficult to not have heard about service mesh in the last years.
I remember a session, at Google Next 2019, introducing Istio, the service mesh concepts and, at the same time, some SRE concepts.
I was quite (totally?) green in the container orchestration landscape at this time and thus did not grasp all of what the speakers explained.
In the following months, we saw a lot about service mesh in Kubernetes, even if to this day, I do not see that much implementations of it.
That being said, if we had to define what a service mesh is, we could take the definition proposed by Luke Kysaw in hos [Consul up & running book](https://www.amazon.fr/Consul-Running-English-Luke-Kysow-ebook/dp/B0B2ZHXLCV/ref=sr_1_1?__mk_fr_FR=%C3%85M%C3%85%C5%BD%C3%95%C3%91&dib=eyJ2IjoiMSJ9.iNTQHYMdF683hZdk2ZpJ0pepua_G3DUVs9IolIOZstk.rYQVxoLJqe2QLynPPjkZJQce75ng-diszCgk-m0LDlk&dib_tag=se&keywords=consul+up+%26+running&qid=1719989878&sr=8-1)

```

A service mesh is an infrastructure layer that enables you to control the network communication from a single control plane

```

In a world of micro-services, it's interesting because we may have a lot of those and maintaining just a basic network segmentation with good firewall is, at most, cumbersome, at worst, innefficient.
Let's draw a bit shall we? Below is an example of many micro services that talk together

![illustration1](/assets/servicemesh/smesh002.png)

imagine now that you want to control the flows between those service. Depending on your network topology, you may be able to implement some measure of control with a firewall, but it's bery difficult to keep up.
Now we can consider microsegmentation with distributed firewall. It's done in all software defined data center, including good old openstack, or our public cloud, with Network Security Group in Azure for example.
But what about the operations on this distributed model? I could tell you that we have a DevOps Approach and that our management plane is in the git repository... &#129300; but it's almost never that easy when troubleshooting comes (which does not mean I don't want distributed firewall).

![illustration2](/assets/servicemesh/smesh003.png)

Let's add some additional complxities and introduce kubernetes in the equation &#129299;.

![illustration3](/assets/servicemesh/smesh004.png)

If we take the asumption that the non-kubernetes landscape is totally under control, we still added a new abstraction layer and, well, the abstraction change the equation thoroughly. We have network overlay, API objects refering to compute & network stuff, and it's simply impossible to keep a segmentation with a solution that is not *kubernetes-aware*.

Enter the Service Mesh.

Considering the needs

```

Maintain segmentation and visibility of said segmentation even in kubernetes environment

```

and the service mesh definition mentioned earlier

```

An infrastructure layer that enables you to control the network communication from a single control plane

```

![illustration4](/assets/servicemesh/smesh005.png)

We seem to have found the perfect candidate.
But how work a service mesh, technically speaking?
Let's refer to the, now archived, [service mesh interface specification](https://github.com/servicemeshinterface/smi-spec?tab=readme-ov-file).
This initiative was, simalarly to all the other *initiatives* in the kubernetes landscape, aiming to standardize the service mesh features. Looking at the documentation provided on this github, we can find that most of the service mesh actors were involved in this initiative.

### Ecosystem

* **Consul Connect\*:** service segmentation ([consul.io/docs/connect](https://consul.io/docs/connect))
* **Flagger:** progressive delivery operator ([flagger.app](https://flagger.app))
* **Gloo Mesh:** Multi-cluster service mesh management plane
 ([solo.io/products/gloo-mesh](https://solo.io/products/gloo-mesh))
* **Istio\*:** connect, secure, control, observe ([servicemeshinterface/smi-adapter-istio](https://github.com/servicemeshinterface/smi-adapter-istio))
* **Linkerd:** ultralight service mesh ([linkerd.io](https://linkerd.io))
* **Traefik Mesh:** simpler service mesh ([traefik.io/traefik-mesh](https://traefik.io/traefik-mesh))
* **Meshery:** the service mesh management plane ([layer5.io/meshery](https://layer5.io/meshery))
* **Rio:** application deployment engine ([rio.io](https://rio.io))
* **Open Service Mesh:** lightweight and extensible cloud native service mesh ([openservicemesh.io](https://openservicemesh.io))
* **Argo Rollouts:** advanced deployment & progressive delivery controller ([argoproj.io](https://argoproj.github.io/argo-rollouts))

### API objects

Now regarding the API objects, we can find the following lists of objects that aimed to provide us with the solution to our requirement: 

|                               |         Latest Release             |  
| :---------------------------- | :--------------------------------: |
| **Core Specification:**       |
| SMI Specification             |  [v0.6.0](/SPEC_LATEST_STABLE.md) |
|                               |
| **Specification Components**  |
| Traffic Access Control  |  [v1alpha3](/apis/traffic-access/v1alpha3/traffic-access.md)  |
| Traffic Metrics   |  [v1alpha1](/apis/traffic-metrics/v1alpha1/traffic-metrics.md)  |
| Traffic Specs  |  [v1alpha4](/apis/traffic-specs/v1alpha4/traffic-specs.md)  |
| Traffic Split  |  [v1alpha4](/apis/traffic-split/v1alpha4/traffic-split.md) |

Typically, the microsegmentation is made possible through object like the Traffic Access Control, the visibility in the control plane is helped with the Traffic Metrics...
If we look a little bit further in the Traffic Access Control  object, in its `v1alpha4` version, we can even see reference to authentication wit an `IdentityBinding` object.

```yaml

apiVersion: access.smi-spec.io/v1alpha4
kind: IdentityBinding
metadata:
 name: service-a
 namespace: default
spec:
 schemes:
   podLabelSelector:
    matchLabels:
      app: service-a
   spiffeIdentities:
     - "cluster.local/ns/default/sa/service-a"
     - "federated.trustdomain/boundary/boundaryName/identifierType/identifier"
   serviceAccount: service-a

```

We'll note the reference to SPIFFE identities, with [SPIFFE](https://spiffe.io/) standing for **Secure Production Identity Framework for Everyone**. It's not the place to discuss in depht of this solution but we'll keep that in mind for mutual authentication scenarios.

Ok, so we have service meshes solutions. But, even with the heterogeneous complexities of those solutions, it was still too simple, and then enter the GAMMA Initiative, associated to the Kubernetes Gateway API topic.



## 2. The impact of Kubernetes Gateway API

We talked about service mesh and how to secure east-west traffic, meaning traffic between services, while using a single control plane. But what about North-South Traffic, or how to manage ingress and egress traffic?
Beyond the simple kubernetes service, we have seen, and used the [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), which, through a [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/), provides a mean to expose application on the layer 7 level. So TLS, http path, rewrite, those are available for the application that need to be exposed.

![illustration5](/assets/servicemesh/smesh006.png)

*Gateway API schema from the [documentation](https://gateway-api.sigs.k8s.io/)*

Now considering the service mesh, it has become well known that its integration with an ingress controller is far from easy. When we consider that the service mesh, by design, manage east-west traffic but also block by default north-south traffic (or at least ingress traffic), we face a quite important issue.
Sure, some Ingress Controllers argue that their implementation/architecture is easily integrable with a service mesh, and vice versa, but we encounter a lack of uniformity in the feature here between the differents solution available.
Still in the lack of uniformity topic, it's also well known that not everything is standard between the different Ingress Controller (but well, did you see an Ingress Controller Interface Initiative?&#x1F92F;), leading, in practice, to the use of custom annotations for extensibility.

So here come the Gateway API.

From the begining, it's designed to fill the lack of the Ingress Controller API. So the topic today is not to dive deep in Gateway API (another time, promise), but to identify why it changes how we should approach east-west traffic management.
It written bluntly in the [Kubenetes documentation](https://kubernetes.io/blog/2023/08/29/gateway-api-v0-8/):

```

While the initial focus of Gateway API was always ingress (north-south) traffic, it was clear almost from the beginning that the same basic routing concepts should also be applicable to service mesh (east-west) traffic. In 2022, the Gateway API subproject started the GAMMA initiative, a dedicated vendor-neutral workstream, specifically to examine how best to fit service mesh support into the framework of the Gateway API resources, without requiring users of Gateway API to relearn everything they understand about the API.

```

Which explains why the service mesh interface is now archived. 

There is another kubernetes project inside the Gateway API project that work on defining how Gateway API can be used for service mesh.



## 3. Where we are

With all that, we are still far from a fully standardized model, and there's a big chance that the knowledge will not be up to date in a uniform way.

We have Ingress Controller providers, progressively upgrading/migrating to the Gateway API approach.
We also have service mesh, sometimes well established, or simply not necessarily focused on Kubernetes only (Hello [Hashicorp Consul](https://developer.hashicorp.com/consul/docs/intro)&#128075;) which follow with a mix of standardized / custom implementations for the Gateway that may allow access in the mesh.

We also have multi-feature one in all approach with some CNI (Did I already talked about [Cilium](https://docs.cilium.io/en/stable/)?&#129300;)

**schemacurrentmesh**

It's still difficult to adopt fully a kubernetes approach now with all those evolution. In my humble opinion, the best way is still the iterative way. We probably do not need all the features from the begining so it's ok to not choose a service mesh-ish implementation (and it may never be the case depending on the hosting governance). It's not ok to not keep up at least with the new concepts that will impact the hosting choice, or the operating model or... whatever your humble tech guy may not think about now.

So what's next? From my side, more exploration of tech feature that may be useful in the environment I interact with. At my level, that's the most I can do ^^



