# GitHub Actions Complete Guide (Beginner to Advanced)
## CI/CD Automation on GitHub

> This file is detailed and practical.
> Covers workflow syntax, runners, secrets, environments, reusable workflows, security, troubleshooting.

---

## 1) What is GitHub Actions and Why it Matters

GitHub Actions is GitHub’s native CI/CD platform to automate build, test, security checks, and deployments.

## Why use it?
- Tight integration with GitHub repos/PRs
- YAML-based workflow-as-code
- Large action ecosystem
- Easy matrix builds and reusable workflows
- Great for automation beyond CI/CD too

---

## 2) Core Concepts

- **Workflow**: automation file in `.github/workflows/*.yml`
- **Event**: trigger (`push`, `pull_request`, `workflow_dispatch`, etc.)
- **Job**: a group of steps running on a runner
- **Step**: individual command/action
- **Runner**: machine that executes job (GitHub-hosted or self-hosted)
- **Action**: reusable automation unit

---

## 3) First Workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Show environment
        run: |
          uname -a
          node -v || true
          python3 --version || true

      - name: Dummy test
        run: echo "CI pipeline running..."
```

---

## 4) Runner Types

## GitHub-hosted runners
- Managed by GitHub
- Fast setup
- Ephemeral clean environment each run

## Self-hosted runners
- Installed in your own infra
- Needed for private network access/custom tooling
- Requires patching and security hardening by your team

---

## 5) Common Triggers

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - "src/**"
      - ".github/workflows/**"
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"
  workflow_dispatch:
```

Use `paths` to avoid unnecessary runs.

---

## 6) Build Matrix Strategy

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [ "3.10", "3.11", "3.12" ]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest -q
```

Why:
- Validate across versions in parallel.

---

## 7) Caching Dependencies

Example (pip cache):
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.11"
    cache: "pip"
```

Example (npm cache):
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "npm"
```

Benefits:
- Faster builds
- Lower CI cost/time

---

## 8) Secrets and Variables

## Repository secrets
- Settings → Secrets and variables → Actions → New repository secret

Usage:
```yaml
env:
  APP_ENV: prod

steps:
  - name: Use secret
    run: echo "Token length: ${#API_TOKEN}"
    env:
      API_TOKEN: ${{ secrets.API_TOKEN }}
```

Best practices:
- Never echo secret values
- Use least-privileged tokens
- Rotate regularly

---

## 9) Environments and Protection Rules

Use environments for controlled deploys:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploying to prod..."
```

Environment features:
- Required reviewers
- Wait timers
- Environment-scoped secrets

---

## 10) Artifacts (Upload/Download)

Upload build output:
```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
```

Download artifact (another job):
```yaml
- name: Download artifact
  uses: actions/download-artifact@v4
  with:
    name: build-output
```

---

## 11) Reusable Workflows

Reusable workflow (`.github/workflows/reusable-ci.yml`):
```yaml
name: Reusable CI
on:
  workflow_call:
    inputs:
      run-tests:
        required: true
        type: boolean

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Reusable workflow"
      - if: ${{ inputs.run-tests }}
        run: echo "Running tests"
```

Caller workflow:
```yaml
jobs:
  call-reusable:
    uses: ./.github/workflows/reusable-ci.yml
    with:
      run-tests: true
```

---

## 12) Composite Actions (Reusable Step Logic)

Use composite action when you want to bundle repeated step logic.
Path example:
- `.github/actions/setup-tools/action.yml`

Good for:
- standardized tool bootstrap
- repeated scripts across workflows

---

## 13) OIDC for Cloud Auth (Recommended)

Instead of long-lived cloud secrets, use OIDC federation for short-lived credentials.

Benefits:
- No static cloud keys in GitHub secrets
- Better security posture

Requires cloud IAM trust config (AWS/Azure/GCP specific setup).

---

## 14) Security Best Practices

1. Pin actions to major or full SHA where possible  
2. Restrict workflow permissions  
3. Use least privilege for `GITHUB_TOKEN`  
4. Protect main branch with required checks  
5. Use CODEOWNERS for workflow changes  
6. Scan dependencies and code (CodeQL/Trivy/SAST)  

Set permissions explicitly:
```yaml
permissions:
  contents: read
```

---

## 15) Example: CI + Docker Build + Trivy Scan

```yaml
name: ci-docker-trivy

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH
```

---

## 16) Deployment Pattern (Branch-based)

Example:
- PR -> run test/lint only
- Merge to main -> build + deploy to dev
- Tag release -> deploy to prod with environment approval

Use conditions:
```yaml
if: github.ref == 'refs/heads/main'
```

---

## 17) Troubleshooting

## 17.1 Workflow not triggering
- wrong branch filter
- workflow file not in default branch
- syntax errors

Use:
- Actions tab logs
- YAML validation
- check `on:` conditions

## 17.2 Permission denied for API/repo write
Add explicit permissions:
```yaml
permissions:
  contents: write
```

## 17.3 Action version issues
- update to supported action version
- pin known stable release

## 17.4 Slow workflows
- enable cache
- run jobs in parallel where possible
- avoid unnecessary steps on docs-only changes (use `paths` filters)

---

## 18) Practice Tasks

1. Create CI workflow for push + PR  
2. Add matrix test for Python/Node versions  
3. Add dependency caching  
4. Add artifact upload/download across jobs  
5. Add Trivy scan step and fail on CRITICAL  
6. Add protected production deployment environment with manual approval  
7. Refactor repeated logic into reusable workflow  

---

## 19) Daily Cheat Sheet

```text
Workflow file path: .github/workflows/*.yml
Common actions:
- actions/checkout@v4
- actions/setup-node@v4
- actions/setup-python@v5
- actions/upload-artifact@v4
- actions/download-artifact@v4
```

Quick checks:
- Trigger event config (`on:`)
- Branch/path filters
- Secrets availability
- Job permissions block

---

## 20) Self-Hosted Runner Setup

## Why
Needed for private network access, custom hardware (GPU), or licensed tooling GitHub-hosted runners don't have.

## Register a runner (from repo/org Settings → Actions → Runners → New self-hosted runner)
```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

./config.sh --url https://github.com/<org>/<repo> --token <REGISTRATION_TOKEN>
sudo ./svc.sh install
sudo ./svc.sh start
```

## Target it in a workflow
```yaml
jobs:
  build:
    runs-on: self-hosted
```
Or with labels for specific hardware:
```yaml
runs-on: [self-hosted, linux, gpu]
```

Security note: self-hosted runners on public repos are risky — a malicious PR can execute code on your infrastructure. Restrict to private repos or require approval for fork PRs (Settings → Actions → Fork pull request workflows).

---

## 21) Concurrency Control

## Why
Prevent overlapping deploys or wasted runs when someone pushes multiple commits quickly.

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

Placed at the workflow or job level — cancels any in-progress run with the same group key when a new one starts.

---

## 22) Container Jobs and Service Containers

## Run the job itself inside a container
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: node:20-alpine
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

## Spin up service containers (e.g., a database for integration tests)
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: psql -h localhost -U postgres -c '\l'
        env:
          PGPASSWORD: test
```

---

## 23) GitHub CLI (`gh`) for Automation

## Why
Script repo/PR/workflow operations from the terminal or from inside workflow steps.

## Install
```bash
sudo apt install -y gh
gh auth login
```

## Common operations
```bash
gh pr create --title "feat: new endpoint" --body "Adds X" --base main
gh pr list
gh pr checks <pr-number>
gh workflow list
gh workflow run ci.yml
gh run list --workflow=ci.yml
gh run watch
gh secret set API_TOKEN --body "abc123"
```

Useful inside a workflow step too (authenticated automatically via `GITHUB_TOKEN`):
```yaml
- name: Comment on PR
  run: gh pr comment ${{ github.event.pull_request.number }} --body "Build passed ✅"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Final Notes

- GitHub Actions is enough for most modern CI/CD needs.
- Standardize a reusable workflow library early.
- Pair with branch protection, security scans, and environment approvals for production-grade pipelines.
- Use concurrency groups to avoid overlapping deploys, and reach for `gh` CLI to script anything the UI would otherwise require clicking through.
