# Helm Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Kubernetes package management)

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Helm and why it matters  
2. Install Helm  
3. Core Helm concepts  
4. Repositories and chart discovery  
5. Install/upgrade/rollback/uninstall lifecycle  
6. Values files and overrides  
7. Create your own chart  
8. Templating basics  
9. Packaging and sharing charts  
10. Helm in CI/CD  
11. Troubleshooting  
12. Practice lab tasks  
13. Daily cheat sheet

---

## 1) What is Helm and Why it Matters

## What
Helm is a package manager for Kubernetes. It installs applications using reusable **charts**.

## Why
- Standardized deployment pattern
- Easy config overrides per environment
- Versioned releases
- Quick rollback to previous release

---

## 2) Install Helm

## Linux install script
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Or via package manager (if available in your distro repo):
```bash
sudo apt update
sudo apt install -y helm
```

Verify:
```bash
helm version
```

---

## 3) Core Helm Concepts

- **Chart**: Helm package (templates + defaults)
- **Release**: installed instance of a chart in cluster
- **values.yaml**: default config values
- **Template**: Kubernetes manifests with Go templating
- **Namespace**: target scope for install

---

## 4) Repositories and Chart Discovery

## Add repositories
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Search charts
```bash
helm search repo nginx
helm search repo prometheus
```

## Show chart info/default values
```bash
helm show chart bitnami/nginx
helm show values bitnami/nginx
```

---

## 5) Install/Upgrade/Rollback/Uninstall Lifecycle

## 5.1 Install chart
```bash
helm install my-nginx bitnami/nginx -n dev --create-namespace
```

## 5.2 List releases
```bash
helm list -n dev
helm list -A
```

## 5.3 Check release status
```bash
helm status my-nginx -n dev
```

## 5.4 Upgrade release
```bash
helm upgrade my-nginx bitnami/nginx -n dev
```

## 5.5 Rollback
```bash
helm history my-nginx -n dev
helm rollback my-nginx 1 -n dev
```

## 5.6 Uninstall
```bash
helm uninstall my-nginx -n dev
```

---

## 6) Values Files and Overrides

## 6.1 Custom values file (`values-dev.yaml`)
```yaml
service:
  type: NodePort
replicaCount: 2
```

Install with values:
```bash
helm install my-nginx bitnami/nginx -n dev -f values-dev.yaml
```

## 6.2 Override inline with `--set`
```bash
helm upgrade my-nginx bitnami/nginx -n dev --set replicaCount=3
```

## 6.3 Use multiple values files
```bash
helm upgrade my-nginx bitnami/nginx -n dev -f values.yaml -f values-dev.yaml
```

Rule:
- Later file overrides earlier file.

---

## 7) Create Your Own Chart

## Create chart skeleton
```bash
helm create hello-chart
tree hello-chart
```

Common structure:
- `Chart.yaml` (metadata)
- `values.yaml` (default values)
- `templates/` (manifest templates)
- `_helpers.tpl` (template helper funcs)

## Lint chart
```bash
helm lint hello-chart
```

## Render templates locally (no cluster apply)
```bash
helm template hello-release hello-chart
```

## Install local chart
```bash
helm install hello-release ./hello-chart -n dev --create-namespace
```

---

## 8) Templating Basics

Example snippet in `templates/deployment.yaml`:
```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Useful objects:
- `.Values` -> values.yaml data
- `.Release.Name` -> release name
- `.Chart.Name` -> chart metadata

Conditional example:
```yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "hello-chart.serviceAccountName" . }}
{{- end }}
```

Loop example:
```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

---

## 9) Packaging and Sharing Charts

## Package chart
```bash
helm package hello-chart
```

Creates:
- `hello-chart-0.1.0.tgz`

## Install from packaged chart
```bash
helm install hello-release ./hello-chart-0.1.0.tgz -n dev
```

## Dependency management
In `Chart.yaml`:
```yaml
dependencies:
  - name: redis
    version: "19.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

Update deps:
```bash
helm dependency update hello-chart
```

---

## 10) Helm in CI/CD

Typical steps:
1. `helm lint`
2. `helm template` (validate rendering)
3. (optional) `helm unittest` plugin tests
4. `helm upgrade --install ...`

Safe deployment pattern:
```bash
helm upgrade --install app ./chart -n prod -f values-prod.yaml --wait --timeout 5m
```

Why `--wait`?
Pipeline waits for resources to become ready.

---

## 11) Troubleshooting

## Check release and resources
```bash
helm status <release> -n <ns>
helm get values <release> -n <ns>
helm get manifest <release> -n <ns>
kubectl get all -n <ns>
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
```

## Common issues

### Issue: template rendering error
Fix:
```bash
helm lint <chart>
helm template test <chart> -f values.yaml
```

### Issue: upgrade failed
- Check wrong values key/type
- Check immutable field change (requires recreate)

### Issue: release stuck in pending
- Check hooks/jobs failures
- Check namespace permissions/RBAC

---

## 12) Practice Lab Tasks

1. Install nginx chart from Bitnami  
2. Override replica count and service type  
3. Upgrade release and verify rollout  
4. Roll back to previous revision  
5. Create custom chart with a simple Deployment + Service  
6. Package and reinstall your chart from `.tgz`  
7. Use `helm template` to debug before deploy  

---

## 13) Daily Helm Cheat Sheet

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx

helm install my-nginx bitnami/nginx -n dev --create-namespace
helm list -A
helm status my-nginx -n dev
helm upgrade my-nginx bitnami/nginx -n dev --set replicaCount=2
helm history my-nginx -n dev
helm rollback my-nginx 1 -n dev
helm uninstall my-nginx -n dev

helm lint ./mychart
helm template test ./mychart -f values.yaml
```

---

## Final Notes

- Use Helm as the default packaging layer for Kubernetes apps.
- Keep environment-specific values separate (`values-dev.yaml`, `values-prod.yaml`).
- Always run `helm lint` + `helm template` before applying in production.