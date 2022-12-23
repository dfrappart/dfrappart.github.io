---
layout: post
title:  "Please Azure Arc Give me a Kubernetes Cluster"
date:   2022-12-28 09:00:00 +0200
categories: AKS, Azure Arc
---

Hello there!

It's been longer than planned ^^, sorry about that.
So today, I propose a look at Azure Arc Enabled Kubernetes.
I wanted to have a look at It since, well, forever, but always lacked time.
Now we are here at last.

The agenda:

1. An overview of Azure Arc and the architecture regarding Arc Enabled Kubernetes.
2. the on boarding process.
3. Arc-enabled Kubernetes from the Azure Ops Point of View​.

There are really a lot of others stuff to discuss, but for a first article, It will definitely be enough.

## 1. Overview of Azure Arc Enabled Kubernetes architecture

As the title implies, let's start with a little bit of architecture.
Even before that, let's talk about Azure Arc.

Azure Arc is not only one product, but more of a family of product.
In this family, one aim: to ease hybridation and specifically operations in multi-cloud environment.
Known members of the family are

- Azure Arc Enabled Server
- Azure Arc Enabled Kubernetes on which this article is about
- Azure Arc enabled Data Services

and others.

![Illustration 1](/assets/arck8s/arck8s001.png)

Let's consider a Kubernetes cluster. This cluster could live anywhere, that's not relevant for now.
When we transform this cluster to an Arc Enabled Kubernetes cluster, a logical object, representing It in Azure, is created.

On the Cluster actual location, the cluster can be managed "as usual", with native tools, such as `kubectl`, `helm` or even with gitOps tooling.
But because It exists in the Azure plane, It can also be managed in some ways in Azure, with Azure tools.

![Illustration 3](/assets/arck8s/arck8s003.png)  
  
Now we mentioned a little bit of architecture, so how does It work inside this Arc Enabled cluster?

Very simply, there is an agent, **the Arc enabled Kubernetes Agent**, which pushes and pulls information to and from the Azure plane.

Once the cluster is onboarded, the agent becomes responsible of updating the state of the cluster against its state in Azure.
When an Azure Ops change the configuration on the Azure side, the agent get the desired state and, if required, pulls container images from Azure container registry.
  
![Illustration 4](/assets/arck8s/arck8s004.png)

the main advantage is this one way connection, only ever initiated from the Cluster side.
An Arc enabled Kubernetes cluster is never accessed from the Azure portal. On the other hand, It requires outbound connectivity to Azure.

The agent has different state in its lifecycle that are summarize in the table following:

| Status | Description |
|-|-|
| Connecting| The Azure Arc-enabled Kubernetes resource has been created in Azure, but the service hasn't received the agent heartbeat yet. |
| Connected | The Azure Arc-enabled Kubernetes service received an agent heartbeat within the previous 15 minutes.​ |
| Offline | The Azure Arc-enabled Kubernetes resource was previously connected, but the service hasn't received any agent heartbeat for 15 minutes.​ |
| Expired | The managed identity certificate of the cluster has expired. In this state, Azure Arc features will no longer work on the cluster. For more information on how to address expired Azure Arc-enabled Kubernetes resources.​ |

And that's about all for the architecture concepts.
There's more regarding the outbound traffic, but we'll detail that in the onboarding process.

## 2. On Boarding Kubernetes to Azure Arc

As discussed, there is really one element only in the Arc Architecture for Kubernetes Cluster, and this is the Azure Arc Enabled Cluster Agent.

To deploy this agent and initiate the on boarding process, we need the following prerequisites:

- the `kubectl` cli installed
- on the server to on board, cluster-admin access
- az cli with `connectedk8s` extension
- helm
- and outbound connectivity

Also, not specified on the documentation, if the cluster to onboard is an EKS or a GKE, it's useful to have aws cli or gcloud cli.
Similarly to the way we get credentials with az cli for AKS cluster, we can use those other clouds cli to get the creds.

About the `kubectl` cli, It should be configured so that the config file point to the cluster to onboard.

About the az cli, an authenticated session is required, with the extension `connectedk8s` for the onboarding.

About the outbound connectivity to Azure, because there is no easy way to allow only IPs to be reach, there is a list of fqdns required to allow the connectivity of the cluster:

| Endpoint | Description |
|-|-|
| https://management.azure.com (for Azure Cloud), https://management.usgovcloudapi.net (for Azure US Government)| Required for the agent to connect to Azure and register the cluster.​ |
| https://<region>.dp.kubernetesconfiguration.azure.com (for Azure Cloud), https://<region>.dp.kubernetesconfiguration.azure.us (for Azure US Government) | Data plane endpoint for the agent to push status and fetch configuration information.​​ |
| https://login.microsoftonline.com, https://<region>.login.microsoft.com, login.windows.net (for Azure Cloud), https://login.microsoftonline.us, <region>.login.microsoftonline.us (for Azure US Government)​ | Required to fetch and update Azure Resource Manager tokens.​​ |
| https://mcr.microsoft.com, https://*.data.mcr.microsoft.com​ | Required to pull container images for Azure Arc agents.​​ |
| https://gbl.his.arc.azure.com (for Azure Cloud), https://gbl.his.arc.azure.us (for Azure US Government) | Required to get the regional endpoint for pulling system-assigned Managed Identity certificates.​ |
| https://*.his.arc.azure.com (for Azure Cloud), https://usgv.his.arc.azure.us (for Azure US Government) | Required to pull system-assigned Managed Identity certificates.​ |
| https://k8connecthelm.azureedge.net​ | az connectedk8s connect uses Helm 3 to deploy Azure Arc agents on the Kubernetes cluster. This endpoint is needed for Helm client download to facilitate deployment of the agent helm chart.​ |
| guestnotificationservice.azure.com, *.guestnotificationservice.azure.com, sts.windows.net, https://k8sconnectcsp.azureedge.net​ | For Cluster Connect and for Custom Location based scenarios.​ |
| *.servicebus.windows.net​ | For Cluster Connect and for Custom Location based scenarios.​ |

Before onboarding a cluster, there is also a list of supported kubernetes distributions.
This list, available on the [Azure Arc documentation](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/validation-program), includes AWS EKS but also Google GKE clusters.

Also, it's not because a cluster is not on the supported list that it's not eligible to onboarding.
But as usual things may or may not work, and in this case, it means that we would be on our own.
We'll have a look at that later.

Ok now, let's onboard a few clusters.
Note that to test this, the [Azure Arc Jumpstart](https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/) provides a very useful samples configuration to help people create sandboxes on other clouds.

To test the onboarding,we have a bunch of Kubernetes cluster:

- A GKE cluster
- A EKS cluster
- A Microk8s cluster

Also, I have a local Minikube that I already onboarded and that we may also have a look at.

Let's start with the GKE cluster.

### 2.1. Onboarding a GKE cluster  
  
![Illustration 5](/assets/arck8s/arck8s005.png)

As mentionned, it's easier to get the kubeconfig file if we have the gcloud cli configured

```bash

gcloud container clusters list
NAME  LOCATION  MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION    NUM_NODES  STATUS
gke1  us-west1  1.24.5-gke.600  34.168.91.163  e2-medium     1.24.5-gke.600  2          PROVISIONING

gcloud container clusters get-credentials gke1 --region us-west1
Fetching cluster endpoint and auth data.
kubeconfig entry generated for gke1.

k config get-contexts
CURRENT   NAME                                    CLUSTER                                 AUTHINFO                                          NAMESPACE
*         gke_terraformgcptesting_us-west1_gke1   gke_terraformgcptesting_us-west1_gke1   gke_terraformgcptesting_us-west1_gke1 

```

Now that the config is ok, we can initiate the onboarding:

```bash

df@df2204lts:~/Documents/clonedrepo/azure_arc/azure_arc_k8s_jumpstart/gke/terraform$ az connectedk8s connect -n gke1 -g arcdemo
This operation might take a while...

{
  "agentPublicKeyCertificate": "",
  "agentVersion": null,
  "connectivityStatus": "Connecting",
  "distribution": "gke",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/arcdemo/providers/Microsoft.Kubernetes/connectedClusters/gke1",
  "identity": {
    "principalId": "00000000-0000-0000-0000-000000000000",
    "tenantId": "00000000-0000-0000-0000-000000000000",
    "type": "SystemAssigned"
  },
  "infrastructure": "gcp",
  "kubernetesVersion": null,
  "lastConnectivityTime": null,
  "location": "eastus",
  "managedIdentityCertificateExpirationTime": null,
  "name": "gke1",
  "offering": null,
  "provisioningState": "Succeeded",
  "resourceGroup": "arcdemo",
  "systemData": {
    "createdAt": "2022-12-13T10:38:29.776311+00:00",
    "createdBy": "david@teknews.cloud",
    "createdByType": "User",
    "lastModifiedAt": "2022-12-13T10:38:29.776311+00:00",
    "lastModifiedBy": "david@teknews.cloud",
    "lastModifiedByType": "User"
  },
  "tags": {},
  "totalCoreCount": null,
  "totalNodeCount": null,
  "type": "microsoft.kubernetes/connectedclusters"
}

```

The cluster is now onboarded.  
  
## 2.2. A more through look at the connected cluster  
  
Ok the cluster is onoarded so checking on the Azure portal, we can see the connected kubernetes:
  
![Illustration 6](/assets/arck8s/arck8s006.png)

```bash

df@df2204lts:~/Documents/dfrappart.github.io$ az connectedk8s list | jq .[2].name
"gke1"
df@df2204lts:~/Documents/dfrappart.github.io$ az connectedk8s list | jq .[2]
{
  "agentPublicKeyCertificate": "",
  "agentVersion": "1.8.14",
  "azureHybridBenefit": "NotApplicable",
  "connectivityStatus": "Connected",
  "distribution": "gke",
  "distributionVersion": null,
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/arcdemo/providers/Microsoft.Kubernetes/connectedClusters/gke1",
  "identity": {
    "principalId": "00000000-0000-0000-0000-000000000000",
    "tenantId": "00000000-0000-0000-0000-000000000000",
    "type": "SystemAssigned"
  },
  "infrastructure": "gcp",
  "kubernetesVersion": "1.24.5-gke.600",
  "lastConnectivityTime": "2022-12-13T13:59:37.472000+00:00",
  "location": "eastus",
  "managedIdentityCertificateExpirationTime": "2023-03-13T10:33:00+00:00",
  "miscellaneousProperties": null,
  "name": "gke1",
  "offering": null,
  "privateLinkScopeResourceId": null,
  "privateLinkState": "Disabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "arcdemo",
  "systemData": {
    "createdAt": "2022-12-13T10:38:29.776311+00:00",
    "createdBy": "david@teknews.cloud",
    "createdByType": "User",
    "lastModifiedAt": "2022-12-13T14:15:06.703912+00:00",
    "lastModifiedBy": "64b12d6e-6549-484c-8cc6-6281839ba394",
    "lastModifiedByType": "Application"
  },
  "tags": {},
  "totalCoreCount": 6,
  "totalNodeCount": 3,
  "type": "microsoft.kubernetes/connectedclusters"
}

```
Under the hood, the `az connectedk8s connect` is actually deploying the agent through helm.
Using the `helm list` command, we can see the deployed chart:
We can see the `azure-arc` release and also the `azurepolicy` release.

```bash

df@df2204lts:~/Documents/dfrappart.github.io$ helm list -A
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                           APP VERSION
azure-arc       default         1               2022-12-13 11:39:07.233181548 +0100 CET deployed        azure-arc-k8sagents-1.8.14                      1.0        
azurepolicy     kube-system     2               2022-12-13 10:54:30.763188969 +0000 UTC deployed        azure-policy-extension-arc-clusters-1.4.0       1      

```

Funny thing, the release is in the default namespace, but if we look at the details of the release we can see that the target namespace is not the default namespace:

```yaml


# Source: azure-arc-k8sagents/templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: azure-arc
  labels:
    control-plane: "true"
    admission.policy.azure.com/ignore: "true"


```

Knowing that, we can have a look in the `azure-arc` namespace:

```bash

df@df2204lts:~/Documents/dfrappart.github.io$ k get all -n azure-arc 
NAME                                            READY   STATUS    RESTARTS   AGE
pod/cluster-metadata-operator-79f6ff784-s6c5k   2/2     Running   0          27m
pod/clusterconnect-agent-5b888b7dd-5g98q        3/3     Running   0          27m
pod/clusteridentityoperator-5f5bd96dbf-mddcc    2/2     Running   0          27m
pod/config-agent-6fbd7cc7bb-vgjh5               2/2     Running   0          27m
pod/controller-manager-5b9bf8b674-t99bw         2/2     Running   0          27m
pod/extension-manager-7c8764b4dc-65254          2/2     Running   0          27m
pod/flux-logs-agent-74cc65d666-92nr2            1/1     Running   0          27m
pod/kube-aad-proxy-587dc6f549-lfhmg             2/2     Running   0          27m
pod/metrics-agent-b9db6f7db-8brgx               2/2     Running   0          27m
pod/resource-sync-agent-5c8f9bc979-bhqzh        2/2     Running   0          27m

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)            AGE
service/flux-logs-agent   ClusterIP   10.39.248.220   <none>        80/TCP             27m
service/kube-aad-proxy    ClusterIP   10.39.246.84    <none>        443/TCP,8080/TCP   27m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cluster-metadata-operator   1/1     1            1           27m
deployment.apps/clusterconnect-agent        1/1     1            1           27m
deployment.apps/clusteridentityoperator     1/1     1            1           27m
deployment.apps/config-agent                1/1     1            1           27m
deployment.apps/controller-manager          1/1     1            1           27m
deployment.apps/extension-manager           1/1     1            1           27m
deployment.apps/flux-logs-agent             1/1     1            1           27m
deployment.apps/kube-aad-proxy              1/1     1            1           27m
deployment.apps/metrics-agent               1/1     1            1           27m
deployment.apps/resource-sync-agent         1/1     1            1           27m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/cluster-metadata-operator-79f6ff784   1         1         1       27m
replicaset.apps/clusterconnect-agent-5b888b7dd        1         1         1       27m
replicaset.apps/clusteridentityoperator-5f5bd96dbf    1         1         1       27m
replicaset.apps/config-agent-6fbd7cc7bb               1         1         1       27m
replicaset.apps/controller-manager-5b9bf8b674         1         1         1       27m
replicaset.apps/extension-manager-7c8764b4dc          1         1         1       27m
replicaset.apps/flux-logs-agent-74cc65d666            1         1         1       27m
replicaset.apps/kube-aad-proxy-587dc6f549             1         1         1       27m
replicaset.apps/metrics-agent-b9db6f7db               1         1         1       27m
replicaset.apps/resource-sync-agent-5c8f9bc979        1         1         1       27m

```

Visible in the output are the deployments for clusterconnect-agent, but also other interesting part such as the kube-aad-proxy, which for instance relies on the OSS project [Guard](https://github.com/kubeguard/guard).

I did not find as much details in the documentation as I would like, but I guess It's Ok because the idea is that It is a managed deployment from the Azure Arc perspective. If you read throughly though, you should find (in the AKS documentation) that Guard logs are related to Azure Active Directory integration.

Ok that was all for the onboarding. Let's look another cluster to onboard.

### 2.3. Onboarding an EKS cluster

In this rather short part, we will carry on with the On boarding, with an EKS cluster.

The process is actually the same. However, there is an unexpected addition due to the nature of the EKS architecture.

Indeed, we can see the EC2 instances used by the EKS cluster as nodes on boarded as Arc enabled servers: 

![Illustration 7](/assets/arck8s/arck8s007.png)  

That's all for the on boarding part, let's have a look from an Azure Ops perspective.
  
## 3. What the Azure Ops can do

Once the cluster is on boarded, what can the ops can do?

Well first the clusters are visible on the Azure portal.
Also, It is possible to interact with those 

```bash


df@df2204lts:~/Documents/dfrappart.github.io$ k get all -n gatekeeper-system 
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-audit-5c55fc4ddf-h5sfw                1/1     Running   0          33m
pod/gatekeeper-controller-manager-59c95b76c4-9dc4c   1/1     Running   0          33m
pod/gatekeeper-controller-manager-59c95b76c4-vc6q6   1/1     Running   0          33m

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   10.39.244.191   <none>        443/TCP   33m

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-audit                1/1     1            1           33m
deployment.apps/gatekeeper-controller-manager   2/2     2            2           33m

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/gatekeeper-audit-5c55fc4ddf                1         1         1       33m
replicaset.apps/gatekeeper-controller-manager-59c95b76c4   2         2         2       33m


```

azure monitor extension

```bash


df@df2204lts:~$ az k8s-extension create --name azuremonitor-containers --cluster-name gke1 --cluster-type connectedClusters --resource-group arcdemo --extension-type Microsoft.AzureMonitor.Containers --configuration-settings logAnalyticsWorkspaceResourceID='/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-fr-poc-taflog/providers/Microsoft.OperationalInsights/workspaces/law-fr-poc-taflog16e85b36'
Ignoring name, release-namespace and scope parameters since microsoft.azuremonitor.containers only supports cluster scope and single instance of this extension.
Defaulting to extension name 'azuremonitor-containers' and release-namespace 'azuremonitor-containers'
{
  "aksAssignedIdentity": null,
  "autoUpgradeMinorVersion": true,
  "configurationProtectedSettings": {
    "amalogs.secret.key": "",
    "amalogs.secret.wsid": "",
    "omsagent.secret.key": "",
    "omsagent.secret.wsid": ""
  },
  "configurationSettings": {
    "logAnalyticsWorkspaceResourceID": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-fr-poc-taflog/providers/Microsoft.OperationalInsights/workspaces/law-fr-poc-taflog16e85b36"
  },
  "customLocationSettings": null,
  "errorInfo": null,
  "extensionType": "microsoft.azuremonitor.containers",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/arcdemo/providers/Microsoft.Kubernetes/connectedClusters/gke1/providers/Microsoft.KubernetesConfiguration/extensions/azuremonitor-containers",
  "identity": {
    "principalId": "77f8d034-d0c4-4e8d-bd1e-e7b4ed81b22d",
    "tenantId": null,
    "type": "SystemAssigned"
  },
  "installedVersion": null,
  "name": "azuremonitor-containers",
  "packageUri": null,
  "provisioningState": "Succeeded",
  "releaseTrain": "Stable",
  "resourceGroup": "arcdemo",
  "scope": {
    "cluster": {
      "releaseNamespace": "azuremonitor-containers"
    },
    "namespace": null
  },
  "statuses": [],
  "systemData": {
    "createdAt": "2022-12-13T12:50:25.023953+00:00",
    "createdBy": null,
    "createdByType": null,
    "lastModifiedAt": "2022-12-13T12:50:25.023953+00:00",
    "lastModifiedBy": null,
    "lastModifiedByType": null
  },
  "type": "Microsoft.KubernetesConfiguration/extensions",
  "version": "3.0.0"
}

```

defender

```bash

df@df2204lts:~$ az k8s-extension create --name microsoft.azuredefender.kubernetes --cluster-name gke1 --cluster-type connectedClusters --resource-group arcdemo --extension-type microsoft.azuredefender.kubernetes
Ignoring name, release-namespace and scope parameters since microsoft.azuredefender.kubernetes only supports cluster scope and single instance of this extension.
Defaulting to extension name 'microsoft.azuredefender.kubernetes' and release-namespace 'mdc'
{
  "aksAssignedIdentity": null,
  "autoUpgradeMinorVersion": true,
  "configurationProtectedSettings": {
    "amalogs.secret.key": "",
    "amalogs.secret.wsid": "",
    "omsagent.secret.key": "",
    "omsagent.secret.wsid": ""
  },
  "configurationSettings": {
    "logAnalyticsWorkspaceResourceID": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/defaultresourcegroup-eus/providers/microsoft.operationalinsights/workspaces/defaultworkspace-00000000-0000-0000-0000-000000000000-eus"
  },
  "customLocationSettings": null,
  "errorInfo": null,
  "extensionType": "microsoft.azuredefender.kubernetes",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/arcdemo/providers/Microsoft.Kubernetes/connectedClusters/gke1/providers/Microsoft.KubernetesConfiguration/extensions/microsoft.azuredefender.kubernetes",
  "identity": {
    "principalId": "6aee9724-4d59-48a0-b077-ac831f29e303",
    "tenantId": null,
    "type": "SystemAssigned"
  },
  "installedVersion": null,
  "name": "microsoft.azuredefender.kubernetes",
  "packageUri": null,
  "provisioningState": "Succeeded",
  "releaseTrain": "Stable",
  "resourceGroup": "arcdemo",
  "scope": {
    "cluster": {
      "releaseNamespace": "mdc"
    },
    "namespace": null
  },
  "statuses": [],
  "systemData": {
    "createdAt": "2022-12-13T13:07:52.219160+00:00",
    "createdBy": null,
    "createdByType": null,
    "lastModifiedAt": "2022-12-13T13:07:52.219160+00:00",
    "lastModifiedBy": null,
    "lastModifiedByType": null
  },
  "type": "Microsoft.KubernetesConfiguration/extensions",
  "version": "0.7.8"
}

```

```bash


df@df2204lts:~$ k create sa arcdemo
W1213 14:44:40.896342   16151 gcp.go:119] WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
To learn more, consult https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
serviceaccount/arcdemo created
df@df2204lts:~$ k create clusterrolebinding arcdemo-binding --clusterrole cluster-admin --serviceaccount default:demo
W1213 14:45:42.582469   16386 gcp.go:119] WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
To learn more, consult https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
clusterrolebinding.rbac.authorization.k8s.io/arcdemo-binding created
df@df2204lts:~$ k create clusterrolebinding arcdemo-binding --clusterrole cluster-admin --serviceaccount default:arcdemo
W1213 14:55:52.453977   19650 gcp.go:119] WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
To learn more, consult https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
clusterrolebinding.rbac.authorization.k8s.io/arcdemo-binding created
df@df2204lts:~$

```

### 2.3. Onboarding a self managed K8S, or what does It means to be a supported distribution

So, this time we will have a look at a microk8s cluster.
The process is **exactly** the same as for the others Kubernetes distribution.
The distribution is however not officially supported.

It does not mean It's not working. But there might be some limits.

for example, when we try to add extension, from time to time it fails, with the following message:

```json

{
  "code":"DeploymentFailed",
  "message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/DeployOperations for usage details.",
  "details":
    [
      {
        "code":"ExtensionOperationFailed",
        "message":"The extension operation failed with the following error:  Error: {Helm installation from path [] for release [azurepolicy] failed with the following error: err [unable to build kubernetes objects from release manifest: resource mapping not found for name: \"gatekeeper-admin\" namespace: \"\" from \"\": no matches for kind \"PodSecurityPolicy\" in version \"policy/v1beta1\"\nensure CRDs are installed first]} occurred while doing the operation : {Installing the extension} on the config."
      }
    ]
}

```

The message is quite clear. The deployment failed because there is no match for the object `PodSecurityPolicy`. That's because the self managed Kubernetes is in a version higher than the supported ones. 
Because all the extensions are installed through a helm wrapper aka the az cli command, there is no customization possible.
So my 2 cents about self managed Kubernetes is that the version should match versions on officially supported Arc Kubernetes.
That's all for the on boarding

