---
layout: post
title:  "Hands on Azure Key Vault Provider for Secrets Store CSI Driver Part 2"
date:   2022-05-19 01:45:00 +0200
categories: AKS
---

Hello again!

This article is about the Azure Key Vault Provider for Secret Store CSI Driver.
This is part 2, in which instead on focusing on the Azure part, we will look into the details of the Kubernetes parts.
  
## Table of content

1. About our playground
2. Basic config
3. Secret rotation from the key vault
4. Select a version of the key vault secret
5. Switching a kubernetes secret with a CSI volume
6. A look at the way it works with the add-on
7. Before leaving


## 1. About our playground

Obviously, we will use an AKS cluster to perform our test.

Also because it's fun, we will use configuration with pod identity first, before doing a few tests with the `useVMManagedIdentity` set to true, discussed in part 1.

## 2. Basic config

In this first use case, we will take a simple pod on which we will mount a CSI volume refering to our secret store.

This secret store is the key vault that we have below:

![Illustration 6](/assets/csisecret/csisecret006.png)
  
And specifically, we want to mount this secret:

![Illustration 7](/assets/csisecret/csisecret007.png)
  
That being said, what should we start with?

Well first let's tell our Kubernetes cluster that it will use the Key Vault we talk about.

For that we will need a yaml manifest refering to our Key vault like that:  
  
```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: akvkv-subsetupconsuluaicsitest1
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"               
    userAssignedIdentityID: e12b1b66-8c1f-4c6b-9e7f-efa3b4406c11
    keyvaultName: akvkv-subsetupconsul
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-csisecret1
          objectAlias: kvs-csisecret1            
          objectType: secret                    
          objectVersion:   
    tenantId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   

```

In this manifest, we can see `keyvaultName: akvkv-subsetupconsul` refering to the Azure Key Vault.
Also the `cloudName: ""` because by default we are using Azure Public Cloud, so we don't need to specify it.

A kubectl command will allows us to verify that the secretProviderClass is now available:

```bash

kubectl get secretproviderclass

NAME                              AGE
akvkv-subsetupconsuluaicsitest1   41h

```
  
Now, note that we specify a `usePodIdentity: "true"`, and then a `userAssignedIdentityID: e12b1b66-8c1f-4c6b-9e7f-efa3b4406c11`.

As mentionned earlier, we have a cluster that leverage Pod Identity and we did created a User Assigned Identity whose client id is *e12b1b66-8c1f-4c6b-9e7f-efa3b4406c11*:

![Illustration 8](/assets/csisecret/csisecret008.png)
  
![Illustration 9](/assets/csisecret/csisecret009.png)

Let's just stop here for one instant. This identity will be used to get access to the Key Vault.

That means on the Azure control plane, we do need to grant it access.

That's what we do by using, in this example, an access policy bound to the managed identity:  

![Illustration 10](/assets/csisecret/csisecret010.png)

Respecting Least Privilege, we only grant this identity the `Get` and `List` verbs in the access policy.

We're nearly there now.

Let's have a pod running, but with a mounted volume refering tou our secret store:

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-akvkv-subsetupconsuluaicsitest1
  labels:
    aadpodidbinding: uaicsitest1-binding
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: akvkv-subsetupconsuluaicsitest1

```

We can see the `volumes` part with a csi driver refering to `driver: secrets-store.csi.k8s.io` and the secret provider class refering to our Key Vault `secretProviderClass: akvkv-subsetupconsuluaicsitest1`.

Also, I said that we are using pod identity, so we have an additional binding to the managed identity which we granted access to the key vault earlier. The label `aadpodidbinding: uaicsitest1-binding` effectively bind the pod to the pod identity object define with the yaml maniest below:

```yaml

apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: uaicsitest1-binding
spec:
  azureIdentity: uaicsitest1
  selector: uaicsitest1-binding
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: uaicsitest1
spec:
  type: 0
  resourceID: /subscriptions/00000000-0000-0000-0000-00000000000/resourceGroups/rsg-consul-spkaks/providers/Microsoft.ManagedIdentity/userAssignedIdentities/uaicsitest1
  clientID: e12b1b66-8c1f-4c6b-9e7f-efa3b4406c11

```
  
Again, we can find that the `clientID` in the `AzureIdentity` object is the client id of the managed identity.

It gives us the following result when checking the mounted secret on the pod:

```bash

kubectl exec pod-akvkv-subsetupconsuluaicsitest1 -- cat /mnt/secrets-store/kvs-csisecret1

!Aaok17)<NE]%9r3

```

Which is the value of our secret in the Key Vault

![Illustration 11](/assets/csisecret/csisecret011.png)

## 3. Secret rotation from the key vault

Ok, now what about rotation?

We did specify the feature to be active at the installation:

```json

      "set2" = {
        ParamName             = "secrets-store-csi-driver.enableSecretRotation"
        ParamValue            = "true"

    }

      "set3" = {
        ParamName             = "secrets-store-csi-driver.rotationPollInterval"
        ParamValue            = "1m"

    }

```

So we just have to test it right?

Let's just add a new value to our secret:

![Illustration 12](/assets/csisecret/csisecret012.png)
  
![Illustration 13](/assets/csisecret/csisecret013.png)

And now just check the pod created earlier:

```bash

kubectl exec pod-akvkv-subsetupconsuluaicsitest1 -- cat /mnt/secrets-store/kvs-csisecret1

Thisisanewsecretversion^^

```

It should be ok after 1 minute since the poll interval was defined like that.
So it's working quite nicely and avoid the password synced in the pod definition itself, or in a secret. No excuse now ^^.
  
## 4. Select a version of the secret

Another test that could be of interest is to managed the secret version from the keyvault.

Remember, there was this parameter in the secretProviderClass called `objectVersion`.

Let's check our Key Vault secret to get the previous version of our secret:

![Illustration 14](/assets/csisecret/csisecret014.png)

Now that we have the version, let's just update our secretProviderClass:

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: akvkv-subsetupconsuluaicsitest1
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"               
    userAssignedIdentityID: e12b1b66-8c1f-4c6b-9e7f-efa3b4406c11
    keyvaultName: akvkv-subsetupconsul
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-csisecret1
          objectAlias: kvs-csisecret1            
          objectType: secret                    
          objectVersion: 49d2813e422642dfa46f852c5447681e  # This is what we changed   
    tenantId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx 

```

And re-apply the manifest with a `kubectl apply -f <secretProviderClassFile.yaml>`

wWthout surprise, we now have the previous version of the secret mounted in our pod:

```bash

kubectl exec pod-akvkv-subsetupconsuluaicsitest1 -- cat /mnt/secrets-store/kvs-csisecret1

!Aaok17)<NE]%9r3

```

We just need to remember for future update that we fixed a version of this secret.

That's all for basics things. Let's have a look at a sample application.

## 5. Switching a kubernetes secret with a CSI volume

In this section, we want to take an existing application with different micro-services and an Azure SQL backend.

We refer to the Driving application that can be found [here](https://github.com/Azure-Samples/MyDriving) and that is used as a sample app in the [OpenHack](https://github.com/Microsoft-OpenHack/containers_artifacts) from Microsoft.

In our environment, we have already deployed the application and we can access the web ui of the application which looks like that:  
  
![Illustration 15](/assets/csisecret/csisecret015.png)  
  
Clicking on **User Profile** will redirect us to this page:  

![Illustration 16](/assets/csisecret/csisecret016.png)  
  
Which, behind the hood is provided by a Kubernetes service and a Kubernetes deployment:

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: userprofile
  name: userprofiledeploy
  namespace: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: userprofile
  strategy: {}
  template:
    metadata:
      labels:
        app: userprofile
    spec:
      containers:
      - image: acrbaqhp.azurecr.io/tripinsights/userprofile:1.0
        name: userprofile
        env:
        - name: SQL_SERVER
          valueFrom:
            configMapKeyRef:
              name: userprofileconfigmap
              key: SQL_SERVER
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: userprofileconfigmap
              key: PORT
        - name: SQL_USER
          valueFrom:
            secretKeyRef:
              name: tripsecret
              key: SQL_USR
        - name: SQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: tripsecret
              key: pwd    
        resources: {}
status: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: userprofile
  name: userprofilesvc
  namespace: api
spec:
  ports:
  - port: 8082
    protocol: TCP
    targetPort: 8082
  selector:
    app: userprofile
status:
  loadBalancer: {}

```

The backend defined by the environment variable `SQL_SERVER` is provided through a configmap, and the password through a kubernetes secret:

```yaml

apiVersion: v1
data:
  SQL_SERVER: mssqlconsul.database.windows.net
  PORT: "8082"
kind: ConfigMap
metadata:
  name: userprofileconfigmap
  namespace: api
---
apiVersion: v1
data:
  SQL_USR: <base64encodedusername>
  pwd: <base64encodedpasswird>
kind: Secret
metadata:
  name: tripsecret
  namespace: api

```

What happen if we remove the part referencing the secret for the sql password:

```yaml

        env:
===================truncated==================
#Remove the part below
#        - name: SQL_PASSWORD
#          valueFrom:
#            secretKeyRef:
#              name: tripsecret
#              key: pwd 

```

Well, the service cannot access the SQL database anymore and we have an error like this:  
  
![Illustration 17](/assets/csisecret/csisecret017.png)
  
Ok, now let's swith to a secret in the key vault.  

We have the folloing secret which is the value of the SQL password:

![Illustration 18](/assets/csisecret/csisecret018.png)

We can thus create a new secretProviderClass, in the same namespace as the application deployment:

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: akvkv-ohtest
  namespace: api
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"               
    userAssignedIdentityID: 42bcf026-c2f7-42e0-a325-396c043cf0fd
    keyvaultName: akvkv-subsetupconsul
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-mssqladminpwd-consul
          objectAlias: SQL_PASSWORD            
          objectType: secret                    
          objectVersion:        
    tenantId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   

```

Note that we specified an `objectAlias: SQL_PASSWORD` because the secret in the application is reffered to with these name.

And then add the following section in the deployment:  

```yaml
===================truncated==================
    spec:
# Add a volume refering to the secretProviderClass
      volumes:
      - name: sqlpwd
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: akvkv-ohtest
      containers:
      - image: acrbaqhp.azurecr.io/tripinsights/userprofile:1.0
        name: userprofile
# Moint the volume in the pod definition
        volumeMounts:
          - mountPath: /secrets
            name: sqlpwd
            readOnly: true

```

The path configured with `mountPath: /secrets` we got from the application documentation [here](https://github.com/Microsoft-OpenHack/containers_artifacts/blob/main/src/userprofile/README.md):

![Illustration 19](/assets/csisecret/csisecret019.png)  

And last, because we are using pod identity we need to add the label `aadpodidbinding: uaicsitest2-binding` binding the pod to a user assigned managed identity which has access to the key vault.

After a `kubectl apply -f` with the manifes file, we can now access the SQL server through the secret stored in the Key Vault and the **User Profile** page is accessible again:

![Illustration 16](/assets/csisecret/csisecret016.png)  

And that's all for now. Sure, we could do this for all the other part of the application, but the concepts are here.

## 6. A look at the way it works with the add-on

Before leaving, let's have a last example, this time with the add-on.

This time, we will define our `secretProviderClass` with the parameter `useVMManagedIdentity: "true"` and the `userAssignedIdentityID:` with the value of the client Id from the Managed Identity provided by the add-on:  

![Illustration 20](/assets/csisecret/csisecret020.png)

We need to grant this Managed Identity access to the Key Vault

![Illustration 21](/assets/csisecret/csisecret021.png)

The Secret Provider Class is defined this way:  

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: akvkv-subsetupconsul-withaddon
spec:
  provider: azure
  parameters:
    useVMManagedIdentity: "true"               
    userAssignedIdentityID: f2218db0-9455-4f0b-8d93-8299cfb874a0 # Client Id of the Managed Identity provisionned at the Add-on installation
    keyvaultName: akvkv-subsetupconsul
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-csisecret1
          objectAlias: kvs-csisecret1            
          objectType: secret                    
          objectVersion:    
    tenantId: e0c45235-95fe-4bd6-96ca-2d529f0ebde4  

```

And we define a simple pod to mount a secret as a volume like this:

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-akvkv-subsetupconsuluaicsitest1
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: akvkv-subsetupconsul-withaddon

```

If we check the volume in the pod, without surprise we have access to the secret:  

```bash

kubectl exec pod-akvkv-subsetupconsuluaicsitest1 -- cat /mnt/secrets-store/kvs-csisecret1

Thisisanewsecretversion^^

```

And as simple as that, just by specifying the Managed Identity that the add-on configured on the cluster, we can get access to the key vault once it is granted on both the Azure control plane and the Kubernetes control plane.

Sure it is simpler than the previous example with pod identity. Noadditional component sich as Pod Identity and access from pod to Key Vault through an Managed IDentity seemingly associated to the secret store object.

Let's dig further. Remember, we had, including the one for the CSI Addon, 4 Azure Managed Identities in our environment:

- azurepolicy-aks-consul
- azurekeyvaultsecretsprovider-aks-consul
- aks-consul-agentpool
- omsagent-aks-consul

Those identities are linked to the vmss that are behind our node pools:

![Illustration 22](/assets/csisecret/csisecret022.png)

If we take, let's say the json definition of the one for the policy extension, we get the following:

```json

{
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/rsg-dfitcfr-lab-cnitest-aksobjectsconsul/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azurepolicy-aks-consul",
    "name": "azurepolicy-aks-consul",
    "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
    "location": "eastus",
    "tags": {},
    "properties": {
        "tenantId": "e0c45235-95fe-4bd6-96ca-2d529f0ebde4",
        "principalId": "d27bbaf8-0fab-4fad-868e-a90c1d42e68b",
        "clientId": "5e69b6e7-a065-4591-abd3-2f7356bf511e"
    }
}

```

Let's keep the value of the client id `"clientId": "5e69b6e7-a065-4591-abd3-2f7356bf511e"`.

First we assign this identity an access policu on the key vault:

![Illustration 23](/assets/csisecret/csisecret023.png)

And then we use it to create a new secretProviderClass object:

```yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: akvkv-subsetupconsul-withpolicyidentity
spec:
  provider: azure
  parameters:
    useVMManagedIdentity: "true"               
    userAssignedIdentityID: 5e69b6e7-a065-4591-abd3-2f7356bf511e # This is the value that we change
    keyvaultName: akvkv-subsetupconsul
    cloudName: ""                               
    objects:  |
      array:
        - |
          objectName: kvs-csisecret1
          objectAlias: kvs-csisecret1            
          objectType: secret                    
          objectVersion:    
    tenantId: e0c45235-95fe-4bd6-96ca-2d529f0ebde4     

```

And mount it on a pod:  

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secretstore-policymsi
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: akvkv-subsetupconsul-withpolicyidentity

```

And sure enough, if we look into the pod, we have access to the secret monted as a csi volume:

```bash

kubectl exec pod-with-secretstore-policymsi -- cat /mnt/secrets-store/kvs-csisecret1

Thisisanewsecretversion^^

```

That means we can probably configure as many identities as we want on the node pools vmss and then define our secretProviderClass to grant easily access to csi secret store volume in pod.
The draw back is that there is no rbac on the Kubernetes side. The managed identities are assigned on the cluster scope and can be used without too much restriction in the Kubernetes Control plane.
Keep that in mind if you further on this road.

## 7. Before leaving

Ok we've seen quite a lot.
You should note that at least:

- Microsoft makes it easier to use a Key vault as a secret provider thanks to the add-on. The main watch point is that we have to rely on a MAnaged IDentity associated to the nodepool, so extra care is required
- It is possible to take the DOY approach and use helm chart to deploy. In thsi case we can even work around the limitation of using only a managed identity at the node pool level, because the use of PodIdentity is possible. But, there are always but, Pod Identity will go away to be replaced by workload Identity. Currently it's not ready and the features are not on the same level as Pod Identity, just so you know.
- replacing secret as environment variable may not be possible if it is not planned on the development side. In one of our example, we had the capability to mount a file for the secret and we used that, with the CSI volume.

Now If I had to give my 2 cents I would say that the add-on is enough if you're not mature withPod Identity. If you already use pod identity, i see no reason not to use it. Anyway, short lifecycle of feature is the new normal. And once Workload Identity is ready, just go for it.

That's all folk ^^
See you around.
