# Grafana Loki Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

---

## 1) What is Loki?

Loki is a log aggregation system optimized for cost-efficient storage by indexing labels, not full log content.

### Why use Loki?
- Lower storage/index cost than full-text indexing systems
- Tight integration with Grafana
- Great with Fluent Bit / Promtail

---

## 2) Install Loki (single binary mode)

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.1.0/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64
sudo mv loki-linux-amd64 /usr/local/bin/loki
```

Create user and dirs:
```bash
sudo useradd --no-create-home --shell /bin/false loki || true
sudo mkdir -p /etc/loki /var/lib/loki
sudo chown -R loki:loki /etc/loki /var/lib/loki
```

---

## 3) Loki config

`/etc/loki/config.yml`
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  allow_structured_metadata: true
```

---

## 4) systemd service

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/loki.service
[Unit]
Description=Loki
After=network.target

[Service]
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

Start Loki:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now loki
sudo systemctl status loki
```

Verify:
```bash
curl -s http://localhost:3100/ready
```

---

## 5) Connect Loki to Grafana

In Grafana:
1. Connections → Data sources
2. Add data source: **Loki**
3. URL: `http://localhost:3100`
4. Save & test

---

## 6) Ingest logs (via Fluent Bit/Fluentd)

Fluent Bit output snippet:
```ini
[OUTPUT]
    Name        loki
    Match       *
    Host        127.0.0.1
    Port        3100
    Labels      job=fluentbit,host=${HOSTNAME}
    Line_Format json
```

---

## 7) LogQL basics

### Show streams
```logql
{job="fluentbit"}
```

### Filter by keyword
```logql
{job="fluentbit"} |= "error"
```

### Regex filter
```logql
{job="fluentbit"} |~ "timeout|failed"
```

### Parse json field
```logql
{job="fluentbit"} | json | level="error"
```

### Count logs over time
```logql
count_over_time({job="fluentbit"}[5m])
```

---

## 8) Troubleshooting

```bash
sudo systemctl status loki
sudo journalctl -u loki -f
curl -s http://localhost:3100/ready
```

Common issues:
- Wrong Loki URL in shipper
- Missing labels causing hard-to-query logs
- Disk permission issues in `/var/lib/loki`
- High cardinality labels (avoid dynamic labels like request_id)

---

## 9) Promtail (Loki's Native Log Shipper)

## Why
Fluent Bit/Fluentd work fine with Loki, but Promtail is Loki's purpose-built agent with tight label/pipeline integration.

## Install
```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.1.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
```

## Config (`/etc/promtail/config.yml`)
```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: ${HOSTNAME}
          __path__: /var/log/syslog
```

Run:
```bash
promtail -config.file=/etc/promtail/config.yml
```

---

## 10) Retention and Compactor Config

## Why
Without retention, Loki keeps chunks forever — disk fills up. The compactor handles deletion/compaction for the filesystem store.

Add to `/etc/loki/config.yml`:
```yaml
compactor:
  working_directory: /var/lib/loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h

limits_config:
  retention_period: 720h   # 30 days
```

Restart Loki and verify with:
```bash
curl -s http://localhost:3100/config | grep -A3 retention
```

---

## 11) Best practices

1. Keep labels low-cardinality (`job`, `host`, `env`)
2. Avoid per-request labels
3. Use retention policy
4. Separate parsing/transforms in shipper layer
5. Monitor Loki ingestion and query latency

---

## 12) Practice tasks

1. Install Loki and verify readiness  
2. Connect Grafana Loki datasource  
3. Ship syslog via Fluent Bit  
4. Query logs with LogQL (`|=`, `|~`, `json`)  
5. Build Grafana logs panel and alert on `"error"` count spike
