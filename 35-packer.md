# HashiCorp Packer Complete Guide (Beginner to Advanced)
## Golden Image Building for DevOps Engineers

> This file is detailed and practical.
> Covers install, HCL2 templates, builders/provisioners, multi-cloud images, CI integration, troubleshooting.

---

## Table of Contents

1. What is Packer and why it matters
2. Install Packer
3. Core concepts
4. First image (AWS AMI example)
5. Provisioners (baking configuration into the image)
6. Variables and locals
7. Building for multiple platforms in parallel
8. Docker image builds
9. Post-processors
10. Validating and formatting templates
11. Integration with Terraform
12. CI/CD integration
13. Security scanning of built images
14. Troubleshooting matrix
15. Best practices
16. Practice lab tasks
17. Daily cheat sheet

---

## 1) What is Packer and Why it Matters

## What
Packer builds identical machine images (AMIs, Azure images, GCP images, Docker images, VM templates) from a single declarative template — the "golden image" pattern.

## Why DevOps teams use it
- Immutable infrastructure: bake config into the image once, deploy identical instances everywhere, stop configuring servers after boot
- Faster boot times than running Ansible/config-management at every instance launch
- One template, multiple cloud targets — build the same image for AWS + Azure + GCP simultaneously
- Pairs naturally with Terraform: Packer builds the image, Terraform provisions instances from it

---

## 2) Install Packer

## 2.1 Add HashiCorp repo and install (Ubuntu)
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y packer
packer version
```

---

## 3) Core Concepts

- **Template**: HCL2 (`.pkr.hcl`) file describing the image build
- **Builder**: the plugin/target platform (`amazon-ebs`, `azure-arm`, `googlecompute`, `docker`, `qemu`, `vsphere-iso`)
- **Provisioner**: how the image is configured (shell scripts, Ansible, Chef, file uploads)
- **Post-processor**: actions after build (compress, upload, tag, generate a manifest)
- **Source**: a named builder configuration, referenced by one or more `build` blocks

---

## 4) First Image (AWS AMI Example)

## 4.1 Template (`web-image.pkr.hcl`)
```hcl
packer {
  required_plugins {
    amazon = {
      source  = "github.com/hashicorp/amazon"
      version = "~> 1.3"
    }
  }
}

source "amazon-ebs" "web" {
  ami_name      = "web-golden-{{timestamp}}"
  instance_type = "t3.micro"
  region        = "us-east-1"
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/*ubuntu-jammy-22.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    owners      = ["099720109477"]
    most_recent = true
  }
  ssh_username = "ubuntu"
}

build {
  name    = "web-image"
  sources = ["source.amazon-ebs.web"]

  provisioner "shell" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
  }
}
```

## 4.2 Initialize (downloads required plugins)
```bash
packer init web-image.pkr.hcl
```

## 4.3 Validate and build
```bash
packer validate web-image.pkr.hcl
packer build web-image.pkr.hcl
```

The resulting AMI ID is printed at the end of the build — reference it in Terraform's `aws_instance` or an autoscaling group launch template.

---

## 5) Provisioners

## 5.1 Shell script file (instead of inline commands)
```hcl
provisioner "shell" {
  script = "scripts/install-app.sh"
}
```

## 5.2 Ansible provisioner
```hcl
provisioner "ansible" {
  playbook_file = "playbooks/web.yml"
}
```
This runs your existing Ansible playbooks (see the Ansible guides in this series) against the temporary build instance — a common way to reuse config-management code for image baking instead of duplicating logic in shell scripts.

## 5.3 File provisioner (upload local files into the image)
```hcl
provisioner "file" {
  source      = "configs/nginx.conf"
  destination = "/tmp/nginx.conf"
}

provisioner "shell" {
  inline = ["sudo mv /tmp/nginx.conf /etc/nginx/nginx.conf"]
}
```

## 5.4 Multiple provisioners run in order
```hcl
build {
  sources = ["source.amazon-ebs.web"]

  provisioner "shell" {
    inline = ["sudo apt update"]
  }
  provisioner "ansible" {
    playbook_file = "playbooks/web.yml"
  }
  provisioner "shell" {
    inline = ["sudo apt clean"]
  }
}
```

---

## 6) Variables and Locals

## 6.1 Input variables
```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "app_version" {
  type = string
}
```

Use in a source block:
```hcl
source "amazon-ebs" "web" {
  instance_type = var.instance_type
  ami_name      = "web-${var.app_version}-{{timestamp}}"
  # ...
}
```

## 6.2 Pass at build time
```bash
packer build -var="app_version=1.4.0" web-image.pkr.hcl
packer build -var-file="prod.pkrvars.hcl" web-image.pkr.hcl
```

## 6.3 Locals
```hcl
locals {
  timestamp = formatdate("YYYY-MM-DD", timestamp())
  full_name = "web-${var.app_version}-${local.timestamp}"
}
```

---

## 7) Building for Multiple Platforms in Parallel

## Why
One template, multiple cloud targets — build AWS + Azure + GCP images from the same provisioning logic in one run.

```hcl
source "amazon-ebs" "web" {
  # ... AWS config
}

source "azure-arm" "web" {
  # ... Azure config
}

build {
  sources = [
    "source.amazon-ebs.web",
    "source.azure-arm.web"
  ]

  provisioner "shell" {
    inline = ["sudo apt update", "sudo apt install -y nginx"]
  }
}
```

Packer builds all named sources in parallel by default, running the same provisioners against each.

---

## 8) Docker Image Builds

## Why
Packer can also build/tag Docker images using the same provisioner logic as VM images — useful for keeping VM and container images configured consistently from shared scripts.

```hcl
source "docker" "web" {
  image  = "ubuntu:22.04"
  commit = true
}

build {
  sources = ["source.docker.web"]

  provisioner "shell" {
    inline = ["apt update", "apt install -y nginx"]
  }

  post-processor "docker-tag" {
    repository = "myrepo/web-golden"
    tags       = ["latest", "1.0"]
  }
}
```

```bash
packer build docker-image.pkr.hcl
docker images | grep web-golden
```

---

## 9) Post-Processors

## Why
Run actions after a successful build — tag, compress, upload, or produce a machine-readable manifest for downstream automation.

## Manifest (record built artifact IDs for CI to consume)
```hcl
post-processor "manifest" {
  output = "manifest.json"
}
```

```bash
packer build web-image.pkr.hcl
cat manifest.json | jq '.builds[].artifact_id'
```

A CI pipeline can parse this manifest to automatically feed the new AMI/image ID into a Terraform variable for the next deploy stage.

## Compress (for local/on-prem VM images)
```hcl
post-processor "compress" {
  output = "web-image.tar.gz"
}
```

---

## 10) Validating and Formatting Templates

```bash
packer fmt .
packer validate web-image.pkr.hcl
```

Run both in CI before every build — `fmt` catches style drift, `validate` catches syntax/config errors without actually launching a build instance (saving time and cloud cost on broken templates).

---

## 11) Integration with Terraform

## Typical flow
1. Packer builds a golden AMI, outputs the AMI ID (via manifest post-processor)
2. Terraform's `aws_instance`/`aws_launch_template` references that AMI ID
3. Instances launch already-configured — no bootstrap-time Ansible/cloud-init needed for baseline setup

```hcl
data "aws_ami" "web_golden" {
  most_recent = true
  owners      = ["self"]
  filter {
    name   = "name"
    values = ["web-golden-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.web_golden.id
  instance_type = "t3.micro"
}
```

This `most_recent` data source pattern means Terraform automatically picks up the latest Packer-built image without hardcoding an AMI ID each time.

---

## 12) CI/CD Integration

## Example (GitHub Actions)
```yaml
jobs:
  build-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-packer@main
      - run: packer init web-image.pkr.hcl
      - run: packer fmt -check web-image.pkr.hcl
      - run: packer validate web-image.pkr.hcl
      - run: packer build web-image.pkr.hcl
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

Trigger on a schedule (weekly AMI rebuild for patching) or on changes to provisioning scripts.

---

## 13) Security Scanning of Built Images

Reuse the Trivy skills from earlier in this series against the built artifact:

```bash
# For Docker-built images
trivy image myrepo/web-golden:latest

# For filesystem-level scanning during the build (via a shell provisioner step)
provisioner "shell" {
  inline = [
    "curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh",
    "trivy fs --exit-code 1 --severity CRITICAL /"
  ]
}
```

Failing the Packer build on CRITICAL findings prevents a vulnerable golden image from ever being published.

---

## 14) Troubleshooting Matrix

## 14.1 Build hangs at SSH/WinRM connection
- Security group/firewall blocking the build instance's SSH port
- Wrong `ssh_username` for the base image (varies by AMI: `ubuntu`, `ec2-user`, `admin`)

## 14.2 "no plugins installed" error
```bash
packer init web-image.pkr.hcl
```
HCL2 templates require explicit plugin installation via `packer init` (unlike the older JSON template format).

## 14.3 Provisioner fails partway through
- Packer leaves the temporary build instance running by default on failure for debugging (unless `-on-error=abort`/cleanup is configured) — SSH in manually to diagnose, then terminate it

## 14.4 Image builds but instances launched from it fail to boot
- Provisioner left the instance in a bad state (e.g., killed networking, corrupted a required service) — review provisioner scripts for destructive operations

## 14.5 Parallel multi-source build fails on only one platform
```bash
packer build -only=amazon-ebs.web web-image.pkr.hcl
```
Isolate and re-run just the failing source to debug faster.

---

## 15) Best Practices

1. Pin the base image with `most_recent` + explicit filters, never a hardcoded "latest" that drifts
2. Run `packer fmt` + `packer validate` in CI before every build
3. Keep provisioning logic in versioned scripts/Ansible playbooks, not giant inline blocks
4. Use the manifest post-processor to hand off artifact IDs to Terraform automatically
5. Rebuild images on a schedule (weekly/monthly) to pick up OS security patches
6. Scan built images with Trivy before publishing/tagging as "latest"
7. Tag images with build metadata (git SHA, build date) for traceability

---

## 16) Practice Lab Tasks

1. Install Packer and build a minimal AWS AMI with nginx installed
2. Add a shell provisioner and a file provisioner to the same build
3. Parameterize the template with variables and a `.pkrvars.hcl` file
4. Add an Ansible provisioner using a playbook from an earlier guide
5. Build for two platforms (e.g., AWS + Docker) in one template
6. Add the manifest post-processor and parse the output AMI ID
7. Reference the built AMI from a Terraform `aws_instance` resource
8. Add a Trivy scan step that fails the build on CRITICAL vulnerabilities

---

## 17) Daily Cheat Sheet

```bash
packer init <template>.pkr.hcl
packer fmt .
packer validate <template>.pkr.hcl
packer build <template>.pkr.hcl
packer build -var="app_version=1.4.0" <template>.pkr.hcl
packer build -only=<source_name> <template>.pkr.hcl
```

---

## Final Notes

- Packer's core value is immutable infrastructure — bake once, deploy identical instances everywhere, avoid configuration drift.
- Reuse existing Ansible playbooks via the Ansible provisioner instead of duplicating logic in shell scripts.
- Pair with Terraform via the manifest post-processor + `most_recent` AMI data source for a fully automated build-then-deploy pipeline.
- Rebuild and rescan images regularly — a golden image is only as good as its last patch cycle.
