# Wazuh Complete Guide (Beginner to Advanced)
## Open-Source SIEM + XDR for DevOps/SecOps (Local/On-Prem)

> Practical guide for deployment, agent onboarding, rule tuning, alerting, integrations, and operations.

---

## 1) What is Wazuh and Why it Matters

Wazuh is an open-source security platform for:
- SIEM (log collection + correlation)
- XDR-like detection and response
- File integrity monitoring (FIM)
- Vulnerability detection
- Security configuration assessment (SCA)
- Incident investigation

## Why DevOps teams use it
- Centralized security visibility across servers/endpoints
- Compliance and audit support
- Threat detection from logs + agent telemetry
- Good self-hosted option for on-prem environments

---

## 2) Wazuh Architecture Overview

Main components:
1. **Wazuh Server/Manager**  
   - processes agent data, runs rules/decoders, generates alerts

2. **Wazuh Indexer**  
   - stores indexed security events (OpenSearch-based in modern stack)

3. **Wazuh Dashboard**  
   - web UI for alerts, dashboards, threat hunting

4. **Wazuh Agents**  
   - installed on endpoints (Linux/Windows/macOS) to send telemetry

Data flow:
Agent -> Manager -> Indexer -> Dashboard

---

## 3) Deployment Options

- Single-node (lab/small setups) ✅ easiest start
- Distributed cluster (production scale/high availability)
- Cloud VM-based deployment
- Docker-based testing setups

For this guide: single-node install on Ubuntu 24.04.

---

## 4) Prerequisites

- Ubuntu 24.04 (or supported Linux distro)
- 4+ CPU, 8+ GB RAM minimum for practical lab
- Open ports as required (dashboard/indexer/agent comms)
- Root/sudo access

Set hostname and time sync properly before install.

---

## 5) Install Wazuh All-in-One (Recommended Quickstart)

Use official assisted install script approach:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.9/config.yml
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

> Version paths may change over time; keep script/version aligned with official Wazuh docs for your target version.

After installation, script outputs credentials and access details.

---

## 6) Verify Services

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

Check listening ports:
```bash
sudo ss -tulnp | egrep "1514|1515|55000|5601|9200"
```

Access Dashboard:
- `https://<wazuh-server-ip>`
- login with generated admin credentials

---

## 7) Install Wazuh Agent (Linux)

On target host:

```bash
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_<VERSION>-1_amd64.deb
sudo WAZUH_MANAGER='<MANAGER_IP>' dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
```

Set agent name/group (optional advanced):
- edit `/var/ossec/etc/ossec.conf`
- restart agent after changes

---

## 8) Registering Agents

Usually automatic with deployment methods above, but manual management commands on manager include:

```bash
sudo /var/ossec/bin/manage_agents
```

Common actions:
- add agent
- list agent IDs
- remove old agent entries
- extract keys for manual auth workflows

---

## 9) Wazuh Agent Configuration Basics

Main file:
`/var/ossec/etc/ossec.conf`

Common monitored areas:
- log files (`<localfile>`)
- syscheck/FIM directories
- rootcheck settings
- SCA policies

After edits:
```bash
sudo systemctl restart wazuh-agent
```

---

## 10) Log Collection Examples

## Monitor auth log (Linux)
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

## Monitor custom app log
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/myapp/security.json</location>
</localfile>
```

---

## 11) File Integrity Monitoring (FIM)

In agent config:
```xml
<syscheck>
  <disabled>no</disabled>
  <directories check_all="yes" realtime="yes">/etc,/usr/bin</directories>
</syscheck>
```

Use-case:
- detect unauthorized file changes
- track tampering on critical paths

---

## 12) Vulnerability Detection + SCA

Wazuh can correlate package inventory/CVE feeds and run compliance checks.

Manager-side modules (depending on version/config):
- vulnerability detector
- SCA benchmark policies (CIS-like checks)

In dashboard:
- Security events -> Vulnerabilities
- Security configuration assessment panels

---

## 13) Rules and Decoders (Detection Logic)

- **Decoders** parse raw logs into fields
- **Rules** determine alert generation/severity

Custom files:
- `/var/ossec/etc/decoders/local_decoder.xml`
- `/var/ossec/etc/rules/local_rules.xml`

Example custom rule skeleton:
```xml
<group name="custom,authentication,">
  <rule id="100100" level="10">
    <if_group>authentication_failed</if_group>
    <match>Failed password</match>
    <description>Custom: Failed SSH password attempt</description>
  </rule>
</group>
```

Validate/restart manager after changes.

---

## 14) Active Response (Automated Actions)

Active response can run scripts/commands (e.g., block IP) when specific alerts trigger.

Use carefully:
- start in monitor-only mode
- avoid aggressive auto-block without tuning
- whitelist trusted systems to prevent accidental lockouts

---

## 15) Integrations

Common integrations:
- Slack/Email alert notifications
- TheHive, MISP, VirusTotal (depending on workflow)
- Ticketing/webhooks/SOAR systems
- Syslog forwarding to external platforms

Plan integration flows around incident response process.

---

## 16) Dashboard Usage Workflow

Daily SOC/DevSecOps flow:
1. Review high-severity alerts
2. Filter by agent/group/rule ID
3. Investigate timeline and related events
4. Pivot on source IP/user/process/file hash
5. Classify true positive vs false positive
6. Tune rules/noise suppression

---

## 17) Performance and Scaling Notes

Single-node is fine for labs/small setups; production may need:
- index lifecycle and retention tuning
- separate manager/indexer nodes
- shard/replica tuning
- agent grouping and policy segmentation

Control data volume:
- avoid ingesting noisy logs blindly
- tune rule levels and ignored events

---

## 18) Backup and Recovery Essentials

Back up:
- Wazuh manager config (`/var/ossec/etc`)
- custom rules/decoders
- certificates/keys
- indexer data snapshots
- dashboard configs

Test restore procedures periodically.

---

## 19) Security Hardening Best Practices

1. Restrict dashboard exposure (VPN/reverse proxy/IP allowlist)  
2. Use TLS everywhere (agent-manager, dashboard access)  
3. Rotate credentials and API keys  
4. Apply least privilege to admin roles  
5. Keep Wazuh and OS patched regularly  
6. Enable audit logging and central time sync  

---

## 20) Troubleshooting Matrix

## Agent not showing up
- manager IP incorrect
- firewall blocks ports
- enrollment/auth mismatch

Check:
```bash
sudo systemctl status wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

## No alerts in dashboard
- ingestion/indexer issue
- rule level filters
- time sync mismatch

Check manager/indexer logs and dashboard time window.

## High CPU on manager/indexer
- too many noisy logs
- heavy rules/regex
- insufficient resources

Action:
- tune ingestion
- scale components
- adjust retention and index policies

## Rule changes not applied
- invalid XML syntax
- service not restarted

---

## 21) Practice Tasks

1. Deploy single-node Wazuh stack  
2. Add 2 Linux agents  
3. Monitor auth logs and generate failed-login events  
4. Enable FIM for `/etc` and validate alerts on file change  
5. Create one custom local rule and verify it triggers  
6. Build dashboard filter for critical auth alerts  
7. Configure basic Slack/email alerting  

---

## 22) Daily Operations Cheat Sheet

```bash
# Services
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard wazuh-agent

# Logs
sudo tail -f /var/ossec/logs/ossec.log

# Agent management
sudo /var/ossec/bin/manage_agents

# API health (example endpoint may vary by version/setup)
curl -k -u <user>:<pass> https://localhost:55000/security/user/authenticate
```

---

## 23) Docker-Based Quickstart (Fast Lab Setup)

## Why
Faster than the full install script for local testing/evaluation — spins up manager + indexer + dashboard via Compose.

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

Access dashboard:
- `https://localhost` (default admin credentials are in the repo's `.env` or docs for the pinned version)

Tear down:
```bash
docker compose down -v
```

Good for evaluation/dev only — use the full install for anything persistent or production-facing.

---

## 24) Wazuh API Usage (Beyond Health Checks)

## Why
Automate agent management, rule queries, and cluster status from scripts/CI instead of clicking through the dashboard.

## Authenticate and get a token
```bash
TOKEN=$(curl -s -u wazuh:<api_password> -k -X POST \
  "https://localhost:55000/security/user/authenticate" | jq -r '.data.token')
```

## List agents
```bash
curl -s -k -X GET "https://localhost:55000/agents" \
  -H "Authorization: Bearer $TOKEN" | jq
```

## Restart an agent remotely
```bash
curl -s -k -X PUT "https://localhost:55000/agents/restart" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"agents_list":["001","002"]}' -H "Content-Type: application/json"
```

## Query active rules
```bash
curl -s -k -X GET "https://localhost:55000/rules?limit=10" \
  -H "Authorization: Bearer $TOKEN" | jq
```

---

## 25) Cluster Mode (Multi-Manager Setup)

## Why
Single manager is a single point of failure and a ceiling on agent capacity — cluster mode distributes load across multiple manager nodes.

## Key config (`/var/ossec/etc/ossec.conf` on each node)
```xml
<cluster>
  <name>wazuh-cluster</name>
  <node_name>node01</node_name>
  <node_type>master</node_type>
  <key>{cluster-key}</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
  <nodes>
    <node>10.0.0.10</node>
  </nodes>
  <disabled>no</disabled>
</cluster>
```
Worker nodes use `<node_type>worker</node_type>` and point to the master's IP in `<nodes>`.

## Verify cluster health
```bash
sudo /var/ossec/bin/cluster_control -l
sudo /var/ossec/bin/cluster_control -i
```

---

## Final Notes

- Wazuh is a strong open-source foundation for SIEM/XDR-style visibility.
- Start with clean onboarding + baseline detections before enabling aggressive responses.
- Success depends on continuous rule tuning, noise reduction, and incident playbook integration.
- Use the Docker Compose path for quick evaluation, the API for automation, and cluster mode once a single manager becomes a bottleneck.
