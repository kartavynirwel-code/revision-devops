# Day 13 — Deployment Strategies + Image Security Revision Notes

## 1. Deployment Strategies

### Rolling Update (Kubernetes default)
- Gradually replaces old Pods with new ones, a few at a time, controlled by `maxSurge`/`maxUnavailable`.
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # how many EXTRA pods can be created above desired count during rollout
      maxUnavailable: 0    # how many pods can be unavailable during rollout
```
- **Trade-off**: Zero extra infra cost (roughly), but both old and new versions run simultaneously during rollout — requires your app to handle version skew gracefully (e.g., API/DB compatibility between old and new code running at once).

### Blue-Green
- Two full environments exist: **Blue** (current live version) and **Green** (new version), fully deployed and tested in isolation.
- Traffic switches **all at once** from Blue to Green (e.g., by updating a Service selector or load balancer target) once Green is verified healthy.
- **Trade-off**: Instant rollback (just switch traffic back to Blue), no version-skew problem (only one version serves traffic at a time) — but costs **double the infrastructure** while both environments exist.

### Canary
- New version deployed alongside old, but only receives a **small percentage of traffic** initially (e.g., 5%), gradually increased if metrics look healthy.
- Implemented via Service mesh (Istio VirtualService weights — recall Day 10) or Ingress annotations, or simpler via just adjusting replica ratios between two Deployments.
- **Trade-off**: Lowest blast radius if something's wrong (only a small % of users affected), but requires solid metrics/monitoring to actually detect issues before rolling out further, and takes longer to fully roll out than blue-green.

### Interview Point — Choosing Between Them
> "Rolling update is the default and cheapest, but assumes version compatibility during transition. Blue-green gives instant, clean rollback at double infra cost. Canary minimizes blast radius but needs strong observability to be effective — you're relying on metrics to catch problems before wider rollout."

## 2. Image Signing (Cosign)

### The Problem It Solves
- Without signing, anyone with registry push access (or a compromised CI pipeline) could push a malicious image with the same tag, and your cluster would happily pull and run it — no way to verify the image is actually what your trusted CI pipeline built.

### How Cosign Works
```bash
# Generate a keypair
cosign generate-key-pair

# Sign an image after building it in CI
cosign sign --key cosign.key myregistry/myapp:1.2.0

# Verify before deploying
cosign verify --key cosign.pub myregistry/myapp:1.2.0
```
- Signature is stored **alongside the image** in the registry (as a separate OCI artifact), not embedded in the image itself.
- **Keyless signing** (newer, via Sigstore/Fulcio): Uses short-lived certificates tied to an OIDC identity (e.g., your GitHub Actions workflow identity) instead of managing long-lived private keys — reduces key management burden.
- **Enforcement**: An admission controller (see below) checks the signature exists and is valid **before** allowing the Pod to be scheduled — rejecting unsigned or tampered images at the cluster boundary.

## 3. SBOM (Software Bill of Materials)

- A structured, machine-readable inventory of every component, library, and dependency inside an image/application — like an ingredient list.
- Common formats: **SPDX**, **CycloneDX**.
- Generate with tools like `syft`:
```bash
syft myregistry/myapp:1.2.0 -o cyclonedx-json > sbom.json
```
- **Why it matters (interview angle)**: When a new CVE is disclosed (e.g., another Log4Shell-style incident), an SBOM lets you instantly query "do any of our running images contain this vulnerable library version?" instead of scanning everything from scratch. It's also increasingly a **compliance requirement** (US Executive Order 14028 for federal software supply chain).

## 4. Admission Controllers (OPA/Gatekeeper, Kyverno)

### What They Do
- Intercept requests to the Kubernetes API **before** an object is persisted (validating or mutating admission webhooks) — this is where you enforce policy that plain RBAC can't express (RBAC controls *who* can do *what*, admission control governs *what the object itself must look like*).

### Common Policies Enforced
- Reject Pods running as root (`runAsNonRoot: false` or missing `securityContext`)
- Require resource requests/limits to be set on every container
- Reject images not from an approved/trusted registry
- Require valid Cosign signatures (ties to image signing above)
- Enforce label/annotation standards (e.g., every resource must have a `team` label for cost tracking)

### OPA/Gatekeeper vs Kyverno
| | OPA/Gatekeeper | Kyverno |
|---|---|---|
| Policy language | Rego (a dedicated policy DSL — steeper learning curve) | Plain YAML (native Kubernetes-style, easier to read for K8s-only use cases) |
| Scope | General-purpose (used beyond K8s too) | Kubernetes-native only |
| Interview point | More powerful/flexible for complex logic | Faster to adopt if your team already thinks in K8s YAML |

### Example (Kyverno — reject unsigned images)
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-signature
    match:
      resources:
        kinds: ["Pod"]
    verifyImages:
    - imageReferences:
      - "myregistry/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              ...
              -----END PUBLIC KEY-----
```

---

## Quick Self-Test (do this without looking)
1. Why does a rolling update require your application to tolerate version skew, but blue-green doesn't?
2. What specific supply-chain attack does image signing (Cosign) protect against that vulnerability scanning alone doesn't?
3. If a new critical CVE drops in a widely-used library, how does having SBOMs already generated for your images save you time?
4. What's the conceptual difference between what RBAC enforces and what an admission controller like Kyverno enforces?
