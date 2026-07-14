# HashiCorp Vault Complete Guide (Beginner to Advanced)
## Secrets Management for DevOps (Local/On-Prem)

> This file is detailed and practical.
> Includes install, init/unseal, policies, auth methods, KV secrets, dynamic secrets basics, CI/CD usage, troubleshooting.

---

## 1) What is Vault?

Vault is a centralized secrets management and data protection platform.

## Why use it?
- Central place for secrets (DB passwords, API keys, certs)
- Fine-grained access control via policies
- Dynamic secrets with automatic lease expiry
- Auditability and secret rotation support

---

## 2) Install Vault (Ubuntu)

## 2.1 Add HashiCorp repo and install
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y vault
vault version
```

---

## 3) Dev mode vs Production mode

## Dev mode (quick learning only)
```bash
vault server -dev
```

- Auto-unsealed
- In-memory storage
- Not for production

## Production style
- Uses config file
- Requires init + unseal
- Uses durable storage backend
- TLS strongly recommended

---

## 4) Vault Server Configuration (file-based)

Create config: `/etc/vault.d/vault.hcl`
```hcl
ui = true
disable_mlock = true

storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "http://127.0.0.1:8201"
```

> Note: `tls_disable = 1` only for lab/testing. Use TLS in real setup.

Create dirs:
```bash
sudo mkdir -p /opt/vault/data
sudo chown -R vault:vault /opt/vault
```

---

## 5) systemd Service

If package includes service, enable it:
```bash
sudo systemctl enable --now vault
sudo systemctl status vault
```

If needed, check unit:
```bash
systemctl cat vault
```

Set env for CLI:
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

---

## 6) Initialize and Unseal Vault

## 6.1 Check status
```bash
vault status
```

## 6.2 Initialize (first time only)
```bash
vault operator init
```

Output includes:
- Unseal keys (multiple)
- Initial root token

Store these securely.

## 6.3 Unseal (repeat with threshold number of keys, e.g., 3)
```bash
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>
```

## 6.4 Login
```bash
vault login <root_token>
```

---

## 7) Enable KV Secrets Engine (v2)

## 7.1 Enable
```bash
vault secrets enable -path=secret kv-v2
```

## 7.2 Write secret
```bash
vault kv put secret/dev/app DB_USER=appuser DB_PASS='StrongPass123!'
```

## 7.3 Read secret
```bash
vault kv get secret/dev/app
```

## 7.4 Update only one field (patch)
```bash
vault kv patch secret/dev/app DB_PASS='NewPass456!'
```

## 7.5 Delete and metadata cleanup
```bash
vault kv delete secret/dev/app
vault kv metadata delete secret/dev/app
```

---

## 8) Policies (Access Control)

## 8.1 Create policy file `dev-policy.hcl`
```hcl
path "secret/data/dev/*" {
  capabilities = ["create", "update", "read", "list"]
}

path "secret/metadata/dev/*" {
  capabilities = ["list", "read"]
}
```

Apply policy:
```bash
vault policy write dev-policy dev-policy.hcl
vault policy read dev-policy
```

---

## 9) Authentication Methods

## 9.1 Enable userpass auth
```bash
vault auth enable userpass
```

Create user:
```bash
vault write auth/userpass/users/devuser password='DevUser@123' policies='dev-policy'
```

Login:
```bash
vault login -method=userpass username=devuser
```

## 9.2 Enable approle (CI/CD-friendly)
```bash
vault auth enable approle
vault write auth/approle/role/jenkins-role token_policies="dev-policy"
vault read auth/approle/role/jenkins-role/role-id
vault write -f auth/approle/role/jenkins-role/secret-id
```

Use role_id + secret_id to fetch token programmatically.

---

## 10) Tokens and Leases

Create token:
```bash
vault token create -policy=dev-policy -ttl=1h
```

Inspect token:
```bash
vault token lookup
```

Revoke token:
```bash
vault token revoke <token>
```

Why leases matter:
- Temporary access
- Automatic expiry reduces long-lived secret risk

---

## 11) Dynamic Secrets (concept + starter)

Vault can generate dynamic DB credentials.
High-level flow:
1. Enable DB secrets engine
2. Configure DB connection
3. Define role for generated creds
4. Read creds at runtime with TTL

(Excellent for reducing static password sprawl.)

---

## 12) Transit Engine (encryption as a service)

Enable:
```bash
vault secrets enable transit
vault write -f transit/keys/app-key
```

Encrypt:
```bash
vault write transit/encrypt/app-key plaintext=$(echo -n "hello" | base64)
```

Decrypt:
```bash
vault write transit/decrypt/app-key ciphertext="<ciphertext>"
```

Use case:
- Encrypt sensitive values without exposing raw keys to apps.

---

## 13) Audit Logging

Enable file audit:
```bash
sudo mkdir -p /var/log/vault
sudo chown vault:vault /var/log/vault
vault audit enable file file_path=/var/log/vault/audit.log
vault audit list
```

Why:
- Trace who accessed what and when
- Compliance and incident response

---

## 14) Backup and Recovery Notes

For file storage backend:
- Backup storage path (`/opt/vault/data`)
- Backup config (`/etc/vault.d`)
- Securely back up unseal keys (or use auto-unseal KMS in production)

If using integrated storage (Raft), use snapshots:
```bash
vault operator raft snapshot save vault.snap
vault operator raft snapshot restore vault.snap
```

---

## 15) Jenkins/CI Integration Pattern

Recommended:
- Use AppRole auth
- Fetch short-lived token at build start
- Read required secret(s)
- Avoid printing secrets in logs

Pseudo flow:
1. Jenkins gets role_id + secret_id from secure credential store
2. Auth to Vault and get token
3. Read `secret/data/dev/app`
4. Inject to env only for needed step

---

## 16) Troubleshooting

## 16.1 Vault sealed
```bash
vault status
vault operator unseal <key>
```

## 16.2 Permission denied
- Wrong policy path (`secret/data/...` for KV v2 read)
- Missing capabilities (`read`, `list`, etc.)

Check:
```bash
vault token lookup
vault policy read <policy-name>
```

## 16.3 Cannot connect to Vault
- Wrong `VAULT_ADDR`
- Service down
- Firewall/network issue

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
curl -s $VAULT_ADDR/v1/sys/health
sudo systemctl status vault
```

## 16.4 KV v2 path confusion
For KV v2:
- CLI logical path: `secret/dev/app`
- Policy/API path often includes `/data/` and `/metadata/`

---

## 17) Practice Tasks

1. Install Vault and start service  
2. Initialize and unseal Vault  
3. Enable KV v2 and store app secret  
4. Create policy and userpass user  
5. Enable AppRole and retrieve role_id/secret_id  
6. Enable transit and encrypt/decrypt sample text  
7. Enable audit logs and verify entries  

---

## 18) Daily Vault Cheat Sheet

```bash
export VAULT_ADDR='http://127.0.0.1:8200'

vault status
vault login <token>

vault secrets list
vault auth list
vault policy list

vault kv put secret/dev/app key=value
vault kv get secret/dev/app

vault token create -policy=dev-policy -ttl=1h
vault token lookup
```

---

## 19) Kubernetes Auth Method

## Why
Let pods authenticate to Vault using their ServiceAccount token instead of static credentials — the standard pattern for apps running in-cluster.

## Enable and configure
```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"
```

## Create a role binding a K8s ServiceAccount to a Vault policy
```bash
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=dev \
  policies=dev-policy \
  ttl=1h
```

## From inside a pod (using the projected SA token)
```bash
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
vault write auth/kubernetes/login role=myapp jwt=$JWT
```

---

## 20) Vault Agent Injector (Automatic Secret Injection)

## Why
Instead of writing Vault API calls into every app, the Vault Agent Injector mutates pods via a webhook, injecting secrets as files into a sidecar — apps just read local files.

## Install (Helm)
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set "injector.enabled=true" \
  --set "server.dev.enabled=true" \
  -n vault --create-namespace
```

## Annotate a Deployment to request injection
```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/dev/app"
```

The sidecar writes the rendered secret to `/vault/secrets/db-creds` inside the pod, refreshed automatically on lease renewal.

---

## 21) PKI Secrets Engine (Certificates as a Service)

## Why
Issue short-lived TLS certificates on demand instead of managing long-lived certs manually.

## Enable and set up a root CA
```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal \
  common_name="example.com" ttl=87600h
```

## Configure a role for issuing certs
```bash
vault write pki/roles/example-dot-com \
  allowed_domains="example.com" \
  allow_subdomains=true \
  max_ttl="720h"
```

## Issue a certificate
```bash
vault write pki/issue/example-dot-com common_name="app.example.com" ttl="24h"
```

Returns a cert, private key, and CA chain — apps or Vault Agent can request fresh certs before expiry, enabling automatic rotation.

---

## Final Notes

- Vault should be central secrets authority in serious DevOps setups.
- Start with KV + policies + AppRole, then adopt dynamic secrets and transit.
- Use TLS, audit logs, and secure key management for production.
- In Kubernetes, prefer the Kubernetes auth method + Agent Injector over static tokens, and use PKI for short-lived certs.
