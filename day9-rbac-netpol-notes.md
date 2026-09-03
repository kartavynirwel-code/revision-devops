# Day 9 — RBAC + Network Policies Revision Notes

## 1. RBAC (Role-Based Access Control)

### Core Objects
- **Role**: A set of permissions (verbs on resources) scoped to a **single namespace**.
- **ClusterRole**: Same idea, but scoped **cluster-wide** (or reusable across namespaces) — needed for cluster-scoped resources like Nodes, PersistentVolumes, or non-resource URLs.
- **RoleBinding**: Binds a Role (or a ClusterRole, used namespace-scoped) to a subject (User, Group, or ServiceAccount) — grants those permissions **within one namespace**.
- **ClusterRoleBinding**: Binds a ClusterRole to a subject **across the entire cluster**.

### Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: my-namespace
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-namespace
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### The Four Combinations (know these cold)
| Role type | Binding type | Effective scope |
|---|---|---|
| Role | RoleBinding | Single namespace only |
| ClusterRole | RoleBinding | Cluster-wide permissions, but granted only within that one namespace (common pattern — reuse a ClusterRole like `view` across many namespaces without redefining it) |
| ClusterRole | ClusterRoleBinding | Entire cluster, all namespaces |
| Role | ClusterRoleBinding | **Not valid** — Roles are namespace-scoped and cannot be bound cluster-wide |

- **Interview point**: The "ClusterRole + RoleBinding" combo is the one people forget — it's how you reuse a common permission set (like the built-in `view`, `edit`, `admin` ClusterRoles) without writing a new Role per namespace, while still keeping the grant scoped to just that namespace.

### ServiceAccount Permissions Model (ties to IRSA week)
- Every Pod runs as a ServiceAccount (defaults to `default` SA in its namespace if not specified).
- RBAC controls what that ServiceAccount can do **against the Kubernetes API** (create pods, read secrets, etc.) — this is separate from IRSA, which controls what that ServiceAccount can do against **AWS APIs**. Two different permission systems, often confused because both attach to the same ServiceAccount object.
- Best practice: never let workloads run as `default` SA with broad permissions — create dedicated, minimally-scoped ServiceAccounts per app.

## 2. NetworkPolicy

### Default Behavior (important gotcha)
- By default, **all Pods can talk to all other Pods** in a cluster, across all namespaces — Kubernetes networking is flat and open unless you explicitly restrict it.
- A `NetworkPolicy` only has effect on Pods it **selects** — if no NetworkPolicy selects a Pod, that Pod remains fully open.

### Default-Deny Pattern (the standard starting point)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-namespace
spec:
  podSelector: {}        # selects ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```
- This blocks **all** ingress and egress traffic to/from every Pod in the namespace. Then you add specific `NetworkPolicy` resources to **allow** exactly what's needed — a whitelist model, much safer than trying to blacklist bad traffic.

### Allowing Specific Traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```
- **Reading this**: "Pods labeled `app: backend` will only accept **ingress** traffic from Pods labeled `app: frontend`, on port 8080. Everything else to `backend` is blocked (because the default-deny is still in effect)."

### Key Fields
- `podSelector`: which Pods this policy applies to (empty `{}` = all Pods in namespace).
- `policyTypes`: `Ingress`, `Egress`, or both.
- `ingress[].from` / `egress[].to`: can match by `podSelector`, `namespaceSelector`, or `ipBlock` (CIDR ranges, e.g. for allowing traffic to/from outside the cluster).
- **Interview point**: NetworkPolicy requires a **CNI plugin that supports it** (Calico, Cilium) — the default `kubenet` or some basic CNIs silently ignore NetworkPolicy resources, giving a false sense of security if you don't verify your CNI supports it.

---

## Quick Self-Test (do this without looking)
1. Which combination of Role/ClusterRole + Binding type is invalid, and why?
2. Why is "ClusterRole + RoleBinding" a useful pattern instead of always using "Role + RoleBinding"?
3. If you apply zero NetworkPolicies in a namespace, what's the default traffic behavior between Pods?
4. In a default-deny setup, what two things determine whether traffic from Pod A to Pod B is actually allowed?
