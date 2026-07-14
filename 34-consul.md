# HashiCorp Consul Complete Guide (Beginner to Advanced)
## Service Discovery, Service Mesh, and KV Store for DevOps

> This file is detailed and practical.
> Covers install, agent modes, service registration/discovery, health checks, KV store, service mesh basics, ACLs, troubleshooting.

---

## Table of Contents

1. What is Consul and why it matters
2. Core concepts
3. Install Consul
4. Dev mode vs production mode
5. Server cluster setup
6. Agent configuration (server vs client)
7. Service registration and discovery
8. Health checks
9. DNS interface
10. KV store
11. Consul Connect (service mesh basics)
12. Consul on Kubernetes
13. ACLs (access control)
14. Multi-datacenter federation (concept)
15. Integration with Vault
16. Troubleshooting matrix
17. Practice lab tasks
18. Daily cheat sheet

---

## 1) What is Consul and Why it Matters

## What
Consul is a service networking platform providing service discovery, health checking, a distributed key-value store, and an optional service mesh (Consul Connect) — all backed by the same Raft-consensus cluster.

## Why DevOps/platform teams use it
- Services find each other by name instead of hardcoded IPs (`payments.service.consul` instead of `10.0.4.17`)
- Automatic health-check-based failover — unhealthy instances drop out of DNS/API responses immediately
- Works across VMs, containers, and Kubernetes in the same mesh
- Pairs naturally with Vault (same vendor, same Raft/gossip patterns) for a full HashiCorp stack

---

## 2) Core Concepts

- **Agent**: the Consul process running on every node — either in **server** or **client** mode
- **Server**: participates in the Raft consensus quorum, stores cluster state (typically 3 or 5 per datacenter)
- **Client**: lightweight agent forwarding requests to servers, runs on every service host
- **Service**: something registered with Consul (name, address, port, health check)
- **Catalog**: the registry of all known services/nodes
- **Health check**: script/HTTP/TCP/TTL check determining if a service instance is healthy
- **KV store**: a hierarchical key-value store for config/feature flags/coordination
- **Datacenter**: a Consul cluster boundary — can be federated with others

---

## 3) Install Consul

## 3.1 Add HashiCorp repo and install (Ubuntu)
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y consul
consul version
```

---

## 4) Dev Mode vs Production Mode

## Dev mode (quick learning only)
```bash
consul agent -dev
```
- Single in-memory node, no persistence, UI enabled by default
- Never use for anything beyond local exploration

## Access the UI
- `http://localhost:8500/ui`

## Verify
```bash
consul members
consul info
```

---

## 5) Server Cluster Setup (Production Style)

## 5.1 Generate a gossip encryption key
```bash
consul keygen
```

## 5.2 Server config (`/etc/consul.d/server.hcl`)
```hcl
datacenter = "dc1"
data_dir   = "/opt/consul/data"
server     = true
bootstrap_expect = 3
bind_addr  = "0.0.0.0"
client_addr = "0.0.0.0"
ui_config {
  enabled = true
}
encrypt = "<KEY_FROM_KEYGEN>"
retry_join = ["10.0.1.10", "10.0.1.11", "10.0.1.12"]
```

## 5.3 systemd service
```bash
cat <<'EOF' | sudo tee /etc/systemd/system/consul.service
[Unit]
Description=Consul
After=network-online.target
Wants=network-online.target

[Service]
User=consul
Group=consul
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo mkdir -p /opt/consul/data /etc/consul.d
sudo useradd --no-create-home --shell /bin/false consul || true
sudo chown -R consul:consul /opt/consul /etc/consul.d
sudo systemctl daemon-reload
sudo systemctl enable --now consul
```

## 5.4 Verify cluster formed
```bash
consul members
consul operator raft list-peers
```

---

## 6) Agent Configuration (Server vs Client)

## Client config (`/etc/consul.d/client.hcl`) — runs on every application host
```hcl
datacenter = "dc1"
data_dir   = "/opt/consul/data"
server     = false
bind_addr  = "0.0.0.0"
retry_join = ["10.0.1.10", "10.0.1.11", "10.0.1.12"]
encrypt    = "<KEY_FROM_KEYGEN>"
```

Clients forward all reads/writes to the server quorum but keep the local gossip pool healthy and run local health checks close to the service.

---

## 7) Service Registration and Discovery

## 7.1 Register a service via config file (`/etc/consul.d/web.json`)
```json
{
  "service": {
    "name": "web",
    "port": 8080,
    "tags": ["v1", "frontend"],
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s"
    }
  }
}
```
```bash
consul reload
```

## 7.2 Register via the HTTP API
```bash
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d '{
    "Name": "web",
    "Port": 8080,
    "Check": {"HTTP": "http://localhost:8080/health", "Interval": "10s"}
  }'
```

## 7.3 Discover a service
```bash
curl -s http://localhost:8500/v1/catalog/service/web | jq
consul catalog services
```

---

## 8) Health Checks

## Check types
- **HTTP**: polls an endpoint, expects 2xx
- **TCP**: verifies a port accepts connections
- **Script/args**: runs a local command, checks exit code
- **TTL**: the service itself must actively "check in" before the TTL expires, or it's marked critical

## Example script check
```json
{
  "check": {
    "args": ["/usr/local/bin/check_disk.sh"],
    "interval": "30s"
  }
}
```

## View health status
```bash
consul catalog services
curl -s http://localhost:8500/v1/health/service/web?passing | jq
```
The `?passing` filter is the key operational pattern — always query for *passing* instances only when routing real traffic.

---

## 9) DNS Interface

## Why
Services can be looked up via standard DNS instead of the HTTP API — no client library needed.

```bash
dig @127.0.0.1 -p 8600 web.service.consul
```

Configure system resolvers to forward `.consul` queries to Consul (e.g., via `dnsmasq` or `systemd-resolved`) so any application can resolve `web.service.consul` transparently, without knowing Consul exists.

---

## 10) KV Store

## Write/read
```bash
consul kv put config/app/feature_flag "true"
consul kv get config/app/feature_flag
```

## List keys under a prefix
```bash
consul kv get -recurse config/app/
```

## Delete
```bash
consul kv delete config/app/feature_flag
```

## Watch for changes (useful for config-reload automation)
```bash
consul watch -type=key -key=config/app/feature_flag ./reload-app.sh
```

Common use: feature flags, dynamic app config, and distributed locks/leader election (`consul lock`).

---

## 11) Consul Connect (Service Mesh Basics)

## Why
Automatic mutual TLS between services, plus traffic policy (who can talk to whom) — the lighter-weight HashiCorp alternative to Istio, integrated directly into the same Consul agent.

## Enable Connect (server config)
```hcl
connect {
  enabled = true
}
```

## Register a service with a sidecar proxy
```json
{
  "service": {
    "name": "web",
    "port": 8080,
    "connect": {
      "sidecar_service": {}
    }
  }
}
```

## Intentions (allow/deny which services can talk to which)
```bash
consul intention create -allow web api
consul intention create -deny api database-admin
```

Default-deny posture (recommended): set default intention to `deny`, then explicitly `allow` only the service-to-service paths that should exist.

---

## 12) Consul on Kubernetes

## Install via Helm (official Consul K8s chart)
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul \
  --set global.name=consul \
  --set connectInject.enabled=true \
  -n consul --create-namespace
```

## Enable mesh injection for a namespace
```yaml
metadata:
  annotations:
    consul.hashicorp.com/connect-inject: "true"
```

Consul's Kubernetes integration auto-registers K8s Services into the Consul catalog and injects Envoy sidecars when `connect-inject` is enabled — similar mechanics to Istio's sidecar injection.

---

## 13) ACLs (Access Control)

## Why
Dev mode has no authentication — production Consul needs ACLs to restrict who can register services, read the KV store, or use the API.

## Bootstrap ACL system
```bash
consul acl bootstrap
```
Save the generated management token securely — it's shown only once.

## Create a policy
```bash
cat <<EOF > web-policy.hcl
service "web" {
  policy = "write"
}
node_prefix "" {
  policy = "read"
}
EOF

consul acl policy create -name "web-policy" -rules @web-policy.hcl
```

## Create a token bound to the policy
```bash
consul acl token create -description "web service token" -policy-name "web-policy"
```

Use the token in agent config or via `CONSUL_HTTP_TOKEN` env var for API calls.

---

## 14) Multi-Datacenter Federation (Concept)

Consul supports WAN federation — linking multiple datacenters' server clusters so services can be discovered across regions/clouds:

```hcl
datacenter = "dc2"
retry_join_wan = ["<dc1-server-ip>"]
```

```bash
consul members -wan
```

Common pattern: keep read/write local (each DC's clients talk to their own DC's servers) but allow cross-DC service discovery for failover/DR scenarios.

---

## 15) Integration with Vault

Consul and Vault pair naturally:
- Vault can use Consul as its storage backend (`storage "consul" { ... }` in Vault's config)
- Vault's Kubernetes/service-mesh workflows often sit alongside Consul Connect for mTLS + secrets in the same platform

Example Vault storage backend using Consul:
```hcl
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}
```

---

## 16) Troubleshooting Matrix

## 16.1 Agent won't join cluster
```bash
consul members
journalctl -u consul -n 200 --no-pager
```
Common causes: wrong `encrypt` key, firewall blocking gossip ports (8301/8302), wrong `retry_join` addresses.

## 16.2 Service shows as "critical"
```bash
curl -s http://localhost:8500/v1/health/service/web | jq
```
Check the health check endpoint/command directly — the check itself is usually failing, not Consul.

## 16.3 DNS resolution not working
```bash
dig @127.0.0.1 -p 8600 web.service.consul
```
Verify the DNS interface is reachable and the service is registered and passing.

## 16.4 Raft quorum lost (servers can't elect a leader)
```bash
consul operator raft list-peers
```
Needs a majority of servers (`bootstrap_expect`) online — check node count and network partitions.

## 16.5 ACL "permission denied"
```bash
consul acl token read -self
```
Verify the token in use has a policy granting the needed permission.

---

## 17) Practice Lab Tasks

1. Run Consul in dev mode and explore the UI
2. Deploy a 3-server production-style cluster with gossip encryption
3. Register a service with an HTTP health check
4. Query the service via both the HTTP API and DNS interface
5. Store and retrieve a config value in the KV store
6. Enable Connect and create an allow intention between two services
7. Bootstrap ACLs and create a scoped token for one service
8. Deploy Consul on Kubernetes with Connect injection enabled

---

## 18) Daily Cheat Sheet

```bash
consul members
consul catalog services
consul catalog nodes

consul kv put <key> <value>
consul kv get <key>

curl -s http://localhost:8500/v1/health/service/<name>?passing | jq

consul intention create -allow <src> <dst>
consul acl token list

consul operator raft list-peers
```

---

## Final Notes

- Consul's value compounds once you have more than a handful of services — hardcoded IPs and manual load balancer config stop scaling fast.
- Always query health-filtered (`?passing`) results when routing real traffic, never the raw catalog.
- Start with service discovery + health checks; add Connect (mesh) and ACLs once the basics are solid.
- On Kubernetes, weigh Consul Connect against Istio — Consul is lighter-weight and pairs well if you're already using Vault.
