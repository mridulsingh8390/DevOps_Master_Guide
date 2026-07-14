# Datadog Complete Guide (Beginner to Advanced)
## Full-Stack Observability + Security Monitoring for DevOps/SRE

> This file is detailed and practical.
> Covers agent install, infrastructure/APM/logs, dashboards, monitors, Kubernetes integration, config-as-code, troubleshooting.

---

## Table of Contents

1. What is Datadog and why it matters
2. Core concepts
3. Install the Datadog Agent (Linux host)
4. Kubernetes install (Helm/Operator)
5. Infrastructure monitoring
6. APM and distributed tracing
7. Log management
8. Dashboards
9. Monitors and alerting
10. Tags — the organizing principle
11. Synthetic monitoring
12. Real User Monitoring (RUM)
13. Notebooks and Watchdog (AI-assisted analysis)
14. Datadog CI Visibility
15. Config as Code (Terraform provider)
16. Security monitoring (CSM/CWS)
17. Cost and usage governance
18. Troubleshooting matrix
19. Best practices
20. Practice lab tasks
21. Daily cheat sheet

---

## 1) What is Datadog and Why it Matters

## What
Datadog is a SaaS observability platform unifying infrastructure metrics, APM/traces, logs, RUM, synthetics, and security monitoring in one product with a shared tagging model.

## Why DevOps/SRE teams use it
- One agent collects metrics, traces, and logs — less tooling sprawl than stitching together separate open-source stacks
- Strong out-of-the-box integrations (700+) for cloud services, databases, queues, etc.
- Correlate a metric spike → related traces → related logs → related deployment, without switching tools
- Widely adopted, so hiring/onboarding engineers already familiar with it is easier than a fully bespoke stack

---

## 2) Core Concepts

- **Agent**: the process running on every host/container collecting and forwarding telemetry
- **Integration**: a pre-built config bundle for a specific technology (Postgres, Nginx, Kafka, etc.)
- **Tag**: a `key:value` label attached to any telemetry — the primary way data is filtered/grouped/scoped across the entire platform
- **Monitor**: Datadog's term for an alert rule
- **Dashboard**: a saved collection of widgets/graphs
- **APM/Trace**: a distributed request trace across services
- **Facet**: an indexed log field usable for filtering/faceted search

---

## 3) Install the Datadog Agent (Linux Host)

## 3.1 One-line install
```bash
DD_API_KEY=<YOUR_API_KEY> DD_SITE="datadoghq.com" bash -c \
  "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"
```

## 3.2 Verify
```bash
sudo systemctl status datadog-agent
sudo datadog-agent status
```

## 3.3 Config file
`/etc/datadog-agent/datadog.yaml`
```yaml
api_key: <YOUR_API_KEY>
site: datadoghq.com
tags:
  - env:prod
  - team:platform
logs_enabled: true
```

## 3.4 Restart after config changes
```bash
sudo systemctl restart datadog-agent
```

---

## 4) Kubernetes Install (Helm)

## 4.1 Install via Helm
```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update

helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=<YOUR_API_KEY> \
  --set datadog.site=datadoghq.com \
  --set datadog.logs.enabled=true \
  --set datadog.apm.portEnabled=true \
  -n datadog --create-namespace
```

## 4.2 Verify
```bash
kubectl get pods -n datadog
kubectl logs -n datadog -l app=datadog-agent -c agent
```

The Helm chart deploys the Agent as a DaemonSet (one per node) plus a Cluster Agent (single deployment) that handles cluster-level metadata and reduces API server load.

---

## 5) Infrastructure Monitoring

Once the agent is running, hosts appear automatically under **Infrastructure → Host Map**.

## Key built-in views
- **Host Map**: visual grid of all hosts colored by a chosen metric (CPU, memory)
- **Container Map**: same concept for containers/pods
- **Process view**: per-host process-level CPU/memory breakdown

## Enable an integration (example: Nginx)
```yaml
# /etc/datadog-agent/conf.d/nginx.d/conf.yaml
instances:
  - nginx_status_url: http://localhost:81/nginx_status/
```
```bash
sudo systemctl restart datadog-agent
sudo datadog-agent status | grep -A5 nginx
```

---

## 6) APM and Distributed Tracing

## 6.1 Enable APM in the agent config
```yaml
apm_config:
  enabled: true
```

## 6.2 Instrument an application (auto-instrumentation example, Python)
```bash
pip install ddtrace
ddtrace-run python app.py
```

Environment variables commonly set alongside:
```bash
export DD_SERVICE="checkout-api"
export DD_ENV="prod"
export DD_VERSION="1.4.0"
```

## 6.3 View traces
**APM → Traces**, filtered by `service:checkout-api`. Datadog automatically builds a service map showing upstream/downstream dependencies from trace data.

---

## 7) Log Management

## 7.1 Enable log collection for a specific log file
```yaml
# /etc/datadog-agent/conf.d/myapp.d/conf.yaml
logs:
  - type: file
    path: /var/log/myapp/*.log
    service: myapp
    source: python
```
```bash
sudo systemctl restart datadog-agent
```

## 7.2 Kubernetes log collection
Enabled via the Helm value `datadog.logs.enabled=true` (Section 4) — the agent auto-discovers pod logs, no per-pod config needed.

## 7.3 Log-based metrics (turn log patterns into a queryable metric)
**Logs → Generate Metrics** — e.g., count of `status:500` log lines per minute, without needing to instrument the app separately.

---

## 8) Dashboards

## Build via UI
Dashboards → New Dashboard → add widgets (timeseries, top list, heatmap, query value).

## Dashboards as JSON (exportable/importable)
```bash
curl -X POST "https://api.datadoghq.com/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

Recommended dashboard set (mirrors the golden-signals pattern from the Dynatrace/Prometheus guides in this series): a service owner dashboard (latency/errors/traffic/saturation), an on-call dashboard (active monitors + recent deploys), and an executive summary.

---

## 9) Monitors and Alerting

## 9.1 Metric monitor (via API)
```bash
curl -X POST "https://api.datadoghq.com/api/v1/monitor" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "High CPU on web hosts",
    "type": "metric alert",
    "query": "avg(last_5m):avg:system.cpu.user{env:prod} > 85",
    "message": "CPU is above 85% on {{host.name}} @slack-alerts-prod",
    "tags": ["team:platform"],
    "options": {"thresholds": {"critical": 85, "warning": 70}}
  }'
```

## 9.2 Notification channels
Reference integrations directly in the monitor message: `@slack-<channel>`, `@pagerduty-<service>`, `@webhook-<name>`, or a plain email address.

## 9.3 Monitor types worth knowing
- **Metric alert**: threshold on a metric query
- **Anomaly monitor**: ML-based baseline deviation instead of a fixed threshold
- **Log alert**: fires on log query volume/patterns
- **Composite monitor**: combines multiple monitors with AND/OR logic to reduce alert noise

---

## 10) Tags — The Organizing Principle

## Why tags matter more in Datadog than most tools
Every graph, monitor, dashboard, and log search is built around tag filtering — a consistent tagging strategy is the single highest-leverage thing to get right early.

## Recommended baseline tags
```yaml
tags:
  - env:prod
  - service:checkout-api
  - team:payments
  - version:1.4.0
```

## Unified Service Tagging
Datadog specifically recommends `env`, `service`, and `version` as standard tags — this trio is what links APM traces, logs, and metrics together automatically into one correlated view per service/deployment.

---

## 11) Synthetic Monitoring

## Why
Proactively test uptime/functionality from outside your infrastructure, before real users hit a problem.

## API test (via UI: Synthetics → New Test → API Test)
- URL, expected status code, expected response time threshold, run frequency, and locations to test from

## Browser test
Scripted multi-step user flows (e.g., login → add to cart → checkout) run on a schedule, screenshotting each step — catches UI regressions synthetic API tests can't.

---

## 12) Real User Monitoring (RUM)

## Why
See actual user experience (page load time, JS errors, geographic/device breakdown) rather than only synthetic checks.

## Initialize (browser SDK)
```javascript
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: '<APP_ID>',
  clientToken: '<CLIENT_TOKEN>',
  site: 'datadoghq.com',
  service: 'checkout-frontend',
  env: 'prod',
  sessionSampleRate: 100,
  trackUserInteractions: true,
});
```

RUM sessions can be correlated directly with backend APM traces when both are configured with matching `service`/`env` tags — end-to-end from a user's click to the backend span that served it.

---

## 13) Notebooks and Watchdog

## Notebooks
Combine graphs, log queries, and free-text analysis in one shareable document — useful for post-incident writeups where you want the actual data embedded, not just screenshots.

## Watchdog (automated anomaly detection)
Datadog's built-in ML layer surfaces anomalies automatically without manually configured monitors — worth checking **Watchdog → Alerts** during incident triage as a second opinion alongside your own monitors.

---

## 14) Datadog CI Visibility

## Why
Track CI pipeline performance and test flakiness the same way you track production services.

## Example (GitHub Actions integration)
```yaml
- name: Upload CI results to Datadog
  uses: DataDog/datadog-ci-github-action@v1
  with:
    api-key: ${{ secrets.DD_API_KEY }}
```

Surfaces slow pipeline stages, flaky test detection, and build failure trends in the Datadog UI — the same triage workflow as production incidents, applied to CI.

---

## 15) Config as Code (Terraform Provider)

## Why
Manage monitors/dashboards/synthetic tests declaratively instead of manual UI clicks — essential once you have more than a couple of environments.

```hcl
terraform {
  required_providers {
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }
}

provider "datadog" {
  api_key = var.dd_api_key
  app_key = var.dd_app_key
}

resource "datadog_monitor" "high_cpu" {
  name    = "High CPU on web hosts"
  type    = "metric alert"
  query   = "avg(last_5m):avg:system.cpu.user{env:prod} > 85"
  message = "CPU is above 85% @slack-alerts-prod"

  monitor_thresholds {
    critical = 85
    warning  = 70
  }
}
```

```bash
terraform plan
terraform apply
```

This is the recommended pattern once monitor/dashboard count grows beyond what's comfortable to manage by hand — same rationale as the Terraform guide elsewhere in this series.

---

## 16) Security Monitoring (CSM/CWS)

## Cloud Security Management (CSM)
Scans cloud resource configuration (similar in spirit to Trivy's `config` scanning) for misconfigurations against CIS benchmarks — visible under **Security → Cloud Security Management**.

## Cloud Workload Security (CWS)
Runtime threat detection on hosts/containers (unexpected process execution, privilege escalation, file integrity events) — requires enabling the security agent module:
```yaml
# datadog.yaml
runtime_security_config:
  enabled: true
```

---

## 17) Cost and Usage Governance

Datadog bills primarily on host count, custom metrics volume, and log ingestion/retention — a few practical controls:

- **Custom metrics**: avoid high-cardinality tags (e.g., tagging by `user_id` or `request_id`) — this is the single most common cause of runaway custom metrics cost
- **Log pipelines**: use exclusion filters to drop noisy/low-value logs before indexing, while still forwarding them to cheaper long-term storage if needed
- **Indexed spans**: APM retention filters control what percentage of traces get indexed (searchable) vs just ingested for metrics — tune per service based on actual debugging value

---

## 18) Troubleshooting Matrix

## 18.1 Host not appearing in Datadog
```bash
sudo datadog-agent status
sudo systemctl status datadog-agent
sudo journalctl -u datadog-agent -n 200 --no-pager
```
Common causes: wrong API key, outbound network blocked to Datadog's intake endpoints, wrong `site` config (`datadoghq.com` vs `datadoghq.eu` vs others).

## 18.2 Integration showing no data
```bash
sudo datadog-agent status | grep -A10 <integration_name>
```
Check the integration's specific config file under `/etc/datadog-agent/conf.d/`.

## 18.3 APM traces missing
- `apm_config.enabled: true` not set in agent config
- Application not actually instrumented (`ddtrace-run` missing, or SDK not initialized)
- `DD_ENV`/`DD_SERVICE` mismatch between app and dashboard filter

## 18.4 Kubernetes pods not sending logs
```bash
kubectl logs -n datadog -l app=datadog-agent -c agent | grep -i log
```
Verify `datadog.logs.enabled=true` was actually set on the Helm install/upgrade.

## 18.5 Monitor too noisy
- Add a composite monitor combining conditions, or increase the `for` duration equivalent (evaluation window) to avoid firing on transient blips

---

## 19) Best Practices

1. Standardize `env`/`service`/`version` tags from day one (Unified Service Tagging)
2. Avoid high-cardinality tags on custom metrics — biggest cost driver
3. Manage monitors/dashboards as code (Terraform provider) once past a handful
4. Build golden-signal dashboards before deep custom ones
5. Use composite monitors to cut alert noise instead of loosening thresholds
6. Correlate deployment markers with monitors for fast MTTR during rollout incidents
7. Review Watchdog anomalies during incident triage as a second signal source

---

## 20) Practice Lab Tasks

1. Install the Agent on a Linux host and verify it reporting
2. Enable an integration (e.g., Nginx or Postgres) and confirm metrics flow
3. Instrument a small app with APM and view its service map
4. Enable log collection for an app log file and build a log-based metric
5. Create a metric monitor with Slack notification
6. Build a dashboard with the four golden signals for one service
7. Manage one monitor via the Terraform provider
8. Set up a synthetic API test for a public endpoint

---

## 21) Daily Cheat Sheet

```bash
# Agent
sudo datadog-agent status
sudo systemctl status datadog-agent
sudo systemctl restart datadog-agent
sudo journalctl -u datadog-agent -f

# Kubernetes
kubectl get pods -n datadog
kubectl logs -n datadog -l app=datadog-agent -c agent
```

Daily UI checks:
- Monitor status (Monitors → Manage Monitors, filter by `Alert`)
- Watchdog anomalies
- APM error rate by service
- Log volume/ingestion trends vs plan limits

---

## Final Notes

- Datadog's biggest strength is correlation — metrics, traces, logs, and RUM all sharing one tagging model in one product.
- Get Unified Service Tagging (`env`/`service`/`version`) right early; almost everything else compounds on top of it.
- Watch custom metric cardinality closely — it's the most common source of unexpected cost.
- Move to Terraform-managed monitors/dashboards once manual UI management stops scaling across environments.
