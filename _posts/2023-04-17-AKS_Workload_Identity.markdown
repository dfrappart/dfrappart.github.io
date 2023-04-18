---
layout: post
title:  "Workload Identity in AKS"
date:   2023-04-17 23:00:00 +0200
year: 2023
categories: AKS Security
---

Hey people!

It's been long overdue. Here is an article about Workload Identity in an AKS context.
The agenda:

- Before Workload Identity, the need and the solution
- Workload Identity concepts
- Implementing Workload Identity in AKS

## 1. Before Workload Identity

So, workload Identity have been around for quite some time now, but is still currently in preview. It was planned to go on GA in early april, but was finally pushed away to later in the month (and it may be GA when I'll publish this ^^).
That being said, let's step back a little and try to understand the need behind the solution.

If we consider microservices, there are usually interaction, even if in a loose coupled configuration, between the differents part of an application (a.k.a microservices). At some point, to have a secured application, we would like to have each service proving who it said it is, which is the process of authentication. And to have a working environment, the authenticated service should have authorizations defined.

![illustration1](/assets/workloadid/workloadidentityauthsvc.png)

Now let's consider Azure. 
In the Azure plane, we have a way to authenticate and authorize a service to do things in Azure, or in an Azure AD integrated Azure service with managed Identity.

![illustration2](/assets/workloadid/workloadidentityuai.png)

And it works fine. No creds to manage, so in theory no risk of finding a connection string or a secret hardcoded in the application code.

But what happen if we consider containers and as such AKS?

Well, first, we must remember that container are, by design, isolated processes, or services running on a shared kernel. Even if the isolation is less complete than a full virtual OS (but also les heavy and thus way more fast), there is still an isolation, and the Azure platform cannot easily see what's happenind inside a container, or in AKS case a pod.

The first answer to this property of the container / pod which was in this case an issue was something called pod identity.
With pod identity, we introduced in AKS a corresponding objet in the Kubernetes plane to an Azure managed identity in the Azure plane.

![illustration3](/assets/workloadid/workloadidentitypodid.png)

A custom resource definition allowed us to  create a managed identity object in the kubernetes plane and was associated to a pod through a managed identity binding. You know, kind of the same way as for services and labels, but for identities.
A daemonset ensured that a pod was running at all time on each node to intercept the Authentication request, and talked with another set of pods organized in a deployement that itself was checking Azure AD for authentication, and authorization.

While looking elegant in terms of Kubernetes approach (you know, because everybody does CRDs ^^), the underlying architecture was limited in many aspects, and it was not developed to a v2 version.

Instead, pod identity v2 was redevelopped, and called workload identity. 

## 2. Workload Identity concepts

Now that the little history is reminded, let's focus on how workload identity works.
A better explanation should be to look at the scope of workload identity.
Outside of a pure Kubernetes scope (Yup, not only AKS, which was one of the limit of pod identity v1 btw), workload identity is defined at the Azure Active Directory level.
There is a dedicated section on the [Microsoft Entra documentation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/) about it.
It describes the different kind of eligible principals in Azure AD that can be used:

- Managed Identities
- Application registrations

And also the concept of workload identity federation. 

That's right, this is really about federating an Identity provider and Azure AD through either an application registration or a managed identity.
Obviously managed identities only work in an Azure context, and non-Azure environment must relies on Application registration.
But here's the interesting part.
By establishing a workload identity federation, we configure the Azure AD security principals to trust token issued from an external IdP.
The beauty of it? 
No need to share a secret or a certificate for external system & application registration scenario.
As described roughly on the schema below, an application, called a workload in this case, check with a local idp to get a token.
This token is known only from the local idp though, and the workload then send this token to the little part of AAD that is federated which is the security principal, an app reg or a managed identity. However, since this identity is federated with the local idp, it checks this local idp for verification of the token. 
If said token is true, then we can move on the authorization part, in which we rely on the role assignment associated to the AAD security principal. That means the security principal in AAD should have the corresponding authorization of the said workload. If not, when sending back the request access token to the workload, it wouldn't grant any permissions. The nice part is the granularity of this federation, scope on one security principal only.

![illustration5](/assets/workloadid/workloadidentityfederation.png)

That's it for the concepts on workload identity. Now let's translate that in an AKS (and really a kubernetes) point of view.

## 3. Workload Identity in AKS

### 3.1. Overview of Workload Identity in AKS

If we take the previous schema in a Kubernetes context, we roughly get this:

![illustration6](/assets/workloadid/workloadidentityfederationk8s.png)

Kubernetes comes with an OIDC url since version `1.20`. Which means that kubetnetes is the de-facto oidc provider acting kind of like the we are refering to in our schema.
Then we have the workload in itself, which is composed of a pod (or rather pods in any kind of desired controller, but let's keep it simple for now ^^) and a service account associated to this pod.
The service account is the part that get the token and that will allow the request token in Azure AD to be used for access required by the pod.
On the overview, it's way more simple (and way more elegant, even for CRDs lovers) than the previous pod identity. Indeed, only basic kubernetes objects are required.

### 3.2. Component of workload identity in AKS

From our schema, we know the main component for workload identity.

In Azure, because we want to use managed identity on AKS:

- A managed identity with federated credential

For other kubernetes solution, we would use an application registration.

In Kubernetes,

- a service account
- the cluster oidc url

The managed identity need to have federated credentials. Looking on the terraform provider we have apperently everyting that we need: 

- The managed Identity resource
- The federated identity credential


```bash

resource "azurerm_resource_group" "RgWorkloadIdentity" {
  name                                  = "rsg-workloadIdentity"
  location                              = "West Europe"
}

# User assigned Identity

resource "azurerm_user_assigned_identity" "WorkloadManagedIdentity" {
  location                              = azurerm_resource_group.RgWorkloadIdentity.location
  name                                  = "WorkloadIdentityUAI"
  resource_group_name                   = azurerm_resource_group.RgWorkloadIdentity.name
}

# Federated Identity Creds, required for Workload Identity

resource "azurerm_federated_identity_credential" "UaiCsiFederated" {
  name                                  = "kubernetesfederatedcredsdemo"
  resource_group_name                   = azurerm_resource_group.RgWorkloadIdentity.name
  audience                              = ["api://AzureADTokenExchange"]
  issuer                                = <aks_oidc_url>
  parent_id                             = "azurerm_user_assigned_identity.WorkloadManagedIdentity.id"
  subject                               = "system:serviceaccount:${var.SANamespace}:${var.SAName}"
}

```

It's not that difficult to create an Identity as shown in the sample above. The federated credential for the identity however requires a bit of planning.
As per [Microsoft Entra Workload Identities documentation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp), the `issuer` and `subject` are the informations that establish the relationship with the external provider.

In AKS case (or other kubernetes cluster for that matter), the oidc issuer url is the issuer value, since it represents the external identity provider, while the subject is a kubernetes service account, referenced with the namespace in which it is located `system:serviceaccount:<service_account_namespace>:<service_account_name>`

```bash

yumemaru@azure$ az aks list | jq .[].oidcIssuerProfile
{
  "enabled": true,
  "issuerUrl": "https://eastus.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/"
}


```

The `audience` recomended value is `api://AzureADTokenExchange`. One would note that while referenced as `audience`, we have a list defined as the value in our sample. Again the Entra documentation gives highligh, since it reference not `audience` but `audiences` and define it as the list of audiences that can appear in the external token.
In our sample, because it comes from the terraform provider, we also have `parent_id` which match the Azure AD principal on which the federated credentials are established.


Now there are other things under the hood that makes everything works. Checking at the cluster, we can see that there is a deployment refering to workload identity in the `kube-system` namespace:

```bash

yumemaru@azure$ k get deployments.apps -n kube-system | grep -i wi
azure-wi-webhook-controller-manager   2/2     2            2           3d12h

yumemaru@azure$ k get secret -n kube-system | grep -i wi
azure-wi-webhook-server-cert       Opaque                          3      38d### 3.3. Using workload Identity

yumemaru@azure$ k describe pod -n kube-system azure-wi-webhook-controller-manager-6989db6bdc-ltdfx
Name:                 azure-wi-webhook-controller-manager-6989db6bdc-ltdfx
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Service Account:      azure-wi-webhook-admin
Node:                 aks-aksnp0sbxcon-29325118-vmss000017/172.16.0.5
Start Time:           Fri, 14 Apr 2023 21:06:28 +0200
Labels:               azure-workload-identity.io/system=true
                      kubernetes.azure.com/managedby=aks
                      pod-template-hash=6989db6bdc
Annotations:          cni.projectcalico.org/containerID: 8ba5785b05de87c4f4301be0ac9d89505487416b74b399bc19075ff9853b77f3
                      cni.projectcalico.org/podIP: 10.244.2.13/32
                      cni.projectcalico.org/podIPs: 10.244.2.13/32
                      container.seccomp.security.alpha.kubernetes.io/manager: runtime/default
Status:               Running
IP:                   10.244.2.13
IPs:
  IP:           10.244.2.13
Controlled By:  ReplicaSet/azure-wi-webhook-controller-manager-6989db6bdc
Containers:
  manager:
    Container ID:  containerd://f241ea66714aacd1edc53532874cdead2c4a7c84babc4d1cdf6b05057b89d344
    Image:         mcr.microsoft.com/oss/azure/workload-identity/webhook:v1.0.0
    Image ID:      mcr.microsoft.com/oss/azure/workload-identity/webhook@sha256:6ccd2cdf34a428f0a714b94c32cf4b366d98caa1ae56f5943457ac9fe4cdbd38
    Ports:         9443/TCP, 9440/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /manager
    Args:
      -log-level=info
      -disable-cert-rotation=true
      -webhook-cert-dir=/certs
    State:          Running
      Started:      Fri, 14 Apr 2023 21:07:26 +0200
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     200m
      memory:  300Mi
    Requests:
      cpu:      100m
      memory:   20Mi
    Liveness:   http-get http://:healthz/healthz delay=15s timeout=1s period=20s #success=1 #failure=6
    Readiness:  http-get http://:healthz/readyz delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:
      KUBERNETES_PORT_443_TCP_ADDR:  akssbxcontainer-4vz01e6k.hcp.eastus.azmk8s.io
      KUBERNETES_PORT:               tcp://akssbxcontainer-4vz01e6k.hcp.eastus.azmk8s.io:443
      KUBERNETES_PORT_443_TCP:       tcp://akssbxcontainer-4vz01e6k.hcp.eastus.azmk8s.io:443
      KUBERNETES_SERVICE_HOST:       akssbxcontainer-4vz01e6k.hcp.eastus.azmk8s.io
      POD_NAMESPACE:                 kube-system (v1:metadata.namespace)
      AZURE_ENVIRONMENT:             AZUREPUBLICCLOUD
      AZURE_TENANT_ID:               00000000-0000-0000-0000-000000000000
    Mounts:
      /certs from cert (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gpfgq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  cert:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  azure-wi-webhook-server-cert
    Optional:    false
  kube-api-access-gpfgq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 CriticalAddonsOnly op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>


yumemaru@azure$ k get pod -n kube-system | grep -i wi
azure-wi-webhook-controller-manager-6989db6bdc-hsk4d   1/1     Running   0              108m
azure-wi-webhook-controller-manager-6989db6bdc-ltdfx   1/1     Running   0              108m

```

This deployment is the webhook that get the token from AAD and feed it to the pods requiring to authenticate through the mutated webhook referenced in the workload id documentation.

And now we can try this out.

### 3.3. Using workload identity with Key Vault CSI Secret store

To get a pragmatic understanding of workload identity, we will try it out on the Azure Key Vault CSI Secret provider.

There's a reason for that. First, using the secret provider, we can avoid using kubernetes secret and mount in the workload the secret directly as CSI volume.
Second, the secret provider is quite easy to use and does not require us to use samples of authenticating library. Useful when you're on the infra side rather than the dev side.

As a reminder, the Key Vault CSI provider is available as an AKS add-on, which ease really the installation. By default however, it deploys a managed identity which is associated to the cluster and is not as granular as we would like, hence the use of workload identity instead of the simple managed identity.

First thing first, we need the CRD which allows us to define our secret store in Kubernetes: 

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: ${SecretProviderClassName}
  namespace: ${SecretProvider_Namespace}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"               
    clientID: ${UAIClientId}
    keyvaultName: ${KVName}
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: ${SecretName}
          objectAlias: ${SecretName}            
          objectType: secret                    
          objectVersion: ${SecretVersion}       
    tenantId: ${TenantId} 

```

There's not much in terms of configuration change. Instead of using a `useVMManagedIdentity` and `userAssignedIdentityID`, we specify a `clientID` in which we specify the Managed Identity with federated credential. And that's about all on the Secret store side.
Remember, we talk about a service account when we created the fedrated credential. That's why we need to define a service account.

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${userAssignedClientId}
  labels:
    azure.workload.identity/use: "true"
  name: ${serviceAccountName}
  namespace: ${serviceAccountNameSpace}

```

The name of our namespace **must** be the same defined in the ferated credential. We can check our identity and the federated credential as follow.
In this sample, the namespace is `nsworkloadidentitydemo` and the service account is `saworkloadidentitydemo`.
Note the label `azure.workload.identity/use: "true"` which validate that the service account is used for worklad identity.
Additional annotations and labels information are available on the github doc



```bash

yumemaru@azure$ az identity federated-credential list --identity-name uaicsisbxcontainer -g rsg-sbxcontainer | jq .
[
  {
    "audiences": [
      "api://AzureADTokenExchange"
    ],
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-sbxcontainer/providers/Microsoft.ManagedIdentity/userAssignedIdentities/uaicsisbxcontainer/federatedIdentityCredentials/kubernetesfederatedcredsdemo",
    "issuer": "https://eastus.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/",
    "name": "kubernetesfederatedcredsdemo",
    "resourceGroup": "rsg-sbxcontainer",
    "subject": "system:serviceaccount:nsworkloadidentitydemo:saworkloadidentitydemo",
    "systemData": null,
    "type": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
  }
]

```

Last but not least, the pods which we want to grant access to our key vault need the service account configured. Once deployed, the token is projected in the pods:

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: workloadidentitydemo
  name: ${deploymentName}
  namespace: ${nameSpace}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workloadidentitydemo
  strategy: {}
  template:
    metadata:
      labels:
        app: workloadidentitydemo
    spec:
      serviceAccount: ${workloadIdentitySA}
      volumes:
      - name: csisecret
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: ${SecretProviderClassName}
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
          - mountPath: /mnt/secrets-store
            name: csisecret
            readOnly: true
        resources: {}
status: {}

```

When the pods are scheduled, we can see first the volume from the CSI provider, and second the projected volume from workload identity which bring the required token: 



```bash

yumemaru@azure$ k get pod -n nsworkloadidentitydemo 
NAME                                       READY   STATUS    RESTARTS   AGE
deployment-kvcsiaks7w2k-64b7cb46c7-4mn66   1/1     Running   0          40h
deployment-kvcsiaks7w2k-64b7cb46c7-dwndt   1/1     Running   0          40h
deployment-kvcsiaks7w2k-64b7cb46c7-lmphn   1/1     Running   0          40h

yumemaru@azure$ k describe pod -n nsworkloadidentitydemo deployment-kvcsiaks7w2k-64b7cb46c7-4mn66 
Name:             deployment-kvcsiaks7w2k-64b7cb46c7-4mn66
Namespace:        nsworkloadidentitydemo
Priority:         0
Service Account:  saworkloadidentitydemo
Node:             aks-npwkid-32253634-vmss000008/172.16.0.7
Start Time:       Sat, 15 Apr 2023 21:31:09 +0200
Labels:           app=workloadidentitydemo
                  pod-template-hash=64b7cb46c7
Annotations:      cni.projectcalico.org/containerID: 16a5757c999e907a85b591fac6a6647bc67fd315a1cfe4b0420896bd7ba865a6
                  cni.projectcalico.org/podIP: 10.244.0.9/32
                  cni.projectcalico.org/podIPs: 10.244.0.9/32
Status:           Running
IP:               10.244.0.9
IPs:
  IP:           10.244.0.9
Controlled By:  ReplicaSet/deployment-kvcsiaks7w2k-64b7cb46c7
Containers:
  nginx:
    Container ID:   containerd://7f2f56c0ec27eaf08f69d149aad6d4fc71940c9fea9a45bd7e7941f309981e4c
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:63b44e8ddb83d5dd8020327c1f40436e37a6fffd3ef2498a6204df23be6e7e94
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 15 Apr 2023 21:32:24 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /mnt/secrets-store from csisecret (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w87lg (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  csisecret:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            secrets-store.csi.k8s.io
    FSType:            
    ReadOnly:          true
    VolumeAttributes:      secretProviderClass=kvcsiaks7w2k
  kube-api-access-w87lg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
yumemaru@azure$ k get pod -n nsworkloadidentitydemo deployment-kvcsiaks7w2k-64b7cb46c7-4mn66 -o json | jq .spec.containers[0].volumeMounts
[
  {
    "mountPath": "/mnt/secrets-store",
    "name": "csisecret",
    "readOnly": true
  },
  {
    "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
    "name": "kube-api-access-w87lg",
    "readOnly": true
  }
]

```

The secret defined in the secret store is available on the pods, with a grnaular access granted with the managed identity on the Azure side and the service account on the kubernetes side:

```bash

yumemaru@azure$ k exec -n nsworkloadidentitydemo deployment-kvcsiaks7w2k-64b7cb46c7-4mn66 -- ls /mnt/secrets-store
workloadidentitydemo

yumemaru@azure$ k exec -n nsworkloadidentitydemo deployment-kvcsiaks7w2k-64b7cb46c7-4mn66 -- cat /mnt/secrets-store/workloadidentitydemo
Thisisasecretdemo

```

## To conclude

Following the pod identity v1 project, Workload Identity is going further in the capabilitiy of being granular on the pod authorization.
The configuration on the kubernetes side is more classical with only the use of service account, while the configuration on the Azure side takes 2 steps, the managed identity (or app registration) and the federated credential.
Because it's finally an Azure AD feature, all the Azure AD security is available with conditional access or access review which allows again more control, not only on the kubernetes / Azure side but definitely on the Azure AD Idp side.
