---
layout: post
title:  "About Cilium native authentication feature"
date:   2024-07-31 18:00:00 +0200
year: 2024
categories: AKS Security
---

Hi there!

We've been talking about Cilium for some time now, but there are so much thing to look at on this, that we knew we would come back to this topic.
Today, I propose that we explore the feature that is available for authenticating workload. Before tackling this topic, We need some definition

Our agenda will be as follow

1. Understanding Cilium authentication feature
2. Trying out the authentication

Let's get started!

## 1. Cilium authentication feature

### 1.1. Why we need authentication

Security in micro-service based architecture is a huge topic. 
In a kubernetes environment, we can look at the network and we implement least privilege by restricting traffic to the only needed port and protocol.
In this case, we can rely on network policies.
For example, this policy deny all ingress traffic in the `basic` namespace

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

This policy allows only traffic to the pod with the label `app=userprofile` on the port `8082` from the pods in the namespace with the labels `tier=front` and `app=tripsinsights`.

```yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-tripviewer-ingress-apiprofile
  namespace: api
spec:
  podSelector: 
    matchLabels:
      app: userprofile
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: front
          app: tripinsights
    ports:
    - protocol: TCP
      port: 8082   

```

But it may not enough, and it's quite difficult to manage globally by lack of an centralized management plane. That's the paradox of the distributed security.

We talked in a previous article of the service mesh which aims to provide a centralized control plane for the network flows. And we've seen that in the initiative of standard, we had some specific traffic control object which included authentication feature.

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


that's what we will have a look at with Cilium service mesh, and specifically the Identity based security. 
It's interesting to have something like that in the infrastructre layer, because it's not always possible to have authentication included in the applictions layers, specifically when we consider multiservice. At most, unfortunately, we may get authentication on the front end, but not much more.
Here, we gt a proposal to add identiy based security.

Let's have a look at how it works.

### 1.2. What's under the hood of authentication in Cilium

Authentication means, at one point, Identities. 
As hinted in the smi spec earlier, and documented in the [Cilium doc](https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication/#identity-management), the technology used is [SPIFFE](https://spiffe.io/), which stands for Secure Production Identity Framework For Everyone. It is also a [graduated project in the CNCF](https://www.cncf.io/projects/spiffe/) since August 2022.
Spiffe spec are defined on a [dedicated github](https://github.com/spiffe/spiffe).

At this point we will remains at a high level overview of the SPIFFE standards, as described on the github repo, which is comprised of:

- The SPIFFE ID, a structured string (represented as a URI) which serves as the "name" of an entity.
- The SPIFFE Verifiable Identity Document (SVID) is a document which carries the SPIFFE ID itself. It is the functional equivalent of a passport - a document which is presented that carries the identity of the presenter.
- The Workload API is the method through which workloads, or compute processes, obtain their SVID(s). It is typically exposed locally (eg. via a Unix domain socket), and explicitly does not include an authentication handshake or authenticating token from the workload.

Also, it's important to mention the trust domain which is is an identity namespace, backed by an issuing authority with a set of cryptographic keys. Together, these keys serve as the cryptographic anchor for all identities residing in the trust domain.

Again, since SPIFFE is a set of specifications, all of those concepts/standards are actually detailed in the [github repository](https://github.com/spiffe/spiffe/tree/main/standards).

So now we know about specification and standards, but there is a lot to do yet, because, well, it's only the specification. Fortunately, there is also the [SPIRE project](https://spiffe.io/docs/latest/spire-about/spire-concepts/), [another graduated CNCF project](https://www.cncf.io/projects/spire/), which is a production ready implementation of the SPIFFE spec.

In Cilium, SPIFFE implementation is based on a SPIRE central server. This SPIRE server takes the role of the trust domain that we mentionned earlier in the spec and standards.
It works with SPIRE agents, one per node, which get its own identity from the server, then validate the identity requests form the workloads.

To understand how SPIRE works, we need again some concepts:

- The **workload registration** is the process by which SPIRE will be able to identify the said workload. It tells SPIRE how to identify the workload and which SPIFFE ID to give it.
- SPIRE achieve the **Attestation** of a workload, a.k.a asserting its identity, by gathering attributes from the workload, and the node running the SPIRE agent. It's interesting also to note that the attestation is done by pieces of software called *attestators*. In our case, we will have a kubernetes attestator for the kubernetes-hosted workloads, but also the node attestator for the SPIRE agents, that need to register first before being able to proced with the workload attestation. On Azure, the node attestation/registration relies on Azure VMs with managed identity.

The Node attestation follows the below steps:

1. The agent node attestor plugin queries the platform for proof of the node’s identity and gives that information to the agent.
2. The agent passes this proof of identity to the server. The server passes this data to its node attestor.
3. The Server node attestor validates proof of identity by calling out to the platform API, using the information it obtained in step 2. The node attestor also creates a SPIFFE ID for the agent, and passes this back to the server process, along with any node selectors it discovered.
4. The server sends back an SVID for the agent node.

![illustration1](/assets/ciliumauth/nodeattestation.png)

A workload requesting an identity follows the below steps:

1. It calls the Workload API exposed by the agent to request an SVID.
2. The agent interrogates invokes the workload attestor plugins, providing it with the informations about the workload.
3. Workload attestors discover additional information about the workload, querying neighboring platform-specific components – such as a Kubernetes kubelet. 
4. The attestors return the discovered information to agent in the form of selectors.
5. The agent determines the workload’s identity by comparing discovered selectors to registration entries, and returns the correct cached SVID to the workload.

![illustration2](/assets/ciliumauth/wlattest.png)

Ok, that's about it for this overview. There are more details in SPIFFE specifications and I can only recommand a more thorough read for a better understanding.
Let's have a look at how it goes when we deploy that in an AKS cluster.

## 2. Trying Cilium authentication

### 2.1. Preparing the environment

To try this feature, we'll use a environment with simply an AKS cluster and its default node pool, in a virtual network.

**shcemainfra**

The authentication requires to be specified at the install of cilium, with the following parameters.

```yaml


```


Once the installation is complete, we can have a llok at the status of cilium.

```bash


```

And we can also look at the additional kubernetes objets: 

- A deployment for SPIRE server
- A deamonset for the agent, running on each nodes.

```bash


```

### 2.2. Using mutual authentication






## 3. Summary



