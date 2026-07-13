# SonarQube Complete Guide (Code Quality + Security)
## For DevOps Engineers (local/on-prem, CI/CD integration)

> This file is detailed and practical.
> Covers install, project scan, quality gates, Jenkins integration, troubleshooting.

---

## 1) What is SonarQube?

SonarQube is a code quality platform for:
- Static code analysis
- Bug/code smell detection
- Security hotspot/vulnerability visibility
- Quality gate enforcement in CI/CD

## Why use it?
- Prevent bad code from merging/deploying
- Track technical debt trend
- Enforce team-wide code standards

---

## 2) Prerequisites

- Java 17 (for SonarQube server versions that require it)
- PostgreSQL (recommended for production; avoid embedded DB for production)
- Minimum resources for lab: 2 CPU, 4 GB RAM

---

## 3) Install SonarQube (Linux, zip-based)

## 3.1 Create user
```bash
sudo useradd -m -d /opt/sonarqube -s /bin/bash sonarqube || true
```

## 3.2 Install Java + unzip
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk unzip wget
java -version
```

## 3.3 Download SonarQube
```bash
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.0.92116.zip
sudo unzip sonarqube-10.6.0.92116.zip -d /opt
sudo mv /opt/sonarqube-10.6.0.92116 /opt/sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube
```

---

## 4) Install PostgreSQL and DB setup

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
sudo -u postgres psql
```

Inside psql:
```sql
CREATE DATABASE sonarqube;
CREATE USER sonar WITH ENCRYPTED PASSWORD 'StrongPassword123!';
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
```

---

## 5) Configure SonarQube

Edit:
`/opt/sonarqube/conf/sonar.properties`

```properties
sonar.jdbc.username=sonar
sonar.jdbc.password=StrongPassword123!
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube

sonar.web.host=0.0.0.0
sonar.web.port=9000
```

---

## 6) systemd service

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=network.target postgresql.service

[Service]
Type=forking
User=sonarqube
Group=sonarqube
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF
```

Start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube
sudo systemctl status sonarqube
```

Open UI:
- `http://<server-ip>:9000`
- Default: `admin/admin` (forced password change)

---

## 7) Install SonarScanner CLI

```bash
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-6.1.0.4477-linux-x64.zip
sudo unzip sonar-scanner-6.1.0.4477-linux-x64.zip -d /opt
sudo ln -s /opt/sonar-scanner-6.1.0.4477-linux-x64/bin/sonar-scanner /usr/local/bin/sonar-scanner
sonar-scanner --version
```

---

## 8) Scan a Project (manual)

Create `sonar-project.properties` in project root:

```properties
sonar.projectKey=myapp
sonar.projectName=My App
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.host.url=http://localhost:9000
sonar.token=<SONAR_TOKEN>
```

Run scan:
```bash
sonar-scanner
```

Check report in SonarQube UI.

---

## 9) Quality Gates (very important)

## What
Quality Gate is pass/fail policy (e.g., no new critical bugs, coverage threshold).

## Why
Block bad code from release pipeline.

Common gate criteria:
- No new blocker/critical issues
- Minimum coverage on new code
- Security rating threshold

---

## 10) Jenkins Integration

## 10.1 Jenkins plugins
- SonarQube Scanner
- Quality Gates plugin (or webhook-based wait)

## 10.2 Jenkins global config
- Add SonarQube server URL
- Add token credential

## 10.3 Pipeline example
```groovy
pipeline {
  agent any
  stages {
    stage('Sonar Scan') {
      steps {
        withSonarQubeEnv('sonarqube-server') {
          sh 'sonar-scanner -Dsonar.projectKey=myapp -Dsonar.sources=. -Dsonar.host.url=http://sonarqube:9000 -Dsonar.token=$SONAR_TOKEN'
        }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
  }
}
```

---

## 11) Branch and PR analysis (concept)

With developer edition+ features:
- Branch analysis
- Pull request decoration
- Inline quality feedback in PR systems

---

## 12) Security and Governance Tips

1. Use dedicated DB (PostgreSQL)  
2. Rotate tokens and use scoped tokens  
3. Restrict admin access  
4. Back up DB + config regularly  
5. Integrate quality gate into mandatory CI checks  

---

## 13) Troubleshooting

## 13.1 SonarQube not starting
```bash
sudo systemctl status sonarqube
sudo journalctl -u sonarqube -n 200 --no-pager
```

## 13.2 Elasticsearch bootstrap/system limits issues
Set kernel params if needed:
```bash
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```

Persist in `/etc/sysctl.conf` where appropriate.

## 13.3 DB connection issue
- Verify PostgreSQL service
- Verify `sonar.jdbc.*` values
- Verify user/password/db grants

## 13.4 Scanner auth failure
- Token invalid/expired
- Wrong `sonar.host.url`
- Proxy/network restrictions

---

## 14) Practice tasks

1. Install SonarQube + PostgreSQL  
2. Scan sample repo with SonarScanner  
3. Create and enforce custom quality gate  
4. Integrate with Jenkins pipeline  
5. Fail build when quality gate fails  

---

## 15) Daily cheat sheet

```bash
# Service
sudo systemctl status sonarqube
sudo journalctl -u sonarqube -f

# Scanner
sonar-scanner -Dsonar.projectKey=myapp -Dsonar.sources=. -Dsonar.host.url=http://localhost:9000 -Dsonar.token=<token>
```

---

## Final Notes

- SonarQube + Quality Gate is one of the most practical DevSecOps controls in CI.
- Start simple (bugs/vulns/smells), then add coverage and stricter gates gradually.