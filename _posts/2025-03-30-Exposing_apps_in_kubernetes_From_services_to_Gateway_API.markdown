---
layout: post
title:  "Exposing apps in Kubernetes: from services to Gateway API"
date:   2025-03-30 18:00:00 +0200
year: 2025
categories: AKS Kubernetes Network
---

Hi!

In this article I propose to have a look at how apps are exposed in Kubernetes.
This is a back to basics mixed with new stuffs from the Kubernetes related project Gateway API.
Hope you'll enjoy it.

The Agenda:


1. Exposing apps 101: The kubernetes service
2. Exposing apps better with the Ingress
3. What was wrong with Ingress again? And what do we do now? A sample with Cilium Gateway API


Let's get started!


## 1. Exposing apps 101: The kubernetes service

In the Kubernetes landscape, we host application within kubernetes in pods. The pods contain... container(s) and usually, are also included in other kind of controllers such as deployments, daemonsets... But that's not the topics of this article right ?

What if we want to access an app in a pod?

We could always find its IP and then try to access the app directly. After all, a simple `kubectl get pod -o wide` will give us this information.

```bash

df@df-2404lts:~$ k get pod -o wide
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE                              NOMINATED NODE   READINESS GATES
demo   1/1     Running   0          19s   100.64.6.7   aks-npuser2-14732456-vmss000008   <none>           <none>

```

Except, well, as mentionned, pods are usually grouped under a controller umbrella, and then we have more than one which we can access, and second, we usually have a network overlay which makes the pods unaccessible directly, from a network standpoint.

**Note:** *There's one option on AKS for which the pods are directly accessible from a workload connected to the Vnet. It's when the Azure CNI is configured, without overlay.*

So that's why kubernetes comes with a native object called the service. Typically, a service to expose the demo pod on port 8080 would look like that in yaml:

```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: demo
  name: demo
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    run: demo
status:
  loadBalancer: {}

```

The service is relying on the pod's label to know which pod to select and to redirect trafic to:

```bash

df@df-2404lts:~$ k get pod demo --show-labels 
NAME   READY   STATUS    RESTARTS   AGE   LABELS
demo   1/1     Running   0          44m   run=demo

```

In the case of the demo pod, it's an nginx based pod, listening on port 80, hence the target port set to `80` while we configured the service with a `8080` port.

```bash

df@df-2404lts:~$ k get service demo 
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
demo   ClusterIP   100.65.106.57   <none>        8080/TCP   6s

```

The attentive reader will notice the `<none>` value for external IP, while we do have a cluster IP address. This pod lives on an AKS cluster configured with Azure CNI Overlay, as shown below, so it is not possible to access it directly.

![illustration1](/assets/svctogapi/svctogapi001.png)

At this point we rely on the Cloud controller manager which allows us to leverage the cloud (in our case Azure) capabilities to provision a load balancer. For this we change the type of service. Instead of a `ClusterIP` type, we will specify a `LoadBalancer` type 

```bash

df@df-2404lts:~$ kubectl get service demo -o json | jq .spec.type
"ClusterIP"

df@df-2404lts:~$ kubectl edit service demo

```

```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: demo
  name: demo
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    run: demo
status:
  loadBalancer: {}

```

```bash

df@df-2404lts:~$ kubectl get service demo
NAME   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
demo   LoadBalancer   100.65.106.57   <pending>     8080:31560/TCP   20h
df@df-2404lts:~$ kubectl get service demo
NAME   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
demo   LoadBalancer   100.65.106.57   74.179.240.209   8080:31560/TCP   20h

```

In our Azure environment, if the flows are allowed on the NSG(s), we should be able to curl the service.
Note that the NSG associated with the node pool(s) will update automatically the rules so that the flow is opened.
However, if there is another NSG on the subnet (which may be something by design), then, both NSGs should be configured to allow the traffic.

![illustration2](/assets/svctogapi/svctogapi002.png)

![illustration3](/assets/svctogapi/svctogapi003.png)

```bash

df@df-2404lts:~$ curl -i -X GET http://74.179.240.209:8080
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Wed, 26 Mar 2025 07:25:21 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 11:06:32 GMT
Connection: keep-alive
ETag: "67a34638-267"
Accept-Ranges: bytes

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

It's interesting to note that the cloud provider gives us options to configure the service through specific [annotations](https://cloud-provider-azure.sigs.k8s.io/topics/loadbalancer/#loadbalancer-annotations), as in the sample below.


```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: demo
  name: demo-internal
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "sub2-vnet-sbx-spokeaks1"
    service.beta.kubernetes.io/azure-load-balancer-ipv4: 172.21.10.132
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    run: demo
status:
  loadBalancer: {}

```

Again, through the cloud controller manager, we get an Azure Load Balancer, except that this time, it's an internal one.
Also because we specified the subnet and the IP, we can check that everything is as expected.

```bash

df@df-2404lts:~$ k get service demo-internal 
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)          AGE
demo-internal   LoadBalancer   100.65.8.60   172.21.10.140   8080:30879/TCP   2m44s

```

![illustration4](/assets/svctogapi/svctogapi004.png)

![illustration5](/assets/svctogapi/svctogapi005.png)

![illustration6](/assets/svctogapi/svctogapi006.png)


Ok, that's quite nice but definitly not enough for web apps for wchich I may need to manage http path.
And that's for this kind of things that we had first the ingress. 


## 2. Exposing apps better with the Ingress

The idea of the ingress is to provide this additional awareness of the L7 LB, which does not exist in default kubernetes services.


![illustration7](/assets/svctogapi/svctogapi007.png)

It relies on an Ingress controller which needs to be installed on the cluster. We don't want to detail to much the installation of an Ingress Controller, there is already quite a lot of documentations about this, with the [nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/), and in an [Azure environment](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/load-bal-ingress-c/create-unmanaged-ingress-controller?tabs=azure-cli#create-an-ingress-controller).

To install a basic ingress controller we could use helm as below:

```bash

df@df-2404lts:~$ helm upgrade external-ingress ingress-nginx/ingress-nginx --namespace externalingress --create-namespace --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz --install
Release "external-ingress" does not exist. Installing it now.
NAME: external-ingress
LAST DEPLOYED: Wed Mar 26 11:37:24 2025
NAMESPACE: externalingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace externalingress external-ingress-ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls

```

Once the installation is complete, we can see in the specified namespace the created resources.

```bash

df@df-2404lts:~$ k get all -n externalingress 
NAME                                                           READY   STATUS    RESTARTS   AGE
pod/external-ingress-ingress-nginx-controller-9bf4566c-cfnqf   1/1     Running   0          114m

NAME                                                          TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/external-ingress-ingress-nginx-controller             LoadBalancer   100.65.73.213   57.152.85.14   80:30177/TCP,443:30403/TCP   114m
service/external-ingress-ingress-nginx-controller-admission   ClusterIP      100.65.214.0    <none>         443/TCP                      114m

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/external-ingress-ingress-nginx-controller   1/1     1            1           114m

NAME                                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/external-ingress-ingress-nginx-controller-9bf4566c   1         1         1       114m

```

We can also see that we have a default ingress class. We'll come back to this later.

```bash

df@df-2404lts:~$ k get ingressclasses.networking.k8s.io 
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       107m

```

Now let's create some apps to illustrate our purpose.

We will use nginx based deployment, with config map for a touch of customization.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: ingressapp-demo
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-main
 namespace: ingressapp-demo
data:
 index.html: |
   <html>
   <h1>Ingress Demo</h1>
   </br>
   <h2>Hi! This is the main page of the app </h2>
   <img src="https://japancraft.co.uk/media/catalog/product/cache/1/image/1800x/040ec09b1e35df139433887a97daa66f/b/a/bans63030_4.jpeg" />
   </html
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-doc
 namespace: ingressapp-demo
data:
 index.html: |
   <html>
   <h1>Ingress Demo</h1>
   </br>
   <h2>Hi! This is the doc page of the app </h2>
   <img src="https://www.previewsworld.com/SiteImage/FBCatalogImage/STL205689.jpg" />
   </html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: main
  name: main
  namespace: ingressapp-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: main
    spec:
      volumes:
      - name: nginx-index-file
        configMap:
          name: index-html-main  
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-index-file
          mountPath: /usr/share/nginx/html/
        resources: {}
status: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: docpage
  name: docpage
  namespace: ingressapp-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docpage
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: docpage
    spec:
      volumes:
      - name: nginx-index-file
        configMap:
          name: index-html-doc  
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-index-file
          mountPath: /usr/share/nginx/html/
        resources: {}
status: {}


```

Once the application is deployed, we want to expose internally with ClusterIP services:

```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: main
  name: main
  namespace: ingressapp-demo
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: main
status:
  loadBalancer: {}
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: docpage
  name: docpage
  namespace: ingressapp-demo
spec:
  ports:
  - port: 8090
    protocol: TCP
    targetPort: 80
  selector:
    app: docpage
status:
  loadBalancer: {}

```

We can verify that our services are accessible from a pod:

```bash

df@df-2404lts:~$ k get service -n ingressapp-demo 
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
docpage   ClusterIP   100.65.182.19    <none>        8090/TCP   8m37s
main      ClusterIP   100.65.133.247   <none>        8080/TCP   9m4s

df@df-2404lts:~$ k exec client -- curl http://docpage.ingressapp-demo.svc.cluster.local:8090
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   171  100   171    0     0   9436     <html>--:-- --:--:-- --:--:--     0
<h1>Ingress Demo</h1>
</br>
<h2>Hi! This is the doc page of the app </h2>
<img src="https://www.previewsworld.com/SiteImage/FBCatalogImage/STL205689.jpg" />
</html
 0 --:--:-- --:--:-- --:--:--  9500

df@df-2404lts:~/$ k exec client -- curl http://main.ingressapp-demo.svc.cluster.local:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<html>
<h1>Ingress Demo</h1>
</br>
<h2>Hi! This is the main page of the app </h2>
<img src="https://japancraft.co.uk/media/catalog/product/cache/1/image/1800x/040ec09b1e35df139433887a97daa66f/b/a/bans63030_4.jpeg" />
</html
100   224  100   224    0     0  15361      0 --:--:-- --:--:-- --:--:-- 16000

```

And now we want to create an ingress, so that the main page is accessible on the `/main` path, and the doc page is accessible on the `/doc` path

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo
  namespace: ingressapp-demo
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    #nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /main
        pathType: Prefix
        backend:
          service:
            name: main
            port: 
              number: 8080
      - path: /doc
        pathType: Prefix
        backend:
          service:
            name: docpage
            port: 
              number: 8090
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main
            port:
              number: 8080

```

![illustration8](/assets/svctogapi/svctogapi008.png)

![illustration9](/assets/svctogapi/svctogapi009.png)

At this point we are now able to expose application publicly.

If we want to expose privately, since the service behind the ingress controller is always the same, we need another ingress controller instance, with an ingress class different from the one already present.

We can install another instance as shown below, by adding the setting the argument `controller.ingressClassResource.name="internal-nginx"` and `controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx-internal"`.
Also, to leverage the capabilities of the cloud controller manager, we can insert the service annotations used earlier.

- `controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true`
- `controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet"="sub2-vnet-sbx-spokeaks1"`

```bash

df@df-2404lts:~$ helm upgrade internal-ingress ingress-nginx/ingress-nginx --namespace internalingress --create-namespace --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet"="sub2-vnet-sbx-spokeaks1" --set controller.ingressClassResource.name="internal-nginx" --set controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx-internal" --install
Release "internal-ingress" does not exist. Installing it now.
NAME: internal-ingress
LAST DEPLOYED: Wed Mar 26 13:26:02 2025
NAMESPACE: internalingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace internalingress internal-ingress-ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: internal-nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls


```

We now have 2 ingress controller classes, and 2 ingress controller instances in different namespace

```bash

df@df-2404lts:~$ k get ingressclasses.networking.k8s.io 
NAME             CONTROLLER                      PARAMETERS   AGE
internal-nginx   k8s.io/ingress-nginx-internal   <none>       3h29m
nginx            k8s.io/ingress-nginx            <none>       5h17m

df@df-2404lts:~$ helm ls -A
NAME                            	NAMESPACE      	REVISION	UPDATED                                	STATUS  	CHART                                                                    	APP VERSION       	           
external-ingress                	externalingress	1       	2025-03-26 11:37:24.18934227 +0100 CET 	deployed	ingress-nginx-4.12.1                                                     	1.12.1     
internal-ingress                	internalingress	1       	2025-03-26 13:26:02.685498353 +0100 CET	deployed	ingress-nginx-4.12.1                                                     	1.12.1   

df@df-2404lts:~$ k get service -A | grep ingress-nginx
externalingress     external-ingress-ingress-nginx-controller             LoadBalancer   100.65.73.213    57.152.85.14     80:30177/TCP,443:30403/TCP   5h22m
externalingress     external-ingress-ingress-nginx-controller-admission   ClusterIP      100.65.214.0     <none>           443/TCP                      5h22m
internalingress     internal-ingress-ingress-nginx-controller             LoadBalancer   100.65.109.85    172.21.10.132    80:32713/TCP,443:32593/TCP   3h34m
internalingress     internal-ingress-ingress-nginx-controller-admission   ClusterIP      100.65.24.220    <none>           443/TCP                      3h34m
df@df-2404lts:~$ 

```

We can create another ingress with the internal-nginx class.

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo-internal
  namespace: ingressapp-demo
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: internal-nginx
  rules:
  - http:
      paths:
      - path: /main
        pathType: Prefix
        backend:
          service:
            name: main
            port: 
              number: 8080
      - path: /doc
        pathType: Prefix
        backend:
          service:
            name: docpage
            port: 
              number: 8090
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main
            port:
              number: 8080

```

And from a network that can access the internel load balancer, check the access to the application

```bash

david [ ~ ]$ curl http://172.21.10.132/main
<html>
<h1>Ingress Demo</h1>
</br>
<h2>Hi! This is the main page of the app </h2>
<img src="https://japancraft.co.uk/media/catalog/product/cache/1/image/1800x/040ec09b1e35df139433887a97daa66f/b/a/bans63030_4.jpeg" />
</html

david [ ~ ]$ curl http://172.21.10.132/doc
<html>
<h1>Ingress Demo</h1>
</br>
<h2>Hi! This is the doc page of the app </h2>
<img src="https://www.previewsworld.com/SiteImage/FBCatalogImage/STL205689.jpg" />
</html
david [ ~ ]$ 

```

And that was the ingress. But nowadays, we talk more and more about the Gateway API. Let's have a look.



## 3. What was wrong with Ingress again? And what do we do now? A sample with Cilium Gateway API

So we've seen the basic kubernetes service, and also how to manage L7 path with ingress controller.
We've also seen that to we get one instance of a controller for all the ingress object afterward, except if we install additional ingress controller and define some ingress class.
That's one problem of the ingress controller, it is not really well designed for multi-tenant cluster. If one team need to manage the ingress controller, it should have access to the related object. That's fine, but what happened when another team also need access?
Then either they share access, or they both have their own instance. It can work but it's not necessarily something that anone can do.

Also, because ingress has been around for a time now, it is higly likely that each providers bring their own customization, often with CRDs or specifics annotations. So we kind of lost the standard along the way.

Those are topics that the Gateway API is aiming to address. We discussed that a bit in a [previous article](/_posts/2024-07-29-Not_getting_lost_in_the_service_mesh_and_gamma_k8s_stuffs.markdown) about service mesh, where we took from the Kubernetes documentation a nice schema explaining how the role separation is addressed in the Gateway API architecture.

![illustration10](/assets/servicemesh/smesh006.png)

while the Ingress controller required a specific installation, and were more or less provider specific, the gateway API is defined by CRDs that define the differents opbjects related to the Gateway API.

The really intersting part here is that we have something that is designed to be used by different teams.

- Infrastructure operators will define the Gateway Class which define what features the Gateway can have
- Cluster operators, or sometimes even some developers, will create gateway based on the gateway class. Some customizations are possibel, also based on CRDs (more on that just after)
- Dev teams just need to know how the apps should be exposed, and refers the available gateways(s) for the jobs.

We mention CRDs, to define the Gateway API capabilities. Those are defined in the [Gateway API documentation](https://kubernetes.io/docs/concepts/services-networking/gateway/) and are available on [Github](https://github.com/kubernetes-sigs/gateway-api/tree/main/config/crd). That's where we'll find:

- The gateway class
- The gateway

and then the different object to manage apps exposure:

- HTTPRoute
- grpcRoutes
- TLSRoute

So there are stil some requirements to use a Gateway API, and the first one is to have those CRDs installed on the target cluster. This can be done quite easily:

```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml


```

To try a bit of the Gateway API, we can use Cilium. In our case, a byocni on an AKS cluster, with specification to install the Gateway API. It will implies that the kube-proxy replacement is enabled.

```go

resource "azapi_update_resource" "kube_proxy_disabled" {
  for_each    = { for k, v in var.AksConfig : k => v if v.CustomizeKubeproxyConfig }
  resource_id = module.AKS[each.key].KubeId
  type        = "Microsoft.ContainerService/managedClusters@2024-06-02-preview"
  body = {
    properties = {
      networkProfile = {
        kubeProxyConfig = {
          enabled = false
        }
      }
    }
  }


}


```

```bash

helm upgrade cilium ./cilium \
    --namespace kube-system \
    --reuse-values \
    --set kubeProxyReplacement=true \
    --set gatewayAPI.enabled=true

```

If everything is installed correctly, cilium cli should be green:

```bash

df@df-2404lts:~$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy       Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium             Running: 3
                       cilium-envoy       Running: 3
                       cilium-operator    Running: 2
                       hubble-relay       Running: 1
                       hubble-ui          Running: 1
Cluster Pods:          51/51 managed by Cilium
Helm chart version:    1.17.2
Image versions         cilium             quay.io/cilium/cilium:v1.17.2@sha256:3c4c9932b5d8368619cb922a497ff2ebc8def5f41c18e410bcc84025fcd385b1: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.31.5-1741765102-efed3defcc70ab5b263a0fc44c93d316b846a211@sha256:377c78c13d2731f3720f931721ee309159e782d882251709cb0fac3b42c03f4b: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.17.2@sha256:81f2d7198366e8dec2903a3a8361e4c68d47d19c68a0d42f0b7b6e3f0523f249: 2
                       hubble-relay       quay.io/cilium/hubble-relay:v1.17.2@sha256:42a8db5c256c516cacb5b8937c321b2373ad7a6b0a1e5a5120d5028433d586cc: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.2@sha256:a034b7e98e6ea796ed26df8f4e71f83fc16465a19d166eff67a03b822c0bfa15: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.2@sha256:9e37c1296b802830834cc87342a9182ccbb71ffebb711971e849221bd9d59392: 1

df@df-2404lts:~$ cilium config view | grep "enable-gateway-api"
enable-gateway-api                                true
enable-gateway-api-alpn                           false
enable-gateway-api-app-protocol                   false
enable-gateway-api-proxy-protocol                 false
enable-gateway-api-secrets-sync                   true    

```

And we should have a gatewayclass available.

```bash

df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io 
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       2d8h

```

If the Gateway is in a pending status, it may help to restart cilium related pods, as mentioned on the Cilium doc. I'll admit that I did miss this one, and solve it with an AKS restart &#128517;.

```bash

kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium

```

If it's working, we can move along with the creation of a gateway.


```yaml

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gw
  namespace: gatewayapi-test
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw
    allowedRoutes:
      namespaces:
        from: Same

```

Then with the following sample app,

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: gatewayapi-test
spec: {}
status: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-configmap1
 namespace: gatewayapi-test
data:
 index.html: |
   <html>
   <h1>Welcome to Demo App 1</h1>
   </br>
   <h2>This is a demo to illustrate Gateway API </h2>
   <img src="https://riseofgunpla.com/wp-content/uploads/2024/12/In-ERA-Lizard-9.jpg" />
   </html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demoapp1
  name: demoapp1
  namespace: gatewayapi-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp1
  template:
    metadata:
      labels:
        app: demoapp1
    spec:
      volumes:
      - name: nginx-index-file
        configMap:
          name: index-html-configmap1
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-index-file
          mountPath: /usr/share/nginx/html
---
apiVersion: v1
kind: Service
metadata:
  name: demosvcapp1
  namespace: gatewayapi-test
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
    app: demoapp1       
---

```

And an HTTPRoute as below

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
  namespace: gatewayapi-test
spec:
  parentRefs:
  - name: demo-gw
  hostnames:
  - "gapitest.app.teknews.cloud"
  rules:
  - backendRefs:
    - name: demosvcapp1
      port: 8080
      kind: Service

```

We have the simplest app with a Gateway API. 
Again, if the Gateway class is working, we should have a Gateway, and an associated service.

```bash

df@df-2404lts:~$ k get gateway -n gatewayapi-test 
NAME      CLASS    ADDRESS          PROGRAMMED   AGE
demo-gw   cilium   48.216.184.196   True         2d7h

df@df-2404lts:~$ k get service -n gatewayapi-test | grep cilium
cilium-gateway-demo-gw   LoadBalancer   100.67.26.63     48.216.184.196   80:30994/TCP   2d7h

```
And the application should be available on `http://gapi.app.teknews.cloud`, under the condition that there is a DNS record for this hostname.

```bash

df@df-2404lts:~$ nslookup gapitest.app.teknews.cloud
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   gapitest.app.teknews.cloud
Address: 48.216.184.196

df@df-2404lts:~$ curl http://gapitest.app.teknews.cloud
<html>
<h1>Welcome to Demo App 1</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://riseofgunpla.com/wp-content/uploads/2024/12/In-ERA-Lizard-9.jpg" />
</html

```

![illustration11](/assets/svctogapi/svctogapi010.png)

Before going to a (temporary) conclusion, 2 things I'd like to discuss today.

First, there is a native option to manage wieght for an application. This is quite well demonstrated on the [Cilium samples](https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/splitting/). With an HTTPRoute as below, it's possible to chose the wieght of 1 service in coomparison to another

```yaml

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gw
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw-echo
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-route-1
spec:
  parentRefs:
  - name: cilium-gw
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /echo
    backendRefs:
    - kind: Service
      name: echo-1
      port: 8080
      weight: 50
    - kind: Service
      name: echo-2
      port: 8090
      weight: 50

```

Second, we have a specific CRD to customize the Gateway. In Cilium case, this is the `CiliumGatewayClassConfig` CRD.

```yaml

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.16.5
  name: ciliumgatewayclassconfigs.cilium.io
spec:
  group: cilium.io
  names:
    categories:
    - cilium
    kind: CiliumGatewayClassConfig
    listKind: CiliumGatewayClassConfigList
    plural: ciliumgatewayclassconfigs
    shortNames:
    - cgcc
    singular: ciliumgatewayclassconfig
  scope: Namespaced
  versions:
  - additionalPrinterColumns:
    - jsonPath: .status.conditions[?(@.type=="Accepted")].status
      name: Accepted
      type: string
    - jsonPath: .metadata.creationTimestamp
      name: Age
      type: date
    - jsonPath: .spec.description
      name: Description
      priority: 1
      type: string
    name: v2alpha1
    schema:
      openAPIV3Schema:
        description: |-
          CiliumGatewayClassConfig is a Kubernetes third-party resource which
          is used to configure Gateways owned by GatewayClass.
        properties:
          apiVersion:
            description: |-
              APIVersion defines the versioned schema of this representation of an object.
              Servers should convert recognized schemas to the latest internal value, and
              may reject unrecognized values.
              More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
            type: string
          kind:
            description: |-
              Kind is a string value representing the REST resource this object represents.
              Servers may infer this from the endpoint the client submits requests to.
              Cannot be updated.
              In CamelCase.
              More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
            type: string
          metadata:
            type: object
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
            type: object
          status:
            description: Status is the status of the policy.
            properties:
              conditions:
                description: Current service state
                items:
                  description: Condition contains details for one aspect of the current
                    state of this API Resource.
                  properties:
                    lastTransitionTime:
                      description: |-
                        lastTransitionTime is the last time the condition transitioned from one status to another.
                        This should be when the underlying condition changed.  If that is not known, then using the time when the API field changed is acceptable.
                      format: date-time
                      type: string
                    message:
                      description: |-
                        message is a human readable message indicating details about the transition.
                        This may be an empty string.
                      maxLength: 32768
                      type: string
                    observedGeneration:
                      description: |-
                        observedGeneration represents the .metadata.generation that the condition was set based upon.
                        For instance, if .metadata.generation is currently 12, but the .status.conditions[x].observedGeneration is 9, the condition is out of date
                        with respect to the current state of the instance.
                      format: int64
                      minimum: 0
                      type: integer
                    reason:
                      description: |-
                        reason contains a programmatic identifier indicating the reason for the condition's last transition.
                        Producers of specific condition types may define expected values and meanings for this field,
                        and whether the values are considered a guaranteed API.
                        The value should be a CamelCase string.
                        This field may not be empty.
                      maxLength: 1024
                      minLength: 1
                      pattern: ^[A-Za-z]([A-Za-z0-9_,:]*[A-Za-z0-9_])?$
                      type: string
                    status:
                      description: status of the condition, one of True, False, Unknown.
                      enum:
                      - "True"
                      - "False"
                      - Unknown
                      type: string
                    type:
                      description: type of condition in CamelCase or in foo.example.com/CamelCase.
                      maxLength: 316
                      pattern: ^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])$
                      type: string
                  required:
                  - lastTransitionTime
                  - message
                  - reason
                  - status
                  - type
                  type: object
                type: array
                x-kubernetes-list-map-keys:
                - type
                x-kubernetes-list-type: map
            type: object
        required:
        - metadata
        type: object
    served: true
    storage: true
    subresources:
      status: {}


```

hypothetically, we should be able to create a gateway with a node port or a clusterip service kind. As of now, I don't know how to make this work &#128531;.

Also, while we could pass the annotations for the service to the Ingress Controller installatin, it seems that we need to rely on an admission controller here to provide something similar.

So let's stop here, and summarize, because it was a long run.

## 4. Summary

Exposing apps in Kubernetes is something that is quite easy to do, for a basic exposition.
We saw that a kubernetes service can do the job, and depending on the cloud provider, it's possible to manage the kind of exposition.

We also had a look at how we can go further with an Ingress Controller, which is the current standard for exposing application. While it may allow a lot of configuration, the different providers made the standardiszation a mess. Also, it's not really easy ton integrate a service mesh with an ingress (but do we want service mesh?).

All those limitations are probably some reason why the Gateway API came to be, with a more standardized definition (through CRDs), more role oriented capabilities and a better service mesh integration (with the GAMMA initiative). However, the different provider are not all completely mature, and passing annotations to the underlying services of the gateway API is not always avaialble.

From my point of view, there are still many different things to try out before the Gateway API is clear for me, so there are chances that I'll write more about it in a near future.

Until then ^^


