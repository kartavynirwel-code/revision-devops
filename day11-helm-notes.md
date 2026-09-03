# Day 11 — Helm Revision Notes

## 1. Why Helm Exists
- Raw Kubernetes manifests get repetitive fast — deploying the same app to dev/staging/prod means copy-pasting YAML and manually changing image tags, replica counts, resource limits, etc.
- **Helm** is a package manager for Kubernetes: bundles manifests into a reusable, versioned, templated **Chart**, parameterized by a single `values.yaml`.
- **Interview point**: The core value is templating + release versioning — same chart, different values per environment, and Helm tracks each install as a **release** you can roll back.

## 2. Chart Structure
```
mychart/
├── Chart.yaml          # metadata: name, version, description
├── values.yaml         # default configuration values
├── charts/             # subcharts/dependencies (other charts this one depends on)
├── templates/          # actual K8s manifest templates (Go templating + YAML)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # reusable template snippets/functions
│   └── NOTES.txt       # printed to user after install/upgrade
└── .helmignore          # files to exclude when packaging
```

### `Chart.yaml` Example
```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my backend app
version: 0.1.0        # chart version (increments with chart changes)
appVersion: "1.2.0"   # version of the actual app being deployed
```
- **Interview point**: `version` (chart version) and `appVersion` (application version) are tracked separately — you can bump the chart's templating logic without changing the app version, or vice versa.

### `values.yaml` Example
```yaml
replicaCount: 2
image:
  repository: myapp
  tag: "1.2.0"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 8080
resources:
  requests:
    cpu: 250m
    memory: 256Mi
```

### Templating Syntax
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-mychart
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```
- `.Values.*` — pulls from `values.yaml` (or `--set`/`-f` overrides at install time).
- `.Release.Name` — the name given at `helm install <release-name> ...`.
- `.Chart.Name` — pulled from `Chart.yaml`.
- `{{- ... }}` / `nindent` — Go template whitespace control and YAML indentation helpers — common source of "why is my YAML broken" debugging.

## 3. Core Commands
```bash
helm install my-release ./mychart                    # install a chart as a new release
helm install my-release ./mychart -f custom-values.yaml   # override with custom values file
helm install my-release ./mychart --set replicaCount=3    # override a single value inline

helm upgrade my-release ./mychart                     # apply changes to an existing release
helm upgrade --install my-release ./mychart           # install if not exists, upgrade if it does (common in CI/CD)

helm rollback my-release 1                            # roll back to revision 1
helm history my-release                               # see all revisions of a release

helm list                                             # list releases in current namespace
helm uninstall my-release                             # remove a release

helm template ./mychart                               # render templates locally WITHOUT installing — great for debugging
helm lint ./mychart                                   # validate chart structure/syntax

helm dependency update                                # pull in subchart dependencies listed in Chart.yaml
```

### Interview Point — `helm template` vs `helm install --dry-run`
- `helm template`: Pure client-side rendering, no cluster contact at all — fastest way to see final YAML output for debugging templating logic.
- `helm install --dry-run`: Renders AND validates against the actual cluster (checks if resources are valid per the API server) without actually creating them — catches issues `helm template` alone might miss.

## 4. Why Helm Over Raw Manifests (summary for interview)
| | Raw manifests | Helm |
|---|---|---|
| Reuse across envs | Copy-paste + manual edits | One chart, different `values.yaml` per env |
| Versioning/rollback | Manual (`kubectl apply` has no built-in history) | `helm rollback` to any prior revision instantly |
| Dependency management | Manual ordering of `kubectl apply` | `charts/` subcharts declared in `Chart.yaml` |
| Templating/logic | None — static YAML only | Go templating: conditionals, loops, functions |
| Packaging/sharing | Share raw YAML files | Package as versioned `.tgz`, publish to a chart repo |

---

## Quick Self-Test (do this without looking)
1. What's the difference between a chart's `version` and `appVersion` fields?
2. How would you debug a Helm template that's producing broken YAML, without actually installing anything to the cluster?
3. Why is `helm upgrade --install` commonly used in CI/CD pipelines instead of separate `helm install`/`helm upgrade` calls?
4. What actual problem does `helm rollback` solve that plain `kubectl apply` doesn't handle out of the box?
