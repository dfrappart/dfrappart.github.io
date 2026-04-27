---
layout: post
title:  "Validating Admission Policy 101"
date:   2026-04-25 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Security
---

Hi!

In this article, we'll have a look at the [`ValidatingAdmissionPolicy object`](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/), how it works, and some example usages.

After playing with the Pod Security Standard and the Pod Security Admission, it felt logical to continue with this, and broaden our view of some of the guardrail options that are natively available in Kubernetes.

Our agenda will be as follow:

1. What is Validating Admission Policy
2. Creating basic `VAP`
3. Looking a bit deeper into the `VAP` object

## 1. What is Validating Admission Policy

`ValidatingAdmissionPolicy`, or `VAP` for lazy people is GA since kubernetes 1.30.

It aims to provide a native feature for validating admission webhooks. 

And webhook in Kubernetes are... &#129300;

I'll make a parallele with Azure policies, which I'm more familiar with.

In Azure, you start with a Policy definition. You define policy definition so that it will allow to  audit, or deny, or even change parameters on Azure objects.
An example could be that we want to audit for the existance of tags, or that an Azure storage should be configured with TLS1.2.
Once the definition is written, a policy assignment, on a specific scope, management group, a subscription, or even a resource group, allows to control that what we defined in the policy is check against the scope set by the assignment.

Some policies (and more and more with time) are called built-in, meaning we only need to consume those with assignment. The definition is in the Cloud provider responsibility scope.

Those built-in policies could be compared to the Pod Security Standard, for the definition, and the Pod Security Admission, for the assignment. Except that it's a bit less rich in kubernetes than in Azure.

Now sometimes, we need to define custom policy on Azure. So we write a specific definition doing something, and we assign on the desired scope. And that's where we have the equivalent with the `VAP`. The ValidatingAdmissionPolicy is the custom policy for which we define the control.

The definition of the policy in the VAP is relying on the Common Expression Language, on which more information are available on the [dedicated google github repo](https://github.com/google/cel-spec/blob/master/doc/langdef.md).

In terms of concepts, as for many others kubernetes objects, we started by creating an object tat define something, in this case the `VAP`, and we bind this object with a related binding object, in this case the ValidatingAdmissionPolicyBinding object. 

To conclude our parallele with Azure Policies, we could say that the `ValidatingAdmissionPolicyBinding` is kind of the Policy assignment. We should add that the `VAP` is only audit or deny, no modification.

![illustration001](/assets/admissioncotntroller/vap001.png)

Let's note the [reference API](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/validating-admission-policy-v1/) for these objects. We'll check that a bit later ^^. For now let's give it a try.

## 2. Creating basic VAPs

There are some sample on the kubernetes documentation, but that would be no fun, so we'll start our own `VAP` from scratch.

Capitalizing from our previous article about PSA, we would like to have a `VAP` that will check if a namespace is configured wxith the proper labels to enable the desired profile.

Basic stuff first.
So we can check how to select the desired resources, in our case, namespaces.
We can also check on which actions we want to trigger this policy, which would be at least at creation, and probably at update.

Those parameters are specified in a `spec` section:

```yaml

spec
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["namespaces"]

```

Now the less easy part, we need to write the rule that will evaluiate this.

As already mentioned, it relies on the CEL language.

First we would like to evaluate the existance of the labels. For this, we can try the `has` macro.

So something like this: 

```yaml

has("object.metadata.labels")

```

Checking the presence of a specific label should be possible with `in`

```yaml

"pod-security.kubernetes.io/enforce" in object.metadata.labels

```

It seems a bit theoretical, but fortunately, we can leverage the [CEL playground](https://playcel.undistro.io/) to validate our expression.

![illustration002](/assets/admissioncontroller/vap002.png)

At this point, I'm capable enough only to evaluate individual expressions. However, at the end, we want to have a `VAP` that look like this, with all of our expressions tested individually on the CEL playground.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-pod-security-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
  validations:
    - expression: >
        has(object.metadata.labels) 
        && "pod-security.kubernetes.io/enforce" in object.metadata.labels
        && object.metadata.labels["pod-security.kubernetes.io/enforce"] in ["restricted", "baseline"]
      message: "Namespaces should have label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline)."
      reason: Invalid

```

that's however not enough to make this work, we also need a binding. At this point, it's a very simple binding.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-pod-security-label-binding
spec:
  policyName: require-pod-security-label
  validationActions: [Audit, Warn]

```

The interesting parameters in the `spec` section are the `spec.policyName`, effectively doing the binding to the `VAP`, and the `spec.validationActions` which allows us to specify what the policy will do.

After applying those manifests, we can try to create a simple namespace, without any label.

```bash

df@df-2404lts:~/kubeadm-single-cilium$ k  get validatingadmissionpolicy
NAME                         VALIDATIONS   PARAMKIND   AGE
require-pod-security-label   1             <unset>     37h
df@df-2404lts:~/kubeadm-single-cilium$ k  get validatingadmissionpolicybindings.admissionregistration.k8s.io 
NAME                                 POLICYNAME                   PARAMREF   AGE
require-pod-security-label-binding   require-pod-security-label   <unset>    37h
df@df-2404lts:~/kubeadm-single-cilium$ k create ns testnsvap --dry-run=server 
Warning: Validation failed for ValidatingAdmissionPolicy 'require-pod-security-label' with binding 'require-pod-security-label-binding': Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline).
namespace/testnsvap created (server dry run)

```

We're using the `--dry-run=server` to get the warning message, and be sure that we'll not create the namespace, because at this point the policy is configured in `warn` and `audit`.

Without the `--dry-run` switch, we still get the warning, but the namespace is created. We can check the audit logs to validate the `audit` parameter of the policy.

```bash

vagrant@cilium2:~$ sudo cat /var/log/kubernetes-audit.log | jq . | grep validation -A2
    "validation.policy.admission.k8s.io/validation_failure": "[{\"message\":\"Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline).\",\"policy\":\"require-pod-security-label\",\"binding\":\"require-pod-security-label-binding\",\"expressionIndex\":0,\"validationActions\":[\"Audit\",\"Warn\"]}]"
  }
}
--
    "validation.policy.admission.k8s.io/validation_failure": "[{\"message\":\"Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline).\",\"policy\":\"require-pod-security-label\",\"binding\":\"require-pod-security-label-binding\",\"expressionIndex\":0,\"validationActions\":[\"Audit\",\"Warn\"]}]"
  }
}

```

That's enough for some basics. Let's go deeper now.

## 3. Looking a bit deeper into the `VAP` related objects

We saw that we manage our custom policies with the `VAP` and the `VAP` Binding.
We mention earlier the API references for those 2 objects.

We can also summarize what we wrote in terms of expression.

| Objectives | Expression CEL |
| --- | --- |
| Check that the `metada.labels` section exists | has(object.metadata.labels) |
| Check that a specific key is present in the labels | "key" in object.metadata.labels | 
| Validate that the value of a key is in a list of allowed value | object.metadata.labels["key"] in ["value1", "value2"] |

Ok, we can now move further in our use cases.

### 3.1. Managing scopes

The validatingadissionpolicies and validatingadmissionpolicybindings are both cluster-wide resources,, which kind of make sense.

```bash

df@df-2404lts:~/kubeadm-single-cilium$ k  api-resources | grep validatingadmission
NAME                                SHORTNAMES                          APIVERSION                           NAMESPACED   KIND
validatingadmissionpolicies                                             admissionregistration.k8s.io/v1      false        ValidatingAdmissionPolicy
validatingadmissionpolicybindings                                       admissionregistration.k8s.io/v1      false        ValidatingAdmissionPolicyBinding

```

But we may want to scope the policies a bit, specifically, we would like to avoid applying the policy on some namespaces. Otherwise, since the current policy act on both CREATE and UPDATE action, trying to add label on the `kube-system` namespace would result to a warning. 

```bash

df@df-2404lts:~/kubeadm-single-cilium$ k  label namespaces kube-system env=prd --dry-run=server
Warning: Validation failed for ValidatingAdmissionPolicy 'require-pod-security-label' with binding 'require-pod-security-label-binding': Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline).
namespace/kube-system labeled (server dry run)

```

Looking into the API reference for the `VAP`, we can find the `spec.matchConstraint.excludeResourceRules` which includes similar arguments to the `spec.matchConstraint.ResourceRules` (except that... we can use it for excluding stuffs)

- apiGroups
- apiVersions
- operations
- resources

It allows us to modify our `VAP` as below, to exclude kube-system, kube-public and kube-node-lease namespaces

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-pod-security-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
    ### Added
    excludeResourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["*"]
      resources: ["namespaces"]
      resourceNames: ["kube-system", "kube-public", "kube-node-lease"]
    ###
  validations:
    - expression: >
        has(object.metadata.labels) 
        && "pod-security.kubernetes.io/enforce" in object.metadata.labels
        && object.metadata.labels["pod-security.kubernetes.io/enforce"] in ["restricted", "baseline"]
      message: "Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline)."
      reason: Invalid

```

After applying the modification, the policy is still in effect, but does not include the excluded namespaces.

```bash

df@df-2404lts:~$ k apply -f ./vapnspsa.yaml 
validatingadmissionpolicy.admissionregistration.k8s.io/require-pod-security-label configured
validatingadmissionpolicybinding.admissionregistration.k8s.io/require-pod-security-label-binding unchanged
df@df-2404lts:~$ k  label namespaces default somelabel=somevalue
The namespaces "default" is invalid: : ValidatingAdmissionPolicy 'require-pod-security-label' with binding 'require-pod-security-label-binding' denied request: Namespaces should have a label 'pod-security.kubernetes.io/enforce' defined (ex: pod-security.kubernetes.io/enforce: baseline).
df@df-2404lts:~$ k  label namespaces kube-system somelabel=somevalue --dry-run=server
namespace/kube-system labeled (server dry run)

```

We mentioned that `spec.matchConstraint.excludeResourceRules` and `spec.matchConstraint.ResourceRules` share similar argument, so it's also possible to target only specific resources by name:

```yaml

spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["namespaces"]
      resourceNames: ["somenamespace"]


```

This may not be the ideal for a broad scope policy scenario.

Let's try other custom scope scenarios.

First we'll create 2 new namespaces.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: testvapns-dev
  labels:
    env: dev
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/enforce-version: v1.35 
    pod-security.kubernetes.io/audit-version: v1.35
    pod-security.kubernetes.io/warn-version: v1.35
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: testvapns-prd
  labels:
    env: prd
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.35
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.35
    pod-security.kubernetes.io/warn-version: v1.35
    pod-security.kubernetes.io/warn: restricted
spec: {}
status: {}

```

This time, we may want to ensure that pods deployed in namespace `testvapns-dev` have the label `env` with the value `dev`.
It would be easy to create a `VAP` scoped on the target namespace, but then the policy would not check condition on the pods but rather on the namespace.

Instead, we'll start with a policy scoped on pods.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-pod-env-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: >
        has(object.metadata.labels) 
        && "env" in object.metadata.labels
        && object.metadata.labels["env"] == "dev"
      message: "Pods should have a label 'env' defined (ex: env: prd)."
      reason: Invalid

```

To select a namespace by its name, we'll rely on the `spec.matchConstraints.namespaceSelector` and make our VAP match the label `kubernetes.io/metadata.name=testvapns-dev` that allows us to identify the namespace by its name.

```yaml

    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: testvapns-dev

```

A pod created in the `testvapns-dev` namespace would trigger the following message. Ignore the warning about PSA and focus on the message related to the `VAP`.

```bash

df@df-2404lts:~$ k  run testpodlabel --image nginx -n testvapns-dev --dry-run=server
Warning: would violate PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "testpodlabel" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "testpodlabel" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "testpodlabel" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "testpodlabel" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

The pods "testpodlabel" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-label-binding' denied request: Pods should have a label 'env' defined (ex: env: prd).

```

For now, we rely mainly on the VAP, and the binding does not bring much to the table.

Let's step back a little. Instead of validating a label env with a specific value, we'll write the rule so that it checks the existance of the env label only.
And to associate this VAP to different namespace, we'll differentiate the bindings.


```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-pod-env-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
    ### Removed  
    #namespaceSelector:
    #  matchLabels:
    #    #env: dev
    #    kubernetes.io/metadata.name: testvapns-dev
  validations:
    - expression: >
        has(object.metadata.labels) 
        && "env" in object.metadata.labels
#        && object.metadata.labels["env"] == "dev"
      message: "Pods should have a label 'env' defined (ex: env: prd)."
      reason: Invalid
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-pod-env-dev-label-binding
spec:
  policyName: require-pod-env-label
  validationActions: [Audit, Deny]   
  matchResources:
    namespaceSelector:
      matchLabels:
        env: dev
        #kubernetes.io/metadata.name: testvapns-dev 
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-pod-env-prd-label-binding
spec:
  policyName: require-pod-env-label
  validationActions: [Audit, Deny]   
  matchResources:
    namespaceSelector:
      matchLabels:
        env: prd
        #kubernetes.io/metadata.name: testvapns-prd  

```

Trying to create pods without the label env should fail.

```bash

df@df-2404lts:~$ k  run prdpod -n testvapns-prd --image=nginx
The pods "prdpod" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-prd-label-binding' denied request: Pods should have a label 'env' defined (ex: env: prd).

df@df-2404lts:~$ k  run devpod -n testvapns-dev --image=nginx
The pods "devpod" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pods should have a label 'env' defined (ex: env: prd).

```

However, if the label env is added, it should work.

```bash

df@df-2404lts:~$ k  run devpod -n testvapns-dev --image=nginx --labels env=dev
pod/devpod created
df@df-2404lts:~$ k get pod -n testvapns-dev --show-labels 
NAME     READY   STATUS    RESTARTS   AGE     LABELS
devpod   1/1     Running   0          3m54s   env=dev

```

We can easilty change the policy so that it check that the pods have either env=dev or env=prd by changing the rule like this.


```bash

  validations:
    - expression: >
        has(object.metadata.labels) 
        && "env" in object.metadata.labels
        && object.metadata.labels["env"] in ["dev", "prd"]
      message: "Pods should have a label 'env' defined (ex: env: prd)."
      reason: Invalid

```

However, at this point, we cannot ensure that the pod has the same env value as its namespace.

Also, the message are not very specific. Checking that into the log does not help much to find where the impacted workload is located.

### 3.2. Be more dynamic in the VAP

So, we want to be more dynamic on our expressions, for the rules, or for the messages that come with the rules.

For that, we can have a look at the [Validation Expression](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#validation-expression) documenation.

Up until now, we used `object` to target the object impacted by the VAP.

For example, `has(object.metadata.labels)` check that the object looked in the VAP (namespace or pod) has labels.

We can also leverage `namespaceObject`. This would allow us to write the expression `has(namespaceObject.metadata.labels)` that checks the namespace of the object evaluated in the expression, and evaluate its labels.
Note that if the VAP was evaluating a cluster-wide resource, the `namespaceObject` would return `null`, and thus our previous expression would always be false.

Ok let's write our new VAP.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-pod-env-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: >
        has(object.metadata.labels) &&
        "env" in object.metadata.labels &&
        has(namespaceObject.metadata.labels) &&
        "env" in namespaceObject.metadata.labels &&
        object.metadata.labels["env"] == namespaceObject.metadata.labels["env"] &&
        namespaceObject.metadata.labels["env"] in ["dev", "prd"]
      messageExpression: >
        'Pod "' + object.metadata.name +
        '" in namespace "' + namespaceObject.metadata.name +
        '" should have a label "env" corresponding to the namespace. ' +
        'Expected value: ' + namespaceObject.metadata.labels["env"] +
        '. Current value of the Pod: ' +
        (has(object.metadata.labels) && "env" in object.metadata.labels ?
         object.metadata.labels["env"] : 'not defined')
      reason: Invalid

```

In the `expression` section:

- **`has(object.metadata.labels)`** checks that the pod has labels
- **"env" in `object.metadata.labels`** checks that the label env is defined
- **`has(namespaceObject.metadata.labels)`** checks that the namespace of the pod has labels
- **"env" in `namespaceObject.metadata.labels`** checks that the namespace of the pod has the label env
- **`object.metadata.labels["env"]` == `namespaceObject.metadata.labels["env"]`** checks the corresponding value between env labels on namespacve and pod.

Noticed also in the `messageExpression` section:

- **'Pod "' + `object.metadata.name`** will translate to **Pod <pod_name>**
- **'" in namespace "' + `namespaceObject.metadata.name`** will translate to **" in namespace <namespace_name>**
- ...

You got the point... &#128527;


Let's try this.

```bash

df@df-2404lts:~$ k run devtest --image nginx -n testvapns-dev --dry-run=server
The pods "devtest" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "devtest" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined
df@df-2404lts:~$ k run devtest --image nginx -n testvapns-dev --labels env=prd --dry-run=server
The pods "devtest" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "devtest" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: prd
df@df-2404lts:~$ k run devtest --image nginx -n testvapns-dev --labels env=dev --dry-run=server
Warning: would violate PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "devtest" must set securityContext.
pod/devtest created (server dry run)
df@df-2404lts:~$ k run devtest --image nginx -n testvapns-prd --labels env=dev --dry-run=server
The pods "devtest" is invalid: : ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-prd-label-binding' denied request: Pod "devtest" in namespace "testvapns-prd" should have a label "env" corresponding to the namespace. Expected value: prd. Current value of the Pod: dev

```

It works as expected. If the pod does not have the label env, it's blocked. If it has the env label but with a value different than the one in the namespace, it's also blocked.

Last thing, we will probably not create pods outside of deployments or other controllers. If we try to create a deployment (and thus some pods), we'll get no message at the deployment creation, but the pods will not be scheduled. checking the replicaset status, or the namespace events will give us the error message related to the VAP.

```bash

df@df-2404lts:~$ k create deployment testdeploydev --image=nginx --replicas 3 --namespace testvapns-dev
Warning: would violate PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/testdeploydev created
df@df-2404lts:~$ k get deployments.apps -n 
cilium-secrets   default          kube-node-lease  kube-public      kube-system      testnsvap        testvapns-dev    testvapns-prd    
df@df-2404lts:~$ k get deployments.apps -n testvapns-dev 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
testdeploydev   0/3     0            0           28s
df@df-2404lts:~$ k get replicasets.apps -n testvapns-dev testdeploydev-7d9d99cd69 -o json | jq .status.conditions
[
  {
    "lastTransitionTime": "2026-04-27T10:22:30Z",
    "message": "pods \"testdeploydev-7d9d99cd69-grgdp\" is forbidden: ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod \"testdeploydev-7d9d99cd69-grgdp\" in namespace \"testvapns-dev\" should have a label \"env\" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined",
    "reason": "FailedCreate",
    "status": "True",
    "type": "ReplicaFailure"
  }
]
df@df-2404lts:~$ k events -n testvapns-dev
LAST SEEN   TYPE      REASON         OBJECT                                MESSAGE
50m         Warning   FailedCreate   ReplicaSet/testdeploydev-7d9d99cd69   Error creating: pods "testdeploydev-7d9d99cd69-l9rqh" is forbidden: ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "testdeploydev-7d9d99cd69-l9rqh" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined
33m         Warning   FailedCreate   ReplicaSet/testdeploydev-7d9d99cd69   Error creating: pods "testdeploydev-7d9d99cd69-zgsxq" is forbidden: ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "testdeploydev-7d9d99cd69-zgsxq" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined
17m         Warning   FailedCreate   ReplicaSet/testdeploydev-7d9d99cd69   Error creating: pods "testdeploydev-7d9d99cd69-xnvf9" is forbidden: ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "testdeploydev-7d9d99cd69-xnvf9" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined
39s         Warning   FailedCreate   ReplicaSet/testdeploydev-7d9d99cd69   Error creating: pods "testdeploydev-7d9d99cd69-2vqbb" is forbidden: ValidatingAdmissionPolicy 'require-pod-env-label' with binding 'require-pod-env-dev-label-binding' denied request: Pod "testdeploydev-7d9d99cd69-2vqbb" in namespace "testvapns-dev" should have a label "env" corresponding to the namespace. Expected value: dev. Current value of the Pod: not defined

```


### 3.3. Using paramKind

If we check the VAP, we can see that we have the `PARAMKIND` column showing `<unset>` for our 2 samples.

That's because to this point, we have not used the paramKind argument.
As the documentation states, this parameter allow a policy configuration to be separate from its definition.

We can illustrate this with the following `configMap`:

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: psaparameters
  namespace: default
data:
  # 
  allowedPSSVersion: "v1.34, v1.35"
  allowedPSSLevel: "restricted, baseline"

```

With this, we create another VAP from the previous one. We'll notice the paramKind section pointing to a configMap.


```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-psa-with-params
spec:
  failurePolicy: Fail
  paramKind:
    apiVersion: v1
    kind: ConfigMap
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["namespaces"]
    excludeResourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["*"]
      resources: ["namespaces"]
      resourceNames: ["kube-system", "kube-public", "kube-node-lease"]
  validations:
  - expression: >
      has(object.metadata.labels) &&
      "pod-security.kubernetes.io/enforce" in object.metadata.labels &&
      object.metadata.labels["pod-security.kubernetes.io/enforce"] in params.data.allowedPSSLevel.split(",")
    messageExpression: >
      'Operation ' + request.operation + ' on Namespace "' + object.metadata.name +
      '" failed: the label "pod-security.kubernetes.io/enforce" must have a value among: ' +
      params.data.allowedPSSLevel + '. Current value: ' +
      (has(object.metadata.labels) && "pod-security.kubernetes.io/enforce" in object.metadata.labels ?
       object.metadata.labels["pod-security.kubernetes.io/enforce"] : 'undefined')
    reason: Invalid  

```

Last we define the binding, in which we point to the `configMap` created earlier. We can notice this time the parameterNotFoundAction, set to Deny, that states how the VAP react if the configMap is not there. In this case, it means that the policy denies if it cannot found the parameters specified for the paramKind section.


```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: restrict-pss-version-with-params-binding
spec:
  policyName: restrict-psa-with-params
  validationActions: ["Audit","Deny"]
  paramRef:
    name: psaparameters      
    namespace: default            
    parameterNotFoundAction: Deny

```

Once everything is created, if we try to create a namespace without the proper labels, it's blocked as expected.

```bash

df@df-2404lts:~$ k create ns testns
The namespaces "testns" is invalid: : ValidatingAdmissionPolicy 'restrict-psa-with-params' with binding 'restrict-pss-version-with-params-binding' denied request: Operation CREATE on Namespace "testns" failed: the label "pod-security.kubernetes.io/enforce" must have a value among: restricted, baseline. Current value: undefined

```

If the configMap is not present, for any reason, we get another deny, but the message is quite clear about the reason why.

```bash

df@df-2404lts:~$ k create ns testns
The namespaces "testns" is invalid: : ValidatingAdmissionPolicy 'restrict-psa-with-params' with binding 'restrict-pss-version-with-params-binding' denied request: failed to configure binding: no params found for policy binding with `Deny` parameterNotFoundAction

```

Reflecting on the use of `paramKind` and `paramRef`, what could we use it for?

Typuically, in the PSS/PSA configuration, we could imagine the configMap as the repository of the required paramters for those.

allowedPSSVersion would allow us to maintain the version of the PSS on a single point rather than on all the poilicy that may use it.
We could also use the configMap to specify the level of `pod-security.kubernetes.io/enforce` required, depending on the environment, and so on for the `pod-security.kubernetes.io/audit` labels.

There is more to this than what we will do in this single article &#128526;.

For now, let's wrap it


## 4. Summary

While in the previous article, we had a look at the built-in policies for Kubernetes by the mean of Pod Security Standard/Admission, this time, we looked at more custom stuff with the `ValidatingAdmissionPolicy`.

That requires understanding the associated language (CEL) and how we should work between the VAP and the VAP Binding.

We also had a look at some way to make it flexible, in the expression, or in the parameters, with the CEL specificities for the former, and the `paramKind`/`paramRef` for the latter. About the `PramKind`, it's interesting to note that while we used configMap, it's also possible to use CRDs, that should be defined obviously. I would say that it is useful to manage cluster-wide resources nstead of namespaced ones such as the configMap.

In our Kubernetes policies journey, we are now at the place of knowing how to to custom policy to deny or audit, but still no automated remediation.

We'll talk about that next time ^^



