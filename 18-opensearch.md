# OpenSearch + OpenSearch Dashboards Complete Guide
## Log Analytics / Search Platform for DevOps (Local/On-Prem)

> This file is detailed and practical.
> Covers install (single-node), index basics, ingestion patterns, dashboards, security notes, troubleshooting.

---

## 1) What is OpenSearch and Why it Matters

OpenSearch is a distributed search and analytics engine (Elasticsearch-compatible lineage), commonly used for logs, metrics-like events, and full-text search.

OpenSearch Dashboards is the visualization/UI layer.

## Why use it?
- Powerful full-text log search
- Rich aggregations and dashboards
- Suitable for SIEM and observability-style use cases
- Works with Fluent Bit/Fluentd/Logstash-like pipelines

---

## 2) Deployment Options

1. **Docker Compose (fast lab setup)** ✅ recommended for local learning  
2. Package/binary install (heavier, more host tuning)  
3. Kubernetes operator/Helm (production-like environments)

This guide uses Docker Compose single-node for simplicity.

---

## 3) Prerequisites

- Docker + Docker Compose plugin installed
- Minimum recommended for lab: 4 CPU, 8 GB RAM

Set vm.max_map_count (Linux host):
```bash
sudo sysctl -w vm.max_map_count=262144
```

Persist:
```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

## 4) Docker Compose Setup (Single Node)

Create `docker-compose.yml`:

```yaml
services:
  opensearch:
    image: opensearchproject/opensearch:2.15.0
    container_name: opensearch
    environment:
      - discovery.type=single-node
      - plugins.security.disabled=true
      - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    ports:
      - "9200:9200"
      - "9600:9600"

  dashboards:
    image: opensearchproject/opensearch-dashboards:2.15.0
    container_name: opensearch-dashboards
    environment:
      - OPENSEARCH_HOSTS=["http://opensearch:9200"]
      - DISABLE_SECURITY_DASHBOARDS_PLUGIN=true
    ports:
      - "5601:5601"
    depends_on:
      - opensearch

volumes:
  opensearch-data:
```

Start:
```bash
docker compose up -d
docker compose ps
```

Verify:
```bash
curl -s http://localhost:9200
curl -s http://localhost:9200/_cluster/health?pretty
```

Dashboards URL:
- http://localhost:5601

---

## 5) Core OpenSearch Concepts

- **Index**: logical collection of documents
- **Document**: JSON record
- **Shard/Replica**: distribution and redundancy units
- **Mapping**: schema/type definition
- **Query DSL**: JSON-based query language

---

## 6) Basic Index Operations

## Create index
```bash
curl -X PUT "localhost:9200/app-logs"
```

## List indices
```bash
curl -s "localhost:9200/_cat/indices?v"
```

## Insert document
```bash
curl -X POST "localhost:9200/app-logs/_doc" \
  -H 'Content-Type: application/json' \
  -d '{"level":"INFO","service":"api","message":"app started","env":"dev","ts":"2026-07-13T12:00:00Z"}'
```

## Search all
```bash
curl -s "localhost:9200/app-logs/_search?pretty"
```

## Term query
```bash
curl -s -X GET "localhost:9200/app-logs/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"term":{"service.keyword":"api"}}}'
```

---

## 7) Mapping and Templates

## Create index with explicit mapping
```bash
curl -X PUT "localhost:9200/app-logs-v2" \
  -H 'Content-Type: application/json' \
  -d '{
    "mappings": {
      "properties": {
        "ts":      { "type": "date" },
        "level":   { "type": "keyword" },
        "service": { "type": "keyword" },
        "message": { "type": "text" },
        "env":     { "type": "keyword" }
      }
    }
  }'
```

Why explicit mapping?
- Prevent accidental wrong field types
- Better query performance/accuracy

---

## 8) Ingestion from Fluent Bit / Fluentd

## Fluent Bit output snippet
```ini
[OUTPUT]
    Name            es
    Match           *
    Host            127.0.0.1
    Port            9200
    Index           fluentbit-logs
    Logstash_Format On
    Retry_Limit     False
```

## Fluentd output snippet
```conf
<match **>
  @type opensearch
  host 127.0.0.1
  port 9200
  logstash_format true
  logstash_prefix fluentd
</match>
```

---

## 9) OpenSearch Dashboards Basics

1. Open http://localhost:5601  
2. Create **Data View** (e.g., `app-logs*`)  
3. Choose timestamp field (`ts` or `@timestamp`)  
4. Use Discover for live search  
5. Build visualizations and dashboards

Common dashboard visuals:
- Logs over time
- Top services by error count
- Error distribution by host/env
- Slow endpoint terms

---

## 10) Useful APIs for Operations

## Cluster health
```bash
curl -s "localhost:9200/_cluster/health?pretty"
```

## Node stats
```bash
curl -s "localhost:9200/_nodes/stats?pretty"
```

## Cat APIs
```bash
curl -s "localhost:9200/_cat/nodes?v"
curl -s "localhost:9200/_cat/indices?v"
curl -s "localhost:9200/_cat/shards?v"
```

---

## 11) Retention and Lifecycle (Concept)

Use Index State Management (ISM) policies to:
- rollover indices
- delete old indices after X days
- optimize storage over time

Example goal:
- keep hot logs 7 days
- warm logs 30 days
- delete after 45 days

---

## 12) Security Notes (Important)

For production:
1. Enable OpenSearch security plugin (TLS + auth)
2. Don’t expose 9200 publicly
3. Put behind reverse proxy/VPN
4. Use role-based permissions
5. Secure dashboards access

> In this lab file, security plugin is disabled for simplicity.

---

## 13) Backup and Restore (Snapshot Concept)

Use snapshot repositories (e.g., S3/NFS) for backups:
- register snapshot repository
- create snapshots periodically
- restore index/cluster data when needed

Critical for production DR planning.

---

## 14) Troubleshooting

## 14.1 Container not starting
```bash
docker compose logs -f opensearch
```
Common causes:
- insufficient memory
- vm.max_map_count not set

## 14.2 Red/yellow cluster
```bash
curl -s "localhost:9200/_cluster/health?pretty"
curl -s "localhost:9200/_cat/shards?v"
```

## 14.3 Dashboards cannot connect
- wrong `OPENSEARCH_HOSTS`
- opensearch container unhealthy
- network bridge issues

Check:
```bash
docker compose ps
docker compose logs -f dashboards
```

## 14.4 Slow search
- too many shards
- unoptimized mappings
- high-cardinality fields used poorly

---

## 15) Practice Tasks

1. Deploy OpenSearch + Dashboards via compose  
2. Create custom mapped index and ingest sample logs  
3. Query by service/level/time range  
4. Send logs from Fluent Bit to OpenSearch  
5. Create dashboard: errors by service over time  
6. Add retention approach (manual old index deletion for lab)  

---

## 16) Daily Cheat Sheet

```bash
docker compose up -d
docker compose ps
docker compose logs -f opensearch
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/indices?v
```

---

## Final Notes

- Choose Loki for cost-efficient label-based logs.
- Choose OpenSearch for rich full-text analytics and complex search use-cases.
- In bigger environments, many teams use both (different use-cases).