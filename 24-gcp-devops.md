# GCP DevOps Complete Guide (gcloud + GKE + Artifact Registry + Cloud Build)
## For DevOps Engineers (Google Cloud practical operations)

> Command-focused, practical guide for daily DevOps workflows on GCP.

---

## 1) Why This Toolset Matters

For GCP DevOps, core tools/services are:
- `gcloud` CLI (operations + automation)
- IAM/service accounts (security model)
- GKE (Kubernetes)
- Artifact Registry (images/packages)
- Cloud Build (CI)
- Secret Manager + Cloud Logging/Monitoring

---

## 2) Install Google Cloud CLI

```bash
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-470.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-cli-470.0.0-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh
exec -l $SHELL
gcloud version
```

Initialize:
```bash
gcloud init
```

Login (if needed):
```bash
gcloud auth login
gcloud auth list
```

Set defaults:
```bash
gcloud config set project <PROJECT_ID>
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
gcloud config list
```

---

## 3) IAM and Service Accounts Basics

Create service account:
```bash
gcloud iam service-accounts create devops-ci \
  --display-name="DevOps CI Service Account"
```

Grant role (example):
```bash
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:devops-ci@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

Best practices:
- least privilege roles
- avoid broad Owner/Editor on automation identities
- prefer workload identity federation over long-lived keys

---

## 4) Compute Engine Basics

Create VM:
```bash
gcloud compute instances create vm-devops-01 \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud
```

List VMs:
```bash
gcloud compute instances list
```

SSH:
```bash
gcloud compute ssh vm-devops-01 --zone=us-central1-a
```

Start/stop:
```bash
gcloud compute instances stop vm-devops-01 --zone=us-central1-a
gcloud compute instances start vm-devops-01 --zone=us-central1-a
```

---

## 5) Cloud Storage Basics

Create bucket:
```bash
gcloud storage buckets create gs://my-devops-artifacts-12345 --location=US-CENTRAL1
```

Upload/download/list:
```bash
gcloud storage cp app.tar.gz gs://my-devops-artifacts-12345/
gcloud storage ls gs://my-devops-artifacts-12345
gcloud storage cp gs://my-devops-artifacts-12345/app.tar.gz .
```

Sync folder:
```bash
gcloud storage rsync ./dist gs://my-devops-artifacts-12345/dist --recursive
```

---

## 6) Artifact Registry (Container Images)

Create Docker repo:
```bash
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repo for apps"
```

Configure Docker auth:
```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Build/tag/push:
```bash
docker build -t us-central1-docker.pkg.dev/<PROJECT_ID>/my-docker-repo/myapp:1.0 .
docker push us-central1-docker.pkg.dev/<PROJECT_ID>/my-docker-repo/myapp:1.0
```

List images:
```bash
gcloud artifacts docker images list us-central1-docker.pkg.dev/<PROJECT_ID>/my-docker-repo
```

---

## 7) GKE (Google Kubernetes Engine) Basics

Create cluster:
```bash
gcloud container clusters create gke-devops \
  --zone us-central1-a \
  --num-nodes 2
```

Get credentials:
```bash
gcloud container clusters get-credentials gke-devops --zone us-central1-a
kubectl get nodes
```

Scale nodes:
```bash
gcloud container clusters resize gke-devops --zone us-central1-a --num-nodes 3
```

Delete cluster:
```bash
gcloud container clusters delete gke-devops --zone us-central1-a
```

---

## 8) Cloud Build (CI/CD Basics)

Create `cloudbuild.yaml`:

```yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args: ["build", "-t", "us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/myapp:$COMMIT_SHA", "."]
images:
  - "us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/myapp:$COMMIT_SHA"
```

Run build:
```bash
gcloud builds submit --config cloudbuild.yaml .
```

List builds:
```bash
gcloud builds list
```

---

## 9) Cloud Run (Fast container deployment option)

Deploy container:
```bash
gcloud run deploy myapp \
  --image us-central1-docker.pkg.dev/<PROJECT_ID>/my-docker-repo/myapp:1.0 \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated
```

List services:
```bash
gcloud run services list --region us-central1
```

---

## 10) Secret Manager

Create secret:
```bash
echo -n "StrongPass123!" | gcloud secrets create db-password --data-file=-
```

Access latest version:
```bash
gcloud secrets versions access latest --secret=db-password
```

Grant secret accessor role:
```bash
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:devops-ci@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

## 11) Logging and Monitoring Basics

Read logs:
```bash
gcloud logging read "resource.type=gce_instance" --limit 20 --format=json
```

List metrics:
```bash
gcloud monitoring metrics list --limit=20
```

Use Cloud Monitoring dashboards + alerting policies for production observability.

---

## 12) GKE + Helm + GitOps Pattern (Recommended)

Typical production flow:
1. Build image in Cloud Build  
2. Push image to Artifact Registry  
3. Update Helm values/tag in Git  
4. Argo CD/Flux syncs to GKE  

This separates build from deploy and keeps audit trail in Git.

---

## 13) Security Best Practices

1. Prefer Workload Identity over node service account broad roles  
2. Use private GKE clusters when possible  
3. Restrict public ingress with LB + WAF controls  
4. Use Binary Authorization / image scanning where required  
5. Store secrets in Secret Manager, not in manifests  
6. Enable audit logging and policy constraints (Org Policy)  

---

## 14) Troubleshooting

## Auth/account confusion
```bash
gcloud auth list
gcloud config list
gcloud config set project <PROJECT_ID>
```

## Permission denied
- missing IAM role
- wrong principal/project
- org policy restrictions

## GKE context issues
```bash
kubectl config get-contexts
kubectl config current-context
gcloud container clusters get-credentials gke-devops --zone us-central1-a
```

## Artifact push denied
- docker auth not configured for Artifact Registry domain
```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

---

## 15) Practice Tasks

1. Install gcloud and configure project/region/zone  
2. Create service account with limited Artifact Registry permissions  
3. Build and push Docker image to Artifact Registry  
4. Create GKE cluster and deploy sample app  
5. Create Cloud Build pipeline for image build  
6. Store app secret in Secret Manager and read it securely  
7. Configure basic log query and alerting baseline  

---

## 16) Daily Cheat Sheet

```bash
gcloud config list
gcloud projects list
gcloud compute instances list
gcloud container clusters list
gcloud artifacts repositories list --location=us-central1
gcloud builds list
gcloud run services list --region=us-central1
gcloud secrets list
```

---

## 17) Workload Identity Federation (Keyless CI Auth)

## Why
Let GitHub Actions/GitLab CI authenticate to GCP without downloading a long-lived service account JSON key.

## Create a workload identity pool + provider
```bash
gcloud iam workload-identity-pools create github-pool \
  --location=global --display-name="GitHub Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"
```

## Bind to a service account
```bash
gcloud iam service-accounts add-iam-policy-binding devops-ci@<PROJECT_ID>.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/github-pool/attribute.repository/<org>/<repo>"
```

CI then exchanges its OIDC token for short-lived GCP credentials — no static key ever stored in secrets.

---

## 18) Terraform on GCP (Common Pattern)

While `gcloud` is great for imperative ops, most production GCP infra is managed with Terraform:

```hcl
provider "google" {
  project = "<PROJECT_ID>"
  region  = "us-central1"
}

resource "google_storage_bucket" "artifacts" {
  name     = "my-devops-artifacts-12345"
  location = "US-CENTRAL1"
}
```

```bash
terraform init
terraform plan
terraform apply
```

Use `gcloud` for day-2 debugging/inspection even when Terraform owns the desired state.

---

## 19) Pub/Sub Basics (Event-Driven Pipelines)

```bash
gcloud pubsub topics create build-events
gcloud pubsub subscriptions create build-events-sub --topic=build-events

# Publish
gcloud pubsub topics publish build-events --message="build finished: myapp:1.0"

# Pull
gcloud pubsub subscriptions pull build-events-sub --auto-ack
```

Common DevOps use: trigger downstream automation (Slack notify, deploy step) on Cloud Build completion events.

---

## 20) Cloud Functions (Lightweight Serverless)

```bash
gcloud functions deploy notify-slack \
  --gen2 \
  --runtime=nodejs20 \
  --trigger-topic=build-events \
  --entry-point=main \
  --region=us-central1
```

List/logs:
```bash
gcloud functions list
gcloud functions logs read notify-slack --limit=20
```

Useful for small glue automation (webhook receivers, notification relays) without standing up a full service.

---

## Final Notes

- GCP DevOps becomes very efficient once you standardize around:
  - Cloud Build
  - Artifact Registry
  - GKE/Cloud Run
  - Secret Manager
- Prioritize IAM design early; it prevents most long-term security problems.
- Use Workload Identity Federation for CI auth instead of service account keys, and let Terraform own infra state while `gcloud` handles day-2 operations.
