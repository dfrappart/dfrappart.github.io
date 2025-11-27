---
layout: post
title:  "Adding some visualization to Falco with Falco Sidekick"
date:   2025-11-23 18:00:00 +0200
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
- Streaming Falco alerts to Microsoft Teams

Ok, let's go!

## 1. About Falco Sidekick

We've seen that Falco is a powerful engine of Security alerting that help better Kubernetes security.
However, It's not very user friendly and it requires some sysadmin type action.

To solve this lack, the Falco Sidekick has been developped.
Falco Sidekick provides an endpoint to forward Falco outputs.
It can be used as a centralized endpoint for multiple Falco instances.
And it can also be used to forward event to different systems.

Among those system is the Falco Sidekick ui, which provides a basic dashboard for Falco.
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

And see that there is an avaialble endpoint through a curl command.

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

![ilustration8](/assets/falco/falcosidekickui007.png)

![ilustration9](/assets/falco/falcosidekickui008.png)

![ilustration10](/assets/falco/falcosidekickui009.png)

And that's all for this part.

What's next?

## 4. Falcosidekick connectivity

In this section, we will have a look at how we can connect more with/to Falco.

First, 