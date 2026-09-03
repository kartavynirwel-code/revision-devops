# Day 8 — Kubernetes Advanced Objects Revision Notes

## 1. StatefulSet vs Deployment

### Why StatefulSet Exists
- A `Deployment`'s Pods are interchangeable — any Pod can be killed and replaced, gets a new random name/IP, no guaranteed order. Fine for stateless apps (web servers, APIs).
- A `StatefulSet` is for workloads that need **stable, unique identity** across restarts — databases, Kafka brokers, Zookeeper, Elasticsearch nodes.

### What StatefulSet Guarantees
- **Stable network identity**: Pods get predictable names — `mysql-0`, `mysql-1`, `mysql-2` (not random suffixes like Deployments). Combined with a **headless Service**, each Pod gets a stable DNS name: `mysql-0.mysql-headless.default.svc.cluster.local`.
- **Ordered deployment/scaling**: Pods are created/deleted in order (`0`, then `1`, then `2`) — important for things like a primary DB node needing to exist before replicas join.
- **Stable storage**: Each Pod gets its own `PersistentVolumeClaim` that persists across Pod rescheduling — `mysql-0` always reattaches to the same volume, even if rescheduled to a different node.

### Example Structure
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql-headless"   # must point to a headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```
- **Interview point**: `volumeClaimTemplates` is the key differentiator — it auto-creates a unique PVC per replica, whereas a Deployment sharing one PVC across replicas would cause data corruption for a database.

## 2. DaemonSet

### What It Does
- Ensures **exactly one Pod runs on every node** (or a filtered subset via node selectors/taints-tolerations) — not a configurable replica count like Deployment.
- New node joins the cluster → DaemonSet Pod automatically scheduled there. Node removed → Pod garbage collected.

### Common Real-World Uses
- **Log collectors**: Fluentd/Fluent Bit/Filebeat — need to read logs from every node's filesystem.
- **Monitoring agents**: Node Exporter (Prometheus), Datadog agent — need per-node system metrics (CPU, disk, network at the host level).
- **Networking/CNI plugins**: Calico, Cilium — need to run on every node to manage pod networking.

### Interview Point
- Why not just use a Deployment with `replicas = number of nodes`? Because that count is manual and doesn't auto-adjust as nodes are added/removed — DaemonSet inherently ties to "one per node" as a guarantee, not a number you maintain.

## 3. HPA (Horizontal Pod Autoscaler)

### What It Does
- Automatically scales the **number of Pod replicas** in a Deployment/StatefulSet based on observed metrics (CPU, memory, or custom metrics via metrics adapters).

### Example
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```
- **Requires**: `metrics-server` installed in the cluster to provide resource metrics (CPU/memory). Custom metrics (like queue length, requests/sec) need a metrics adapter (e.g., Prometheus Adapter).
- **Interview point**: HPA scales **Pods horizontally** (more replicas). Contrast with **VPA (Vertical Pod Autoscaler)**, which adjusts a Pod's CPU/memory requests/limits instead. They generally shouldn't be used together on the same metric — conflicting adjustments.

## 4. PodDisruptionBudget (PDB)

### What It Does
- Limits how many Pods of an application can be **voluntarily** disrupted at once (node drains for maintenance, cluster upgrades, `kubectl drain`) — does NOT protect against involuntary disruptions (node crash, OOM kill).
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: 2      # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: backend
```
- **Why it matters**: Without a PDB, a cluster admin draining a node for maintenance could accidentally kill enough replicas of your app simultaneously to cause a full outage. PDB tells Kubernetes "never let available replicas drop below this number during voluntary drains."

## 5. Resource Requests/Limits + QoS Classes

### Requests vs Limits
- **Request**: What the scheduler guarantees/reserves for the Pod — used for scheduling decisions (which node has room).
- **Limit**: The hard ceiling — Pod gets CPU-throttled if it exceeds CPU limit, or **OOMKilled** if it exceeds memory limit.
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### QoS Classes (Kubernetes derives this automatically)
- **Guaranteed**: requests == limits for both CPU and memory on every container — highest priority, last to be evicted under node pressure.
- **Burstable**: requests set but less than limits (or only some resources set) — medium priority.
- **BestEffort**: no requests/limits set at all — first to be evicted under node memory pressure.
- **Interview point**: This directly answers "why did my Pod get evicted before another one on a memory-pressured node?" — QoS class ranking is the mechanism.

---

## Quick Self-Test (do this without looking)
1. Why would you use a StatefulSet instead of a Deployment for a database, specifically regarding storage?
2. A DaemonSet exists for log collection — why not just set a Deployment's replica count equal to the node count?
3. What's the difference between what HPA scales and what VPA scales?
4. Two Pods are on a memory-pressured node — one is BestEffort QoS, one is Guaranteed. Which gets evicted first, and why?
