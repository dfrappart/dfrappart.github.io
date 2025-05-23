---
layout: post
title:  "Exposing apps in Kubernetes: from services to Gateway API"
date:   2025-05-20 18:00:00 +0200
year: 2025
categories: Aks Security Network
---

Hi!

Following the last post about exposing apps in kubernetes, we're going to dig a little more into the gateway API capabilities.
In this article, we want first to have a look at the options that are available for a Gateway API implementation.
Then we'll focus mainly on 2 of the basics Gateway API objects : the gateway class, and the gateway.

The Agenda:


1. A few Gateway API providers (that we may look)
2. More on the Gateway class object
3. More on the Gateway object
4. Conclusion


Let's get started!


## 1. A few Gateway API providers (that we may look) and what is needed before working with any of them

We start discussing about Gateway API in a previous article about applications exposition.
As mentionned in the [kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/), the Ingress API is now frozen, and the Gateway API is the official replacement to all the Ingress implementations currently used.
Again, we can find infoirmation about the available implementation of the Gateway API in the [documentation](https://gateway-api.sigs.k8s.io/implementations/).
Present in this list are, among others: 

- Nginx Gateway Fabric
- Traefik Proxy
- Istio
- Application Gateway for Containers
- Cilium

Apart from Cilium that is still in Beta state, all those other providers mentionned here are in the GA state.
We could also note Hashicorp Consul which can be an interesting option in non-full Kubernetes environment (again, among others...).

Now about the installation, there are some prerequisites which are the CRDs related to the gateway API. The corresponding objects are:

- The Gateway Class
- The Gateway
- The HTTP Route

Additionally, there are also the following objects: 

- The gRPC route
- The TLS route
- The reference grant

It's important to note that all those resources are not at the same state.

Details on the CRDs definition are available on the relate [github](https://github.com/kubernetes-sigs/gateway-api/tree/main/config/crd/standard) repository.

![illustration1](/assets/gapi/github.png)

You'll notice that the stable version is currently 1.3.

We can check the presence of the CRDs on the lcuster with a kubectl get command.

```bash

df@df-2404lts:~$ k get crd | grep gateway.networking.k8s.io

gatewayclasses.gateway.networking.k8s.io                     2025-03-28T13:47:53Z
gateways.gateway.networking.k8s.io                           2025-03-28T13:47:55Z
grpcroutes.gateway.networking.k8s.io                         2025-03-28T13:48:00Z
httproutes.gateway.networking.k8s.io                         2025-03-28T13:47:57Z
referencegrants.gateway.networking.k8s.io                    2025-03-28T13:47:58Z
tlsroutes.gateway.networking.k8s.io                          2025-03-28T13:48:01Z

```

Once thispoint validated it's time to install a gateway provider.

With a cluster using Cilium, the installation can be done with the additional argument

```bash

helm upgrade cilium ./cilium \
    --namespace kube-system \
    --reuse-values \
    --set kubeProxyReplacement=true \
    --set gatewayAPI.enabled=true

```

If the installation is complete, and after restarting the cilium pods, we should have the cilium gateway class available

```bash

df@df-2404lts:~$ kubectl get gatewayclasses.gateway.networking.k8s.io cilium -o yaml


```

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  annotations:
    meta.helm.sh/release-name: cilium
    meta.helm.sh/release-namespace: kube-system
  creationTimestamp: "2025-03-28T13:50:30Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: cilium
  resourceVersion: "112501"
  uid: 143c4014-2854-4a5c-b6ec-210a0530be0a
spec:
  controllerName: io.cilium/gateway-controller
  description: The default Cilium GatewayClass
status:
  conditions:
  - lastTransitionTime: "2025-03-28T13:51:50Z"
    message: Valid GatewayClass
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted

```

To install any Gateway API provider, while the kubernetes/gateway API provide good guidance, it is recommended to check the provider documentation to validate which CRDs specifications are supported.

For instance, looking at [Traefik doc](https://doc.traefik.io/traefik/v3.2/routing/providers/kubernetes-gateway/), one could find that the suppported specifications are limited to the `1.2.1` version.

![illustration2](/assets/gapi/traefik.png)

The same can be applied to Cilium where we can find that the supported version is `1.2.0`.

So that's one warning point: Validate the CRDs versions is the one supported by the Gateway API proovider.

Now let's have a look at the objects and do some stuff.

## 2. More on the Gateway class object


### 2.1. A bit of concepts


Installing a Gateway API provider will give us a gateway class.
With an AKS configured with Cilium, we have the following default Gateway class:

```bash

df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io 
NAME      CONTROLLER                                   ACCEPTED   AGE
cilium    io.cilium/gateway-controller                 True       53d

f@df-2404lts:~/Documents/dfrappart.github.io$ k describe gatewayclasses.gateway.networking.k8s.io cilium 
Name:         cilium
Namespace:    
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: cilium
              meta.helm.sh/release-namespace: kube-system
API Version:  gateway.networking.k8s.io/v1
Kind:         GatewayClass
Metadata:
  Creation Timestamp:  2025-03-28T13:50:30Z
  Generation:          1
  Resource Version:    112501
  UID:                 143c4014-2854-4a5c-b6ec-210a0530be0a
Spec:
  Controller Name:  io.cilium/gateway-controller
  Description:      The default Cilium GatewayClass
Status:
  Conditions:
    Last Transition Time:  2025-03-28T13:51:50Z
    Message:               Valid GatewayClass
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
Events:                    <none>

df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io cilium -o yaml

```

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  annotations:
    meta.helm.sh/release-name: cilium
    meta.helm.sh/release-namespace: kube-system
  creationTimestamp: "2025-03-28T13:50:30Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: cilium
  resourceVersion: "112501"
  uid: 143c4014-2854-4a5c-b6ec-210a0530be0a
spec:
  controllerName: io.cilium/gateway-controller
  description: The default Cilium GatewayClass
status:
  conditions:
  - lastTransitionTime: "2025-03-28T13:51:50Z"
    message: Valid GatewayClass
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted

```

Looking at the yaml configuration, we can see in the `status` section important information about... the gateway class status &#129325;
The `conditions` section should display `type: Accepted` and `status: "True"`.

Also, in the `spec` section, wee can find the controllerName parameter which specifies which provider is used. Here, we note the reference to Cilium with `io.cilium/gateway-controller`.

Refering the the schema of the role based organization of the Gateway API, it is considered a fact that the gateway class ismanaged by the infrastructure provider

![illustration3](/assets/gapi/gapiresourcemodel.png)

Thus, as is defined in the documentation, it make sense to imagine an Infrastructure provider defining one class for public traffic and another one for private traffic.

```yaml

kind: GatewayClass
metadata:
  name: internet
  ...
---
kind: GatewayClass
metadata:
  name: private
  ...

```

LEt's have a look at the [API object](https://gateway-api.sigs.k8s.io/reference/spec/#gatewayclass) to find what we could do to further configure a gateway class.

The main section are as below

| Field | Description | Value |
|-|-|-|
| apiVersion | A string defining as usual the API version. | `gateway.networking.k8s.io/v1` |
| kind | Another string defining the object type. | `GatewayClass`|
| metadata | The metadata section contains information related to the identification of the object, such as the name||
| spec | The spec section contains information related to the object specifications | |

Let's look a little more on the `spec` section sub parameters.

| Field | Description | Value |
|-|-|-|
| controllerName | A string defining the controller name | Spoecific to the Gateway API provider. For Cilium `io.cilium/gateway-controller` |
| parametersRef | Another sub-section to further define the gateway class, specific to the controller | |
| description | A string that allow to describe the class ||

The `parametersRef` sub-section is used for provider specific configuration and can refers either to a CRD or a configmap.
In the case of Cilium, there is indeed a CRD called `CiliumGatewayClassConfig` that can be used to feed specific parameters to the `parametersRef` section

Now let's manipulate a little the gateway class and all those parameters.


### 2.2. Playing with GatewayClass

We identified previously the default Cilium Gateway class with a kubectl command.
Let's try to create additional gateway classes, for internal and external traffic

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: external-gateway-class
spec:
  controllerName: io.cilium/gateway-controller
  description: A Cilium GatewayClass for external traffic
---  
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: internal-gateway-class
spec:
  controllerName: io.cilium/gateway-controller
  description: A Cilium GatewayClass for internal traffic

```

We shoudl have now 2 additional classes.

```bash

df@df-2404lts:~/Documents/myrepo/AKSPerso$ k get gatewayclasses.gateway.networking.k8s.io 
NAME                     CONTROLLER                     ACCEPTED   AGE
cilium                   io.cilium/gateway-controller   True       54d
external-gateway-class   io.cilium/gateway-controller   True       9s
internal-gateway-class   io.cilium/gateway-controller   True       8s

```

However, athits point, there is nothing that allows us to specify in our classes if the child gateways will be public or private.

Because we're using Cilium, we will refers the available CRD `CiliumGatewayClassConfig`. The spec section of this CRD, shown on the [Cilium documentation](https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/parameterized-gatewayclass/), look like this.


```yaml

spec:
  description: Spec is a human-readable of a GatewayClass configuration.
  properties:
    description:
      description: Description helps describe a GatewayClass configuration
        with more details.
      maxLength: 64
      type: string
    service:
      description: |-
        Service specifies the configuration for the generated Service.
        Note that not all fields from upstream Service.Spec are supported
      properties:
        allocateLoadBalancerNodePorts:
          description: Sets the Service.Spec.AllocateLoadBalancerNodePorts
            in generated Service objects to the given value.
          type: boolean
        externalTrafficPolicy:
          default: Cluster
          description: Sets the Service.Spec.ExternalTrafficPolicy in generated
            Service objects to the given value.
          type: string
        ipFamilies:
          description: Sets the Service.Spec.IPFamilies in generated Service
            objects to the given value.
          items:
            description: |-
              IPFamily represents the IP Family (IPv4 or IPv6). This type is used
              to express the family of an IP expressed by a type (e.g. service.spec.ipFamilies).
            type: string
          type: array
          x-kubernetes-list-type: atomic
        ipFamilyPolicy:
          description: Sets the Service.Spec.IPFamilyPolicy in generated
            Service objects to the given value.
          type: string
        loadBalancerClass:
          description: Sets the Service.Spec.LoadBalancerClass in generated
            Service objects to the given value.
          type: string
        loadBalancerSourceRanges:
          description: Sets the Service.Spec.LoadBalancerSourceRanges in
            generated Service objects to the given value.
          items:
            type: string
          type: array
          x-kubernetes-list-type: atomic
        loadBalancerSourceRangesPolicy:
          default: Allow
          description: |-
            LoadBalancerSourceRangesPolicy defines the policy for the LoadBalancerSourceRanges if the incoming traffic
            is allowed or denied.
          enum:
          - Allow
          - Deny
          type: string
        trafficDistribution:
          description: Sets the Service.Spec.TrafficDistribution in generated
            Service objects to the given value.
          type: string
        type:
          default: LoadBalancer
          description: Sets the Service.Spec.Type in generated Service objects
            to the given value.
          type: string
      type: object

```

We can also note that the resource is namespaced, and that its short name is `cgcc`

```yaml

scope: Namespaced

```

```yaml

shortNames:
  - cgcc

```

Because we can apparently specify the type of service, let's create a `CiliumGatewayClassConfig` and change the kind of service to `ClusterIP`

```yaml

apiVersion: cilium.io/v2alpha1
kind: CiliumGatewayClassConfig
metadata:
  name: internal-gateway-class-config
  namespace: default
spec:
  service:
    type: ClusterIP
  

```

We also need to update the gateway class.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: internal-gateway-class
spec:
  controllerName: io.cilium/gateway-controller
  description: A Cilium GatewayClass for internal traffic
  parametersRef:
    group: cilium.io
    kind: CiliumGatewayClassConfig
    name: custom-gateway-class  

```

At this point, we have seen how to create a gateway class, and with the use of a dedicated CRD, how to add some configurations.

However, it seems there is no simple way to passe Cloud provider annotations, such as `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` so that the gateways can inherit it from the gateway class.
One could say that the gataway class is more akeen to the ingress class, and thus it may be more on the gateway side that we can configure such annotations.
So let's move on to the gateway object.

## 3. More on the Gateway object

Let's dig a little bit deeper into the Gateway object.

### 3.1. Gateway basic concepts

AS displayed on the below schema, the gateway is the first element that we access to when we try to reach an app on a kubernetes environment.

![illustration3](/assets/gapi/gateway.png)

With this, and the role based organization schemla, it's clear that we diverge from the ingress concepts.

Indeed, the ingress controller defined an ingress class an a unique entry point while the gateway class can be parent to many different gateways.

Below is a sample gateway definition, using the default cilium gateway class.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: test-gw
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: test-gw
    allowedRoutes:
      namespaces:
        from: Same 

```

Creating the gateway will result in the creation of an associated service, prefixed with `cilium-gateway-`.

```bash

df@df-2404lts:~$ k get gateway
NAME      CLASS    ADDRESS        PROGRAMMED   AGE
test-gw   cilium   4.156.201.47   True         39s

df@df-2404lts:~$ k get service
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
cilium-gateway-test-gw   LoadBalancer   100.67.19.53   4.156.201.47   80:31947/TCP   44s

```

Looking at the service details, we can see that it is by default a `LoadBalancer` type.

```bash

df@df-2404lts:~$ k describe service cilium-gateway-test-gw 
Name:                     cilium-gateway-test-gw
Namespace:                default
Labels:                   gateway.networking.k8s.io/gateway-name=test-gw
                          io.cilium.gateway/owning-gateway=test-gw
Annotations:              <none>
Selector:                 <none>
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       100.67.19.53
IPs:                      100.67.19.53
LoadBalancer Ingress:     4.156.201.47
Port:                     port-80  80/TCP
TargetPort:               80/TCP
NodePort:                 port-80  31947/TCP
Endpoints:                
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  96s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   81s   service-controller  Ensured load balancer

```

We can also check the `metadata.ownerReferences` and see that the service does refer to the gateway.

```bash

df@df-2404lts:~$ k get service cilium-gateway-test-gw -o json | jq .metadata.ownerReferences
[
  {
    "apiVersion": "gateway.networking.k8s.io/v1",
    "controller": true,
    "kind": "Gateway",
    "name": "test-gw",
    "uid": "c5811794-2aea-4dc4-96ef-94e124abbd59"
  }
]

```

Let's play a bit with the previously creatred class and its custom configuration now.

### 3.2. Using a custom class with a custom config

In this section, we want to test how our `CiliumGatewayClassConfig` may impact the creation of a gateway and the underlying service.

We have the internal gateway class, and its custom config already existing on the cluster.

```bash

df@df-2404lts:~$ k describe gatewayclasses.gateway.networking.k8s.io internal-gateway-class 
Name:         internal-gateway-class
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         GatewayClass
Metadata:
  Creation Timestamp:  2025-05-22T12:46:47Z
  Generation:          2
  Resource Version:    5771734
  UID:                 05261898-84d7-4067-a6f4-f1b7ff71a669
Spec:
  Controller Name:  io.cilium/gateway-controller
  Description:      A Cilium GatewayClass for internal traffic
  Parameters Ref:
    Group:      cilium.io
    Kind:       CiliumGatewayClassConfig
    Name:       internal-gateway-class-config
    Namespace:  default
Status:
  Conditions:
    Last Transition Time:  2025-05-22T14:41:03Z
    Message:               Valid GatewayClass
    Observed Generation:   2
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
Events:                    <none>
df@df-2404lts:~$ k describe cgcc internal-gateway-class-config 
Name:         internal-gateway-class-config
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2alpha1
Kind:         CiliumGatewayClassConfig
Metadata:
  Creation Timestamp:  2025-05-22T14:41:03Z
  Generation:          3
  Resource Version:    6328937
  UID:                 1df57def-9c80-4706-b8dd-37900dd2089a
Spec:
  Service:
    External Traffic Policy:             Cluster
    Load Balancer Source Ranges Policy:  Allow
    Type:                                ClusterIP
Events:                                  <none>

```

And the following definition for a gateway.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gw-testclusterip
spec:
  gatewayClassName: internal-gateway-class
  listeners:
  - protocol: HTTP
    port: 80
    name: gw-testclusterip
    allowedRoutes:
      namespaces:
        from: Same 

```

Let's apply this and see.

```bash

df@df-2404lts:~$ k get gateway
NAME               CLASS                    ADDRESS        PROGRAMMED   AGE
gw-testclusterip   internal-gateway-class   4.156.38.14    True         108s
test-gw            cilium                   4.156.201.47   True         55m
df@df-2404lts:~$ k get service
NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
cilium-gateway-gw-testclusterip   LoadBalancer   100.67.221.210   4.156.38.14    80:30850/TCP   115s
cilium-gateway-test-gw            LoadBalancer   100.67.19.53     4.156.201.47   80:31947/TCP   55m
kubernetes                        ClusterIP      100.67.0.1       <none>         443/TCP        56d

```

Well, it seems that the `CiliumGatewayClassConfig` did not work as expected. But it may also be related to the nature of thegateway, which may not be cluster internal &#129335;

Anyway, comparing it to the Ingress Controller implementation which allows us to pass annotations for the underlying service, it is not satisfying.

Since we can access to the service, we may be able to add manually the annotation. Let's try this. Here is a new gateway definition.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gw-internal-manual
  namespace: default
spec:
  gatewayClassName: cilium
  listeners:
  - allowedRoutes:
      namespaces:
        from: Same
    name: cilium-gw-internal-manual
    port: 80
    protocol: HTTP


```

As before we get the corresponding service, prefixed with `cilium-gateway-` wioth the type `LoadBalancer`

```bash

df@df-2404lts:~$ k get gateway cilium-gw-internal-manual 
NAME                        CLASS    ADDRESS         PROGRAMMED   AGE
cilium-gw-internal-manual   cilium   4.236.239.187   True         66s
df@df-2404lts:~$ k get service cilium-gateway-cilium-gw-internal-manual -o wide
NAME                                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE   SELECTOR
cilium-gateway-cilium-gw-internal-manual   LoadBalancer   100.67.101.42   4.236.239.187   80:32037/TCP   69s   <none>

```

```bash
df@df-2404lts:~$ k edit service cilium-gateway-cilium-gw-internal-manual 
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
  creationTimestamp: "2025-05-23T10:32:15Z"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  labels:
    gateway.networking.k8s.io/gateway-name: cilium-gw-internal-manual
    io.cilium.gateway/owning-gateway: cilium-gw-internal-manual
  name: cilium-gateway-cilium-gw-internal-manual
  namespace: default
  ownerReferences:
  - apiVersion: gateway.networking.k8s.io/v1
    controller: true
    kind: Gateway
    name: cilium-gw-internal-manual
    uid: 5365172f-fddb-4aef-80f2-86e8c68bbe03
  resourceVersion: "6341722"
  uid: 6c1b6eef-a5ec-4a28-ad80-63cc7e2a7379
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 100.67.101.42
  clusterIPs:
  - 100.67.101.42
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:

service/cilium-gateway-cilium-gw-internal-manual edited


```

After some time, the service is updated and is using an internal loadbalancer.

```bash

df@df-2404lts:~$ k get service cilium-gateway-cilium-gw-internal-manual 
NAME                                       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
cilium-gateway-cilium-gw-internal-manual   LoadBalancer   100.67.101.42   172.21.17.13   80:32037/TCP   7m10s

```

But that's not really efficient. So let's have amore thorough look at the gateway object to find out if we can be more accurate in our configuration


### 3.3. The gateway object definition

Similarly to the gateway class object, we can find the details of the available confuiguration on the gateway api documentation, in the gateway [section](https://gateway-api.sigs.k8s.io/reference/spec/#gateway).

In the Gateway `spec` section, we can find a sub-section named `infrastructure` in which we can specify `AnnotationKey` and `AnnotationValue`. Those annotations, per the filed description, are applied to resources created following the gateway creation.

| Field | Description | Validation |
|-|-|-|
| annotations object (keys, AnnotationValue) | Annotations that SHOULD be applied to any resources created in response to this Gateway. </br> For implementations creating other Kubernetes objects, this should be the metadata.annotations field on resources. </br>For other implementations, this refers to any relevant (implementation specific) "annotations" concepts. </br>An implementation may chose to add additional implementation-specific annotations as they see fit. | MaxProperties: </br>8 |

We can define a new gateway configuration which includes the `infrastructure` sub-section.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gw-internal-auto
  namespace: default
spec:
  gatewayClassName: cilium
  infrastructure:
    annotations:
      "service.beta.kubernetes.io/azure-load-balancer-internal": "true"
  listeners:
  - allowedRoutes:
      namespaces:
        from: Same
    name: cilium-gw-internal-auto
    port: 80
    protocol: HTTP

```

We get an associated service which inherit the `"service.beta.kubernetes.io/azure-load-balancer-internal": "true"` annotation.

```bash

df@df-2404lts:~$ k get gateway cilium-gw-internal-auto
NAME                      CLASS    ADDRESS        PROGRAMMED   AGE
cilium-gw-internal-auto   cilium   172.21.17.14   True         49s

df@df-2404lts:~$ k get service cilium-gateway-cilium-gw-internal-auto -o json | jq .metadata.annotations
{
  "service.beta.kubernetes.io/azure-load-balancer-internal": "true"
}

```

So finally, the capability to define an internal gateway is not on the gateway class but on the gateway itself.
It may feel counter-intuitive, but considering the segregation of duty define in the gateway api, it kind of make sens since we create as many gateway as we need.

Let's wrap it for today.

## 4. Summary

Today we saw in more depth 2 of the Gateway API objects: 

- The Gateway class
- The Gateway

Gateway Classes, similarly to the Ingress Classes, are used to define properties that are inherited by the gateways.

Gateways did not have equivalence in the Ingress Controller architecture. A gateway provide an entry point on the cluster and can be used for the exposition of applications.
Gateway may also be shared accross namespace for different application, but we'll see this in another article.

We will also soon have a look at the `httproutes.gateway.networking.k8s.io`, the most mature object of the Gateway API to manage application exposition.

But as I said, It wioll be for another time.

Until then ^^







