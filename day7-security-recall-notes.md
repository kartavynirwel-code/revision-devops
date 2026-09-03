# Day 7 — Security (Vault + ESO) + Mixed Recall

## 1. Vault + External Secrets Operator (ESO) — Full Flow

### Why This Exists
- Kubernetes Secrets are only **base64-encoded**, not encrypted at rest by default (unless you enable encryption at rest on etcd) — not a real secrets management solution on its own.
- Vault is a dedicated secrets engine — centralized storage, dynamic secrets, encryption, access policies, audit logging.
- **ESO (External Secrets Operator)** bridges the two: it reads secrets from Vault (or AWS Secrets Manager, etc.) and syncs them into native Kubernetes `Secret` objects that Pods can consume normally.

### The Chain
```
1. Vault runs (dev mode for learning: `vault server -dev`)
   → In dev mode, Vault auto-unseals and gives you a root token — 
     NEVER use dev mode in production (no persistence, no real unsealing).

2. Enable Kubernetes Auth Method in Vault
   vault auth enable kubernetes
   vault write auth/kubernetes/config \
     kubernetes_host="https://<k8s-api-server>"
   → This lets Vault verify Kubernetes ServiceAccount tokens as a
     valid form of authentication (Vault talks to the K8s API to
     validate the token via TokenReview).

3. Create a Vault Policy (what secrets can be read)
   vault policy write my-policy - <<EOF
   path "secret/data/myapp/*" {
     capabilities = ["read"]
   }
   EOF

4. Create a Vault Role (binds K8s ServiceAccount to a Vault Policy)
   vault write auth/kubernetes/role/my-role \
     bound_service_account_names=my-sa \
     bound_service_account_namespaces=my-namespace \
     policies=my-policy \
     ttl=1h

5. Deploy ESO's ClusterSecretStore (points ESO at Vault)
   apiVersion: external-secrets.io/v1beta1
   kind: ClusterSecretStore
   metadata:
     name: vault-backend
   spec:
     provider:
       vault:
         server: "http://vault:8200"
         path: "secret"
         auth:
           kubernetes:
             mountPath: "kubernetes"
             role: "my-role"
             serviceAccountRef:
               name: my-sa

6. Create an ExternalSecret (what to pull, and where to put it)
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: my-app-secret
   spec:
     secretStoreRef:
       name: vault-backend
       kind: ClusterSecretStore
     target:
       name: my-app-k8s-secret   # the native K8s Secret it creates
     data:
       - secretKey: db-password
         remoteRef:
           key: myapp/config
           property: db-password

7. ESO controller syncs on an interval (refreshInterval) 
   → Reads from Vault → writes/updates the native K8s Secret
   → Pod mounts/references that native Secret as normal (env var or volume)
```

### One-Line Summary (interview-ready)
> "ESO uses Kubernetes ServiceAccount tokens to authenticate to Vault via the Kubernetes auth method, matched against a Vault role bound to that specific ServiceAccount and namespace — Vault returns the secret, and ESO writes it into a native Kubernetes Secret that pods consume normally, so the app never needs to know Vault exists."

### Why It Matters (security angle)
- App code stays unchanged — it just reads a normal K8s Secret/env var.
- Centralized rotation: update the secret in Vault, ESO's refresh interval propagates it — no manual `kubectl` secret updates across clusters.
- Fine-grained access: Vault policies + roles mean a ServiceAccount in `namespace-a` can't read secrets scoped to `namespace-b`.

## 2. Mixed Command Recall (say these out loud without looking)

**Pick one Docker, one kubectl, one Terraform command and explain what each does and when you'd use it:**
- Docker example to recall: `docker inspect -f '{{.NetworkSettings.IPAddress}}' <container>` — extracts a container's internal IP for debugging networking issues.
- kubectl example to recall: `kubectl logs --previous <pod>` — views logs from a crashed instance before the current restart, critical for debugging CrashLoopBackOff.
- Terraform example to recall: `terraform plan -out=tfplan` then `terraform apply tfplan` — separates the planning step from the apply step, so what gets applied is exactly what was reviewed (avoids drift between plan-time and apply-time state).

## 3. Full-Week Rapid Recall Checklist
Go through this list and rate yourself honestly (know it cold / shaky / blank):
- [ ] `git reflog` recovery flow
- [ ] `set -euo pipefail` — what each flag does
- [ ] Why multi-stage Docker builds reduce CVEs
- [ ] Default bridge network vs user-defined bridge (DNS difference)
- [ ] Full Deployment → ReplicaSet → Pod chain
- [ ] Why Ingress is needed over a plain ClusterIP Service
- [ ] SAST vs SCA — what each scans
- [ ] ArgoCD selfHeal + prune behavior
- [ ] Three separate IAM roles in an EKS setup (cluster/node/IRSA)
- [ ] Full IRSA chain: OIDC → trust policy → ServiceAccount → AssumeRoleWithWebIdentity
- [ ] `histogram_quantile` query structure for p99 latency
- [ ] What Tempo solves that metrics/logs alone can't
- [ ] Full Vault + ESO chain: K8s auth → Vault role → ClusterSecretStore → ExternalSecret → native Secret

---

## Quick Self-Test (do this without looking)
1. Why can't a Kubernetes Secret alone be considered a proper secrets management solution?
2. Walk through how a Pod's ServiceAccount ends up authenticating to Vault — what does Vault actually verify?
3. If you rotate a secret's value in Vault, how does the running Pod eventually get the updated value — does the app need to restart or change code?
4. What Vault configuration ties a specific Kubernetes ServiceAccount + namespace to a specific set of readable secret paths?
