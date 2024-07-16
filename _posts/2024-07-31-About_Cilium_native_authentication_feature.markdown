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

Authentication means, at one point, Identity. 
As hinted in the smi spec earlier, and documented in the [Cilium doc](https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication/#identity-management), the technology used is [SPIFFE](https://spiffe.io/), which stands for Secure Production Identity Framework For Everyone. It is also a [graduated project in the CNCF](https://www.cncf.io/projects/spiffe/) since August 2022.
Spiffe spec are defined on a [dedicated github](https://github.com/spiffe/spiffe).

At this point we will remains at a high level overview of the SPIFFE standards, as described on the github repo, which is comprised of:

- The SPIFFE ID, a structured string (represented as a URI) which serves as the "name" of an entity.
- The SPIFFE Verifiable Identity Document (SVID) is a document which carries the SPIFFE ID itself. It is the functional equivalent of a passport - a document which is presented that carries the identity of the presenter.
- The Workload API is the method through which workloads, or compute processes, obtain their SVID(s). It is typically exposed locally (eg. via a Unix domain socket), and explicitly does not include an authentication handshake or authenticating token from the workload.

So that's the SPIFFE standard, but there is a lot to do yet, because, it's only the standard. Fortunately, there is also the SPIRE project, another graduated CNCF project, which is a production ready implementation of the SPIFFE standard.




## 2. Trying Cilium authentication





## 3. Summary



