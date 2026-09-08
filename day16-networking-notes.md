# Day 16 — Networking Deep Dive (CoreDNS + CNI) Revision Notes

## 1. CoreDNS — Kubernetes Service Discovery

### What It Does
- CoreDNS is the default DNS server running **as a Deployment inside the cluster** (in `kube-system`) that resolves Kubernetes Service/Pod names to their ClusterIPs.
- Every Pod's `/etc/resolv.conf` is automatically configured (by kubelet) to point to CoreDNS's ClusterIP as the nameserver.

### DNS Naming Convention
```
<service-name>.<namespace>.svc.cluster.local
```
- Same namespace: just `<service-name>` works (e.g., `backend` resolves fine from another Pod in the same namespace).
- Cross-namespace: need `<service-name>.<namespace>` (e.g., `backend.other-namespace`).
- Full FQDN: `<service-name>.<namespace>.svc.cluster.local` — always works, most explicit.
- **Pods** also get DNS records (if using a headless Service, ties back to Day 8 StatefulSets): `<pod-ip-dashed>.<service>.<namespace>.svc.cluster.local`, or for StatefulSets specifically: `<pod-name>.<service>.<namespace>.svc.cluster.local`.

### How CoreDNS Actually Resolves Service Names
- CoreDNS watches the Kubernetes API for Service and Endpoint objects (via the `kubernetes` plugin in its Corefile config) — it's not a static zone file, it dynamically reflects cluster state.
- When a new Service is created, CoreDNS almost immediately can resolve it — no manual DNS record management needed.

### Interview Point
> "CoreDNS is just a normal DNS server (based on the CoreDNS project, pluggable architecture) running as pods in the cluster, configured with a Kubernetes-aware plugin that watches the API server and dynamically serves records for Services and Pods — this is the layer that makes name-based service discovery (Day 2/3's `backend:8080`) actually work."

## 2. CNI (Container Network Interface) — How Pod Networking Gets Wired

### The Problem CNI Solves
- Kubernetes itself has **no built-in networking implementation** — it defines requirements (every Pod gets a unique IP, Pods can reach each other without NAT) but delegates the actual implementation to a **CNI plugin**.
- When kubelet creates a Pod, it calls the configured CNI plugin to set up that Pod's network namespace (assign IP, wire up virtual interfaces, program routing).

### Common CNI Plugins
- **Calico**: Popular for its **NetworkPolicy enforcement** (recall Day 9) and BGP-based routing option for large clusters; can also run in overlay mode (VXLAN).
- **Cilium**: eBPF-based — programs kernel-level packet filtering directly instead of relying on iptables, generally faster at scale, also provides deep observability (Hubble) and can enforce L7-aware NetworkPolicies.
- **Flannel**: Simpler, overlay-network-only (VXLAN), doesn't support NetworkPolicy on its own — often paired with Calico for policy enforcement (Canal = Flannel + Calico policy).

### Interview Point — Why CNI Choice Matters
> "Not every CNI plugin supports NetworkPolicy (Day 9) — if your cluster uses plain Flannel, applying a NetworkPolicy resource does nothing; it's silently ignored. This is a common production gotcha — always verify your CNI plugin actually enforces the policies you're writing."

## 3. DNS Troubleshooting Flow Inside a Pod

### Step-by-Step (say this out loud as a flow)
```bash
# 1. Get a shell into a pod with networking tools (or spin up a debug pod)
kubectl run debug --image=busybox:1.36 -it --rm -- sh

# 2. Check basic DNS resolution
nslookup backend.default.svc.cluster.local
# or
dig backend.default.svc.cluster.local

# 3. Check the pod's resolv.conf — confirm it's pointing at CoreDNS
cat /etc/resolv.conf
# nameserver should be CoreDNS's ClusterIP (commonly 10.96.0.10 or similar)

# 4. Check if CoreDNS pods themselves are healthy
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# 5. Check if the target Service actually has Endpoints
kubectl get endpoints backend
# empty Endpoints = Service selector doesn't match any Pod labels — 
# DNS will resolve the Service IP fine, but traffic has nowhere to go
```

### Common Root Causes (in likely order)
1. **Service selector doesn't match Pod labels** — DNS resolves fine (Service object exists), but `Endpoints` is empty, so connections fail/hang. This is the single most common "my service isn't working" bug.
2. **NetworkPolicy blocking traffic** (Day 9) — DNS works, connection attempt is silently dropped.
3. **CoreDNS pods themselves unhealthy/crashlooping** — DNS resolution fails entirely, cluster-wide.
4. **Wrong namespace referenced** — using short name (`backend`) across namespaces without the namespace suffix.

### Interview Point
> "The most common Kubernetes networking bug isn't actually a networking bug — it's a label selector mismatch between a Service and its target Pods. Always check `kubectl get endpoints` before assuming it's a CNI or DNS problem; empty endpoints means the Service has literally nothing to route to."

---

## Quick Self-Test (do this without looking)
1. What's the full DNS name format for a Service in a different namespace than the calling Pod?
2. What Kubernetes objects does CoreDNS actually watch to build its DNS records dynamically?
3. If a NetworkPolicy isn't having any effect at all in your cluster, what's the first thing you should check?
4. A Service exists, DNS resolves its ClusterIP fine, but connections to it hang/fail. What's the most likely single cause, and which command reveals it?
