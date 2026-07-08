---
layout: post
title:  "Mutating Admission Policy Part 2"
date:   2026-07-15 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Security
---

Hi!

As promised, we're following from part 1 to go further in our usage of `MAP` and try to enforce the `PSS` controls for the restricted profile

Our agenda will be as follow:

1. Rapid review of the PSS controls and where we are
2. Initiating the MAP for the Pod level controls
3. Adding the container level controls

## 1. Rapid review of the PSS controls and where we are

### 1.1. A reminder of PSS Restricted requirements.

We talked about those in a [previous article](https://blog.teknews.cloud/kubernetes/aks/security/2026/04/16/Playing_with_Pod_Security_Admission.html) specifically, but since we're going to use `MAP` to mutate the pod manifests, it's not a loss of time to review the [PSS requirement](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

We can see from this table that we need to add stuffs in the `spec.securityContext` section of the pod, and also in the ``

| Control | Restricted Fields | Allowed Values |
| --- | --- | --- |
| Volume Types | spec.volumes[*] | Each item in spec.volumes[*] must set one of the following fields to a non-null value: configMap, csi, downwardAPI, emptyDir, ephemeral, persistentVolumeClaim, projected, secret. All other types (e.g., hostPath) are forbidden. |
| Privilege Escalation (v1.8+) | spec.containers[*].securityContext.allowPrivilegeEscalation, spec.initContainers[*].securityContext.allowPrivilegeEscalation, spec.ephemeralContainers[*].securityContext.allowPrivilegeEscalation | false (Linux only if spec.os.name != windows) |
| Running as Non-root | spec.securityContext.runAsNonRoot, spec.containers[*].securityContext.runAsNonRoot, spec.initContainers[*].securityContext.runAsNonRoot, spec.ephemeralContainers[*].securityContext.runAsNonRoot | true (Container fields may be nil if pod-level field is true) |
| Running as Non-root User (v1.23+) | spec.securityContext.runAsUser, spec.containers[*].securityContext.runAsUser, spec.initContainers[*].securityContext.runAsUser, spec.ephemeralContainers[*].securityContext.runAsUser | Any non-zero value (e.g., 1000) or nil/undefined |
| Seccomp (v1.19+) | spec.securityContext.seccompProfile.type, spec.containers[*].securityContext.seccompProfile.type, spec.initContainers[*].securityContext.seccompProfile.type, spec.ephemeralContainers[*].securityContext.seccompProfile.type | RuntimeDefault or Localhost (Forbidden: Unconfined or absence of profile) (Linux only if spec.os.name != windows) |
| Capabilities (v1.22+) | Drop: spec.containers[*].securityContext.capabilities.drop, spec.initContainers[*].securityContext.capabilities.drop, spec.ephemeralContainers[*].securityContext.capabilities.drop Add: spec.containers[*].securityContext.capabilities.add, spec.initContainers[*].securityContext.capabilities.add, spec.ephemeralContainers[*].securityContext.capabilities.add | Drop: Must include ALL (e.g., ["ALL"]) Add: nil/undefined or NET_BIND_SERVICE (Linux only if spec.os.name != windows) |

Looking at this list, we'll confirm that the policies details match the error or warn messages we saw earlier.

## 1.2. Where we are

As a reminder, we should have a `VAP` that deny the namespace creation without the env label.

```yaml

  validations:
  - expression: >
      has(object.metadata.labels) 
      && "env" in object.metadata.labels
      && object.metadata.labels["env"] in ["dev", "prd", "ppr", "staging"]
    message: "Namespace should have a valid 'env' defined (dev|prd|ppr|staging)."
    reason: Invalid

```
We should also have a `MAP`, relying on the previous policy, that enforce the `PSA` restricted controls in warn for non-prd environments, and in enforce for prd environment.

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

```yaml

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

```

Ok fine, let's create our new `MAP` then.

## 2. Initiating the MAP for the Pod level controls

To get started, we can consider the additions required on the pod level. Those are the following:

| Pod Level Control | Restricted Fields | Allowed Values |
| --- | --- | --- |
| Running as Non-root | spec.securityContext.runAsNonRoot | true |
| Running as Non-root User (v1.23+) | spec.securityContext.runAsUser | Any non-zero value (e.g., 1000) or nil/undefined |
| Seccomp (v1.19+) | spec.securityContext.seccompProfile.type | RuntimeDefault or Localhost (Forbidden: Unconfined or absence of profile) (Linux only if spec.os.name != windows) |

So we can write our first mutation for the future `MAP`

```json

        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsUser",
            value: 1000
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsNonRoot",
            value: true
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/seccompProfile",
            value: {"type": "RuntimeDefault"}
          }
        ]

```

At this point, it still quite easy: simple path, simple path action. However, we should also take into considerations, as earlier, the existence or not of the `securityContext` section. This previous expression works if `securityContexct` exists. If not we should write something like this:

```json

        [
          JSONPatch{
            op: "add", 
            path: "/spec/securityContext",
            value: {"runAsUser": 1000}
          },
          JSONPatch{
            op: "add", 
            path: "/spec/securityContext",
            value: {"runAsNonRoot": true}
          }
        ]

````

Or like this:

```json

        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext",
            value: Object.spec.securityContext{
              runAsUser: 1000,
              runAsNonRoot: true,
              seccompProfile: Object.spec.securityContext.seccompProfile{type: "RuntimeDefault"}
            }
          }
        ]

```

Both works so we can pick any (It may be that I'll change my mind later about the 2 workings the same way, because TBH I'm still learning CEL &#128541;).

Remember that we manage the condition this way:

```bash

<condition> ? true : false

```

So we write finally the first mutation as below.

```yaml

  mutations:

  # Pod-level securityContext: runAsUser, runAsNonRoot, seccompProfile
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        has(object.spec.securityContext) ?
        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsUser",
            value: 1000
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsNonRoot",
            value: true
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/seccompProfile",
            value: {"type": "RuntimeDefault"}
          }
        ]
        :
        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext",
            value: Object.spec.securityContext{
              runAsUser: 1000,
              runAsNonRoot: true,
              seccompProfile: Object.spec.securityContext.seccompProfile{type: "RuntimeDefault"}
            }
          }
        ]

```

Thas was the easy part, now let's work on the containers ' `securityContext`.

## 3. Adding the container level controls

### 3.1. Managing the `runAsNonroot` and `allowPrivilegeEscalation`

The difficulty here is that it's containers plural, which reflect the fact that we can have more than one container.

I'll be honest and tell you I relied on LLM for help for this.

To get started we'll use the `ApplyConfiguration` patch.

up until now, we use solely the `JSONPatch`, which is kind of imperative in its nature. We specify the operation in the `ops` field.

The `Applyconfiguration` is on the other hand declarative and use a server-side apply.

By iterating on an LLM, we get the following mutation.

```yaml

  - patchType: "ApplyConfiguration"
    applyConfiguration:
      expression: >
        Object{
          spec: Object.spec{
            containers: object.spec.containers.map(c, Object.spec.containers{
              name: c.name,
              securityContext: Object.spec.containers.securityContext{
                allowPrivilegeEscalation: false,
                runAsNonRoot: true
              }
            })
          }
        }

```

This expression applies on the containers (all of them) the `securityContext.allowPrivilegeEscalation` with the value `false` and the `securityContext.runAsNonroot` with the value `true`.

The first `Object{...}` takes the object that is to be mutated, which is the pod, per the `matchConstraint` section of the `MAP`

```yaml

spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]

```

Then, inside this `Object{}`, we define another level with the `spec: Object.spec{}` which direct the mutation to look at the `spec` section of the pod.

Going further, we can see

```json

spec: Object.spec{
  containers: object.spec.containers.map(c, ...)
}

```

`object.spec.containers` allow us to read the list of containers in the pod. By writing

```json

object.spec.containers.map(c, Object.spec.containers{
  name: c.name,
  securityContext: ...
})

```

We create a for each loop that take the `c` as the iteration of the containers map. Because we have `name: c.name`, we use the name of the container as the key and we tell the server to apply the values listed after `securityContext`.

It's interesting to note that the `ApplyConfiguration` patch merges the specified configuration with the existing manifest.
So either there is no `securityContext` in the container, and it write it, or there is one and it overwrites it. That's why we don't need conditional expression as previously.

Using `JSONPatch` for this would result in something like this:

```json

[
  JSONPatch{ op: "add", path: "/spec/containers/0/securityContext/allowPrivilegeEscalation", value: false },
  JSONPatch{ op: "add", path: "/spec/containers/0/securityContext/runAsNonRoot", value: true },
  JSONPatch{ op: "add", path: "/spec/containers/1/securityContext/allowPrivilegeEscalation", value: false },
  JSONPatch{ op: "add", path: "/spec/containers/1/securityContext/runAsNonRoot", value: true },
  ...
]

```

Which would not work well for an unpredictible number of containers (which is all the time...).

Ok, now let's see how we can work on the `securityContext.capabilities` configuration.

### 3.2. Managing the `securityContext.capabilities` configuration

So, why did we not manage the `capabilities` with the same mutation as previously, using the `applyConfiguration` patch?

That's because `capabilities.drop` is declared as listType: atomic in the Kubernetes API Go types. Atomic means the entire list is treated as a single indivisible unit owned by one manager — server-side apply.
Remember that it is what is used by the `ApplyConfiguration` patch.
Taking this into account, it means that we have to switch back to `JSONPatch`.

After some iterations, we write an expression as below.

```yaml

  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        object.spec.containers.map(c,
          JSONPatch{
            op: "add",
            path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/capabilities",
            value: Object.spec.containers.securityContext.capabilities{
              drop: ["ALL"]
            }
          }
        )

```

We take the containers list by using `object.spec.containers`. the `object` refering to the object filtered by the `MAP`.

Then the `map()` allows us to manage the items of the `containers` map.

The `c,JSONPatch{}` in `map()` allows us to loop on each element of the containers map, so we're good.

After that, the `path` expression written `"/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/capabilities"` uses the `string(object.spec.containers.indexOf(c))` and build the appropriate path for each container, `/spec/containers/0/securityContext/capabilities` for the first container, `/spec/containers/1/securityContext/capabilities` for the second one...

And at last, `value: Object.spec.containers.securityContext.capabilities{drop: ["ALL"]}` build the expected value for each containers. A sample for a 3 containers pod would give the following:

```json

[
  JSONPatch{
    op: "add",
    path: "/spec/containers/0/securityContext/capabilities",
    value: {drop: ["ALL"]}
  },
  JSONPatch{
    op: "add",
    path: "/spec/containers/1/securityContext/capabilities",
    value: {drop: ["ALL"]}
  },
  JSONPatch{
    op: "add",
    path: "/spec/containers/2/securityContext/capabilities",
    value: {drop: ["ALL"]}
  }
]

```

Notice that we don't do a conditional statement as before with the other `JSONPatch` And that's because the differents mutations are applied sequently following the order of writing. 
Because we used the `applyConfiguration` before this `JSONPatch`, it's ok to assume that there is now a `securityContext` for each containers, hence no conditional.

That looks ok, let's try this now.

### 3.3. Testing the `MAP`

After all this, we have the full `MAP` and its binding.

```yaml

apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: mapenforcerestrictedpsa-on-ppr-namespaces
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
  mutations:

  # Pod-level securityContext: runAsUser, runAsNonRoot, seccompProfile
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        has(object.spec.securityContext) ?
        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsUser",
            value: 1000
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/runAsNonRoot",
            value: true
          },
          JSONPatch{
            op: "add",
            path: "/spec/securityContext/seccompProfile",
            value: {"type": "RuntimeDefault"}
          }
        ]
        :
        [
          JSONPatch{
            op: "add",
            path: "/spec/securityContext",
            value: Object.spec.securityContext{
              runAsUser: 1000,
              runAsNonRoot: true,
              seccompProfile: Object.spec.securityContext.seccompProfile{type: "RuntimeDefault"}
            }
          }
        ]
  # Container-level securityContext: allowPrivilegeEscalation, runAsNonRoot
  - patchType: "ApplyConfiguration"
    applyConfiguration:
      expression: >
        Object{
          spec: Object.spec{
            containers: object.spec.containers.map(c, Object.spec.containers{
              name: c.name,
              securityContext: Object.spec.containers.securityContext{
                allowPrivilegeEscalation: false,
                runAsNonRoot: true
              }
            })
          }
        }
  # JSONPatch handles capabilities.drop (atomic list — can't be touched by ApplyConfiguration)
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        object.spec.containers.map(c,
          JSONPatch{
            op: "add",
            path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/capabilities",
            value: Object.spec.containers.securityContext.capabilities{
              drop: ["ALL"]
            }
          }
        )
  #Audit-trail annotation marking this policy touched the pod
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        has(object.metadata.annotations) ?
        [
          JSONPatch{
            op: "add",
            path: "/metadata/annotations/" + jsonpatch.escapeKey("policy.modifiedby.com/mapenforcerestrictedpsa-on-ppr-namespaces"),
            value: "true"
          }
        ]
        :
        [
          JSONPatch{
            op: "add",
            path: "/metadata/annotations",
            value: {"policy.modifiedby.com/mapenforcerestrictedpsa-on-ppr-namespaces": "true"}
          }
        ]
---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: mapenforcerestrictedpsa-on-ppr-namespaces-binding
spec:
  policyName: mapenforcerestrictedpsa-on-ppr-namespaces
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: env
          operator: In
          values:
            - ppr
            - prd

```

We can try to create some pods, directly or embedded in deployments.

```yaml

apiVersion: v1
kind: Pod
metadata:
  annotations: {}
  labels:
    run: testmapppr3
  name: testmapppr3
  namespace: testmap5
spec:
  securityContext: {}
  containers:
  - image: nginxinc/nginx-unprivileged
    name: testmapppr3
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapppr4
  name: testmapppr4
  namespace: testmap5
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginxinc/nginx-unprivileged
    name: testmapppr4
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapppr5
  name: testmapppr5
  namespace: testmap5
spec:
  securityContext:
    runAsUser: 1000
  containers:
 
  - image: nginxinc/nginx-unprivileged
    name: testmapppr5
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapppr6
  name: testmapppr6
  namespace: testmap5
spec:
  containers:
  - image: nginxinc/nginx-unprivileged
    name: testmapppr6
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: testmap-deploy
  name: testmap-deploy
  namespace: testmap5
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testmap-deploy
  strategy: {}
  template:
    metadata:
      labels:
        app: testmap-deploy
    spec:
      containers:
      - image: nginxinc/nginx-unprivileged
        name: nginx
        resources: {}
status: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: testmap-deploy-multicontainer
  name: testmap-deploy-multicontainer
  namespace: testmap5
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testmap-deploy-multicontainer
  strategy: {}
  template:
    metadata:
      labels:
        app: testmap-deploy-multicontainer
    spec:
      containers:
      - image: nginxinc/nginx-unprivileged
        name: nginx
        resources: {}
      - name: busybox
        image: busybox:latest
        args:
        - sleep
        - "1000000"
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapppr7
  name: testmapppr7
  namespace: testmap5
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - image: nginxinc/nginx-unprivileged
    name: nginx
    resources: {}
  - name: busybox
    image: busybox:latest
    args:
    - sleep
    - "1000000"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapprd1
  name: testmapprd1
  namespace: testmap1
spec:
  containers:
  - image: nginxinc/nginx-unprivileged
    name: nginx
    resources: {}
  - name: busybox
    image: busybox:latest
    args:
    - sleep
    - "1000000"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testmapprd2
  name: testmapprd2
  namespace: testmap1
spec:
  containers:
  - image: nginxinc/nginx-unprivileged
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test-deploy-prd
  name: test-deploy-prd
  namespace: testmap1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-deploy-prd
  strategy: {}
  template:
    metadata:
      labels:
        app: test-deploy-prd
    spec:
      containers:
      - image: nginxinc/nginx-unprivileged
        name: nginx-unprivileged
        resources: {}
status: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test-deploy-multicontainer-prd
  name: test-deploy-multicontainer-prd
  namespace: testmap1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-deploy-multicontainer-prd
  strategy: {}
  template:
    metadata:
      labels:
        app: test-deploy-multicontainer-prd
    spec:
      containers:
      - image: nginxinc/nginx-unprivileged
        name: nginx-unprivileged
        resources: {}
      - name: busybox
        image: busybox:latest
        args:
        - sleep
        - "1000000"
status: {}


```

After applying this manifest, we get the following output.

```bash

➜  ~ k $cil1 apply -f  /Users/df/Documents/myrepo/k8slocal/yamlconfig/map/pod.yaml
pod/testmapppr3 created
pod/testmapppr4 created
pod/testmapppr5 created
pod/testmapppr6 created
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/testmap-deploy created
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (containers "nginx", "busybox" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (containers "nginx", "busybox" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or containers "nginx", "busybox" must set securityContext.runAsNonRoot=true), seccompProfile (pod or containers "nginx", "busybox" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/testmap-deploy-multicontainer created
pod/testmapppr7 created
pod/testmapprd1 created
pod/testmapprd2 created
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "nginx-unprivileged" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx-unprivileged" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx-unprivileged" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx-unprivileged" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/test-deploy-prd created
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (containers "nginx-unprivileged", "busybox" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (containers "nginx-unprivileged", "busybox" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or containers "nginx-unprivileged", "busybox" must set securityContext.runAsNonRoot=true), seccompProfile (pod or containers "nginx-unprivileged", "busybox" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/test-deploy-multicontainer-prd created

```

The first thing we can notice is the absence of error, which means that even the pods targeting a prd namespace are ok, while the restricted `PSA` is enforced.

If we look to the pod inside `testmap1` `namespace`, we can see our mutations worked as expected.

```bash

➜  ~ k $cil1 get ns testmap1 --show-labels 
NAME       STATUS   AGE   LABELS
testmap1   Active   4d    env=prd,kubernetes.io/metadata.name=testmap1,pod-security.kubernetes.io/enforce-version=v1.36,pod-security.kubernetes.io/enforce=restricted

➜  ~ k $cil1 get pods -n testmap1 -o custom-columns=Name:.metadata.name,NS:.metadata.namespace,Env:.metadata.labels.env 
Name                                             NS         Env
test-deploy-multicontainer-prd-d68f798bc-8td4m   testmap1   prd
test-deploy-prd-7d975b8578-pnpmd                 testmap1   prd
testmapprd1                                      testmap1   prd
testmapprd2                                      testmap1   prd

➜  ~ k $cil1 get pod -n testmap1 -o json| jq '.items[].metadata.name, .items[].spec.securityContext, .items[].spec.containers[].securityContext'

```

```json

"test-deploy-multicontainer-prd-d68f798bc-8td4m"
"test-deploy-prd-7d975b8578-pnpmd"
"testmapprd1"
"testmapprd2"
{
  "runAsNonRoot": true,
  "runAsUser": 1000,
  "seccompProfile": {
    "type": "RuntimeDefault"
  }
}
{
  "runAsNonRoot": true,
  "runAsUser": 1000,
  "seccompProfile": {
    "type": "RuntimeDefault"
  }
}
{
  "runAsNonRoot": true,
  "runAsUser": 1000,
  "seccompProfile": {
    "type": "RuntimeDefault"
  }
}
{
  "runAsNonRoot": true,
  "runAsUser": 1000,
  "seccompProfile": {
    "type": "RuntimeDefault"
  }
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}

```

In more details, the 

```bash

➜  ~ k $cil1 get pod -n testmap1 -o json test-deploy-multicontainer-prd-d68f798bc-8td4m |jq '.spec.securityContext, .spec.containers[].name, .spec.containers[].securityContext' 

```
```json

{
  "runAsNonRoot": true,
  "runAsUser": 1000,
  "seccompProfile": {
    "type": "RuntimeDefault"
  }
}
"nginx-unprivileged"
"busybox"
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "runAsNonRoot": true
}

```

The strange point here is related to the warning that we got for the deployments, as if the mutations were not performed, and a `warn` parameter was added in the `PSA` labels, but as we can see on our namespace, it's not the case.

```bash

Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (container "nginx-unprivileged" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx-unprivileged" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx-unprivileged" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx-unprivileged" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/test-deploy-prd created
Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false (containers "nginx-unprivileged", "busybox" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (containers "nginx-unprivileged", "busybox" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or containers "nginx-unprivileged", "busybox" must set securityContext.runAsNonRoot=true), seccompProfile (pod or containers "nginx-unprivileged", "busybox" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/test-deploy-multicontainer-prd created

```

It could make sense for the pod deployed in `testmap2` or `testmap5` which do have the `PSA` configured with `warn`

```bash
➜  ~ k get namespace $cil1 testmap2 -o json |jq '.metadata.name,.metadata.labels'                                      
```
```json
"testmap2"
{
  "env": "dev",
  "kubernetes.io/metadata.name": "testmap2",
  "pod-security.kubernetes.io/audit": "restricted",
  "pod-security.kubernetes.io/audit-version": "v1.36",
  "pod-security.kubernetes.io/warn": "restricted",
  "pod-security.kubernetes.io/warn-version": "v1.36"
}
```
```bash
➜  ~ k get namespace $cil1 testmap5 -o json |jq '.metadata.name,.metadata.labels'
```
```json
"testmap5"
{
  "env": "ppr",
  "kubernetes.io/metadata.name": "testmap5",
  "pod-security.kubernetes.io/audit": "restricted",
  "pod-security.kubernetes.io/audit-version": "v1.36",
  "pod-security.kubernetes.io/warn": "restricted",
  "pod-security.kubernetes.io/warn-version": "v1.36"
}

```

All in all, we're happy to attain our objectives. 
Before wrapping this, a few additional considerations.

### 3.4. Further considerations

In our last `MAP`, we used the 2 kind of pathes available. But we could keep everything with `JSONPatch` since we could achieve a loop over the list of containers.

In this case, we would need to use the conditional statement as discussed earlier.

The mutations at the container level could be merge as one as below:

```yaml

  # 2. Container-level securityContext — JSONPatch only, one patch list per container
  - patchType: "JSONPatch"
    jsonPatch:
      expression: >
        object.spec.containers.map(c,
          has(c.securityContext) ?
          [
            JSONPatch{
              op: "add",
              path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/allowPrivilegeEscalation",
              value: false
            },
            JSONPatch{
              op: "add",
              path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/runAsNonRoot",
              value: true
            },
            JSONPatch{
              op: "add",
              path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/capabilities",
              value: Object.spec.containers.securityContext.capabilities{
                drop: ["ALL"]
              }
            },
            JSONPatch{
              op: "add",
              path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext/seccompProfile",
              value: Object.spec.containers.securityContext.seccompProfile{
                type: "RuntimeDefault"
              }
            }
          ]
          :
          [
            JSONPatch{
              op: "add",
              path: "/spec/containers/" + string(object.spec.containers.indexOf(c)) + "/securityContext",
              value: Object.spec.containers.securityContext{
                allowPrivilegeEscalation: false,
                runAsNonRoot: true,
                capabilities: Object.spec.containers.securityContext.capabilities{
                  drop: ["ALL"]
                },
                seccompProfile: Object.spec.containers.securityContext.seccompProfile{
                  type: "RuntimeDefault"
                }
              }
            }
          ]
        ).flatten()

```

If some wondered about using the `applyconfiguration` patch to replace the audit trail mutation, please note that it's not possible &#128517;.
`metadata.annotations` is declared mapType: atomic in the Kubernetes API schema — the same atomicity constraint that blocks `capabilities.drop` from ApplyConfiguration applies here. So that won't work.

So this is time to wrap the `MAP` experiment


## 4. Summary

In this 2 part article, we were able to get a better understanding of `MutatingAdmissionPolicy` and had the oppportunity to manipulate more `CEL`.

The conclusion of all of this could be:

- `MAP` is a powerfull tool that allows to bring better control in a k8s environment. In addition to Pod Security Admissions & `ValidatingAdmissionPolicy`, it is possible to instantiate complex scenarios of control on the 
- Its concepts are simple yet the full usage relies on a (very) good understanding of the `CEL`. 




