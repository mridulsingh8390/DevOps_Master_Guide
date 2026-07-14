# Apache Kafka Complete Guide (Beginner to Advanced)
## Event Streaming Platform for DevOps Engineers

> This file is detailed and practical.
> Covers install, core concepts, CLI operations, Kafka Connect, monitoring, Kubernetes deployment, troubleshooting.

---

## Table of Contents

1. What is Kafka and why it matters
2. Core concepts
3. Install Kafka (KRaft mode, no ZooKeeper)
4. Topics — create, list, describe, configure
5. Producing and consuming from the CLI
6. Consumer groups and offsets
7. Partitions, replication, and durability
8. Kafka Connect (data pipelines in/out)
9. Schema Registry basics
10. Kafka on Kubernetes (Strimzi Operator)
11. Monitoring Kafka
12. Security (SASL/TLS/ACLs)
13. Performance tuning basics
14. Troubleshooting matrix
15. Practice lab tasks
16. Daily cheat sheet

---

## 1) What is Kafka and Why it Matters

## What
Kafka is a distributed event streaming platform — a durable, ordered, replayable log that producers write to and consumers read from.

## Why DevOps/platform teams use it
- Decouples services (producers/consumers don't call each other directly)
- Durable buffer between systems — smooths out traffic spikes, survives consumer downtime
- Replayable — consumers can re-read history, not just "the latest" like a queue
- Backbone for event-driven architectures, log aggregation pipelines, and CDC (change data capture)

---

## 2) Core Concepts

- **Topic**: a named stream of records (like a table, append-only)
- **Partition**: a topic is split into partitions for parallelism; order is guaranteed only within a partition
- **Broker**: a single Kafka server; a cluster is made of multiple brokers
- **Producer**: writes records to a topic
- **Consumer**: reads records from a topic
- **Consumer Group**: a set of consumers sharing the work of reading a topic — each partition is read by exactly one consumer within a group
- **Offset**: a consumer's position (per partition) in the log
- **Replication factor**: how many broker copies each partition has, for fault tolerance
- **KRaft**: Kafka's modern built-in consensus mode (replaces ZooKeeper as of Kafka 3.x+/4.x)

---

## 3) Install Kafka (KRaft Mode, No ZooKeeper)

## 3.1 Prerequisites
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk wget
java -version
```

## 3.2 Download and extract
```bash
cd /tmp
wget https://downloads.apache.org/kafka/3.8.0/kafka_2.13-3.8.0.tgz
tar -xzf kafka_2.13-3.8.0.tgz
sudo mv kafka_2.13-3.8.0 /opt/kafka
```

## 3.3 Generate a cluster ID and format storage (KRaft-specific, one-time)
```bash
cd /opt/kafka
KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

## 3.4 Start Kafka (single-node broker+controller combo, lab-friendly)
```bash
bin/kafka-server-start.sh config/kraft/server.properties
```

## 3.5 systemd service (production-style)
```bash
cat <<'EOF' | sudo tee /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka
After=network.target

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo useradd --no-create-home --shell /bin/false kafka || true
sudo chown -R kafka:kafka /opt/kafka
sudo systemctl daemon-reload
sudo systemctl enable --now kafka
sudo systemctl status kafka
```

---

## 4) Topics — Create, List, Describe, Configure

## Create a topic
```bash
bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic app-events \
  --partitions 3 \
  --replication-factor 1
```

## List topics
```bash
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

## Describe a topic
```bash
bin/kafka-topics.sh --describe --topic app-events --bootstrap-server localhost:9092
```

## Alter partition count (can only increase, never decrease)
```bash
bin/kafka-topics.sh --alter --topic app-events --partitions 6 --bootstrap-server localhost:9092
```

## Configure retention
```bash
bin/kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name app-events \
  --add-config retention.ms=604800000
```

## Delete a topic
```bash
bin/kafka-topics.sh --delete --topic app-events --bootstrap-server localhost:9092
```

---

## 5) Producing and Consuming from the CLI

## Produce messages (interactive)
```bash
bin/kafka-console-producer.sh --topic app-events --bootstrap-server localhost:9092
> {"event":"user_signup","user_id":123}
> {"event":"user_login","user_id":123}
```

## Consume messages (from the beginning)
```bash
bin/kafka-console-consumer.sh \
  --topic app-events \
  --from-beginning \
  --bootstrap-server localhost:9092
```

## Consume only new messages (default behavior, no `--from-beginning`)
```bash
bin/kafka-console-consumer.sh --topic app-events --bootstrap-server localhost:9092
```

## Produce with a key (controls which partition a message lands in)
```bash
bin/kafka-console-producer.sh \
  --topic app-events \
  --property "parse.key=true" \
  --property "key.separator=:" \
  --bootstrap-server localhost:9092
> user-123:{"event":"login"}
```

---

## 6) Consumer Groups and Offsets

## List consumer groups
```bash
bin/kafka-consumer-groups.sh --list --bootstrap-server localhost:9092
```

## Describe a group (shows lag — how far behind each consumer is)
```bash
bin/kafka-consumer-groups.sh --describe --group my-app-group --bootstrap-server localhost:9092
```

Key output columns: `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG` — lag is the most important number to watch operationally; growing lag means consumers can't keep up with producers.

## Reset offsets (replay from the beginning — useful after fixing a consumer bug)
```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-app-group --topic app-events \
  --reset-offsets --to-earliest --execute
```

## Reset to a specific timestamp
```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-app-group --topic app-events \
  --reset-offsets --to-datetime 2026-07-01T00:00:00.000 --execute
```

---

## 7) Partitions, Replication, and Durability

## Why partition count matters
More partitions = more parallelism (more consumers in a group can work simultaneously), but also more overhead and longer rebalances. A common starting point is partitions = expected max consumer count.

## Why replication factor matters
`replication-factor 3` means each partition has 3 copies across different brokers — the cluster tolerates 2 broker failures without data loss (with `min.insync.replicas=2`).

## Producer durability settings (`acks`)
- `acks=0`: fire and forget, fastest, no durability guarantee
- `acks=1`: leader broker acknowledges, fair durability
- `acks=all` (or `-1`): all in-sync replicas acknowledge — strongest durability, use for anything you can't afford to lose

```bash
bin/kafka-console-producer.sh --topic app-events \
  --producer-property acks=all \
  --bootstrap-server localhost:9092
```

---

## 8) Kafka Connect (Data Pipelines In/Out)

## Why
Move data between Kafka and external systems (databases, S3, Elasticsearch) declaratively, without writing custom producer/consumer code.

## Start Connect in standalone mode (lab)
```bash
bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties
```

## Common connector patterns (via Connect REST API, distributed mode)
```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "file-source",
    "config": {
      "connector.class": "FileStreamSource",
      "tasks.max": "1",
      "file": "/var/log/app.log",
      "topic": "app-logs"
    }
  }'
```

## Check connector status
```bash
curl -s http://localhost:8083/connectors/file-source/status | jq
```

Widely used connectors in real deployments: Debezium (CDC from Postgres/MySQL), S3 Sink, JDBC Sink, Elasticsearch Sink.

---

## 9) Schema Registry Basics

## Why
Enforce a consistent message structure (Avro/Protobuf/JSON Schema) between producers and consumers — prevents "producer changed the format and broke every consumer" incidents.

## Register a schema (Confluent Schema Registry example)
```bash
curl -X POST http://localhost:8081/subjects/app-events-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "{\"type\":\"record\",\"name\":\"Event\",\"fields\":[{\"name\":\"event\",\"type\":\"string\"}]}"}'
```

## List subjects
```bash
curl -s http://localhost:8081/subjects | jq
```

Schema Registry enforces compatibility rules (backward/forward/full) so producers can't push a breaking schema change without an explicit override.

---

## 10) Kafka on Kubernetes (Strimzi Operator)

## Why
Run Kafka clusters natively on Kubernetes with CRD-driven configuration instead of hand-managed VMs.

## Install the operator
```bash
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

## Deploy a cluster (KRaft mode)
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
  annotations:
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.8.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 20Gi
  entityOperator:
    topicOperator: {}
    userOperator: {}
```
```bash
kubectl apply -f kafka-cluster.yaml -n kafka
kubectl get kafka -n kafka
```

## Create a topic via CRD
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: app-events
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
```

---

## 11) Monitoring Kafka

## Key metrics to watch (via JMX → Prometheus, e.g. using the Prometheus/Grafana guide in this series)
- **Consumer lag** (per group/topic) — the single most important operational signal
- **Under-replicated partitions** — should be 0; nonzero means a broker is behind or down
- **Request/response time** (produce/fetch latency)
- **Broker disk usage** — Kafka retains data by size/time, plan capacity accordingly

## Expose JMX metrics for Prometheus scraping
```bash
export KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx_exporter/kafka.yml"
bin/kafka-server-start.sh config/kraft/server.properties
```
Then add a Prometheus scrape job for `localhost:9404` (see the Prometheus + Grafana guide, Section 3).

---

## 12) Security (SASL/TLS/ACLs)

## Enable SASL authentication (`server.properties`)
```properties
listeners=SASL_PLAINTEXT://0.0.0.0:9092
sasl.enabled.mechanisms=PLAIN
sasl.mechanism.inter.broker.protocol=PLAIN
security.inter.broker.protocol=SASL_PLAINTEXT
```

## Create ACLs (restrict who can produce/consume what)
```bash
bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:app-producer \
  --operation Write --topic app-events

bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:app-consumer \
  --operation Read --topic app-events --group my-app-group
```

## List ACLs
```bash
bin/kafka-acls.sh --bootstrap-server localhost:9092 --list
```

For production, terminate TLS on the listener and use SASL_SSL rather than plaintext SASL.

---

## 13) Performance Tuning Basics

- **Batch size / linger.ms** (producer): larger batches = higher throughput, slightly higher latency
```properties
batch.size=32768
linger.ms=10
```
- **Compression**: reduces network/disk usage
```properties
compression.type=snappy
```
- **fetch.min.bytes / fetch.max.wait.ms** (consumer): tune for throughput vs latency trade-off
- **Partition count**: too few limits parallelism, too many increases broker overhead and rebalance time — benchmark, don't guess

---

## 14) Troubleshooting Matrix

## 14.1 Broker won't start
```bash
sudo journalctl -u kafka -n 200 --no-pager
```
Common cause: storage not formatted (`kafka-storage.sh format`) or cluster ID mismatch after a config change.

## 14.2 Consumer lag growing steadily
- Consumer processing is slower than production rate — scale out consumers (up to partition count) or optimize processing logic

## 14.3 "Not enough in-sync replicas"
- `min.insync.replicas` set higher than available healthy replicas — check broker health, or the setting is too strict for current cluster size

## 14.4 Messages not appearing for a consumer group
- Check offset reset policy: `auto.offset.reset=latest` means new groups only see *future* messages, not history
```properties
auto.offset.reset=earliest
```

## 14.5 Under-replicated partitions
```bash
bin/kafka-topics.sh --describe --under-replicated-partitions --bootstrap-server localhost:9092
```
Indicates a broker is down or struggling — check broker logs and disk/network health.

---

## 15) Practice Lab Tasks

1. Install Kafka in KRaft mode and start a single-node broker
2. Create a topic with 3 partitions and produce/consume via CLI
3. Start two consumers in the same group and observe partition assignment
4. Intentionally lag a consumer and observe `--describe` lag output
5. Reset a consumer group's offsets to earliest and replay messages
6. Set up a simple file-source Kafka Connect connector
7. Deploy a 3-broker cluster on Kubernetes via Strimzi
8. Add SASL auth and one ACL restricting a topic to a specific principal

---

## 16) Daily Cheat Sheet

```bash
# Topics
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
bin/kafka-topics.sh --describe --topic <topic> --bootstrap-server localhost:9092

# Produce/consume
bin/kafka-console-producer.sh --topic <topic> --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic <topic> --from-beginning --bootstrap-server localhost:9092

# Consumer groups
bin/kafka-consumer-groups.sh --list --bootstrap-server localhost:9092
bin/kafka-consumer-groups.sh --describe --group <group> --bootstrap-server localhost:9092

# Service
sudo systemctl status kafka
sudo journalctl -u kafka -f
```

---

## Final Notes

- Kafka's core value is the durable, replayable log — design topics and partition keys around that, not around "just another queue."
- Consumer lag is the metric to watch daily; everything else is secondary to that signal.
- Use Kafka Connect for standard integrations instead of hand-rolled producer/consumer glue code.
- On Kubernetes, Strimzi is the de facto standard operator — prefer it over hand-managed StatefulSets.
