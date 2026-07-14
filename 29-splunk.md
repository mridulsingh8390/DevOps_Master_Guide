# Splunk Complete Guide (Beginner to Advanced)
## Enterprise Log Management + SIEM Foundations for DevOps/SecOps

> Practical guide for installation, data onboarding, SPL queries, dashboards, alerts, scaling, and operations.

---

## 1) What is Splunk and Why it Matters

Splunk is a platform for collecting, indexing, searching, and analyzing machine data (logs/events/metrics).

## Why teams use Splunk
- Centralized observability and security visibility
- Powerful search language (SPL)
- Real-time alerting and dashboards
- Mature enterprise ecosystem (SIEM, SOAR, compliance integrations)

---

## 2) Splunk Core Architecture

Key components:

1. **Forwarders**  
   - Universal Forwarder (lightweight data shipper)
   - Heavy Forwarder (advanced parsing/routing use-cases)

2. **Indexer**  
   - receives data, parses/indexes, stores events

3. **Search Head**  
   - runs searches, dashboards, alerts, apps

4. **Deployment Server / Cluster Manager (advanced)**  
   - centralized forwarder config and indexer clustering controls

Data flow:
Source -> Forwarder -> Indexer -> Search Head

---

## 3) Splunk Editions (High-Level)

- Splunk Enterprise (self-managed)
- Splunk Cloud Platform (managed service)
- Splunk Enterprise Security (SIEM app layer on top)

This guide focuses on self-managed Splunk Enterprise fundamentals.

---

## 4) Install Splunk Enterprise (Linux)

## 4.1 Download package (example)
```bash
# Place downloaded .tgz or .deb/.rpm package on server
# Example tar install path:
sudo tar -xvzf splunk-<version>-Linux-x86_64.tgz -C /opt
sudo /opt/splunk/bin/splunk start --accept-license
```

Set admin credentials during first start prompt.

Enable at boot:
```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

Service check:
```bash
sudo /opt/splunk/bin/splunk status
```

Web UI:
- `http://<server-ip>:8000`

---

## 5) First Login and Basic Configuration

After login:
1. Create indexes (for clean data separation)
2. Add data inputs (files/syslog/forwarders/APIs)
3. Build saved searches + alerts
4. Create dashboards for teams

---

## 6) Splunk Data Onboarding Paths

Common sources:
- Linux syslog/auth logs
- Application logs (JSON/text)
- Nginx/Apache logs
- Cloud logs (AWS/Azure/GCP integrations)
- Kubernetes/container logs

Ingestion methods:
- monitor file path
- syslog TCP/UDP
- HTTP Event Collector (HEC)
- Universal Forwarder shipping

---

## 7) Universal Forwarder (UF) Basics

Install Splunk UF on source host, then configure output to indexer.

Common commands (host running UF):
```bash
/opt/splunkforwarder/bin/splunk start --accept-license
/opt/splunkforwarder/bin/splunk add forward-server <indexer-ip>:9997
/opt/splunkforwarder/bin/splunk add monitor /var/log
/opt/splunkforwarder/bin/splunk restart
```

Best practice:
- use deployment server for centralized forwarder config at scale.

---

## 8) Index Design and Data Governance

Create separate indexes by domain/team:
- `infra_logs`
- `app_logs`
- `security_logs`
- `audit_logs`

Why:
- better RBAC
- faster search targeting
- cleaner retention and cost control

Set retention by index importance/compliance needs.

---

## 9) SPL (Search Processing Language) Fundamentals

Base search:
```spl
index=app_logs error
```

Filter + fields:
```spl
index=app_logs sourcetype=nginx status>=500
| table _time host source status uri
```

Aggregation:
```spl
index=app_logs sourcetype=nginx
| stats count by status
| sort - count
```

Timechart:
```spl
index=app_logs sourcetype=nginx status>=500
| timechart span=5m count
```

Top values:
```spl
index=security_logs failed password
| top limit=10 src_ip
```

---

## 10) Field Extractions and Parsing

For structured logs (JSON), Splunk can auto-extract fields depending on sourcetype settings.
For custom logs:
- define field extractions (regex/DELIMS)
- use props.conf/transforms.conf for advanced parsing

Good parsing = better dashboards + alert quality.

---

## 11) Saved Searches, Alerts, and Reports

Example alert search:
```spl
index=security_logs "Failed password"
| stats count by src_ip
| where count > 20
```

Alert settings:
- schedule (e.g., every 5 min)
- trigger condition (if results > 0)
- throttle (avoid spam)
- action (email/webhook/notable event)

---

## 12) Dashboards and Visualizations

Useful dashboards:
- infrastructure error trends
- auth failures and brute-force indicators
- top failing APIs by endpoint
- latency and status code breakdowns
- deployment-time anomaly tracking

Use dashboard inputs (time range, host, app filters) for fast triage.

---

## 13) Splunk for Security (SIEM Basics)

Security detections often include:
- repeated failed logins
- impossible travel/auth anomalies
- suspicious process execution
- privilege escalation indicators
- lateral movement patterns

If using Splunk ES:
- correlation searches
- notable events
- risk-based alerting

---

## 14) HTTP Event Collector (HEC)

Enable HEC in Splunk and create token.

Send sample event:
```bash
curl -k https://<splunk-host>:8088/services/collector/event \
  -H "Authorization: Splunk <HEC_TOKEN>" \
  -d '{"event":"hello from app","sourcetype":"custom_app","index":"app_logs"}'
```

Great for app-native logging integration.

---

## 15) Roles, Users, and RBAC

Best practices:
- no shared admin accounts
- least privilege roles per team
- restrict index access per role
- audit user activity

Common roles:
- platform-admin
- secops-analyst
- dev-team-readonly
- app-observability-owner

---

## 16) Performance and Scaling Basics

At scale, separate tiers:
- indexer cluster
- search head cluster
- dedicated deployment server
- optional heavy forwarders

Performance tips:
- search specific indexes/sourcetypes
- avoid wildcard-heavy searches
- use summary indexing/accelerations where suitable
- tune retention and data model usage

---

## 17) Backup and DR Essentials

Back up:
- Splunk configs (`$SPLUNK_HOME/etc`)
- apps and knowledge objects
- index data volumes (per retention/DR policy)
- cluster manager metadata (if clustered)

Test restores regularly, not just backups.

---

## 18) Security Hardening Checklist

1. Enable TLS for web/data forwarding  
2. Restrict management ports and web UI access  
3. Rotate admin/HEC credentials  
4. Use SSO/SAML where possible  
5. Patch Splunk and OS regularly  
6. Enable audit logging and monitor admin actions  

---

## 19) Troubleshooting Matrix

## Splunk not starting
```bash
/opt/splunk/bin/splunk status
/opt/splunk/bin/splunk show splunkd-port
```
Check logs:
```bash
tail -f /opt/splunk/var/log/splunk/splunkd.log
```

## No data in index
- input misconfigured
- forwarder not connected
- wrong index/sourcetype
- time parsing issues

## Slow searches
- broad time range + no index constraint
- too many raw regex operations
- under-sized indexers/storage bottlenecks

## License warnings
- data ingest exceeds license entitlement
- review noisy sources and control ingestion

---

## 20) Common SPL Patterns (Daily Use)

Error count by host:
```spl
index=infra_logs (error OR failed)
| stats count by host
| sort -count
```

Auth failures trend:
```spl
index=security_logs "Failed password"
| timechart span=10m count
```

Top 10 noisy sources:
```spl
index=* earliest=-24h
| stats count by source
| sort - count
| head 10
```

---

## 21) Practice Tasks

1. Install Splunk Enterprise and login  
2. Create separate indexes for app/infra/security logs  
3. Onboard Linux auth log + Nginx access log  
4. Write SPL queries for 5xx spikes and failed logins  
5. Build dashboard with time-based error trends  
6. Configure alert for repeated failed auth attempts  
7. Ingest one app event via HEC token  

---

## 22) Daily Operations Cheat Sheet

```bash
# Service
sudo /opt/splunk/bin/splunk status
sudo /opt/splunk/bin/splunk restart

# Logs
tail -f /opt/splunk/var/log/splunk/splunkd.log
```

SPL quick start:
```spl
index=<index_name> earliest=-15m
| head 20
```

---

## 23) Splunkbase Apps and Add-ons

## Why
Most data sources (AWS, Nginx, Kubernetes, Windows) have pre-built Technology Add-ons (TAs) on Splunkbase that handle parsing/field extraction/dashboards out of the box — avoid rebuilding this from scratch.

## Install via UI
Apps → Find More Apps → search (e.g., "Splunk Add-on for Amazon Web Services") → Install

## Install via CLI (offline package)
```bash
/opt/splunk/bin/splunk install app /path/to/app.tar.gz -auth admin:<password>
sudo /opt/splunk/bin/splunk restart
```

Common starting set: Splunk Add-on for Unix and Linux, Splunk Add-on for AWS, Splunk App for Nginx, Splunk Add-on for Kubernetes.

---

## 24) CIM (Common Information Model)

## Why
CIM normalizes field names across different data sources (e.g., `src_ip` means the same thing whether the data came from a firewall or a web server) — required for Splunk ES correlation searches and for dashboards/apps that expect standardized fields.

## Key idea
Map your custom sourcetype's fields to CIM-compliant field names using field aliases or `eval` in props.conf:

```
[my_custom_app]
FIELDALIAS-cim = source_ip AS src_ip, user_name AS user
```

## Validate compliance
Install the "CIM Validation" or "Splunk Add-on Builder" app, or spot-check with:
```spl
| datamodel Authentication search
```
If your events don't appear in the relevant data model search, the CIM mapping needs work.

Getting this right up front saves significant rework once you adopt Enterprise Security or any CIM-dependent app.

---

## 25) Splunk Operator for Kubernetes

## Why
Run Splunk indexer/search-head clusters natively on Kubernetes instead of static VMs — useful when the rest of your platform is already K8s-native.

## Install operator
```bash
kubectl apply -f https://github.com/splunk/splunk-operator/releases/latest/download/splunk-operator-install.yaml
```

## Deploy a standalone instance
```yaml
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: s1
  namespace: splunk
spec:
  splunkVolumes: []
```
```bash
kubectl apply -f standalone.yaml -n splunk
kubectl get pods -n splunk
```

For production, use `ClusterMaster` + `IndexerCluster` + `SearchHeadCluster` CRDs together — see the operator docs for the full clustered topology.

---

## Final Notes

- Splunk is extremely powerful when data onboarding and field hygiene are done well.
- Start with clean index strategy, structured sourcetypes, and actionable alerts.
- Focus on signal quality over log volume to control cost and improve incident response.
- Lean on Splunkbase add-ons instead of hand-rolling parsers, map custom sourcetypes to CIM early, and consider the Splunk Operator if your platform is already Kubernetes-centric.
