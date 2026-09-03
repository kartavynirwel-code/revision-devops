# Day 14 — Backup/DR + Cost Optimization + Full Mixed Recall

## 1. Velero (Cluster Backup/Restore)

### What It Does
- Backs up Kubernetes **objects** (Deployments, Services, ConfigMaps, etc. — the etcd-level resource definitions) AND optionally the **persistent volume data** attached to them, storing backups in object storage (S3, etc.).
- Used for: disaster recovery, cluster migration (move workloads to a new cluster), and periodic scheduled backups of namespaces.

### Core Commands
```bash
velero backup create my-backup --include-namespaces=my-namespace
velero backup get
velero restore create --from-backup my-backup

velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces=my-namespace
# cron-style scheduled backups
```
- **How PV data backup works**: Velero uses **volume snapshots** (via the cloud provider's native snapshot API — EBS snapshots on AWS) for efficient backup, or **File System Backup** (restic/kopia integration) for volume types that don't support native snapshots.
- **Interview point**: Velero backs up the **desired state definitions**, not a live memory dump — restoring re-creates the objects, and Kubernetes controllers reconcile from there (e.g., a restored Deployment spins up new Pods from scratch, restored PVs reattach from snapshots).

## 2. etcd Backup Basics

- `etcd` is the cluster's **source of truth** — every K8s object (Pods, Secrets, ConfigMaps, RBAC rules, everything) lives here. Losing etcd without a backup means losing the entire cluster's state.
- **Backup command** (on a control plane node with etcd access):
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```
- **Restore** requires stopping the API server, restoring the snapshot to a fresh data directory, and restarting — a more invasive process than Velero's object-level restore.
- **Interview point — Velero vs etcd backup**: etcd backup is the **lowest-level, full cluster disaster recovery** mechanism (rebuild the entire cluster's brain from scratch). Velero operates at a **higher, more granular level** (backup/restore specific namespaces or resources, migrate between clusters) and is what you'd use for day-to-day operational recovery, not a full control-plane disaster.
- Managed Kubernetes (EKS, GKE, AKS) handles etcd backup for you as part of the control plane service — you generally only deal with etcd backup directly on **self-managed** clusters (kubeadm, on-prem).

## 3. Cost Optimization

### Spot Instances
- Spare cloud capacity sold at a steep discount (up to 70-90% off on-demand pricing) — but can be **reclaimed by the cloud provider with short notice** (AWS gives a 2-minute warning via an interruption notice).
- **Good fit**: Stateless, fault-tolerant, interruption-tolerant workloads — batch jobs, CI runners, stateless web tiers with enough replicas that losing one node isn't a problem.
- **Bad fit**: Stateful workloads (databases), anything without graceful handling of sudden Pod eviction.
- In EKS: create a separate **node group** using Spot instances, use taints/tolerations or node affinity to schedule only appropriate workloads there, and combine with **Cluster Autoscaler** to replace reclaimed nodes automatically.

### Resource Right-Sizing
- Setting requests/limits too high wastes cluster capacity (you're reserving more than you use, blocking other Pods from scheduling); too low risks throttling/OOMKills.
- Tools like **Vertical Pod Autoscaler (VPA)** in recommendation mode, or metrics-based analysis (actual usage vs requested), help right-size over time instead of guessing.
- **Interview point**: This connects back to Day 8's QoS classes — over-requesting resources you don't use is a common, invisible cost leak that doesn't show up until someone actually reviews utilization metrics.

### Cluster Autoscaler (Cost Angle)
- Scales the **number of nodes** up/down based on whether Pods are unschedulable (need more nodes) or nodes are underutilized (can be safely drained and removed).
- Combined with HPA (Day 8: scales Pods) and Spot instances, this is the standard three-layer cost/scale strategy: **HPA scales Pod count → Cluster Autoscaler scales node count to fit → Spot instances reduce the cost of those nodes**.

---

## Full 14-Day Mixed Recall — Say These Out Loud Without Looking

1. `git reflog` recovery flow — full steps
2. `set -euo pipefail` — what each flag actually does
3. Why multi-stage Docker builds reduce image vulnerabilities
4. Default bridge vs user-defined bridge network — the DNS difference, and how you'd manually pass an IP without a custom network
5. Full chain: Deployment → ReplicaSet → Pod, and why browsers can't reach ClusterIP directly
6. SAST vs SCA — what each actually scans
7. ArgoCD `selfHeal` + `prune` — what happens on manual cluster drift
8. Three separate IAM roles in EKS (cluster, node, IRSA) and who assumes each
9. Full IRSA chain: OIDC → trust policy → ServiceAccount annotation → AssumeRoleWithWebIdentity
10. `histogram_quantile` query structure for p99 latency, explained piece by piece
11. What Tempo/tracing solves that metrics and logs alone can't
12. Full Vault + ESO chain: K8s auth → Vault role → ClusterSecretStore → ExternalSecret → native K8s Secret
13. StatefulSet vs Deployment — specifically about storage (volumeClaimTemplates)
14. Why DaemonSet instead of a fixed-replica Deployment for per-node agents
15. HPA vs VPA — what each actually scales
16. The one invalid RBAC combination (Role + ClusterRoleBinding) and why
17. Default-deny NetworkPolicy pattern — what happens with zero policies applied
18. Sidecar pattern — how Envoy intercepts traffic without app code changes
19. DestinationRule vs VirtualService responsibilities in Istio
20. Chart `version` vs `appVersion` in Helm
21. `terraform state mv` vs `state rm` vs `import` — three different intents
22. Why separate state files per environment beat workspaces for blast-radius isolation
23. Rolling vs blue-green vs canary — the actual trade-off in each
24. What Cosign image signing protects against that scanning alone doesn't
25. Velero vs etcd backup — which layer of disaster recovery each one handles
26. The three-layer cost/scale strategy: HPA + Cluster Autoscaler + Spot instances

---

## Quick Self-Test (do this without looking)
1. Your Spot instance node just got a 2-minute interruption warning — what kind of workload should have been running on it to make this a non-issue?
2. If your entire cluster's control plane is lost (etcd gone) on a self-managed cluster, does a Velero backup alone save you? Why or why not?
3. Explain the three-layer relationship between HPA, Cluster Autoscaler, and Spot instances in one sentence each.
4. Pick any 3 questions from the Full Mixed Recall list above (not the same 3 as last time) and answer them out loud right now.
