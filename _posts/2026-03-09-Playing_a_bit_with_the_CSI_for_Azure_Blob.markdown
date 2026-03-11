---
layout: post
title:  "Playing a bit with the CSI for Azure Blob"
date:   2026-03-09 18:00:00 +0200
year: 2026
categories: Kubernetes AKS
---

Hi!

Recently, I was asked about the use of Azure storage blob from kubernetes pods.
Because I was not able to answer everything, I had to look for some answers.

This article is the summary of my trials on this.

The agenda will be as follow:

- Azure CSI Blob basics
- Managing `pvc` with existing storage account
- Options on the CSI `StorageClass`

## 1. Azure CSI Blob basics

To get started, let's have a look on the storage classes available on an AKS cluster.

```bash

df@df-2404lts:~$ k get storageclasses.storage.k8s.io 
NAME                     PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azureblob-fuse-premium   blob.csi.azure.com   Delete          Immediate              true                   14m
azureblob-nfs-premium    blob.csi.azure.com   Delete          Immediate              true                   14m
azurefile                file.csi.azure.com   Delete          Immediate              true                   14m
azurefile-csi            file.csi.azure.com   Delete          Immediate              true                   14m
azurefile-csi-premium    file.csi.azure.com   Delete          Immediate              true                   14m
azurefile-premium        file.csi.azure.com   Delete          Immediate              true                   14m
default (default)        disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   14m
managed                  disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   14m
managed-csi              disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   14m
managed-csi-premium      disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   14m
managed-premium          disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   14m

```

These days, we have quite the choice for storage in Azure, and that's only built-in classes, there are other stuff that we can do.

The one that we are interested in are those relying on the `blob.csi.azure.com` provisioner.

```bash

df@df-2404lts:~$ k describe storageclasses.storage.k8s.io azureblob-fuse-premium 
Name:                  azureblob-fuse-premium
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           blob.csi.azure.com
Parameters:            skuName=Premium_LRS
AllowVolumeExpansion:  True
MountOptions:
  -o allow_other
  --file-cache-timeout-in-seconds=120
  --use-attr-cache=true
  --cancel-list-on-mount-seconds=10
  -o attr_timeout=120
  -o entry_timeout=120
  -o negative_timeout=120
  --log-level=LOG_WARNING
  --cache-size-mb=1000
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>
df@df-2404lts:~$ k describe storageclasses.storage.k8s.io azureblob-nfs-premium 
Name:                  azureblob-nfs-premium
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           blob.csi.azure.com
Parameters:            protocol=nfs,skuName=Premium_LRS
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>

```

If we leverage volumes with NFS and a blob storage underneath, then we rely on the `azureblob-nfs-premium`. If we want to mount volume based on blob storage account, then it's the `azureblob-fuse-premium`.

There is a prerequisite though. The Csi Driver should be enabled on the cluster. We can easily check this.

```bash

df@df-2404lts:~$ az aks list | jq .[] | grep blob -A5 -B5
    "name": "Base",
    "tier": "Free"
  },
  "status": null,
  "storageProfile": {
    "blobCsiDriver": {
      "enabled": true
    },
    "diskCsiDriver": {
      "enabled": true,
      "version": "v1"


```

If the driver is not enabled, the az cli can help us fix this.

```bash

az aks update --enable-blob-driver --name $aks_cluster_name --resource-group $aks_resource_group_name

```

A very basic example, leveraging the `StorageClass` and the dynamic volume creation look like this.

```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blobdynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: azureblob-fuse-premium

```

We should have a `pvc` available in the default namespace.

```bash

df@df-2404lts:~$ k get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             VOLUMEATTRIBUTESCLASS   AGE
pvc-blobdynamic   Bound    pvc-2d5c36cf-88c2-4e3c-b408-561359b62f48   10Gi       RWX            azureblob-fuse-premium   <unset>                 15m

```

Notice that corresponding pv is also created.

```bash

df@df-2404lts:~$ k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS             VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-2d5c36cf-88c2-4e3c-b408-561359b62f48   10Gi       RWX            Delete           Bound    default/pvc-blobdynamic   azureblob-fuse-premium   <unset>                          14m

```

We also get a storage account in the managed resource group of the Aks cluster. We can notice the tag `k8s-azure-created-by`.

```bash

df@df-2404lts:~$ az storage account list -g rsg-aksobjectslab1 | jq .[0].name
"fusea145379910064399a65"
df@df-2404lts:~$ az storage account list -g rsg-aksobjectslab1 | jq .[0].tags
{
  "k8s-azure-created-by": "azure"
}

```

It's interesting to note the existence of a kubernetes secret, named after the storage account, and its data section which define the account name and access key.

```bash

df@df-2404lts:~$ k get secret
NAME                                                   TYPE     DATA   AGE
azure-storage-account-fusea145379910064399a65-secret   Opaque   2      42m

df@df-2404lts:~$ k get secret azure-storage-account-fusea145379910064399a65-secret -o custom-columns=Name:.metadata.name,Namespace:.metadata.namespace,StorageAccountName:.data.azurestorageaccountname,StorageAccountKey:.data.azurestorageaccountkey
Name                                                   Namespace   StorageAccountName                 StorageAccountKey
azure-storage-account-fusea145379910064399a65-secret   default     ZnVzZWExNDUzNzk5MTAwNjQzOTlhNjU=   --hidden--

df@df-2404lts:~$ k get secret azure-storage-account-fusea145379910064399a65-secret -o jsonpath='{.data.azurestorageaccountname}' | base64 -d
fusea145379910064399a65

df@df-2404lts:~$ k get secret azure-storage-account-fusea145379910064399a65-secret -o jsonpath='{.data.azurestorageaccountkey}' | base64 -d
--hidden-- 


```

We can verify the access key value with another az cli command.

```bash

df@df-2404lts:~$ az storage account keys list -n fusea145379910064399a65 -g rsg-aksobjectslab1 | jq .[0].value
"--hidden--"

```

So it means that through the `StorageClass`, we have a storage account automatically created, and the authentication managed through a kubernetes secret and the storage access key.

We can mount this volume on pods, using the usual volumes parameters.

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: dynamicblocpod
  name: dynamicblocpod
spec:
  volumes:
  - name: blobvol
    persistentVolumeClaim:
      claimName: pvc-blobdynamic
  containers:
  - image: nginx
    name: dynamicblocpod
    resources: {}
    volumeMounts:
    - mountPath: "/mnt/blob"
      name: blobvol
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

The volume is available in the `/mnt/blob` path. We can create a sample file.

```bash

df@df-2404lts:~/$ k exec dynamicblocpod -- touch /mnt/blob/file1.txt

df@df-2404lts:~/$ k exec dynamicblocpod -it -- sh
# echo "hello file" >> /mnt/blob/file1.txt        
# exit

```

And check the content on the Azure storage.

![illustration1](/assets/csiblob/csiblob001.png)

Ok, let's move on with other topics.


## 2. Managing `pvc` with existing storage account

In this section, we'll take a few existing storage account and check how we can use those in an AKS context.
It may happen that a blob storage account is already availalbe, fed with data outside of kubernetes. 
In this situation, we cannot rely on the dynamic `pvc` provisioning. We'll need to do things manually, and create the pv first

### 2.1. Mounting a blob storage account in a pod

To get started, we'll follow the sample on the Azure documentation for static volume.

We have in this first use case a simple blob storage account without any network restriction.

![illustration2](/assets/csiblob/csiblob002.png)

![illustration3](/assets/csiblob/csiblob003.png)

As we saw in the previous section, we need to generate a secret specifying the access key and the storage account name.

```bash

df@df-2404lts:~$ export staname="stak8s1"

df@df-2404lts:~$ export staaccesskey=$(az storage account keys list --account-name stak8s1 -g rsg-cluster-aks --query "[0].value" -o tsv)

df@df-2404lts:~$ kubectl create secret generic azure-secret-stak8s1 --from-literal azurestorageaccountname=$staname --from-literal azurestorageaccountkey=$staaccesskey --type=Opaque
secret/azure-secret-stak8s1 created

```

Then we can create the `pv` and `pvc` related to the storage.

```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: blob.csi.azure.com
  name: pv-blob
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-premium
  mountOptions:
    - -o allow_other
    - --file-cache-timeout-in-seconds=120
  csi:
    driver: blob.csi.azure.com
    volumeHandle: stak8scontainer1
    volumeAttributes:
      containerName: stak8scontainer1
    nodeStageSecretRef:
      name: azure-secret-stak8s1
      namespace: default
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-blob
  storageClassName: azureblob-fuse-premium

```

And lastly a sample pod configured with the volume.

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: blobpod
  name: blobpod
spec:
  volumes:
  - name: blobvol
    persistentVolumeClaim:
      claimName: pvc-blob
  containers:
  - image: nginx
    name: blobpod
    volumeMounts:
    - mountPath: "/mnt/blob"
      name: blobvol
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

If everything is working well, the pod should move the status Running.

```bash

df@df-2404lts:~$ k get pod
NAME      READY   STATUS              RESTARTS   AGE
blobpod   0/1     ContainerCreating   0          5s

df@df-2404lts:~$ k get pod
NAME      READY   STATUS    RESTARTS   AGE
blobpod   1/1     Running   0          7s

```

Notice that if the pod fails to mount the volume, it may be either the access key that could be wrong in the secret, or some network configuration blocking the pod access to the volume.

We can have a look in the next sections.


### 2.2. Adding network control on storage

To illustrate our purpose, we'll start by adding some restriction on the storage account.

We can configure the storage to allow only some IPs.

![illustration4](/assets/csiblob/csiblob004.png)

![illustration5](/assets/csiblob/csiblob005.png)

If we try to reach teh volume from the pod, it's not working anymore.

```bash

df@df-2404lts:~$ k exec blobpod -- ls /mnt/blob
ls: reading directory '/mnt/blob': Input/output error
command terminated with exit code 2

```

The pod is running from a known network, so if we allow specifically the subnet in the allowed network, it'll work again.

![illustration6](/assets/csiblob/csiblob006.png)

![illustration7](/assets/csiblob/csiblob007.png)

```bash

df@df-2404lts:~$ k exec blobpod -- ls /mnt/blob
df@df-2404lts:~$ k exec blobpod -- touch /mnt/blob/file1.txt
df@df-2404lts:~$ k exec blobpod -it -- sh
# ls /mnt
blob
# echo "write to file" >> /mnt/blob/file1.txt
# exit

```

After allowing access from our Public IP, we can check in the storage that it was indeed accessed.

![illustration9](/assets/csiblob/csiblob009.png)

Can we do more?

### 2.3. The case of the private storage account

Usually, we want ot leverage private endpoint on storage, because publicly exposing stuffs is considered bad, most of the time &#128540;.

So this time we have a storage account secured with a private endpoint.
If the private endpoint is confuigured correctly, along with the vnet from which we want to access it, everything will work smoothly.

As a reminder, we need to have a link on the Vnet to the private DNS zone used by the private endpoint.

![illustration10](/assets/csiblob/csiblob010.png)

![illustration11](/assets/csiblob/csiblob011.png)

![illustration12](/assets/csiblob/csiblob012.png)

![illustration13](/assets/csiblob/csiblob013.png)

![illustration14](/assets/csiblob/csiblob014.png)



Now let's create, again, the `pv`, `pvc` and a sample pod, without forgetting the required secret.

```bash

df@df-2404lts:~$ export staaccesskey=$(az storage account keys list --account-name stak8s2 -g rsg-cluster-aks --query "[0].value" -o tsv)
df@df-2404lts:~$ kubectl create secret generic azure-secret-stak8s2 --from-literal azurestorageaccountname=stak8s2 --from-literal azurestorageaccountkey=$staaccesskey --type=Opaque

```

```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: blob.csi.azure.com
  name: pv-blob2
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain 
  storageClassName: azureblob-fuse-premium
  mountOptions:
    - -o allow_other
    - --file-cache-timeout-in-seconds=120
  csi:
    driver: blob.csi.azure.com

    volumeHandle: stak8scontainer2
    volumeAttributes:
      containerName: stak8scontainer2
    nodeStageSecretRef:
      name: azure-secret-stak8s2
      namespace: default
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob2
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-blob2
  storageClassName: azureblob-fuse-premium
---
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: newblobpod2
  name: newblobpod2
spec:
  volumes:
  - name: blobvol2
    persistentVolumeClaim:
      claimName: pvc-blob2
  containers:
  - image: nginx
    name: newblobpod2
    volumeMounts:
    - mountPath: "/mnt/blob2"
      name: blobvol2
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}


```

```bash

df@df-2404lts:~$ k apply -f ./yamlconfig/blobvolume/blob2.yaml
persistentvolume/pv-blob2 created
persistentvolumeclaim/pvc-blob2 created
pod/newblobpod2 created
df@df-2404lts:~$ k get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS             VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-blob    10Gi       RWX            Retain           Bound    default/pvc-blob    azureblob-fuse-premium   <unset>                          43m
pv-blob2   10Gi       RWX            Retain           Bound    default/pvc-blob2   azureblob-fuse-premium   <unset>                          3s
df@df-2404lts:~$ k get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS             VOLUMEATTRIBUTESCLASS   AGE
pvc-blob    Bound    pv-blob    10Gi       RWX            azureblob-fuse-premium   <unset>                 43m
pvc-blob2   Bound    pv-blob2   10Gi       RWX            azureblob-fuse-premium   <unset>                 8s
df@df-2404lts:~$ k get pod
NAME          READY   STATUS    RESTARTS   AGE
blobpod       1/1     Running   0          38m
newblobpod2   1/1     Running   0          11s
df@df-2404lts:~$ k exec newblobpod2 -- ls /mnt/blob2
df@df-2404lts:~$ k exec newblobpod2 -- touch /mnt/blob2/file2.txt
df@df-2404lts:~$ k exec newblobpod2 -- ls /mnt/blob2
file2.txt

```

Another intersting point is that we can have the private endpoint in any vnet, it will work as long as the Vnet hosting the pod (and its node) can resolved the IP address of the private endpoint, through the DNS link, and that there is connectivity to the other Vnet, through direct peering, or hub and spoke topology and stuff like this.

Well it all work nicely, but it's not really dynamic, so let's have a look at the driver to see if we may be abble to customize the `StorageClass` shall we ?

## 3. A look at the options on the CSI `StorageClass`

### 3.1. The csi driver for blob on an AKS cluster

If we look for it, we can find a github repository dedicated to this [csi driver](https://github.com/kubernetes-sigs/blob-csi-driver/tree/master).

The first interesting thing is the version, which we can check on the cluster, for comparison with the current version.

![illustration15](/assets/csiblob/csiblob015.png)

Under the condition that the driver is enabled on the aks cluster, we should be able to display its version.

```bash

df@df-2404lts:~$ k describe csidrivers.storage.k8s.io blob.csi.azure.com 
Name:         blob.csi.azure.com
Namespace:    
Labels:       addonmanager.kubernetes.io/mode=Reconcile
Annotations:  csiDriver: v1.27.2
API Version:  storage.k8s.io/v1
Kind:         CSIDriver
Metadata:
  Creation Timestamp:  2026-03-02T15:02:23Z
  Resource Version:    86981
  UID:                 f0062af4-6ae5-4846-b1eb-70e1e5e1362b
Spec:
  Attach Required:     false
  Fs Group Policy:     ReadWriteOnceWithFSType
  Pod Info On Mount:   true
  Requires Republish:  true
  Se Linux Mount:      false
  Storage Capacity:    false
  Token Requests:
    Audience:            api://AzureADTokenExchange
    Expiration Seconds:  3600
  Volume Lifecycle Modes:
    Persistent
    Ephemeral
Events:  <none>

```

We can check other things such as the daemon set related to the blob driver, and its `ServiceAccount`

```bash

df@df-2404lts:~$ k get daemonsets.apps -n kube-system
NAME                                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
aks-secrets-store-csi-driver               3         3         3       3            3           <none>                     4h7m
aks-secrets-store-csi-driver-windows       0         0         0       0            0           <none>                     4h7m
aks-secrets-store-provider-azure           3         3         3       3            3           kubernetes.io/os=linux     4h7m
aks-secrets-store-provider-azure-windows   0         0         0       0            0           kubernetes.io/os=windows   4h7m
ama-metrics-node                           3         3         3       3            3           <none>                     4h5m
ama-metrics-win-node                       0         0         0       0            0           <none>                     4h5m
azure-cns                                  3         3         3       3            3           <none>                     4h7m
azure-cns-win                              0         0         0       0            0           <none>                     4h7m
azure-ip-masq-agent                        3         3         3       3            3           <none>                     4h7m
cilium                                     3         3         3       3            3           <none>                     4h7m
cloud-node-manager                         3         3         3       3            3           <none>                     4h7m
cloud-node-manager-windows                 0         0         0       0            0           <none>                     4h7m
csi-azuredisk-node                         3         3         3       3            3           <none>                     4h7m
csi-azuredisk-node-win                     0         0         0       0            0           <none>                     4h7m
csi-azurefile-node                         3         3         3       3            3           <none>                     4h7m
csi-azurefile-node-win                     0         0         0       0            0           <none>                     4h7m
csi-blob-node                              3         3         3       3            3           <none>                     15m
retina-agent                               0         0         0       0            0           <none>                     4h7m
retina-agent-win                           0         0         0       0            0           kubernetes.io/os=windows   4h7m
windows-kube-proxy-initializer             0         0         0       0            0           <none>                     4h7m

df@df-2404lts:~$ k get daemonsets.apps -n kube-system csi-blob-node -o json |jq .spec.template.spec.serviceAccount
"csi-blob-node-sa"

```

And looking a little bit more, we can identify which `clusterroles` are associated to this `ServiceAccount`.

```bash

df@df-2404lts:~$ k get clusterrolebindings.rbac.authorization.k8s.io csi-blob-node-secret-binding -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile","kubernetes.io/bootstrapping":"rbac-defaults"},"name":"csi-blob-node-secret-binding"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"csi-blob-node-secret-role"},"subjects":[{"kind":"ServiceAccount","name":"csi-blob-node-sa","namespace":"kube-system"}]}
  creationTimestamp: "2026-03-02T11:10:24Z"
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/bootstrapping: rbac-defaults
  name: csi-blob-node-secret-binding
  resourceVersion: "768"
  uid: 637cac73-28c1-42b6-809a-a6a9b6ab5380
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: csi-blob-node-secret-role
subjects:
- kind: ServiceAccount
  name: csi-blob-node-sa
  namespace: kube-system

```

```bash

df@df-2404lts:~$ k get clusterrole csi-blob-node-secret-role -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRole","metadata":{"annotations":{},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile","kubernetes.io/bootstrapping":"rbac-defaults"},"name":"csi-blob-node-secret-role"},"rules":[{"apiGroups":[""],"resources":["secrets"],"verbs":["get"]}]}
  creationTimestamp: "2026-03-02T11:10:24Z"
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/bootstrapping: rbac-defaults
  name: csi-blob-node-secret-role
  resourceVersion: "767"
  uid: cbb075e9-1364-4d0c-8c26-c675b5ad4891
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get

```

With that we find out how the provider gain access to secrets provided to grant access to the blob storage account.

Ok fine, let's dig a little bit more in the [provider configuration](https://github.com/kubernetes-sigs/blob-csi-driver/blob/master/docs/driver-parameters.md).

Name | Meaning | Example | Mandatory | Default value
--- | --- | --- | --- | ---
skuName | Azure storage account type (alias: `storageAccountType`) | `Standard_LRS`, `Premium_LRS`, `Standard_GRS`, `Standard_RAGRS`, `Standard_ZRS`, `Premium_ZRS`  | No | `Standard_LRS`
location | Azure location | `eastus`, `westus`, etc. | No | if empty, driver will use the same location name as current k8s cluster
resourceGroup | Azure resource group name | existing resource group name | No | if empty, driver will use the same resource group name as current k8s cluster
subscriptionID | specify Azure subscription ID in which blob storage directory will be created | Azure subscription ID | No | if not empty, `resourceGroup` must be provided
storageAccount | specify Azure storage account name| STORAGE_ACCOUNT_NAME | No | When a specific storage account name is not provided, the driver will look for a suitable storage account that matches the account settings within the same resource group. If it fails to find a matching storage account, it will create a new one. However, if a storage account name is specified, the storage account must already exist.
protocol | specify blobfuse, blobfuse2 or NFSv3 mount | `fuse`, `fuse2`, `nfs` | No | `fuse`
networkEndpointType | specify network endpoint type for the storage account created by driver. If `privateEndpoint` is specified, a private endpoint will be created for the storage account. For other cases, a service endpoint will be created for `nfs` protocol by default. | "",`privateEndpoint` | No | "",<br>for AKS cluster, make sure cluster Control plane identity (that is, your AKS cluster name) is added to the Contributor role in the resource group hosting the VNet
storageEndpointSuffix | specify Azure storage endpoint suffix | `core.windows.net`, `core.chinacloudapi.cn`, etc | No | if empty, driver will use default storage endpoint suffix according to cloud environment, e.g. `core.windows.net`
containerName | specify the existing container(directory) name | existing container name | No | if empty, driver will create a new container name, starting with `pvc-fuse` for blobfuse or `pvc-nfs` for NFSv3
containerNamePrefix | specify Azure storage directory prefix created by driver | can only contain lowercase letters, numbers, hyphens, and length should be less than 21 | No |
server | specify Azure storage account server address | existing server address, e.g. `accountname.blob.core.chinacloudapi.cn` | No | if empty, driver will use the default Azure storage account server address based on cloud provider config
accessTier | [Access tier for storage account](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) | Standard account can choose `Hot` or `Cool`, and Premium account can only choose `Premium` | No | empty(use default setting for different storage account types)
allowBlobPublicAccess | Allow or disallow public access to all blobs or containers for storage account created by driver | `true`,`false` | No | `false`
allowSharedKeyAccess | Allow or disallow shared key access for storage account created by driver (only applicable for NFS mount or blobfuse mount with managed identity) | `true`,`false` | No | `true`
requireInfraEncryption | specify whether or not the service applies a secondary layer of encryption with platform managed keys for data at rest for storage account created by driver | `true`,`false` | No | `false`
storageEndpointSuffix | specify Azure storage endpoint suffix | `core.windows.net`, `core.chinacloudapi.cn`, etc | No | if empty, driver will use default storage endpoint suffix according to cloud environment
tags | [tags](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources) would be created in newly created storage account | tag format: 'foo=aaa,bar=bbb' | No | ""
matchTags | whether matching tags when driver tries to find a suitable storage account | `true`,`false` | No | `false`
useDataPlaneAPI | specify whether use data plane API for blob container create/delete, this could solve the SRP API throttling issue since data plane API has almost no limit, while it would fail when there is firewall or vnet setting on storage account | `true`,`false` | No | `false`
softDeleteBlobs | Enable [soft delete for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview), specify the days to retain deleted blobs | "7" | No | Soft Delete Blobs is disabled if empty
softDeleteContainers | Enable [soft delete for containers](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview), specify the days to retain deleted containers | "7" | No | Soft Delete Containers is disabled if empty
enableBlobVersioning | Enable [blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview), can't enabled when `protocol` is `nfs` or `isHnsEnabled` is `true` | `true`,`false` | No | versioning for blobs is disabled if empty
--- | **Following parameters are only for blobfuse** | --- | --- |
storeAccountKey | Should the storage account key be stored in a Kubernetes secret <br> (Note:  if set to `false`, the driver will use the kubelet identity to obtain the account key during volume mount) | `true`,`false` | No | `true`
getLatestAccountKey | whether getting the latest account key based on the creation time, this driver would get the first key by default | `true`,`false` | No | `false`
secretName | specify secret name to store account key | | No |
secretNamespace | specify the namespace of secret to store account key | `default`,`kube-system`, etc | No | pvc namespace
isHnsEnabled | enable `Hierarchical namespace` for Azure DataLake storage account | `true`,`false` | No | `false`
--- | **Following parameters are only for NFS protocol** | --- | --- |
mountPermissions | mounted folder permissions. The default is `0777`, if set as `0`, driver will not perform `chmod` after mount | `0777` | No |
fsGroupChangePolicy | indicates how volume's ownership will be changed by the driver, pod `securityContext.fsGroupChangePolicy` is ignored  | `OnRootMismatch`(by default), `Always`, `None` | No | `OnRootMismatch`
--- | **Following parameters are only for vnet setting** | --- | --- |
vnetResourceGroup | specify vnet resource group where virtual network is | existing resource group name | No | if empty, driver will use the `vnetResourceGroup` value in azure cloud config file
vnetName | virtual network name | existing virtual network name | No | if empty, driver will use the `vnetName` value in azure cloud config file
subnetName | subnet name | existing subnet name(s) of the agent node, if you want to update service endpoints on multiple subnets, separate them using a comma (`,`) | No | if empty, driver will update all the subnets under the cluster virtual network
vnetLinkName | virtual network link name associated with private dns zone |  | No | if empty, driver will use the `vnetName + "-vnetlink"` by default
publicNetworkAccess | `PublicNetworkAccess` property of created storage account by the driver | `Enabled`, `Disabled`, `SecuredByPerimeter` | No |


A few parameters seem interesting:

- `networkEndpointType` that we want to set to `privateEndpoint`
- `publicNetworkAccess` that we want to set ot `Disabled`
- `storeAccountKey` may be interesting to avoid managing access with access key and secrets.

We'll note that the `networkEndpointType` should probably be used with `vnetResourceGroup`, `vnetName` and `subnetName`

Now we want to experiment &#128526;.

### 3.2. Our first custom blob `StorageClass`

To get started, we'll try the following sotrage class with only `requireInfraEncryption`

```yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: blob-fuse-infrastructure-encryption-enabled
provisioner: blob.csi.azure.com
parameters:
  skuName: Premium_LRS
  requireInfraEncryption: "true"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  - --use-attr-cache=true
  - --cancel-list-on-mount-seconds=10
  - -o attr_timeout=120
  - -o entry_timeout=120
  - -o negative_timeout=120
  - --log-level=LOG_WARNING
  - --cache-size-mb=1000

```

Once the `StorageClass` is created, we can create a pvc, an da pod using it.

```bash

df@df-2404lts:~$ k describe storageclasses.storage.k8s.io blob-fusecustom1 
Name:            blob-fusecustom1
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"allowVolumeExpansion":true,"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"blob-fusecustom1"},"mountOptions":["-o allow_other","--file-cache-timeout-in-seconds=120","--use-attr-cache=true","--cancel-list-on-mount-seconds=10","-o attr_timeout=120","-o entry_timeout=120","-o negative_timeout=120","--log-level=LOG_WARNING","--cache-size-mb=1000"],"parameters":{"allowBlobPublicAccess":"false","containerNamePrefix":"staakslab1","networkEndpointType":"privateEndpoint","publicNetworkAccess":"Disabled","requireInfraEncryption":"true","skuName":"Premium_LRS","subnetName":"sub2-vnet-sbx-aks1","vnetName":"vnet-sbx-aks1","vnetResourceGroup":"rsg-spoke-aks"},"provisioner":"blob.csi.azure.com","reclaimPolicy":"Delete","volumeBindingMode":"Immediate"}

Provisioner:           blob.csi.azure.com
Parameters:            allowBlobPublicAccess=false,containerNamePrefix=staakslab1,networkEndpointType=privateEndpoint,publicNetworkAccess=Disabled,requireInfraEncryption=true,skuName=Premium_LRS,subnetName=sub2-vnet-sbx-aks1,vnetName=vnet-sbx-aks1,vnetResourceGroup=rsg-spoke-aks
AllowVolumeExpansion:  True
MountOptions:
  -o allow_other
  --file-cache-timeout-in-seconds=120
  --use-attr-cache=true
  --cancel-list-on-mount-seconds=10
  -o attr_timeout=120
  -o entry_timeout=120
  -o negative_timeout=120
  --log-level=LOG_WARNING
  --cache-size-mb=1000
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>

```

```yaml

---
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: blob-infra-encryption
spec: {}
status: {}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blobfuse-infra-encryption
  namespace: blob-infra-encryption
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: blob-fuse-infrastructure-encryption-enabled
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: blobfusestapod
  name: blobfusestapod
  namespace: blob-infra-encryption
spec:
  volumes:
  - name: blobvol
    persistentVolumeClaim:
      claimName: pvc-blobfuse-infra-encryption
  containers:
  - image: nginx
    name: blobfusestapod
    resources: {}
    volumeMounts:
    - mountPath: "/mnt/blob"
      name: blobvol
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}  

```

It takes some time for the underlying storage account to be created.

```bash

df@df-2404lts:~$ k describe pvc -n blob-infra-encryption 
Name:          pvc-blobfuse-infra-encryption
Namespace:     blob-infra-encryption
StorageClass:  blob-fuse-infrastructure-encryption-enabled
Status:        Bound
Volume:        pvc-d7df48b3-78c0-4b0e-a1a2-3774bd2bb32a
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: blob.csi.azure.com
               volume.kubernetes.io/storage-provisioner: blob.csi.azure.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      10Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Used By:       blobfusestapod
Events:
  Type    Reason                 Age                 From                                                                                          Message
  ----    ------                 ----                ----                                                                                          -------
  Normal  Provisioning           3m                  blob.csi.azure.com_csi-blob-controller-696d9b5c58-r5thn_46dba3da-37d1-46b3-b3a9-76bae056e0b0  External provisioner is provisioning volume for claim "blob-infra-encryption/pvc-blobfuse-infra-encryption"
  Normal  ExternalProvisioning   2m40s (x4 over 3m)  persistentvolume-controller                                                                   Waiting for a volume to be created either by the external provisioner 'blob.csi.azure.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
  Normal  ProvisioningSucceeded  2m39s               blob.csi.azure.com_csi-blob-controller-696d9b5c58-r5thn_46dba3da-37d1-46b3-b3a9-76bae056e0b0  Successfully provisioned volume pvc-d7df48b3-78c0-4b0e-a1a2-3774bd2bb32a

df@df-2404lts:~$ k get pvc -n blob-infra-encryption 
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                                  VOLUMEATTRIBUTESCLASS   AGE
pvc-blobfuse-infra-encryption   Bound    pvc-d7df48b3-78c0-4b0e-a1a2-3774bd2bb32a   10Gi       RWX            blob-fuse-infrastructure-encryption-enabled   <unset>                 28s

df@df-2404lts:~$ k get pod -n blob-infra-encryption 
NAME             READY   STATUS    RESTARTS   AGE
blobfusestapod   1/1     Running   0          32s

```

We can check the storage and vrify that the infrastructre encryption is enableds

We may have a hint with this section of the documentation that state required authorizations on the csi driver.

![illustration16](/assets/csiblob/csiblob018.png)


![illustration17](/assets/csiblob/csiblob019.png)

Since it's working, let's try other things.

### 3.3. Creating a `StorageClass` from an existing storage account

This time, we want to create a custom storage class from an existing storage account.

![illustration18](/assets/csiblob/csiblob020.png)


![illustration19](/assets/csiblob/csiblob021.png)

Our storage account is protected by a private endpoint, and we want to be able to identify the containers that the `StorageClass` will generate when we create new `pvc`.

To achieve this, we can use the following parameters:

- `storageAccount`
- `containerNamePrefix`

The manifest for the `StorageClasse` looks like below.

```yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: blob-fusecustom-existingstorage
provisioner: blob.csi.azure.com
parameters:
  skuName: Standard_LRS  
  containerNamePrefix: "staakslab1"
  storageAccount: "stak8s5"
  resourceGroup: "rsg-cluster-aks"
reclaimPolicy: "Delete"
volumeBindingMode: "Immediate"
allowVolumeExpansion: true 
mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  - --use-attr-cache=true
  - --cancel-list-on-mount-seconds=10 
  - -o attr_timeout=120
  - -o entry_timeout=120
  - -o negative_timeout=120
  - --log-level=LOG_WARNING 
  - --cache-size-mb=1000

```


```bash

df@df-2404lts:~$ k describe storageclasses.storage.k8s.io blob-fusecustom-existingstorage 
Name:            blob-fusecustom-existingstorage
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"allowVolumeExpansion":true,"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"blob-fusecustom-existingstorage"},"mountOptions":["-o allow_other","--file-cache-timeout-in-seconds=120","--use-attr-cache=true","--cancel-list-on-mount-seconds=10","-o attr_timeout=120","-o entry_timeout=120","-o negative_timeout=120","--log-level=LOG_WARNING","--cache-size-mb=1000"],"parameters":{"containerNamePrefix":"staakslab1","resourceGroup":"rsg-cluster-aks","skuName":"Standard_LRS","storageAccount":"stak8s5"},"provisioner":"blob.csi.azure.com","reclaimPolicy":"Delete","volumeBindingMode":"Immediate"}

Provisioner:           blob.csi.azure.com
Parameters:            containerNamePrefix=staakslab1,resourceGroup=rsg-cluster-aks,skuName=Standard_LRS,storageAccount=stak8s5
AllowVolumeExpansion:  True
MountOptions:
  -o allow_other
  --file-cache-timeout-in-seconds=120
  --use-attr-cache=true
  --cancel-list-on-mount-seconds=10
  -o attr_timeout=120
  -o entry_timeout=120
  -o negative_timeout=120
  --log-level=LOG_WARNING
  --cache-size-mb=1000
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>

```

We can now create `pvc` pointing to this StorageClass, and mount those in a pod. But first, we need to provide a secret that contains the access key to the storage account, as we did in the early sections.

```yaml

---
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: blob-existingsta
spec: {}
status: {}

```

```bash

df@df-2404lts:~$ export stak8s5Key=$(az storage account keys list --account-name stak8s5 -g rsg-cluster-aks --query '[0].value' -o tsv)

df@df-2404lts:~$ kubectl create secret generic azure-secret-stak8s5 --from-literal azurestorageaccountname=stak8s5 --from-literal azurestorageaccountkey=$stak8s5Key --type=Opaque -n blob-existingsta

```

```yaml

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-existingsta
  namespace: blob-existingsta
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: blob-fusecustom-existingstorage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-existingsta2
  namespace: blob-existingsta
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: blob-fusecustom-existingstorage
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: existingstapod
  name: existingstapod
  namespace: blob-existingsta
spec:
  volumes:
  - name: blobvol
    persistentVolumeClaim:
      claimName: pvc-blob-existingsta
  - name: blobvol2
    persistentVolumeClaim:
      claimName: pvc-blob-existingsta2
  containers:
  - image: nginx
    name: existingstapod
    resources: {}
    volumeMounts:
    - mountPath: "/mnt/blob"
      name: blobvol
    - mountPath: "/mnt/blob2"
      name: blobvol2
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

If everything works correctly, we should have 2 `pvc` in the `blob-existingsta` `namespace`, and one `pod` mouting those volumes.

```bash

df@df-2404lts:~$ k get pvc -n blob-existingsta 
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                      VOLUMEATTRIBUTESCLASS   AGE
pvc-blob-existingsta    Bound    pvc-3e837bf9-db8f-48d7-838a-922b3d5c857d   10Gi       RWX            blob-fusecustom-existingstorage   <unset>                 3m29s
pvc-blob-existingsta2   Bound    pvc-e9aa1a7e-d7c1-42b4-a2fb-17bfc603b03c   10Gi       RWX            blob-fusecustom-existingstorage   <unset>                 9s

df@df-2404lts:~$ k get pod -n blob-existingsta 
NAME             READY   STATUS    RESTARTS   AGE
existingstapod   1/1     Running   0          16s

```

And also additional containers on the storage, prefiexed with **staakslab1**

![illustration20](/assets/csiblob/csiblob022.png)

### 3.4. `StorageClass` with private endpoint

This time we want to push it even more with a `StorageClass` that would generate automatically storage with private endpoint.

Our new storage class will be define with the parameters from earlier:


- `networkEndpointType` that we want to set to `privateEndpoint`
- `publicNetworkAccess` that we want to set ot `Disabled`
- `vnetResourceGroup`
- `vnetName`
- `subnetName`

```yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: blob-fuse-public-access-disabled
provisioner: blob.csi.azure.com
parameters:
  skuName: Premium_LRS 
  requireInfraEncryption: "true"
  publicNetworkAccess: "Disabled"
  networkEndpointType: "privateEndpoint"
  containerNamePrefix: "staakslab1"
  allowBlobPublicAccess: "false"
  vnetResourceGroup: "rsg-spoke-aks"
  vnetName: "vnet-sbx-aks1"
  subnetName: "sub2-vnet-sbx-aks1"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  - --use-attr-cache=true
  - --cancel-list-on-mount-seconds=10
  - -o attr_timeout=120
  - -o entry_timeout=120
  - -o negative_timeout=120
  - --log-level=LOG_WARNING
  - --cache-size-mb=1000
---

```

Then we define a `pvc` using this new `StorageClass`

```yaml

apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: blob-publicaccess-disabled
spec: {}
status: {}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-blob-public-access-disabled
  namespace: blob-publicaccess-disabled
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: blob-fuse-public-access-disabled

```

The pvc seems to be stuck in pending. 

```bash

df@df-2404lts:~$ k get pvc -n blob-publicaccess-disabled 
NAME                              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS                       VOLUMEATTRIBUTESCLASS   AGE
pvc-blob-public-access-disabled   Pending                                      blob-fuse-public-access-disabled   <unset>                 32s

df@df-2404lts:~$ k describe -n blob-publicaccess-disabled pvc pvc-blob-public-access-disabled 
Name:          pvc-blob-public-access-disabled
Namespace:     blob-publicaccess-disabled
StorageClass:  blob-fuse-public-access-disabled
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: blob.csi.azure.com
               volume.kubernetes.io/storage-provisioner: blob.csi.azure.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       blobfusestapod2
Events:
  Type    Reason                Age               From                                                                                          Message
  ----    ------                ----              ----                                                                                          -------
  Normal  Provisioning          64s               blob.csi.azure.com_csi-blob-controller-6bfbd84d8c-ql22b_196f9691-2019-42d5-a291-dc854af359b9  External provisioner is provisioning volume for claim "blob-publicaccess-disabled/pvc-blob-public-access-disabled"
  Normal  ExternalProvisioning  7s (x6 over 64s)  persistentvolume-controller                                                                   Waiting for a volume to be created either by the external provisioner 'blob.csi.azure.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.

```

Looking at the kubernetes events, we can find some usefull informations.

```bash

df@df-2404lts:~$ k events -n blob-publicaccess-disabled 

LAST SEEN   TYPE      REASON               OBJECT                                                  MESSAGE

28s         Warning   ProvisioningFailed   PersistentVolumeClaim/pvc-blob-public-access-disabled   rpc error: code = Internal desc = ensure storage account failed with create virtual link for vnet(vnet-sbx-aks1) and DNS Zone(privatelink.blob.core.windows.net) in resourceGroup(rsg-spoke-aks): GET https://management.azure.com/subscriptions/00e23d33-3e41-4cd9-b5b4-f66391a4c77a/resourceGroups/rsg-spoke-aks/providers/Microsoft.Network/privateDnsOperationStatuses/RnJvbnRFbmRBc3luY09wZXJhdGlvbjtVcHNlcnRWaXJ0dWFsTmV0d29ya0xpbms7NmM5NGY3YWEtNmE0Yy00ODQwLWExNGItZTFkYWU2YzJmYmYwXzAwZTIzZDMzLTNlNDEtNGNkOS1iNWI0LWY2NjM5MWE0Yzc3YQ==
--------------------------------------------------------------------------------
RESPONSE 200: 200 OK
ERROR CODE: BadRequest
--------------------------------------------------------------------------------
{
  "error": {
    "code": "BadRequest",
    "message": "A virtual network cannot be linked to multiple zones with overlapping namespaces. You tried to link virtual network with 'privatelink.blob.core.windows.net' and 'privatelink.blob.core.windows.net' zones."
  },
  "status": "Failed"
}
--------------------------------------------------------------------------------
27s (x2 over 95s)   Normal    Provisioning           PersistentVolumeClaim/pvc-blob-public-access-disabled   External provisioner is provisioning volume for claim "blob-publicaccess-disabled/pvc-blob-public-access-disabled"
8s (x8 over 95s)    Normal    ExternalProvisioning   PersistentVolumeClaim/pvc-blob-public-access-disabled   Waiting for a volume to be created either by the external provisioner 'blob.csi.azure.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.

```

The message is clear, and it makles sense. We already have a private dns zone lonked to the vnet of our cluster, to be used by the previous storage class, and by one of the storage account that we configured manually. A rapid check will show us that we have an additional private dns zone with the same dns namespaces.

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ az network private-dns zone list -o table |grep blob
privatelink.blob.core.windows.net    rsg-dns-akslab   4             25000            2                      1000                      0                                      100                                       Succeeded
privatelink.blob.core.windows.net    rsg-spoke-aks    1             25000            0                      1000                      0                                      100                                       Succeeded


```

Ok, let's do some clean up, and remove the previous dns zone link.

After some time we have our `pvc` mounted in a `pod`.

```bash

df@df-2404lts:~$ k get pvc -n blob-publicaccess-disabled pvc-blob-public-access-disabled 
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                       VOLUMEATTRIBUTESCLASS   AGE
pvc-blob-public-access-disabled   Bound    pvc-56514ce8-3d67-44ae-9e80-81ca7f212614   10Gi       RWX            blob-fuse-public-access-disabled   <unset>                 5m9s

df@df-2404lts:~$ k get pv pvc-56514ce8-3d67-44ae-9e80-81ca7f212614 
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                        STORAGECLASS                       VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-56514ce8-3d67-44ae-9e80-81ca7f212614   10Gi       RWX            Delete           Bound    blob-publicaccess-disabled/pvc-blob-public-access-disabled   blob-fuse-public-access-disabled   <unset>                          4m15s

df@df-2404lts:~$ k describe pod -n blob-publicaccess-disabled blobfusestapod2 
Name:             blobfusestapod2
Namespace:        blob-publicaccess-disabled
Priority:         0
Service Account:  default
Node:             aks-aksnp0lab1-35475183-vmss000002/172.16.0.7
Start Time:       Wed, 11 Mar 2026 17:00:07 +0100
Labels:           run=blobfusestapod2
Annotations:      <none>
Status:           Running
IP:               100.64.1.151
IPs:
  IP:  100.64.1.151
Containers:
  blobfusestapod2:
    Container ID:   containerd://9aa50a1f857a8b1f5d53136dac6bc245a253f744c878d66855abc3c45e9d852e
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:bc45d248c4e1d1709321de61566eb2b64d4f0e32765239d66573666be7f13349
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 11 Mar 2026 17:00:10 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /mnt/blob from blobvol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w5lhf (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  blobvol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-blob-public-access-disabled
    ReadOnly:   false
  kube-api-access-w5lhf:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason             Age                    From                Message
  ----     ------             ----                   ----                -------
  Warning  FailedScheduling   7m11s                  default-scheduler   0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
  Warning  FailedScheduling   5m21s (x2 over 5m21s)  default-scheduler   0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
  Normal   Scheduled          5m21s                  default-scheduler   Successfully assigned blob-publicaccess-disabled/blobfusestapod2 to aks-aksnp0lab1-35475183-vmss000002
  Normal   NotTriggerScaleUp  7m10s                  cluster-autoscaler  pod didn't trigger scale-up: 1 pod has unbound immediate PersistentVolumeClaims
  Normal   Pulling            5m21s                  kubelet             Pulling image "nginx"
  Normal   Pulled             5m19s                  kubelet             Successfully pulled image "nginx" in 1.053s (1.053s including waiting). Image size: 62960551 bytes.
  Normal   Created            5m19s                  kubelet             Created container: blobfusestapod2
  Normal   Started            5m19s                  kubelet             Started container blobfusestapod2

```

On the Azure side, we can see the newly created storage, with its private endpint.

![illustration2](/assets/csiblob/csiblob023.png)

I could probably try some other stuffs, but let's call it a day &#128519;.

## 4. Before leaving

In this article, we went deep in what we can do with the csi driver for Azure Blob storage.

By default, we can leverage the `StorageClass` `azureblob-fuse-premium` and dynamically create `pv` and storage account.

If we have existing storage, we can associate those to pv and pvc in a more static manner, allowing us to use configuration that are not available in the default `StorageClass`

Finally, it's possible to leverage custom storage classes, based on the blob csi driver, with custom parameters that we saw in this article.
We can either create a storage class using a single storage under the hood, or we can configure a `StorageClass` that dynamically create storage account with private endpoint. 
However, the private dns zone cannot be specified at this point so it implies that we lose the central management of those.

That was fun. See you soon

