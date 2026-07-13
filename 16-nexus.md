# Nexus Repository Manager Complete Guide (Beginner to Advanced)
## Artifact Management for DevOps (Local/On-Prem)

> This file is detailed and practical.
> Covers install, repo types, users/roles, Docker/Maven/npm usage, CI integration, backups, troubleshooting.

---

## 1) What is Nexus and Why it Matters

Nexus Repository Manager is an artifact repository for storing and distributing build outputs.

## Why use it?
- Single source for artifacts (JAR, npm package, Docker image, etc.)
- Caching proxy for external repos (faster and reliable builds)
- Versioned artifacts with retention control
- CI/CD integration for publish + promote workflows

---

## 2) Install Nexus (Ubuntu, tar-based)

## 2.1 Install Java
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

## 2.2 Create nexus user
```bash
sudo useradd -m -d /opt/nexus -s /bin/bash nexus || true
```

## 2.3 Download and extract Nexus
```bash
cd /tmp
wget https://download.sonatype.com/nexus/3/nexus-3.72.0-04-unix.tar.gz
sudo tar -xzf nexus-3.72.0-04-unix.tar.gz -C /opt
sudo mv /opt/nexus-3.72.0-04 /opt/nexus
sudo mkdir -p /opt/sonatype-work
sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work
```

## 2.4 Run as nexus user
Edit:
`/opt/nexus/bin/nexus.rc`
```bash
run_as_user="nexus"
```

---

## 3) systemd Service

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/nexus.service
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOF
```

Start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nexus
sudo systemctl status nexus
```

Default URL:
- `http://<server-ip>:8081`

---

## 4) Initial Login and Password Setup

Initial admin password file:
```bash
sudo cat /opt/sonatype-work/nexus3/admin.password
```

Login:
- user: `admin`
- password: from file

Then:
1. Change admin password
2. Configure anonymous access policy as needed

---

## 5) Repository Types and Strategy

Nexus repo types:
- **Hosted**: your internal artifacts
- **Proxy**: cache external upstream repos
- **Group**: aggregate hosted + proxy under one URL

Common formats:
- Maven2
- npm
- Docker
- PyPI
- NuGet
- Raw

Recommended pattern:
- one hosted + one proxy + one group per ecosystem

---

## 6) Maven Repository Setup

## Suggested repos
- `maven-central` (proxy)
- `maven-releases` (hosted)
- `maven-snapshots` (hosted)
- `maven-public` (group)

## Maven settings.xml example
`~/.m2/settings.xml`
```xml
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://<nexus-host>:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

Deploy artifacts:
- Configure distributionManagement in `pom.xml`
- Use credentials in settings.xml server block

---

## 7) npm Repository Setup

Create:
- `npm-hosted`
- `npm-proxy` (registry.npmjs.org)
- `npm-group`

Set npm registry:
```bash
npm config set registry http://<nexus-host>:8081/repository/npm-group/
npm ping
```

Login publish:
```bash
npm login --registry=http://<nexus-host>:8081/repository/npm-hosted/
npm publish --registry=http://<nexus-host>:8081/repository/npm-hosted/
```

---

## 8) Docker Registry with Nexus

Create Docker repos in Nexus:
- `docker-hosted`
- `docker-proxy`
- `docker-group`

Optionally configure HTTP connector port (e.g., 8082) for Docker repo.

Docker login:
```bash
docker login <nexus-host>:8082
```

Tag and push:
```bash
docker tag myapp:1.0 <nexus-host>:8082/myapp:1.0
docker push <nexus-host>:8082/myapp:1.0
```

Pull:
```bash
docker pull <nexus-host>:8082/myapp:1.0
```

---

## 9) Users, Roles, and Privileges

Best practice:
- Disable broad admin token usage in CI
- Create service accounts per pipeline/team
- Grant least privilege (read/publish only where needed)

Typical roles:
- Read-only consumers
- CI publishers
- Repo admins

---

## 10) Cleanup and Retention Policies

Why:
- Avoid disk bloat and stale snapshots

Use Nexus cleanup policies:
- delete old snapshots
- delete components not downloaded for X days
- keep latest N versions

Apply policies per repo (especially snapshots/docker tags).

---

## 11) Backups and Restore

Important paths:
- App binaries: `/opt/nexus`
- Data: `/opt/sonatype-work/nexus3`

Backup strategy:
1. Stop Nexus
2. Backup data directory
3. Start Nexus

Example:
```bash
sudo systemctl stop nexus
sudo tar -czf /tmp/nexus-backup-$(date +%F).tar.gz /opt/sonatype-work/nexus3
sudo systemctl start nexus
```

Restore:
- stop service
- restore backup to same path
- fix ownership
- start service

---

## 12) CI/CD Integration Pattern

### Jenkins publish flow (example)
1. Build artifact/image
2. Run tests and quality/security scans
3. Publish artifact to Nexus hosted repo
4. Tag build with artifact version
5. Deploy from Nexus artifact (not workspace build output)

Why:
- Promotes immutability and traceability

---

## 13) Troubleshooting

## 13.1 Service not starting
```bash
sudo systemctl status nexus
sudo journalctl -u nexus -n 200 --no-pager
```

Nexus logs:
```bash
ls -lah /opt/sonatype-work/nexus3/log
tail -f /opt/sonatype-work/nexus3/log/nexus.log
```

## 13.2 Port conflict
Check:
```bash
sudo ss -tulnp | grep 8081
```

## 13.3 Upload/auth failures
- Wrong repo URL (hosted vs group)
- Insufficient privileges
- Wrong credentials/token

## 13.4 Low disk
```bash
df -h
du -sh /opt/sonatype-work/nexus3/*
```
Apply cleanup policies and remove stale blobs carefully.

---

## 14) Security Best Practices

1. Put Nexus behind reverse proxy + TLS
2. Disable anonymous access if not needed
3. Use role-based access control
4. Rotate service account passwords/tokens
5. Keep Nexus updated (security patches)
6. Restrict network access (only required subnets)

---

## 15) Practice Tasks

1. Install Nexus and login as admin  
2. Create Maven hosted/proxy/group repos  
3. Configure Maven to use Nexus mirror  
4. Publish sample artifact to hosted repo  
5. Configure npm group and install package through cache  
6. Configure Docker hosted repo and push/pull image  
7. Create CI service account with minimal privileges  
8. Configure cleanup policy for snapshot repo  

---

## 16) Daily Nexus Cheat Sheet

```bash
# Service
sudo systemctl status nexus
sudo systemctl restart nexus

# Logs
tail -f /opt/sonatype-work/nexus3/log/nexus.log

# Backup
sudo tar -czf /tmp/nexus-backup-$(date +%F).tar.gz /opt/sonatype-work/nexus3
```

---

## Final Notes

- Nexus is foundational for enterprise CI/CD maturity.
- Treat artifact repository as critical infrastructure.
- Promote artifacts through environments instead of rebuilding each stage.