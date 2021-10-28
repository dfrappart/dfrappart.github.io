---
layout: post
title:  "Levering Logic apps to do ops stuff on AKS"
date:   2021-10-28 17:28:00 +0200
categories: AKS, LogicApps
---

## 1. Introduction

</br>

In this multi part article, i propose that we have a look on some Ops actions regarding AKS cluster and how, as is proper for a lazy Ops, it is possible to automate some of those actions with Azure services.

The content below was highly inspired by Thomas Stringer who published the basis of the stuff that we will do on his [blog](https://trstringer.com/schedule-aks-start-stop-automatically).
I found that i had to dive in his solution before being able to makes something works and this article is the result of my lacks and struggles.
Nevertheless, kudos to him. He didi help me achieve things i really wanted to do.
And with that let's get going.

</br>

## 2. Review of our need

</br>
So, i mentionned Ops actions on AKS cluster. what are we talking about and why it caused me a few headhaches.
First let's note that i usually look first on PowerShell and Azure Automation runbook to automate stuff in Azure.
That is, once the deployment is completed.

Regarding AKS, it seems that the product team prefers az cli over Azure PowerShell, and features are available first on az cli.
I would like to perform some of the actions available on AKS with az cli such as:

- [start and stop aks cluster](https://docs.microsoft.com/en-us/azure/aks/start-stop-cluster?tabs=azure-cli)
- [rotate aks certificate](https://docs.microsoft.com/en-us/azure/aks/certificate-rotation)

I also do not want to wait for a feature being available on one cli or another, so i will have a look at the API, which by default is the first available interface ^^.
That's why it's one of my first stop when looking on an action on a service. I check the [Rest API reference](https://docs.microsoft.com/en-us/rest/api/azure/).

I can find for AKS the start stop action:

</br>

```http

POST https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerService/managedClusters/{resourceName}/start?api-version=2021-05-01

POST https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerService/managedClusters/{resourceName}/stop?api-version=2021-05-01

```

</br>
and the rotate certificate:

</br>

```http

POST https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerService/managedClusters/{resourceName}/rotateClusterCertificates?api-version=2021-05-01

```

</br>
Obviously, i don't really want to call the api. I could, but i'm not good at it and i prefer to find something easier.
So, how could i do that ?
With an Azure service of course, and this service is in this case [Logic App](https://docs.microsoft.com/en-us/azure/logic-apps/). Let's have a look.

</br>

## 3. Interacting with Azure resource through Azure Logic Apps

</br>
So first thing first, i need a Logic apps.
I won't go in the detail on how to create a blanck logic app, it's quite easy and it can be done many way, such as through terraform for example ^^:

</br>

```bash

resource "azurerm_logic_app_workflow" "LGA" {


  name                                  = "lga${var.LGASuffix}"
  location                              = var.LGALocation
  resource_group_name                   = var.RGName
  integration_service_environment_id    = var.LGAISEId
  logic_app_integration_account_id      = var.LGAIAId
  workflow_schema                       = var.LGASchema
  workflow_version                      = var.LGAWorkflowVersion
  parameters                            = var.LGAParam

  tags = {...
  } 

}

```

</br>

Afterward we have something like that:

</br>

![Illustration 1](/assets/aksops01.png)

</br>

and we can start exploring the logic apps actions.
After configuring a trigger, in our case a timer, we will look upon the Azure Resource Manager.
Since we want to to invoke an operation, the action `Invoke resource operation` seems nice, but so does the `List resources from a Resource Group` to avoid writing something too static:

</br>
  
![Illustration 2](/assets/aksops02.png)

</br>

There is a connection button which will manage all the authentication behind the scene.

</br>

![Illustration 3](/assets/aksops03.png)

</br>

Since i have other connection existing it points to an existing one but let's try another one.
For that, first we will save our logic app and then navigate in the identity section.

</br>
  
![Illustration 4](/assets/aksops04.png)

</br>

We will just activate the system managed identity

</br>

![Illustration 5](/assets/aksops05.png)

</br>

And grant it access on the subsccrption

</br>

![Illustration 6](/assets/aksops06.png)</br>
![Illustration 7](/assets/aksops07.png)

</br>

After that we can go back to the Designer and create an new connection.

</br>

![Illustration 8](/assets/aksops08.png)

</br>

Selecting `connect with managed identity`</br>

![Illustration 9](/assets/aksops09.png)
![Illustration 10](/assets/aksops10.png)

</br>

Navigating in the portal, you may notice the presence of an Azure resource called API Connection

</br>

![Illustration 11](/assets/aksops11.png)
![Illustration 12](/assets/aksops12.png)

Which is, as the name implies, the object behind the scene responsible for allowing the interaction between the logic app managed identity and the ARM API.
As a matter of fact, now that we added the details of the invoke operation:

</br>

![Illustration 13](/assets/aksops13.png)

</br>

let's look on the `code view`

</br>

![Illustration 14](/assets/aksops14.png)

```json

    "parameters": {
        "$connections": {
            "value": {
                "arm_1": {
                    "connectionId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-tra-cpt-AzAuto/providers/Microsoft.Web/connections/arm-1",
                    "connectionName": "arm-1",
                    "connectionProperties": {
                        "authentication": {
                            "type": "ManagedServiceIdentity"
                        }
                    },
                    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Web/locations/westeurope/managedApis/arm"
                }
            }
        }
    }

```

</br>

This section is where the logic apps gets the connection information and matches with the `API Connection`.
Ok, we are done for now, we have a very simple logic app which is should be able to stop our cluster aks `aks-csi1`.
But as i said, we want some dynamic thing, so let's insert a `List Resource From Resource Group` first:

![Illustration 15](/assets/aksops15.png)

Just to be sure, let's trigger our logic app and see what's happens.

</br>

![Illustration 16](/assets/aksops16.png)</br>
![Illustration 17](/assets/aksops17.png)

</br>

It worked!!!
Let's see the Azure resource side:

</br>

In the Activity logs, we can see the Stop action done by our managed identity `lgaaksops`

</br>

![Illustration 18](/assets/aksops18.png)</br>
![Illustration 19](/assets/aksops19.png)

</br>

And the cluster is indeed stopped:

</br>

![Illustration 20](/assets/aksops20.png)

</br>

```bash

PS C:\Users\jubei.yagyu> az aks show -n aks-csi1 -g rsgcsimeetup1 --query powerState
{
  "code": "Stopped"
}
PS C:\Users\jubei.yagyu>

```

</br>

That sound good enough for me now.

Next time we will look how to exploite the Logic app capabilities better, in terms of dynamically getting the resources and of alerting.

Hope you enjoyed this one.
See you ^^
