# Terraform Complete Guide (Beginner to Advanced)
## Infrastructure as Code for DevOps Engineers

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands/HCL
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Terraform and why it matters
2. Install Terraform
3. Core concepts
4. Providers and the first resource
5. Variables, outputs, and locals
6. State management
7. Remote state and locking
8. Data sources
9. Meta-arguments (count, for_each, depends_on, lifecycle)
10. Modules
11. Workspaces
12. Provisioners (and why to avoid them)
13. Import and state manipulation
14. Terraform Cloud/Enterprise concepts
15. Terragrunt (DRY multi-environment wrapper)
16. CI/CD integration
17. Security scanning (tfsec/Checkov, recap with Trivy)
18. Troubleshooting matrix
19. Best practices
20. Practice lab tasks
21. Daily cheat sheet

---

## 1) What is Terraform and Why it Matters

## What
Terraform is a declarative Infrastructure as Code (IaC) tool. You describe the desired end state of your infrastructure in HCL (HashiCorp Configuration Language), and Terraform figures out how to get there.

## Why
- Cloud-agnostic (AWS, Azure, GCP, and hundreds of other providers via one workflow)
- Declarative — describe the "what", not the "how"
- Plan before apply — see exactly what will change before it happens
- State tracking — Terraform knows what it created and can update/destroy it precisely
- Reusable modules — package infra patterns once, use everywhere

---

## 2) Install Terraform

## 2.1 Add HashiCorp repo and install (Ubuntu)
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y terraform
```

## 2.2 Verify
```bash
terraform version
```

## 2.3 Enable tab completion
```bash
terraform -install-autocomplete
```

---

## 3) Core Concepts

- **Configuration**: `.tf` files describing desired infrastructure
- **Provider**: plugin that talks to an API (AWS, Azure, GCP, Kubernetes, etc.)
- **Resource**: a single infrastructure object (a VM, a bucket, a DNS record)
- **State**: Terraform's record of what it has created (`terraform.tfstate`)
- **Plan**: a preview of changes Terraform will make
- **Apply**: execute the plan against real infrastructure
- **Module**: a reusable, packaged set of resources

## The core workflow
```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

---

## 4) Providers and the First Resource

## 4.1 Provider block (`main.tf`)
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.7.0"
}

provider "aws" {
  region = "us-east-1"
}
```

## 4.2 First resource
```hcl
resource "aws_s3_bucket" "artifacts" {
  bucket = "my-devops-artifacts-12345"
}
```

## 4.3 Initialize and apply
```bash
terraform init
terraform plan
terraform apply
```

Terraform downloads the provider plugin during `init`, computes a diff during `plan`, and creates/updates real resources during `apply`.

## 4.4 Destroy
```bash
terraform destroy
```

---

## 5) Variables, Outputs, and Locals

## 5.1 Input variables (`variables.tf`)
```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
  default     = "my-devops-artifacts-12345"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

Use in a resource:
```hcl
resource "aws_s3_bucket" "artifacts" {
  bucket = var.bucket_name
  tags = {
    Environment = var.environment
  }
}
```

## 5.2 Passing variable values
```bash
terraform apply -var="environment=prod"
terraform apply -var-file="prod.tfvars"
```

`prod.tfvars`:
```hcl
environment = "prod"
bucket_name = "prod-artifacts-98765"
```

## 5.3 Outputs (`outputs.tf`)
```hcl
output "bucket_arn" {
  value = aws_s3_bucket.artifacts.arn
}
```
```bash
terraform output
terraform output bucket_arn
```

## 5.4 Locals (computed/derived values)
```hcl
locals {
  full_name = "${var.environment}-${var.bucket_name}"
}
```

---

## 6) State Management

## Why state matters
Terraform state is the source of truth mapping your `.tf` config to real-world resource IDs. Without it, Terraform can't know what already exists.

## Inspect state
```bash
terraform state list
terraform state show aws_s3_bucket.artifacts
```

## Rename a resource in state (after refactoring code)
```bash
terraform state mv aws_s3_bucket.old_name aws_s3_bucket.new_name
```

## Remove a resource from state (without destroying it)
```bash
terraform state rm aws_s3_bucket.artifacts
```

> **Never hand-edit `terraform.tfstate`.** Use `terraform state` subcommands.

---

## 7) Remote State and Locking

## Why
Local state (`terraform.tfstate` on your laptop) doesn't work for teams — no shared source of truth, no locking against concurrent applies.

## S3 + DynamoDB backend (AWS example)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "envs/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

The DynamoDB table provides state locking — prevents two people from running `apply` at the same time and corrupting state.

## Azure backend example
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateacct12345"
    container_name        = "tfstate"
    key                   = "prod.terraform.tfstate"
  }
}
```

## GCS backend example
```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "envs/prod"
  }
}
```

## Migrate from local to remote state
```bash
terraform init -migrate-state
```

---

## 8) Data Sources

## Why
Reference existing infrastructure you didn't create with Terraform (or created in a different state file).

```hcl
data "aws_vpc" "default" {
  default = true
}

resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.default.id
  cidr_block = "10.0.1.0/24"
}
```

Data sources are read-only — Terraform queries the provider but never modifies the referenced object.

---

## 9) Meta-Arguments

## 9.1 `count` — create N identical resources
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags = {
    Name = "web-${count.index}"
  }
}
```

## 9.2 `for_each` — create resources from a map/set (preferred over `count` for named resources)
```hcl
variable "buckets" {
  type    = set(string)
  default = ["logs", "artifacts", "backups"]
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "myapp-${each.key}"
}
```

Why `for_each` over `count` here: if you remove an item from the middle of a `count`-based list, Terraform re-indexes and may destroy/recreate unrelated resources. `for_each` keys by name, so removing one item only affects that item.

## 9.3 `depends_on` — explicit ordering
```hcl
resource "aws_iam_role_policy" "policy" {
  # ...
}

resource "aws_instance" "app" {
  depends_on = [aws_iam_role_policy.policy]
  # ...
}
```
Usually Terraform infers dependencies automatically from references; use `depends_on` only when a dependency isn't visible in the config (e.g., IAM eventual consistency).

## 9.4 `lifecycle`
```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags]
  }
}
```
- `create_before_destroy`: avoid downtime on replacement (build new before killing old)
- `prevent_destroy`: safety guard against accidental `destroy` of critical resources
- `ignore_changes`: stop Terraform from fighting external changes to specific attributes

---

## 10) Modules

## Why
Package a reusable infrastructure pattern (e.g., "a standard VPC" or "a standard web app stack") once, parameterize it, and reuse across environments/teams.

## 10.1 Module structure
```
modules/s3-bucket/
├── main.tf
├── variables.tf
└── outputs.tf
```

`modules/s3-bucket/main.tf`:
```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}
```

`modules/s3-bucket/variables.tf`:
```hcl
variable "bucket_name" {
  type = string
}
```

## 10.2 Use the module
```hcl
module "artifacts_bucket" {
  source      = "./modules/s3-bucket"
  bucket_name = "my-devops-artifacts-12345"
}
```

## 10.3 Remote modules (Git/Registry)
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
}
```

```bash
terraform get -update
```

---

## 11) Workspaces

## Why
Manage multiple instances of the same configuration (e.g., dev/staging/prod) without duplicating code — each workspace gets its own state.

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace list
terraform workspace select dev
```

Reference in config:
```hcl
resource "aws_instance" "app" {
  tags = {
    Environment = terraform.workspace
  }
}
```

> Caution: workspaces share the same backend config and variable files by default — many teams prefer separate directories/Terragrunt for environments with meaningfully different configs (different instance sizes, different modules) rather than workspaces.

---

## 12) Provisioners (and Why to Avoid Them)

## What
Provisioners run scripts on a resource after creation — a last-resort escape hatch, not a primary tool.

```hcl
resource "aws_instance" "app" {
  # ...
  provisioner "remote-exec" {
    inline = ["sudo apt update", "sudo apt install -y nginx"]
  }
}
```

## Why avoid them
- Not idempotent/declarative — breaks Terraform's plan/apply model
- No drift detection on what the script did
- Prefer: bake configuration into the image (Packer), or use Ansible/cloud-init/user-data after provisioning, or use a dedicated K8s/Helm deployment step

Use provisioners only for truly one-off bootstrapping that nothing else can do.

---

## 13) Import and State Manipulation

## Why
Bring existing, manually-created infrastructure under Terraform management without recreating it.

## Import an existing resource (Terraform 1.5+ `import` block, or CLI)
```bash
terraform import aws_s3_bucket.artifacts my-existing-bucket-name
```

Or declaratively (`import.tf`, Terraform 1.5+):
```hcl
import {
  to = aws_s3_bucket.artifacts
  id = "my-existing-bucket-name"
}
```
```bash
terraform plan -generate-config-out=generated.tf
```
This generates matching HCL for the imported resource automatically — review and adjust before applying.

---

## 14) Terraform Cloud/Enterprise Concepts

## Why
Adds a managed backend with remote runs, policy enforcement, and team collaboration on top of open-source Terraform.

Key features:
- **Remote runs**: `plan`/`apply` executes in Terraform Cloud, not your laptop
- **Sentinel/OPA policy-as-code**: block applies that violate org policy (e.g., "no public S3 buckets")
- **VCS-driven workflow**: PR opens → plan runs automatically → comment posted with diff

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-infra"
    }
  }
}
```
```bash
terraform login
terraform init
```

---

## 15) Terragrunt (DRY Multi-Environment Wrapper)

## Why
Terraform alone leads to copy-pasted backend config and provider blocks across dev/stage/prod. Terragrunt keeps configuration DRY and orchestrates multiple modules together.

## Install
```bash
curl -Lo terragrunt https://github.com/gruntwork-io/terragrunt/releases/latest/download/terragrunt_linux_amd64
chmod +x terragrunt
sudo mv terragrunt /usr/local/bin/
```

## Root `terragrunt.hcl` (shared backend config)
```hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket         = "my-terraform-state-bucket"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

## Per-environment `terragrunt.hcl`
```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules//s3-bucket"
}

inputs = {
  bucket_name = "prod-artifacts-98765"
}
```

## Run
```bash
cd envs/prod
terragrunt plan
terragrunt apply
```

`terragrunt run-all apply` applies across all environments/modules in dependency order — useful for larger multi-module estates.

---

## 16) CI/CD Integration

## Typical pipeline stages
```bash
terraform fmt -check
terraform validate
terraform plan -out=tfplan
# manual approval gate
terraform apply tfplan
```

## Example (GitHub Actions)
```yaml
jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -out=tfplan
      - name: Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

Why plan as a saved artifact (`-out=tfplan`) matters: guarantees the exact plan reviewed is the exact plan applied — no drift between plan and apply steps.

---

## 17) Security Scanning (tfsec / Checkov)

Both the Makefile guide and Trivy guide in this series already cover `trivy config` for Terraform — here are the two dedicated Terraform-focused scanners:

## tfsec
```bash
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
tfsec .
```

## Checkov
```bash
pip install checkov --break-system-packages
checkov -d .
```

Both catch misconfigurations like public S3 buckets, unencrypted volumes, overly permissive security groups — run in CI before `apply`.

---

## 18) Troubleshooting Matrix

## 18.1 State lock stuck
```bash
terraform force-unlock <LOCK_ID>
```
Only use after confirming no other apply is actually running.

## 18.2 "Resource already exists" on apply
- The resource exists outside Terraform's state — use `terraform import` instead of recreating.

## 18.3 Drift (real infra changed outside Terraform)
```bash
terraform plan
```
Shows the diff between state and reality. Either `terraform apply` to reconcile back to code, or update code to match reality if the manual change was intentional.

## 18.4 Provider version conflicts
```bash
terraform init -upgrade
```

## 18.5 "Error acquiring the state lock"
- Check DynamoDB table / backend lock table for stale locks
- Verify no other CI job is mid-apply

---

## 19) Best Practices

1. Always run `terraform plan` before `apply`, and review it
2. Use remote state with locking for any team environment
3. Pin provider and module versions (`~>` constraints)
4. Use `for_each` over `count` for named/keyed resources
5. Keep environments in separate state files (workspaces or directory-per-env/Terragrunt)
6. Never commit `.tfstate` or `.tfvars` containing secrets to Git
7. Run `tflint`/`tfsec`/`checkov` in CI before every apply
8. Use modules for anything repeated more than once

---

## 20) Practice Lab Tasks

1. Install Terraform and provision a single S3 bucket/storage resource
2. Add variables, outputs, and a `.tfvars` file
3. Migrate state to a remote S3/Azure/GCS backend with locking
4. Refactor the bucket into a reusable module
5. Use `for_each` to create 3 buckets from a variable list
6. Import a manually-created resource into state
7. Add `terraform fmt`, `validate`, and `tfsec` to a CI pipeline
8. Set up two Terragrunt-managed environments (dev/prod) from one module

---

## 21) Daily Cheat Sheet

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy

terraform state list
terraform state show <resource>
terraform output

terraform workspace list
terraform workspace select <name>

tflint
tfsec .
checkov -d .
```

---

## Final Notes

- Terraform's power comes from state + plan/apply — respect the workflow, don't fight it with provisioners or manual edits.
- Remote state with locking is non-negotiable for any team beyond one person.
- Start with modules early; retrofitting them onto a large flat configuration is painful.
- Pair Terraform with tfsec/Checkov in CI, and consider Terragrunt once you have more than 2-3 environments to manage.
