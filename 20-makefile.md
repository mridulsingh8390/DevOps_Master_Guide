# Makefile + pre-commit Complete Guide
## DevOps Automation and Local Quality Gates for Every Repository

> This file is practical and implementation-ready.
> Covers Makefile patterns, pre-commit setup, CI alignment, hooks for Terraform/K8s/Docker/Python/Markdown.

---

## 1) Why Makefile + pre-commit?

## Makefile gives:
- One-command developer workflows (`make test`, `make lint`, `make deploy`)
- Consistent commands across team members
- Easy CI reuse

## pre-commit gives:
- Automatic checks before commit
- Prevents bad formatting/lint/security issues entering Git history
- Faster feedback loop than waiting for CI

Together:
- **Local guardrails + repeatable automation**

---

## 2) Install Prerequisites

```bash
sudo apt update
sudo apt install -y make python3 python3-pip shellcheck
pip3 install --user pre-commit
```

If `pre-commit` not in PATH:
```bash
export PATH="$HOME/.local/bin:$PATH"
```

Verify:
```bash
make --version
pre-commit --version
```

---

## 3) Makefile Fundamentals

Create `Makefile`:

```make
SHELL := /bin/bash
.DEFAULT_GOAL := help

.PHONY: help fmt lint test validate clean

help: ## Show available targets
	@grep -E '^[a-zA-Z_-]+:.*?## ' Makefile | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "%-20s %s\n", $$1, $$2}'

fmt: ## Format source files
	@echo "Formatting..."
	@terraform fmt -recursive || true
	@prettier -w . || true

lint: ## Run linters
	@echo "Linting..."
	@tflint --recursive || true
	@shellcheck scripts/*.sh || true
	@markdownlint . || true

test: ## Run tests
	@echo "Running tests..."
	@pytest -q || true

validate: ## Validate IaC
	@terraform init -backend=false || true
	@terraform validate || true
	@kubectl kustomize k8s/ >/dev/null || true

clean: ## Cleanup generated artifacts
	@rm -rf .terraform .pytest_cache dist build
```

Run:
```bash
make help
make fmt
make lint
```

---

## 4) Better Makefile Patterns (Production-Ready)

## 4.1 Environment variables with defaults
```make
ENV ?= dev
NAMESPACE ?= default
IMAGE_TAG ?= latest
```

## 4.2 Fail-fast style
Avoid masking errors with `|| true` unless intentional.

## 4.3 Split large Makefiles
```make
include makefiles/lint.mk
include makefiles/test.mk
include makefiles/deploy.mk
```

## 4.4 Self-documenting target comments
Pattern:
```make
target: ## Description
```

---

## 5) pre-commit Setup

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-yaml
      - id: check-json
      - id: detect-private-key

  - repo: https://github.com/terraform-linters/tflint
    rev: v0.52.0
    hooks:
      - id: tflint

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint

  - repo: https://github.com/markdownlint/markdownlint
    rev: v0.12.0
    hooks:
      - id: markdownlint
```

Install hooks:
```bash
pre-commit install
```

Run all checks:
```bash
pre-commit run --all-files
```

---

## 6) Useful Hook Categories by Stack

## Generic
- trailing whitespace
- eof fixer
- yaml/json validators
- private key detection

## Terraform
- `terraform fmt`
- `terraform validate`
- `tflint`
- optional: `tfsec`, `checkov`

## Kubernetes
- YAML lint
- kubeconform/kubeval
- helm lint (if charts repo)

## Docker
- hadolint for Dockerfile

## Python
- black, isort, flake8, mypy

## Shell
- shellcheck, shfmt

## Markdown
- markdownlint

---

## 7) Add Security Scanners to pre-commit (Recommended)

Example adding Trivy config scan via local hook:

```yaml
- repo: local
  hooks:
    - id: trivy-config
      name: trivy-config
      entry: trivy config .
      language: system
      pass_filenames: false
```

Example adding gitleaks:
```yaml
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.18.4
  hooks:
    - id: gitleaks
```

---

## 8) Combine Make + pre-commit

In Makefile:
```make
precommit: ## Run pre-commit on all files
	pre-commit run --all-files

ci-check: fmt lint test precommit ## Local CI-equivalent checks
	@echo "All local checks passed"
```

This gives one command:
```bash
make ci-check
```

---

## 9) CI/CD Alignment (Important)

Mirror local checks in CI:
- Same linters
- Same formatting rules
- Same versions/pinned hook revs

Pipeline stage example:
1. `pre-commit run --all-files`
2. unit tests
3. build
4. security scans
5. deploy gates

---

## 10) Monorepo Strategy

For monorepos:
- Use path filters in hooks
- Use multiple Makefile include modules
- Use tool-specific targets (`make lint-terraform`, `make lint-python`)

Pre-commit hook example restricting files:
```yaml
- id: markdownlint
  files: ^docs/
```

---

## 11) Troubleshooting

## Hook not found
```bash
pre-commit autoupdate
pre-commit clean
pre-commit install
```

## Different results between local and CI
- Pin exact versions in hooks
- Ensure same toolchain/runtime versions

## Slow pre-commit
- Run only changed files by default (automatic)
- Keep heavy scans in CI/nightly or as manual make targets
- Use `pass_filenames: false` only when needed

## Make command fails due to tabs/spaces
Makefiles require **tabs** before command lines.

---

## 12) Recommended Starter Bundle (for DevOps repos)

Include at minimum:
1. trailing whitespace + EOF fixer  
2. YAML/JSON checks  
3. terraform fmt + validate + tflint  
4. shellcheck  
5. markdownlint  
6. detect-private-key + gitleaks  
7. optional trivy config scan  

---

## 13) Practice Tasks

1. Create Makefile with `help fmt lint test` targets  
2. Add `.pre-commit-config.yaml` and install hooks  
3. Run `pre-commit run --all-files`  
4. Intentionally break formatting and observe hook failure  
5. Add `make ci-check` and use before each push  
6. Replicate same checks in Jenkins/GitHub Actions  

---

## 14) Example Files (Ready to Copy)

## 14.1 Minimal Makefile
```make
SHELL := /bin/bash
.DEFAULT_GOAL := help
.PHONY: help fmt lint precommit ci-check

help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?## ' Makefile | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "%-15s %s\n", $$1, $$2}'

fmt: ## Format
	terraform fmt -recursive

lint: ## Lint
	tflint --recursive

precommit: ## Run pre-commit
	pre-commit run --all-files

ci-check: fmt lint precommit ## Run all checks
	@echo "Checks complete"
```

## 14.2 Minimal pre-commit config
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: detect-private-key
```

---

## 15) Daily Cheat Sheet

```bash
pre-commit install
pre-commit run --all-files
pre-commit autoupdate

make help
make fmt
make lint
make ci-check
```

---

## Final Notes

- If your team adopts only one local quality tool, choose pre-commit.
- If your team adopts one workflow standardizer, choose Makefile.
- Using both creates a strong baseline DevOps engineering system.