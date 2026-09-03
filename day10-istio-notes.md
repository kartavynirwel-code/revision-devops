# Day 10 — Service Mesh (Istio) Revision Notes

## 1. Why a Service Mesh Exists Over Plain Kubernetes Networking

- Plain K8s gives you Services + kube-proxy for basic routing, but nothing built-in for:
  - Traffic splitting/canary routing between versions of a service
  - Automatic mTLS encryption between all services
  - Fine-grained observability (per-request tracing, retries, timeouts) without changing app code
  - Circuit breaking, fault injection for testing resilience
- A **service mesh** adds this infrastructure layer transparently, without touching application code — it operates at the network layer via proxies.

## 2. Sidecar Proxy Pattern (Envoy)

### How It Works
- Istio injects an **Envoy proxy container** into every Pod alongside your app container — this is the "sidecar."
- **All traffic in and out of the Pod** is transparently intercepted by the sidecar (via `iptables` rules set up by an init container) — your app doesn't know the proxy exists; it just makes normal HTTP/gRPC calls.
- Sidecars talk to each other directly (data plane), while `istiod` (control plane) configures all the sidecars with routing rules, certificates, etc.

### Data Plane vs Control Plane
- **Data plane**: The Envoy sidecars — actually handle every request, apply routing rules, collect telemetry, enforce mTLS.
- **Control plane** (`istiod`): Watches Kubernetes objects (VirtualServices, DestinationRules, etc.), compiles them into Envoy configuration, and pushes that config to all sidecars. Also acts as the Certificate Authority for mTLS.
- **Interview point**: This separation means you configure routing/security declaratively via YAML (control plane), and the actual traffic enforcement happens locally at each Pod (data plane) — no single point of failure for traffic handling itself.

## 3. VirtualService + DestinationRule

### DestinationRule — Defines Subsets
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-destination
spec:
  host: backend.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```
- Defines **named subsets** of a Service based on Pod labels — think of it as "which Pods count as v1 vs v2."

### VirtualService — Defines Routing Rules
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-routing
spec:
  hosts:
  - backend.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: backend.default.svc.cluster.local
        subset: v1
      weight: 90
    - destination:
        host: backend.default.svc.cluster.local
        subset: v2
      weight: 10
```
- Routes traffic based on weights (canary), headers, URI paths, etc. — here 90% of traffic goes to v1, 10% to v2 (classic canary release pattern).
- **Interview point**: DestinationRule defines "who the groups are," VirtualService defines "how traffic gets split between those groups." They work together — VirtualService references the subset names defined in DestinationRule.

## 4. mTLS Between Services

- Istio can automatically encrypt **all** service-to-service traffic with mutual TLS, without any application code change.
- **PeerAuthentication** resource controls the mTLS mode:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT   # or PERMISSIVE (accepts both mTLS and plaintext, useful during migration)
```
- **STRICT**: Only accepts mTLS-encrypted traffic — rejects plaintext.
- **PERMISSIVE**: Accepts both — useful when migrating an existing cluster into the mesh gradually (not all Pods have sidecars injected yet).
- `istiod` issues and rotates the certificates automatically — no manual cert management per service.

## 5. Observability Istio Adds

- Since every request passes through Envoy, Istio gets **free telemetry** without app instrumentation:
  - Golden signals per service: request rate, error rate, latency (P50/P90/P99) — visible in tools like Kiali or Grafana dashboards Istio ships with.
  - Automatic distributed tracing spans (integrates with Tempo/Jaeger) — since every hop goes through a proxy, trace context propagation is largely automatic.
  - **Kiali**: Istio's dedicated visualization tool — shows the live service graph, traffic flow, and health between services.
- **Interview point**: This is a major reason meshes are adopted — you get consistent observability across services written in different languages (Java, Python, Go) without each team instrumenting their own app differently.

---

## Quick Self-Test (do this without looking)
1. Explain the sidecar pattern — how does an app's outbound request actually get intercepted by Envoy without app code changes?
2. What's the difference in responsibility between DestinationRule and VirtualService?
3. You're migrating an existing cluster into an Istio mesh gradually — which PeerAuthentication mode do you use and why?
4. Why does a service mesh give you consistent observability across services written in different languages, when a stateless API written in Python vs Java wouldn't naturally share instrumentation?
