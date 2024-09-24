---
layout: post
title:  "Recording sessions with Bastion premium"
date:   2024-09-21 18:00:00 +0200
year: 2024
categories: Security Network
---

Hello!

Today I wanted to take a bit of time to look at a new feature of Azure Bastion, the sessions recording.
It's been a long awaited feature, at least from my side, so I could not miss the opportunity when It came to test this 	&#128526;
Our agenda will be as follow:

1. A rapid review of Azure Bastion concepts
2. The Premium sku
3. Testing session recording

Let's get started!

## 1. A rapid review of Azure Bastion concepts

As its name implies, Azure bastion is... a managed bastion service.
The idea is to provide amean to access Azure virtual machines on SSH or RDP, without requiring an direct exposure on the Internet with a public IP.
Without Azure Bastion, either a VM with a public IP and required flows configured on NSG are required, of a VPN connection is needed to provide access inside the Virtual Network.

![illustration1](/assets/bastionpremium/nobastion.png)

![illustration2](/assets/bastionpremium/bastion.png)

There's plenty of [documentation](https://learn.microsoft.com/en-us/azure/bastion/) on Bastion available. To summarize, we should remember the following:

- A bastion host lives in a subnet in a virtual network. The said subnet ame is imposed to `AzureBastionSubnet` and should be at least a `/26`
- It does have a public IP, that users access through the Azure API. It requires an Entra Id authentication on the tenant, and proper RBAC configuration on both the Bastion host and the target VM
- If NSG are implemented (as it should be) Infrastructure flows are required on the subnet to allow Azure Bastion to work.

In terms of features, aprt from the one that we are interested in, it's important ot get that those are related to the sku. There are 3 skus

- `developper`
- `basic`
- `standard`

and a new `premium` sku, stillin preview currently.

The following table, from the [documentation](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview#sku), details the different features available per sku

| Feature | Developer SKU	| Basic SKU	| Standard SKU	| Premium SKU |
|-|-|-|-|-|
| Connect to target VMs in same virtual network | Yes| Yes	| Yes | Yes |
| Connect to target VMs in peered virtual networks	| No	| Yes	| Yes	| Yes |
| Support for concurrent connections	| No	| Yes	| Yes	| Yes |
| Access Linux VM Private Keys in Azure Key Vault (AKV)	| No	| Yes	| Yes	| Yes |
| Connect to Linux VM using SSH	| Yes	| Yes	| Yes	| Yes |
| Connect to Windows VM using RDP	| Yes	| Yes	| Yes	| Yes |
| Connect to Linux VM using RDP | 	No | 	No	| Yes	| Yes |
| Connect to Windows VM using SSH | 	No| 	No	| Yes	| Yes |
| Specify custom inbound port | No| No | Yes | Yes |
| Connect to VMs using Azure CLI | No | No | Yes	| Yes |
| Host scaling | No | No | Yes | Yes |
| Upload or download files | 	No | No	| Yes	| Yes |
| Kerberos authentication	| No	| Yes	| Yes	| Yes |
| Shareable link| No | No	| Yes	| Yes |
| Connect to VMs via IP address| 	No | No	| Yes	| Yes |
| VM audio output	| Yes	| Yes	| Yes	| Yes |
| Disable copy/paste (web-based clients)| 	No | No	| Yes	| Yes |
| Session recording | No | No | No	| Yes |
| Private-only deployment	| No	|No| 	No	| Yes |

And this table shows the differences in price:

| Azure Bastion sku | Price |
|-|-|
|Azure Bastion Developer	| Free |
|Azure Bastion Basic	| €0.171 per hour |
|Azure Bastion Standard	| €0.261 per hour |
|Additional Standard Instance	|€0.126 per hour |
|Azure Bastion Premium	| €0.405 per hour |
|Additional Premium Instance	| €0.198 per hour |

Let's discuss a bit the new premium sku!

## 2. The premium sku


As mentionned, this new sku is still in preview. But it does bring interesting feature such as the session recording, or the private-only deployment.
We'll focus specifically on the session recording. The private-only dpeloyment will be for another day.

Now about the deployment.

Because it's currently in preview, we have some limit on how we can deploy, or update.
It seems that the portal is currently the only way.

az cli gives the following message for the `create` or `update` command:



```bash

yumemaru@azure$ az network bastion create -n bastiontest -g rsg-spokeAks --sku premium
az network bastion create: 'premium' is not a valid value for '--sku'. Allowed values: Basic, Standard.


yumemaru@azure$ az network bastion update -n bst-spokeAks -g rsg-spokeAks --sku premium
az network bastion update: 'premium' is not a valid value for '--sku'. Allowed values: Basic, Standard.


```

It does seems that the API is not ready yet, as we can see on the [Azure Resource manager Template reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.network/bastionhosts?pivots=deployment-language-arm-template#sku-2)

![illustration3](/assets/bastionpremium/bastionsku.png)

So for now, we are indeed stuck with a portal only deployment.
It's easily done once we located the proper section on the bastion host.
Notice however that we cannot keep the native client with the session recording session. That's a bit of a let down but not totally a surprise.

![illustration4](/assets/bastionpremium/nativeclientfeature.png)


Ok, let's try the session recording now, to understand a bit how it works.

## 3. Testing session recording

Making the assumption that we have an available Bastion host in premium sku, what are our steps to make session recording works?

Let's look at how the feature work.

![illustration5](/assets/bastionpremium/sessionrecording.png)

So, as a managed service, the underlying storage for session recording is... a storage account.
It make sense and allows us to not worry about the storage size.

On the other hand, what if we have a policy of PaaS with Private Endpoint only?
The documentation does not state that it's a supported scenario.
Also, Bastion is not listed in the [trusted Azure service](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security?tabs=azure-portal#grant-access-to-trusted-azure-services) that may be allowed on the Azure storage firewall.

Ok then. 

On the storage part, we add some security with the CORS configuration, for which we allow specifically the Azure Bastion host by its fqdn. 

```go

resource "azapi_update_resource" "storageupdate" {
  type = "Microsoft.Storage/storageAccounts/blobServices@2023-01-01"
  name = "default"
  parent_id = azurerm_storage_account.BstSta.id
  body = jsonencode(
    {
      properties = {
        cors = {
          corsRules = [
            {
              allowedOrigins = ["https://${azurerm_bastion_host.Bastion["spoke"].dns_name}"]
              allowedMethods = ["GET"]
              maxAgeInSeconds = 86400
              allowedHeaders = []
              exposedHeaders = []
            }
          ]
        }
      
      }
    }
  )
}

```

Fun fact, the cors properties do not work taht much if youtry to configure it through the azurerm terraform provider &#128541;.

We also specify a shared access signature on a designated storage container, which will be the repository of our records. We can do this either on the portal or through cli


![illustration6](/assets/bastionpremium/sascreation.png)

```bash

yumemaru@azure$ az storage container generate-sas -n <container_name> --account-name <storage_account_name> --expiry $end --permissions rwlc --start $start --account-key key2 --https-only

```

The SAS must grant `READ`, `WRITE`, `LIST` and `CREATE` permissions.
Also, the expiration date should be far away enough to allow the exploitation of the video.

Once all of that is configured, we can finalize the configuration in the bastion host

![illustration7](/assets/bastionpremium/recording001.png)

![illustration8](/assets/bastionpremium/recording002.png)

![illustration9](/assets/bastionpremium/recording003.png)


And after doing some stuff on a server through the Bastion host, we can view the video


![illustration10](/assets/bastionpremium/recording004.png)

![illustration11](/assets/bastionpremium/recording005.png)

## 4. Summary

So, at last, ession recoridng is available in bastion premium.

Currently, the feature remains in preview, making automation a bit less fluid.
Also, we have to choose between session recording and native client. 
Last, no firewall configuration on the storage account. 

But all in all, it's still pretty neat and answer a quite big demand from security team so it's cool &#128526;	

See you soon!



