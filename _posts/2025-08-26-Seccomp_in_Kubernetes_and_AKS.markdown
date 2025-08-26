---
layout: post
title:  "Seccomp in Kubernetes and AKS"
date:   2025-08-26 18:00:00 +0200
year: 2025
categories: Kubernetes AKS Security
---

Hi!

I'm currently trying (with moderate succes, hence the trying part &#128573;) the CKS certification.
Among the multitude of topics to understand, there is one call **Seccomp**.
So this article will not be anything in the breaking technologies stuff, but clearly a kind of walkthrough to use Seccomp in Kubernetes and AKS.
Hope you'll enjoiy it, and well, at least It will be a good cheat sheet for me &#129325;

So the Agenda:


1. A little introduction to Seccomp
2. Trying to use Seccomp on a Kubernetes Cluster
3. What about Seccomp on AKS
3. Conclusion




## 1. A little introduction to Seccomp

### 1.1. A bit of history and prerequisites

From a Kubernetes standpoint, Seccomp, for Secure Compute, is a feature from Linux to restrict the calls that a process is able to perform, from the user namespace to the kernel.

If we dig a little bit more, we can find on [wikipedia](https://en.wikipedia.org/wiki/Seccomp) or [lwn.net](https://lwn.net/Articles/656307/) that Seccomp is quite old.
It was introduced around 2005, to secure the execution of untrusted programs in grid computing.
With time, it was more thoroughly adopted, notably by Chrome, to sandbox the execution of Adobe Flash, in Docker, and many others programs such as Firefox, OpenSSH...

Back to our use case now. Using Seccomp is a mean to ensure that programs running on Kubernetes, a.k.a in pods, are isolated from the hosts and limited in the system calls that can be done.

Obviously we need a recent kernel, above:

- Linux kernel ≥ 2.6.12 from which basic seccomp strict mode was introduced.
- Linux kernel ≥ 3.5 from which `seccomp-BPF` filter mode (the flexible one that allows whitelists/blacklists) was added.
- Linux kernel ≥ 3.17 from which dedicated `seccomp()` syscall introduced (instead of only `prctl`).

Modern linux distributions fill those requirement. We can check this wit the `uname -r` command

```bash

vagrant@k8scilium1:~$ uname -r
6.14.0-28-generic

```

Additionaly, it seems that the kernel should be compiled **CONFIG_SECCOMP=y** and **CONFIG_SECCOMP_FILTER=y** options.

```bash

vagrant@k8scilium1:~$ grep SECCOMP /boot/config-$(uname -r)
CONFIG_HAVE_ARCH_SECCOMP=y
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
# CONFIG_SECCOMP_CACHE_DEBUG is not set

```

And, last but not least, the container engine and kubernetes should be recent enough to support Seccomp. It means something like kubernete `1.19` at least, which is kind of old already.

```bash

vagrant@k8scilium1:~$ k version
Client Version: v1.32.8
Kustomize Version: v5.5.0
Server Version: v1.32.8

vagrant@k8scilium1:~$ k get node -o wide
NAME         STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8scilium1   Ready    control-plane   2d23h   v1.32.8   192.168.56.17   <none>        Ubuntu 24.04.2 LTS   6.8.0-64-generic   containerd://1.7.27


``` 

Now back to more Seccomp in Kubernetes.


### 1.2. Seccomp in Kubernetes basics

Taking the asumptions that Seccomp can be used on a Kubernetes Cluster, how can it be used to secure more the environment?

Well, the basics are not too difficult. 

Leveraging the `spec.securityContext.seccompProfile` section in a pod manifest, we can configure Seccomp to limit what a process in the pod can do.

**Note**: Seccomp is not the only thing that can be configured in the `spec.securityContext` section, but we want to focus on Seccomp today.

In the `seccompProfile` section, we can set three values:

- `Unconfined`: This value will enfore no restrictions. Let's say that it is not our preferred configuration &#128517;. 
- `RuntimeDefault`: This value will enforce the container runtime’s default profile. More on that later
- `Localhost`: This value is probably the most intersting, and the worst at the same time.It allows the use of a custom profile from the node’s filesystem.

About the container runtimle default's profile, it seems that whatever the runtime i.e Docker, Containerd, Cri-O, it is based on Docker's default profile. We can find information about it on the [Docker documentation](https://docs.docker.com/engine/security/seccomp/).

It's a whitelist based profile, meaning that its default action is block syscalls, except those that are specificallly allowed. The doc includes a portion of the whitelisted syscall, with explanations for most syscalls.
Digging a bit more, we can find on Docker's github the [`default.json`](https://github.com/microsoft/docker/blob/master/profiles/seccomp/default.json) that is used for the profile.

Copy/pasting the full profile is not really relevant, but just for information, the structure looks like this.


```json

{
	"defaultAction": "SCMP_ACT_ERRNO",
	"architectures": [
		"SCMP_ARCH_X86_64",
		"SCMP_ARCH_X86",
		"SCMP_ARCH_X32"
	],
	"syscalls": [
		{
			"name": "accept",
			"action": "SCMP_ACT_ALLOW",
			"args": []
		},
====truncated====
    ]
}

```

The default action `"SCMP_ACT_ERRNO"` blocks the unspecified syscalls, and the `syscalls` list let us specify all the allowed syscall.

This structure is also what is used for the custom profiles loaded with the `Localhost` value. 

Ok, enough with the concepts, let's try some stuffs ^^

## 2. Trying to use Seccomp on a Kubernetes cluster

### 2.1. Using the container runtime default profile

For this first hands-on part, I'll use a single node kubeadm cluster. That's because this way, I get full access to the node, which is not that easy with managed kubernetes such as AKS. But we'll have a look at that in the next part.

For now, we'll start with a sample pod, in which we'll add the container runtime default in the security

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample
  name: sample
spec:
# Seccomp config
  securityContext:
    seccompProfile:
      type: RuntimeDefault
# Seccomp config end      
  containers:
  - image: nginx
    name: sample
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

With this specific container, a.k.a nginx, it's working fine. It's considered a security best practice to implement this profile by default.

However, there are some case where it's either not secure enough, or where it's too much, depending on the apps. In this case, we need to switch to custom profiles.

### 2.2. Using custom seccomp profile

Relying on the [kubernetes documentation](https://kubernetes.io/docs/tutorials/security/seccomp/#enable-the-use-of-runtimedefault-as-the-default-seccomp-profile-for-all-workloads), we can find the following profiles.

- `audit.json`, a profile that only audit syslogs

```json

{
    "defaultAction": "SCMP_ACT_LOG"
}

```

- `violation.json`, a profile that litteraly blocks all syslogs

```json

{
    "defaultAction": "SCMP_ACT_ERRNO"
}

```

We can see that the `defaultAction` differs in both profiles. We can get information from another page of the [kubernetes documentation](https://kubernetes.io/docs/reference/node/seccomp/).

| Seccomp profile action| Description |
|-|-|
| SCMP_ACT_ERRNO | Return the specified error code. |
| SCMP_ACT_ALLOW | Allow the syscall to be executed. |
| SCMP_ACT_KILL_PROCESS | Kill the process. |
| SCMP_ACT_KILL_THREAD and SCMP_ACT_KILL | Kill only the thread. |
| SCMP_ACT_TRAP | Throw a SIGSYS signal. |
| SCMP_ACT_NOTIFY and SECCOMP_RET_USER_NOTIF. | Notify the user space. |
| SCMP_ACT_TRACE | Notify a tracing process with the specified value. |
| SCMP_ACT_LOG | Allow the syscall to be executed after the action has been logged to syslog or auditd. |


Ok, Let's create another pod, this time with the `violation.json` profile

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample
  name: sample
spec:
# Seccomp config
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/violation.json
# Seccomp config end      
  containers:
  - image: nginx
    name: sample
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

In this case we specify a path so that kubernetes can find the file and load it. The path evalueated is referenced by the seccomp location, which is, for a kubeadm cluster `/var/lib/kubelet/seccomp`. To load custom profiles, we need to have our json files in this path.

```bash

vagrant@k8scilium1:~$ ls /var/lib/kubelet/seccomp/profiles/
audit.json  finegrained.json  violation.json

```

Note that if we create a pod with a reference to an unexisting profile, it will naturally fail, as we can see below:

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample
  name: sample
spec:
# Seccomp config
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/thisprofiledoesnotexist.json
# Seccomp config end      
  containers:
  - image: nginx
    name: sample
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

```bash

vagrant@k8scilium1:~$ k get pod sample-profilenotexisting 
NAME                        READY   STATUS                 RESTARTS   AGE
sample-profilenotexisting   0/1     CreateContainerError   0          10m

agrant@k8scilium1:~$ k get pod sample-profilenotexisting -o json | jq .status.containerStatuses[0].state
{
  "waiting": {
    "message": "failed to create containerd container: cannot load seccomp profile \"/var/lib/kubelet/seccomp/profiles/thisprofiledoesnotexist.json\": open /var/lib/kubelet/seccomp/profiles/thisprofiledoesnotexist.json: no such file or directory",
    "reason": "CreateContainerError"
  }
}

```

What about the sample pod with the `violation` profile. This time, we expect the pod to fail because we do block all the syscalls.

```bash

vagrant@k8scilium1:~$ k get pod sample-violationprofile 
NAME                      READY   STATUS              RESTARTS     AGE
sample-violationprofile   0/1     RunContainerError   2 (8s ago)   26s

vagrant@k8scilium1:~$ k get pod sample-violationprofile -o json | jq .status.containerStatuses
[
  {
    "containerID": "containerd://1c307e53296a09559e7eec632c47af3d27d8e3a46b073b0675d2313a156e58c6",
    "image": "docker.io/library/nginx:latest",
    "imageID": "docker.io/library/nginx@sha256:33e0bbc7ca9ecf108140af6288c7c9d1ecc77548cbfd3952fd8466a75edefe57",
    "lastState": {
      "terminated": {
        "containerID": "containerd://1c307e53296a09559e7eec632c47af3d27d8e3a46b073b0675d2313a156e58c6",
        "exitCode": 128,
        "finishedAt": "2025-08-22T15:28:10Z",
        "message": "failed to start containerd task \"1c307e53296a09559e7eec632c47af3d27d8e3a46b073b0675d2313a156e58c6\": cannot start a stopped process: unknown",
        "reason": "StartError",
        "startedAt": "1970-01-01T00:00:00Z"
      }
    },
    "name": "sample-violationprofile",
    "ready": false,
    "restartCount": 3,
    "started": false,
    "state": {
      "waiting": {
        "message": "back-off 40s restarting failed container=sample-violationprofile pod=sample-violationprofile_default(ec188dd8-4dfd-4bb7-8eb6-b88f9910b843)",
        "reason": "CrashLoopBackOff"
      }
    },
    "volumeMounts": [
      {
        "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
        "name": "kube-api-access-dp9qb",
        "readOnly": true,
        "recursiveReadOnly": "Disabled"
      }
    ]
  }
]

```

This means that Seccomp work as expected, and avoid pods that would require too much syscalls. But how are the Seccomp profiles defined?

Well, that's a bit more complex. Let's have a look.

### 2.3. Creating a custom Seccomp profile

Because a Seccomp profile is used to allow only the necessary, or at least the acceptable, syscalls, we need a way to find out which syscalls is used by the app.

One way to do this, among orhers, is the audit profile that we used earlier. Remember, this profile uses the `SCMP_ACT_LOG` which allows the syscall to be executed after logging its action in syslog.

So if we create a pod with this profile, as defined in the kubernetes documentation, we can then theorically parse the syslog to find about which syscall are used.

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: audit-pod
  labels:
    app: audit-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: test-container
    image: hashicorp/http-echo:1.0
    args:
    - "-text=just made some syscalls!"
    securityContext:
      allowPrivilegeEscalation: false
---

```

Looking in the syslog for the pod related logs, we can find this.

```bash

vagrant@k8scilium1:~$ cat /var/log/syslog |grep "http-echo"
2025-08-25T07:54:39.281965+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108479.280:484): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:54:39.282463+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108479.281:485): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T07:55:39.282778+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108539.281:486): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:55:39.283486+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108539.282:487): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T07:56:39.284482+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108599.283:488): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:56:39.284497+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108599.283:489): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T07:57:39.285585+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108659.283:490): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:57:39.285612+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108659.283:491): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T07:58:39.285689+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108719.284:492): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:58:39.285731+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108719.284:493): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T07:59:39.285663+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108779.284:494): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T07:59:39.285692+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108779.284:495): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T08:00:39.286492+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108839.285:496): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T08:00:39.286513+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108839.285:497): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T08:01:39.286584+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108899.285:498): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T08:01:39.287884+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108899.286:499): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T08:02:39.287542+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108959.286:500): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T08:02:39.288536+00:00 k8scilium1 kernel: audit: type=1326 audit(1756108959.287:501): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
2025-08-25T08:03:39.289216+00:00 k8scilium1 kernel: audit: type=1326 audit(1756109019.287:502): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
2025-08-25T08:03:39.289231+00:00 k8scilium1 kernel: audit: type=1326 audit(1756109019.287:503): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=4640 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000

```

We can find reference to syscalls `35` and `202`. To find out which syscall corresponds to which number, we can look on the syscall table available on the [chromium documentation](https://www.chromium.org/chromium-os/developer-library/reference/linux-constants/syscalls/).

| Syscall number | Syscall name |
|-|-|
| 35 | nanosleep |
| 202 | futex |


Another way to identify syscalls used is the `strace` tool.
For this to work, we need the Pid associated to the container. We can get this through `crictl`.

```bash

vagrant@k8scilium1:~$ sudo crictl ps |grep audit-pod
842fd0188adce       04fa556e62bdd       2 hours ago         Running             test-container            0                   6379e71292e6a       audit-pod                            default
vagrant@k8scilium1:~$ 

vagrant@k8scilium1:~$ sudo crictl inspect 842fd0188adce |grep pid
            "pid": 1
    "pid": 4640,
            "type": "pid"

```

Then use strace to get information on the syscall

```bash

cricvagrant@k8scilium1:~$ sudo strace -p 4640 -f
strace: Process 4640 attached with 7 threads
[pid  4658] futex(0xc0000da948, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid  4657] futex(0xc0000da548, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid  4655] epoll_pwait(5,  <unfinished ...>
[pid  4640] futex(0x86d3a8, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid  4656] futex(0x89b738, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid  4653] restart_syscall(<... resuming interrupted futex ...> <unfinished ...>
[pid  4654] futex(0x89b8c0, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
[pid  4653] <... restart_syscall resumed>) = -1 ETIMEDOUT (Connection timed out)
[pid  4653] nanosleep({tv_sec=0, tv_nsec=10000000}, NULL) = 0
[pid  4653] futex(0x86d760, FUTEX_WAIT_PRIVATE, 0, {tv_sec=60, tv_nsec=0}

```

We can see, as expectted, that the syslogs registered are the same as when parsing the syslog. We can also see some additional syscalls such as `epoll_pwait`

So we can create a custom seccomp profile as below.

```json

{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "futex",
                "epoll_pwait",
                "nanosleep"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}

```

Write a pod definition with the corresponding profile.

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: audit-pod-custom
  labels:
    app: audit-pod-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom.json
  containers:
  - name: test-container
    image: hashicorp/http-echo:1.0
    args:
    - "-text=just made some syscalls!"
    securityContext:
      allowPrivilegeEscalation: false

```

Create the pod... and see it failing &#128517;

```bash

vagrant@k8scilium1:~$ k get pod audit-pod-custom 
NAME               READY   STATUS             RESTARTS      AGE
audit-pod-custom   0/1     CrashLoopBackOff   5 (64s ago)   3m58s

vagrant@k8scilium1:~$ k describe pod audit-pod-custom 
Name:              audit-pod-custom
Namespace:         default
Priority:          0
Service Account:   default
Node:              k8scilium1/192.168.56.17
Start Time:        Mon, 25 Aug 2025 11:58:12 +0200
Labels:            app=audit-pod-custom
Annotations:       <none>
Status:            Running
SeccompProfile:    Localhost
LocalhostProfile:  profiles/custom.json
IP:                100.64.0.117
IPs:
  IP:  100.64.0.117
Containers:
  test-container:
    Container ID:  containerd://01a40fc001d251ec4d4aeab63f72e401124efe85914fee427c6ff3feb2713af8
    Image:         hashicorp/http-echo:1.0
    Image ID:      docker.io/hashicorp/http-echo@sha256:fcb75f691c8b0414d670ae570240cbf95502cc18a9ba57e982ecac589760a186
    Port:          <none>
    Host Port:     <none>
    Args:
      -text=just made some syscalls!
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       StartError
      Message:      failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: seek /sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pode0fedf80_62ac_4559_a22f_b4b30e285fc9.slice/cri-containerd-01a40fc001d251ec4d4aeab63f72e401124efe85914fee427c6ff3feb2713af8.scope/cgroup.freeze: no such device: unknown
      Exit Code:    128
      Started:      Thu, 01 Jan 1970 01:00:00 +0100
      Finished:     Mon, 25 Aug 2025 11:58:13 +0200
    Ready:          False
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4nwpt (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-4nwpt:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  14s               default-scheduler  Successfully assigned default/audit-pod-custom to k8scilium1
  Warning  Failed     13s               kubelet            Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: seek /sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pode0fedf80_62ac_4559_a22f_b4b30e285fc9.slice/cri-containerd-test-container.scope/cgroup.freeze: no such device: unknown
  Warning  BackOff    12s               kubelet            Back-off restarting failed container test-container in pod audit-pod-custom_default(e0fedf80-62ac-4559-a22f-b4b30e285fc9)
  Normal   Pulled     1s (x3 over 14s)  kubelet            Container image "hashicorp/http-echo:1.0" already present on machine
  Normal   Created    1s (x3 over 14s)  kubelet            Created container: test-container
  Warning  Failed     0s (x2 over 13s)  kubelet            Error: failed to start containerd task "test-container": cannot start a stopped process: unknown

```

Refering to the Kubernetes documentation, we can see the proposed fine-grained profile is more like this. If you wonder why the `strace` analysis did not gave us all the syscalls, welcome to the band &#128518;.

```json

{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "accept4",
                "epoll_wait",
                "pselect6",
                "futex",
                "madvise",
                "epoll_ctl",
                "getsockname",
                "setsockopt",
                "vfork",
                "mmap",
                "read",
                "write",
                "close",
                "arch_prctl",
                "sched_getaffinity",
                "munmap",
                "brk",
                "rt_sigaction",
                "rt_sigprocmask",
                "sigaltstack",
                "gettid",
                "clone",
                "bind",
                "socket",
                "openat",
                "readlinkat",
                "exit_group",
                "epoll_create1",
                "listen",
                "rt_sigreturn",
                "sched_yield",
                "clock_gettime",
                "connect",
                "dup2",
                "epoll_pwait",
                "execve",
                "exit",
                "fcntl",
                "getpid",
                "getuid",
                "ioctl",
                "mprotect",
                "nanosleep",
                "open",
                "poll",
                "recvfrom",
                "sendto",
                "set_tid_address",
                "setitimer",
                "writev",
                "fstatfs",
                "getdents64",
                "pipe2",
                "getrlimit"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}

```

A new pod with this profile would look like this

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: audit-pod-custom
  labels:
    app: audit-pod-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom.json
  containers:
  - name: test-container
    image: hashicorp/http-echo:1.0
    args:
    - "-text=just made some syscalls!"
    securityContext:
      allowPrivilegeEscalation: false

```

Remember that the profile needs to be available on the seccomp path, which is `var/lib/kubelet/seccomp`.
This time the pod runs. However, it showed us that it's far from easy to get a full list of syscalls for a running container.

Last but not least for this part, it could be tempting to get a custom profile through Gen AI. I won't go to the process on how to get results out of ChatGpt. To summarize, we get a GEN-AI generated profile that looks like this.

```json

{
  "defaultAction": "SCMP_ACT_ERRNO",
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
      ]
    }
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "brk",
        "close",
        "dup",
        "dup2",
        "dup3",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "eventfd2",
        "execve",
        "exit",
        "exit_group",
        "fstat",
        "futex",
        "getpid",
        "getppid",
        "getrandom",
        "madvise",
        "mmap",
        "mprotect",
        "munmap",
        "nanosleep",
        "openat",
        "pipe2",
        "pread64",
        "prlimit64",
        "read",
        "recvfrom",
        "recvmsg",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sendmsg",
        "sendto",
        "setitimer",
        "setsockopt",
        "sigaltstack",
        "socket",
        "stat",
        "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}


```

But the most important thing, IMHO, are the sources mentioned to get this result:

- The Docker documentation, and all the references to the runtime default profile, that we already mentioned.
- The kubernetes documentation, and specifically the fine grained profile that is defined in the samples.
- The [syscall2seccomp github repository](https://github.com/antitree/syscall2seccomp/tree/master), which funily, mention a lot of hours to debug why strace does not show all the required syscalls.

And last, a very interesting article on [4armed.com](https://www.4armed.com/blog/kubernetes-seccomp/) that details some exotic ways to upload custom profile on a cluster. Since this is something that will be helpful for configuring Seccomp on an AKS cluster, we'll discuss this in the next part.


## 3. What about Seccomp on AKS

### 3.1. What we can do through AKS - the normal way

Up until now, we focused on the Seccomp configuration, and how it works, but we did not consider an AKS cluster (or any managed kubernetes cluster for instance) where we don't get access to the nodes.

Out of the box, on an AKS cluster, what can we do?

For a reminder, AKS is composed of node pools, which are actually Virtual Machine Scale Sets, that are visible in the Azure portal, but managed by AKS. 
Which means also that we should not interact with those nodes directly on the portal but only through the AKS API. The counter part is also true by the way, managing nodes from the Kubernetes control plane, while working, may not persist on scaling events for example.

![illustration1](/assets/K8ssecu/aksseccomp001.png)

However, AKS is still a Kubernetes cluster, which follows quite closely the Kubernetes release rhythm. So that means we have all the Seccomp features available that we've discussed, somehow.

There is a [documentation page](https://docs.azure.cn/en-us/aks/secure-container-access) on the topic, but not detailed enough from my point of view.

Considering the following deployment, the default runtime container profile works fine on an AKS cluster.

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demoapp
  name: demoapp
  namespace: demoapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp
  strategy: {}
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      securityContext: 
        seccompProfile:
          type: RuntimeDefault     
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}

```

What else?

Well, we can consider the node customization, described on the [documentation](https://docs.azure.cn/en-us/aks/custom-node-configuration?tabs=linux-node-pools), and on this [community article](https://techcommunity.microsoft.com/blog/appsonazureblog/customising-node-level-configuration-in-aks/4435038).

Looking deeper into the cluster and node pool objects, we can find in the [ARM reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters/agentpools?pivots=deployment-language-terraform) the following paramter inside the kubelet configuration.

![illustration2](/assets/K8ssecu/aksseccomp002.png)

It gets interesting if we cross-reference the previously mentionned documentation:

- On the Seccomp for AKS page, we mentioned the option to use the node customzation  for the use of the default runtime container profile
- On the Node customization reference, we can find samples on how to perform the node customization.

![illustration3](/assets/K8ssecu/aksseccomp003.png)

First thing first, we need some prerequisite. The `KubeletDefaultSeccompProfilePreview` feature should be registered on the AKS provider.

We can check this with the following command.

```bash

yumemaru@azure~$ az feature show --namespace "Microsoft.ContainerService" --name "KubeletDefaultSeccompProfilePreview"
{
  "id": "/subscriptions/49816259-cb52-4fe5-8d6f-9358ad94332c/providers/Microsoft.Features/providers/Microsoft.ContainerService/features/KubeletDefaultSeccompProfilePreview",
  "name": "Microsoft.ContainerService/KubeletDefaultSeccompProfilePreview",
  "properties": {
    "state": "Registered"
  },
  "type": "Microsoft.Features/providers/features"
}

```

If it shows `Registered`, everything is fine, if not, it must be registered. Again, the AKS Seccomp doc details the steps required.

Once we are ok on this feature activation, the next step is the customization of the nodes. It works with the `--kubelet-config` and a json file which is read when creating either the cluster or the node pool.

We will test this with a node pool here. The command to create the node pool would look like this.

```bash

az aks nodepool add --name <nodepool_name> --cluster-name myAKSCluster --resource-group <resourcegroup_name> --kubelet-config ./linuxkubeletconfig.json

```

And the json file to pass through the cli looks like this.

```json

{
  "seccompDefault": "RuntimeDefault"
}

```

After the provisioning of the node pool, we can check its kubelet configuration, versus the other(s) node pools for which we did not specified the seccomp parameter.

```bash

yumemaru@azure~$ az aks nodepool list --cluster-name <aks_cluster_name> -g <aks_rg_name> -o json | jq '.[].name,.[].kubeletConfig.seccompDefault'
"aksnp0lab"
"npuser1"
"npseccomp"
null
null
"RuntimeDefault"

```

It's mentionned in the AKS documentation that afterward, we should connect to the node(s) and check the seccomp configuration, but the steps are not specified (I did say it was not detailed enough for me...). Instead, we will create 2 differents pods, with node afinities. For this test, I added a specific label to my nodes through the AKS command `az aks nodepool update` and the `--labels` argument. Specifically, I set the label `seccompDefaultEnabled=true/false`.

```bash

yumemaru@azure$ az aks nodepool update --cluster-name <aks_cluster_name> -g <aks_rg_name> --name aksnp0lab --labels seccompDefaultEnabled=false

```

The nodes' labels can be displayed with an `kubectl` command.

```bash


yumemaru@azure~$ k get nodes -o json | jq .items[].metadata.labels
{
  "agentpool": "aksnp0lab",
======truncated======
  "kubernetes.io/hostname": "aks-aksnp0lab-61429621-vmss000000",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "standard_d2s_v4",
  "seccompDefaultEnabled": "false",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.disk.csi.azure.com/zone": "",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "0"
}
{
  "agentpool": "aksnp0lab",
======truncated======
  "kubernetes.io/hostname": "aks-aksnp0lab-61429621-vmss000001",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "standard_d2s_v4",
  "seccompDefaultEnabled": "false",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.disk.csi.azure.com/zone": "",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "0"
}
{
  "agentpool": "aksnp0lab",
======truncated======
  "kubernetes.io/hostname": "aks-aksnp0lab-61429621-vmss000002",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "standard_d2s_v4",
  "seccompDefaultEnabled": "false",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.disk.csi.azure.com/zone": "",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "0"
}
{
  "agentpool": "npseccomp",
======truncated======
  "kubernetes.io/hostname": "aks-npseccomp-10613327-vmss000001",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "Standard_D4ds_v5",
  "seccompDefaultEnabled": "true",
  "topology.disk.csi.azure.com/zone": "francecentral-2",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "francecentral-2"
}
{
  "agentpool": "npuser1",
======truncated======
  "kubernetes.io/hostname": "aks-npuser1-78178449-vmss000000",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "Standard_D2S_v4",
  "seccompDefaultEnabled": "false",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.disk.csi.azure.com/zone": "francecentral-2",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "francecentral-2"
}
{
  "agentpool": "npuser1",
======truncated======
  "kubernetes.io/hostname": "aks-npuser1-78178449-vmss000002",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "Standard_D2S_v4",
  "seccompDefaultEnabled": "false",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.disk.csi.azure.com/zone": "francecentral-1",
  "topology.kubernetes.io/region": "francecentral",
  "topology.kubernetes.io/zone": "francecentral-1"
}

```

Finally, to test the differences, we deploy pods with the `amicontained` image, an useful container image for Seccomp analysis taht I found parsing [kodekloud notes](https://notes.kodekloud.com/docs/Certified-Kubernetes-Security-Specialist-CKS/System-Hardening/Implement-Seccomp-in-Kubernetes).

```yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: amicontained
  name: amicontained
spec:
  affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: seccompDefaultEnabled
                operator: In
                values:
                - "true" 
#  securityContext:
#    seccompProfile:
#      type: Unconfined #RuntimeDefault #
  containers:
    - args:
        - amicontained
      image: yumemaru1979/amicontained
      name: amicontained
      securityContext:
        allowPrivilegeEscalation: false
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: amicontained-nodeselector
  name: amicontained-nodeselector
spec:
  affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: seccompDefaultEnabled
                operator: In
                values:
                - "true" 
#  securityContext:
#    seccompProfile:
#      type: Unconfined #RuntimeDefault #
  containers:
    - args:
        - amicontained
      image: yumemaru1979/amicontained
      name: amicontained-nodeselector
      securityContext:
        allowPrivilegeEscalation: false

```

Creating the pods, we can check the expected nodes are used..
We can also see that, because we did not specified the `spec.securitycontext.seccompProfile`, we do not see the configuration of a seccomp profile reflected in the pod configuration output.

```bash

yumemaru@azure~$ k get pod -o wide
NAME                        READY   STATUS    RESTARTS        AGE     IP             NODE                                NOMINATED NODE   READINESS GATES
amicontained                1/1     Running   1 (3m35s ago)   3m36s   100.64.3.81    aks-npuser1-78178449-vmss000002     <none>           <none>
amicontained-nodeselector   1/1     Running   0               12s     100.64.6.122   aks-npseccomp-10613327-vmss000001   <none>           <none>

yumemaru@azure~$ k get pod -o json |jq .items[].spec.securityContext
{}
{}


```

However, when we check the log of the pods, we can see the Seccomp value either to disabled or filtering, depending on the node pool configuration. We can also see the difference with the number of syscalls filtered for the different nodes.

```bash

yumemaru@azure~$ k logs amicontained
Container Runtime: not-found
Has Namespaces:
        pid: true
        user: false
AppArmor Profile: cri-containerd.apparmor.d (enforce)
Capabilities:
        BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: disabled
Blocked Syscalls (21):
        SYSLOG SETPGID SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY SWAPON REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE KEXEC_LOAD PERF_EVENT_OPEN FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD
Looking for Docker.sock

yumemaru@azure~$ k logs amicontained-nodeselector 
Container Runtime: not-found
Has Namespaces:
        pid: true
        user: false
AppArmor Profile: cri-containerd.apparmor.d (enforce)
Capabilities:
        BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (55):
        MSGRCV SYSLOG SETPGID SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT OPEN_BY_HANDLE_AT SETNS KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD
Looking for Docker.sock


```

So we have access to the default seccomp profile, either through the pods configuration, or by setting the parameter on the node pools. The second option is quite interesting to enforce default security, at the node level rather than the pod level.

Now what about the use of custom profile?

### 3.2. What we can do on AKS - the exotic way

Currently, there are only 2 options for the seccomp configuration at the node level. Either we don't set anything and we get the unconfined profile, or we set the default runtime profile, which allows for a minimal security baseline.

As we've seen in the previous part, the use of custom profiles requires access to the node, by adding the profiles definition in a specific folder.

And that's where the trouble start, because we're not really supposed to access the node, and it implies that we access each nodes, every time an scale-out event occurs.

If we take a scenario with a node pool without autoscaling, we can use the `kubectl node-shell` command as specified in the [documentation](https://docs.azure.cn/en-us/aks/manage-ssh-node-access?tabs=node-shell) to access the node and add the seccomp profile file in the `/var/lib/kubelet/seccomp` folder

```bash

yumemaru@azure~$ k node-shell aks-npseccomp-10613327-vmss000001
spawning "nsenter-j91j0e" on "aks-npseccomp-10613327-vmss000001"
If you don't see a command prompt, try pressing enter.

```

```bash 

root@aks-npseccomp-10613327-vmss000001:/#
root@aks-npseccomp-10613327-vmss000001:/# mkdir -p /var/lib/kubelet/seccomp/profiles
root@aks-npseccomp-10613327-vmss000001:/# vim /var/lib/kubelet/seccomp/profiles/audit.json

```

```json

{
    "defaultAction": "SCMP_ACT_LOG"
}

```

Then we can create a pod using this profile, from another terminal not connected to the node

```bash

yumemaru@azure~$ k get pod audit-pod 
NAME        READY   STATUS    RESTARTS   AGE
audit-pod   1/1     Running   0          7m24s

yumemaru@azure~$ k describe pod audit-pod 
Name:              audit-pod
Namespace:         default
Priority:          0
Service Account:   default
Node:              aks-npseccomp-10613327-vmss000001/172.21.17.75
Start Time:        Tue, 26 Aug 2025 15:43:31 +0200
Labels:            app=audit-pod
Annotations:       <none>
Status:            Running
SeccompProfile:    Localhost
LocalhostProfile:  profiles/audit.json
IP:                100.64.6.129
IPs:
  IP:  100.64.6.129
Containers:
  test-container:
    Container ID:  containerd://c378a2d81cb1b4dc17ab06791114888317fc48fb7169bfd858c72f99b154d0a3
    Image:         hashicorp/http-echo:1.0
    Image ID:      docker.io/hashicorp/http-echo@sha256:fcb75f691c8b0414d670ae570240cbf95502cc18a9ba57e982ecac589760a186
    Port:          <none>
    Host Port:     <none>
    Args:
      -text=just made some syscalls!
    State:          Running
      Started:      Tue, 26 Aug 2025 15:43:34 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pnc8t (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-pnc8t:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  13s   default-scheduler  Successfully assigned default/audit-pod to aks-npseccomp-10613327-vmss000001
  Normal  Pulling    12s   kubelet            Pulling image "hashicorp/http-echo:1.0"
  Normal  Pulled     10s   kubelet            Successfully pulled image "hashicorp/http-echo:1.0" in 2.344s (2.344s including waiting). Image size: 4631705 bytes.
  Normal  Created    10s   kubelet            Created container: test-container
  Normal  Started    10s   kubelet            Started container test-container



```

And going back into the node's shell, check the syslog for the syscall audit log

```bash

root@aks-npseccomp-10613327-vmss000001:/# tail /var/log/syslog | grep 'http-echo'
Aug 26 13:45:34 aks-npseccomp-10613327-vmss000001 kernel: [18362.129539] audit: type=1326 audit(1756215934.835:338): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=267486 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=35 compat=0 ip=0x4685d7 code=0x7ffc0000
Aug 26 13:45:34 aks-npseccomp-10613327-vmss000001 kernel: [18362.129613] audit: type=1326 audit(1756215934.835:339): auid=4294967295 uid=65532 gid=65532 ses=4294967295 subj=cri-containerd.apparmor.d pid=267486 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x468ba3 code=0x7ffc0000
root@aks-npseccomp-10613327-vmss000001:/# 

```

And that works.
But that's not very practical. So how could we avoid creating manually on each nodes the profiles?

Time to refers back to the [4armed.com](https://www.4armed.com/blog/kubernetes-seccomp/). It leverage the use of an init container to create the

From a custom seccomp profile file, we create a secret.

```bash

yumemaru@azure~$ k create secret generic customseccompprofilesecret --from-file <path_to_profile> --dry-run=client -o yaml > <path_to_yaml_file>

```

We then use this secret as a volume in a pod which also mount the `/var/lib/kubelet` folder from the host. The init container uses a busybox container and pass the `mkdir -p /host/seccomp && cp /seccomp/*.json /host/seccomp/` command

```yaml

apiVersion: v1
data:
  customprofile.json: ewogIC=====truncated=====BdCn0K
kind: Secret
metadata:
  creationTimestamp: null
  name: customseccompprofilesecret
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  volumes:
  - name: hostkubelet
    hostPath:
      path: /var/lib/kubelet
      type: Directory
  - name: seccomp-profiles
    secret:
      secretName: customseccompprofilesecret
  - name: localvol
    emptyDir: {}
  initContainers:
  - name: seccomp
    image: busybox
    volumeMounts:
    - name: hostkubelet
      mountPath: /host
    - name: seccomp-profiles
      mountPath: /seccomp
    - name: localvol
      mountPath: /local
    command:
    - "sh"
    - "-c"
    - "mkdir -p /host/seccomp && cp /seccomp/*.json /host/seccomp/; test -f /host/seccomp/customprofile.json && echo 'customprofile.json exists.' > /local/checkfile"
  containers:
  - name: web
    image: nginx
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: customprofile.json
    volumeMounts:
    - name: localvol
      mountPath: /local
    livenessProbe:
      exec:
        command:
        - cat
        - /local/checkfile
    resources: {}

```

We can check that everything works.

```bash

yumemaru@azure~$ k get pod nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP            NODE                              NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          3m31s   100.64.4.74   aks-npuser1-78178449-vmss000000   <none>           <none>

yumemaru@azure~$ k describe pod nginx 
Name:             nginx
Namespace:        default
Priority:         0
Service Account:  default
Node:             aks-npuser1-78178449-vmss000000/172.21.17.72
Start Time:       Tue, 26 Aug 2025 16:34:15 +0200
Labels:           app=nginx
Annotations:      <none>
Status:           Running
IP:               100.64.4.74
IPs:
  IP:  100.64.4.74
Init Containers:
  seccomp:
    Container ID:  containerd://0b04053cb97a138396b380d1e5bdbfb4d55c87939464fe16bef8399095078186
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:ab33eacc8251e3807b85bb6dba570e4698c3998eca6f0fc2ccb60575a563ea74
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      mkdir -p /host/seccomp && cp /seccomp/*.json /host/seccomp/
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 26 Aug 2025 16:34:18 +0200
      Finished:     Tue, 26 Aug 2025 16:34:18 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /host from hostkubelet (rw)
      /seccomp from seccomp-profiles (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ckk5p (ro)
Containers:
  web:
    Container ID:        containerd://0844ee9181cbd7cff4ad5aa7db614df8a4c4d50a40795969cfc019d9d4a250c2
    Image:               nginx
    Image ID:            docker.io/library/nginx@sha256:33e0bbc7ca9ecf108140af6288c7c9d1ecc77548cbfd3952fd8466a75edefe57
    Port:                <none>
    Host Port:           <none>
    SeccompProfile:      Localhost
      LocalhostProfile:  customprofile.json
    State:               Running
      Started:           Tue, 26 Aug 2025 16:34:19 +0200
    Ready:               True
    Restart Count:       0
    Environment:         <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ckk5p (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  hostkubelet:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet
    HostPathType:  Directory
  seccomp-profiles:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  customseccompprofilesecret
    Optional:    false
  kube-api-access-ckk5p:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  34s   default-scheduler  Successfully assigned default/nginx to aks-npuser1-78178449-vmss000000
  Normal  Pulling    34s   kubelet            Pulling image "busybox"
  Normal  Pulled     31s   kubelet            Successfully pulled image "busybox" in 2.501s (2.501s including waiting). Image size: 2223685 bytes.
  Normal  Created    31s   kubelet            Created container: seccomp
  Normal  Started    31s   kubelet            Started container seccomp
  Normal  Pulling    31s   kubelet            Pulling image "nginx"
  Normal  Pulled     30s   kubelet            Successfully pulled image "nginx" in 737ms (737ms including waiting). Image size: 72324501 bytes.
  Normal  Created    30s   kubelet            Created container: web
  Normal  Started    30s   kubelet            Started container web

```

Connecting to the node, we can see that the profile is present as expected.

```bash

yumemaru@azure~$ k node-shell aks-npuser1-78178449-vmss000000
spawning "nsenter-oc2wnq" on "aks-npuser1-78178449-vmss000000"
If you don't see a command prompt, try pressing enter.
root@aks-npuser1-78178449-vmss000000:/# ls /var/lib/kubelet/seccomp/
customprofile.json
root@aks-npuser1-78178449-vmss000000:/# cat /var/lib/kubelet/seccomp/customprofile.json 
{
  "defaultAction": "SCMP_ACT_LOG",
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
      ]
    }
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "brk",
        "close",
        "dup",
        "dup2",
        "dup3",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "eventfd2",
        "execve",
        "exit",
        "exit_group",
        "fstat",
        "futex",
        "getpid",
        "getppid",
        "getrandom",
        "madvise",
        "mmap",
        "mprotect",
        "munmap",
        "nanosleep",
        "openat",
        "pipe2",
        "pread64",
        "prlimit64",
        "read",
        "recvfrom",
        "recvmsg",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sendmsg",
        "sendto",
        "setitimer",
        "setsockopt",
        "sigaltstack",
        "socket",
        "stat",
        "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}

```

That's that.

Additionaly, we could also use the same concept but instead of an init container, which impact the time to start the pod, use a daemonset that would copy the profile on each node. A yaml definition would look like this.

```yaml

apiVersion: v1
data:
  customprofileds.json: ewogIC=====truncated=====BdCn0K
kind: Secret
metadata:
  name: anothercustomseccompprofilesecret
  namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: customseccompprofileset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: seccomp-ds
  template:
    metadata:
      labels:
        name: seccomp-ds
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: copycustomseccomp
        image: busybox
        resources: {}
        volumeMounts:
        - name: hostkubelet
          mountPath: /host
        - name: seccomp-profiles
          mountPath: /seccomp
        - name: localvol
          mountPath: /local
        command:
        - "sh"
        - "-c"
        - "mkdir -p /host/seccomp && cp /seccomp/*.json /host/seccomp/; test -f /host/seccomp/customprofileds.json && echo 'customprofileds.json exists.' > /local/checkfile; sleep 15"
      terminationGracePeriodSeconds: 30
      volumes:
      - name: hostkubelet
        hostPath:
          path: /var/lib/kubelet
          type: Directory
      - name: seccomp-profiles
        secret:
          secretName: anothercustomseccompprofilesecret
      - name: localvol
        emptyDir: {}
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-testds
  labels:
    app: ginx-testds
spec:
  volumes:
  - name: hostkubelet
    hostPath:
      path: /var/lib/kubelet
      type: Directory
  containers:
  - name: web
    image: nginx
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: customprofileds.json
    volumeMounts:
    - name: hostkubelet
      mountPath: /host
      readOnly: true
    readinessProbe:
      exec:
        command:
        - "sh"
        - "-c"
        - test -f /host/seccomp/customprofileds.json && echo 'customprofileds.json exists.'
      initialDelaySeconds: 20
      periodSeconds: 5
    resources: {}

```

One might question the relevance of mounting folders from the host to achieve this security requirement though &#129323;

**Note:** While those methods works on a cluster without access to the nodes, the results would be the sames with a self-managed cluster on which we can access to the nodes.

To avoid that, we could use the dedicated operator to manage seccomp profile. 

The documentation can be found [here](https://github.com/kubernetes-sigs/security-profiles-operator/tree/main). Because it deserves a more thorough study, we are only mentionning today, and may come back to it another day.

Ok time to wrap up!

## 4. Summary

Soooooo!

It's been an eventful journey right?

To summarize:

- Seccomp is a powerful (while not new) tool to give us a measure of security on kubernetes
- As expected for a security feature, it implies that we know a bit about kubernetes and the potential limitations of the environment.
- There is room for customisation, but one would say that the minimum requirement should be to at least enforce the runtime's default profile.
- Because AKS also needs security, and people at microsoft think about our need, we found that there is a way to enforce this default profile inb Azure environment
- Custom profiles are a pain, both for the creation but to manage, specifically on multi-nodes clusters, and even more on cloud-managed cluster. While we can find ways to push profiles to clusters, creating profiles is far from easy.

And that's all!

I hope it was useful. Now I'll go back to study to the CKS, which is far from done &#129322;

Until then...









