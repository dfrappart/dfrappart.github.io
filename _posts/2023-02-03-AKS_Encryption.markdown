---
layout: post
title:  "AKS encryption options"
date:   2023-02-03 18:00:00 +0200
year: 2023
categories: AKS Security
---

Hi everyone!

In this article, I'll talk about AKS and the options for encrypting the different part of the Kubernetes managed cluster.

The agenda is below:

1. Reminder of Azure platform encryption options
2. Manage AKS control plane encryption
3. Manage AKS worker plane encryption
4. Conclusion

## 1. Reminder of Azure platform encryption options

Before diving on AKS encryption specificities, let's review a bit what is it about encryption and Azure.

Usually, the first encryption considered for Azure services is encryption at rest.

As stated in [Azure security documentation](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-atrest), encryption at rest is used first and foremost to protect stored data. 

In this regard, encryption at rest is related to encrypt the storage layer on Azure, which is Azure storage.
PaaS service or Azure managed disks for IaaS both use Azure storage to store data. We will find reference of [Server Side Encryption in the documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption) (or `SSE`)for the encryption of Azure storage.

By default, the platform manages the key used for encryption. In this case we talk about platform managed key.
Scenarios requiring more control on the key can also be implemented with a customer managed key brought from the outside (of Azure) and a key vault instance (or a cloud HSM in Azure, managed or dedicated, but that's another story).

But, because there's always a but ^^, in the case of Azure managed disk, it differs a little.
Remember, one of the main advantage of the Azure managed disk  is to **NOT see** the underlying Storage account.
In the past, we used to manage on our own the storage accounts used to store VHD files acting as disk for IaaS workload. Trust me when I say that It is far easier to go through the manage disk.
Here comes the trouble though, how could we manage encryption with Azure managed disk, since we cannot see the storage beneath the disk?

The technical answer to this question is something called an Azure Disk encryption set, which act as an intermediary between the key vault storing the encryption key and the managed disk.

![illustration1](/assets/aksencryption/aksencryption001.png)

*Note:* 

For Azure managed disks, there is another solution which is called [Azure Disk Encryption](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption#server-side-encryption-versus-azure-disk-encryption) (or `ADE`). It leverage either Dm-crypt for Linux VMs, or Bitlocker for Windows VMs. 
While there is an option to also managed encryption with a Customer managed key, the capability for byok scenarios with the Server Side encryption makes it a bit redundant and less flexible due to the OS dependancy.
Interestingly enough though, it's worth noting that Azure Defender still recommands to implement ADE, even when SSE is activated by default.

One explanation could be that the SSE does not provide end to end encryption between the virtual host and the storage layer. On this point, there is now the capability to use [Host encryption](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-enable-host-based-encryption-portal?tabs=azure-powershell).
Host encryption, as its name implies, encrypt the data end to end between the Compute layer and the storage layer. There is not always an option to manage with a customer key the host encryption as of now.


Ok, that's all for this overview of encryption in Azure. Now let's focus specifically on encryption for AKS

## 2. Manage AKS control plane encryption

### 2.1. AKS control plane encryption concepts

We will start by encryption for the control plane.
As showed on the schema below, Kubernetes architecture (hence AKS architecture) is composed of the control plane and the worker plane:

![illustration2](/assets/aksencryption/aksencryption002.png)  
  
For an AKS cluster, the control plane is managed by the platform, and is something of the order of a PaaS instance, while the worker plane, that we'll look at afterward, is more in the IaaS family, even if it is for the most part managed by the AKS control plane.

It's important to get that right, because it does define which are the options for encrypting AKS.

Because the control plane is part of the PaaS family, there is no user access to its underlying system.
If we dive just a little bit deeper, and consider the different piece of the control plane:

![illustration3](/assets/aksencryption/aksencryption003.png)  

We can see that only etcd is stateful, and thus requires access to storage. 
That means that only etcd should be considered when looking at encryption matters.

When looking at the AKS documentation, we can find de dedicated section about adding [kms etcd encryption to AKS](https://learn.microsoft.com/en-us/azure/aks/use-kms-etcd-encryption).
The concepts of kms encryption for etcd is not an AKS only feature. There is a dedicated section on the kubernetes [documentation](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/#verifying-that-the-data-is-encrypted) about how to use a kms provider for etcd encryption at rest. 

*Note*: There's also an [OSS project](https://github.com/Azure/kubernetes-kms) to allow the use of an Azure Key Vault as a kubernetes kms provider for other kubernetes distribution.

As stated by the documentation, the feature aims at providing an external provider to encrypt Kubernetes secrets in etcd:

```

The KMS encryption provider uses an envelope encryption scheme to encrypt data in etcd. The data is encrypted using a data encryption key (DEK); a new DEK is generated for each encryption. The DEKs are encrypted with a key encryption key (KEK) that is stored and managed in a remote KMS. The KMS provider uses gRPC to communicate with a specific KMS plugin. The KMS plugin, which is implemented as a gRPC server and deployed on the same host(s) as the Kubernetes control plane, is responsible for all communication with the remote KMS.

```

The feature is currently in beta on AKS, and that may be because there are still 2 versions of kms for kubernetes, depending on the kubernetes version.
Also, there is no need to deep too dive in what's happening below the surgace of kubernetes, since the idea is to rely on the Azure platform to manage that.

As expected, similarly to other customisable feature in Azure, it uses a key vault key in a key vault instance to allows encryption and decryption of secrets in AKS managed etcd.

![illustration4](/assets/aksencryption/aksencryption004.png)

Let's stop here to see what we need.
So obviously, we need to have an AKS cluster.
Because this AKS cluster will need to gain access to the store hosting the key encryption key (`KEK`), we need a managed identity on our cluster. 
And because we juste mention it, there's a key vault instance used to store the `KEK` used by the control plane
Here start the plannification. There are two case for an AKS cluster related to the managed identity associated to the control plane (remember, we are talking about etcd encryption, so this is in the control plane!):

- System assigned managed identity (`SAI`), where the identity is created at the same time as the cluster. Simple in its configuration, also more restrictive, because only the cluster can use its system managed identity
- User assigned managed identity (`UAI`), where the identity is created beforehand. More steps (actually, 2 steps, creation and role assignments vs only role assignment for `SAI`)

Because the id of the `SAI` associated to the control plane is only known after the cluster is created, it would not be possible to assign the required role on the key vault before the cluster is created, hence a failure to provision the cluster.
That means we need to use an `UAI` for the cluster so that we can configure beforehand the role.
It makes sense in terms of management because usually, the key vault and the `KEK` are created beforehand by a dedicated security team. 
It is possible to update an exiting AKS cluster to use kms encryption, so it may be possible to act in 2 steps and use an `SAI`. 
But that's far from ideal in terms of automation, and that's probably why the documentation document only shows example with an `UAI`.

Let's see now how to activate this feature.

### 2.2. AKS kms encryption configuration

To use the kms encryption, it's possible to use the az cli command `az aks create` or the `az aks update`and specify the `--enable-azure-keyvault-kms` and the `--azure-keyvault-kms-key-id` parameters which are self-explanatory.
Note also the parameter `--azure-keyvault-kms-key-vault-network-access` which can be used to specify if the used key vault is public or private.

Depending on the key vault authorization model, either the `UAI` should be assigned an access policy with `decrypt` and `encrypt` action for keys, or the `Key Vault Crypto User` role on the key vault instance used to host the `KEK`.

At the time of the AKS cluster creation, the command should be like below:

```bash

az aks create --name myAKSCluster --resource-group MyResourceGroup --assign-identity $IDENTITY_RESOURCE_ID --enable-azure-keyvault-kms --azure-keyvault-kms-key-vault-network-access "Public" --azure-keyvault-kms-key-id $KEY_ID

```

For an existing cluster, the command should look like this:

```bash

az aks update --name myAKSCluster --resource-group MyResourceGroup --enable-azure-keyvault-kms --azure-keyvault-kms-key-vault-network-access "Public" --azure-keyvault-kms-key-id $KEY_ID

```

If you are a terraform afficionado, know that the feature is available in the azurerm provider since version 3.40.0 (since January 19th, 2023).

```bash

key_management_service {
    key_vault_key_id            = var.KmsKeyId
    key_vault_network_access    = var.AKSKmsKvAccess
}

```

If the provider is not yet up to date It's possible to work that around through the use of the azapi provider. 

```bash

resource "azapi_update_resource" "aksENableKVKMS" {
  for_each                              = toset(var.TrainingList) 
  
  type                                  = "Microsoft.ContainerService/managedClusters@2022-09-02-preview"
  resource_id                           = module.AKS[each.value].KubeId

  body                                  = jsonencode({
                                                properties = {
                                                  securityProfile = {
                                                    azureKeyVaultKms = {
                                                    enabled = true
                                                    keyId = azurerm_key_vault_key.akskmskey[each.value].id
                                                    keyVaultNetworkAccess = "Public"
                                                    keyVaultResourceId = null
                                                  }    
                                                },
                                              }
                                            })

}

```

To check if the feature is activated, we can look the json description of the cluster through az cli `az aks show` command: 

```bash

spike@azure:~$ az aks show -n <aks_cluster_name> -g <aks_rg_name> | jq .securityProfile.azureKeyVaultKms
{
  "enabled": true,
  "keyId": "https://<keyvault_name>.vault.azure.net/keys/<key_encryption_name>/34473cbf99454440874f1ba194d52841",
  "keyVaultNetworkAccess": "Public",
  "keyVaultResourceId": null
}


```

When enabled is set to true, we're good to go.

### 2.3. secrets management with kms encryption

Once the feature is activated, there are things to do.
First, existing kubernetes secrets should be updated. If not, those won't be encrypted with the kms key.
the update is done pretty easily through the below `kubectl` command.

```bash

kubectl get secrets --all-namespaces -o json | kubectl replace -f -

```

In a fully self-managed kubernetes, using `etcdctl` would allow to check that the secret is correctly encrypted through the kms provider.
That's not the case in AKS, so we just need to trust the feature here :p

Apart from that, we need to consider the key rotation. First because a key should always have a rotation policy, and second, because if the key is provisioned through terraform, we may (should) move the key management away from terraform to avoid keeping sensitive data in the state (that' usuallly a topic of discussion). If this is the case, the `ignore_changes` meta argument in the `lifecycle` block will be a must.

There's an az cli command available to rotate the key. Actually, this is just the same  `az aks update` command to activate the featre on an existing AKS clusterr.
Note that this command requires to the key vault key id, hence the key rotator operator (human or machine) also need access to the key vault hosting the key.

We use az cli to get the key vault key id.

```bash

spike@azure:~$ export keyid=$(az keyvault key show --name akskmskeyuaiaksdavidfrappart --vault-name kvkmsaks8u3k --query key.kid -o tsv)

```

Before using the update command on the AKS cluster

```bash

spike@azure:~$ az aks update -n <aks_cluster_name> -g <resourceGroup_name> --enable-azure-keyvault-kms --azure-keyvault-kms-key-vault-network-access "Public" --azure-keyvault-kms-key-id $keyid 

``` 

As an example, the samples below show the AKS kms configuration before the update and after. Note the difference in the `keyId` value.

```bash

spike@azure:~$ az aks show -n <aks_cluster_name> -g <resourceGroup_name> | jq .securityProfile.azureKeyVaultKms
WARNING: The behavior of this command has been altered by the following extension: aks-preview
{
  "enabled": true,
  "keyId": "https://kvkmsaks8u3k.vault.azure.net/keys/akskmskeyuaiaksdavidfrappart/f34473cbf99454440874f1ba194d52841",
  "keyVaultNetworkAccess": "Public",
  "keyVaultResourceId": null
}

```

```bash

spike@azure:~$ az aks show -n <aks_cluster_name> -g <resourceGroup_name> | jq .securityProfile.azureKeyVaultKms
WARNING: The behavior of this command has been altered by the following extension: aks-preview
{
  "enabled": true,
  "keyId": "https://kvkmsaks8u3k.vault.azure.net/keys/akskmskeyuaiaksdavidfrappart/fa9b05df2db94bafb0ed29e6ef359b82",
  "keyVaultNetworkAccess": "Public",
  "keyVaultResourceId": null
}


```

Afterward, the kubernetes secrets should be updated. It can be done with a `kubectl replace` command.

And that's about it for the control plane encryption. Let's move to the worker plane.

## 3. Manage AKS worker plane encryption

Moving to the encryption options for worker plane, we did mention earlier our options:

- Server side encryption for managed disk with Customer managed key, through the use of a disk encryption set
- Host encryption, to add an additional layer of encryption from hosts to storage arrays.

The schema below aims to illustrate this:

![illustration5](/assets/aksencryption/aksencryption005.png)

Let's have a look on the configuration details for each of those solutions.

### 3.1. Disk encryption with a Customer managed key

We discussed about the disk encryption set object, which act as a kind of middleman between a managed disk, a key vault instance and the storage layer control plane.

AKS can leverage a disk encryption set and use it to encrypt disks with a customer managed key.
There is a single parameter to set at cluster creation. That's atually one of the limitation.

Also, with just the configuration from the Azure control plane side, data disks used by kubernetes workloads are not encrypted by default. We need to add a few configuration steps, in the Kubernetes control plane.

First let's summarize the requirement for AKS worker plane encryption at rest:

- We need a key vault instance, with soft delete enabled an dpurge protection. Also, the key vault must be configured to allow disk encryption.
- We then need a disk encryption set, with a `UAI` assigned proper access on the key vault instance.
- Last, we need the key that we will use to encrypt / decrypt data on the storage.

```bash

resource "azurerm_key_vault" "akskmskv" {
  for_each                              = toset(var.TrainingList)
  name                                  = "kvkmsaks${resource.random_string.randomstringkvkms[each.value].result}"
  resource_group_name                   = azurerm_resource_group.RG[each.value].name
  location                              = azurerm_resource_group.RG[each.value].location
  enabled_for_disk_encryption           = true
  tenant_id                             = data.azurerm_client_config.currentclientconfig.tenant_id
  purge_protection_enabled              = true
  soft_delete_retention_days            = 7

  sku_name                              = "standard"


}

# Disk Encryption Set

module "UaiDesAks" {

  for_each                              = toset(var.TrainingList)
  #Module location
  source = "github.com/dfrappart/Terra-AZModuletest//Modules_building_blocks/441_UserAssignedIdentity/"
  
  #Module variable
  UAISuffix                             = replace("des-aks-${each.value}", ".", "")
  TargetRG                              = azurerm_resource_group.RG[each.value].name

}

resource "azurerm_role_assignment" "UaiDesAksReader" {
  for_each                              = toset(var.TrainingList)
  scope                                 = azurerm_resource_group.RG[each.value].id
  role_definition_name                  = "Reader"
  principal_id                          = module.UaiDesAks[each.value].PrincipalId
}


resource "azurerm_disk_encryption_set" "AKSEncryptionSet" {
  for_each                              = toset(var.TrainingList)
  name                                  = "des-aks-${each.value}"
  resource_group_name                   = azurerm_resource_group.RG[each.value].name
  location                              = azurerm_resource_group.RG[each.value].location
  key_vault_key_id                      = azurerm_key_vault_key.AksDesKey[each.value].id
  auto_key_rotation_enabled             = true 
  encryption_type                       = "EncryptionAtRestWithCustomerKey"

  identity {
    type = "UserAssigned"
    identity_ids = [
      module.UaiDesAks[each.value].FullUAIOutput.id
    ]
  }

  
}

# Access Policy for AKS Disk Encryption set

resource "azurerm_key_vault_access_policy" "AKSDESAccessPolicy" {
  for_each                              = toset(var.TrainingList)

  key_vault_id                          = azurerm_key_vault.akskmskv[each.value].id
  tenant_id                             = data.azurerm_client_config.currentclientconfig.tenant_id
  object_id                             = module.UaiDesAks[each.value].PrincipalId

  key_permissions                       = var.Keyperms_AKSDesUAI_AccessPolicy

  depends_on = [
    azurerm_key_vault.akskmskv
  ]
}


resource "azurerm_key_vault_key" "AksDesKey" {
  for_each                              = toset(var.TrainingList)
  name                                  = replace("des-aks-key${each.value}", ".", "")
  key_vault_id                          = azurerm_key_vault.akskmskv[each.value].id
  key_type                              = "RSA"
  key_size                              = 4096

  depends_on = [
    azurerm_key_vault_access_policy.AKSDESAccessPolicy
  ]

  key_opts = [
    "decrypt",
    "encrypt",
    "sign",
    "unwrapKey",
    "verify",
    "wrapKey",
  ]


}

```

Once those prerequisite are enabled, it's actually quite easy to set the feature in AKS, with the argument `disk_encryption_set_id` in Terraform or `--node-osdisk-diskencryptionset-id` in az cli command `az aks create`

We can check the encryption set in the aks json definition

```bash

spike@azure:~$ az aks show -n <aks_cluster_name> -g <rgname> | jq .diskEncryptionSetId
"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/<rgname>/providers/Microsoft.Compute/diskEncryptionSets/des-aks-david.frappart"

```

With that, node OS disks are encrypted with a custom managed key.

Now, workloads hosted on the cluster won't use the Os Disks but rather Managed disk provisionned through the storage classes avaiable in AKS.
Nowadays, those storage classes rely n the csi provider, and specifically, for volume based on Azure managed disk, on the `disk.csi.azure.com` provisioner.

Getting the definition of the storage classes shows that there is no default config for using the disk encryption set that we specified for the nodes.
However, we can find in the AKS documentation or in the [Azure disk CSI driver documentation](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/driver-parameters.md), there's a parameter to specify the disk encryption set

|Name	| Meaning	| Available Value |
|-|-|-|
| diskEncryptionSetID	| ResourceId of the disk encryption set to use for enabling encryption at rest	| format: /subscriptions/{subs-id}/resourceGroups/{rg-name}/providers/Microsoft.Compute/diskEncryptionSets/{diskEncryptionSet-name} |


With taht we can create a storage class with the disk encryption set id:
```yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1  
metadata:
  name: byok
provisioner: disk.csi.azure.com
parameters:
  skuname: StandardSSD_LRS
  kind: managed
  diskEncryptionSetID: "/subscriptions/{myAzureSubscriptionId}/resourceGroups/{myResourceGroup}/providers/Microsoft.Compute/diskEncryptionSets/{myDiskEncryptionSetName}"
  
```

And then a persistent volume using this class:

```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: byok
  resources:
    requests:
      storage: 5Gi

```

And to conclude, a pod using this volume

```yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: encryptedpvc
  name: nginxencryptedpvc
spec:
  containers:
  - image: nginx
    name: c1
    env:                
    - name: MY_NODE_NAME
      valueFrom:        
        fieldRef:       
          fieldPath: spec.nodeName
    volumeMounts:
    - name: vol
      mountPath: /vol
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:                 
    - name: vol            
      persistentVolumeClaim:
        claimName: azure-managed-disk

```

There's a catch though.
If we use the same encryption id that we used to encrypt the Os Disks, it does not live in the AKS managed resource groups, thus we should grant access to AKS control plane to this resource, as tstated by the error message below

```

Events:
  Type     Reason                Age                     From                                                                                               Message
  ----     ------                ----                    ----                                                                                               -------
  Normal   ExternalProvisioning  3m15s (x26 over 9m10s)  persistentvolume-controller                                                                        waiting for a volume to be created, either by external provisioner "disk.csi.azure.com" or manually created by system administrator
  Normal   Provisioning          37s (x10 over 9m10s)    disk.csi.azure.com_csi-azuredisk-controller-5bb9864b4c-jf85c_5b992153-3dd0-40f7-893c-edd7b6d2ab69  External provisioner is provisioning volume for claim "default/azure-managed-disk"
  Warning  ProvisioningFailed    37s (x10 over 9m9s)     disk.csi.azure.com_csi-azuredisk-controller-5bb9864b4c-jf85c_5b992153-3dd0-40f7-893c-edd7b6d2ab69  failed to provision volume with StorageClass "byok": rpc error: code = Internal desc = Retriable: false, RetryAfter: 0s, HTTPStatusCode: 403, RawError: {"error":{"code":"LinkedAuthorizationFailed","message":"The client '00000000-0000-0000-0000-000000000000' with object id '00000000-0000-0000-0000-000000000000' has permission to perform action 'Microsoft.Compute/disks/write' on scope '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-dfitcfr-dev-tfmodule-aksobjectsdavidfrappar/providers/Microsoft.Compute/disks/pvc-12a558e5-ef54-4f94-a0ed-6e4aca47a625'; however, it does not have permission to perform action 'read' on the linked scope(s) '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg-david.frappart/providers/Microsoft.Compute/diskEncryptionSets/des-aks-david.frappart' or the linked scope(s) are invalid."}}

```

Once it's configured, checking on the pvc, we should see that it does use the encryption set, as expected.

![ilustration6](/assets/aksencryption/aksencryption006.png)

And that's it for encryption of disks in AKS. Note that we could wonder how it works with the Azure File based storage class.
First, there's no dependency with a disk encryption set. 
Second, If we were to look at the csi provider documentation, there is a reference to encryption, but at this point, I'm not sure it allows a byok scenario. 
Another alternative could be to provision Azure storage accounts with a `CMK` and create statically assigned volumes from those storage accounts.
We may look at that in another article ^^.

With all of this, we covered the encrypiton of the storage layer. Let's see how we can add another security layer with host encryption.


### 3.2. End-to-end encryption with host encryption

While disk encryption sets are the answer provided by the Azure platform to give us the capability of bring our own key for the Server Side Encryption, Host encryption provides additional encryption by garanteeing end-to-end encryption between the hosts a.k.a the node pools and the storage layer.

The fact that it's about AKS should not make us forget that the worker nodes are actually virtual machine scake sets.
Which means that the host encryption feature need to be enabled on the compute provider.
This is something that can be checked with the `az feature show` command.

```bash

spike@azure:~$ az feature show --name EncryptionAtHost --namespace Microsoft.Compute
{
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Features/providers/Microsoft.compute/features/encryptionAtHost",
  "name": "Microsoft.compute/encryptionAtHost",
  "properties": {
    "state": "Registered"
  },
  "type": "Microsoft.Features/providers/features"
}

```

If the status is not set to `Registered`, then it need to be register with the `az feature register` command.

Specifically on AKS, by specifying the parameter `--enable-encryption-at-host`, with either `az aks create` or `az aks nodepool add` commands, it becomes possible to have end to end encryption.

As the commands imply, host encryption is only possible on new nodes, or at cluster creation.

The parameter is expecting a boolean, as for the terraform resources `azurerm_kubernetes_cluster` and `azurerm_kubernetes_cluster_node_pool` with the corresponding argument ``.

If the encryption is activated, it's visible in the json definition of the object:

```bash

spike@azure:~$ az aks show -n aks-davidfrappar -g rsg-david.frappart | grep -i enableEncryptionAtHost
      "enableEncryptionAtHost": true,

```

## 4. To summarize

Without a surprise, AKS leverage the Azure platform capabilities when the encryption questions come around:

- Encryption at rest for the storage layer
- Host encryption to ensure end to end encryption from the host to the storage layer

We have the choice to rely on the platform managed encryption which is enabled by default.
We also have th option to prefer scenario where the `CMK` is the way to go. 
About that, there's the operationnal overhead to consider, which requires the management of the encryption key, its rotation and most of all of its hosting in... an Azure Key Vault.
One could wonder what's the gain to host the encryption key in the not so trusted cloud provider, but htat's more a compliance topic, than a technical security topic, which is the aim of this article.
And that's all. See you around!



