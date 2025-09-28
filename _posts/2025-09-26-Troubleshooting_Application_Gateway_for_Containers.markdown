---
layout: post
title:  "Troubleshooting Application Gateway for container"
date:   2025-09-26 18:00:00 +0200
year: 2025
categories: Kubernetes AKS Network
---

Hi!

We juste completed a view of the Application Gateway For containers in a previous [article](/_posts/2025-09-25-Azure_Native_Gateway_API_with_Application_Gateway_for_Containers.markdown).

You may have felt that there was more to the story than juste something working smoothly.

This article is my journey to make it works. And because I don't really care about looking good (that's because I know a lost cause whan I see one &#128569;), I'll show you all the issues I encountered, and how I, when I could, troubleshooted those isues.

Our Agenda will be:


1. Troubleshooting the ALB Controller deployment
2. Is it really working with overlay?
3. Subnet size related issues
3. Conclusion

Let's get started!


## 1. Troubleshooting the ALB Controller deployment

The first issues encountered were at the very begining of the deployment, at the ALB installation. To be hosnest, I'm pretty sure those are recent issues, because I'm also quite sure It work before when I tried AGC a few month ago. But well, That was then, and now...

### 1.1. Chart the f&€k!

From the Azure documentation, we know that once our AKS cluster is up & running, and we have our identity configured as needed, we can deploy the ALB controller with the associated helm chart.

We don't use a standard `helm repo add` to get the repositopry of the chart but instead an [oci reference](https://helm.sh/docs/topics/registries/#using-an-oci-based-registry), because the chart is made available from Microsoft Container Registry.

In our case, the oci lionk is `oci://mcr.microsoft.com/application-lb/charts/alb-controller` and the helm install command is as below.

```bash

# Getting first the managed identity client id

df@df-2404lts:~$ az identity show -n azure-alb-identity -g rsg-spoke-agw
{
  "clientId": "00000000-0000-0000-0000-000000000000",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-spoke-agw/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azure-alb-identity",
  "location": "francecentral",
  "name": "azure-alb-identity",
  "principalId": "0aae3c50-08ac-4910-a1cd-76ab5950f93c",
  "resourceGroup": "rsg-spoke-agw",
  "systemData": null,
  "tags": {},
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"

# Installing the chart

df@df-2404lts:~$ helm upgrade alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller --version 1.7.9 --namespace "azure-alb-system" --cr
eate-namespace --set albController.namespace="azure-alb-system" --set albController.podIdentity.clientId="00000000-0000-0000-0000-000000000000" --install
Release "alb-controller" does not exist. Installing it now.
Pulled: mcr.microsoft.com/application-lb/charts/alb-controller:1.7.9
Digest: sha256:b75af198823bfc19df5b7613d6c51a8f60d03fc310df8c4b362692e1bea620c4
Error: values don't meet the specifications of the schema(s) in the following chart(s):
alb-controller:
json-pointer in "file:///values.schema.json#/properties/definitions/properties/imagePullSecret" not found

```

Except,as per the output, it's not working as we would like it to. The error messsage hint something related to a `values.schema.json` file.

So let's download the chart locally. 

We can achieve this with a `helm pull` command

```bash

df@df-2404lts:~/Documents$ helm pull oci://mcr.microsoft.com/application-lb/charts/alb-controller --untar --untardir ./localhelmcharts --version 1.7.9
Pulled: mcr.microsoft.com/application-lb/charts/alb-controller:1.7.9
Digest: sha256:b75af198823bfc19df5b7613d6c51a8f60d03fc310df8c4b362692e1bea620c4

```

And we get the following files in the `localhelmchart` folder.

```bash

df@df-2404lts:~/Documents$ tree ./localhelmcharts
./localhelmcharts
└── alb-controller
    ├── Chart.yaml
    ├── README.md
    ├── templates
    │   ├── alb-controller-cleanup-job.yaml
    │   ├── alb-controller-deployment.yaml
    │   ├── alb-controller-ha.yaml
    │   ├── alb-controller-service.yaml
    │   ├── common.yaml
    │   ├── _helper.tpl
    │   └── NOTES.txt
    ├── values.schema.json
    └── values.yaml

```

There is indeed a values.schema.json file present. The good thing is that with the download chart, we can have a more thorough look at it. 

But that's not why we are here, so let's do something rash, and just remove this file before retrying to install our chart.

```bash

df@df-2404lts:~/Documents$ rm ./localhelmcharts2/alb-controller/values.schema.json
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

This time it worked, and we have our service account and deployment in the `azure-alb-system` namespace.

```bash

df@df-2404lts:~$ k get sa -n azure-alb-system 
NAME                SECRETS   AGE
alb-controller-sa   0         3m28s
default             0         26m
df@df-2404lts:~$ k get deployments.apps -n azure-alb-system 
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
alb-controller   2/2     2            2           3m35s

```

So let's move on and try to deploy a Gateway.

### 1.2. Where is my workload identity in this mess?

This time, we are expecting to be able to deploy a Gateway, from a Bring Your Own deployment ALB. This was by far the more tricky I encountered and It took me some time to tackle It.

We have the following Gateway definition.

```yaml

apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gateway-01
  namespace: agctest
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab
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
      value: albfe-aks-lab


```

We did check that the GatewayClass was available.

```bash

df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io 
NAME                 CONTROLLER                               ACCEPTED   AGE
azure-alb-external   alb.networking.azure.io/alb-controller   True       30m
df@df-2404lts:~$ k get gatewayclasses.gateway.networking.k8s.io -o json |jq .items[].status
{
  "conditions": [
    {
      "lastTransitionTime": "2025-09-26T09:44:18Z",
      "message": "Valid GatewayClass",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    }
  ]
}

```

After deploying our Gateway however, we can see that it stays in an unknown status.

```bash

df@df-2404lts:~$ k get gateway -n agctest 
NAME         CLASS                ADDRESS   PROGRAMMED   AGE
gateway-01   azure-alb-external             Unknown      9s
df@df-2404lts:~$ k get gateway -n agctest gateway-01 -o json |jq .status
{
  "conditions": [
    {
      "lastTransitionTime": "1970-01-01T00:00:00Z",
      "message": "Waiting for controller",
      "reason": "Pending",
      "status": "Unknown",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "1970-01-01T00:00:00Z",
      "message": "Waiting for controller",
      "reason": "Pending",
      "status": "Unknown",
      "type": "Programmed"
    }
  ]
}

```

It does not look good. 

Let's have a look at the logs from the alb controller.

```bash

df@df-2404lts:~$ k logs -n azure-alb-system deployments/alb-controller 

```

```json

{
  "level": "info",
  "version": "1.7.9",
  "Timestamp": "2025-09-26T10:18:17.54527184Z",
  "message": "Retrying GetTrafficController after error: failed to get Application Gateway for Containers alb-aks-lab: DefaultAzureCredential: failed to acquire a token.\nAttempted credentials:\n\tEnvironmentCredential: missing environment variable AZURE_CLIENT_ID\n\tWorkloadIdentityCredential authentication failed. \n\t\tPOST https://login.microsoftonline.com/00000000-0000-0000-0000-000000000000/oauth2/v2.0/token\n\t\t--------------------------------------------------------------------------------\n\t\tRESPONSE 400: 400 Bad Request\n\t\t--------------------------------------------------------------------------------\n\t\t{\n\t\t  \"error\": \"unauthorized_client\",\n\t\t  \"error_description\": \"AADSTS700016: Application with identifier 'https://francecentral.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/' was not found in the directory 'dfitc'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant. You may have sent your authentication request to the wrong tenant. Trace ID: ce2bad1f-7eb1-4159-a2a1-1e4185da0e00 Correlation ID: 3459235a-918a-4d58-be57-8dc6ee4bfd4f Timestamp: 2025-09-26 10:18:17Z\",\n\t\t  \"error_codes\": [\n\t\t    700016\n\t\t  ],\n\t\t  \"timestamp\": \"2025-09-26 10:18:17Z\",\n\t\t  \"trace_id\": \"ce2bad1f-7eb1-4159-a2a1-1e4185da0e00\",\n\t\t  \"correlation_id\": \"3459235a-918a-4d58-be57-8dc6ee4bfd4f\",\n\t\t  \"error_uri\": \"https://login.microsoftonline.com/error?code=700016\"\n\t\t}\n\t\t--------------------------------------------------------------------------------\n\t\tTo troubleshoot, visit https://aka.ms/azsdk/go/identity/troubleshoot#workload. Attempt: 2"
}
{
  "level": "info",
  "version": "1.7.9",
  "Timestamp": "2025-09-26T10:19:17.591747763Z",
  "message": "Retrying GetTrafficController after error: failed to get Application Gateway for Containers alb-aks-lab: DefaultAzureCredential: failed to acquire a token.\nAttempted credentials:\n\tEnvironmentCredential: missing environment variable AZURE_CLIENT_ID\n\tWorkloadIdentityCredential authentication failed. \n\t\tPOST https://login.microsoftonline.com/00000000-0000-0000-0000-000000000000/oauth2/v2.0/token\n\t\t--------------------------------------------------------------------------------\n\t\tRESPONSE 400: 400 Bad Request\n\t\t--------------------------------------------------------------------------------\n\t\t{\n\t\t  \"error\": \"unauthorized_client\",\n\t\t  \"error_description\": \"AADSTS700016: Application with identifier 'https://francecentral.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/' was not found in the directory 'dfitc'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant. You may have sent your authentication request to the wrong tenant. Trace ID: 741042a7-9880-48bf-86ea-af26d20c0e00 Correlation ID: 4be0058d-407e-4783-a40b-af987cef1f50 Timestamp: 2025-09-26 10:19:17Z\",\n\t\t  \"error_codes\": [\n\t\t    700016\n\t\t  ],\n\t\t  \"timestamp\": \"2025-09-26 10:19:17Z\",\n\t\t  \"trace_id\": \"741042a7-9880-48bf-86ea-af26d20c0e00\",\n\t\t  \"correlation_id\": \"4be0058d-407e-4783-a40b-af987cef1f50\",\n\t\t  \"error_uri\": \"https://login.microsoftonline.com/error?code=700016\"\n\t\t}\n\t\t--------------------------------------------------------------------------------\n\t\tTo troubleshoot, visit https://aka.ms/azsdk/go/identity/troubleshoot#workload. Attempt: 3"
}
{
  "level": "error",
  "version": "1.7.9",
  "AGC": "alb-aks-lab",
  "error": "failed to get location for Application Gateway for Containers resource, failed to get Application Gateway for Containers alb-aks-lab: DefaultAzureCredential: failed to acquire a token.\nAttempted credentials:\n\tEnvironmentCredential: missing environment variable AZURE_CLIENT_ID\n\tWorkloadIdentityCredential authentication failed. \n\t\tPOST https://login.microsoftonline.com/00000000-0000-0000-0000-000000000000/oauth2/v2.0/token\n\t\t--------------------------------------------------------------------------------\n\t\tRESPONSE 400: 400 Bad Request\n\t\t--------------------------------------------------------------------------------\n\t\t{\n\t\t  \"error\": \"unauthorized_client\",\n\t\t  \"error_description\": \"AADSTS700016: Application with identifier 'https://francecentral.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/' was not found in the directory 'dfitc'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant. You may have sent your authentication request to the wrong tenant. Trace ID: 741042a7-9880-48bf-86ea-af26d20c0e00 Correlation ID: 4be0058d-407e-4783-a40b-af987cef1f50 Timestamp: 2025-09-26 10:19:17Z\",\n\t\t  \"error_codes\": [\n\t\t    700016\n\t\t  ],\n\t\t  \"timestamp\": \"2025-09-26 10:19:17Z\",\n\t\t  \"trace_id\": \"741042a7-9880-48bf-86ea-af26d20c0e00\",\n\t\t  \"correlation_id\": \"4be0058d-407e-4783-a40b-af987cef1f50\",\n\t\t  \"error_uri\": \"https://login.microsoftonline.com/error?code=700016\"\n\t\t}\n\t\t--------------------------------------------------------------------------------\n\t\tTo troubleshoot, visit https://aka.ms/azsdk/go/identity/troubleshoot#workload",
  "Timestamp": "2025-09-26T10:19:17.591808665Z",
  "message": "Internal error. See error tag for more information"
}

```

There is a lot of information here.

One message could hint that the identity may be misconfigured. To check this, we will use the same identity with a CSI secret store also using the azure alb identity.

For this, we need a `SecretProviderClass` and a deployment mounting the keyvault secret through a CSI volume.
Obviously we need to prepare the proper requirement for the key vault to be used, which are the role assignment on the key vault mainly. 
We will rely on the service account used by the alb controller, and, using the same identity, 

```bash

df@df-2404lts:~$ az role assignment list --resource-group rsg-kv | jq '.[].roleDefinitionName,.[].principalName'
"Key Vault Secrets User"
"Reader"
"00000000-0000-0000-0000-000000000000"
"00000000-0000-0000-0000-000000000000"

```

```yaml

---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: kvlab-testwithalbidentity
  namespace: azure-alb-system
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"               
    clientID: "00000000-0000-0000-0000-000000000000"
    keyvaultName: kvdemo
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: newsecret
          objectAlias: ""            
          objectType: secret                    
          objectVersion: ""       
    tenantId: "00000000-0000-0000-0000-000000000000"     
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: testalbidentitykvcsi
  name: testalbidentitykvcsi
  namespace: azure-alb-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testalbidentitykvcsi
  strategy: {}
  template:
    metadata:
      labels:
        app: testalbidentitykvcsi
    spec:
      serviceAccount: alb-controller-sa
      volumes:
      - name: csisecret
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: kvlab-testwithalbidentity
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
          - mountPath: /mnt/secrets-store
            name: csisecret
            readOnly: true
        resources: {}        


```

We can check that it's working correctly by looking at the mounted volume, which should match the SecretProviderClass name, and looking to the mounted secret inside the pod.

```bash

df@df-2404lts:~$ az keyvault secret show --vault-name kvdemo --name newsecret
{
  "attributes": {
    "created": "2025-09-26T07:57:20+00:00",
    "enabled": true,
    "expires": null,
    "notBefore": null,
    "recoverableDays": 90,
    "recoveryLevel": "Recoverable",
    "updated": "2025-09-26T07:57:20+00:00"
  },
  "contentType": "",
  "id": "https://kvdemo.vault.azure.net/secrets/newsecret/6255dc4bf5c44ee6be18d716c58f850c",
  "kid": null,
  "managed": null,
  "name": "newsecret",
  "tags": {},
  "value": "this_is_a_secret"
}

df@df-2404lts:~$ k get pod -n azure-alb-system testalbidentitykvcsi-7847957c4f-8f5w6 -o json |jq .spec.volumes[0]
{
  "csi": {
    "driver": "secrets-store.csi.k8s.io",
    "readOnly": true,
    "volumeAttributes": {
      "secretProviderClass": "kvlab-testwithalbidentity"
    }
  },
  "name": "csisecret"
}

df@df-2404lts:~$ k exec -n azure-alb-system testalbidentitykvcsi-7847957c4f-8f5w6 -- ls /mnt/secrets-store
newsecret
df@df-2404lts:~$ k exec -n azure-alb-system testalbidentitykvcsi-7847957c4f-8f5w6 -- cat /mnt/secrets-store/newsecret
this_is_a_secret

```

Well, it means that the managed identity is not misconfigured, so what else could we look?

Another error message mention some missing environment variable so let's have a look at the object configuration.

```bash

df@df-2404lts:~$ k get serviceaccounts -n azure-alb-system alb-controller-sa -o yaml

```

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ""
    meta.helm.sh/release-name: alb-controller
    meta.helm.sh/release-namespace: azure-alb-system
  creationTimestamp: "2025-09-26T10:07:10Z"
  labels:
    app.kubernetes.io/managed-by: Helm
    azure.workload.identity/use: "true"
  name: alb-controller-sa
  namespace: azure-alb-system
  resourceVersion: "36406"
  uid: 6677e447-52f6-48d1-bb0b-029b06ea87fd

```

First we can see that the value for the annotation `azure.workload.identity/client-id` is not filled. However, It should not casue problem, since the same service account is used to grant acess to a Key Vault.

Now we can look at the alb deployment, and specifically the `env` section of the containers.

```bash

df@df-2404lts:~$ k get deployments.apps -n azure-alb-system alb-controller -o json |jq .spec.template.spec.containers[].env
[
  {
    "name": "AZURE_CLIENT_ID"
  }
]

```

Ok, let's the deployment environment variable first.

```bash

df@df-2404lts:~$ k get deployments.apps -n azure-alb-system alb-controller -o json |jq .spec.template.spec.containers[].env
[
  {
    "name": "AZURE_CLIENT_ID",
    "value": "00000000-0000-0000-0000-000000000000"
  }
]

```

And then, the service account, just to be thorough in our configuration.

```bash

df@df-2404lts:~$ k get -n azure-alb-system serviceaccounts alb-controller-sa -o json |jq .metadata.annotations
{
  "azure.workload.identity/client-id": "00000000-0000-0000-0000-000000000000",
  "meta.helm.sh/release-name": "alb-controller",
  "meta.helm.sh/release-namespace": "azure-alb-system"
}

```

Once we recreated our prvious Gateway, we can see in the alb controller logs that it seems better.

```json

{
  "level": "info",
  "version": "1.7.9",
  "AGC": "alb-aks-lab",
  "alb-resource-id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab",
  "operationID": "dee31c49-97d2-4dad-9c07-95421c11520c",
  "Timestamp": "2025-09-26T13:42:27.643367539Z",
  "message": "Application Gateway for Containers resource config update OPERATION_STATUS_SUCCESS with operation ID dee31c49-97d2-4dad-9c07-95421c11520c"
}

```

And indeed, the Gateway is in an accepted state, with a frontend associated, giving us an fqdn.

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ k get gateway -n agctest gateway-01 
NAME         CLASS                ADDRESS                               PROGRAMMED   AGE
gateway-01   azure-alb-external   bhapesfdhjhsfkbf.fz66.alb.azure.com   True         3m4s

df@df-2404lts:~/Documents/dfrappart.github.io$ k get gateway -n agctest gateway-01 -o json |jq .status
{
  "addresses": [
    {
      "type": "Hostname",
      "value": "bhapesfdhjhsfkbf.fz66.alb.azure.com"
    }
  ],
  "conditions": [
    {
      "lastTransitionTime": "2025-09-26T13:42:26Z",
      "message": "Valid Gateway",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-09-26T13:42:27Z",
      "message": "Application Gateway for Containers resource has been successfully updated.",
      "observedGeneration": 1,
      "reason": "Programmed",
      "status": "True",
      "type": "Programmed"
    }
  ],
  "listeners": [
    {
      "attachedRoutes": 0,
      "conditions": [
        {
          "lastTransitionTime": "2025-09-26T13:42:26Z",
          "message": "",
          "observedGeneration": 1,
          "reason": "ResolvedRefs",
          "status": "True",
          "type": "ResolvedRefs"
        },
        {
          "lastTransitionTime": "2025-09-26T13:42:26Z",
          "message": "Listener is Accepted",
          "observedGeneration": 1,
          "reason": "Accepted",
          "status": "True",
          "type": "Accepted"
        },
        {
          "lastTransitionTime": "2025-09-26T13:42:27Z",
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

So that's how we can make AGC work, Azure CNI Overlay or not. Speaking of Overlay, let's have a look at some other issues encountered.

## 2. Is it really working with overlay?

This issue was encountered with a bring your own deployement. 
The secificity was regarding the network configuration, in which the aim was to deploy AGC in another vnet than the AKS cluster.

We get 2 Vnets peered vnet with the alb and its child resources created beforehand in a dedicated Vnet.

Of course, we follow the requirements in terms of subnet size which should be `/24`

![illustration7](/assets/agc/agcdeployment2vnets.png)

If we deploy a Gateway, followed by an Http route, everything seems fine.

```yaml

apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gateway-01
  namespace: agctest
  annotations:
    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab 
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
      value: albfe-aks-lab
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-httproute
  namespace: agctest
spec:
  parentRefs:
  - name: gateway-01
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

However, the app is not reachable as can be seen in the below output.

```bash

df@df-2404lts:~$ curl -i -X GET bhapesfdhjhsfkbf.fz66.alb.azure.com
HTTP/1.1 503 Service Unavailable
content-length: 19
content-type: text/plain
date: Thu, 25 Sep 2025 21:38:17 GMT
server: Microsoft-Azure-Application-LB/AGC

no healthy upstream

```

Indeed, checking the logs, it seems that it does not reconcile the association because it's not in the range of Vnet used by AKS.

```bash

df@df-2404lts:~$ k logs -n azure-alb-system deployments/alb-controller |grep gateway-01

{"level":"error","version":"1.7.9","AGC":"alb-aks-lab","alb-resource-id":"/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-agw/providers/Microsoft.ServiceNetworking/trafficControllers/alb-aks-lab","error":"failed to reconcile subnet association: failed to reconcile overlay resources: Overlay extension config azure-alb-system/alb-overlay-extension-config-d22b74b6 failed with error: OEC IP range 172.21.4.0/24 is not within any of the VNet CIDRs 172.16.0.0/16","Timestamp":"2025-09-25T21:33:01.991979766Z","message":"Retrying to process the request."}

```

Azure CNI Overlay seems to be limited to deployment within the same vnet for AGC and the AKS cluster. An important point to take into account.

Let's move on and check another scenario

## 3. Subnet size related issues

### 3.1. Subnet too small?

In this scenario, we have an AKS cluster with Azure CNI overlay. We want to test the managed deployment, so we only have to prepare a subnet of the appropriate size which, from the documentation, is supposedly a `/24`.

```bash

df@df-2404lts:~$ az network vnet subnet list --vnet-name vnet-sbx-aks1 -g rsg-spoke-aks -o json |jq .[4]
{
  "addressPrefix": "172.16.14.0/24",
  "defaultOutboundAccess": true,
  "delegations": [
    {
      "actions": [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ],
      "etag": "W/\"98902553-60ab-4fe9-9fbb-cc8d09e88575\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged/delegations/AlbDelegation",
      "name": "AlbDelegation",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg-spoke-aks",
      "serviceName": "Microsoft.ServiceNetworking/trafficControllers",
      "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
    }
  ],
  "etag": "W/\"98902553-60ab-4fe9-9fbb-cc8d09e88575\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged",
  "ipConfigurations": [...],
  "name": "AlbSubnetmanaged",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-spoke-aks",
  "serviceEndpoints": [],
  "type": "Microsoft.Network/virtualNetworks/subnets"
}

``` 

We then define an alb from its corresponding crd in kubernetes.

```bash

apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-test2
  namespace: agcmanaged
spec:
  associations:
  - /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged


```

Checking the status of the alb, we have an on going deployment at first, which seems ok.

```bash


df@df-2404lts:~$ k get applicationloadbalancer.alb.networking.azure.io -n agcmanaged alb-test2 -o json | jq .status
{
  "conditions": [
    {
      "lastTransitionTime": "2025-09-25T18:31:59Z",
      "message": "Valid Application Gateway for Containers resource",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-09-25T18:31:59Z",
      "message": "Application Gateway for Containers resource alb-497eb140 is undergoing an update.",
      "observedGeneration": 1,
      "reason": "InProgress",
      "status": "True",
      "type": "Deployment"
    }
  ]
}


```

Logs from the alb controller looks ok too.

```bash

df@df-2404lts:~$ k logs -n azure-alb-system deployments/alb-controller |grep alb-test2
Found 2 pods, using pod/alb-controller-86d7bc7694-nl8gt
Defaulted container "alb-controller" out of: alb-controller, init-alb-controller-crds (init), init-alb-controller-bootstrap (init)
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","object":{"name":"alb-test2","namespace":"agcmanaged"},"namespace":"agcmanaged","name":"alb-test2","reconcileID":"6f1cc301-7a99-4ac9-810c-8c045823b72a","Timestamp":"2025-09-25T18:31:59.177027962Z","message":"Reconciling"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T18:31:59.177050662Z","message":"Received request for object agcmanaged/alb-test2"}
{"level":"info","version":"1.7.9","controller":"lb-resources-reconciler","object":{"name":"alb-test2","namespace":"agcmanaged"},"namespace":"agcmanaged","name":"alb-test2","reconcileID":"6f1cc301-7a99-4ac9-810c-8c045823b72a","Timestamp":"2025-09-25T18:31:59.177061563Z","message":"Reconcile successful"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T18:31:59.177076563Z","message":"Processing requested object agcmanaged/alb-test2"}
{"level":"info","version":"1.7.9","Timestamp":"2025-09-25T18:31:59.177943274Z","message":"Successfully processed object agcmanaged/alb-test2"}
{"level":"info","version":"1.7.9","operationID":"462051cc-ee0a-4161-a1a6-031256e05d07","Timestamp":"2025-09-25T18:31:59.177965774Z","message":"Starting event handler for Application Gateway for Containers resource agcmanaged/alb-test2"}
{"level":"info","version":"1.7.9","AGC":"agcmanaged/alb-test2","operationID":"462051cc-ee0a-4161-a1a6-031256e05d07","operationID":"462051cc-ee0a-4161-a1a6-031256e05d07","Timestamp":"2025-09-25T18:31:59.177974274Z","message":"Creating new Application Gateway for Containers resource Handler"}
{"level":"info","version":"1.7.9","operationID":"0b163b87-49c1-4820-aa33-8f3c5361e149","Timestamp":"2025-09-25T18:31:59.272533228Z","message":"Triggering a config update for Application Gateway for Containers resource agcmanaged/alb-test2"}
{"level":"info","version":"1.7.9","operationID":"fb096b57-4413-4ada-888e-bbafc9a1e088","Timestamp":"2025-09-25T18:31:59.272542628Z","message":"Triggering an endpoint update for Application Gateway for Containers resource agcmanaged/alb-test2 with resources"}
{"level":"info","version":"1.7.9","AGC":"agcmanaged/alb-test2","Timestamp":"2025-09-25T18:31:59.382784926Z","message":"Deploying Application Gateway for Containers resource: agcmanaged/alb-test2"}

```

On the Azure side, we find the alb as expected, in the AKS managed resource group.

```bash

df@df-2404lts:~$ az network alb list -o table
Location       Name          ProvisioningState    ResourceGroup
-------------  ------------  -------------------  -----------------
francecentral  alb-aks-lab   Succeeded            rsg-spoke-agw
francecentral  alb-aks-lab2  Succeeded            rsg-spoke-aks
francecentral  alb-497eb140  Succeeded            rsg-aksobjectslab

```

However, checking again the alb in kubernetes, this time we get a strange behavior. The deployment status is showing `false`

```bash
df@df-2404lts:~$  k get applicationloadbalancer.alb.networking.azure.io -A
NAMESPACE    NAME        DEPLOYMENT   AGE
agcmanaged   alb-test2   False        21m

```

Specifically, we can find an error related to the size of the subnet, which seems to be too small.

```bash

df@df-2404lts:~$ k describe applicationloadbalancer.alb.networking.azure.io -n agcmanaged alb-test2 
Name:         alb-test2
Namespace:    agcmanaged
Labels:       <none>
Annotations:  <none>
API Version:  alb.networking.azure.io/v1
Kind:         ApplicationLoadBalancer
Metadata:
  Creation Timestamp:  2025-09-25T18:31:59Z
  Generation:          1
  Resource Version:    128816
  UID:                 17f32f08-3fc1-4e1f-8c11-01fda1661e68
Spec:
  Associations:
    /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged
Status:
  Conditions:
    Last Transition Time:  2025-09-25T18:43:23Z
    Message:               Valid Application Gateway for Containers resource
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2025-09-25T18:43:23Z
    Message:               error=Unable to create association for Application Gateway for Containers resource /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-497eb140: failed to create association as-717d8cf9 for Application Gateway for Containers alb-497eb140: PUT https://management.azure.com/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-497eb140/associations/as-717d8cf9
--------------------------------------------------------------------------------
RESPONSE 400: 400 Bad Request
ERROR CODE: AppGwForContainersSubnetAssociationNotEnoughIPv4AddressesInSubnet
--------------------------------------------------------------------------------
{
  "error": {
    "code": "AppGwForContainersSubnetAssociationNotEnoughIPv4AddressesInSubnet",
    "message": "Application Gateway for Containers: Subnet '/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanaged' does not have enough IPv4 addresses. At least 500 addresses are required to create Association '/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-497eb140/associations/as-717d8cf9' for this subnet."
  },
  "status": "Failed"
}
--------------------------------------------------------------------------------

    Observed Generation:  1
    Reason:               Ready
    Status:               False
    Type:                 Deployment
Events:                   <none>

```

It seems that the alb association is failing because the subnet does not have enough addresses.
That's strange, but well, let's admit that and try to fix the issue. 

We can try to manually create the association from the portal. It's not a fully managed anymore but well, let's try it anyway.

![illustration8](/assets/agc/subnettoosmall001.png)

![illustration9](/assets/agc/subnettoosmall002.png)

![illustration10](/assets/agc/subnettoosmall003.png)

Well, at least the result is consistent, so it's not working at all.

We can try with a larger subnet.


### 3.2. Still not the proper size

So this time we are trying with a bigger subnet, with a `/23` cidr

```bash

df@df-2404lts:~$ az network vnet subnet list --vnet-name vnet-sbx-aks1 -g rsg-spoke-aks -o table
AddressPrefix    DefaultOutboundAccess    Name                    PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
---------------  -----------------------  ----------------------  --------------------------------  -----------------------------------  -------------------  ---------------
172.16.12.0/26   True                     AzureBastionSubnet      Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.0.0/22    True                     sub1-vnet-sbx-aks1      Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.8.0/22    True                     sub3-vnet-sbx-aks1      Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.4.0/22    True                     sub2-vnet-sbx-aks1      Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.14.0/24   True                     AlbSubnetmanaged        Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.13.0/24   True                     AlbSubnetbyo            Disabled                          Enabled                              Succeeded            rsg-spoke-aks
172.16.16.0/23   True                     AlbSubnetmanagedbigger  Disabled                          Enabled                              Succeeded            rsg-spoke-aks

df@df-2404lts:~$ az network vnet subnet list --vnet-name vnet-sbx-aks1 -g rsg-spoke-aks -o json |jq .[6]
{
  "addressPrefix": "172.16.16.0/23",
  "defaultOutboundAccess": true,
  "delegations": [
    {
      "actions": [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ],
      "etag": "W/\"8910a5c7-b9d6-42b4-b4de-89feb5608970\"",
      "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanagedbigger/delegations/AlbDelegation",
      "name": "AlbDelegation",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg-spoke-aks",
      "serviceName": "Microsoft.ServiceNetworking/trafficControllers",
      "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
    }
  ],
  "etag": "W/\"8910a5c7-b9d6-42b4-b4de-89feb5608970\"",
  "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanagedbigger",
  "ipConfigurations": [...],
  "name": "AlbSubnetmanagedbigger",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg-spoke-aks",
  "serviceEndpoints": [],
  "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

Defining a new alb, we are full of hope &#128568;.

```yaml

apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-test3
  namespace: agcmanaged
spec:
  associations:
  - /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanagedbigger


```

But then

```bash

df@df-2404lts:~$ k describe applicationloadbalancer.alb.networking.azure.io -n agcmanaged alb-test3
Name:         alb-test3
Namespace:    agcmanaged
Labels:       <none>
Annotations:  <none>
API Version:  alb.networking.azure.io/v1
Kind:         ApplicationLoadBalancer
Metadata:
  Creation Timestamp:  2025-09-25T19:10:40Z
  Finalizers:
    alb-deployment-exists-finalizer.alb.networking.azure.io
  Generation:        1
  Resource Version:  140948
  UID:               3a1dd387-9448-4795-97bb-bf22747560d2
Spec:
  Associations:
    /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanagedbigger
Status:
  Conditions:
    Last Transition Time:  2025-09-25T19:17:44Z
    Message:               Valid Application Gateway for Containers resource
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2025-09-25T19:17:44Z
    Message:               alb-id=/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53 error=failed to reconcile overlay resources: Overlay extension config azure-alb-system/alb-overlay-extension-config-78712412 failed with error: extensionIPRange prefix length too short (/23). Must be 24 or higher.
    Observed Generation:   1
    Reason:                Ready
    Status:                False
    Type:                  Deployment
Events:                    <none>

```

And I'll admit I'm lost this time. I mean, it was too few IPs earlier, and now `/23` is not enough, we need `/24`.

It's even more confusing because it does look like the association is working.

```bash


df@df-2404lts:~$ az network alb association list --alb-name alb-97d3de53 -g rsg-aksobjectslab
[
  {
    "associationType": "subnets",
    "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53/associations/as-78712412",
    "location": "francecentral",
    "name": "as-78712412",
    "provisioningState": "Succeeded",
    "resourceGroup": "rsg-aksobjectslab",
    "subnet": {
      "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/virtualNetworks/vnet-sbx-aks1/subnets/AlbSubnetmanagedbigger",
      "resourceGroup": "rsg-spoke-aks"
    },
    "systemData": {
      "createdAt": "2025-09-25T19:12:03.549084Z",
      "createdBy": "c180ca71-1ca8-4166-bcf6-40fc1d26f04b",
      "createdByType": "Application",
      "lastModifiedAt": "2025-09-25T19:12:03.549084Z",
      "lastModifiedBy": "c180ca71-1ca8-4166-bcf6-40fc1d26f04b",
      "lastModifiedByType": "Application"
    },
    "tags": {
      "managed-by-alb-controller": "true"
    },
    "type": "Microsoft.ServiceNetworking/TrafficControllers/Associations"
  }
]

```

We don't have a frontend though.

```bash

df@df-2404lts:~$ az network alb frontend list --alb-name alb-97d3de53 -g rsg-aksobjectslab
[]

```

We could try to deploy one from the portal, and thus moving to the not so managed area.
Somehow it's working, as we can see with the command output below

```bash

df@df-2404lts:~$ az network alb frontend list --alb-name alb-97d3de53 -g rsg-aksobjectslab
[
  {
    "fqdn": "czefhpb8gcfhf3gd.fz37.alb.azure.com",
    "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourcegroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53/Frontends/albfetb44",
    "location": "francecentral",
    "name": "albfetb44",
    "provisioningState": "Succeeded",
    "resourceGroup": "rsg-aksobjectslab",
    "systemData": {
      "createdAt": "2025-09-25T19:55:25.8009959Z",
      "createdBy": "david@teknews.cloud",
      "createdByType": "User",
      "lastModifiedAt": "2025-09-25T19:55:25.8009959Z",
      "lastModifiedBy": "david@teknews.cloud",
      "lastModifiedByType": "User"
    },
    "type": "Microsoft.ServiceNetworking/TrafficControllers/Frontends"
  }
]

```

Let's try to move forward with a gateway.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-05
  namespace: agcmanaged
  annotations:
#    alb.networking.azure.io/alb-namespace: agcmanaged
#    alb.networking.azure.io/alb-name: alb-test3
    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All 
  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: albfetb44  

```

Notice the commented values for `alb.networking.azure.io/alb-namespace` and `alb.networking.azure.io/alb-name`, replaced by the `alb.networking.azure.io/alb-id`.

We get this result, with a seemingly working gateway

```bash

df@df-2404lts:~$ k get gateway -n agcmanaged gateway-05
NAME         CLASS                ADDRESS                               PROGRAMMED   AGE
gateway-05   azure-alb-external   czefhpb8gcfhf3gd.fz37.alb.azure.com   True         3m19s

df@df-2404lts:~$ k get gateway -n agcmanaged gateway-05 -o json | jq .status
{
  "addresses": [
    {
      "type": "Hostname",
      "value": "czefhpb8gcfhf3gd.fz37.alb.azure.com"
    }
  ],
  "conditions": [
    {
      "lastTransitionTime": "2025-09-25T20:06:20Z",
      "message": "Valid Gateway",
      "observedGeneration": 1,
      "reason": "Accepted",
      "status": "True",
      "type": "Accepted"
    },
    {
      "lastTransitionTime": "2025-09-25T20:06:21Z",
      "message": "Application Gateway for Containers resource has been successfully updated.",
      "observedGeneration": 1,
      "reason": "Programmed",
      "status": "True",
      "type": "Programmed"
    }
  ],
  "listeners": [
    {
      "attachedRoutes": 0,
      "conditions": [
        {
          "lastTransitionTime": "2025-09-25T20:06:20Z",
          "message": "",
          "observedGeneration": 1,
          "reason": "ResolvedRefs",
          "status": "True",
          "type": "ResolvedRefs"
        },
        {
          "lastTransitionTime": "2025-09-25T20:06:20Z",
          "message": "Listener is Accepted",
          "observedGeneration": 1,
          "reason": "Accepted",
          "status": "True",
          "type": "Accepted"
        },
        {
          "lastTransitionTime": "2025-09-25T20:06:21Z",
          "message": "Application Gateway for Containers resource has been successfully updated.",
          "observedGeneration": 1,
          "reason": "Programmed",
          "status": "True",
          "type": "Programmed"
        }
      ],
      "name": "http",
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

By using `alb.networking.azure.io/alb-id` and setting the value to the alb id, we are moving to a kind of bring your own deployment, because we admit that the alb should exist in Azure already so that we can get its Id. 
The managed deployment  relies on the `alb.networking.azure.io/alb-namespace` and `alb.networking.azure.io/alb-name` annotations.
Also, in this semi-managed mode, we used the section `addresses` in which an existing frontend is specified.

```yaml

  addresses:
    - type: alb.networking.azure.io/alb-frontend
      value: albfetb44  

```

If we do not specify this section, we have a new front end created. 

For the purpose of illustration, we will define 2 new gateways with different combinations of the annotations.

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-06
  namespace: agcmanaged
  annotations:
    alb.networking.azure.io/alb-namespace: agcmanaged
    alb.networking.azure.io/alb-name: alb-test3
    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All 
#  addresses:
#    - type: alb.networking.azure.io/alb-frontend
#      value: albfetb44  
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-07
  namespace: agcmanaged
  annotations:
    alb.networking.azure.io/alb-namespace: agcmanaged
    alb.networking.azure.io/alb-name: alb-test3
#    alb.networking.azure.io/alb-id: /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-aksobjectslab/providers/Microsoft.ServiceNetworking/trafficControllers/alb-97d3de53
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All 
#  addresses:
#    - type: alb.networking.azure.io/alb-frontend
#      value: albfetb44  

```

Both gateways show a accepted status, with an address corresponding to the created frontends.

```bash

df@df-2404lts:~$ k get gateway -n agcmanaged -o custom-columns=name:.metadata.name,namespace:.metadata.namespace,address:.status.addresses[0].value,condition:.status.conditions[0].reason
name         namespace    address                               condition
gateway-05   agcmanaged   czefhpb8gcfhf3gd.fz37.alb.azure.com   Accepted
gateway-06   agcmanaged   dvgeb7bgg5gxgugk.fz41.alb.azure.com   Accepted
gateway-07   agcmanaged   f7gwewdqb2d7a2bv.fz73.alb.azure.com   Accepted

```

![illustration11](/assets/agc/agcmanaged001.png)



Ok time to wrap up!

## 4. Summary

Soooooo!

Another eventful journey right?


To summarize:

Application Gateway works... supposedly.

Currently there are some isue with the chart which does not propagate the client Id of the managed identity where it should.
A not so easy issue to tackle, as mentionned already.

Once the ALB controller is deployed and working, depending on the underlying network, we may have more trouble.

First, Azure CNI Overlay is fine, as long as the deployment of the AGC is in the same Vnet range than the Cluster. Otherwise, we encounter errors, hinting that the multi-vnet deployment is not functionnal yet.

Second, the managed deployment is not totally working. For this, I'm not totaly clear on what's not ok and what is. Supposedly, a CIDR of `/24` fopr the ALB subnet. But we saw that we can get error with a /24 or a /23. So maybe it's just the managed deployment which is not working fantastically. Or maybe it's more related to the Azure CNI Overlay. I did not push it to find out what may not work with an Azure CNI Pod Subnet (I mean for the managed deployment).

Also, I did not try with an Azure CNI Powered by Cilium. I kind of though it may have been too risky &#129325;

Anyway, I hope it was intersting for some of you.

I'll be back soon for more AGC I think, and definitly more Gateway API.

Have fun ^^







