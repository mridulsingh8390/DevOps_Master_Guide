# Trivy Complete Guide (Image, Filesystem, IaC, K8s Scanning)
## For DevOps Engineers (Security in CI/CD)

---

## 1) What is Trivy?

Trivy is a fast vulnerability and misconfiguration scanner for:
- Container images
- Filesystem/projects
- IaC (Terraform, K8s YAML, Dockerfile)
- Kubernetes clusters

## Why use it?
- Catch vulnerabilities early (shift-left security)
- Easy CI/CD integration
- Supports SBOM and policy checks

---

## 2) Install Trivy (Ubuntu)

```bash
sudo apt update
sudo apt install -y wget gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy
trivy --version
```

---

## 3) Scan container images

```bash
trivy image nginx:1.27
trivy image --severity HIGH,CRITICAL nginx:1.27
trivy image --ignore-unfixed nginx:1.27
```

Save report:
```bash
trivy image -f table -o trivy-image-report.txt nginx:1.27
trivy image -f json -o trivy-image-report.json nginx:1.27
```

---

## 4) Scan local filesystem/project

```bash
trivy fs .
trivy fs --severity HIGH,CRITICAL .
trivy fs --scanners vuln,secret,misconfig .
```

Useful for:
- dependency CVEs
- leaked secrets
- insecure configurations

---

## 5) Scan IaC (Terraform/Kubernetes/Dockerfile)

```bash
trivy config .
trivy config ./k8s/
trivy config ./terraform/
```

Why:
Find risky settings like:
- privileged containers
- public S3/storage
- weak network rules

---

## 6) Scan Kubernetes cluster

```bash
trivy k8s --report summary cluster
trivy k8s --report all cluster
```

You need `kubectl` context configured with permissions.

---

## 7) SBOM generation

Generate SBOM from image:
```bash
trivy image --format cyclonedx --output sbom-cyclonedx.json nginx:1.27
trivy image --format spdx-json --output sbom-spdx.json nginx:1.27
```

---

## 8) CI/CD usage pattern

Fail pipeline on CRITICAL findings:
```bash
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

Allow only report generation:
```bash
trivy image --exit-code 0 --format json -o trivy.json myapp:latest
```

---

## 9) Ignore rules and baseline handling

Use `.trivyignore` (example):
```text
CVE-2023-12345
AVD-KSV-0001
```

Then:
```bash
trivy image --ignorefile .trivyignore myapp:latest
```

---

## 10) Troubleshooting

DB update issue:
```bash
trivy image --download-db-only
```

Slow scans:
- cache DB locally
- reduce scanners/scope

Permission issues in Docker:
```bash
docker pull image:tag
trivy image image:tag
```

---

## 11) Practice tasks

1. Scan nginx image and export JSON report  
2. Scan your repo with `trivy fs .`  
3. Scan Terraform/K8s manifests via `trivy config`  
4. Add `--exit-code 1` gate in Jenkins pipeline  
5. Generate SBOM and archive artifact

---

## 12) Daily cheat sheet

```bash
trivy image nginx:1.27
trivy image --severity HIGH,CRITICAL myapp:latest
trivy fs .
trivy config .
trivy k8s --report summary cluster
trivy image --format cyclonedx -o sbom.json myapp:latest
```