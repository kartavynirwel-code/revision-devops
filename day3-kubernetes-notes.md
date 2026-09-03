# Day 3 — Kubernetes Core Revision Notes

## 1. Pod → Deployment → Service → Ingress Flow

### Pod
- Smallest deployable unit in Kubernetes. Wraps one or more containers that share network namespace (same IP, can talk via `localhost`) and storage volumes.
- You almost never create bare Pods directly in production — they have no self-healing (if a Pod dies, it's gone).

### Deployment
- Manages a **ReplicaSet**, which manages **Pods**. Gives you:
  - Declarative desired state (`replicas: 3`)
  - Self-healing (Pod dies → ReplicaSet creates a new one)
  - Rolling updates and rollbacks (`kubectl rollout undo`)
- **Chain**: `Deployment → ReplicaSet → Pods`

### Service
- Stable network identity (a virtual IP + DNS name) in front of a dynamic set of Pods, since Pod IPs change every time a Pod is recreated.
- Selects Pods via **label selectors** (matches `spec.selector` to Pod labels).
- Types:
  - **ClusterIP** (default) — internal-only, reachable only within the cluster
  - **NodePort** — exposes on a static port on every node's IP (30000-32767 range)
  - **LoadBalancer** — provisions an external cloud load balancer (AWS ELB/NLB, etc.), routes to NodePort internally
  - **ExternalName** — maps a Service to an external DNS name (no proxying)

### Ingress
- Sits **above** Services — routes external HTTP/HTTPS traffic based on hostname/path rules to different Services.
- Needs an **Ingress Controller** (e.g., NGINX Ingress Controller) actually running in the cluster to do anything — the Ingress resource itself is just a config object.
- **Why you need it over a plain Service**: A `ClusterIP` Service has no route from outside the cluster at all. Even `LoadBalancer` gives you one external IP per Service — if you have 10 microservices, that's 10 load balancers ($$$) and no path-based routing (`/api` → service A, `/app` → service B). Ingress gives you **one entry point** with host/path-based routing to many Services behind it.

**Full flow**: `Client → Ingress Controller → Ingress rules (host/path match) → Service (ClusterIP) → label selector → Pods`

## 2. Why Browser Fetch Needs Ingress, Not Just a Service (your own insight — nail this)
- A `ClusterIP` Service only has a virtual IP valid **inside** the cluster's network — kube-proxy programs iptables/IPVS rules that only cluster nodes/pods know how to route. A browser outside the cluster has no route to that IP at all.
- Even `NodePort`/`LoadBalancer` technically work without Ingress, but:
  - `NodePort` exposes a raw high port per node — not browser-friendly (no domain, no path routing, no TLS termination built-in).
  - `LoadBalancer` costs one cloud LB per Service and still has no path-based routing or centralized TLS management.
- **Ingress** solves this properly: one external IP/LB, TLS termination (cert-manager integration), host-based virtual hosting, and path-based routing — the same pattern a browser expects from a real web server (like Nginx reverse proxy, which is literally what the NGINX Ingress Controller is under the hood).

## 3. kubectl Debugging Commands
```bash
kubectl get pods                          # list pods
kubectl get pods -o wide                  # + node, IP info
kubectl describe pod <pod-name>           # events, conditions, resource limits, why it's Pending/CrashLooping

kubectl logs <pod-name>                   # logs from single-container pod
kubectl logs <pod-name> -c <container>    # specific container in multi-container pod
kubectl logs -f <pod-name>                # follow/stream logs
kubectl logs --previous <pod-name>        # logs from PREVIOUS crashed instance (crucial for CrashLoopBackOff)

kubectl exec -it <pod-name> -- sh         # shell into a pod
kubectl exec -it <pod-name> -c <container> -- bash

kubectl get events --sort-by='.lastTimestamp'   # cluster-wide events, chronological — great for "what just broke"

kubectl get pods --field-selector=status.phase=Pending   # filter by status
kubectl top pod                           # live CPU/memory usage (needs metrics-server)

kubectl rollout status deployment/<name>  # watch rollout progress
kubectl rollout undo deployment/<name>    # rollback to previous revision
kubectl rollout history deployment/<name> # see revision history
```

### Debugging Priority Order (interview-style answer)
1. `kubectl get pods` → is it Pending, CrashLoopBackOff, ImagePullBackOff, Running?
2. `kubectl describe pod` → check Events section at the bottom (scheduling failures, image pull errors, readiness/liveness probe failures, resource limit issues)
3. `kubectl logs` (and `--previous` if it's crash-looping) → actual application error
4. `kubectl exec -it` → shell in and check env vars, configs, network reachability (`curl`, `nslookup`) if the app itself doesn't explain it

---

## Quick Self-Test (do this without looking)
1. Explain the full object chain from Deployment down to a running container.
2. A Pod is stuck in `ImagePullBackOff` — which single command tells you why, and where in the output do you look?
3. Why can't a browser reach a `ClusterIP` Service directly, even if you know its virtual IP?
4. Your app was crash-looping and has just restarted — how do you see the logs from the crash itself, not the new instance?
