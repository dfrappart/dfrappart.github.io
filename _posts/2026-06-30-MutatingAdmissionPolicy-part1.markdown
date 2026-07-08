---
layout: post
title:  "Mutating Admission Policy Part 1"
date:   2026-06-30 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Security
---

Hello there!

It's been some time since the last article.

Since march, we played with different kind of native admission controller, so that we could manage guardrails, or policies, or whatever we want to call it, in kubernetes.

This article will be a continuation of those last articles and we'll have a look at the recently GA Mutating Admission Policy. Also, And it was planned at the beginning, it will be a 2 part article, because, well, there is a lot to say

Our agenda will be as follow:

1. What is Mutating Admission Policy
2. Creating basic `MAP`
3. Combining all of our options to secure a kubernetes cluster

## 1. What is Mutating Admission Policy

Similarly to  `ValidatingAdmissionPolicy`, that I call `VAP` (because I'm lazy, told you last time &#128527;), we also have 
`MutatingAdmissionPolicy`, that I'll call `MAP` in all future reference, without surprise.

As one could expect, `MAP` are a (native) way to mutate object that are send to the API server, before those are created.
Taking again the Azure policy comparison, it would be the custom policies with [`modify`](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-modify), or [`deployifnotexist`](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deploy-if-not-exists) scenarios.
Attentive readers will notice that there is a preview for Azure policies with `mutate` effect, but that's outside of our scope... today &#128526;

For a long time, it was not something that we could do natively, and we had to relied on 3rd party tool such as [kyverno](https://kyverno.io/).
Now, I'm not saying that we should stop using kyverno here. It does its job quite well IMHO.
But for people not already kyverno expert, it's worth a look I guess.

If we try to draw how the `MAP` works, it's very similar to the `VAP`.

![illustration001](/assets/admissioncontroller/map001.png)

The reference API is available [here](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/).

## 2. Creating basic MAPs

### 2.1. Undersrtanding a bit the `MutatingAdmissionPolicy` configuration

Last time we went directly into the creation of a `VAP`, without looking first into the API.
Let's not do that this time &#128518; and start by checking some information.

So basically, we'll have the `MAP` and its own binding.

In the `MAP` we have the [`spec`](https://kubernetes.io/docs/reference/kubernetes-api/admissionregistration/mutating-admission-policy-v1/#MutatingAdmissionPolicySpec) section in which everything is defined.

Without copy-pasting all the API reference, we can summarize.

- `failurePolicy` defines how failures are handle. It can be `Fail` or `Ignore`
- `matchConstraints` specifies what resources this policy is designed to validate. The example below shows how to use `matchConstraint` with `resourceRules`. Other arguments are detailled in the API documentation. This specifc `matchConstraints` configure the `MAP` to trigger at `namespace` creation.

```yaml

spec:
  # Target only namespace creations
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["namespaces"]

```

- `matchConditions` filters requests already validated by `matchConstraints`. As in the example below, we can for example add a condition on the namespaces with a specific label. In this specific section, we already define that the trigger occur at the creation of any namespace. We add a requirement that the namespace shoud have also the label `env=prd`.

```yaml

  # Condition : Apply only if the namespace has the label `env=prd`
  matchConditions:
    - name: "has-env-prd-label"
      expression: >
        has(object.metadata.labels) &&
        'env' in object.metadata.labels &&
        object.metadata.labels['env'] == 'prd'

```

- `paramKind`, as in `VAP`, allows to manage parameters of the `MAP` independantly. We can use `ConfigMaps` or `CRDs` and add the reference in the binding.
- `reinvocationPolicy` allows to determine if the modification should occur once or more.
- `variables` open interesting possibilities to manipulates expressions inside the `MAP`. Think terraform locals if you need another something similar.
- `mutations`, at last, is where the modification are defined. The example below define a mutation that use the jsonPatch kind and that adds on namespaces some security related labels.

```yaml

mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1enforce",
              value: "restricted"
            },
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1enforce-version",
              value: "v1.36"
            }
          ]

```

On the `path`, the `/` acts as a delimitation to indicate subsection. Meaning that `path: "/metadata/labels/something"` with `value: "this"` would be in yaml

```yaml

metadata:
  labels:
    something: "this"

```

And that's where the `~1` is useful. It is understood by the `MAP` as the character `/`. Adding simply the `/` in the `pod-security.kubernetes.io~1enforce` would be translated as 

```yaml

metadata:
  labels:
    pod-security.kubernetes.io: 
      enforce: "restricted"

```

Apart from the `JSONPatch`, we can also leverage the `applyConfiguration`. We'll have a look at this one later.

Below is our first sample `MAP`

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: enforce-restricted-pss-on-prd-namespaces
spec:
  # Target only namespace creations
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["namespaces"]

  # Condition : Apply only if the namespace has the label `env=prd`
  matchConditions:
    - name: "has-env-prd-label"
      expression: >
        has(object.metadata.labels) &&
        'env' in object.metadata.labels &&
        object.metadata.labels['env'] == 'prd'

  # Behavior on failure
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded

  # Mutation : Add labels for Pod Security Admission
  mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1enforce",
              value: "restricted"
            },
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1enforce-version",
              value: "v1.36"
            }
          ]
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          has(object.metadata.annotations) ?
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations/" + jsonpatch.escapeKey("policy.modifiedby.com/enforce-restricted-pss-on-prd-namespaces"),
              value: "true"
            }
          ]
          :
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations",
              value: {"policy.modifiedby.com/" + "enforce-restricted-pss-on-prd-namespaces": "true"}
            }
          ]          

```

And its associated binding.

```yaml

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: enforce-restricted-pss-on-prd-namespaces-binding
spec:
  policyName: enforce-restricted-pss-on-prd-namespaces
  matchResources:
    namespaceSelector: {}

```

It's quite straightforward to use the `MutatingAdmissionPolicyBinding`. We need to reference the `MAP` that we want to bind and the `matchResource` allows us to select, in this case, all namespaces. As for the `VAP`, this is also in the `MutatingAdmissionPolicyBinding` that we have the `paramRef` allowing us to referencde an `CRD` or a `configMap` to act as a source of parameters.

Now, detailing the path section, we have `op: add` which means that we are adding something in our manifest.

Then, we have the `path: "/metadata/labels/pod-security.kubernetes.io~1enforce"` and the `value: "restricted"`.

All this means that the policy add the label `pod-security.kubernetes.io~1enforce="restricted"`.

Notice the other `JSONPatch` mutation which is used to add annotions.

```yaml

    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          has(object.metadata.annotations) ?
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations/" + jsonpatch.escapeKey("policy.modifiedby.com/enforce-restricted-pss-on-prd-namespaces"),
              value: "true"
            }
          ]
          :
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations",
              value: {"policy.modifiedby.com/" + "enforce-restricted-pss-on-prd-namespaces": "true"}
            }
          ]

```

The idea here is to annotate the object targeted by the `MAP` (in this case a namespace, as you can read in the `spec.matchConstraint`) so that we keep track of the mutation.

There is also the `has(object.metadata.annotations) ?` followed by 2 `JSONPatch` CEL expression, delimited by a `:`.

That's because we want to manage 2 scenarios here:

- if there is already an `annotations` section. That's the first CEL expression. And that's why the path is written with the trailing `/` in `/metadata/annotations/`. It means that we look in the annotation section and the `/` is here to define that we'll go in the `annotations` section, to add `policy.modifiedby.com/enforce-restricted-pss-on-prd-namespaces` with the value `"true"`.
- The second CEL expression, after the `:` manage the scenario where there is no `annotations` section. By specifying the `path: "/metadata/annotations"`, without the trailing `/` we add the `annotations` section, and the value is a json object `{"policy.modifiedby.com/" + "enforce-restricted-pss-on-prd-namespaces": "true"}`

### 2.2. Testing our policy

Now that everything is (a bit &#128517;) clearer, let's try this policy.

We defined it so that each time a namespace with the label env=prd is created, it would had the labels for the Pod Security Standard Restricted level.

Let's write a manifest for a namespace following this requirement.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: checkmapns
  labels:
    env: prd

```

Upon creating the namespace, we can see that it does have the additional labels resulting of the mutation.

```bash

➜  ~ k get namespaces checkmapns -o json |jq '.metadata.labels'
{
  "env": "prd",
  "kubernetes.io/metadata.name": "checkmapns",
  "pod-security.kubernetes.io/enforce": "restricted",
  "pod-security.kubernetes.io/enforce-version": "v1.36"
}

```

We can also verify that there is an annotation to track the modification of our `MAP`

```bash

➜  ~ k get namespaces checkmapns -o json |jq '.metadata.annotations'
{
  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"labels\":{\"env\":\"prd\"},\"name\":\"checkmapns\"},\"spec\":{},\"status\":{}}\n",
  "policy.modifiedby.com/enforce-restricted-pss-on-prd-namespaces": "true"
}

```

If on the other hand we create another namespace without the `env` label, the policy is not triggered

```yaml

➜  ~ k create namespace checkmapns2
➜  ~ k get ns $cil1 checkmapns2
NAME          STATUS   AGE
checkmapns2   Active   24s
➜  ~ k get ns $cil1 checkmapns2 -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2026-07-03T12:41:43Z"
  labels:
    kubernetes.io/metadata.name: checkmapns2
  name: checkmapns2
  resourceVersion: "211196"
  uid: 9de5abc1-924f-4033-a037-2dd334660083
spec:
  finalizers:
  - kubernetes
status:
  phase: Active

```

And there is nothing in the `annotations` or the `labels` sections.

Ok let's move on.

## 3. Combining all of our options to secure a kubernetes cluster

In this section, we'll try to push further our usage of the `MutatingAdmissionPolicies` with a scenario.

### 3.1. Defining a scenario

The scenario, following our first sample in indeed quite simple (in what we need, not the implementation ^^).

We would like to ensure that the Pod Security Standard are applied on our namespaces and the pods that live in those.
We already have a `MAP` that can mutate the namespaces with the env=prd label.

We want to add a clear requirement:

- All namespaces with `env=prd` label should be configured to enforce the PSS restricted.
- We also add that the namespaces with env!=prd should be configure to enforce the PSS baseline, but also warn on the PSS restricted requirements for pods

First thing first, we should ensure that all namespace have the env label. We can achieve this with a VAP actually.

### 3.2. Write a VAP denying the creation of namespaces without env label defined.

We saw in the last article how to manage `VAP`. So let's just write it, and it's binding.

```yaml

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-env-label-on-namespace
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["namespaces"]
  validations:
  - expression: >
      has(object.metadata.labels) 
      && "env" in object.metadata.labels
      && object.metadata.labels["env"] in ["dev", "prd", "ppr", "staging"]
    message: "Namespace should have a valid 'env' defined (dev|prd|ppr|staging)."
    reason: Invalid
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-env-label-on-namespace-binding
spec:
  policyName: require-env-label-on-namespace
  validationActions: [Deny]
  matchResources:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["namespaces"] 

```

We can validate that the `ValidatingAdmissionPolicy` is working be trying to create a new namespace, without the `env=prd`.

```bash

➜  ~ k create namespace checkvapns1 -o yaml                                        
The namespaces "checkvapns1" is invalid: : ValidatingAdmissionPolicy 'require-env-label-on-namespace' with binding 'require-env-label-on-namespace-binding' denied request: Namespace should have a valid 'env' defined (dev|prd|ppr|staging).

```

### 3.3. Adding a MAP for the non prd namespace

We already wrote the `MAP` to add the required labels for PSS on prd namespace. So the non prd namespaces `MAP` is quite similar.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: enforce-restricted-pss-on-non-prd-namespaces
spec:
  # Target only namespace creations
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["namespaces"]

  # Condition : Apply only if the namespace has the label `env` different from `prd`
  matchConditions:
    - name: "has-env-non-prd-label"
      expression: >
        has(object.metadata.labels) &&
        'env' in object.metadata.labels &&
        object.metadata.labels['env'] != 'prd'

  # Behavior on failure
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded

  # Mutation : Add labels for Pod Security Admission
  mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1audit",
              value: "restricted"
            },
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1warn",
              value: "restricted"
            },
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1audit-version",
              value: "v1.36"
            },
            JSONPatch{
              op: "add", path: "/metadata/labels/pod-security.kubernetes.io~1warn-version",
              value: "v1.36"
            }
          ]
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          has(object.metadata.annotations) ?
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations/" + jsonpatch.escapeKey("policy.modifiedby.com/enforce-restricted-pss-on-non-prd-namespaces"),
              value: "true"
            }
          ]
          :
          [
            JSONPatch{
              op: "add",
              path: "/metadata/annotations",
              value: {"policy.modifiedby.com/enforce-restricted-pss-on-non-prd-namespaces": "true"}
            }
          ]          

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: enforce-restricted-pss-on-non-prd-namespaces-binding
spec:
  policyName: enforce-restricted-pss-on-non-prd-namespaces
  matchResources:
    namespaceSelector: {}

```

The important part here is the `matchConditions` section which specify that the policy should apply only for namespaces with the label not equal to `prd`

```yaml

  matchConditions:
    - name: "has-env-non-prd-label"
      expression: >
        has(object.metadata.labels) &&
        'env' in object.metadata.labels &&
        object.metadata.labels['env'] != 'prd'

```

Because we also have the `VAP` that accepts only the env values `dev|ppr|prd|staging`, it's enough has a filter.

### 3.4. Inheriting the namespace env label

Now that we have some basics in terms of admission controllers on the `namespaces`, we would like to go to the next stpe, which is to manipulate admission for the `pods`.

Before diving into the ultimate aims which is to secure our `pods`, we'll start with a first `MAP` that allows us to configure the env label inheritance from the `namespace` to the `pod`.

To attain this, we have to consider the trigger. So we'll write a condition that is trigger for `pod` creation

```yaml

  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]

```

With an additional `matchConditions` that check for pods without `labels` or without specifically the `env` label.

```yaml

  matchConditions:
    - name: "has-label-and-env-label-is-absent"
      expression: >
        !has(object.metadata.labels) ||
        !('env' in object.metadata.labels)

```

Let's verify this by creating a bunch of namespaces.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: testmap1
  labels:
    env: prd
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: testmap2
  labels:
    env: dev
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: testmap3
  labels:
    noenvdefined: thatslife
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: testmap4
  labels:
    env: notavalidenv
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: testmap5
  labels:
    env: ppr
spec: {}
status: {}

```

And with our full `MAP`

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: inherit-env-label-from-namespace
spec:
  # Target only pod creations and updates
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
# Condition : Apply only if the namespace has the label `env` is absent
  matchConditions:
    - name: "has-label-and-env-label-is-absent"
      expression: >
        !has(object.metadata.labels) ||
        !('env' in object.metadata.labels)
  # Behavior on failure
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
    # Mutation : Inherit the `env` label from the namespace to the pod
  mutations:
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        has(object.metadata.labels) ?
        [
          JSONPatch{
            op: "add",
            path: "/metadata/labels/env",
            value: namespaceObject.metadata.labels['env']
          }
        ]
        :
        [
          JSONPatch{
            op: "add",
            path: "/metadata/labels",
            value: {"env": namespaceObject.metadata.labels['env']}
          }
        ]
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        has(object.metadata.annotations) ?
        [
          JSONPatch{
            op: "add",
            path: "/metadata/annotations/" + jsonpatch.escapeKey("policy.modifiedby.com/inherit-env-label-from-namespace"),
            value: "true"
          }
        ]
        :
        [
          JSONPatch{
            op: "add",
            path: "/metadata/annotations",
            value: {"policy.modifiedby.com/inherit-env-label-from-namespace": "true"}
          }
        ]  

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: inherit-env-label-from-namespace-binding
spec:
  policyName: inherit-env-label-from-namespace
  matchResources:
    namespaceSelector: 
      matchExpressions:
          - { key: env, operator: In, values: [dev,ppr,prd] }


```

Creating the `namespaces` should generate errors, because some of those do not respect the requirements of the `VAP`

```bash

➜  ~ k apply -f ./04_ns.yaml
namespace/testmap1 created
namespace/testmap2 created
namespace/testmap5 created
Error from server (Invalid): error when creating "./04_ns.yaml": namespaces "testmap3" is forbidden: ValidatingAdmissionPolicy 'require-env-label-on-namespace' with binding 'require-env-label-on-namespace-binding' denied request: Namespace should have a valid 'env' defined (dev|prd|ppr|staging).
Error from server (Invalid): error when creating "./04_ns.yaml": namespaces "testmap4" is forbidden: ValidatingAdmissionPolicy 'require-env-label-on-namespace' with binding 'require-env-label-on-namespace-binding' denied request: Namespace should have a valid 'env' defined (dev|prd|ppr|staging).

```

Now we can try to create some pods in the namespaces that were allowed.

```bash

➜  ~ k get pod -n testmap3
No resources found in testmap3 namespace.
➜  ~ k run testmapdev1 -n testmap2 --image nginx
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "testmapdev1" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "testmapdev1" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "testmapdev1" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "testmapdev1" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/testmapdev1 created
➜  ~ k run testmapppr1 -n testmap5 --image nginx
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "testmapppr1" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "testmapppr1" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "testmapppr1" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "testmapppr1" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/testmapppr1 created
➜  ~ k run testmapprd1 -n testmap1 --image nginx
Error from server (Forbidden): pods "testmapprd1" is forbidden: violates PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "testmapprd1" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "testmapprd1" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "testmapprd1" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "testmapprd1" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

```

We can notice that the creation fails in the namespace with the `env=prd`, because of the `enforce` status of the `PSS`.
Also we can check the `env` label inheritance for the other pods.

```yaml

➜  ~ k get pod -n testmap2 -o json |jq '.items[].metadata.labels'
{
  "env": "dev",
  "run": "testmapdev1"
}
➜  ~ k get pod -n testmap5 -o json |jq '.items[].metadata.labels'
{
  "env": "ppr",
  "run": "testmapppr1"
}

```

Ok, that's nice, and we'll wrap here for today.

## 4. Summary

So in this article, we moved from the `ValildatingAdmissionPolicy` to the `MutatingAdmissionPolicy`.

The principles are quite the same. First create a policy, then bind it with a policy binding. 
Still using the `CEL`.
The main difference reside in what we want to achieve, which is classified as mutations.
Coming next is  a usecase of a MAP to enforce the `PSS` with a `MAP` which help us to go deeper into the `CEL` syntax and the mutations.



