---
layout: post
title:  "Azure Native Gateway API with Application Gateway for container"
date:   2025-09-25 18:00:00 +0200
year: 2025
categories: Kubernetes AKS Network
---

Hi!

After a little (and definitly unfinished) incursion in Kubernetes (or should I say Linux) native security features, we're going back to Gateway API stuffs, and specifically Azure related Gateway API with the Application Gateway for Containers.

There's a lot of reading about this already, but as I said in the past, this blog also aimed o help me summarize stuff for myself so... Application Gateway for containers, here we come.

Our Agenda will be:


1. Components of the Application Gateway for containers
2. Prerequisites and deployment on an AKS cluster with Azure CNI overlay powered by Cilium
3. Managing Gateway and HTTP routes with AGC
3. Conclusion

Let's get started!


## 1. Components of the Application Gateway for Containers

Application Gateway for Containers, or AGC for short, is the successor to Azure's own implementation of the Ingress Controller, the formerly (and also a little bit unfamous &#129323;) Application Gateway Ingress Controller (AGIC).

Under the hood, some similarities. AGC is an L7 load balancing solution aimed to provides such capabilities for K8S hosted workloads.
As the name implies, because the (regionalized) L7 LB in Azure is Application Gateway, it comes without surprise that we're still relying on this platform service for AGC.

Except that, as opposite to AGIC, the object is much more managed, and as a corollary much more abstracted than with its predecessor.
Indeed, creating an AGC will show in Azure a corresponding resource (more detail on this later), classified in the Application Gateway, but with a brand new icon.

![illustration1](/assets/agc/agc001.png)

The fun's only get started, because the az cli command to list application gateway WILL NOT list the agc as an application gateway (which make sense IMHO)

```bash

df@df-2404lts:~$ az network application-gateway list
[]


```

there's in fact a dedicated resource in the az cli for agc, or specifically one of its Azure components.

```bash

df@df-2404lts:~$ az network alb list | jq .[]
{
  "associations": [
    {
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab/associations/AlbSubnetAssociation",
      "resourceGroup": "rsg-spoke-agw"
    }
  ],
  "configurationEndpoints": [
    "ae0b5824c789458b89f76f38055e34e2.alb.azure.com"
  ],
  "frontends": [
    {
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab/frontends/albfe-aks-lab",
      "resourceGroup": "rsg-spoke-agw"
    }
  ],
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab",
  "location": "francecentral",
  "name": "alb-aks-lab",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-spoke-agw",
  "securityPolicies": [],
  "systemData": {
    "createdAt": "2025-09-16T08:34:06.3189045Z",
    "createdBy": "00000000-0000-0000-0000-000000000000",
    "createdByType": "Application",
    "lastModifiedAt": "2025-09-16T08:34:06.3189045Z",
    "lastModifiedBy": "00000000-0000-0000-0000-000000000000",
    "lastModifiedByType": "Application"
  },
  "type": "Microsoft.ServiceNetworking/TrafficControllers"
}


```

So, kind of an Application Gateway but not really. 

Ok, let's have a look at the components, as decribed in the [documentation](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components).

AGC is organized with a hierearchy in mind, and also with a duality of control planes.

About the control planes, we have resources in Azure, hence our first control plane being Azure, and additional resources in the Kubernetes cluster for management with the kubernetes API, and thus our second control plane being K8S.

On the Azure side, we get first the parent resource, the said Application Gateway for Containers. Looking into this resource, we can find 3 child resources: 

- [Frontend](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components#application-gateway-for-containers-frontends)
- [Network association](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components#application-gateway-for-containers-associations)
- [Security policy](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components#application-gateway-for-containers-associations)

The frontend resource is what is available from a user standpoint. Thus it comes with an fqdn which is supposed to be associated to a CNAME record in a custom DNS. there can be more than one frontend for one AGC.
The network association connect the front end to an Azure subnet, which should be accessible from the AKS cluster. There is a limit of 1 association per AGC.
Last, the security policy allows to map a WAF policy to the Application Gateway for Controller instance.

On the Kubernetes side, we have the [ALB controller](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components#application-gateway-for-containers-alb-controller), which, as its name implies, controls the Application Gateway for Containers resources. It comes under the form of pods, managed in a deployment, and deployed currently through an Helm Chart. There is no, at this date, AKS add-on to simplify/automate all this deployment, as is the case with its predecessor the Application Gateway Ingress Controller. And we will see that it makes our life a bit harder, depending on how you happen to deploy your AGC.

![illustration2](/assets/agc/agccomponents.png)

The ALB controller may manage everything, or not, depending on the deployment model chosen. There are 2 models available:

- **The Application Gateway for Containers - managed by ALB controller**, which means that we deploy the ALB controller in a Kubernetes cluster, and it will manage all the Azure plane resources that we mentionned earlier.
- **The Application Gateway for Containers - Bring Your Own Deployment**, for which the Azure resources are deployed beforehand, and the ALB controller takes controls after its deployment.

So that's about all for architecture concepts. Let's move on to prerequisites and a sandbox deployment.


## 2. Prerequisites and deployment on an AKS cluster

### 2.1. Prerequisites

In terms of prerequisites, we have a bunch of things to think about.

First, it may be obvious, but we need the following providers to be registered on the subscription: 

- `Microsoft.ContainerService`
- `Microsoft.Network`
- `Microsoft.NetworkFunction`
- `Microsoft.ServiceNetworking`

```bash

df@df-2404lts:~$ az provider show -n Microsoft.ContainerService | jq .registrationState
"Registered"
df@df-2404lts:~$ az provider show -n Microsoft.Network | jq .registrationState
"Registered"
df@df-2404lts:~$ az provider show -n Microsoft.NetworkFunction | jq .registrationState
"Registered"
df@df-2404lts:~$ az provider show -n Microsoft.ServiceNetworking | jq .registrationState
"Registered"

```

To interact with the Application Gateway For containers through az cli, we would also need to add the alb extension through the `az extension add` command.

So that was the easy part.

Now, about the AKS cluster, Azure CNI or Azure CNI OVerlay is supported. So no BYOCNI in this case, but that's all right.
We also need to have the OIDC provider enabled, because the ALB controller works with Azure Workload Identity.

And that means that we need an Azure managed Identity with federated credentials to make the controller works.
Depending on the deployment model, and also the underlying network configuration, this user assigned identity may need access on different scopes.

For a fully managed deployment, the ALB controller creates and manages the ressources in the Azure plane, and by default, those resources (well, only the alb in fact, because the others resources are child resources, remember? &#129299;) are added in the AKS managed resource group.
So it make sense that in this case, we need to scope authorization on this resource group.
To interact with the AGC Azure resources, the identity needs a specific role called `AppGw for Containers Configuration Manager`. This role includes specific authorizations in the data plane.

```json

{
    "id": "/providers/Microsoft.Authorization/roleDefinitions/fbc52c3f-28ad-4303-a892-8a056630b8f1",
    "properties": {
        "roleName": "AppGw for Containers Configuration Manager",
        "description": "Allows access and configuration updates to Application Gateway for Containers resource.",
        "assignableScopes": [
            "/"
        ],
        "permissions": [
            {
                "actions": [
                    "Microsoft.ServiceNetworking/trafficControllers/read",
                    "Microsoft.ServiceNetworking/trafficControllers/write",
                    "Microsoft.ServiceNetworking/trafficControllers/delete",
                    "Microsoft.ServiceNetworking/trafficControllers/frontends/read",
                    "Microsoft.ServiceNetworking/trafficControllers/frontends/write",
                    "Microsoft.ServiceNetworking/trafficControllers/frontends/delete",
                    "Microsoft.ServiceNetworking/trafficControllers/associations/read",
                    "Microsoft.ServiceNetworking/trafficControllers/associations/write",
                    "Microsoft.ServiceNetworking/trafficControllers/associations/delete",
                    "Microsoft.ServiceNetworking/trafficControllers/*/read",
                    "Microsoft.ServiceNetworking/trafficControllers/*/write",
                    "Microsoft.ServiceNetworking/trafficControllers/*/delete",
                    "Microsoft.Resources/subscriptions/resourcegroups/deployments/read",
                    "Microsoft.Resources/subscriptions/resourcegroups/deployments/write",
                    "Microsoft.Resources/subscriptions/resourcegroups/deployments/operations/read",
                    "Microsoft.Resources/subscriptions/resourcegroups/deployments/operationstatuses/read"
                ],
                "notActions": [],
                "dataActions": [
                    "Microsoft.ServiceNetworking/trafficControllers/serviceRoutingConfigurations/read",
                    "Microsoft.ServiceNetworking/trafficControllers/serviceRoutingConfigurations/write",
                    "Microsoft.ServiceNetworking/trafficControllers/serviceRoutingConfigurations/delete"
                ],
                "notDataActions": []
            }
        ]
    }
}

```

For a Bring Your Own Deployment, the same authorizations are required, but chances are that the Azure resources do not live in the AKS managed RG, so the target scope should be identified clearly before hand. Also, because the managed identity still needs to be able to at least read what happens to the AKS Azure resources, a `Reader` role is still required on the AKS managed RG.

Now, another very important point is the underlying network. 
As discussed, the association resource point to an Azure subnet, which means that the said subnet should be manageable from the identity. Thus a `Network Contributor` role may be required (or a more granular custom role if desired). And we did not discussed this before, but the subnet is delegated to `Microsoft.ServiceNetworking/trafficControllers`.
Last about the subnet, a /24 is the minimal size acceptable.

There are others specificities but we meet them along the way. Let's get to the deployment.

### 2.2. Deployment of Azure resources

We will deploy AGC in an AKS Cluster with Azure CNI Overlay. To keep things simple, we'll deploy everything in a unique subnet.

![illustration3](/assets/agc/agcdeployment.png)

Deploying AGC as a bring your own deployment requires to create the Azure plane resources. The parent resource is an Azure Application Load Balancer.

```go

resource "azurerm_application_load_balancer" "Alb" {
  name                = "alb-aks-lab"
  resource_group_name = azurerm_resource_group.RGVnet["agw"].name
  location            = azurerm_resource_group.RGVnet["agw"].location
}

```

And the following resources are:

- A Frontend

```go

resource "azurerm_application_load_balancer_frontend" "AlbFe" {
  name                         = "albfe-aks-lab"
  application_load_balancer_id = azurerm_application_load_balancer.Alb.id

}


```

- And a subnet association

```go

resource "azurerm_application_load_balancer_subnet_association" "AlbSubnetAssociation" {
  name                         = "AlbSubnetAssociation"
  application_load_balancer_id = azurerm_application_load_balancer.Alb.id
  subnet_id                    = "${module.vnet["aks"].VNetFullOutput.id}/subnets/AlbSubnetbyo"

}

```

The association points to a subnet created before hand, with delegation configured to traffic controllers.

```json

df@df-2404lts:~$ az network vnet subnet show -n AlbSubnetbyo --vnet-name vnet-sbx-aks1 -g rsg-spoke-aks
{
  "addressPrefix": "172.16.13.0/24",
  "defaultOutboundAccess": true,
  "delegations": [
    {
      "actions": [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ],
      "etag": "W/\"98902553-60ab-4fe9-9fbb-cc8d09e88575\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetbyo/delegations/AlbDelegation",
      "name": "AlbDelegation",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg-spoke-aks",
      "serviceName": "Microsoft.ServiceNetworking/trafficControllers",
      "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
    }
  ],
  "etag": "W/\"98902553-60ab-4fe9-9fbb-cc8d09e88575\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetbyo",
  "name": "AlbSubnetbyo",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-spoke-aks",
  "serviceEndpoints": [],
  "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

Last but not least, we must prepare the managed identity and its federated credentials

```json

df@df-2404lts:~$ az identity list |jq .[0]
{
  "clientId": "00000000-0000-0000-0000-000000000000",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-spoke-agw/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azure-alb-identity",
  "location": "francecentral",
  "name": "azure-alb-identity",
  "principalId": "00000000-0000-0000-0000-000000000000",
  "resourceGroup": "rsg-spoke-agw",
  "systemData": null,
  "tags": {},
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}

df@df-2404lts:~$ az identity federated-credential list --identity-name azure-alb-identity -g rsg-spoke-agw |jq .
[
  {
    "audiences": [
      "api://AzureADTokenExchange"
    ],
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-spoke-agw/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azure-alb-identity/federatedIdentityCredentials/azure-alb-identity",
    "issuer": "https://francecentral.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/",
    "name": "azure-alb-identity",
    "resourceGroup": "rsg-spoke-agw",
    "subject": "system:serviceaccount:azure-alb-system:alb-controller-sa",
    "type": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
  }
]


```

![illustration4](/assets/agc/agcdeployment.png)

Last, something that should not be missed, the federated credential name must be `azure-alb-identity`

In case of the managed deployment, we only need the subnet, and the proper configuration on the identity. The application load balancer is created from the Kubernetes plane. We'll see this in the following section.

![illustration5](/assets/agc/albrbac.png)

### 2.3. Deployment of the ALB controller

Once the Azure resources are available with the AKS cluster, we can take care of the ALB controller.

As discussed earlier, it is currently installed through an helm chart, with information available on the [Azure documentation](), but also on the [artifacthub.io].
We need to uri of the chart, and a few parameters, specifically one being the reference to the managed identity created before, since ALB controller uses workload identity to interact with.

```bash

#!/bin/bash

echo "pulling the alb-controller helm chart from the microsoft container registry"

helm pull oci://mcr.microsoft.com/application-lb/charts/alb-controller --untar --untardir ./localhelmcharts --version 1.7.9

echo "removing the values.schema.json file which is currently causing problems with helm install"

rm ./localhelmcharts/alb-controller/values.schema.json

echo "Installing chart from local path"

helm upgrade alb-controller ./localhelmcharts/alb-controller --version 1.7.9 --namespace "azure-alb-system" --create-namespace --set albController.namespace="azure-alb-system" --set albController.podIdentity.clientId="<client_id_of_the_managed_identity>" --install

```

Notice that we download the chart and install from a local path. We'll discuss the why of this, among other things, later.
We should get something similar if the installation is sucessful.

```bash

df@df-2404lts:~/Documents$ helm upgrade alb-controller ./localhelmcharts/alb-controller --version 1.7.9 --namespace "azure-alb-system" --create-namespace --set albController.namespace="azure-alb-system" --set albController.podIdentity.clientId="00000000-0000-0000-0000-000000000000" --install
Release "alb-controller" does not exist. Installing it now.
NAME: alb-controller
LAST DEPLOYED: Fri Sep 26 12:07:10 2025
NAMESPACE: azure-alb-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Congratulations! The ALB Controller has been installed in your Kubernetes cluster!


```

And find a deployment, along with a service account inside the `azure-alb-system`

```bash

f@df-2404lts:~$ k get deployments.apps -n azure-alb-system alb-controller 
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
alb-controller   2/2     2            2           4h10m
f@df-2404lts:~$ k get sa -n azure-alb-system 
NAME                SECRETS   AGE
alb-controller-sa   0         4h10m
default             0         4h10m

```

checking the logs for error helps us to validate that everything is as expected.

```bash

f@df-2404lts:~$ k logs -n azure-alb-system deployments/alb-controller > alblog.json
Found 2 pods, using pod/alb-controller-86d7bc7694-nl8gt
Defaulted container "alb-controller" out of: alb-controller, init-alb-controller-crds (init), init-alb-controller-bootstrap (init)

```


```json

{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T13:05:19.799583379Z","message":"Starting alb-controller"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T13:05:19.803113126Z","message":"Starting alb-controller version 1.7.9"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T13:05:19.944422518Z","message":"attempting to acquire leader lease azure-alb-system/alb-controller-leader-election..."}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T13:05:22.889427539Z","message":"successfully acquired lease azure-alb-system/alb-controller-leader-election"}
{"level":"info","version":"1.7.9","name":"events","type":"Normal","object":{"kind":"Lease","namespace":"azure-alb-system","name":"alb-controller-leader-election","uid":"5ef1a476-c87c-46ce-a8a7-614f565ddc16","apiVersion":"coordination.k8s.io/v1","resourceVersion":"8939"},"reason":"LeaderElection","Timestamp":"2025-09-25T13:05:22.889774344Z","message":"alb-controller-86d7bc7694-nl8gt_e09b6ac8-e506-4e90-81c2-8a5f11a04300 became leader"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.Secret","Timestamp":"2025-09-25T13:05:22.890184549Z","message":"Starting EventSource"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.FrontendTLSPolicy","Timestamp":"2025-09-25T13:05:22.890394152Z","message":"Starting EventSource"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.IngressClass","Timestamp":"2025-09-25T13:05:22.890529254Z","message":"Starting EventSource"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.Ingress","Timestamp":"2025-09-25T13:05:22.890630155Z","message":"Starting EventSource"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.IngressExtension","Timestamp":"2025-09-25T13:05:22.890660455Z","message":"Starting EventSource"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","source":"kind source: *v1.Service","Timestamp":"2025-09-25T13:05:22.890718856Z","message":"Starting EventSource"}

```

Ok, let's admit that we live in a perfect world and everything went smoothly. Again, more on that in a dedicated part &#128517;

Let's try to actually expose apps.

## 3. Managing Gateway and HTTP routes with AGC

### 3.1. Using AGC Bring Your Own Deployment

To expose an app with the Gateway API, we need a Gateway, an Http route and then we reach the apps.

![illustration6](/assets/gapi/gatewayupstreamdownstream.png)

If we have 2 demo apps as below.

```yaml


apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-configmap1
 namespace: agctest
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
  namespace: agctest
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
  namespace: agctest
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
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: client1
  name: client1
  namespace: agctest
spec:
  replicas: 3
  selector:
    matchLabels:
      app: client1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: client1
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: index-html-configmap2
 namespace: agctest
data:
 index.html: |
   <html>
   <h1>Welcome to Demo App 2</h1>
   </br>
   <h2>This is a demo to illustrate Gateway API </h2>
   <img src="https://riseofgunpla.com/wp-content/uploads/2020/07/GUN83120_9.jpg" />
   </html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demoapp2
  name: demoapp2
  namespace:  agctest
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp2
  template:
    metadata:
      labels:
        app: demoapp2
    spec:
      volumes:
      - name: nginx-index-file
        configMap:
          name: index-html-configmap2
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
  name: demosvcapp2
  namespace:  agctest
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"
spec:
  type: ClusterIP
  ports:
  - port: 8090
    targetPort: 80
    protocol: TCP
  selector:
    app: demoapp2    

```

We can then create a Gateway. But before we check the GatewayClass.

```bash

df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io 
NAME                 CONTROLLER                               ACCEPTED   AGE
azure-alb-external   alb.networking.azure.io/alb-controller   True       4h24m

```

The Gateway definition is like this:

```bash

apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gateway-03
  namespace: agctest
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab2 
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http-listener
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: albfe-aks-lab2    

```

You may notice the from all namespaces configuration, which allows us to have Http routes from all namespaces associate to this Gateway.
You should also notice the addresses section, with its type specific to AGC and set to the front end that we created earlier.
Also, the annotation `alb.networking.azure.io/alb-id` which references the alb created in the Azure plane.

```bash

df@df-2404lts:~$ az network alb frontend list --alb-name alb-aks-lab2 -g rsg-spoke-aks
[
  {
    "fqdn": "esf3fzdegnfwacgb.fz50.alb.azure.com",
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab2/frontends/albfe-aks-lab2",
    "location": "francecentral",
    "name": "albfe-aks-lab2",
    "provisioningState": "Succeeded",
    "resourceGroup": "rsg-spoke-aks",
    "systemData": {
      "createdAt": "2025-09-25T13:47:27.3116839Z",
      "createdBy": "00000000-0000-0000-0000-000000000000",
      "createdByType": "Application",
      "lastModifiedAt": "2025-09-25T13:47:27.3116839Z",
      "lastModifiedBy": "00000000-0000-0000-0000-000000000000",
      "lastModifiedByType": "Application"
    },
    "tags": {},
    "type": "Microsoft.ServiceNetworking/TrafficControllers/Frontends"
  }
]

```

If the Gateway is working correctly, we should have an `Accepted` status, and we should find in the `addresses` section a `Hostname` type with an fqdn ending with `alb.azure.com`

```bash

df@df-2404lts:~$ k get gateway -n agctest gateway-03 -o json | jq .status
{
  "addresses": [
    {
      "type": "Hostname",
      "value": "esf3fzdegnfwacgb.fz50.alb.azure.com"
    }
  ],
  "conditions": [
    {
      "lastTransitionTime": "2025-09-25T14:23:27Z",
      "message": "Valid Gateway",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-09-25T14:23:27Z",
      "message": "Application Gateway for Containers resource has been successfully updated.",
      "observedGeneration": 1,
      "reason": "Programmed",
      "status": "True",
      "type": "Programmed"
    }
  ],
  "listeners": [
    {
      "attachedRoutes": 1,
      "conditions": [
        {
          "lastTransitionTime": "2025-09-25T14:23:27Z",
          "message": "",
          "observedGeneration": 1,
          "reason": "ResolvedRefs",
          "status": "True",
          "type": "ResolvedRefs"
        },
        {
          "lastTransitionTime": "2025-09-25T14:23:27Z",
          "message": "Listener is Accepted",
          "observedGeneration": 1,
          "reason": "Accepted",
          "status": "True",
          "type": "Accepted"
        },
        {
          "lastTransitionTime": "2025-09-25T14:23:27Z",
          "message": "Application Gateway for Containers resource has been successfully updated.",
          "observedGeneration": 1,
          "reason": "Programmed",
          "status": "True",
          "type": "Programmed"
        }
      ],
      "name": "http-listener",
      "supportedKinds": [
        {
          "group": "gateway.networking.k8s.io",
          "kind": "HTTPRoute"
        },
        {
          "group": "gateway.networking.k8s.io",
          "kind": "GRPCRoute"
        }
      ]
    }
  ]
}

```

We can now create an Http route to expose our demo app, as below.

```bash

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-httproute2
  namespace: agctest
spec:
  parentRefs:
  - name: gateway-03
  rules:
  - backendRefs:
    - name: demosvcapp1
      port: 8080
      kind: Service
  - backendRefs:
    - name: demosvcapp2
      port: 8090
      kind: Service
    matches:
    - path:
        type: PathPrefix
        value: /test 
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /     

```

If we try to reach the host on `/` or `/test`, we should get our application.

```bash

df@df-2404lts:~$ curl -i -X GET esf3fzdegnfwacgb.fz50.alb.azure.com
HTTP/1.1 200 OK
server: Microsoft-Azure-Application-LB/AGC
date: Thu, 25 Sep 2025 17:36:43 GMT
content-type: text/html
content-length: 188
last-modified: Thu, 25 Sep 2025 13:00:23 GMT
etag: "68d53ce7-bc"
accept-ranges: bytes

<html>
<h1>Welcome to Demo App 1</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://riseofgunpla.com/wp-content/uploads/2024/12/In-ERA-Lizard-9.jpg" />
</html
df@df-2404lts:~$ curl -i -X GET esf3fzdegnfwacgb.fz50.alb.azure.com/test
HTTP/1.1 200 OK
server: Microsoft-Azure-Application-LB/AGC
date: Thu, 25 Sep 2025 17:36:46 GMT
content-type: text/html
content-length: 183
last-modified: Thu, 25 Sep 2025 13:00:27 GMT
etag: "68d53ceb-b7"
accept-ranges: bytes

<html>
<h1>Welcome to Demo App 2</h1>
</br>
<h2>This is a demo to illustrate Gateway API </h2>
<img src="https://riseofgunpla.com/wp-content/uploads/2020/07/GUN83120_9.jpg" />
</html

```

Ok, so that's that, now what about the managed deployment?

### 3.2. Using AGC with the managed deployment.

As we said earlier, we do not really interact with the Azure plane in this scenario. 

We do however need a delegated subnet, on which the azure alb identity mush have access.

For this scenario, we switch to an Azure CNI Pod subnet cluster, instead of an Azure CNI overlay cluster. To keep it short, it's not working as expected betweenb Azure CNI Overlay and the managed AGC deployment.

```bash

df@df-2404lts:~$ az network vnet list -o table
Name           ResourceGroup    Location       NumSubnets    Prefixes       DnsServers    DDOSProtection    VMProtection
-------------  ---------------  -------------  ------------  -------------  ------------  ----------------  --------------
vnet-sbx-agw1  rsg-spoke-agw    francecentral  2             172.21.4.0/23                False
vnet-sbx-aks1  rsg-spoke-aks    francecentral  7             172.16.0.0/16                False

df@df-2404lts:~$ az network vnet subnet list --vnet-name vnet-sbx-agw1 -g rsg-spoke-agw -o table
AddressPrefix    DefaultOutboundAccess    Name                PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
---------------  -----------------------  ------------------  --------------------------------  -----------------------------------  -------------------  ---------------
172.21.4.0/24    True                     sub1-vnet-sbx-agw1  Disabled                          Enabled                              Succeeded            rsg-spoke-agw
172.21.5.0/24    True                     sub2-vnet-sbx-agw1  Disabled                          Enabled                              Succeeded            rsg-spoke-agw

df@df-2404lts:~$ az network vnet subnet list --vnet-name vnet-sbx-agw1 -g rsg-spoke-agw -o json |jq .[1]
{
  "addressPrefix": "172.21.5.0/24",
  "defaultOutboundAccess": true,
  "delegations": [
    {
      "actions": [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ],
      "etag": "W/\"95a80019-2123-49be-88f0-be8e1bf72ee5\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.Network/virtualNetworks/vnet-sbx-agw1/subnets/sub2-vnet-sbx-agw1/delegations/AlbDelegation",
      "name": "AlbDelegation",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg-spoke-agw",
      "serviceName": "Microsoft.ServiceNetworking/trafficControllers",
      "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
    }
  ],
  "etag": "W/\"95a80019-2123-49be-88f0-be8e1bf72ee5\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.Network/virtualNetworks/vnet-sbx-agw1/subnets/sub2-vnet-sbx-agw1",
  "name": "sub2-vnet-sbx-agw1",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-spoke-agw",
  "serviceEndpoints": [],
  "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

Instead of creating the ALB from the Azure plane, we use a specific crd related to AGC.

```bash

df@df-2404lts:~$ k get crd |grep alb.networking.azure.io
applicationloadbalancer.alb.networking.azure.io              2025-09-25T13:01:13Z
backendloadbalancingpolicy.alb.networking.azure.io           2025-09-25T13:01:13Z
backendtlspolicies.alb.networking.azure.io                   2025-09-25T13:01:13Z
frontendtlspolicies.alb.networking.azure.io                  2025-09-25T13:01:13Z
healthcheckpolicy.alb.networking.azure.io                    2025-09-25T13:01:13Z
ingressextension.alb.networking.azure.io                     2025-09-25T13:01:13Z
routepolicies.alb.networking.azure.io                        2025-09-25T13:01:13Z
webapplicationfirewallpolicy.alb.networking.azure.io         2025-09-25T13:01:16Z

```

The one we are interested in is the `applicationloadbalancer.alb.networking.azure.io`, with a configuration like this.

```yaml

apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-test2
  namespace: agcmanaged
spec:
  associations:
  - /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged


```

In this case, we kind of create the association ourselve with the `spec.associations` parameter, for which we specify the subnet shown earlier.

We can check the status of the ALB.

```bash

df@df-2404lts:~$ k get applicationloadbalancer.alb.networking.azure.io -n agcmanaged alb-test -o json|jq .status
{
  "conditions": [
    {
      "lastTransitionTime": "2025-09-26T14:06:15Z",
      "message": "Valid Application Gateway for Containers resource",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-09-26T14:06:15Z",
      "message": "alb-id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87",
      "observedGeneration": 1,
      "reason": "Ready",
      "status": "True",
      "type": "Deployment"
    }
  ]
}

```

And the alb controller logs.

```bash

df@df-2404lts:~$ k logs -n azure-alb-system deployments/alb-controller |grep alb-test
Found 2 pods, using pod/alb-controller-55884bcd8b-kh9n7
Defaulted container "alb-controller" out of: alb-controller, init-alb-controller-crds (init), init-alb-controller-bootstrap (init)
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","object":{"name":"alb-test","namespace":"agcmanaged"},"namespace":"agcmanaged","name":"alb-test","reconcileID":"4a30cb50-1c81-4692-b908-6ebefc96f404","Timestamp":"2025-09-26T14:01:52.688314105Z","message":"Reconciling"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-26T14:01:52.688333806Z","message":"Received request for object agcmanaged/alb-test"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","object":{"name":"alb-test","namespace":"agcmanaged"},"namespace":"agcmanaged","name":"alb-test","reconcileID":"4a30cb50-1c81-4692-b908-6ebefc96f404","Timestamp":"2025-09-26T14:01:52.688343306Z","message":"Reconcile successful"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-26T14:01:52.688356606Z","message":"Processing requested object agcmanaged/alb-test"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-26T14:01:52.688664611Z","message":"Successfully processed object agcmanaged/alb-test"}
{"level":"info","version":"1.7.9","operationID":"5a123591-abf9-4dba-b9f1-fe291e672f0d","Timestamp":"2025-09-26T14:01:52.688679811Z","message":"Starting event handler for Application Gateway for Containers resource agcmanaged/alb-test"}
{"level":"info","version":"1.7.9","AGC":"agcmanaged/alb-test","operationID":"5a123591-abf9-4dba-b9f1-fe291e672f0d","operationID":"5a123591-abf9-4dba-b9f1-fe291e672f0d","Timestamp":"2025-09-26T14:01:52.688687011Z","message":"Creating new Application Gateway for Containers resource Handler"}
{"level":"info","version":"1.7.9","operationID":"0a25824e-2b2b-476e-9d55-4650a3eea57a","Timestamp":"2025-09-26T14:01:52.768327785Z","message":"Triggering a config update for Application Gateway for Containers resource agcmanaged/alb-test"}
{"level":"info","version":"1.7.9","operationID":"04009b2e-fca6-42c3-a38b-78905df1bcb3","Timestamp":"2025-09-26T14:01:52.768337185Z","message":"Triggering an endpoint update for Application Gateway for Containers resource agcmanaged/alb-test with resources"}
{"level":"info","version":"1.7.9","AGC":"agcmanaged/alb-test","Timestamp":"2025-09-26T14:01:52.840321436Z","message":"Deploying Application Gateway for Containers resource: agcmanaged/alb-test"}

===============================truncated===============================

{"level":"info","version":"1.7.9","AGC":"agcmanaged/alb-test","alb-resource-id":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87","operationID":"2d7deeac-428c-400d-a40a-3fbcba02fea2","Timestamp":"2025-09-26T14:07:11.471908397Z","message":"Application Gateway for Containers resource config update OPERATION_STATUS_SUCCESS with operation ID 2d7deeac-428c-400d-a40a-3fbcba02fea2"}



```

We can find the newly created alb. We'll notice that the name in the kubernetes plane is not reflected on the Azure plane

```bash

df@df-2404lts:~$ az network alb list -o table |grep alb-b4577b87
francecentral  alb-b4577b87  Succeeded            rsg-aksobjectslab

df@df-2404lts:~$ az network alb show -n alb-b4577b87 -g rsg-aksobjectslab
{
  "associations": [
    {
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87/associations/as-83a18fd9",
      "resourceGroup": "rsg-aksobjectslab"
    }
  ],
  "configurationEndpoints": [
    "c0661d1af0aa462ba3dc6a07ce43d6d8.alb.azure.com"
  ],
  "frontends": [
    {
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87/frontends/fe-ac7d1a3b",
      "resourceGroup": "rsg-aksobjectslab"
    }
  ],
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87",
  "location": "francecentral",
  "name": "alb-b4577b87",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-aksobjectslab",
  "securityPolicies": [],
  "systemData": {
    "createdAt": "2025-09-26T14:01:53.3372852Z",
    "createdBy": "00000000-0000-0000-0000-000000000000",
    "createdByType": "Application",
    "lastModifiedAt": "2025-09-26T14:01:53.3372852Z",
    "lastModifiedBy": "00000000-0000-0000-0000-000000000000",
    "lastModifiedByType": "Application"
  },
  "tags": {
    "managed-by-alb-controller": "true"
  },
  "type": "Microsoft.ServiceNetworking/TrafficControllers"
}


df@df-2404lts:~$ az network alb association list --alb-name alb-b4577b87 -g rsg-aksobjectslab
[
  {
    "associationType": "subnets",
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-b4577b87/associations/as-83a18fd9",
    "location": "francecentral",
    "name": "as-83a18fd9",
    "provisioningState": "Succeeded",
    "resourceGroup": "rsg-aksobjectslab",
    "subnet": {
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.Network/virtualNetworks/vnet-sbx-agw1/subnets/sub2-vnet-sbx-agw1",
      "resourceGroup": "rsg-spoke-agw"
    },
    "systemData": {
      "createdAt": "2025-09-26T14:03:18.5224401Z",
      "createdBy": "00000000-0000-0000-0000-000000000000",
      "createdByType": "Application",
      "lastModifiedAt": "2025-09-26T14:03:18.5224401Z",
      "lastModifiedBy": "00000000-0000-0000-0000-000000000000",
      "lastModifiedByType": "Application"
    },
    "tags": {
      "managed-by-alb-controller": "true"
    },
    "type": "Microsoft.ServiceNetworking/TrafficControllers/Associations"
  }
]

```

So we can create a Gateway associated to an alb that is managed by the kubernetes plane. That's themanaged deployment.
Quite nice but not always working as expected. But for now we're ok.


Ok time to wrap up!

## 4. Summary

In this article, we started a new step in our Gateway API journey.

Indeed, we moved to an Azure Native Gateway API with the Application Gateway for Container.

As we've seen, it's possible to get a Gateway API relying on an Azure Application Gateway, kind of &#128517;.

Rather than a clasic Application Gateway we have a managed version of it, controlled through a kubernetes controller, which is the ALB controller.

After that, it's just Gateway API stuff.

There are some specificities though.

The first one being the 2 deployment models: either Bring Your Own, in which Azure resources are created from the Azure plan, and the managed one, where we delegazte as much as possible to the kubernetes plane.

Both scenarios are interesting, and answer different needs, not so much technicals rather than Organizationals.

The other specificity is the same one we had with AGIC: A Gateway API deployment that relies on Azure objects, so the operational considerations may be more in the Azure side than the Kubernetes side.

Typically, the WAF part, which we just intrduced, is kept in the Azure plane. Also the monitoring part, regarding alerting and logging is also kept in the Azure plane, an din Azure Monitor features.

All of this means that we will probably play more with AGC. Until then.








