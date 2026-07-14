# GitLab CI/CD Complete Guide (Beginner to Advanced)
## CI/CD Automation with GitLab

> This file is detailed and practical.
> Covers pipeline syntax, runners, variables/secrets, environments, templates, security scanning, troubleshooting.

---

## Table of Contents

1. What is GitLab CI/CD and why it matters
2. Core concepts
3. First pipeline
4. Runners (shared, group, project, self-hosted)
5. Stages, jobs, and rules
6. Variables and secrets
7. Artifacts and caching
8. Environments and deployments
9. `include` and reusable templates
10. Parent-child and multi-project pipelines
11. Built-in security scanning (SAST/DAST/dependency scanning)
12. Docker-in-Docker builds
13. Review apps
14. Merge request pipelines and rules
15. GitLab CLI (`glab`)
16. Troubleshooting matrix
17. Practice lab tasks
18. Daily cheat sheet

---

## 1) What is GitLab CI/CD and Why it Matters

## What
GitLab CI/CD is GitLab's built-in automation platform, driven by a single `.gitlab-ci.yml` file per repository.

## Why use it?
- Tightly integrated with GitLab repos/MRs/issues — no separate tool to wire up
- Free shared runners on GitLab.com, or self-hosted runners for private infra
- Built-in container registry, security scanning, and environments
- Strong support for parent-child and multi-project pipelines at scale

---

## 2) Core Concepts

- **Pipeline**: the full CI/CD run triggered by an event (push, MR, schedule, manual)
- **Stage**: a logical phase (`build`, `test`, `deploy`) — stages run sequentially, jobs within a stage run in parallel
- **Job**: an individual unit of work (a script + its config)
- **Runner**: the agent that executes jobs (shared, group, or project-specific)
- **Artifact**: files produced by a job, passed to later stages or downloadable
- **`.gitlab-ci.yml`**: the pipeline definition file at the repo root

---

## 3) First Pipeline

Create `.gitlab-ci.yml`:

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the app..."

test-job:
  stage: test
  script:
    - echo "Running tests..."

deploy-job:
  stage: deploy
  script:
    - echo "Deploying..."
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Push and check **CI/CD → Pipelines** in the GitLab UI.

---

## 4) Runners

## 4.1 Types
- **Shared runners**: provided by GitLab.com, available to any project
- **Group runners**: shared across projects in a group
- **Project runners**: dedicated to one project
- **Self-hosted runners**: registered by you, needed for private network access or custom tooling

## 4.2 Install and register a self-hosted runner
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner

sudo gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token <TOKEN> \
  --executor docker \
  --docker-image alpine:latest \
  --description "my-shell-runner"
```

## 4.3 Check status
```bash
sudo gitlab-runner status
sudo gitlab-runner list
```

## 4.4 Target a specific runner via tags
```yaml
deploy-job:
  stage: deploy
  tags:
    - self-hosted
  script:
    - echo "Deploying via self-hosted runner"
```

---

## 5) Stages, Jobs, and Rules

## 5.1 Custom stage order
```yaml
stages:
  - lint
  - build
  - test
  - scan
  - deploy
```

## 5.2 `rules` (modern conditional execution, preferred over `only`/`except`)
```yaml
deploy-prod:
  stage: deploy
  script:
    - echo "Deploying to prod"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
```

## 5.3 `when` options
- `on_success` (default): run if prior stages succeeded
- `on_failure`: run only if something failed (e.g., a rollback job)
- `manual`: require a person to click "run" in the UI
- `always`: run regardless of prior job status (e.g., cleanup)

---

## 6) Variables and Secrets

## 6.1 Project-level (Settings → CI/CD → Variables)
Mask and protect sensitive values in the UI — masked variables won't appear in job logs.

## 6.2 Use in a job
```yaml
deploy-job:
  script:
    - echo "Deploying with token length ${#API_TOKEN}"
  variables:
    DEPLOY_ENV: "production"
```

## 6.3 Predefined variables (always available)
- `CI_COMMIT_SHA`, `CI_COMMIT_BRANCH`, `CI_PROJECT_NAME`, `CI_PIPELINE_ID`, `CI_MERGE_REQUEST_IID`

```yaml
script:
  - echo "Building commit $CI_COMMIT_SHA on branch $CI_COMMIT_BRANCH"
```

## 6.4 Protected variables
Mark a variable "Protected" so it's only exposed on protected branches/tags — keeps prod secrets out of feature-branch pipelines.

---

## 7) Artifacts and Caching

## 7.1 Artifacts (pass files between stages, or download from UI)
```yaml
build-job:
  stage: build
  script:
    - mkdir -p dist && echo "built" > dist/app.txt
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
```

## 7.2 Cache (speed up repeated dependency installs)
```yaml
test-job:
  stage: test
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  script:
    - npm ci
    - npm test
```

Difference: artifacts are the job's *output* (passed forward, often downloadable); cache is *reused input* (like `node_modules`) to speed up future runs — not guaranteed to persist and shouldn't be relied on for correctness.

---

## 8) Environments and Deployments

```yaml
deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging"
  environment:
    name: staging
    url: https://staging.example.com

deploy-prod:
  stage: deploy
  script:
    - echo "Deploying to production"
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

Environments show up under **Deployments → Environments** in the UI, with deployment history and a one-click rollback to any prior successful deployment.

---

## 9) `include` and Reusable Templates

## Why
Avoid duplicating pipeline logic across many repos — centralize in a shared template repo.

## 9.1 Include a local file
```yaml
include:
  - local: '.gitlab/ci/build.yml'
```

## 9.2 Include from another project
```yaml
include:
  - project: 'platform/ci-templates'
    ref: main
    file: '/templates/docker-build.yml'
```

## 9.3 Extend a job template (`extends`)
```yaml
.base-deploy:
  script:
    - echo "Common deploy logic"
  tags:
    - self-hosted

deploy-staging:
  extends: .base-deploy
  environment:
    name: staging
```

---

## 10) Parent-Child and Multi-Project Pipelines

## Child pipeline (`trigger`)
```yaml
generate-config:
  stage: build
  script:
    - echo "generating child pipeline config"

trigger-child:
  stage: test
  trigger:
    include:
      - artifact: generated-config.yml
        job: generate-config
```

## Multi-project pipeline (trigger a pipeline in another repo)
```yaml
trigger-downstream:
  stage: deploy
  trigger:
    project: platform/deploy-repo
    branch: main
```

Useful for monorepo-style fan-out (one job per service) or triggering a separate deploy repo from an app repo.

---

## 11) Built-in Security Scanning

GitLab ships pre-built CI templates for common scans — just include them:

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
```

Results appear in **Security & Compliance → Vulnerability Report**, and as inline annotations on merge requests. DAST additionally requires a running review app/target URL:

```yaml
include:
  - template: Security/DAST.gitlab-ci.yml

dast:
  variables:
    DAST_WEBSITE: https://staging.example.com
```

---

## 12) Docker-in-Docker (DinD) Builds

## Why
Build container images inside a CI job that itself runs in a container.

```yaml
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

`$CI_REGISTRY_IMAGE` and `$CI_REGISTRY_USER`/`$CI_REGISTRY_PASSWORD` are predefined when using GitLab's built-in Container Registry — no manual credential setup needed.

---

## 13) Review Apps

## Why
Spin up a temporary, isolated environment per merge request for reviewers to click through before merging.

```yaml
review-app:
  stage: deploy
  script:
    - echo "Deploying review app for $CI_MERGE_REQUEST_IID"
    - ./deploy-review.sh $CI_ENVIRONMENT_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.review.example.com
    on_stop: stop-review-app
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

stop-review-app:
  stage: deploy
  script:
    - ./teardown-review.sh $CI_ENVIRONMENT_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
```

---

## 14) Merge Request Pipelines and Rules

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

This `workflow: rules` block controls whether a pipeline runs *at all* — a common pattern is "only run on MRs or on main", avoiding duplicate pipelines for every push to a branch that also has an open MR.

---

## 15) GitLab CLI (`glab`)

## Install
```bash
sudo apt install -y glab
glab auth login
```

## Common operations
```bash
glab mr create --title "feat: new endpoint" --description "Adds X"
glab mr list
glab mr view 42
glab ci status
glab ci view
glab ci trace
```

---

## 16) Troubleshooting Matrix

## 16.1 Pipeline not triggering
- `.gitlab-ci.yml` has a syntax error — check **CI/CD → Pipelines → validate** or the pipeline editor's lint view
- `workflow: rules` excludes the current event

## 16.2 Job stuck "pending"
- No runner available matching the job's tags
```bash
sudo gitlab-runner list
```

## 16.3 Docker-in-Docker failures
- Missing `services: [docker:24-dind]`
- TLS cert dir mismatch — check `DOCKER_TLS_CERTDIR`

## 16.4 Secret variable not available in job
- Variable marked "Protected" but branch isn't protected
- Variable scoped to wrong environment

## 16.5 Cache not speeding things up
- Cache key changes every run (e.g., includes commit SHA) — use a stable key like `${CI_COMMIT_REF_SLUG}`

---

## 17) Practice Lab Tasks

1. Create a 3-stage pipeline (build/test/deploy)
2. Register a self-hosted runner and target it via tags
3. Add artifacts passed from build to test stage
4. Add dependency caching for a package manager
5. Configure a staging environment with manual promotion to prod
6. Include GitLab's SAST and Secret Detection templates
7. Build and push a Docker image using DinD to the built-in registry
8. Set up a review app that tears down on MR close

---

## 18) Daily Cheat Sheet

```bash
# Runner
sudo gitlab-runner status
sudo gitlab-runner list
sudo gitlab-runner register

# CLI
glab mr create
glab ci status
glab ci trace

# Predefined variables often used in scripts
echo $CI_COMMIT_SHA
echo $CI_COMMIT_BRANCH
echo $CI_PROJECT_NAME
echo $CI_PIPELINE_ID
```

---

## Final Notes

- GitLab CI/CD's biggest strength is how much comes built-in: registry, security scanning, environments, review apps — no separate tools to wire together.
- Use `rules`/`workflow: rules` over the older `only`/`except` syntax for anything new.
- Centralize shared logic with `include`/`extends` once you have more than a couple of pipelines.
- Pair with self-hosted runners for private infra access, and lean on the built-in security templates before reaching for external scanners.
