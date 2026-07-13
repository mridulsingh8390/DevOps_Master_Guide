# Graylog Complete Guide (Beginner to Advanced)
## Centralized Log Management for DevOps (Local/On-Prem)

> Practical guide for installing Graylog stack, onboarding logs, parsing, alerting, and operating at scale.

---

## 1) What is Graylog and Why it Matters

Graylog is a centralized log management and analysis platform.

## Why DevOps teams use it
- Central place for infrastructure/app logs
- Fast troubleshooting and incident investigation
- Alerts from log patterns
- Pipelines for parsing/enrichment
- Better operational visibility across services

---

## 2) Graylog Architecture Overview

Graylog stack typically includes:

1. **Graylog Server**  
   - receives/processes logs, handles streams/pipelines/alerts, provides API/UI

2. **OpenSearch (or Elasticsearch in older setups)**  
   - stores/searches indexed log data

3. **MongoDB**  
   - stores Graylog metadata/config (users, dashboards, streams, etc.)

Data flow:
Shippers/agents/syslog -> Graylog Inputs -> Processing (extractors/pipelines) -> OpenSearch -> Search/Dashboards/Alerts

---

## 3) Deployment Options

- Single-node VM (lab/small env) ✅
- Multi-node Graylog + OpenSearch cluster (production scale)
- Containerized deployment (Docker/K8s)
- Managed/open marketplace variants (if available)

This guide uses Ubuntu single-node style for clarity.

---

## 4) Prerequisites (Ubuntu 24.04)

- 4+ CPU, 8+ GB RAM minimum (more for realistic loads)
- Java runtime (required by Graylog/OpenSearch components)
- Proper hostname + time sync
- Firewall ports opened as needed:
  - Graylog web/API (commonly 9000)
  - Syslog/GELF/Beats inputs (custom)
  - OpenSearch (9200 internal)
  - MongoDB (27017 internal)

---

## 5) Install MongoDB and OpenSearch

> In production, follow version compatibility matrix from Graylog docs.  
> Graylog versions are strict about supported OpenSearch/MongoDB versions.

### 5.1 Install MongoDB (example via official repo)
```bash
sudo apt update
sudo apt install -y gnupg curl
# Add MongoDB repo per your chosen supported version
# Install and start mongodb
sudo systemctl enable --now mongod
sudo systemctl status mongod
```

### 5.2 Install OpenSearch
```bash
# Add OpenSearch repo and install package (version compatible with Graylog release)
sudo apt update
sudo apt install -y opensearch
sudo systemctl enable --now opensearch
sudo systemctl status opensearch
```

Set required kernel setting:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

## 6) Install Graylog Server

```bash
# Add Graylog repository for your target version
sudo apt update
sudo apt install -y graylog-server
```

---

## 7) Configure Graylog Server

Main config:
`/etc/graylog/server/server.conf`

Important keys:
```properties
password_secret = <long-random-secret>
root_password_sha2 = <sha256-of-admin-password>
http_bind_address = 0.0.0.0:9000
http_external_uri = http://<SERVER_IP>:9000/
mongodb_uri = mongodb://127.0.0.1:27017/graylog
opensearch_hosts = http://127.0.0.1:9200
```

Generate values:

Random secret:
```bash
pwgen -N 1 -s 96
```

Admin password SHA256:
```bash
echo -n 'YourStrongAdminPassword' | sha256sum
```

Start Graylog:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now graylog-server
sudo systemctl status graylog-server
```

UI:
- `http://<SERVER_IP>:9000`

---

## 8) First Login and Initial Setup

1. Open Graylog UI
2. Login with admin username and configured password
3. Create initial input(s)
4. Start receiving logs
5. Build streams and dashboards

---

## 9) Inputs: How Logs Enter Graylog

Common input types:
- Syslog UDP/TCP
- GELF UDP/TCP/HTTP
- Beats input
- Raw/Plaintext TCP/UDP

### Example: Create Syslog UDP input
- System -> Inputs
- Select “Syslog UDP”
- Bind address `0.0.0.0`
- Port `1514` (or your chosen port)
- Start input

Send test log:
```bash
logger -n <GRAYLOG_IP> -P 1514 "graylog test message"
```

---

## 10) Integrating Log Shippers

You can forward logs via:
- rsyslog/syslog-ng
- Filebeat/Winlogbeat
- Fluent Bit/Fluentd (GELF/syslog output)
- Custom app GELF libraries

For container/K8s setups, Fluent Bit is common.

---

## 11) Streams (Logical Routing)

Streams let you route/select messages based on rules, e.g.:
- `env=prod`
- `source=nginx`
- `level=error`

Why streams matter:
- clean separation by app/team/env
- targeted alerts and dashboards
- better access control boundaries

---

## 12) Parsing and Enrichment

Graylog supports:
- extractors (simple parsing at input level)
- pipeline rules (advanced transform/enrichment)

Typical transformations:
- parse JSON
- extract IP/user/request-id
- normalize log levels
- geoip enrichment

---

## 13) Pipelines and Rules (Core Power)

Pipeline example use-cases:
- route security logs to `security` stream
- tag failed login attempts
- drop noisy health-check logs
- convert fields to right data types

Keep rules version-controlled (export or API-driven backup workflow).

---

## 14) Search and Investigation Workflow

Typical troubleshooting flow:
1. Search by service/hostname
2. Narrow time range
3. Filter on error keywords
4. Pivot by request_id/session_id
5. Save search/query for reuse

Use relative time quick ranges during incidents (last 5m, 15m, 1h).

---

## 15) Dashboards and Alerts

Build dashboards with:
- error rate over time
- top noisy hosts/services
- auth failure trends
- latency/error keyword distributions

Alert/Event definitions:
- trigger when count of matching logs crosses threshold
- notify via email/webhook/integrations

Example alert ideas:
- repeated auth failures
- application “panic/fatal” patterns
- sudden 5xx spikes in ingress logs

---

## 16) Index Management and Retention

Graylog uses index sets for retention strategy.

Configure:
- index rotation (size/time/message count based)
- retention (delete/close/archive old indices)
- shard/replica strategy

Why:
- control disk growth
- maintain search performance

---

## 17) Security Best Practices

1. Put Graylog behind HTTPS reverse proxy  
2. Restrict admin UI/API access by IP/VPN  
3. Enforce RBAC users/roles (no shared admin accounts)  
4. Rotate credentials and integration secrets  
5. Keep Graylog/OpenSearch/MongoDB patched  
6. Avoid exposing backend ports publicly  

---

## 18) Backup and Recovery

Back up:
- MongoDB (Graylog metadata/config/users)
- OpenSearch snapshots (log data indices)
- Graylog config files
- pipeline/stream/dashboard exports if applicable

Test restore regularly in non-prod.

---

## 19) Performance and Scaling Tips

- Separate nodes for Graylog/OpenSearch at scale
- Increase JVM heap carefully (OpenSearch/Graylog)
- Tune index rotation and retention
- Reduce noisy/unnecessary logs at source
- Use streams/pipelines efficiently (avoid expensive parsing on all messages)

---

## 20) Troubleshooting Matrix

## Graylog server not starting
```bash
sudo systemctl status graylog-server
sudo journalctl -u graylog-server -n 200 --no-pager
```

## No logs arriving
- input not started
- wrong port/protocol
- firewall drop
- sender misconfigured

## Search slow
- insufficient resources
- too many shards
- unbounded queries and large time windows

## OpenSearch unhealthy
```bash
curl -s http://127.0.0.1:9200/_cluster/health?pretty
```

## Time mismatch issues
- ensure NTP/time sync on all senders + Graylog nodes

---

## 21) Practice Tasks

1. Deploy single-node Graylog stack  
2. Create Syslog input and ingest test logs  
3. Create stream for `ERROR` logs  
4. Build parser/extractor for app JSON logs  
5. Create dashboard for top errors by service  
6. Add alert for repeated failed login pattern  
7. Configure index retention for 7 days lab data  

---

## 22) Daily Operations Cheat Sheet

```bash
# Services
sudo systemctl status graylog-server mongod opensearch

# Logs
sudo journalctl -u graylog-server -f

# OpenSearch health
curl -s http://127.0.0.1:9200/_cluster/health?pretty
```

Graylog UI daily checks:
- Inputs running
- Journal size/backlog
- Failed processing streams/pipeline errors
- Alert backlog and notification health

---

## Final Notes

- Graylog is excellent for centralized log operations and incident triage.
- Focus on good stream design + parsing quality + retention policy early.
- The biggest win comes from reducing noise and creating actionable alerts.