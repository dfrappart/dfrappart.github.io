---
layout: post
title:  "Playing with Pod Security Admission"
date:   2026-04-16 18:00:00 +0200
year: 2026
categories: Kubernetes AKS Security
---

Hi!

In this article, we'll have a look at pod security admission.

I had this in my todo since quite some times, and since It's something discussed in the CKS, I though whyt not now?

Today's agenda will be:

1. Introducing Pod Security Admission
2. Using PSA


## 1. Introducing Pod Security Standard and Pod Security Admission

Before talking about Pod Security Admission (that I'll refer to as PSA from now on, because... I'm lazy &#128517;), we should start with [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

As mentioned in the kubernetes documentation, Pod Security Standards (that we'll refer to PSS, told you I was lazy right?) define 3 levels of policies to enforce security configuration. Taken for the doc, we have the 3 existing profiles:

| Profile | Description |
|-|-|
| Privileged | Unrestricted policy, providing the widest possible level of permissions. This policy allows for known privilege escalations. |
| Baseline | Minimally restrictive policy which prevents known privilege escalations. Allows the default (minimally specified) Pod configuration. |
| Restricted | Heavily restricted policy, following current Pod hardening best practices. |


The `privileged` profile odes not enforce restreiction, so we can leave it here.

Now if we look at the `baseline` profile, we can find that there are alreday quite some requirements. Without copy pasting the documentaiton, we can list :

- Privilleged containers parameters blocking the `securityContext.privileged` in containers.
- Seccomp  parameters that block the `unconfined` profile
- HostPath volumes parameters that block mounting HostPAth volumes.

The `restricted` profile goes further, specifically regarding all the `securityContext` section. It blocks for example:

- containers that do not specify running as non root
- the absence of a seccomp profile
- capabilities on the container, at the exception of `NET_BIND_SERVICE`

Ok so that was the PSS, let's see how we can enforce those. And that is with... the [PSA](https://kubernetes.io/docs/concepts/security/pod-security-admission/)! &#128540;

It's good to know that this feature is stable since kubernetes 1.25, which means quite a lot of time in the kubernetes lifecycle.
It works with a built-in admission controller, at the namespace level.

![illustration001](/assets/admissioncotntroller/admissioncontroller001.png)

The built-in admission controller works with labels on the namespace level.

As per the [documentation](https://kubernetes.io/docs/concepts/security/pod-security-admission/), 3 modes are availables:

| Mode | Description |
|-|-|
| enforce | Policy violations will cause the pod to be rejected.|
| audit | Policy violations will trigger the addition of an audit annotation to the event recorded in the audit log, but are otherwise allowed. |
| warn | Policy violations will trigger a user-facing warning, but are otherwise allowed. |

The format of the labels is `pod-security.kubernetes.io/<MODE>: <LEVEL>` where `<MODE>` is one of `enforce`, `audit` or `warn`, and <LEVEL> is one of `privileged`, `baseline` or `restricted`.

Another label `pod-security.kubernetes.io/<MODE>-version: <VERSION>` is used to specify which version of PSA is used. This version being dependant of the kubernetes version, it expects a valid kubernetes version.

Now that we discussed the basics, let's move on to some experimentations.



## 2. Using PSA

### 2.1. Testing deployment in baseline and restricted namespaces

To start PSA, we'll begin with a bunch of namespace, on which we'll add the labels discussed previously.

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: psa-baseline
  labels:
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/audit-version: v1.35
    pod-security.kubernetes.io/warn-version: v1.35
spec: {}
status: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: psa-restricted
  labels:
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit-version: v1.35
    pod-security.kubernetes.io/warn-version: v1.35
spec: {}
status: {}

```

Here we have 2 namespaces, one configured to warn and audit the PSA at `baseline` level, the other at `restricted` level. This means that we should get warning when the policies are not followed, and also logs, taking the hypopthesis that audit logs are configured on our cluster.

Now we want to see the PSA in action. For this we'll create a deploytment in each of our namespaces.

```bash

df@df-2404lts:~$ k create deployment restricteddeploy -n psa-restricted --replicas 3 --image nginx --dry-run=client -o yaml

```

This should gives us the following yaml manifest.

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: "2026-04-16T07:58:13Z"
  generation: 1
  labels:
    app: baselinedeploy
  name: baselinedeploy
  namespace: psa-baseline
  uid: 967731e5-fc2a-4cc4-adde-bb4939db88a2
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: baselinedeploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: baselinedeploy
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}

```

No warning at this point from the baseline policies.

We follow up with another deployment in the `restricted` namespace.

```yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: "2026-04-16T07:57:08Z"
  generation: 1
  labels:
    app: restricteddeploy
  name: restricteddeploy
  namespace: psa-restricted
  uid: ced74992-4eed-4eb8-96c5-70c773278f1e
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: restricteddeploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: restricteddeploy
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}

```

Notice the `--dry-run=server` so that we get the server answer from this command. The warning message is due to the `pod-security.kubernetes.io/warn: baseline` label on the namespace.

```bash

df@df-2404lts:~$ k create deployment restricteddeploy -n psa-restricted --replicas 3 --image nginx --dry-run=server -o yaml
deploy.yaml
Warning: would violate PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")


```

This time we do get some warning. We'll get to see this more thoroughly later. For now we want to dig a bit on the baseline profile.

### 2.2. Diving in the baseline profile

We refered the documentation on the baseline profile in anoter section of this article.
From the section, we saw that a simple deployment is not necessarily blocked by the baseline profile. 

To validate that it does bloc some things, we'll modify our manifest as follow:

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: baselinedeploy
  name: baselinedeploy
  namespace: psa-baseline
spec:
  replicas: 3
  selector:
    matchLabels:
      app: baselinedeploy
  strategy: {}
  template:
    metadata:
      labels:
        app: baselinedeploy
    spec:      
      containers:
      - image: nginx
        name: nginx
        resources: {}
        ### Added
        securityContext:
          seccompProfile:
            type: Unconfined
        ###
status: {}

```

The `--dry-run=server` gives us the following warning

```bash

df@df-2404lts:~$ k apply -f ./baselinedeploy.yaml --dry-run=server
Warning: would violate PodSecurity "baseline:v1.35": seccompProfile (container "nginx" must not set securityContext.seccompProfile.type to "Unconfined")
deployment.apps/baselinedeploy created (server dry run)

```

Let's add another forbidden configuration on the manifest. 

We'll add the following file on the cluster, and mount this host path azs a volume in the pod.

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: baselinedeploy
  name: baselinedeploy
  namespace: psa-baseline
spec:
  replicas: 3
  selector:
    matchLabels:
      app: baselinedeploy
  strategy: {}
  template:
    metadata:
      labels:
        app: baselinedeploy
    spec:      
      containers:
      - image: nginx
        name: nginx
        resources: {}
        securityContext:
          seccompProfile:
            type: Unconfined
        ### Added
        volumeMounts:
        - name: index-html
          mountPath: /usr/share/nginx/html
        ###
        terminationMessagePath: /dev/termination-log
status: {}

```

```bash

df@df-2404lts:~$ k apply -f ./baselinedeploy.yaml
Warning: would violate PodSecurity "baseline:v1.35": hostPath volumes (volume "index-html"), seccompProfile (container "nginx" must not set securityContext.seccompProfile.type to "Unconfined")
deployment.apps/baselinedeploy created
df@df-2404lts:~$ k get deployments.apps -n psa-baseline 
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
baselinedeploy   3/3     3            3           15s
df@df-2404lts:~$ k get pod -n psa-baseline 
NAME                             READY   STATUS    RESTARTS   AGE
baselinedeploy-98cddb9f4-4kd9r   1/1     Running   0          3m30s
baselinedeploy-98cddb9f4-55kx5   1/1     Running   0          3m30s
baselinedeploy-98cddb9f4-kgvm8   1/1     Running   0          3m30s

```

Because the namespace is configured only with the labels `pod-security.kubernetes.io/audit=baseline` and `pod-security.kubernetes.io/warn=baseline`, while we violate the profile, it's still enabled by the PSA. We can however check the audit logs to verify what's visible.

```bash

vagrant@k8scilium1:~$ sudo cat /var/log/kubernetes/audit/audit.log | jq . |grep pod-security
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")",
    "pod-security.kubernetes.io/enforce-policy": "baseline:v1.35"
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")",
    "pod-security.kubernetes.io/enforce-policy": "baseline:v1.35"
        "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"labels\":{\"pod-security.kubernetes.io/audit\":\"baseline\",\"pod-security.kubernetes.io/audit-version\":\"v1.35\",\"pod-security.kubernetes.io/warn\":\"baseline\",\"pod-security.kubernetes.io/warn-version\":\"v1.35\"},\"name\":\"psa-baseline\"},\"spec\":{},\"status\":{}}\n"
        "pod-security.kubernetes.io/enforce": null,
        "pod-security.kubernetes.io/enforce-version": null
        "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"labels\":{\"pod-security.kubernetes.io/audit\":\"restricted\",\"pod-security.kubernetes.io/audit-version\":\"v1.35\",\"pod-security.kubernetes.io/warn\":\"restricted\",\"pod-security.kubernetes.io/warn-version\":\"v1.35\"},\"name\":\"psa-restricted\"},\"spec\":{},\"status\":{}}\n"
        "pod-security.kubernetes.io/enforce": null,
        "pod-security.kubernetes.io/enforce-version": null
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")"
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")"
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")",
    "pod-security.kubernetes.io/enforce-policy": "privileged:latest"
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")",
    "pod-security.kubernetes.io/enforce-policy": "privileged:latest"
    "pod-security.kubernetes.io/audit-violations": "would violate PodSecurity \"baseline:v1.35\": hostPath volumes (volume \"index-html\"), seccompProfile (container \"nginx\" must not set securityContext.seccompProfile.type to \"Unconfined\")",
    "pod-security.kubernetes.io/enforce-policy": "privileged:latest"

```

And we can see that we have logs about the PSA.

Ok let's have a look at the `restricted` profile.

### 2.3. The restricted profile

We've already seen that, as its name implies, the `restricted` profile is much more... restrictive &#128517;

We have a namespace with some lables to audit and warn about the `restricted` profile

```bash

df@df-2404lts:~$ k get ns psa-restricted -o json |jq .metadata.labels
{
  "kubernetes.io/metadata.name": "psa-restricted",
  "pod-security.kubernetes.io/audit": "restricted",
  "pod-security.kubernetes.io/audit-version": "v1.35",
  "pod-security.kubernetes.io/warn": "restricted",
  "pod-security.kubernetes.io/warn-version": "v1.35"
}

```

This time, we'll go further in the enforcement our the PSA and add the associated labels.

```bash

df@df-2404lts:~$ k label namespaces psa-restricted "pod-security.kubernetes.io/enforce"="restricted"
namespace/psa-restricted labeled
df@df-2404lts:~$ k label namespaces psa-restricted "pod-security.kubernetes.io/enforce-version"="v1.35"
namespace/psa-restricted labeled
df@df-2404lts:~$ k get ns psa-restricted -o json |jq .metadata.labels
{
  "kubernetes.io/metadata.name": "psa-restricted",
  "pod-security.kubernetes.io/audit": "restricted",
  "pod-security.kubernetes.io/audit-version": "v1.35",
  "pod-security.kubernetes.io/enforce": "restricted",
  "pod-security.kubernetes.io/enforce-version": "v1.35",
  "pod-security.kubernetes.io/warn": "restricted",
  "pod-security.kubernetes.io/warn-version": "v1.35"
}

```

If we try to apply the following manifest:

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:

  labels:
    app: restricteddeploy
  name: restricteddeploy
  namespace: psa-restricted
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: restricteddeploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: restricteddeploy
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}

```

We'll get the warning messages exprected, because we do have the warn configured on the namesapce.

```bash

df@df-2404lts:~$ k apply -f /home/df/yamlconfig/podsecurityadmission/restricteddeploy.yaml
Warning: would violate PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/restricteddeploy created

```

The deployment is created, but the pods do not pop.

```bash

df@df-2404lts:~$ k get pod -n psa-restricted 
No resources found in psa-restricted namespace.
df@df-2404lts:~$ k get deployments.apps -n psa-restricted 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
restricteddeploy   0/3     0            0           28m

```

We can review the PSA messagesin the replicaset directly, or throiugh the namùespace events.

```bash

df@df-2404lts:~$ k describe replicasets.apps -n psa-restricted restricteddeploy-6475d54f49 
Name:           restricteddeploy-6475d54f49
Namespace:      psa-restricted
Selector:       app=restricteddeploy,pod-template-hash=6475d54f49
Labels:         app=restricteddeploy
                pod-template-hash=6475d54f49
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/restricteddeploy
Replicas:       0 current / 3 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=restricteddeploy
           pod-template-hash=6475d54f49
  Containers:
   nginx:
    Image:         nginx
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                From                   Message
  ----     ------        ----               ----                   -------
  Warning  FailedCreate  32m                replicaset-controller  Error creating: pods "restricteddeploy-6475d54f49-rv5rp" is forbidden: violates PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
==================================================truncated==================================================
  Warning  FailedCreate  10m                replicaset-controller  Error creating: pods "restricteddeploy-6475d54f49-78rvr" is forbidden: violates PodSecurity "restricted:v1.35": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

```

```bash
df@df-2404lts:~$ k events -n psa-restricted -o json |jq .

```

```json
{
  "kind": "EventList",
  "apiVersion": "v1",
  "metadata": {},
  "items": [
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {...},
    {
      "kind": "Event",
      "apiVersion": "v1",
      "metadata": {
        "name": "restricteddeploy-6475d54f49.18a721a948084fed",
        "namespace": "psa-restricted",
        "uid": "082eab30-4415-4282-a662-0bcb6618a5d1",
        "resourceVersion": "803137",
        "creationTimestamp": "2026-04-17T11:33:08Z"
      },
      "involvedObject": {
        "kind": "ReplicaSet",
        "namespace": "psa-restricted",
        "name": "restricteddeploy-6475d54f49",
        "uid": "99658fa9-c658-4e12-a01c-826b5219dd04",
        "apiVersion": "apps/v1",
        "resourceVersion": "800761"
      },
      "reason": "FailedCreate",
      "message": "Error creating: pods \"restricteddeploy-6475d54f49-78rvr\" is forbidden: violates PodSecurity \"restricted:v1.35\": allowPrivilegeEscalation != false (container \"nginx\" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container \"nginx\" must set securityContext.capabilities.drop=[\"ALL\"]), runAsNonRoot != true (pod or container \"nginx\" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container \"nginx\" must set securityContext.seccompProfile.type to \"RuntimeDefault\" or \"Localhost\")",
      "source": {
        "component": "replicaset-controller"
      },
      "firstTimestamp": "2026-04-17T11:33:08Z",
      "lastTimestamp": "2026-04-17T11:33:08Z",
      "count": 1,
      "type": "Warning",
      "eventTime": null,
      "reportingComponent": "replicaset-controller",
      "reportingInstance": ""
    }
  ]
}

``` 

Adding all the restricted profile requirement, we get the following.

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: restricteddeploy
  name: restricteddeploy
  namespace: psa-restricted
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: restricteddeploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: restricteddeploy
    spec:
      ### Added
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      ###    
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ### Added
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        ### 
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
status: {}

```

The good point is that we seems to not trigger anymore the PSA.

```bash

df@df-2404lts:~$ k apply -f ./restricteddeploy.yaml
deployment.apps/restricteddeploy configured

```

However, still no pods, and an error status for the replicaset.

```bash

df@df-2404lts:~$ k get deployments.apps -n psa-restricted 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
restricteddeploy   0/3     1            0           43m
df@df-2404lts:~$ k get pod -n psa-restricted 
NAME                                READY   STATUS   RESTARTS      AGE
restricteddeploy-66bddd5dc8-9ggwb   0/1     Error    2 (18s ago)   20s
df@df-2404lts:~$ k get pod -n psa-restricted 
NAME                                READY   STATUS   RESTARTS        AGE
restricteddeploy-66bddd5dc8-9ggwb   0/1     Error    6 (3m44s ago)   6m32s

```

The event gives us a little bit more of details.

```bash

df@df-2404lts:~$ k events -n psa-restricted -o json |jq .items[26]


```

```json

{
  "kind": "Event",
  "apiVersion": "v1",
  "metadata": {
    "name": "restricteddeploy-66bddd5dc8-9ggwb.18a722d7f6019475",
    "namespace": "psa-restricted",
    "uid": "94590512-544e-4c60-bef4-d0b5539cf371",
    "resourceVersion": "806118",
    "creationTimestamp": "2026-04-17T11:54:48Z"
  },
  "involvedObject": {
    "kind": "Pod",
    "namespace": "psa-restricted",
    "name": "restricteddeploy-66bddd5dc8-9ggwb",
    "uid": "8dcdd4e8-981e-43f3-92a8-456f5b5bb591",
    "apiVersion": "v1",
    "resourceVersion": "805468",
    "fieldPath": "spec.containers{nginx}"
  },
  "reason": "BackOff",
  "message": "Back-off restarting failed container nginx in pod restricteddeploy-66bddd5dc8-9ggwb_psa-restricted(8dcdd4e8-981e-43f3-92a8-456f5b5bb591)",
  "source": {
    "component": "kubelet",
    "host": "k8scilium1"
  },
  "firstTimestamp": "2026-04-17T11:54:48Z",
  "lastTimestamp": "2026-04-17T12:00:17Z",
  "count": 10,
  "type": "Warning",
  "eventTime": null,
  "reportingComponent": "kubelet",
  "reportingInstance": "k8scilium1"
}

```

The culprit for this is the parameter `securityContext.capabilities.drop=["ALL"]` in the container.

Indeed, nginx tries to bind the port 80 which is not something possible without the NET_BIND_SERVICE capabilites.

Checking the documentation of the [PSS](https://kubernetes.io/docs/concepts/security/pod-security-standards/), we can see that it was taken into consideration because this capability is listed in the allowed ones.

Restricted Fields

    spec.containers[*].securityContext.capabilities.drop
    spec.initContainers[*].securityContext.capabilities.drop
    spec.ephemeralContainers[*].securityContext.capabilities.drop

Allowed Values

    Any list of capabilities that includes ALL

Restricted Fields

    spec.containers[*].securityContext.capabilities.add
    spec.initContainers[*].securityContext.capabilities.add
    spec.ephemeralContainers[*].securityContext.capabilities.add

Allowed Values

    Undefined/nil
    NET_BIND_SERVICE

So we can modify the manifest by adding the required capability and it should work.


```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: restricteddeploy
  name: restricteddeploy
  namespace: psa-restricted
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: restricteddeploy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: restricteddeploy
    spec:
      ### Added
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      ###    
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ### Added
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]
        ### 
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
status: {}

```

```bash

df@df-2404lts:~$ k apply -f ./restricteddeploy.yaml
deployment.apps/restricteddeploy created
df@df-2404lts:~$ k get replicasets.apps -n psa-restricted 
NAME                          DESIRED   CURRENT   READY   AGE
restricteddeploy-7dfd7f7884   3         3         0       11s

```

So at first it seems to be ok. But, waiting a bit, we can see errors. The thing is, those errors are not due to the PSA but to what we implemented to respect the PSA.

```bash

df@df-2404lts:~$ k get pod -n psa-restricted 
NAME                                 READY   STATUS    RESTARTS      AGE
restricteddeploy-7dfd7f7884-6w2mt    0/1     Error     2 (31s ago)   36s
restricteddeploy-7dfd7f7884-mn2bn    0/1     Error     2 (33s ago)   36s
restricteddeploy-7dfd7f7884-rg46k    0/1     Error     2 (32s ago)   36s

df@df-2404lts:~$ k logs -n psa-restricted restricteddeploy-7dfd7f7884-6w2mt
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/04/17 14:41:46 [warn] 1#1: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
2026/04/17 14:41:46 [emerg] 1#1: mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)

```

We can see some permission denied messages, that may be due to the `securityContext.runAsNonRoot` parameter that is required by the profile.

To avoid adding the `NET_BIND_SERVICE` capability, and also working around the root requirement that seems inherent to the nginx proces, we can find other images that require less privileges, such as [`nginxinc/nginx-unprivileged`](https://hub.docker.com/r/nginxinc/nginx-unprivileged/).

This time we are able to run our pods.

```bash

df@df-2404lts:~$ k get pod -n psa-restricted 
NAME                                 READY   STATUS             RESTARTS        AGE
nginx-unprivileged-c57c79b44-99bcm   1/1     Running            0               36s
nginx-unprivileged-c57c79b44-mbpn6   1/1     Running            0               36s
nginx-unprivileged-c57c79b44-vnpbm   1/1     Running            0               36s
restricteddeploy-7dfd7f7884-6w2mt    0/1     CrashLoopBackOff   9 (2m37s ago)   23m
restricteddeploy-7dfd7f7884-mn2bn    0/1     CrashLoopBackOff   9 (2m16s ago)   23m
restricteddeploy-7dfd7f7884-rg46k    0/1     CrashLoopBackOff   9 (2m33s ago)   23m

```

That about all we wanted to see, let's wrap this

## 3. Before leaving

In this article, we had a look at the Pod Security Standard, and how tose are puyshed through Pod Secuyrity Admission/
The feture is avaialble since a few version of kubernetes, so we do not have any excuse to not use it.

Apart from that, we saw that there are different level of configuration. either warn, audit or enforce. In a progresive implementation, the warn/audit is quite useful to get informations on the potential security violations.
IMHO, those level should be confuigured by default to get informations on the securty posture inside kubernetes.

Now about the different profile, while the `baseline` seems to be permissive enough to be enforced without too much impact, we could see that the `restricted` does implies the use of hardened, small footprint images, as was the case with the nginx-unprivileged image that we used. If the teams are already mature regarding those security aspects, no impact, if not, well, let's say that starting with audit and warn is necessary 

And that will be all.

Next, I'll probably have a look at Validating Admission Policies &#128526;.



