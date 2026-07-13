# Fluentd Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

---

## 1) What is Fluentd?

Fluentd is a flexible log collector/aggregator (heavier than Fluent Bit, more plugin ecosystem).

### Why use it?
- Rich plugin support
- Powerful routing and transformations
- Centralized log aggregation layer

---

## 2) Install Fluentd (td-agent alternative possible)

```bash
sudo apt update
sudo apt install -y ruby ruby-dev build-essential
sudo gem install fluentd --no-document
```

Run test:
```bash
fluentd --version
```

---

## 3) Core concepts

- **source**: input plugin
- **filter**: transform/enrich
- **match**: route to outputs
- **buffer**: batch/retry behavior

---

## 4) Minimal config (forward syslog file to stdout)

`fluent.conf`
```conf
<source>
  @type tail
  path /var/log/syslog
  pos_file /tmp/syslog.pos
  tag syslog
  <parse>
    @type none
  </parse>
</source>

<filter syslog>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match **>
  @type stdout
</match>
```

Run:
```bash
fluentd -c fluent.conf -vv
```

---

## 5) Output to Loki (plugin-based)

Install plugin:
```bash
fluent-gem install fluent-plugin-grafana-loki
```

Config:
```conf
<match **>
  @type loki
  url "http://127.0.0.1:3100"
  extra_labels {"job":"fluentd","host":"${hostname}"}
</match>
```

---

## 6) Buffering and retry (important)

```conf
<match **>
  @type loki
  url "http://127.0.0.1:3100"

  <buffer>
    @type file
    path /tmp/fluent-buffer
    flush_interval 5s
    retry_forever true
  </buffer>
</match>
```

Why:
- Prevent log loss during backend outage
- Smooth burst traffic

---

## 7) Troubleshooting

```bash
fluentd --dry-run -c fluent.conf
fluentd -c fluent.conf -vv
```

Common issues:
- Missing plugin
- Parser mismatch
- Permission on tailed files
- Buffer path not writable

---

## 8) Practice tasks

1. Tail syslog and print to stdout  
2. Add record transformer fields  
3. Enable file buffer  
4. Send logs to Loki and verify in Grafana