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

➜  ~ k get gateway -n shared-gw shared-gw-tls-envoy-nodeport        
NAME                           CLASS            ADDRESS         PROGRAMMED   AGE
shared-gw-tls-envoy-nodeport   envoy-nodeport   192.168.56.17   True         15h
➜  ~ k get service -n envoy-gateway-system envoy-shared-gw-shared-gw-tls-envoy-nodeport-cee0d807 
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

Through this aproach, the cluster team is able to keep ownership of the cluster wide part, and the `gateway` is included.

Refering to the listenetset documentation, we can find its main configuration in its `spec` section:

- `parentsRef` where is defined which `gateway` the listener depends on.
- `listeners`, as on the `gateway`, which allows to define listener configuration such as hostnames, ports, protocols but also TLS configuration.

An interesting information specified on this documentation is the augmented number of listeners that can be used through the `listenersets`. Indeed, with only the `listeners` section of the `gateway` there is a limitation of 64 listeners.

Another worthy of interest information is the mention of being able to configure hostnames on the listenersets. Something that can be configured on the `HTTProute`.

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

However, we never configured any hostname on the `gateway`. Checking the [API documentation](https://gateway-api.sigs.k8s.io/reference/api-spec/1.6/spec/#listener), we can find that the field does exist, and it's mentionned that if it is not specified, then all hostnames are matched.

That's why it can work the way we did it. The `gateway` match all hostnames, and we have a specific hostname on the `HTTPRoute`.

However, we'll keep that in mind for our experiment part.

As a reference, below is a table listing the different fields of the `spec` section for the `listenerset`.

| Fields | Description |
|-|-|
| `parentRef` | ParentRef references the Gateway that the listeners are attached to. |
| `listeners` | Listeners associated with this ListenerSet. Listeners define logical endpoints that are bound on this referenced parent Gateway’s addresses. Max number of items is 64, min is 1 |

And more specifically, the fields of the `listeners` section if the listenerset `spec`.

| Fields | Description |
|-|-|
| `name` | Name is the name of the Listener. This name MUST be unique within a `ListenerSet`. Max length is 253. |
| `hostname` | Hostname specifies the virtual hostname to match for protocol types that define this concept. When unspecified, all hostnames are matched. Maxlength is 253 |
| `port` | The port number. Multiple listeners can use the same port. |
| `protocol` | Network protocol |
| `tls` | Configuration for the `listener` set with protocol `HTTPS` or `TLS` |``
| `allowedRoutes` | Define the route that can be attached to the listenerset, as for the gateway. Filtering capabilities available |

Let's stop a bit here. 

Attentive readers may get the impression that `listenerSets` were not initially present in the Gateway API implementations.

And that's right. This object was made available in the standard channel only starting from v1.5.0 And that means we again have to be very careful with the compatibility matrix of the Gateway API implementation that we use.

As a matter of fact, not all implementation are `listenerSet`-ready.

Before trying some `listenerSet` configurations, we need to verufy the version of the Gateway API CRDs.

```sh

➜  ~ k get crd gatewayclasses.gateway.networking.k8s.io -o yaml |grep -i bundle-version
    gateway.networking.k8s.io/bundle-version: v1.5.1

```

Then we can check some of the GatewayClasses that we have already and see what is displayed.

```sh

➜  ~ k get gatewayclasses.gateway.networking.k8s.io nginx -o json |jq .status.supportedFeatures |grep -i listenerset -A1 -B1 

```
```json
  {
    "name": "ListenerSet"
  },

```

As the result of the command shows, some implementations include in the `status.supportedFeatures` the list of supported features. The currently installed version of nginx seems to be `listenerSet` compatible.

However, cilium does not.

```sh

➜  ~ k get gatewayclasses.gateway.networking.k8s.io cilium -o json |jq .status.supportedFeatures 
```
```json
[
  {
    "name": "BackendTLSPolicy"
  },
  {
    "name": "GRPCRoute"
  },
  {
    "name": "GRPCRouteNamedRouteRule"
  },
  {
    "name": "Gateway"
  },
  {
    "name": "GatewayAddressEmpty"
  },
  {
    "name": "GatewayFrontendClientCertificateValidationInsecureFallback"
  },
  {
    "name": "GatewayHTTPListenerIsolation"
  },
  {
    "name": "GatewayInfrastructurePropagation"
  },
  {
    "name": "GatewayPort8080"
  },
  {
    "name": "GatewayStaticAddresses"
  },
  {
    "name": "HTTPRoute"
  },
  {
    "name": "HTTPRouteBackendProtocolH2C"
  },
  {
    "name": "HTTPRouteBackendProtocolWebSocket"
  },
  {
    "name": "HTTPRouteBackendRequestHeaderModification"
  },
  {
    "name": "HTTPRouteBackendTimeout"
  },
  {
    "name": "HTTPRouteCORS"
  },
  {
    "name": "HTTPRouteDestinationPortMatching"
  },
  {
    "name": "HTTPRouteHostRewrite"
  },
  {
    "name": "HTTPRouteMethodMatching"
  },
  {
    "name": "HTTPRouteNamedRouteRule"
  },
  {
    "name": "HTTPRoutePathRedirect"
  },
  {
    "name": "HTTPRoutePathRewrite"
  },
  {
    "name": "HTTPRoutePortRedirect"
  },
  {
    "name": "HTTPRouteQueryParamMatching"
  },
  {
    "name": "HTTPRouteRequestMirror"
  },
  {
    "name": "HTTPRouteRequestMultipleMirrors"
  },
  {
    "name": "HTTPRouteRequestPercentageMirror"
  },
  {
    "name": "HTTPRouteRequestTimeout"
  },
  {
    "name": "HTTPRouteResponseHeaderModification"
  },
  {
    "name": "HTTPRouteSchemeRedirect"
  },
  {
    "name": "Mesh"
  },
  {
    "name": "MeshClusterIPMatching"
  },
  {
    "name": "MeshHTTPRouteBackendRequestHeaderModification"
  },
  {
    "name": "MeshHTTPRouteNamedRouteRule"
  },
  {
    "name": "MeshHTTPRouteQueryParamMatching"
  },
  {
    "name": "MeshHTTPRouteRedirectPath"
  },
  {
    "name": "MeshHTTPRouteRedirectPort"
  },
  {
    "name": "MeshHTTPRouteRewritePath"
  },
  {
    "name": "MeshHTTPRouteSchemeRedirect"
  },
  {
    "name": "ReferenceGrant"
  },
  {
    "name": "TLSRoute"
  },
  {
    "name": "TLSRouteModeMixed"
  }
]
```

If the `GatewayClass` does not show anything in the `status` section, we have to fall back to the documentation to verify the compatibility matrix.

In the next section, we'll rely on Envoy Gateway v1.8.2 for which we did check the compatibility of `ListenerSets` &#128527;

## 3. Experimenting with `listenersets`

Taking from our sample in part 1, we have the following gateway.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw-tls-envoy-nodeport
  namespace: shared-gw
spec:
  gatewayClassName: envoy-nodeport
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
    tls:
     certificateRefs:
     - kind: Secret
       group: ""
       name: gundamapp-tls
       namespace: certificates

```

Checking its `GatewayClass`, we can see no info displayed about the compatibility as just discussed.

```yaml

status:
  conditions:
  - lastTransitionTime: "2026-07-08T06:47:56Z"
    message: Valid GatewayClass
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted

```

At this point, we have a `gateway` that is configured to match all hostnames. 
It's also configured to accept `httproutes` from `namespaces` `gundam` and `demoapp`.
The hostname is referenced only on the `httproute`, as detailed in section 1.

We still lack something for now, but let's write a configuration for a listenerset.

Our main goal is to give more capabilities to the developer team, such as a mean to manage the hostname and certificate.

Let's take the hypothesis that the dev team will use the hostname `listenerset.app.teknews.cloud`. We'll place the `listenerset`, and another `HTTPRoute` in the app `namespace`, in this case `gundam`

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  name: gundam-listenerset-envoy
  namespace: gundam
spec:
  parentRef:
    namespace: shared-gw
    name: shared-gw-tls-envoy-nodeport
    kind: Gateway
    group: gateway.networking.k8s.io
  listeners:
  - name: first
    hostname: listenerset.app.teknews.cloud
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ""
        name: gundamapp-listenerset-tls

```

Notice the TLS section in which we specify the secret used to reference the certificate. 
With this we are reaching our aim to grant the same level of independancy we had with the `ingress` object.

Our HTTPRoute is looking like this.

```yaml

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gundam-httproute-with-listenerset-envoy
  namespace: gundam
spec:
  parentRefs:
  - name: gundam-listenerset-envoy
    kind: ListenerSet
    group: gateway.networking.k8s.io
  hostnames:
  - listenerset.app.teknews.cloud
  rules:
  - backendRefs:
    - name: titansvc
      port: 8080
      kind: Service
  - backendRefs:
    - name: evasvc
      port: 8090
      kind: Service
    matches:
    - path:
        type: PathPrefix
        value: /eva
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
```

It's referencing other kubernetes services as backend, but this is not what we are interested in.

Here, we can see that we have a match between the hostname in the `listenerset`, and the `HTTPRoute`.

If we try to create our object like this, it won't work.

Firstly because, as mentionned before, our gateway matches all hostnames, so we currently have a conflict between the listener define in our listenerset, which should allow `listenerset.app.teknews.cloud`, and our `gateway`, which does not specify any hostname, and thus allows all hostnames including `gateway.app.tekenws.cloud` referenced in our `HTTPRoute` from earlier, abut also `listenerset.app.teknews.cloud` which should only match for the `listenerset`.

Secondly, because, similarly to the `HTTPRoutes` filters, we need to specifically allow `listenerset` on the `gateway`.

So our gateway becomes somthing like this.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw-tls-envoy-nodeport
  namespace: shared-gw
spec:
  gatewayClassName: envoy-nodeport
  allowedListeners:
    namespaces:
      from: Selector
      selector:
        matchExpressions:
        - { key: kubernetes.io/metadata.name, operator: In, values: [shared-gw,gundam,demoapp] }
  listeners:
  - protocol: HTTPS
    port: 443
    name: shared-gw-tls-envoy-nodeport
    hostname: gateway.app.teknews.cloud
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchExpressions:
          - { key: kubernetes.io/metadata.name, operator: In, values: [gundam,demoapp] }
    tls:
     certificateRefs:
     - kind: Secret
       group: ""
       name: gundamapp-tls
       namespace: certificates

```

Lst but not least, the HTTPRoute must reference in its parentsRef section the listenerset, instead of the gateway, as before.

```yaml

spec:
  parentRefs:
  - name: gundam-listenerset-envoy
    kind: ListenerSet
    group: gateway.networking.k8s.io

```

With the apps defined in the `HTTPRoute` backend deployed, we have the following result for the listener define on the `gateway` level.

```bash

➜  ~ curl -k -i -X GET https://gateway.app.teknews.cloud:32086/barbatos
HTTP/2 200 
server: nginx/1.31.2
date: Sat, 11 Jul 2026 21:10:34 GMT
content-type: text/html
content-length: 288
last-modified: Wed, 08 Jul 2026 07:04:11 GMT
etag: "6a4df66b-120"
accept-ranges: bytes

<html>
<h1>Welcome to Gundam App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/AfLpq5XX4tK6TtxoWLDbd_665qDaxYgPAJKBCxVl5aE/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tLm1l/ZGlhLWFtYXpvbi5j/b20vaW1hZ2VzL0kv/NjFyYkhlLTdCbEwu/anBn" />
</html>

```

And on the listenerset level.

```bash

➜  ~ curl -k -i -X GET https://listenerset.app.teknews.cloud:32086/
HTTP/2 200 
server: nginx/1.31.2
date: Sat, 11 Jul 2026 21:10:50 GMT
content-type: text/html
content-length: 221
last-modified: Sat, 11 Jul 2026 21:09:01 GMT
etag: "6a52b0ed-dd"
accept-ranges: bytes

<html>
<h1>Welcome to Gundam App</h1>
</br>
<h2>This is a demo to illustrate Gateway API and the use of listenersets</h2>
<img src="https://i.pinimg.com/originals/5d/35/52/5d3552354ab9f1faed486a70c90d4aea.jpg" />
</html>

```

We know that for each envoy gateway, we have a corresponding deployment in the namespace.

```sh

➜  ~ k get deployments.apps -n envoy-gateway-system 
NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
envoy-gateway                                           1/1     1            1           24d
envoy-gundam-gundam-gw-8863faac                         1/1     1            1           3d11h
envoy-shared-gw-shared-gw-tls-envoy-eff02edc            1/1     1            1           3d4h
envoy-shared-gw-shared-gw-tls-envoy-nodeport-cee0d807   1/1     1            1           3d4h

```

Checking the logs of the one used by our gateway, we can see the references to our curl command from earlier.

```json

{
  ":authority": "gateway.app.teknews.cloud:32086",
  "bytes_received": 0,
  "bytes_sent": 288,
  "connection_termination_details": null,
  "downstream_local_address": "100.64.0.249:10443",
  "downstream_remote_address": "192.168.56.1:51212",
  "duration": 1,
  "method": "GET",
  "protocol": "HTTP/2",
  "requested_server_name": "gateway.app.teknews.cloud",
  "response_code": 200,
  "response_code_details": "via_upstream",
  "response_flags": "-",
  "route_name": "httproute/gundam/gundam-httproute-with-shared-gw/rule/1/match/0/gateway_app_teknews_cloud",
  "start_time": "2026-07-11T21:14:16.614Z",
  "upstream_cluster": "httproute/gundam/gundam-httproute-with-shared-gw/rule/1",
  "upstream_host": "100.64.0.45:80",
  "upstream_local_address": "100.64.0.249:53442",
  "upstream_transport_failure_reason": null,
  "user-agent": "curl/8.7.1",
  "x-envoy-origin-path": "/",
  "x-envoy-upstream-service-time": null,
  "x-forwarded-for": "192.168.56.1",
  "x-request-id": "94e559ff-9450-4965-ac37-49d36f1bb18f"
}
{
  ":authority": "gateway.app.teknews.cloud:32086",
  "bytes_received": 0,
  "bytes_sent": 288,
  "connection_termination_details": null,
  "downstream_local_address": "100.64.0.249:10443",
  "downstream_remote_address": "192.168.56.1:51215",
  "duration": 0,
  "method": "GET",
  "protocol": "HTTP/2",
  "requested_server_name": "gateway.app.teknews.cloud",
  "response_code": 200,
  "response_code_details": "via_upstream",
  "response_flags": "-",
  "route_name": "httproute/gundam/gundam-httproute-with-shared-gw/rule/1/match/0/gateway_app_teknews_cloud",
  "start_time": "2026-07-11T21:14:18.373Z",
  "upstream_cluster": "httproute/gundam/gundam-httproute-with-shared-gw/rule/1",
  "upstream_host": "100.64.0.45:80",
  "upstream_local_address": "100.64.0.249:53442",
  "upstream_transport_failure_reason": null,
  "user-agent": "curl/8.7.1",
  "x-envoy-origin-path": "/",
  "x-envoy-upstream-service-time": null,
  "x-forwarded-for": "192.168.56.1",
  "x-request-id": "8934ec06-9d1d-4da6-9f88-da5d5b976c0c"
}

```

Which shows we do have waht we want. So let's wrap for today.

## 4. Summary

The idea with this article was to have a look at the fix for something that was missing in the Gateway API.

With listenersets, app developers can now manage by themselves the listeners and certificate for their apps.

On the other hand, the `gateway` remains in the cluster team scope. 

The change to take into account here are 

- The *all hostnames match* that should go on the `gateway` listeners, replace at least by something similar to the `Ingress` default backend.
- Each `listenerset` should have its own hostname, and be allowed to on the `gateway`.
- `HTTPRoute` (and other downstream part) should have a `parentRefs` defining the `listenerset` instead of the `gateway`. Remember the schema in part 2.

There are other things to look at on `listenersets`, but for some basics, it's enough, and we now have a mean to workaround the previously lost autonomy of dev teams used to manage their certificate.

See you next time ^^.





