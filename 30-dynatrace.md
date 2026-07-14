# Dynatrace Complete Guide (Beginner to Advanced)
## Full-Stack Observability + AIOps + Security for DevOps/SRE

> Practical guide for onboarding, instrumentation, dashboards, SLOs, alerting, and operations.

---

## 1) What is Dynatrace and Why it Matters

Dynatrace is an observability and application security platform covering:
- Infrastructure monitoring
- APM (application performance monitoring)
- Real user monitoring (RUM)
- Distributed tracing
- Logs and metrics analytics
- AIOps-based problem detection
- Kubernetes/cloud-native observability

## Why DevOps/SRE teams use it
- End-to-end service dependency visibility
- Fast root cause analysis
- AI-assisted incident correlation
- SLO/error-budget based reliability operations
- Unified view across infra, apps, and user experience

---

## 2) Dynatrace Core Architecture Concepts

Main concepts (platform terminology may vary by edition/version):
- **OneAgent**: host/container agent collecting infra + process + trace telemetry
- **ActiveGate**: secure routing/proxy/extension execution (optional/enterprise scenarios)
- **Management zones**: scope/isolation by team/environment/app
- **Entities**: hosts, processes, services, applications, k8s objects
- **Problems/Events**: correlated incidents and anomalies
- **Davis AI**: causation and anomaly analytics engine

---

## 3) Deployment/Onboarding Models

1. SaaS tenant + OneAgent rollout ✅ common starting point  
2. Managed/self-hosted cluster model (enterprise-controlled environments)  
3. Kubernetes Operator-based onboarding  
4. Cloud integrations (AWS/Azure/GCP accounts and services)

For most teams: start with SaaS + OneAgent + K8s integration.

---

## 4) Initial Setup Checklist

1. Create/select Dynatrace environment (tenant)
2. Define naming conventions (env/app/team/service tags)
3. Create management zones (dev/stage/prod/team boundaries)
4. Onboard hosts/K8s clusters
5. Integrate cloud provider APIs
6. Configure alert routing (email/Slack/PagerDuty/webhook)
7. Build golden dashboards + SLOs

---

## 5) OneAgent Installation (Linux Host)

From Dynatrace UI:
- Deploy Dynatrace -> Start installation -> Linux
- Copy generated command with environment-specific token

Typical pattern:
```bash
sudo sh Dynatrace-OneAgent-Linux.sh
```

Verify on host:
```bash
systemctl status oneagent
```

In Dynatrace UI:
- check host appears healthy
- confirm process/service detection

> Use platform-generated install command for your tenant (contains exact endpoint/token/options).

---

## 6) Kubernetes Monitoring with Dynatrace Operator

High-level steps:
1. Install Dynatrace Operator CRDs
2. Create `DynaKube` custom resource with API URL and token secrets
3. Enable full-stack or application-only injection mode
4. Validate pods, node metrics, workloads, service flow

Typical benefits:
- automatic workload/process discovery
- cluster/node/pod health analytics
- service-to-service dependency mapping

---

## 7) Tags, Auto-Tagging, and Naming Strategy

Use tags for:
- `env:prod|stage|dev`
- `team:payments|platform|security`
- `service_tier:frontend|backend|db`

Why this matters:
- cleaner dashboards
- scoped alerting
- role-based views and ownership clarity

Set auto-tagging rules based on:
- K8s labels
- cloud metadata
- process group names
- host naming conventions

---

## 8) Service Detection and Distributed Tracing

Dynatrace auto-detects many technologies and service endpoints.
For trace-rich setups:
- ensure supported runtimes/frameworks
- propagate trace context across services
- validate service flow map and latency breakdown

Use traces to answer:
- where latency originates
- which downstream dependency is failing
- what % of requests affected

---

## 9) Real User Monitoring (RUM) and Synthetic Monitoring

## RUM
- browser/mobile user experience visibility
- page load, JS errors, geo/device breakdown

## Synthetic
- scripted/browser/http monitors
- proactive external checks for uptime and response time

Recommended:
- combine synthetic uptime checks + RUM actual user impact for incident prioritization.

---

## 10) Logs and Metrics in Dynatrace

Dynatrace can ingest logs and metrics for context-rich analysis.

Best practices:
- onboard only high-value logs first (avoid noise explosion)
- structure logs (JSON) where possible
- normalize key fields (service, env, request_id, level)

Use correlation:
- problems -> related logs -> traces -> infra events

---

## 11) Dashboards and Notebooks/Analysis

Build team dashboards with:
- golden signals: latency, traffic, errors, saturation
- pod/node health and restart trends
- top failing endpoints
- DB call latency and external dependency errors
- deployment markers vs incident spikes

Create separate views:
- Executive reliability summary
- SRE on-call dashboard
- Service owner dashboard

---

## 12) Alerting and Problem Routing

Set up alert profiles/notification integrations:
- PagerDuty/Opsgenie
- Slack/Teams
- Email/webhooks
- ITSM integrations (ServiceNow/Jira flows)

Good alerting practice:
1. route by ownership tags/management zones  
2. avoid duplicate channels per same event  
3. tune thresholds to reduce noise  
4. include runbook links in notifications  

---

## 13) SLOs and Error Budgets

Define SLOs around user-facing reliability:
- availability % (e.g., 99.9%)
- latency target (e.g., p95 < 400ms)
- error rate threshold

Track:
- current SLO burn
- error budget remaining
- release impact on SLO compliance

Use SLOs for release gating and post-incident review.

---

## 14) Release/Deployment Tracking

Integrate CI/CD with Dynatrace deployment events:
- annotate version releases
- correlate incident onset with deployment timeline
- compare pre/post release service health

This dramatically improves MTTR during rollout incidents.

---

## 15) Security and Access Control

Key controls:
- SSO/SAML integration
- role-based access per team/zone
- token scoping (least privilege API tokens)
- audit trail for config changes
- network controls for ActiveGate/agent communications

Never share broad admin API tokens in CI logs.

---

## 16) API and Automation Use Cases

Dynatrace APIs are commonly used for:
- dashboard as code
- metric extraction to external systems
- automated SLO reporting
- problem/event export to incident tools
- config drift checks in platform setup

Automate recurring admin tasks via CI jobs.

---

## 17) Performance and Cost Governance

Control telemetry cost/cardinality by:
- limiting noisy custom metrics dimensions
- filtering low-value logs
- controlling retention tiers
- tagging standards to prevent duplicate entities

Observability maturity = signal quality, not raw data volume.

---

## 18) Troubleshooting Matrix

## Host not showing in Dynatrace
- OneAgent install failed
- outbound connectivity blocked
- wrong tenant endpoint/token

Checks:
```bash
systemctl status oneagent
journalctl -u oneagent -n 200 --no-pager
```

## Kubernetes data missing
- Operator/DynaKube misconfiguration
- token secret issues
- namespace injection settings incorrect

## Too many alerts/noise
- broad anomaly thresholds
- missing ownership scoping
- duplicate notification configs

## Service not auto-detected
- unsupported runtime mode
- traffic too low
- process grouping not matching expectations

---

## 19) Best Practices (DevOps/SRE)

1. Start with one production-critical service first  
2. Standardize tags and management zones early  
3. Build golden signal dashboards before advanced custom views  
4. Tune alerting with on-call feedback every sprint  
5. Correlate deployments with problems by default  
6. Use SLOs to drive reliability decisions  
7. Document runbooks linked from alerts/problems  

---

## 20) Practice Tasks

1. Onboard 2 Linux hosts with OneAgent  
2. Integrate one Kubernetes cluster using Operator  
3. Create tags for env/team/service ownership  
4. Build dashboard for latency, errors, restarts  
5. Configure one alert route to Slack/PagerDuty  
6. Define one availability SLO for key API  
7. Validate incident workflow: problem -> trace -> root cause -> remediation  

---

## 21) Daily Operations Cheat Sheet

- Check **Problems** feed and impacted services
- Review **SLO burn/error budget**
- Verify **deployment markers vs anomalies**
- Inspect **top service latency and error contributors**
- Validate **K8s workload health** (restarts, saturation)
- Review **alert noise** and tune rules weekly

Host quick checks:
```bash
systemctl status oneagent
journalctl -u oneagent -f
```

---

## 22) Grail (Unified Data Lakehouse)

## Why
Newer Dynatrace platform versions store logs, metrics, traces, and events in a unified data lakehouse called **Grail**, queried with a single language (DQL) instead of separate silos per data type.

## DQL basics (Notebooks or API)
```dql
fetch logs
| filter k8s.namespace.name == "dev"
| filter loglevel == "ERROR"
| summarize count(), by:{dt.entity.service}
```

```dql
fetch spans
| filter request.is_failed == true
| summarize count(), by:{service.name}
| sort count() desc
```

Grail's advantage over the classic model: correlate logs/traces/metrics/events in one query without separate export/import pipelines.

---

## 23) Configuration as Code (Monaco)

## Why
Manage dashboards, alerting profiles, management zones, and other config declaratively in Git instead of clicking through the UI per environment — essential once you have more than one tenant (dev/stage/prod).

## Install Monaco
```bash
curl -L https://github.com/Dynatrace/dynatrace-configuration-as-code/releases/latest/download/monaco-linux-amd64 -o monaco
chmod +x monaco
```

## Example config structure
```
project/
├── manifest.yaml
└── alerting-profile/
    └── config.yaml
```

## Deploy config to an environment
```bash
./monaco deploy manifest.yaml
```

Pairs naturally with a Git-based promotion flow: change reviewed in a PR → applied to dev → promoted to prod after validation.

---

## 24) Extensions Framework (Custom Metrics/Integrations)

## Why
Pull in telemetry Dynatrace doesn't natively understand — custom hardware, niche databases, internal services — via the Extensions 2.0 framework.

## High-level flow
1. Define an extension YAML (metrics, endpoints, polling interval)
2. Package and upload via `dt-cli` or the UI
3. Deploy to an ActiveGate or OneAgent-monitored host
4. Metrics appear alongside native Dynatrace data, usable in dashboards/alerts like any other metric

```bash
dt extension init my-custom-extension
dt extension build
dt extension upload my-custom-extension.zip
```

---

## 25) Davis AI / Problems API (Automation Hook)

## Why
Pull correlated "Problems" (Davis AI's root-cause-analyzed incidents) into external tools (ticketing, ChatOps) instead of only viewing them in the UI.

## Query recent problems
```bash
curl -X GET "https://<tenant>.live.dynatrace.com/api/v2/problems" \
  -H "Authorization: Api-Token <TOKEN>"
```

## Filter to open, high-impact problems
```bash
curl -G "https://<tenant>.live.dynatrace.com/api/v2/problems" \
  -H "Authorization: Api-Token <TOKEN>" \
  --data-urlencode 'problemSelector=status(OPEN),severityLevel(AVAILABILITY)'
```

Use this to auto-create tickets, post to Slack, or feed an internal incident dashboard — the same root-cause data Davis surfaces in-app.

---

## Final Notes

- Dynatrace is strongest when you combine: topology + traces + logs + SLOs + deployment context.
- Success depends on thoughtful tagging, alert tuning, and ownership mapping.
- Start small, standardize onboarding, then scale across teams.
- Once running multiple tenants/environments, manage config with Monaco instead of manual UI clicks, and use the Problems API to wire Davis AI's findings into your existing incident workflow.
