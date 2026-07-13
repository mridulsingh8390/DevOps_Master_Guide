# Argo CD Complete Guide (Beginner to Advanced)
## GitOps Continuous Delivery for Kubernetes

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

1. What is Argo CD and why it matters  
2. Core GitOps concepts  
3. Install Argo CD  
4. Initial login and CLI setup  
5. Registering Git repositories  
6. Creating applications (manual + declarative)  
7. Sync policies (manual vs auto-sync)  
8. Helm/Kustomize support in Argo CD  
9. App of Apps pattern  
10. Projects, RBAC, and multi-team control  
11. Webhooks and refresh behavior  
12. Health checks and customizations  
13. Drift detection and self-heal  
14. Backup and restore basics  
15. Troubleshooting matrix  
16. Practice lab tasks  
17. Daily Argo CD cheat sheet

---

## 1) What is Argo CD and Why it Matters

## What
Argo CD is a Kubernetes-native GitOps CD tool that keeps cluster state aligned with Git state.

## Why
- Git becomes source of truth
- Drift detection + optional self-healing
- Safer, auditable deployments
- Works naturally with Helm and Kustomize

---

## 2) Core GitOps Concepts

- **Desired state**: manifests/charts in Git
- **Live state**: resources currently running in cluster
- **Sync**: reconcile live state to desired state
- **OutOfSync**: drift detected
- **Auto-sync**: automatic reconciliation
- **Self-heal**: fix manual cluster changes automatically
- **Prune**: remove resources deleted from Git

---

## 3) Install Argo CD

## 3.1 Create namespace and install manifests
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 3.2 Verify pods
```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Wait until all key pods are Running:
- argocd-server
- argocd-repo-server
- argocd-application-controller
- argocd-dex-server (if enabled)
- redis

---

## 4) Initial Login and CLI Setup

## 4.1 Expose Argo CD server (lab method)
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

UI:
- https://localhost:8080

## 4.2 Get initial admin password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

## 4.3 Install Argo CD CLI
```bash
# Linux amd64 example
curl -sSL -o argocd \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
argocd version --client
```

## 4.4 CLI login
```bash
argocd login localhost:8080 --username admin --password <password> --insecure
argocd account update-password
```

---

## 5) Registering Git Repositories

## Add repository via CLI
```bash
argocd repo add https://github.com/<org>/<repo>.git
```

For private repo (HTTPS):
```bash
argocd repo add https://github.com/<org>/<repo>.git \
  --username <git-user> --password <token>
```

For SSH:
```bash
argocd repo add git@github.com:<org>/<repo>.git --ssh-private-key-path ~/.ssh/id_ed25519
```

List repos:
```bash
argocd repo list
```

---

## 6) Creating Applications

## 6.1 Imperative app creation (CLI)
```bash
argocd app create demo-app \
  --repo https://github.com/<org>/<repo>.git \
  --path k8s/demo-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev
```

## 6.2 Sync app
```bash
argocd app sync demo-app
argocd app get demo-app
```

---

## 6.3 Declarative app (`application.yaml`)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<repo>.git
    targetRevision: main
    path: k8s/demo-app
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply:
```bash
kubectl apply -f application.yaml
kubectl get applications -n argocd
```

---

## 7) Sync Policies (Manual vs Auto)

## Manual sync
- Argo CD detects drift but waits for operator action.

## Automated sync
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### Meaning
- `prune: true` -> removes resources deleted in Git
- `selfHeal: true` -> restores drifted resources

---

## 8) Helm and Kustomize Support

Argo CD supports:
- Plain YAML
- Helm charts
- Kustomize overlays

## 8.1 Helm source example
```yaml
source:
  repoURL: https://github.com/<org>/<repo>.git
  targetRevision: main
  path: charts/myapp
  helm:
    valueFiles:
      - values.yaml
      - values-prod.yaml
```

## 8.2 Kustomize source example
```yaml
source:
  repoURL: https://github.com/<org>/<repo>.git
  targetRevision: main
  path: kustomize/overlays/prod
```

---

## 9) App of Apps Pattern

## What
Parent Argo CD Application manages child Application manifests.

## Why
- Bootstrap many apps/clusters from one root app
- Standardized team/platform rollout

Parent example points to folder containing child `Application` YAMLs.

---

## 10) Projects, RBAC, Multi-Team Control

## 10.1 AppProject example
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - 'https://github.com/<org>/*'
  destinations:
    - namespace: team-a-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

## Why AppProject?
- Restrict allowed repos, destinations, and resource scope
- Team isolation

---

## 11) Webhooks and Refresh Behavior

Without webhook:
- Argo CD polls repository periodically.

With webhook:
- Instant refresh on push event.

For GitHub webhook:
- Point webhook to Argo CD API endpoint (as per setup)
- Configure secret if used

Force refresh:
```bash
argocd app get demo-app --refresh
```

---

## 12) Health Checks and Customizations

Argo CD tracks:
- Sync status (`Synced/OutOfSync`)
- Health status (`Healthy/Degraded/Progressing`)

Useful commands:
```bash
argocd app get demo-app
argocd app history demo-app
argocd app resources demo-app
```

---

## 13) Drift Detection and Self-Heal

Drift example:
- Someone edits Deployment in cluster manually.
- Argo CD marks app OutOfSync.
- If auto-sync + self-heal enabled, Argo CD reverts to Git desired state.

Diff check:
```bash
argocd app diff demo-app
```

---

## 14) Backup and Restore Basics

Important to back up:
- Argo CD namespace resources (Applications, Projects, Secrets, ConfigMaps)
- Repo credentials/secrets
- SSO/RBAC config

Simple backup idea:
```bash
kubectl get all,cm,secret,applications,appprojects -n argocd -o yaml > argocd-backup.yaml
```

Restore:
```bash
kubectl apply -f argocd-backup.yaml
```

(For production, use stronger backup strategy and secret management.)

---

## 15) Troubleshooting Matrix

## 15.1 Argo CD pods not healthy
```bash
kubectl get pods -n argocd
kubectl describe pod <pod> -n argocd
kubectl logs <pod> -n argocd
```

## 15.2 Repo connection failure
```bash
argocd repo list
argocd repo get https://github.com/<org>/<repo>.git
```
Check credentials/SSH key/network.

## 15.3 App stuck OutOfSync
```bash
argocd app diff <app>
argocd app get <app>
```
Check:
- wrong path
- wrong branch/tag
- missing CRDs/resources
- immutable field changes

## 15.4 Permission denied in destination namespace
- Check ServiceAccount/RBAC used by Argo CD controller
- Check AppProject destination restrictions

---

## 16) Practice Lab Tasks

1. Install Argo CD in `argocd` namespace  
2. Login via UI and CLI  
3. Add Git repo (public, then private)  
4. Create Application and sync manually  
5. Enable auto-sync + prune + self-heal  
6. Test drift correction by editing live resource  
7. Create AppProject to restrict scope  
8. Implement small App-of-Apps bootstrap

---

## 17) Daily Argo CD Cheat Sheet

```bash
# Login
argocd login localhost:8080 --username admin --password <password> --insecure

# Repo
argocd repo list
argocd repo add https://github.com/<org>/<repo>.git

# Apps
argocd app list
argocd app get <app>
argocd app sync <app>
argocd app diff <app>
argocd app history <app>
argocd app delete <app>
```

---

## Final Notes

- Keep all deployable state in Git (single source of truth).
- Start manual sync first; enable auto-sync when stable.
- Use AppProjects and RBAC early for team boundaries.
- Pair Argo CD with Helm/Kustomize for scalable GitOps workflows.