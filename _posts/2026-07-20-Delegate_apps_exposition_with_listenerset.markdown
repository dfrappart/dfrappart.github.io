---
layout: post
title:  "Delegate apps exposition with ListenerSet"
date:   2026-07-20 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Network
---

Hi!

After our experimentations with all the native admision controllers, we're back again on some Gateway API stuff &#128526;.

We are already familiar with basics such as `gatewayclasses`, `gateways`, `httproutes`, `referencegrants`.
And we saw previously how to expose apps specifically  with `httproutes`.

But in this article, we'll dive in `listenersets` and see what use case can be leveraged with this spceific `CRD`.

Our agenda:


1. Review of the workflow to expose apps with the Gateway API
2. What need the `listenersets` answer
3. Experimenting with `listenersets`

## 1. Review of the workflow to expose apps with the Gateway API

Before looking into the `listenersets`, we'll review the connection workflow to an application with the Gateway API.

It can be summarized with this schema.

[!illustration1](/assets/gapi/gatewayupstreamdownstream.png).

As illustrated, we have on the upstream part the `gateway`, which is the exposed part, and the access to the backend service is achieved through the `httproute`.

On the gateway, we define at least 1 listener, and this is a mandatory field, in which the port and protocol is defined.

```yaml

  listeners:
  - protocol: HTTPS
    port: 443
    name: shared-gw-tls-envoy-nodeport
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
          - { key: kubernetes.io/metadata.name, operator: In, values: [gundam,demoapp] }

```

We can also notice the filter on the allowed routes with the `allowedRoutes`.

Because we specified `HTTPS` in the protocol section, the gateway expects a `TLS` section.

```yaml

    tls:
     certificateRefs:
     - kind: Secret
       group: ""
       name: gundamapp-tls
       namespace: certificates

```

Once the gateway is running, we can see the underlying service deployed by the Gateway API provider, in this case Envoy Gateway. 

```bash

➜  ~ k $cil1 get gateway -n shared-gw shared-gw-tls-envoy-nodeport        
NAME                           CLASS            ADDRESS         PROGRAMMED   AGE
shared-gw-tls-envoy-nodeport   envoy-nodeport   192.168.56.17   True         15h
➜  ~ k $cil1 get service -n envoy-gateway-system envoy-shared-gw-shared-gw-tls-envoy-nodeport-cee0d807 
NAME                                                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
envoy-shared-gw-shared-gw-tls-envoy-nodeport-cee0d807   NodePort   100.65.138.164   <none>        443:32086/TCP   15h

```

We could also note the fact that our `gateway` uses a `NodePort` `service`, which we can configure through the `GatewayClass`.

```yaml

apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: nodeport-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: NodePort
        annotations:
          "service.beta.kubernetes.io/azure-load-balancer-internal": "true"
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-nodeport
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: nodeport-proxy
    namespace: envoy-gateway-system

```

Next we have the `httproute` which point for the upstream to the `gateway`, and configure the backend kubernetes `services`.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gundam-httproute-with-shared-gw
  namespace: gundam
spec:
  parentRefs:
  - name: shared-gw-tls-envoy-nodeport
    namespace: shared-gw
  hostnames:
  - "gateway.app.teknews.cloud"
  rules:
  - backendRefs:
    - name: gundamappsvc
      port: 8080
      kind: Service

```

We'll note that a hostname is referenced here and that should gives us access to the backend.

![illustration2](/assets/listenersets/listset001.png)

Ok fine, we already knew it worked so what's the deal with `listenersets`?

## 2. What need the `listenersets` answer

### 2.1. Understanding the changes brought by the Gateway API

To understand the need, we have to go back to the role oriented model of the Gateway API.

![illustration3](/assets/servicemesh/smesh006.png)

And the way most of the kubernetes users are used to manipulate the apps exposition, which was the `ingress`

![illustration4](/assets/svctogapi/svctogapi007.png)

One important difference between the Gateway API and the Ingress Controller is that we can have as many gateways as we want while we usually had only one controller.

At this point, it may not seem important, but if we refer to the role model, we can see that the `gateway` is more on the cluster operator scope, while the downstream objects such as the `httproute` are more app operator scope.

Which means that managing the certificate is on the gateway side, while managing the hostname is on the httproute side.

Now if we check the [documentation on the `ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/), we can find a manifest as follow, with TLS management.

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80


```

While the idea of the role oriented model remains a good idea, we just moved the responsibility to manage the certificate to the cluster operators scope (&#128561; is the face that those people will make when you tell them).

Regarding TLS management, we could effectively remove the TLS management from the apps developer, but it would means managing the different certificates and listeners on the cluster ops.

Or we could let it in the apps developer scope and have a distributed model for the Gateway, meaning one `app == one gateway`. Definitely not following the role oriented model as planned &#128565;.

On the first scenario, if we thinbk about it, we could have an approach with a wild card certificate. So every hosts for a domain name could be covered. But, apart from the fact that the wild card may not be acceptable from a security point of view, a limit remains if we have different domain for our hosts.
The other approach could be to managed different listeners, with differents hostnames, and certificates. But we are back to the change implied by the role model, and the necessity to have the cluster ops manage something they were not doing before.

And that's where we introduce the `listenerset`.

### 2.2. `ListenerSet` concepts

The proposal of the listenerset is to provide a means to keep the role oriented approach, while not puting overhead on the cluster ops.

The idea is to add another object between the downstream (in our case, mainly the `httproute`) and the upstream, a.k.a the `gateway`.

A nice schema is presented on the gateway api documentation.

![illustration5](/assets/listenersets/listset002.png)

## 3. Experimenting with `listenersets`




## 4. Summary

In this 2 part article, we were able to get a better understanding of `MutatingAdmissionPolicy` and had the oppportunity to manipulate more `CEL`.

The conclusion of all of this could be:

- `MAP` is a powerfull tool that allows to bring better control in a k8s environment. In addition to Pod Security Admissions & `ValidatingAdmissionPolicy`, it is possible to instantiate complex scenarios of control on the 
- Its concepts are simple yet the full usage relies on a (very) good understanding of the `CEL`. 




