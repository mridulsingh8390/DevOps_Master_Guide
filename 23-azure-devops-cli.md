# Azure CLI + Azure DevOps Complete Guide
## DevOps Operations with Azure (IAM, Compute, AKS, ACR, Pipelines)

> Practical, command-focused guide for DevOps engineers using Azure.
> Covers Azure CLI fundamentals + Azure DevOps pipelines basics.

---

## 1) Why This Toolset Matters

Azure DevOps engineers typically need:
- **Azure CLI (`az`)** for infrastructure/service operations
- **Azure DevOps** for repos, boards, pipelines, artifacts
- Strong handling of IAM (Entra ID/RBAC), AKS, ACR, Key Vault, monitoring

---

## 2) Install Azure CLI (Ubuntu)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az version
```

Login:
```bash
az login
```

If multiple subscriptions:
```bash
az account list -o table
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
az account show -o table
```

---

## 3) Core Azure CLI Basics

## Resource groups
```bash
az group create -n rg-devops-lab -l eastus
az group list -o table
```

## Common output formats
```bash
az vm list -o table
az vm list -o json
az vm list --query "[].{name:name, location:location}" -o table
```

---

## 4) IAM and RBAC Essentials

Assign role to user/service principal at scope:

```bash
az role assignment create \
  --assignee <principal-id-or-upn> \
  --role "Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-devops-lab
```

List role assignments:
```bash
az role assignment list --assignee <principal-id-or-upn> -o table
```

Best practice:
- least privilege
- scope as narrow as possible
- prefer managed identities/service principals for automation

---

## 5) VM and Compute Basics

Create VM:
```bash
az vm create \
  -g rg-devops-lab \
  -n vm-devops-01 \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys
```

Open port 80:
```bash
az vm open-port -g rg-devops-lab -n vm-devops-01 --port 80
```

Start/stop:
```bash
az vm stop -g rg-devops-lab -n vm-devops-01
az vm start -g rg-devops-lab -n vm-devops-01
```

---

## 6) Storage Account and Blob Basics

Create storage account:
```bash
az storage account create \
  -n devopsstorage12345 \
  -g rg-devops-lab \
  -l eastus \
  --sku Standard_LRS
```

Create container:
```bash
az storage container create \
  --name artifacts \
  --account-name devopsstorage12345 \
  --auth-mode login
```

Upload file:
```bash
az storage blob upload \
  --account-name devopsstorage12345 \
  --container-name artifacts \
  --name app.tar.gz \
  --file ./app.tar.gz \
  --auth-mode login
```

---

## 7) Azure Container Registry (ACR)

Create ACR:
```bash
az acr create -g rg-devops-lab -n myacrdevops123 --sku Basic
```

Login:
```bash
az acr login -n myacrdevops123
```

Build and push image using ACR build:
```bash
az acr build -r myacrdevops123 -t myapp:1.0 .
```

List repos/images:
```bash
az acr repository list -n myacrdevops123 -o table
az acr repository show-tags -n myacrdevops123 --repository myapp -o table
```

---

## 8) AKS (Azure Kubernetes Service) Basics

Create AKS cluster:
```bash
az aks create \
  -g rg-devops-lab \
  -n aks-devops \
  --node-count 2 \
  --enable-managed-identity \
  --generate-ssh-keys
```

Get kubeconfig:
```bash
az aks get-credentials -g rg-devops-lab -n aks-devops
kubectl get nodes
```

Scale node count:
```bash
az aks scale -g rg-devops-lab -n aks-devops --node-count 3
```

---

## 9) Key Vault for Secrets

Create vault:
```bash
az keyvault create -n kv-devops-12345 -g rg-devops-lab -l eastus
```

Set secret:
```bash
az keyvault secret set --vault-name kv-devops-12345 --name db-password --value "StrongPass123!"
```

Get secret:
```bash
az keyvault secret show --vault-name kv-devops-12345 --name db-password --query value -o tsv
```

---

## 10) Azure DevOps CLI Setup

Install extension:
```bash
az extension add --name azure-devops
```

Set defaults:
```bash
az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>
```

Login (PAT):
```bash
az devops login
```

---

## 11) Azure Repos and Pipelines (CLI + YAML)

List repos:
```bash
az repos list -o table
```

List pipelines:
```bash
az pipelines list -o table
```

Run pipeline:
```bash
az pipelines run --name "<pipeline-name>"
```

---

## 12) Azure Pipelines YAML (Starter)

`azure-pipelines.yml`
```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - checkout: self
    - script: echo "Installing dependencies"
    - script: echo "Run tests"
    - script: echo "Build complete"

- stage: SecurityScan
  jobs:
  - job: TrivyScan
    steps:
    - script: |
        echo "Run Trivy scan here"

- stage: DeployDev
  dependsOn: SecurityScan
  jobs:
  - job: Deploy
    steps:
    - script: echo "Deploy to dev"
```

---

## 13) Service Connections and Managed Identity Pattern

For deployments from pipelines:
- use service connections with least privilege
- prefer workload identity / managed identity where possible
- avoid hardcoded secrets in pipeline variables

---

## 14) Monitoring and Logs

## Azure Monitor basics
```bash
az monitor activity-log list --max-events 10 -o table
```

For AKS/container logs, integrate:
- Log Analytics workspace
- Container insights
- Managed Prometheus/Grafana (if adopted)

---

## 15) Security Best Practices

1. Enforce MFA and Conditional Access  
2. Use RBAC least privilege  
3. Use Key Vault for secrets  
4. Prefer managed identities over client secrets  
5. Restrict public network exposure  
6. Enable Defender for Cloud recommendations  
7. Audit activity logs and pipeline permissions  

---

## 16) Troubleshooting

## `az login` issues
```bash
az account show
az logout
az login
```

## Wrong subscription
```bash
az account list -o table
az account set --subscription "<id>"
```

## AKS credentials/context confusion
```bash
kubectl config get-contexts
kubectl config current-context
```

## Pipeline permission failures
- service connection lacks role
- missing repo/environment approvals
- secret variable permissions not granted

---

## 17) Practice Tasks

1. Install Azure CLI and set subscription  
2. Create resource group + VM  
3. Create ACR and push image  
4. Create AKS and deploy sample app  
5. Store/retrieve secret from Key Vault  
6. Create Azure Pipeline YAML with build/test/deploy stages  
7. Trigger pipeline using Azure DevOps CLI  

---

## 18) Daily Cheat Sheet

```bash
az account show -o table
az group list -o table
az vm list -o table
az acr list -o table
az aks list -o table
az keyvault list -o table
az pipelines list -o table
```

---

## Final Notes

- Azure CLI + Azure DevOps is a powerful enterprise DevOps combination.
- Build around managed identity, Key Vault, and least privilege from day one.
- Standardize pipeline templates to scale across teams.