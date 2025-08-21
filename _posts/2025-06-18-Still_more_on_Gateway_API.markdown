---
layout: post
title:  "Still More on Gateway API. The HTTP Route"
date:   2025-06-18 18:00:00 +0200
year: 2025
categories: Aks Security Network
---

Hello!

Last time we look at the Gateway class and the Gateway, which are the begining of the Gateway API usage.
But we stopped before looking really at how to expose apps.
In this article, we carry on from this point and we look specifically at the HTTP Route and reflect on how we manage the exposure of apps.

The Agenda:


1. The HTTP Route - basics
2. Adding TLS
3. Conclusion


As a reminder, we went through the creation of gateway class, and some gateway.
Doing so, we identified that while there is a way to pass annotations to the service created by the gateway through the `spec.infrastructure.annotations` property.
While there is also a dedicated crd ciliumgatewayclassconfigs.cilium.io to customize the gateway class, finally we pushed the customization on the gateway side rather than on the gateway class side.

At this point, a gateway manifest looks like this:

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gw
  namespace: demo
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: gundam-gw
    allowedRoutes:
      namespaces:
        from: Same 

```

And we leverage the Azure capabilities to use an internal load balancer with a manifest looking like this:

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gw-internal
  namespace: demo
spec:
  gatewayClassName: custom-gateway-class
  infrastructure:
    annotations:
      "service.beta.kubernetes.io/azure-load-balancer-internal": "true"
  listeners:
  - protocol: HTTP
    port: 80
    name: demo-gw-internal
    allowedRoutes:
      namespaces:
        from: Same 

```

Now let's move on and look at how we can manage applications exposition.

## 1. The HTTP Route

### 1.1. basics

Exposing an application is done with the http route. Details on this api object are available on the [gateway api documentation](https://gateway-api.sigs.k8s.io/reference/spec/#httproute).

Let's say that we have an app based on a deployment, and a service.

```yaml


apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-configmapnginx
 namespace: gundam
data:
 index.html: |
   <html>
   <h1>Welcome to Demo App Akatsuki</h1>
   </br>
   <h2>This is a demo to illustrate Gateway API </h2>
   <img src="https://sakura-pink.jp/img-items/1-gundam-2024-6-4-1-4.jpg" />
   </html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demoappakatsuki
  name: demoappakatsuki
  namespace: gundam
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoappakatsuki
  template:
    metadata:
      labels:
        app: demoappakatsuki
    spec:
      volumes:
      - name: nginx-index-file
        configMap:
          name: index-html-configmapnginx
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-index-file
          mountPath: /usr/share/nginx/html
        - name: nginx-index-file
          mountPath: /usr/share/nginx/html/test          
        resources:
          requests:
            cpu: "50m"
            memory: "50Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"       
---
apiVersion: v1
kind: Service
metadata:
  name: akatsukisvc
  namespace: gundam
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
  selector:
    app: demoappakatsuki    

```

With the default ClusterIP service configuration, the application is only accessible internally, through any pod actually.

```bash

df@df-2404lts:~$ k get service -n gundam akatsukisvc 
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
akatsukisvc   ClusterIP   100.67.251.70   <none>        8080/TCP   3d6h

df@df-2404lts:~$ k exec api-contact -- curl http://akatsukisvc.gundam.svc.cluster.local:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   182  100   182    0     0  16602      0 --:--:-- --:--:-- --:--:-- 18200
<html>
<h1>Welcome to Demo App Akatsuki</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://sakura-pink.jp/img-items/1-gundam-2024-6-4-1-4.jpg" />
</html

```

Let's add a simple httproute.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gundam-httproute
  namespace: gundam
spec:
  parentRefs:
  - name: gundam-gw
  hostnames:
  - "gundam.app.teknews.cloud"
  rules:
  - backendRefs:
    - name: akatsukisvc
      port: 8080
      kind: Service

```

In the hostnames section, we specified an host which should be DNS resolvable and point to the Public IP of the gateway (In my case, this is through an Azure DNS zone, but it's quite simple so I won't detailled that here &#128541;).

```bash

df@df-2404lts:~$ nslookup gundam.app.teknews.cloud
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   gundam.app.teknews.cloud
Address: 134.33.245.251

df@df-2404lts:~$ k get gateway -n gundam 
NAME                 CLASS                  ADDRESS          PROGRAMMED   AGE
gundam-gw            cilium                 134.33.245.251   True         3d7h

```

And we get a nice app like this

![illustration1](/assets/gapi/basichttproute.png)

Now let's add some additional services to our app and see how it can be managed.

### 1.2. Path management basics in http route

Before going fully on this topics, let's step back a little.

With an Nginx Ingress controller, if we want to expose, let's say, 3 differents services on specific path, we create an ingress object as below.

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress-external
  namespace: demo
  labels:
    app: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: external-ingress-nginx
  rules:
  - host: demoingress.app.teknews.cloud
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
      - path: /two
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-two
            port:
              number: 80
      - path: /three
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-three
            port:
              number: 80

```

Assuming the underlying kubernetes services exist (and the associated apps), we would get something like this.

```bash

df@df-2404lts:~$ curl http://demoingress.app.teknews.cloud
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <link rel="stylesheet" type="text/css" href="/static/default.css">
    <title>Welcome to Azure Kubernetes Service (AKS) &#39;App One&#39; 1</title>

    <script language="JavaScript">
        function send(form){
        }
    </script>

</head>
<body>
    <div id="container">
        <form id="form" name="form" action="/"" method="post"><center>
        <div id="logo">Welcome to Azure Kubernetes Service (AKS) &#39;App One&#39; 1</div>
        <div id="space"></div>
        <img src="/static/acs.png" als="acs logo">
        <div id="form">      
        </div>
    </div>     
</body>
</html>df@df-2404lts:~$ curl http://demoingress.app.teknews.cloud/two
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <link rel="stylesheet" type="text/css" href="/static/default.css">
    <title>Welcome to Azure Kubernetes Service (AKS) &#39;App two&#39; 2</title>

    <script language="JavaScript">
        function send(form){
        }
    </script>

</head>
<body>
    <div id="container">
        <form id="form" name="form" action="/"" method="post"><center>
        <div id="logo">Welcome to Azure Kubernetes Service (AKS) &#39;App two&#39; 2</div>
        <div id="space"></div>
        <img src="/static/acs.png" als="acs logo">
        <div id="form">      
        </div>
    </div>     
</body>
</html>df@df-2404lts:~$ curl http://demoingress.app.teknews.cloud/three
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <link rel="stylesheet" type="text/css" href="/static/default.css">
    <title>Welcome to Azure Kubernetes Service (AKS) &#39;App three&#39; 3</title>

    <script language="JavaScript">
        function send(form){
        }
    </script>

</head>
<body>
    <div id="container">
        <form id="form" name="form" action="/"" method="post"><center>
        <div id="logo">Welcome to Azure Kubernetes Service (AKS) &#39;App three&#39; 3</div>
        <div id="space"></div>
        <img src="/static/acs.png" als="acs logo">
        <div id="form">      
        </div>
    </div>     
</body>
</html>df@df-2404lts:~$ 
</html>

```

But the really interesting part here, is the annotation `nginx.ingress.kubernetes.io/rewrite-target: /` which, as it implies, rewrite the paths on the ingress to the`/` path of our apps.

Ok time to try this with the httproute.

This time, we have a bunch of aditional deployments and services.

```bash

df@df-2404lts:~$ k get deployments.apps -n gundam
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
barbatos           3/3     3            3           3d7h
client1            1/1     1            1           3d7h
demoappakatsuki    3/3     3            3           3d7h
demoapptallgeese   3/3     3            3           3d7h
eva02              3/3     3            3           3d7h
exia               3/3     3            3           3d7h

df@df-2404lts:~$ k get service -n gundam | grep svc
akatsukisvc                         ClusterIP      100.67.251.70    <none>           8080/TCP       3d7h
barbatossvc                         ClusterIP      100.67.137.175   <none>           8090/TCP       3d7h
evasvc                              ClusterIP      100.67.248.215   <none>           8091/TCP       3d7h
exiasvc                             ClusterIP      100.67.202.174   <none>           8092/TCP       3d7h
tallgeesesvc                        ClusterIP      100.67.225.39    <none>           8093/TCP       3d7h

```

We want to expose, let's say, the barbatos apps to our http route, so we add an additional rule in the `spec.rules` section : 

```yaml

  - backendRefs:
    - name: barbatossvc
      port: 8090
      kind: Service
    matches:
    - path:
        type: PathPrefix
        value: /barbatos

```

Does it works ?

```bash

df@df-2404lts:~$ curl http://gundam.app.teknews.cloud/barbatos
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.27.5</center>
</body>
</html>

```


Well, no because we need to specify the rewrite rule. It is clearly related to the way the http route forward traffic. The nginx based error is clear when we look at the pod's logs (we scale down the deploiement to 1 replicas, to be sure that we cvheck the correect pod logs). It gets a request for an unexisting path.

```bash

df@df-2404lts:~$ k logs -n gundam deployments/barbatos 
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2025/06/13 08:32:34 [notice] 1#1: using the "epoll" event method
2025/06/13 08:32:34 [notice] 1#1: nginx/1.27.5
2025/06/13 08:32:34 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14) 
2025/06/13 08:32:34 [notice] 1#1: OS: Linux 5.15.0-1084-azure
2025/06/13 08:32:34 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2025/06/13 08:32:34 [notice] 1#1: start worker processes
2025/06/13 08:32:34 [notice] 1#1: start worker process 29
2025/06/13 08:32:34 [notice] 1#1: start worker process 30
100.66.0.23 - - [13/Jun/2025:13:07:32 +0000] "GET / HTTP/1.1" 200 285 "-" "curl/8.5.0" "81.220.209.253"
100.66.1.56 - - [13/Jun/2025:13:10:52 +0000] "GET / HTTP/1.1" 200 289 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:139.0) Gecko/20100101 Firefox/139.0" "81.220.209.253"
2025/06/16 16:26:12 [error] 29#29: *3 open() "/usr/share/nginx/html/barbatos" failed (2: No such file or directory), client: 100.66.1.56, server: localhost, request: "GET /barbatos HTTP/1.1", host: "gundam.app.teknews.cloud"
100.66.1.56 - - [16/Jun/2025:16:26:12 +0000] "GET /barbatos HTTP/1.1" 404 153 "-" "curl/8.5.0" "81.220.209.253"

```

Let's look at the httproute object to find how it can be done. 

In the `spec.rules` section, we already added a matches sections, which contain a path. From the [documentation](https://gateway-api.sigs.k8s.io/reference/spec/#httproutefiltertype), we can see that a `filters` section can be added. 

Specifically, we can use the `URLRewrite` type 

| Field | Description | Default | Validation |
|-|-|-|-|
| `type` | Type defines the type of path modifier. | Enum: [ReplaceFullPath ReplacePrefixMatch] |
| `replaceFullPath` | Specifies the value with which to replace the full path of a request during a rewrite or redirect. || MaxLength: 1024 |
| `replacePrefixMatch` | Specifies the value with which to replace the prefix match of a request during a rewrite or redirect. |  | MaxLength: 1024 |

And then specify the appropriate properties to modify the path:

- a `type`, which accept ReplaceFullPath and ReplacePrefixMatch
- the `replacePrefixMatch` which in this case will be `/` to replace the prefix on the http route with the `/` path on the container.

It would look like this.

```yaml

filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /

```

And thus gives us a working httproute for the url `http://gundam.app.teknwes.cloud/barbatos`

![illustration2](/assets/gapi/pathrewrite.png)


The attentive reader (or experienced Ingress user &#129325;) probably noticed that the rewrite configuration is in this case managed for each backend.
This was not necessarily the case with an Ingress controller which often relied on annotations, such as `nginx.ingress.kubernetes.io/rewrite-target: /` that we used in our previous example.

While this worked, and was quite simple, we have here more granularity in the same http route. All in all, the rewrite may seem less simple, but the granularity is quite nice.

Before moving to the TLS part, let's have a look at one last item, the weight management.

### 1.3. Managing weight in the httproute

We talk very rapidly about this in the [Exposing apps article](/_posts/2025-03-30-Exposing_apps_in_kubernetes_From_services_to_Gateway_API.markdown). 
The http route has a native capability of weight management. 

Again, from the documentation, we can find the spec.rules.backendRefs.weight:

| Field | Description | Default | Validation |
|-|-|-|-|
| `weight` | Weight specifies the proportion of requests forwarded to the referenced backend. This is computed as weight/(sum of all weights in this BackendRefs list). | 1 | Max 1e+06 </br> Min 0 |

Transposed in an http route, it looks like this:

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gundamsplit
  namespace: gundam
spec:
  parentRefs:
  - name: gundam-gw
  hostnames:
  - "gundamsplit.app.teknews.cloud"
  rules:
  - backendRefs:
    - kind: Service
      name: akatsukisvc
      port: 8080
      weight: 50
    - kind: Service
      name: exiasvc
      port: 8092
      weight: 50   

```

We can check the load balancing with a bash command as below

```bash

df@df-2404lts:~$ while true; do curl -s -k "http://gundamsplit.app.teknews.cloud" >> curlresponses.txt ;done
^C

df@df-2404lts:~$ cat curlresponses.txt | grep -i exia | wc -l
31

df@df-2404lts:~$ cat curlresponses.txt | grep -i akatsuki | wc -l
26

```

In this case we have roughly the 50/50 balancing.

If we modify the httproute weight like this:

```yaml

rules:
  - backendRefs:
    - kind: Service
      name: akatsukisvc
      port: 8080
      weight: 90
    - kind: Service
      name: exiasvc
      port: 8092
      weight: 10  

```


We get this time the following result, validating the capability of the `weight` parameter.

```bash

df@df-2404lts:~$ while true; do curl -s -k "http://gundamsplit.app.teknews.cloud" >> curlresponses.txt ;done
^C
df@df-2404lts:~$ cat curlresponses.txt | grep -i exia | wc -l
2
df@df-2404lts:~$ cat curlresponses.txt | grep -i akatsuki | wc -l
16

```

Ok that was fun now let's have a look at some encryption ^^

## 2. Adding TLS

### 2.1. TLS consiuderations with Gateway API

For this section, no surprise, we will again rely on the gateway api [documentation](https://gateway-api.sigs.k8s.io/guides/tls/) &#128518;.

We need to consider the traffic from the gateway point of view. Taking this into account, we have 2 parts:

- The downstream connection, happening betwxeen the client and the gateway itself.
- The upstream connection, happening between the gateway and the backend service (most of the time)

![illustration3](/assets/gapi/gatewayupstreamdownstream.png)

Considering this, the gateway api objects available to manage TLS connectivity may answer to different scenario. With the http route that we will work with today, we are limited to a TLS termination on the gateway, as opposite to a TLS passthrough. We'll note that it does not mean the traffic has to go on unencrypted after the http route.

The table below  summarize the different available scenarios depending on the object used.

| Listener protocol | TLS mode | Route Type supported |
|-|-|-|
| TLS | Passthrough | TLS Route |
| TLS | Terminate | TCPRoute |
| HTTPS | Terminate | HTTPRoute |
| gRPC | Terminate | GRPCRoute |

As mentioned, in the next part, we'll focus on the TLS scenario with http route.

### 2.2. Configuring TLS

To configure TLS, we have to act first at the gateway level. Which make sense, since we mention a 2 way connection, the upstream and the downstream.

searching in the documentation, we can find information for the `spec.listeners.tls` section:

| Field	| Description	| Default	| Validation |
|-|-|-|-|
| `mode` | Mode defines the TLS behavior for the TLS session initiated by the client. There are two possible modes:</br>- Terminate: The TLS session between the downstream client and the Gateway is terminated at the Gateway. This mode requires certificates to be specified in some way, such as populating the certificateRefs field. </br>- Passthrough: The TLS session is NOT terminated by the Gateway. This implies that the Gateway can't decipher the TLS stream except for the ClientHello message of the TLS protocol. The certificateRefs field is ignored in this mode.|Terminate	| Enum: [Terminate Passthrough] |
| `certificateRefs` array	| CertificateRefs contains a series of references to Kubernetes objects that contains TLS certificates and private keys. </br>These certificates are used to establish a TLS handshake for requests that match the hostname of the associated listener. </br>A single CertificateRef to a Kubernetes Secret has "Core" support. </br>Implementations MAY choose to support attaching multiple certificates to a Listener, but this behavior is implementation-specific. </br>References to a resource in different namespace are invalid UNLESS there is a ReferenceGrant in the target namespace that allows the certificate to be attached.</br> If a ReferenceGrant does not allow this reference, the "ResolvedRefs" condition MUST be set to False for this listener with the "RefNotPermitted" reason. </br>This field is required to have at least one element when the mode is set to "Terminate" (default) and is optional otherwise. </br>CertificateRefs can reference to standard Kubernetes resources, i.e. Secret, or implementation-specific custom resources. |	|	MaxItems: 64 |

And the child object `spec.listerners.tls.certificateRefs`

| Field	| Description	| Default	| Validation |
|-|-|-|-|
| `group`	| Group is the group of the referent. For example, "gateway.networking.k8s.io". When unspecified or empty string, core API group is inferred. |  | MaxLength: 253 </br>Pattern: ^$\|^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$ |
| `kind` 	| Kind is kind of the referent. For example "Secret".	| Secret	| MaxLength: 63 </br> MinLength: 1 </br>Pattern: ^[a-zA-Z]([-a-zA-Z0-9]*[a-zA-Z0-9])?$
| `name` 	| Name is the name of the referent.	| | | MaxLength: 253 </br> MinLength: 1
| `namespace` 	| Namespace is the namespace of the referenced object. When unspecified, the localnamespace is inferred. </br> Note that when a namespace different than the local namespace is specified, a ReferenceGrant object is required in the referent namespace to allow that namespace's owner to accept the reference. | |	MaxLength: 63</br> MinLength: 1 </br> Pattern: ^[a-z0-9]([-a-z0-9]*[a-z0-9])?$ |

Which allow us to write a gateway configuration as below (with a secret, because at this point we still need a secret &#128541;).

```yaml

apiVersion: v1
data:
  tls.crt: LS0tLS1--hidden
  tls.key: LS0tLS1--hidden
  kind: Secret
metadata:
  name: apptekewscloud
  namespace: gundam
type: kubernetes.io/tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gundam-tls-gw
  namespace: gundam
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
    allowedRoutes:
      namespaces:
        from: Same    

```

By the way, the secret can be created with a `kubectl` command as below : 


```bash

df@df-2404lts:~$ kubectl create secret tls apptekewscloud --key <path_to_key> --cert <path_to_pem> 


```

And about how to manage the certificate creation?

Because I'm definitely not that good in cert management, I'll let this topic outside of thescope of this article.
Between openssl command and [mkcert-like](https://github.com/FiloSottile/mkcert) stuff, it's quite easy to generate cert for tests. It's definitely another thing for prod ready environment, but again, out of scope for today.

Once our objects are created, we should have our new gateway and httproute available.

```bash

df@df-2404lts:~$ k get gateway -n gundam gundam-tls-gw -o wide
NAME            CLASS    ADDRESS        PROGRAMMED   AGE
gundam-tls-gw   cilium   20.242.243.1   True         46h
        
df@df-2404lts:~$ k get httproutes.gateway.networking.k8s.io -n gundam gundamtls 
NAME        HOSTNAMES                         AGE
gundamtls   ["gundamtls.app.teknews.cloud"]   23h

df@df-2404lts:~$ k get httproutes.gateway.networking.k8s.io -n gundam gundamtls -o wide
NAME        HOSTNAMES                         AGE
gundamtls   ["gundamtls.app.teknews.cloud"]   23h

```

We can then try a curl on the different path available

```bash

df@df-2404lts:~$ k describe httproutes.gateway.networking.k8s.io -n gundam gundamtls 
Name:         gundamtls
Namespace:    gundam
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2025-06-18T16:03:59Z
  Generation:          2
  Resource Version:    18652261
  UID:                 443bcf25-c91d-4992-afda-c29cd2c7647c
Spec:
  Hostnames:
    gundamtls.app.teknews.cloud
  Parent Refs:
    Group:  gateway.networking.k8s.io
    Kind:   Gateway
    Name:   gundam-tls-gw
  Rules:
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    akatsukisvc
      Port:    8080
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    barbatossvc
      Port:    8090
      Weight:  1
    Filters:
      Type:  URLRewrite
      URL Rewrite:
        Path:
          Replace Prefix Match:  /
          Type:                  ReplacePrefixMatch
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /barbatos
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    evasvc
      Port:    8091
      Weight:  1
    Filters:
      Type:  URLRewrite
      URL Rewrite:
        Path:
          Replace Prefix Match:  /
          Type:                  ReplacePrefixMatch
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /eva02
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    exiasvc
      Port:    8092
      Weight:  1
    Filters:
      Type:  URLRewrite
      URL Rewrite:
        Path:
          Replace Prefix Match:  /
          Type:                  ReplacePrefixMatch
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /exia
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    tallgeesesvc
      Port:    8093
      Weight:  1
    Filters:
      Type:  URLRewrite
      URL Rewrite:
        Path:
          Replace Prefix Match:  /
          Type:                  ReplacePrefixMatch
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /tallgeese
Status:
  Parents:
    Conditions:
      Last Transition Time:  2025-06-18T16:06:18Z
      Message:               Accepted HTTPRoute
      Observed Generation:   2
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2025-06-18T16:06:18Z
      Message:               Service reference is valid
      Observed Generation:   2
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         io.cilium/gateway-controller
    Parent Ref:
      Group:  gateway.networking.k8s.io
      Kind:   Gateway
      Name:   gundam-tls-gw
Events:       <none>

df@df-2404lts:~$ curl https://gundamtls.app.teknews.cloud
<html>
<h1>Welcome to Demo App Akatsuki</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://sakura-pink.jp/img-items/1-gundam-2024-6-4-1-4.jpg" />
</html

df@df-2404lts:~$ curl https://gundamtls.app.teknews.cloud/exia
<html>
<h1>Welcome to Exia App</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://imgs.search.brave.com/F1Miw5SAIQpxwxWpbkMxtFzznghhOZrQ3Z7zqpCBrfI/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9tZWRp/YS5jZG53cy5jb20v/X2kvNTg5MzQvMzcw/NS8xNzk3LzcvYmFu/ZGFpLTIwNDU0MzEx/MjUyMjI3Ni5qcGVn" />
</html

df@df-2404lts:~$ curl https://gundamtls.app.teknews.cloud/eva02
<html>
<h1>Welcome to EVA 02 App 3</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://riseofgunpla.com/wp-content/uploads/2020/07/75424765_719470355234279_422176357275926528_n-1000x1000.jpg" />
</html 

```

Ok, time to wrap up.


## 4. Summary

So this time we look really at the way apps are exposed with gateway and http route.
We still have things to see, with notably the other objects mentioned that provided a way of exposure.
Also, we have seen in the API definition that there are some way to act accross namespace, and thus manage shared gateway.

I'll try to work on this last topic in another session, and maybe also some of the other objects for apps exposure.

Until next time.







