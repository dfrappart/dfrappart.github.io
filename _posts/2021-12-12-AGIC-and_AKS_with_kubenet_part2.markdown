---
layout: post
title:  "AGIC and AKS with Kubenet - Part 2"
date:   2021-12-12 15:30:00 +0200
categories: AKS Network
---


In the second part of our AGIC & AKS Kubenet serie, we will look at the installation part again, but with a more customizable way.

Instead of the add-on, we will look to the **Open Source Project** (which is the origin of the add-on btw) and look how to deploy through **Helm** (and also terraform)

Again, I hope you'll enjoy it.  

## Table of content

1. Installing the control freak way - The AGIC OSS project, Helm and Pod Identity
2. Azure Prerequisites with the help of Terraforml
3. Helm for installation inside Kubernetes
4. Take away
  
## 1. Installing the control freak way - The OSS project, Helm and Pod Identity  
  
As the title implies, we are now going to go back to the begining of the AGIC feature, a.k.a the Open source Project at the origin of the add-on.

While technical content can be found on [Azure Documentation](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview), I would recommand to prefer refering to the project documention available on [github](https://azure.github.io/application-gateway-kubernetes-ingress/
).
  
To summarize, we want to install AGIC by ourselve, meaning as kubernetes componants. But we also identified that to work, specifically in a kubenet model AKS, we have prerequisites in terms of Azure resources and authorizations.
Also, something that is quite not clear with the add-on but explicitly mentionned in the OSS documentation, we need to be able to interact with the Azure control plane from the kubernetes control plane.

That required some Azure Role assignments and a way to interact with Azure AD from within Kubernetes.

And what better to do that than relying on the Azure AD integration and the Pod Identity feature?

The outline of what we will perform for the manual installation is:  
  
- On the Azure control plane:

  - RBAC Assignlment for Managed Identity
  - UDR association

- On the Kubernetes control plane

  - Setup Pod Identity
  - Deploy AGIC

We will use terraform azurerm provider for Azure configuration, and helm chart and terraform provider for helm to perform full AKS AGIC configuration.
  
## 2. Azure Prerequisites with the help of Terraform
  
As we just said, we need to perform a few preparations for AGIC to work, especially in an AKS with Kubenet environment.

First, about the networking. Remember that Kubenet, because its isolate the pods network from the Azure Virtual Network, requires a User Defined Route which is mamanged by AKS control plane. While it would bepossible to bring your own route, I found that it is a burden since we would need to update this route manually each time a node is added following an autoscaling event on the node(s) pool(s).  
  
![Illustration 1](/assets/agic014.png)  
  
The tricky part is to get this route id dynamically. The fact that it is associated to the subnet hosting the node pools is the solution, since we can get the associated route table as a property of the subnet:  
  
```bash

PS C:\Users\jubei.yagyu> az network vnet subnet show -n subFEagicmeetup2 --vnet-name vnetagicmeetup2 -g rsgagicmeetup2 --query routeTable
{
  "disableBgpRoutePropagation": null,
  "etag": null,
  "id": "/subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-dfitcfr-lab-agic-aksobjects2/providers/Microsoft.Network/routeTables/aks-agentpool-27051013-routetable",
  "location": null,
  "name": null,
  "provisioningState": null,
  "resourceGroup": "rsg-dfitcfr-lab-agic-aksobjects2",
  "resourceGuid": null,
  "routes": null,
  "subnets": null,
  "tags": null,
  "type": null
}

```
  
Taking this fact, the route table id is available  as a property of the subnet data source to use in the route table association resource:  

```bash

resource "azurerm_subnet_route_table_association" "agwtoaksroutetable" {
  subnet_id                               = data.azurerm_subnet.AGWSubnet.id
  route_table_id                          = data.azurerm_subnet.AKSSubnet.route_table_id
}

```

And that's it for the route configuration. Let's go on with the other prerequisites.

You may remember that follwoing the add-on installation, a User Assigned Identity was created as part of the AKS resources:  
  
![Illustration 2](/assets/agic009.png)  
  
Since we are doing it manually, we need to create this identity. That is, the UAI and the appropriate role assignment, which is, with tha add-on, `Contributor` on the resource group where the apllication gateway lives. We can do it as below:

```bash


resource "azurerm_user_assigned_identity" "UAI" {
  resource_group_name                   = var.TargetRG
  location                              = var.TargetLocation

  name                                  = "uai${lower(var.UAISuffix)}"



  tags = {...}
}


resource "azurerm_role_assignment" "TerraAssignedBuiltin" {
  scope                               = var.RBACScope
  role_definition_name                = var.BuiltinRoleName
  principal_id                        = azurerm_user_assigned_identity.terraUAI.principal_id
}

```
  
Note that I use a module that I created for this kind of use and it's available [here](https://github.com/dfrappart/Terra-AZModuletest/tree/master/Custom_Modules/Kube_UAI)

Regarding AGIC prerequisites, we have it. Now let's move on to the Kubernetes control plane and have some fun with Helm.  
  
## 3. Helm for installation inside Kubernetes
  
We are now at the point of installing AGIC from the Helm chart. Before going further, let's discuss about Helm.  

Some people, and I am still part of those sometimes, have a tendency to refuse by default any talk about Helm, invoking security questions.
The power of Helm is double edged, as for all powerfull tools, and it can indeed bring security questions, when a chart is taken from a helm repository, without further analysis.  

On the other hand, a Helm chart is specifically designed to package complex application with a way to use paramters for custom configuration.
AGIC is specifically a quite complex feature, in the way that it connect Azure control plane and Kubernetes control plane. Pod Identity, which we will need to make our AGIC deployment works, is also this kind of feature.  

So I guess that the best answer **IMHO** is to take the time to look at the Helm chart, in terms of where it comes from, and which parameters are the best for a secure deployemnt.
That being said, it is kind of out of the scope of this article, but let's just say that we will only use Helm chart from Microsoft initiated Open Source Project for specifically AKS.

So I will make the assumption that those Helm chart are secure ^^  
  
### 3.1. Pod Identity requirement
  
As mentionned before, we need pod identity to provide an identity to AGIC inside of Kubernetes.
For more details about Pod Identity, you can either check the documentation or this [article](https://blog.teknews.cloud/aks/2021/02/10/podidentityjourney.html) I wrote last year.
As a reminder, the AKS cluster Identities require the roles **Managed Identity Operator** and **Virtual Machine Contributor**.
Since we are in a kubenet-based network environment, we have to specify that on the installation.
Which gives us the following parameters pushed to the helm chart (through terraform remember ^^):

```bash

variable "HelmPodIdentityParam" {
  type                  = map
  description            = "A map used to feed the dynamic blocks of the pod identity helm chart"
  default                = {

      "set1" = {
        ParamName             = "nmi.allowNetworkPluginKubenet"
        ParamValue            = "true"

    }
      "set2" = {
        ParamName             = "installCRDs"
        ParamValue            = "true"

    }

  }

}

```
  
And the following resource to deploy the chart:  
  
```bash

resource "helm_release" "podidentity" {
  name                                = "podidentity"
  repository                          = "https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts"
  chart                               = "aad-pod-identity"
  version                             = var.PodIdChartVer
  namespace                           = "podidentity"
  create_namespace                    = true


  dynamic "set" {
    for_each                          = var.HelmPodIdentityParam
    iterator                          = each
    content {
      name                            = each.value.ParamName
      value                           = each.value.ParamValue
    }

  }


}

```
  
Now let's see the AGIC install.

### 3.2. Installing AGIC
  
Similarly to what we just did, we have to install AGIC through its available helm chart. There are a few tricks to make it works however.

We install Pod Identity because AGIC will use one to interact with the Application Gateway. That's why we assigned the **Contributor** role to a managed identity in the prerequisites.
We do not do that with the add-on because AKS managed for us the creation of the managed identity and the role assignment.

We are pretty clear about the Azure control plane. Now about the Kubernetes control plane, following the documentation, we rely on the yaml config file template and pass it to the helm chart deployment. This file looks like that:  

```yaml

# This file contains the essential configs for the ingress controller helm chart

# Verbosity level of the App Gateway Ingress Controller
verbosityLevel: 3

################################################################################
# Specify which application gateway the ingress controller will manage
#
appgw:
    subscriptionId: ${subid}
    resourceGroup: ${rgname}
    name: ${agicname}
    usePrivateIP: false

    # Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
    # This prohibits AGIC from applying config for any host/path.
    # Use "kubectl get AzureIngressProhibitedTargets" to view and change this.
    shared: false

################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
# Leaving this variable out or setting it to blank or empty string would
# result in Ingress Controller observing all acessible namespaces.
#
# kubernetes:
#   watchNamespace: <namespace>

################################################################################
# Specify the authentication with Azure Resource Manager
#
# Two authentication methods are available:
# - Option 1: AAD-Pod-Identity (https://github.com/Azure/aad-pod-identity)
armAuth:
    type: aadPodIdentity
    identityResourceID: ${PodIdentityId}
    identityClientID:  ${PodIdentityclientId}

## Alternatively you can use Service Principal credentials
# armAuth:
#    type: servicePrincipal
#    secretJSON: <<Generate this value with: "az ad sp create-for-rbac --subscription <subscription-uuid> --sdk-auth | base64 -w0" >>

################################################################################
# Specify if the cluster is RBAC enabled or not
rbac:
    enabled: ${IsRBACEnabled}

```
  
Using the template data source, we can inject the proper value:  

```bash

data "template_file" "agicyamlconfig" {
  template                                 = file("./template/agicyamlconfig.yaml")
  vars = {
    subid                                   = ""
    rgname                                  = ""
    agicname                                = ""
    PodIdentityId                           = ""
    PodIdentityclientId                     = ""
    IsRBACEnabled                           = true
  }

}

```

Afterward, it's the helm chart:  

```bash

resource "helm_release" "agic" {
  name                                = "agic"
  repository                          = "https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/"
  chart                               = "ingress-azure"
  version                             = var.AgicChartVer
  namespace                           = "agic"
  create_namespace                    = true

  values = [data.template_file.agicyamlconfig.rendered]

  depends_on = [
    helm_release.podidentity
  ]


}

```
  
Note the `values = [data.template_file.agicyamlconfig]` that reference the template.

After deployment, we should get the following:

- 1 namespace for pod identity with  
![Illustration 3](/assets/agic015.png)
  
  - 1 deployment MIC  
![Illustration 4](/assets/agic016.png)  
  
  - 1 daemonset NMI  
![Illustration 5](/assets/agic017.png)  
- 1 namespace AGIC with

  - 1 deployment agic  
![Illustration 6](/assets/agic018.png)  
  
A good practice would be to check the logs:  

```bash

k logs -n agic deployment/agic-ingress-azure
I1212 11:18:58.607908       1 utils.go:115] Using verbosity level 3 from environment variable APPGW_VERBOSITY_LEVEL
I1212 11:18:58.924050       1 environment.go:248] KUBERNETES_WATCHNAMESPACE is not set. Watching all available namespaces.
I1212 11:18:58.924078       1 main.go:117] Using User Agent Suffix='agic-ingress-azure-56bcb97f74-kb6fj' when communicating with ARM
I1212 11:18:58.945376       1 main.go:136] Appication Gateway Details: Subscription="00000000-0000-0000-0000-000000000000" Resource Group="rsgagicmeetup2" Name="agwagicmeetup2"
I1212 11:18:58.945480       1 httpserver.go:57] Starting API Server on :8123
I1212 11:18:58.955593       1 auth.go:46] Creating authorizer from Azure Managed Service Identity
E1212 11:19:09.662013       1 client.go:170] Code="ErrorApplicationGatewayForbidden" Message="Unexpected status code '403' while performing a GET on Application Gateway. You can use 'az role assignment create --role Reader --scope /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup2 --assignee 3ba035ff-dfd2-4d8f-add4-5767928e0d67; az role assignment create --role Contributor --scope /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup2/providers/Microsoft.Network/applicationGateways/agwagicmeetup2 --assignee 3ba035ff-dfd2-4d8f-add4-5767928e0d67' to assign permissions. AGIC Identity needs atleast has 'Contributor' access to Application Gateway 'agwagicmeetup2' and 'Reader' access to Application Gateway's Resource Group 'rsgagicmeetup2'." InnerError="network.ApplicationGatewaysClient#Get: Failure responding to request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error. Status=403 Code="AuthorizationFailed" Message="The client '70492b56-d752-4e87-80c2-077347902736' with object id '70492b56-d752-4e87-80c2-077347902736' does not have authorization to perform action 'Microsoft.Network/applicationGateways/read' over scope '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsgagicmeetup2/providers/Microsoft.Network/applicationGateways/agwagicmeetup2' or the scope is invalid. If access was recently granted, please refresh your credentials.""
I1212 11:19:09.662036       1 retry.go:33] Retrying in 10s
I1212 11:19:19.774804       1 main.go:183] Ingress Controller will observe all namespaces.
I1212 11:19:19.904694       1 context.go:138] k8s context run started
I1212 11:19:19.904728       1 context.go:188] Waiting for initial cache sync
I1212 11:19:20.014721       1 context.go:201] Initial cache sync done
I1212 11:19:20.014820       1 context.go:202] k8s context run finished
I1212 11:19:20.015101       1 worker.go:39] Worker started
I1212 11:19:20.262721       1 mutate_app_gateway.go:177] BEGIN AppGateway deployment
I1212 11:19:21.192853       1 client.go:201] OperationID='112c4f8a-4407-45c2-bb57-1944b8fa00ca'
I1212 11:20:06.623445       1 mutate_app_gateway.go:185] Applied generated Application Gateway configuration
I1212 11:20:06.623473       1 mutate_app_gateway.go:200] cache: Updated with latest applied config.
I1212 11:20:06.624647       1 mutate_app_gateway.go:204] END AppGateway deployment
I1212 11:20:06.624666       1 controller.go:151] Completed last event loop run in: 46.609504527s

```

Everything seems fine, after a refresh. that's because I created the managed identity for AGIC in the same terraform config as the helm deployment.

## 4. Take away
  
It's time to wrap up this 2nd part already.  
To conclude, one interesting fact:
AGIC is a candidate for Pod Identity. That's exactly what we did in this article.
Second, we performed a full unmanaged installation of AGIC with its requirement `Pod Identity`, and well, there are a lot of moving part.
I guess that the reason to use the helm chart instead of the add-on should really be discussed thoroughly.
Probably, the capabilities of Pod Identity may give us more segregation options in the Kubernetes control plan than the AGIC add-on.  

but that's up to you ^^

Coming soon, a discussion about deploying AGIC through the terraform provider, because now it's available, and maybe a few sample of ingress, but since it's really Kubernetes stuff, i'm not really in my game with taht, so i'll see.

Thanks for taking the time to read this.

See you soon!
