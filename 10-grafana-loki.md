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

## 9) Best practices

1. Keep labels low-cardinality (`job`, `host`, `env`)
2. Avoid per-request labels
3. Use retention policy
4. Separate parsing/transforms in shipper layer
5. Monitor Loki ingestion and query latency

---

## 10) Practice tasks

1. Install Loki and verify readiness  
2. Connect Grafana Loki datasource  
3. Ship syslog via Fluent Bit  
4. Query logs with LogQL (`|=`, `|~`, `json`)  
5. Build Grafana logs panel and alert on `"error"` count spike