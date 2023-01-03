---
layout: post
title:  "Levering Logic apps to do ops stuff on AKS - Part 2"
date:   2021-12-16 17:28:00 +0200
categories: AKS Ops
---

Hi!

This is the second part of my serie about AKS and Logic Apps.

In [part 1](https://blog.teknews.cloud/aks/2021/10/28/Levering_Logic_apps_to_do_ops_stuff_on_AKS_part1.html), we just explore how to use a logic app to start an AKS cluster. We were thus capable of validating that the idea is working.
Now we want to go a little further than that.
First, we will add some intelligence in our logic app to get AKS cluster related information dynamically.

And second we wil create another logic app to automate the cluster certificate rotation.

Let's get going!
  
## Table of content

1. Adding intellignce to our Start/Stop AKS logic app
2. Automate AKS Certificate rotation
3. Take Away

## 1. Adding intelligence to our Start/Stop AKS logic app  

Ok, we stopped last time with a workflow like that:  
  
![Illustration 1](/assets/aksops15.png)
  
A very simple workflow, in which we put manually the aks cluster name.

Not very dynamic!  

We did however already added a `List resources by resource group action`.
This is the basis to make our workflow more dynamic.
After running, we can check the row input:  

```json

{
    "method": "get",
    "queries": {
        "x-ms-api-version": "2016-06-01"
    },
    "path": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/resources",
    "host": {
        "connection": {
            "name": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetupkured1/providers/Microsoft.Web/connections/arm"
        }
    }
}

```
  
and output:  
  
```json

{
    "statusCode": 200,
    "headers": {
        "Pragma": "no-cache",
        "x-ms-ratelimit-remaining-subscription-reads": "11996",
        "x-ms-request-id": "ec7132da-ab79-4a40-8eb9-40d531b1fc8b",
        "x-ms-correlation-request-id": "ec7132da-ab79-4a40-8eb9-40d531b1fc8b",
        "x-ms-routing-request-id": "WESTEUROPE:20211215T121809Z:ec7132da-ab79-4a40-8eb9-40d531b1fc8b",
        "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
        "X-Content-Type-Options": "nosniff",
        "Timing-Allow-Origin": "*",
        "x-ms-apihub-cached-response": "true",
        "Cache-Control": "no-cache",
        "Date": "Wed, 15 Dec 2021 12:18:09 GMT",
        "Content-Length": "7653",
        "Content-Type": "application/json",
        "Expires": "-1"
    },
    "body": {
        "value": [
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.ContainerService/managedClusters/aks-1",
                "name": "aks-1",
                "type": "Microsoft.ContainerService/managedClusters",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Insights/activityLogAlerts/malt-ListAKSAdminCredsEvent-aks-1",
                "name": "malt-ListAKSAdminCredsEvent-aks-1",
                "type": "Microsoft.Insights/activityLogAlerts",
                "location": "global",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Insights/metricalerts/malt-NodeCPUPercentageThreshold-aks-1",
                "name": "malt-NodeCPUPercentageThreshold-aks-1",
                "type": "Microsoft.Insights/metricalerts",
                "location": "global",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Insights/metricalerts/malt-NodeDiskPercentageThreshold-aks-1",
                "name": "malt-NodeDiskPercentageThreshold-aks-1",
                "type": "Microsoft.Insights/metricalerts",
                "location": "global",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Insights/metricalerts/malt-NodeWorkingSetMemoryPercentageThreshold-aks-1",
                "name": "malt-NodeWorkingSetMemoryPercentageThreshold-aks-1",
                "type": "Microsoft.Insights/metricalerts",
                "location": "global",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Insights/metricalerts/malt-UnschedulablePodCountThreshold-aks-1",
                "name": "malt-UnschedulablePodCountThreshold-aks-1",
                "type": "Microsoft.Insights/metricalerts",
                "location": "global",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/uai-agwagicmeetup1",
                "name": "uai-agwagicmeetup1",
                "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.ManagedIdentity/userAssignedIdentities/uaiaks-1",
                "name": "uaiaks-1",
                "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "tflab",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "tfmodule",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/applicationGateways/agwagicmeetup1",
                "name": "agwagicmeetup1",
                "type": "Microsoft.Network/applicationGateways",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/networkSecurityGroups/nsg-agwagicmeetup1",
                "name": "nsg-agwagicmeetup1",
                "type": "Microsoft.Network/networkSecurityGroups",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/networkSecurityGroups/nsg-subBEagicmeetup1",
                "name": "nsg-subBEagicmeetup1",
                "type": "Microsoft.Network/networkSecurityGroups",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/networkSecurityGroups/nsg-subFEagicmeetup1",
                "name": "nsg-subFEagicmeetup1",
                "type": "Microsoft.Network/networkSecurityGroups",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/publicIPAddresses/pubip-agwagicmeetup1",
                "name": "pubip-agwagicmeetup1",
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                    "name": "Standard"
                },
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            },
            {
                "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.Network/virtualNetworks/vnetagicmeetup1",
                "name": "vnetagicmeetup1",
                "type": "Microsoft.Network/virtualNetworks",
                "location": "westeurope",
                "tags": {
                    "CostCenter": "labaks",
                    "Country": "fr",
                    "Environment": "lab",
                    "ManagedBy": "Terraform",
                    "Project": "agic",
                    "ResourceOwner": "That would be me"
                }
            }
        ]
    }
}

```
  
We want to get only the AKS cluster so we will add a filter:
  
![Illustration 2](/assets/aksops21.png)  
  
The logic app will gently proposes us some dynamic content. We will put the `value` of `List resource by resource group` action in the `From` and filter on the `Type` that we will want to be equal to **Microsoft.containerService/ManagedCluster**
  
![Illustration 3](/assets/aksops22.png)  
  
The result output will be the AKS cluster:  

```json

{
    "body": [
        {
            "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup1/providers/Microsoft.ContainerService/managedClusters/aks-1",
            "name": "aks-1",
            "type": "Microsoft.ContainerService/managedClusters",
            "location": "westeurope",
            "tags": {
                "CostCenter": "labaks",
                "Country": "fr",
                "Environment": "lab",
                "ManagedBy": "Terraform",
                "Project": "agic",
                "ResourceOwner": "That would be me"
            }
        }
    ]
}

```
  
With that we can now change the `Invoke resource operation`with its static value to something more dynamic. To do so, we first add a `For each` control action
  
![Illustration 4](/assets/aksops23.png)  
  
And select as the output the `Body`of our `filter` action
  
![Illustration 5](/assets/aksops24.png)  
  
Then we put our `Invoke resource operation` inside our `For each` and use the dynamic content to get the name
  
![Illustration 6](/assets/aksops25.png)  
  
![Illustration 7](/assets/aksops26.png)  

We also want to add a notification so we can add Teams `Post message` action before and after. Again we use dynamic content in the message
  
![Illustration 8](/assets/aksops27.png)  
  
![Illustration 9](/assets/aksops28.png)  
  
And that's it, we have a nice Logic app schedule to start our cluster(s) in a specified resource group.
It will send a teams message like this so that ops are sure that everyone knows  
  
![Illustration 10](/assets/aksops29.png)  
  
Now, to have the same logic app for the `stop` action, we just have to reproduce the same logic app and change the `Invoke resource operation`.

Or, the lazy way, to copy paste the content of the code view in a new logic app.

```json

{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {...},
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {...}
    },
    "parameters": {
        "$connections": {
            "value": {
                "arm": {
                    "connectionId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/$rgname/providers/Microsoft.Web/connections/arm",
                    "connectionName": "arm",
                    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Web/locations/westeurope/managedApis/arm"
                },
                "office365": {
                    "connectionId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/$rgname/providers/Microsoft.Web/connections/office365",
                    "connectionName": "office365",
                    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Web/locations/westeurope/managedApis/office365"
                },
                "teams": {
                    "connectionId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/$rgname/providers/Microsoft.Web/connections/teams-2",
                    "connectionName": "teams-2",
                    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Web/locations/westeurope/managedApis/teams"
                }
            }
        }
    }
}

```

Just beware, since we are using managed identity, to activate the feature before the copy paste.  
  
![Illustration 11](/assets/aksops30.png)  
  
## 2. Automate AKS Certificate rotation  

So now we have a logic app to automate start and stop of AKS cluster.
another action that we would like to automate is the `rotateCertificates` action.

As mentioned in the [documentation](https://docs.microsoft.com/en-us/azure/aks/certificate-rotation), the renewal of the certificate before the expiration is a new feature in GA, and rolled out until february.

However, we may need to trigger the renewal manually.

On the logic app part, we will again use an `Invoke resource operation` and specify the `rotateCertificates` operation.
This time, since this action is not without impact, a validation could be nice.
Logic apps come with an Outlook action which answer to this:  
  
![Illustration 12](/assets/aksops31.png)  

Depending on the answer, we will have to implement a different path in the logic app. We will do that with a `Condition` control action:

![Illustration 13](/assets/aksops32.png)  
  
Note the condition in code view:  

```json

    "expression": {
        "and": [
            {
                "equals": [
                    "@body('Send_approval_email')?['SelectedOption']",
                    "Approve"
                ]
            }
        ]
    }

```
  
When the workflow is triggered, we get a nice email such as this one:  

![Illustration 14](/assets/aksops33.png)  
  
Which we approve or rject to complete the workflow:
  
![Illustration 15](/assets/aksops34.png)  
  
![Illustration 16](/assets/aksops35.png)  
  
And that's it ^^

## 3. Take away

In this 2 part article, we were able to create logic apps to manage Ops activities on AKS cluster.
SO we have two main area of benefits here.
The first one is obviosly on the AKS operations, for which we can rely on the Azure Resource Manager actions available in Logic App designer.
The second is whazt we found out while designing those logic apps.
We do have the Azure Resource Manager actions, and using the Data Operations and the Control actions, we can leverage an automated workflow, on any Azure resources, without managing password thanks to the support for Managed Identity.

To conclude, another interesting operations on AKS is the `runCommand`. I will probably have a look in the near futur. On that, see you soon.
