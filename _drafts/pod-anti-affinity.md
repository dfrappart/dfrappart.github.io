# Pod anti-affinity explained

## The rule

```json
{
  "podAntiAffinity": {
    "preferredDuringSchedulingIgnoredDuringExecution": [
      {
        "podAffinityTerm": {
          "labelSelector": {
            "matchLabels": {
              "app": "nginx-unprivileged"
            }
          },
          "topologyKey": "kubernetes.io/hostname"
        },
        "weight": 100
      }
    ]
  }
}
```

## What it does

This is a soft ("preferred") pod anti-affinity rule. In plain terms, it tells the scheduler: try not to put this pod on a node that already runs a pod labelled `app=nginx-unprivileged`.

| Field | Meaning |
|---|---|
| `podAntiAffinity` | Keep this pod away from certain other pods. `podAffinity` would pull it toward them instead. |
| `preferredDuringScheduling…` | This is a preference, not a requirement. If no node satisfies it, the pod is still scheduled, even next to a matching pod. |
| `…IgnoredDuringExecution` | The rule only applies when the pod is scheduled. If labels change later, or pods move, nothing gets evicted. |
| `labelSelector.matchLabels: app=nginx-unprivileged` | Which pods to avoid. By default only pods in the same namespace as this one are checked. |
| `topologyKey: kubernetes.io/hostname` | The "domain" to stay apart in. Every node has a unique hostname label, so the domain is a single node. |
| `weight: 100` | How strongly the scheduler wants this (1–100). For each candidate node, the scheduler adds the weight of every preferred term that node satisfies to the node's score. 100 is the strongest preference possible. |

When this sits in the Deployment's own pod template and the selector matches the pod's own label, the effect is to **spread the replicas across nodes, best effort**. With 3 replicas and 3 or more nodes, you usually get one per node. With 3 replicas and 2 nodes, the third pod still runs, sharing a node, and doesn't get stuck in `Pending`. That's why this is a safe default for high availability.

## Other ways to write anti-affinity

Reference: [Assigning Pods to Nodes – Inter-pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

### 1. Hard anti-affinity (required)

Two matching pods are never placed on the same node. If replicas outnumber the eligible nodes, the extra pods stay `Pending`.

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: nginx-unprivileged
    topologyKey: kubernetes.io/hostname
```

There's no `weight` and no `podAffinityTerm` wrapper here; the term goes directly in the list.

### 2. Spread across availability zones instead of nodes

Change the topology domain:

```yaml
topologyKey: topology.kubernetes.io/zone
```

In AKS this keeps replicas in different zones (1/2/3), which protects against losing a whole zone.

### 3. Several weighted preferences

Here the scheduler strongly prefers a different zone and, less strongly, a different node:

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels: { app: nginx-unprivileged }
      topologyKey: topology.kubernetes.io/zone
  - weight: 50
    podAffinityTerm:
      labelSelector:
        matchLabels: { app: nginx-unprivileged }
      topologyKey: kubernetes.io/hostname
```

### 4. Mixing hard and soft rules

You can use `required` for zones and `preferred` for nodes in the same `podAntiAffinity` block.

### 5. Selecting pods with `matchExpressions`

This gives more flexible matching (`In`, `NotIn`, `Exists`, `DoesNotExist`):

```yaml
labelSelector:
  matchExpressions:
  - key: app
    operator: In
    values: [nginx-unprivileged, nginx]
```

### 6. Matching pods in other namespaces

Use `namespaces` (a fixed list) or `namespaceSelector`. An empty `namespaceSelector: {}` matches all namespaces:

```yaml
- labelSelector:
    matchLabels: { app: nginx-unprivileged }
  namespaceSelector:
    matchLabels: { team: web }
  topologyKey: kubernetes.io/hostname
```

### 7. `matchLabelKeys` / `mismatchLabelKeys`

These are newer fields (beta from Kubernetes 1.31). They add the values of the incoming pod's own labels to the selector. A common use is `matchLabelKeys: [pod-template-hash]`, so a rolling update only spreads pods of the same ReplicaSet and isn't blocked by pods from the old version:

```yaml
- labelSelector:
    matchLabels: { app: nginx-unprivileged }
  matchLabelKeys: [pod-template-hash]
  topologyKey: kubernetes.io/hostname
```

### 8. Alternative: `topologySpreadConstraints`

If the goal is an even spread rather than "never together", the docs recommend [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/). They let you set an allowed imbalance (`maxSkew`) and choose what happens when the spread can't be met (`whenUnsatisfiable: DoNotSchedule | ScheduleAnyway`):

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels: { app: nginx-unprivileged }
```

## Caveats from the docs

- Inter-pod affinity and anti-affinity take a lot of processing and can slow scheduling noticeably in clusters with more than several hundred nodes.
- Every node needs the label named in `topologyKey`, or behaviour can be unexpected.
