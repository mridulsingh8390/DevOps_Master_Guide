# Prometheus + Grafana Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What are Prometheus and Grafana?  
2. Install Prometheus  
3. Configure Prometheus scrape jobs  
4. Install Node Exporter  
5. Prometheus service management and health checks  
6. PromQL essentials (with examples)  
7. Alerting basics (rules + Alertmanager integration pattern)  
8. Install Grafana  
9. Connect Grafana to Prometheus  
10. Build dashboards and panels  
11. Basic alerting in Grafana  
12. Troubleshooting matrix  
13. Practice lab tasks  
14. Daily ops cheat sheet

---

## 1) What are Prometheus and Grafana?

## Prometheus (What/Why)
- Time-series metrics database + scraper
- Pulls metrics from targets (`/metrics`)
- Supports powerful query language: PromQL
- Common for infra + app monitoring

## Grafana (What/Why)
- Visualization and dashboard platform
- Connects to Prometheus and other data sources
- Alerts, notifications, dashboard sharing

---

## 2) Install Prometheus

## 2.1 Create user and directories
```bash
sudo useradd --no-create-home --shell /bin/false prometheus || true
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

## 2.2 Download and install binaries
```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
tar -xvf prometheus-2.54.1.linux-amd64.tar.gz
cd prometheus-2.54.1.linux-amd64

sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

## 2.3 Create systemd service
```bash
cat <<'EOF' | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF
```

## 2.4 Start service
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

---

## 3) Configure Prometheus Scrape Jobs

## 3.1 Basic config (`/etc/prometheus/prometheus.yml`)
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```

## 3.2 Validate config
```bash
sudo promtool check config /etc/prometheus/prometheus.yml
```

## 3.3 Reload configuration
```bash
curl -X POST http://localhost:9090/-/reload
# or
sudo systemctl restart prometheus
```

---

## 4) Install Node Exporter

## 4.1 Create user and install binary
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter || true
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
```

## 4.2 Create service
```bash
cat <<'EOF' | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

## 4.3 Start service
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

Verify metrics endpoint:
```bash
curl -s http://localhost:9100/metrics | head
```

---

## 5) Prometheus Service Management and Health Checks

## Service/log commands
```bash
sudo systemctl status prometheus
sudo journalctl -u prometheus -f
```

## Health and readiness
```bash
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/-/ready
```

## Check active targets
```bash
curl -s http://localhost:9090/api/v1/targets | jq
```

Open UI:
- `http://<server-ip>:9090`

---

## 6) PromQL Essentials

## 6.1 Basic target availability
```promql
up
up{job="node"}
```

## 6.2 CPU usage (host-level)
```promql
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

## 6.3 Memory usage percentage
```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

## 6.4 Disk usage percentage
```promql
(1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100
```

## 6.5 Network receive/transmit rate
```promql
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

## 6.6 Load average
```promql
node_load1
```

## Notes
- `rate()` for counters
- `irate()` for short-window fast changes
- always filter noisy labels where needed

---

## 7) Alerting Basics

## 7.1 Create alert rules file (`/etc/prometheus/alert_rules.yml`)
```yaml
groups:
- name: host-alerts
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance down"
      description: "Target {{ $labels.instance }} is down."

  - alert: HighCPU
    expr: (100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU"
      description: "CPU > 85% on {{ $labels.instance }}"
```

## 7.2 Reference rule file in prometheus.yml
```yaml
rule_files:
  - /etc/prometheus/alert_rules.yml
```

## 7.3 Validate and reload
```bash
sudo promtool check rules /etc/prometheus/alert_rules.yml
sudo promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

## Alertmanager integration pattern
In `prometheus.yml`:
```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["localhost:9093"]
```

(Install Alertmanager separately if required.)

---

## 8) Install Grafana

## 8.1 Install package
```bash
cd /tmp
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo apt install -y adduser libfontconfig1 musl
sudo dpkg -i grafana_11.1.4_amd64.deb
```

## 8.2 Start Grafana
```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

## 8.3 Reset admin password (if needed)
```bash
sudo grafana-cli admin reset-admin-password 'StrongPassword@123'
```

Open UI:
- `http://<server-ip>:3000`
- Default login: `admin/admin` (first login prompts password change)

---

## 9) Connect Grafana to Prometheus

1. Grafana UI → **Connections** → **Data sources**  
2. Add **Prometheus**  
3. URL: `http://localhost:9090`  
4. Click **Save & test**

If Grafana and Prometheus are on different hosts, use Prometheus host IP.

---

## 10) Build Dashboards and Panels

## 10.1 Create dashboard
- Dashboard → New → New Dashboard → Add Visualization

## 10.2 Example panels
- **CPU %** query:
```promql
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
- **Memory %** query:
```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```
- **Disk %** query:
```promql
(1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100
```

## 10.3 Use variables
Create dashboard variable:
- Name: `instance`
- Query:
```promql
label_values(node_uname_info, instance)
```
Then filter panels with `{instance="$instance"}`.

---

## 11) Basic Alerting in Grafana

## Example alert rule
1. Open panel → Alert → Create alert rule
2. Condition: CPU query > 85 for 5m
3. Set contact point (email/webhook/slack)
4. Save rule

Best practice:
- Avoid noisy alerts
- Use proper `for` duration
- Add clear summary/description

---

## 12) Troubleshooting Matrix

## 12.1 Prometheus not starting
```bash
sudo systemctl status prometheus
sudo journalctl -u prometheus -n 200 --no-pager
sudo promtool check config /etc/prometheus/prometheus.yml
```

## 12.2 Target DOWN
- Check endpoint reachable
- Check firewall/security group
- Confirm scrape target and port

```bash
curl -s http://target-ip:9100/metrics | head
```

## 12.3 Grafana cannot connect to Prometheus
- Wrong URL (`localhost` confusion)
- Service down
- Network/firewall issue

```bash
curl -s http://localhost:9090/-/healthy
sudo systemctl status grafana-server prometheus
```

## 12.4 No data in dashboard
- Wrong PromQL
- Wrong time range
- Metric name mismatch

Use Prometheus UI first to validate query.

---

## 13) Practice Lab Tasks

1. Install Prometheus and Node Exporter  
2. Add scrape config and validate targets  
3. Create 3 PromQL queries (CPU, memory, disk)  
4. Create alert rules and verify they appear in Prometheus  
5. Install Grafana and connect Prometheus datasource  
6. Build dashboard with host metrics  
7. Create one CPU alert in Grafana  

---

## 14) Daily Ops Cheat Sheet

```bash
# Service status
sudo systemctl status prometheus node_exporter grafana-server

# Logs
sudo journalctl -u prometheus -f
sudo journalctl -u node_exporter -f
sudo journalctl -u grafana-server -f

# Health checks
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets | jq
curl -I http://localhost:3000/login

# Config validation
sudo promtool check config /etc/prometheus/prometheus.yml
sudo promtool check rules /etc/prometheus/alert_rules.yml
```

---

## Final Notes

- Prometheus is your source of truth for metrics.
- Grafana is your lens (visualization + alert routing).
- Start with essential infra metrics, then add app/business metrics.
- Keep alerting actionable, not noisy.