# Day 17 — Kubernetes Scheduling Revision Notes

## 1. Taints and Tolerations

### Core Idea
- **Taint**: Applied to a **Node** — repels Pods from scheduling there unless the Pod explicitly "tolerates" it. Think of it as the node saying "don't schedule here unless you specifically say you're okay with this condition."
- **Toleration**: Applied to a **Pod** — allows (but doesn't force) that Pod to be scheduled on a tainted node.

### Applying a Taint
```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```
- Format: `key=value:effect`

### Taint Effects
- **NoSchedule**: New Pods without a matching toleration won't be scheduled here. Existing Pods already running are unaffected.
- **PreferNoSchedule**: Soft version — scheduler tries to avoid this node but will use it if there's no better option.
- **NoExecute**: Strongest — new Pods without toleration won't schedule, AND existing Pods without toleration get **evicted** if the taint is added while they're already running.

### Matching Toleration
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

### Real-World Use Case
- Dedicating specific nodes to specific workloads — e.g., GPU nodes tainted so only ML workload Pods (with the matching toleration) land there, keeping regular workloads off expensive GPU hardware.
- Also how Kubernetes itself keeps regular Pods off control-plane nodes by default (`node-role.kubernetes.io/control-plane:NoSchedule` taint).

### Interview Point
> "Taints and tolerations are a **repel** mechanism — the node pushes Pods away unless they explicitly tolerate it. Tolerating a taint doesn't guarantee a Pod lands there, it just makes it *possible* — for guaranteeing placement, you'd combine this with node affinity."

## 2. Node Affinity / Anti-Affinity

### Core Idea
- Opposite direction from taints — this is the **Pod** expressing a preference for *which nodes* it wants, based on Node labels.

### Example
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - us-east-1a
```
- **`requiredDuringSchedulingIgnoredDuringExecution`**: Hard requirement — Pod won't schedule at all unless satisfied (like a mandatory filter).
- **`preferredDuringSchedulingIgnoredDuringExecution`**: Soft preference with a weight — scheduler tries to honor it but will schedule elsewhere if needed.
- **"IgnoredDuringExecution"** (in both): Once the Pod is running, if node labels change and no longer match, the Pod is **not** evicted — the rule only applies at scheduling time.

### Interview Point — Taints/Tolerations vs Node Affinity
> "They solve complementary problems. Taints/tolerations let a node **repel** Pods it doesn't want. Node affinity lets a Pod **attract** itself toward nodes it wants. Combining both — taint the GPU nodes AND give the ML Pods both the toleration and a required node affinity for `disktype=gpu` labeled nodes — guarantees the ML Pods land exactly there and nothing else does."

## 3. Pod Affinity / Anti-Affinity

### Core Idea
- Unlike node affinity (Pod-to-Node relationship), this is **Pod-to-Pod** — schedule this Pod relative to where *other Pods* are running.

### Pod Affinity Example (co-locate)
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - cache
      topologyKey: "kubernetes.io/hostname"
```
- **Use case**: Schedule an app Pod on the **same node** as its cache Pod to minimize network latency (co-location).

### Pod Anti-Affinity Example (spread out)
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - backend
      topologyKey: "kubernetes.io/hostname"
```
- **Use case**: Spread replicas of the **same app** across different nodes so a single node failure doesn't take down every replica — critical for high availability.
- `topologyKey` defines the "domain" of spreading — `kubernetes.io/hostname` means "per node," but could also be `topology.kubernetes.io/zone` to spread across **availability zones** instead.

### Interview Point
> "Pod anti-affinity with `topologyKey: zone` is how you actually guarantee zone-level high availability for a Deployment — replicas spread across AZs means an entire AZ outage doesn't take down your whole service, which plain `replicas: 3` alone does NOT guarantee (the scheduler could put all 3 in the same zone)."

## 4. Priority Classes and Preemption

### PriorityClass
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production workloads"
```
- Assign to a Pod via `priorityClassName: high-priority` in its spec.
- Higher `value` = higher priority.

### Preemption
- When a high-priority Pod can't be scheduled due to insufficient resources, the scheduler can **evict lower-priority Pods** on a node to make room — this is preemption.
- **Interview point**: This is why critical workloads (like core system components: CoreDNS, CNI daemonsets) often get high PriorityClasses — ensures they get scheduled even under resource pressure, at the expense of evicting less critical Pods.

### Custom Schedulers (brief awareness)
- Kubernetes allows running **multiple schedulers** simultaneously — a Pod specifies `schedulerName: my-custom-scheduler` to opt into a non-default scheduler.
- Real-world use: specialized scheduling logic for batch/ML workloads (e.g., gang scheduling — all-or-nothing scheduling for distributed training jobs) that the default scheduler doesn't natively support.

---

## Quick Self-Test (do this without looking)
1. What's the fundamental directional difference between a taint/toleration and a node affinity rule?
2. Why does `replicas: 3` on a Deployment NOT guarantee zone-level high availability by itself, and what fixes that?
3. What actually happens when a `NoExecute` taint is added to a node that already has running Pods without a matching toleration?
4. A high-priority Pod can't be scheduled due to resource pressure — what mechanism lets the scheduler make room for it, and what happens to the Pods that get displaced?
