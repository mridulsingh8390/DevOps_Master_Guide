# Fluent Bit Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

---

## 1) What is Fluent Bit?

Fluent Bit is a lightweight log processor/forwarder designed for high-performance log collection.

### Why use it?
- Low CPU/memory footprint
- Great for edge nodes and Kubernetes DaemonSet logging
- Supports many inputs/filters/outputs (Loki, Elasticsearch, Kafka, etc.)

---

## 2) Install Fluent Bit (Ubuntu)

```bash
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
sudo apt update
sudo apt install -y fluent-bit
```

Verify:
```bash
fluent-bit --version
sudo systemctl status fluent-bit
```

---

## 3) Core concepts

- **Input**: where logs come from (`tail`, `systemd`, `forward`)
- **Filter**: modify/enrich logs (`grep`, `record_modifier`, `kubernetes`)
- **Output**: where logs go (`loki`, `stdout`, `es`)
- **Parser**: parse raw text/json

---

## 4) Example config (tail file -> stdout)

`/etc/fluent-bit/fluent-bit.conf`
```ini
[SERVICE]
    Flush        1
    Daemon       Off
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name   tail
    Path   /var/log/syslog
    Tag    syslog

[FILTER]
    Name   record_modifier
    Match  *
    Record host ${HOSTNAME}

[OUTPUT]
    Name   stdout
    Match  *
```

Restart:
```bash
sudo systemctl restart fluent-bit
sudo journalctl -u fluent-bit -f
```

---

## 5) Send logs to Loki

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

## 6) Kubernetes DaemonSet usage (concept)

- Deploy Fluent Bit as DaemonSet
- Input: `/var/log/containers/*.log`
- Filter: `kubernetes` metadata enrichment
- Output: Loki/Elasticsearch

---

## 7) Troubleshooting

```bash
sudo systemctl status fluent-bit
sudo journalctl -u fluent-bit -n 200 --no-pager
fluent-bit -c /etc/fluent-bit/fluent-bit.conf
```

Common issues:
- Wrong file path in `tail`
- Permission denied on log files
- Output backend unreachable

---

## 8) Practice tasks

1. Tail syslog and print stdout  
2. Add host metadata via filter  
3. Send logs to Loki  
4. Simulate log line and verify in destination