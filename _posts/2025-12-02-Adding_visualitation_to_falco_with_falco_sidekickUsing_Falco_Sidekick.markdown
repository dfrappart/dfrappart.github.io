---
layout: post
title:  "Adding some visualization to Falco with Falco Sidekick"
date:   2025-12-2 18:00:00 +0200
year: 2025
categories: Kubernetes AKS Security
---

Hi!

following our serie on Falco, this time, we'll have a look at how we can get a better visualization of Falco outputs.

Up until now, we've checked how Falco works, and how we can install, and customize Falco on either self-managed cluster, or Cloud-managed kubernetes.
However, we've only gone to the point where we know how to check output in text mode.

What if we want to better Falco output visibility?

That's the topic of today's post with Falco sidekick.

Our agenda:

- About Falco Sidekick
- Configuring on AKS
- Configuring on self-managed cluster
- Falcosidekick connectivity

Ok, let's go!

## 1. About Falco Sidekick

We've seen that Falco is a powerful engine of Security alerting that help to better Kubernetes security.
However, It's not very user friendly and it requires some sysadmin type action.

To solve this lack, [Falco Sidekick](https://github.com/falcosecurity/falcosidekick) has been developped.
Falco Sidekick provides an endpoint to forward Falco outputs.
It can be used as a centralized endpoint for multiple Falco instances.
And it can also be used to forward event to different systems.

The first system that can be leverage for visualization is Falcosidekick and its UI, which provides a basic dashboard for Falco.

There is a quite long list of messenging solution such as Microsoft Teams, that we'll try out later in this article, but also metrics solutions, such as Open Telemetry, or Prometheus.

![illustration1](/assets/falco/falcosidekick_forwarding.png).

Ok let's dive in!

## 2. Configuring on AKS

We'll start this time by a configuration on AKS.

That's because adding Falco sidekick can be done on the Helm chart installation.

Remember that we install Falco with the following command last time.

```bash

df@df-2404lts:~$ helm upgrade falco --set falcosidekick.enabled=true falcosecurity/falco --namespace falco --create-namespace --install
Release "falco" does not exist. Installing it now.
NAME: falco
LAST DEPLOYED: Mon Nov 24 11:53:30 2025
NAMESPACE: falco
STATUS: deployed
REVISION: 1
NOTES:
Falco agents are spinning up on each node in your cluster. After a few
seconds, they are going to start monitoring your containers looking for
security issues.

```

Notice that we have specified the `--set falcosidekick.enabled=true`.

If we check, we do have a `service` called `falco-falcosidekick` in our Falco namespace. We can port-forward the `service` to our local host

```bash
df@df-2404lts:~/Documents/dfrappart.github.io$ k get service -n falco falco-falcosidekick
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
falco-falcosidekick   ClusterIP   100.65.237.201   <none>        2801/TCP,2810/TCP   7h20m
df@df-2404lts:~/Documents/dfrappart.github.io$ k port-forward -n falco services/falco-falcosidekick 2801:2801
Forwarding from 127.0.0.1:2801 -> 2801
Forwarding from [::1]:2801 -> 2801

```

And see that there is an available endpoint through a curl command.

```bash

df@df-2404lts:~$ curl -i -X GET localhost:2801/healthz
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 24 Nov 2025 18:27:16 GMT
Content-Length: 16

{"status": "ok"}

```

Ok, an answer through curl, but still not very user friendly right?

Checking the Falco [helm chart doc](https://github.com/falcosecurity/charts/tree/master/charts/falco#deploy-falcosidekick-with-falco), we find that `falcosidekick.webui.enabled=true` is another usefull parameter which can be used to enabled a web ui.

Updating our configuration, we get an additional `service` and we can access the web ui.

```bash
df@df-2404lts:~$ helm upgrade falco --set falcosidekick.enabled=true falcosecurity/falco --set falcosidekick.webui.enabled=true --namespace falco --create-namespace --install -f ./rules.yaml 
df@df-2404lts:~$ k get service -n falco
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
falco-falcosidekick            ClusterIP   100.65.237.201   <none>        2801/TCP,2810/TCP   8h
falco-falcosidekick-ui         ClusterIP   100.65.187.226   <none>        2802/TCP            4h30m
falco-falcosidekick-ui-redis   ClusterIP   100.65.108.231   <none>        6379/TCP            4h30m

```

After trigerring some rules, we can see some stuff on the web ui.

![illustration2](/assets/falco/falcosidekickui001.png)

![illustration3](/assets/falco/falcosidekickui002.png)

![illustration4](/assets/falco/falcosidekickui003.png)

![illustration5](/assets/falco/falcosidekickui004.png)

![illustration6](/assets/falco/falcosidekickui005.png)

![illustration7](/assets/falco/falcosidekickui006.png)

It is not the best UI you'll ever get but it is an improvement from the text output only we had until now.

Now that we've seen the simple installation, we'll switch to the self-managed kubernetes and have a look at the falco.yaml details.

## 3. Configuring on a self-managed cluster with falco.yaml

We just saw that Falco Sidekick can be installed, embeded in the Falco Chart.
This time, we cannot use this method because we have a kubeadm cluster with Falco installed as a service.

```bash

vagrant@k8scilium1:~$ systemctl status falco
● falco-modern-bpf.service - Falco: Container Native Runtime Security with modern ebpf
     Loaded: loaded (/usr/lib/systemd/system/falco-modern-bpf.service; enabled; preset: enabled)
     Active: active (running) since Wed 2025-11-26 06:05:10 UTC; 18min ago
       Docs: https://falco.org/docs/
   Main PID: 716 (falco)
      Tasks: 14 (limit: 3472)
     Memory: 94.9M (peak: 104.8M)
        CPU: 17.053s
     CGroup: /system.slice/falco-modern-bpf.service
             └─716 /usr/bin/falco -o engine.kind=modern_ebpf

Nov 26 06:06:30 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:30.848867920: Warning 🚀 Container spawned! container=testpod2 image=docker.io/library/nginx:latest user=<>
Nov 26 06:06:32 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:32.505585744: Warning 🚀 Container spawned! container=morefalco image=docker.io/library/nginx:latest user=>
Nov 26 06:06:34 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:34.697346479: Warning 🚀 Container spawned! container=nginx image=docker.io/library/nginx:latest user=<NA>>
Nov 26 06:06:36 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:36.943953359: Warning 🚀 Container spawned! container=backend2 image=docker.io/library/nginx:latest user=<>
Nov 26 06:06:38 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:38.793102002: Warning 🚀 Container spawned! container=backend image=docker.io/library/nginx:latest user=<N>
Nov 26 06:06:40 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:40.780689576: Warning 🚀 Container spawned! container=newpod image=docker.io/library/nginx:latest user=<NA>
Nov 26 06:06:42 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:42.663425117: Warning 🚀 Container spawned! container=testpod image=docker.io/library/nginx:latest user=<N>
Nov 26 06:06:44 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:44.811838058: Warning 🚀 Container spawned! container=newhellofalco image=docker.io/library/nginx:latest u>
Nov 26 06:06:48 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:06:48.108455271: Warning 🚀 Container spawned! container=testpod2 image=docker.io/library/nginx:latest user=<>
Nov 26 06:07:00 k8scilium1 falco[716]: {"hostname":"k8scilium1","output":"06:07:00.329538478: Warning 🚀 Container spawned! container=testpodagin image=docker.io/library/nginx:latest use>
lines 1-21/21 (END)

```

There are reference to use Falco Sidekick in this case. Without too much surprise, it points to the `falco.yaml` file, specifically the output part.

Remember, we saw in a previous article that we can configure Falco to output its data to an http endpoint. We saw in the previous section that Falco Sidekick provide an http endpoint. So we have to configure the `falco.yaml` with the proper output.

Additionaly, the json_output should be configured. The parameters to be modified are nicely summarized in the [documentation](https://github.com/falcosecurity/falcosidekick#with-helm) and cpy pasted below

```yaml

json_output: true
json_include_output_property: true
http_output:
  enabled: true
  url: "http://localhost:2801/"

```

That's for configuring Falco to stream events to Falco Sidekick, whoch should be installed. With Helm (without Falco), it look like this.

```bash

vagrant@k8scilium1:~$ helm repo add falcosecurity https://falcosecurity.github.io/charts
vagrant@k8scilium1:~$ helm repo update

vagrant@k8scilium1:~$ helm install falcosidekick --set config.debug=true falcosecurity/falcosidekick

```

Once intalled, we have the following object available.

```bash

vagrant@k8scilium1:~$ k get deployments.apps 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
falcosidekick      2/2     2            2           44d
falcosidekick-ui   1/1     1            1           44d
vagrant@k8scilium1:~$ k get service
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
falcosidekick            ClusterIP   100.65.98.171   <none>        2801/TCP,2810/TCP   44d
falcosidekick-ui         ClusterIP   100.65.158.52   <none>        2802/TCP            44d
falcosidekick-ui-redis   ClusterIP   100.65.3.18     <none>        6379/TCP            44d
kubernetes               ClusterIP   100.65.0.1      <none>        443/TCP             83d
vagrant@k8scilium1:~$ 

```

At this point we have both `falcosidekick` `service`, and `falcosidekick-ui` `service`, but both are configured as `ClusterIP` so it would not be possible to reach any of those directly from the kubernetes node.

In a cloud-managed kubernetes, we could rely on the cloud controller manager and we would get a LoadBalancer `service` type available from outside the kubernetes overlay. But we cannot do this here, so we'll rely on an HTTPRoute for the falcosidekick endpoint.

If you started to look at the Gateway API, you probably know that it still relies on a kubernetes `service` under the hood. Because we're still not on the cloud, we will need to use `NodePort` `service` type. So here is our GatewayClass configuration.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nodeport-gateway-class
spec:
  controllerName: io.cilium/gateway-controller
  description: A Cilium GatewayClass for local k8s testing with NodePort svc.
  parametersRef:
    group: cilium.io
    kind: CiliumGatewayClassConfig
    name: gateway-class-config
    namespace: default
---
apiVersion: cilium.io/v2alpha1
kind: CiliumGatewayClassConfig
metadata:
  name: gateway-class-config
  namespace: default
spec:
  service:
    type: NodePort   

```

We should have a new GatewayClass available after that.

We can now move on and create a Gateway for Falcosidekick endpoint, along with an HTTProute.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: sharedgw
spec: {}
status: {}
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: falco-gw-tls
  namespace: sharedgw
spec:
  gatewayClassName: nodeport-gateway-class
  listeners:
  - protocol: HTTPS
    port: 443
    name: cilium-gateway
    allowedRoutes:
      namespaces:
        from: All
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: k8scilium1tls
        namespace: sharedtls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-falco
spec:
  parentRefs:
  - name: falco-gw-tls
    namespace: sharedgw
  hostnames:
  - "k8scilium1"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: falcosidekick
      port: 2801
      kind: Service

```
and again for Falco sidekick UI.

```yaml

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: falco-ui-gw-tls
  namespace: sharedgw
spec:
  gatewayClassName: nodeport-gateway-class
  listeners:
  - protocol: HTTPS
    port: 443
    name: cilium-gateway
    allowedRoutes:
      namespaces:
        from: All
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: k8scilium1tls
        namespace: sharedtls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-falco-ui
spec:
  parentRefs:
  - name: falco-ui-gw-tls
    namespace: sharedgw
  hostnames:
  - "k8scilium1"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: falcosidekick-ui
      port: 2802
      kind: Service

```

Notice the use of a `ReferenceGrant` to use the `secret` from another `namespace`.


```yaml

---
apiVersion: v1
kind: Namespace
metadata:
  name: sharedtls
spec: {}
status: {}
---
apiVersion: v1
data:
  tls.crt: LS0t========truncated=========LQo=
  tls.key: LS0t========truncated=========LQo=
kind: Secret
metadata:
  name: k8scilium1tls
  namespace: sharedtls
type: kubernetes.io/tls
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-ciliumgateway-to-ref-secrets
  namespace: sharedtls
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: sharedgw
  to:
  - group: ""
    kind: Secret  

```

So we should have our 2 HTTPRoutes with the corresponding Gateway.

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ k get gatewayclasses.gateway.networking.k8s.io 
NAME                     CONTROLLER                     ACCEPTED   AGE
cilium                   io.cilium/gateway-controller   True       83d
nodeport-gateway-class   io.cilium/gateway-controller   True       43d
df@df-2404lts:~/Documents/dfrappart.github.io$ k get gateway -n sharedgw 
NAME              CLASS                    ADDRESS   PROGRAMMED   AGE
falco-gw-tls      nodeport-gateway-class             True         42d
falco-ui-gw-tls   nodeport-gateway-class             True         42d
df@df-2404lts:~/Documents/dfrappart.github.io$ k get service -n sharedgw 
NAME                             TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
cilium-gateway-falco-gw-tls      NodePort   100.65.165.108   <none>        443:32310/TCP   42d
cilium-gateway-falco-ui-gw-tls   NodePort   100.65.165.153   <none>        443:30666/TCP   42d
df@df-2404lts:~/Documents/dfrappart.github.io$ k get httproutes.gateway.networking.k8s.io 
NAME             HOSTNAMES        AGE
https-falco      ["k8scilium1"]   42d
https-falco-ui   ["k8scilium1"]   42d

```

Our `falco.yaml` is configured with the HTTPRoute as its endpoint, and we also a parameter to accept the self signed certifcate used.

```yaml

vagrant@k8scilium1:~$ cat /etc/falco/falco.yaml |grep "http_output:" -A8
http_output:
  # -- Enable sending alerts to an HTTP endpoint or webhook.
  enabled: true
  # -- URL of the remote server to send the alerts to.
  url: "https://k8scilium1:32310"
  # -- User agent string to be used in the HTTP request.
  user_agent: "falcosecurity/falco"
  # -- Tell Falco to not verify the remote server.
  insecure: true

```

As for the AKS cluster, once we triggered a few rules, we can see the ui updating.

![illustration8](/assets/falco/falcosidekickui007.png)

![illustration9](/assets/falco/falcosidekickui008.png)

![illustration10](/assets/falco/falcosidekickui009.png)

And that's all for this part.

What's next?

## 4. Falcosidekick connectivity

In this section, we will have a look at how we can connect more with/to Falco.

### 4.1. Using Falcosidekick as a hub for other Falco instances

Because Falcosidekick can be configured as a hub for other Falco instances, we want to test this.

So in this section, we'll use 2 AKS clusters. We'll install Falco on both clusters but only one with Falcosidekick.
Then from our previous experimentations, we'll configure Falcosidekick so that it can accept incoming traffic from the other AKS cluster.

First, we have our 2 clusters.

```bash

df@df-2404lts:~$ az aks list -o table
Name        Location       ResourceGroup     KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
----------  -------------  ----------------  -------------------  --------------------------  -------------------  ----------------------------------------------
aks-falco   francecentral  rsg-cluster-aks   1.33.5               1.33.5                      Creating             aksfalco-gyzguww0.hcp.francecentral.azmk8s.io
aks-falco2  francecentral  rsg-cluster-aks2  1.33.5               1.33.5                      Creating             aksfalco2-w38mogb7.hcp.francecentral.azmk8s.io

```

After getting the credentials from az cli, we should have the following with the `kubectl config` command.

```bash

df@df-2404lts:~$ k config get-contexts 
CURRENT   NAME                      CLUSTER      AUTHINFO                                  NAMESPACE
*         aks-falco                 aks-falco    clusterUser_rsg-cluster-aks_aks-falco     
          aks-falco2                aks-falco2   clusterUser_rsg-cluster-aks2_aks-falco2   
          kubernetes-admin@cilium   kubernetes   kubernetes-admin                          
df@df-2404lts:~$ k config current-context 
aks-falco

```

We can now install Falcosidekick.

```bash

df@df-2404lts:~$ helm  upgrade falcosidekick --set config.debug=true falcosecurity/falcosidekick \
    --namespace falcosidekick \
    --create-namespace \
    --set webui.enabled=true \
    --install

```

We get 3 services with this installation, we want to make the falcosidekick available from outside of the cluster.

```bash

df@df-2404lts:~$ k get service -n falcosidekick 
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
falcosidekick            ClusterIP   100.65.188.4    <none>        2801/TCP,2810/TCP   100m
falcosidekick-ui         ClusterIP   100.65.124.45   <none>        2802/TCP            99m
falcosidekick-ui-redis   ClusterIP   100.65.42.47    <none>        6379/TCP            99m

```

To prepare for the next step, which is connecting the Falco instance from the other cluster to this instance of Falcosidekick, we need to expose the `service` `falcosidekick-ui`. We'll do this with an HTTPRoute

The configuration being similar to what we did in the self-hosted cluster section, we'll just show our configuration for the HTTPRoute.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: falcosidekick-httproute-tls
  namespace: falcosidekick
spec:
  parentRefs:
  - name: shared-gw-tls
    namespace: sharedgw
  hostnames:
  - "falcosidekick.app.teknews.cloud"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: falcosidekick
      port: 2801

```

Notice the `hostnames` section. We created a record on an Azure DNS zone that point to the IP of our Gateway.

![illustration11](/assets/falco/falcosidekickui010.png)

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ k get httproute -n falcosidekick 
NAME                          HOSTNAMES                             AGE
falcosidekick-httproute-tls   ["falcosidekick.app.teknews.cloud"]   60m

```

If the HTTPRoute is in an accepted state, checking access to falcosidekick with curl should gives us something like this.


```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ curl -k -i -X GET https://falcosidekick.app.teknews.cloud/healthz
HTTP/1.1 200 OK
content-type: application/json
date: Mon, 01 Dec 2025 22:01:52 GMT
content-length: 16
x-envoy-upstream-service-time: 1
server: envoy

{"status": "ok"}

```

Ok let's install Falco on this cluster, the one with the Falcosidekick instance.
Because this is sthe same cluster, we can here rely on the ClusterIP Ip address.


```bash

df@df-2404lts:~$ k get service -n falcosidekick falcosidekick
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
falcosidekick            ClusterIP   100.65.188.4    <none>        2801/TCP,2810/TCP   100m

```

We can install Falco with the following configuration.

```bash

helm upgrade falco falcosecurity/falco \
    --create-namespace \
    --namespace falco \
    --set falco.http_output.enabled=true \
    --set falco.http_output.url="http://100.65.188.4:2801" \
    --set falco.http_output.insecure=true \
    --set falco.json_output=true \
    --set json_include_output_property=true \
    --install

```

Switching on the other cluster, we use the nearly same configuration, we just switch the internal ClusterIP address to the HTTPRoute.
Because in this case the certiuficate is a self-signed one, we also need to specify the `falco.http_output.insecure=true` parameter.

```bash

helm upgrade falco falcosecurity/falco \
    --create-namespace \
    --namespace falco \
    --set falco.http_output.enabled=true \
    --set falco.http_output.url="https://falcosidekick.app.teknews.cloud" \
    --set falco.http_output.insecure=true \
    --set falco.json_output=true \
    --set json_include_output_property=true \
    --install

```

We can now trigger some rules and we should get alerts from both clusters.

![illustration12](/assets/falco/falcosidekickui011.png)

![illustration13](/assets/falco/falcosidekickui012.png)

![illustration14](/assets/falco/falcosidekickui013.png)

Ok, that's nice.
Moving on with connection Falco to an external chat system

### 4.2. Forwarding Falco event to Teams

In this section we'll use the previously installed Falcosidekick instance and configure it to forward Falco events to a teams channel.

It can be done with plenty of Chat tools, but it's convenient for me to try this out with Teams, so here we are.

The workflow is actually pretty simple.
We have a few parameters to do both on Falco side, and on the target Teams channel.

First on the Teams side. Accessing a Teams channel, we need to add an incoming webhook.

![illustration15](/assets/falco/falcototeams001.png)

![illustration16](/assets/falco/falcototeams002.png)

![illustration17](/assets/falco/falcototeams003.png)

![illustration18](/assets/falco/falcototeams004.png)

At the end of the configuration, we got the webhook url. 

From the Falcosidekick documentation for Teams, we can see that we have the additional parameter to set: `teams.webhookurl`
We can also find accross 

With those informations we can upgrade the Falcosidekick installation.

```bash

df@df-2404lts:~$ helm  upgrade falcosidekick --set config.debug=true falcosecurity/falcosidekick \
    --namespace falcosidekick \
    --create-namespace \
    --set webui.enabled=true \
    --set config.teams.webhookurl="<teams_channel_webhook>" \
    --set config.teams.outputformat=all \
    --install

```

If the channel is configured correctly, messages should pop like this.

![illustration19](/assets/falco/falcototeams005.png)

![illustration20](/assets/falco/falcototeams006.png)

![illustration21](/assets/falco/falcototeams007.png)

And that's it.
A little review before going &#128526;

## 5. Summary

This time, we looked how to connect Falco to other systems.
We found that we can achieve this with Falcosidekick which act as a proxy between Falco instances (plural intended) and those other systems.
The first visualization is brought by Falcosidekick own UI.
Apart from that, there are many systems that can connect to Falco. We shaw that the connection to Microsoft Teams is quite simple.

Another steps to extend Falco capabilities would be the plugins, but that will be for another time.

Until then^^

