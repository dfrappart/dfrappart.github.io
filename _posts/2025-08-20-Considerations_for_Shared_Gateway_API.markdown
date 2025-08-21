---
layout: post
title:  "Consideration for a shared Gateway"
date:   2025-08-20 18:00:00 +0200
year: 2025
categories: Kubernetes
---

Hi!

It's been a little long coming, but well, it's summer and I needed a break &#128526;

That being said, vacations are over so here we are.
We're still talking about the Gateway API, and at this point, still using Cilium's which is perfect for our use case.
This time I wanted to look at scenario where the Gateway is shared, and managed by someone else.

So the Agenda:


1. Scenario for a shared Gateway
2. Configuring HTTPRoutes and Gateways accross different namespaces
3. Managing Secrets, also in different namespaces
3. Conclusion




## 1. Scenario for a shared Gatewaay

### 1.1. thoughs on the need to share a gateway

Up till now, we managed the gateway in a distributed approach.
We had a namespace, containing manifests for apps and, living in the same namespace, a Gateway and an Httproute to expose the application.

And that works fine. But what about the segregation of duty?

Remember the schema for role based organization with the gateway API:

![illustration1](/assets/servicemesh/smesh006.png)

Considering that the namespace is often a security boundary, with rbac applied, letting all the objects related to the exposure in the same place may be not appropriate for our security requirements.

Regarding the gateway classes, those are not namespaced resources anyway, so the control of access, while still to be taken into accound, is not addressed through namespace separation.

```bash

df@df-2404lts:~$ k api-resources | grep gatewayclass
ciliumgatewayclassconfigs           cgcc                                cilium.io/v2alpha1                   true         CiliumGatewayClassConfig
gatewayclasses                      gc                                  gateway.networking.k8s.io/v1         false        GatewayClass

```

The gateway is a namespaced resource however, and it makes sense to think about a way to put it in another namespace, managed by another team.

```bash

df@df-2404lts:~$ k api-resources | grep gateways
gateways                            gtw                                 gateway.networking.k8s.io/v1         true         Gateway

```

We could imagine an organization as below:

- namespace managed by cluster operators containing the shared gateway, exposed on the external network.
- namespaces managed by apps owners, containing the Httproutes and all the other apps related resources.

Also, as a reminder, we can add annotations to the underlying service of a gateway, to make the said gateway an internal gateway:

```yaml

spec:
  gatewayClassName: cilium
  infrastructure:
    annotations:
      "service.beta.kubernetes.io/azure-load-balancer-internal": "true"

```

Taking that into account, it should be possible to apply a policy that force a gateway created in an app namespace to add the `spec.infrastructure.annotations` parameters that we want. But that's a story for another time &#x1F61D;

Last, regarding TLS configuration, which is absolutely mandatory in a real world scenario, we saw that the certificate is referenced at the gateway level.

```yaml

spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTPS
    port: 443
    name: gundam-tls-gw
    tls:
      mode: Terminate
      certificateRefs:
      - name: apptekewscloud
        kind: Secret
        namespace: gundam
        group: ""

```

We will not dig in secrets engines or sexy operators in this article. For now, we will just consider a scenario where, as for the gateway, the secrets associated to the TLS configurations are managed in another namespace.

### 1.2. A word on the lab environment

Before getting into the heart of the topic, a little bit about the lab environment.

For this article, the lab used relies ont on an AKS server but on a local kubeadm single node cluster.
I used Vagrant with a vagrant file as below:

```bash

K8SSERVER_COUNT = 1
IMAGE = "bento/ubuntu-24.04"

Vagrant.configure("2") do |config|

  (1..K8SSERVER_COUNT).each do |i|
    config.vm.define "k8sserver#{i}" do |k8sservers|
      k8sservers.vm.box = IMAGE
      k8sservers.vm.hostname = "k8scilium#{i}"
      k8sservers.vm.network  :private_network, ip: "192.168.56.#{i+16}"
      k8sservers.vm.provision "shell", privileged: true,  path: "scripts/k8s_install.sh"
      config.vm.provision "file", source: "scripts/k8s_postconfig.sh", destination: "/home/vagrant/k8s_postconfig.sh"
      config.vm.provision "file", source: "yamlconfig/ciliumgw.yaml", destination: "/home/vagrant/yamlconfig/ciliumgw.yaml"
      config.vm.provision "file", source: "yamlconfig/demoapp.yaml", destination: "/home/vagrant/yamlconfig/demoapp.yaml"
      config.vm.provision "file", source: "seccompprofiles/audit.json", destination: "/var/lib/kubelet/seccomp/profiles"
      config.vm.provision "file", source: "seccompprofiles/violation.json", destination: "/var/lib/kubelet/seccomp/profiles"
      config.vm.provision "file", source: "seccompprofiles/fine-grained.json", destination: "/var/lib/kubelet/seccomp/profiles"
    end
  end
end

```

For those who would like to have a look on the differents scripts, everyting is availble on a dedicated [github repository](https://github.com/dfrappart/k8slocal).

Additionaly, I created a dedicated gateway class.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: custom-cilium-gateway-class
spec:
  controllerName: io.cilium/gateway-controller
  description: A GatewayClass with NodePort services.
  parametersRef:
    group: cilium.io
    kind: CiliumGatewayClassConfig
    name: gateway-class-config
    namespace: ciliumgateway

```

With a Cilium CRD to force the Gateway underlying service to be of the NodePort type.

```yaml

apiVersion: cilium.io/v2alpha1
kind: CiliumGatewayClassConfig
metadata:
  name: gateway-class-config
  namespace: ciliumgateway
spec:
  service:
    type: NodePort


```

Now we need an app. We can use the same basis as in [our previous article on Httproute](/_posts/2025-06-18-Still_more_on_Gateway_API.markdown), which gives us some pod managed by a deployment, and the associated service, plus a confgmap associated to the pod configuration.

```bash

df@df-2404lts:~$ k get all -n gundam 
NAME                            READY   STATUS    RESTARTS   AGE
pod/barbatos-5798674fd7-kcsck   1/1     Running   0          3m16s
pod/barbatos-5798674fd7-nwvpt   1/1     Running   0          3m16s
pod/barbatos-5798674fd7-s6twk   1/1     Running   0          3m16s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/barbatossvc   ClusterIP   100.65.120.78   <none>        8090/TCP   3m16s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/barbatos   3/3     3            3           3m16s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/barbatos-5798674fd7   3         3         3       3m16s

df@df-2404lts:~$ k describe configmaps -n gundam index-html-barbatos 
Name:         index-html-barbatos
Namespace:    gundam
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
<html>
<h1>Welcome to Barbatos App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/AfLpq5XX4tK6TtxoWLDbd_665qDaxYgPAJKBCxVl5aE/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tLm1l/ZGlhLWFtYXpvbi5j/b20vaW1hZ2VzL0kv/NjFyYkhlLTdCbEwu/anBn" />
</html



BinaryData
====

Events:  <none>

```

If we want to expose this app, we define a gateway as below:

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gundam-gw
  namespace: gundam
spec:
  gatewayClassName: custom-cilium-gateway-class
  listeners:
  - protocol: HTTP
    port: 80
    name: gundam-gw

```

And an httproute like this:

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gundam-httproute
  namespace: gundam
spec:
  parentRefs:
  - name: gundam-gw
  rules:
  - backendRefs:
    - name: barbatossvc
      port: 8090
      kind: Service
    matches:
    - path:
        type: PathPrefix
        value: /barbatos
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /

```

Because we are on a single kubeadm cluster, we used, as said earlier, a `gateway-class-config`, specific to Cilium Gateway API to change the underlying service from a `LoadBalancer` to a `NodePort`

```bash

df@df-2404lts:~$ k get service -n ciliumgateway 
NAME                            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
cilium-gateway-cilium-gateway   NodePort   100.65.91.189   <none>        443:30246/TCP   27h

```

If the gateway is accessible externaly, we should get something like this with a curl command:

```bash

df@df-2404lts:~$ curl -i -X GET http://192.168.56.17:30977/barbatos
HTTP/1.1 200 OK
server: envoy
date: Wed, 20 Aug 2025 11:56:59 GMT
content-type: text/html
content-length: 289
last-modified: Wed, 20 Aug 2025 10:02:42 GMT
etag: "68a59d42-121"
accept-ranges: bytes
x-envoy-upstream-service-time: 0

<html>
<h1>Welcome to Barbatos App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/AfLpq5XX4tK6TtxoWLDbd_665qDaxYgPAJKBCxVl5aE/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tLm1l/ZGlhLWFtYXpvbi5j/b20vaW1hZ2VzL0kv/NjFyYkhlLTdCbEwu/anBn" />
</html

```

However, at this point, we still host the gateway in the same namespace as the Httproute and the application, which is not our target. 

Ok, let's go into to topic and get started with the shared gateway part.

## 2. Configuring HTTPRoutes and Gateways accross different namespaces

### 2.1. Looking at some of the Gateway properties

To achieve our goal, we need to dig in the API specifications.

We can find in the [`spec.listeners` description](https://gateway-api.sigs.k8s.io/reference/spec/#listener) a reference to the `allowedRoutes` which contains a `namespace` field. Its default value, as displayed in the below table is `Same` which means that by default, it configures the gateway to accept Httproutes from the same namespace.

| Field	| Description	| Default	|
|-|-|-|
| `namespaces` |	Namespaces indicates namespaces from which Routes may be attached to this Listener. This is restricted to the namespace of this Gateway by default. | { from:Same }	|

Following the links in the documentation, we find that the accepted values.

| Field	| Description |
|-|-|
| `All` | Routes/ListenerSets in all namespaces may be attached to this Gateway. |
| `Selector` | Only Routes/ListenerSets in namespaces selected by the selector may be attached to this Gateway. |
| `Same` | Only Routes/ListenerSets in the same namespace as the Gateway may be attached to this Gateway. |
| `None`| No Routes/ListenerSets may be attached to this Gateway. |

Ok, let's create a new gateway API, this time, in another namespace.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: ciliumgateway
spec:
  gatewayClassName: custom-cilium-gateway-class
  listeners:
  - protocol: HTTP
    port: 80
    name: shared-gw

```

We should have the following objects after applying the manifests.

```bash

df@df-2404lts:~$ k get service -n ciliumgateway cilium-gateway-
cilium-gateway-cilium-gateway  cilium-gateway-shared-gw       
df@df-2404lts:~$ k get service -n ciliumgateway cilium-gateway-shared-gw 
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
cilium-gateway-shared-gw   NodePort   100.65.216.238   <none>        80:32708/TCP   37s

```

If we update the httproute created earlier with the reference to this new gateway.

```yaml

spec:
  parentRefs:
  - name: shared-gw
    namespace: ciliumgateway

```

We get the following status.

```bash

Status:
  Parents:
    Conditions:
      Last Transition Time:  2025-08-20T17:04:32Z
      Message:               HTTPRoute is not allowed to attach to this Gateway due to namespace restrictions
      Observed Generation:   1
      Reason:                NotAllowedByListeners
      Status:                False
      Type:                  Accepted
      Last Transition Time:  2025-08-20T17:04:32Z
      Message:               Service reference is valid
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         io.cilium/gateway-controller
    Parent Ref:
      Group:      gateway.networking.k8s.io
      Kind:       Gateway
      Name:       shared-gw
      Namespace:  ciliumgateway
Events:           <none>

```

Which makes sense because we did not specify yet the appropriate parameters to make our gateway a shared gateway.
Let's do this. As found earlier, we need to add the following in the listener configuration.

```yaml

    allowedRoutes:
      namespaces:
        from: All

```

And this time, the status shows us that the gateway accepted the Httproute.

```bash

Status:
  Parents:
    Conditions:
      Last Transition Time:  2025-08-20T17:16:32Z
      Message:               Accepted HTTPRoute
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2025-08-20T17:04:32Z
      Message:               Service reference is valid
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         io.cilium/gateway-controller
    Parent Ref:
      Group:      gateway.networking.k8s.io
      Kind:       Gateway
      Name:       shared-gw
      Namespace:  ciliumgateway
Events:           <none>

```

And it's reflected by the result of a curl command on the node.

```bash

df@df-2404lts:~$ curl http://192.168.56.17:32708/barbatos
<html>
<h1>Welcome to Barbatos App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/AfLpq5XX4tK6TtxoWLDbd_665qDaxYgPAJKBCxVl5aE/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tLm1l/ZGlhLWFtYXpvbi5j/b20vaW1hZ2VzL0kv/NjFyYkhlLTdCbEwu/anBn" />
</html

```

However, allowing Httproutes from any namespace is a bit much. We should be able to use the `Selector` instead to reference the namespaces that we want to specifically allow.

Checking the `gundam` namespace, we can get the default label.

```bash

df@df-2404lts:~$ k get namespace --show-labels gundam 
NAME     STATUS   AGE     LABELS
gundam   Active   7h33m   kubernetes.io/metadata.name=gundam

```

We can change the gateway listener to this.

```yaml

    allowedRoutes:
      namespaces:
        #from: All
        from: Selector
        selector:
          matchLabels:
            kubernetes.io/metadata.name: "gundam"

```

The status is not changed, and the curl command result is still the same. But you can test on your own if you don't believe me ^^

Now we will add another app in a new namespace called `demoapp`.

```bash

df@df-2404lts:~$ k get all -n demoapp 
NAME                          READY   STATUS    RESTARTS        AGE
pod/demoapp-d67cd5b89-bjxst   1/1     Running   3 (7h41m ago)   30h
pod/demoapp-d67cd5b89-hjwp9   1/1     Running   3 (7h41m ago)   30h
pod/demoapp-d67cd5b89-qmdp2   1/1     Running   3 (7h41m ago)   30h

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/demoappsvc   ClusterIP   100.65.98.75   <none>        8080/TCP   30h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demoapp   3/3     3            3           30h

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/demoapp-d67cd5b89   3         3         3       30h

```

with its associated Httproute.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-httproute
  namespace: demoapp
spec:
  parentRefs:
  - name: shared-gw
    namespace: ciliumgateway
  hostnames:
  - "k8scilium1"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: demoappsvc
      port: 8080

```

The status shows that the gateway does not accept the route due to selector restrictions.

```bash

Conditions:
      Last Transition Time:  2025-08-20T17:28:44Z
      Message:               HTTPRoute is not allowed to attach to this Gateway due to namespace selector restrictions

```

It's interesting to note that, while the selector field can be used to provide more than one label, it works as an `AND` base operator. 
So to add Httproutes from the `demoapp` namespace in the allowed list, setting the listener as below does not work.

```yaml

from: Selector
        selector:
          matchLabels:
            kubernetes.io/metadata.name: "gundam"
            kubernetes.io/metadata.name: "demoapp"
        
```

Here, it's because it's not possible to have the `kubernetes.io/metadata.name` with both the values `gundam` and `demoapp`.
We could add another labels though, like this.

But since `matchLabels` acts as an `AND`, it would not work, neither for the `gundam` namespace, nor the `demoapp` namespace.

Instead, the equivalent of an `OR` expression in the selector section looks like this

```yaml

from: Selector
        selector:
          #matchLabels:
          #  kubernetes.io/metadata.name: "gundam"
          matchExpressions:
          - { key: kubernetes.io/metadata.name, operator: In, values: [gundam,demoapp] }

```

Ok, that's it, the gateway is shared between either all namespaces, or between a set of namespaces, depending of the `from` section in the `spec.listeners[].allowedRoutes.namespaces`.

Let's have a look at the TLS part now.

## 3. Managing Secrets, also in different namespaces

With the Gateway APi, TLS management is done on the Gateway level. The `listeners[].protocol` have to be set to `HTTPS` and the `listeners[].port` is usually to `443`.

Also, the `tls` section contains the information for the certificate. 
A gateway configured with a listener with tls look like this, with its associated secret.

```yaml

apiVersion: v1
data:
  tls.crt: LS0t===Truncated===tLQo=
  tls.key: LS0t===Truncated===tLQo=
kind: Secret
metadata:
  name: k8scilium1tls
  namespace: ciliumgateway
type: kubernetes.io/tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw-tls
  namespace: ciliumgateway
spec:
  gatewayClassName: custom-cilium-gateway-class
  listeners:
  - protocol: HTTPS
    port: 443
    name: cilium-gateway
    from: Selector
    selector:
      matchExpressions:
      - { key: kubernetes.io/metadata.name, operator: In, values: [gundam,demoapp] }
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: k8scilium1tls
---

```

When the certificate is referenced like this, it implies that the secret is in the same namespace as the gateway, as stated in the documentation extract below.


| Field |	Description |	Default |
|-|-|-|
| `group` |	Group is the group of the referent. For example, "gateway.networking.k8s.io". When unspecified or empty string, core API group is inferred. ||
| `kind` |	Kind is kind of the referent. For example "Secret". |	Secret |
| `name` |	Name is the name of the referent. 	||
| `namespace` | Namespace is the namespace of the referenced object. When unspecified, the local namespace is inferred. Note that when a namespace different than the local namespace is specified, a `ReferenceGrant` object is required in the referent namespace to allow that namespace's owner to accept the reference. See the `ReferenceGrant` documentation for details. ||

To perform further tests, we'll create a new namespace and recreat the certificate in this namespace.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: certificates
spec: {}
status: {}
---
apiVersion: v1
data:
  tls.crt: LS0t===Truncated===tLQo=
  tls.key: LS0t===Truncated===tLQo=
kind: Secret
metadata:
  creationTimestamp: null
  name: k8scilium1tls
  namespace: certificates
type: kubernetes.io/tls

```

And add the `namespace` field to the gateway listener.

```yaml

    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: k8scilium1tls
        namespace: certificates

```

Checking the status, we can see that the reference to the certificate is not allowed.

```bash

Status:
  Conditions:
    Last Transition Time:  2025-08-21T12:26:43Z
    Message:               Gateway successfully scheduled
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2025-08-21T12:26:43Z
    Message:               Gateway successfully reconciled
    Observed Generation:   1
    Reason:                Programmed
    Status:                True
    Type:                  Programmed
  Listeners:
    Attached Routes:  0
    Conditions:
      Last Transition Time:  2025-08-21T12:26:43Z
      Message:               Invalid CertificateRef
      Reason:                Invalid
      Status:                False
      Type:                  Programmed
      Last Transition Time:  2025-08-21T12:26:43Z
      Message:               Listener Accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2025-08-21T12:26:43Z
      Message:               CertificateRef is not permitted
      Reason:                RefNotPermitted
      Status:                False
      Type:                  ResolvedRefs
    Name:                    cilium-gateway
    Supported Kinds:
      Group:  gateway.networking.k8s.io
      Kind:   HTTPRoute

```

Which is as expected, because we did not use a `ReferenceGrant` as specified in the documentation. This object is used to specify which objects can refer to another object. In our case, we want to allow the gateway to reference secrets located in another namespace, namely, the `certificates` namespace, so we get this.

```yaml

apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-ciliumgateway-to-ref-secrets
  namespace: certificates
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: ciliumgateway
  to:
  - group: ""
    kind: Secret  

```

Let's curl the cluster on its node port to validate that everything works.

```bash

df@df-2404lts:~$ curl -k -i -X GET https://192.168.56.17:31487/barbatos
HTTP/1.1 200 OK
server: envoy
date: Thu, 21 Aug 2025 12:43:08 GMT
content-type: text/html
content-length: 289
last-modified: Wed, 20 Aug 2025 10:02:42 GMT
etag: "68a59d42-121"
accept-ranges: bytes
x-envoy-upstream-service-time: 0

<html>
<h1>Welcome to Barbatos App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/AfLpq5XX4tK6TtxoWLDbd_665qDaxYgPAJKBCxVl5aE/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tLm1l/ZGlhLWFtYXpvbi5j/b20vaW1hZ2VzL0kv/NjFyYkhlLTdCbEwu/anBn" />
</html

df@df-2404lts:~$ curl -k -i -X GET https://k8scilium1:31487/
HTTP/1.1 200 OK
server: envoy
date: Thu, 21 Aug 2025 12:43:50 GMT
content-type: text/html
content-length: 615
last-modified: Wed, 13 Aug 2025 14:33:41 GMT
etag: "689ca245-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 0

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

The attentive readers may have noticed that the curl is done once on the IP and the other time on the hostname. That's because the Httproute in one case reference a value for the hostname, while it does not in the other case.
Lastly, all of the hostname may work with the gateway because we did not specified any value in its configuration.

Ok that's all for today ^^

## 4. Summary

Going further on the Gateway API usage, this time we explored a shared gateway scenario and what it implies. We saw that the gateway can be configured quite easily to be more selective on which namespace the Httproutes should origin from.
And last we saw the `ReferenceGrant` that is used to list which objects, and from where, can reference another object in another namespace. 

Hope it was useful. IT was for me anyway &#128568;







