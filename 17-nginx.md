# Nginx Complete Guide (Reverse Proxy, TLS, Load Balancing)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

> This file is detailed and practical.
> Covers install, core config, reverse proxy, TLS, load balancing, rate limiting, logs, troubleshooting.

---

## 1) What is Nginx and Why it Matters

Nginx is a high-performance web server and reverse proxy.

## Why use it in DevOps?
- Expose internal tools (Jenkins, Grafana, SonarQube, Nexus) behind one entrypoint
- TLS termination (HTTPS)
- Load balancing across app instances
- Security controls (rate-limit, headers, access rules)

---

## 2) Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
nginx -v
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

Verify:
```bash
curl -I http://localhost
```

---

## 3) Nginx Directory Layout (Ubuntu)

- Main config: `/etc/nginx/nginx.conf`
- Site available: `/etc/nginx/sites-available/`
- Site enabled symlink: `/etc/nginx/sites-enabled/`
- Logs: `/var/log/nginx/access.log`, `/var/log/nginx/error.log`
- Web root default: `/var/www/html`

Test and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4) Basic Server Block

Create:
`/etc/nginx/sites-available/app.conf`
```nginx
server {
    listen 80;
    server_name app.local;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable:
```bash
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/app.conf
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5) Reverse Proxy (Most common DevOps use)

Example: Proxy Jenkins running on `localhost:8080`

```nginx
server {
    listen 80;
    server_name jenkins.local;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 90;
    }
}
```

Use same pattern for:
- Grafana (`3000`)
- SonarQube (`9000`)
- Nexus (`8081`)
- Argo CD (via HTTPS/TLS aware config)

---

## 6) Load Balancing Upstream

```nginx
upstream backend_app {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
    server 10.0.0.13:8080;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://backend_app;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Load balancing methods:
- Round robin (default)
- Least connections (`least_conn;`)
- IP hash (`ip_hash;`)

---

## 7) TLS (HTTPS) Setup

## 7.1 Self-signed cert (lab)
```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt
```

## 7.2 HTTPS server block
```nginx
server {
    listen 443 ssl;
    server_name app.local;

    ssl_certificate     /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}

server {
    listen 80;
    server_name app.local;
    return 301 https://$host$request_uri;
}
```

---

## 8) Security Headers and Hardening

Add inside server block:
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-XSS-Protection "1; mode=block" always;
```

Hide version:
In `/etc/nginx/nginx.conf` under `http {}`:
```nginx
server_tokens off;
```

---

## 9) Rate Limiting and Basic Abuse Protection

Define in `http {}` context:
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

Use in location:
```nginx
location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
    proxy_pass http://backend_app;
}
```

---

## 10) Gzip Compression

Inside `http {}`:
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_min_length 1024;
```

---

## 11) WebSocket Proxy Support

```nginx
location /ws/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

---

## 12) Access Control

Allow only specific subnet:
```nginx
location /admin/ {
    allow 10.0.0.0/24;
    deny all;
    proxy_pass http://127.0.0.1:9000;
}
```

Basic auth:
```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

Config:
```nginx
location /secure/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:8081;
}
```

---

## 13) Logging and Observability

Default logs:
```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

Custom log format in `http {}`:
```nginx
log_format main_ext '$remote_addr - $host [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$request_time"';
access_log /var/log/nginx/access.log main_ext;
```

---

## 14) Common Reverse Proxy Examples for DevOps Tools

## Jenkins
```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

## Grafana
```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
}
```

## SonarQube
```nginx
location / {
    proxy_pass http://127.0.0.1:9000;
}
```

## Nexus
```nginx
location / {
    proxy_pass http://127.0.0.1:8081;
}
```

(Enhance each with forwarded headers from earlier template.)

---

## 15) Troubleshooting Matrix

## 15.1 Config syntax error
```bash
sudo nginx -t
```
Fix line from output and reload.

## 15.2 502 Bad Gateway
- Upstream app down
- Wrong host/port in `proxy_pass`
- Firewall/local bind issue

Checks:
```bash
curl -I http://127.0.0.1:8080
sudo ss -tulnp | grep 8080
```

## 15.3 404 from Nginx
- wrong root path / location match
- wrong server_name hit

Debug:
```bash
curl -H "Host: app.local" http://127.0.0.1/
```

## 15.4 Permission denied
- wrong file perms for web root/cert/log

---

## 16) Practice Tasks

1. Install Nginx and serve static page  
2. Configure reverse proxy to Jenkins  
3. Add HTTPS with self-signed cert  
4. Add rate limit on `/api/`  
5. Add basic auth for `/secure/`  
6. Configure upstream load balancing for 2 backend apps  
7. Add security headers and verify in browser/network tab  

---

## 17) Daily Nginx Cheat Sheet

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart nginx
sudo systemctl status nginx

tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

---

## 18) Nginx as a Kubernetes Ingress Controller

## Why
The standalone Nginx skills above (reverse proxy, TLS, rate limiting, headers) map almost directly onto the Nginx Ingress Controller's annotations when Nginx runs inside Kubernetes instead of on a VM.

## Install (see also the Kubernetes guide, Section 19)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

## Translate common VM-Nginx patterns to Ingress annotations

Rate limiting:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

Basic auth (via a Secret instead of `.htpasswd` file):
```bash
htpasswd -c auth admin
kubectl create secret generic basic-auth --from-file=auth -n dev
```
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Restricted"
```

TLS termination (cert from a Secret, often issued by cert-manager):
```yaml
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
```

Custom snippet (raw nginx.conf injection, if the controller allows snippets):
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
```

---

## 19) ModSecurity (WAF Basics)

## Why
Rate limiting and access control stop abuse patterns; a WAF (Web Application Firewall) inspects request content for injection/XSS/known attack signatures.

## Enable ModSecurity module (Ubuntu, standalone Nginx)
```bash
sudo apt install -y libnginx-mod-http-modsecurity
```

Enable in `/etc/nginx/nginx.conf` (`http {}` block):
```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

`/etc/nginx/modsec/main.conf`:
```
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/owasp-crs/crs-setup.conf
Include /etc/nginx/owasp-crs/rules/*.conf
```

Use the OWASP Core Rule Set (CRS) for baseline protection — install via `git clone https://github.com/coreruleset/coreruleset` and point the includes at it.

## On Kubernetes (Ingress Controller)
The Nginx Ingress Controller ships an optional ModSecurity build; enable via ConfigMap:
```yaml
data:
  enable-modsecurity: "true"
  enable-owasp-modsecurity-crs: "true"
```

## Test in detection-only mode first
```
SecRuleEngine DetectionOnly
```
Review `/var/log/nginx/modsec_audit.log` before switching to `On` (blocking) mode to avoid false-positive outages.

---

## Final Notes

- Nginx is the front door of most self-hosted DevOps stacks.
- Keep configs modular (one file per app).
- Always run `nginx -t` before reload.
- Add TLS + security headers + access controls by default.
- The same reverse-proxy/rate-limit/auth concepts carry over to Kubernetes Ingress via annotations; add a WAF (ModSecurity + OWASP CRS) for internet-facing endpoints.
