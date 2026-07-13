# Jenkins Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Jenkins and why it matters  
2. Installation (Ubuntu 24.04)  
3. Initial setup and security basics  
4. Jenkins architecture fundamentals  
5. Job types (Freestyle, Pipeline, Multibranch)  
6. Pipeline syntax (Declarative + Scripted)  
7. Jenkinsfile deep examples  
8. Credentials and secret handling  
9. Parameters, environment variables, and build tools  
10. Integrating Git and webhooks  
11. Docker usage in Jenkins pipelines  
12. Agents/Nodes (controller-agent model)  
13. Backup and restore  
14. Jenkins CLI basics  
15. Plugin management best practices  
16. Logs, monitoring, and troubleshooting  
17. Hardening and production tips  
18. Practice lab tasks  
19. Daily command cheat sheet

---

## 1) What is Jenkins and Why it Matters

## What
Jenkins is an automation server used to run CI/CD pipelines.

## Why
- Automates build/test/deploy
- Reduces manual release errors
- Enforces repeatable workflows
- Integrates with Git, Docker, Kubernetes, notifications, and more

---

## 2) Installation (Ubuntu 24.04)

## 2.1 Install Java (required)
```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

## 2.2 Add Jenkins apt repository and install Jenkins
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
```

## 2.3 Start and enable service
```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

## 2.4 Verify port
```bash
sudo ss -tulnp | grep 8080
```

---

## 3) Initial Setup and Security Basics

## 3.1 Get unlock password
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:
- `http://<server-ip>:8080`

## 3.2 Initial web setup checklist
1. Paste initial admin password
2. Install suggested plugins
3. Create admin user
4. Configure Jenkins URL

## 3.3 Basic security settings (important)
- Enable Matrix/Role-based authorization
- Disable anonymous read access
- Enforce strong password policy
- Restrict script console access to admins

---

## 4) Jenkins Architecture Fundamentals

- **Controller (Master)**: schedules builds, stores configuration
- **Agents (Nodes)**: execute build tasks
- **Executor**: slot to run one build at a time
- **Workspace**: per-job directory for checked-out code
- **JENKINS_HOME**: persistent Jenkins data (`/var/lib/jenkins`)

Useful paths:
```bash
ls -lah /var/lib/jenkins
ls -lah /var/log/jenkins
```

---

## 5) Job Types

## 5.1 Freestyle Project
Simple UI-driven job configuration.

## 5.2 Pipeline Job
Pipeline as code via Jenkinsfile.

## 5.3 Multibranch Pipeline
Automatically discovers branches/PRs and runs Jenkinsfile per branch.

---

## 6) Pipeline Syntax Basics

## 6.1 Declarative pipeline (preferred for most teams)

```groovy
pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  stages {
    stage('Build') {
      steps {
        sh 'echo Build step'
      }
    }
    stage('Test') {
      steps {
        sh 'echo Test step'
      }
    }
  }
  post {
    success { echo 'Success' }
    failure { echo 'Failure' }
    always  { echo 'Always runs' }
  }
}
```

## 6.2 Scripted pipeline (more flexible)

```groovy
node {
  stage('Build') {
    sh 'echo Build'
  }
  stage('Test') {
    sh 'echo Test'
  }
}
```

---

## 7) Jenkinsfile Deep Examples

## 7.1 Practical CI pipeline with parameters + artifacts

```groovy
pipeline {
  agent any

  parameters {
    string(name: 'APP_ENV', defaultValue: 'dev', description: 'Target environment')
    booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests?')
  }

  environment {
    APP_NAME = "demo-app"
    BUILD_TAG = "${BUILD_NUMBER}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install') {
      steps { sh 'echo Installing dependencies' }
    }

    stage('Build') {
      steps { sh 'echo Building ${APP_NAME}:${BUILD_TAG}' }
    }

    stage('Test') {
      when { expression { return params.RUN_TESTS } }
      steps { sh 'echo Running tests' }
    }

    stage('Package') {
      steps {
        sh 'mkdir -p artifacts && echo "build ${BUILD_TAG}" > artifacts/build.txt'
        archiveArtifacts artifacts: 'artifacts/**', fingerprint: true
      }
    }

    stage('Deploy') {
      steps {
        sh 'echo Deploying to ${APP_ENV}'
      }
    }
  }

  post {
    success {
      echo 'Pipeline success'
    }
    failure {
      echo 'Pipeline failed'
    }
    always {
      cleanWs()
    }
  }
}
```

---

## 8) Credentials and Secret Handling

## 8.1 Why credentials matter
Never hardcode passwords/tokens in Jenkinsfile or repo.

## 8.2 Credential types
- Username/Password
- Secret text (token/API key)
- SSH private key
- Secret file (cert, kubeconfig)

## 8.3 Use secret text in pipeline
```groovy
withCredentials([string(credentialsId: 'api-token', variable: 'API_TOKEN')]) {
  sh 'echo "Token length: ${#API_TOKEN}"'
}
```

## 8.4 Use username/password
```groovy
withCredentials([usernamePassword(credentialsId: 'docker-creds',
  usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
  sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
}
```

---

## 9) Parameters, Env Vars, Tools

## 9.1 Build parameters
Use `parameters {}` block in Declarative pipeline.

## 9.2 Environment variables
```groovy
environment {
  ENV_NAME = "dev"
}
```

## 9.3 Built-in env vars examples
- `BUILD_NUMBER`
- `JOB_NAME`
- `WORKSPACE`
- `GIT_COMMIT`

Inspect:
```groovy
sh 'printenv | sort'
```

---

## 10) Integrating Git and Webhooks

## 10.1 Git checkout via SCM
```groovy
checkout scm
```

## 10.2 Manual Git in shell
```groovy
sh 'git rev-parse --short HEAD'
```

## 10.3 Webhook flow
1. Push to Git provider
2. Webhook triggers Jenkins endpoint
3. Jenkins starts pipeline automatically

Typical endpoint:
- `http://jenkins.example.com/github-webhook/`

---

## 11) Docker in Jenkins

## 11.1 Allow Jenkins user to run Docker
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
id jenkins
```

## 11.2 Pipeline docker build/run example
```groovy
pipeline {
  agent any
  stages {
    stage('Build Image') {
      steps {
        sh 'docker build -t demo:${BUILD_NUMBER} .'
      }
    }
    stage('Smoke Test') {
      steps {
        sh 'docker run --rm demo:${BUILD_NUMBER} echo "ok"'
      }
    }
  }
}
```

---

## 12) Agents/Nodes

## 12.1 Why agents
Separate build execution from controller for scale and isolation.

## 12.2 Node labels
Assign jobs using:
```groovy
agent { label 'linux-docker' }
```

## 12.3 Check nodes in script
```groovy
node('linux-docker') {
  sh 'uname -a'
}
```

---

## 13) Backup and Restore

## 13.1 Backup
```bash
sudo systemctl stop jenkins
sudo tar -czf /tmp/jenkins_home_backup_$(date +%F_%H-%M).tar.gz /var/lib/jenkins
sudo systemctl start jenkins
```

## 13.2 Restore
```bash
sudo systemctl stop jenkins
sudo tar -xzf /tmp/jenkins_home_backup_YYYY-MM-DD_HH-MM.tar.gz -C /
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo systemctl start jenkins
```

## 13.3 What backup includes
- Jobs
- Plugins
- Credentials (encrypted)
- Build history
- Global config

---

## 14) Jenkins CLI Basics

## 14.1 Download CLI jar
```bash
wget http://localhost:8080/jnlpJars/jenkins-cli.jar
```

## 14.2 Authenticate and list jobs
```bash
java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:apitoken list-jobs
```

## 14.3 Build job from CLI
```bash
java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:apitoken build my-job -s -v
```

---

## 15) Plugin Management Best Practices

- Install only required plugins
- Keep plugin versions updated
- Test plugin updates in staging
- Avoid obsolete/unmaintained plugins

Check plugin updates from UI:
- Manage Jenkins → Plugins → Updates

---

## 16) Logs, Monitoring, Troubleshooting

## 16.1 Service logs
```bash
sudo journalctl -u jenkins -f
sudo journalctl -u jenkins -n 200 --no-pager
```

## 16.2 Health checks
```bash
sudo systemctl status jenkins
curl -I http://localhost:8080/login
sudo ss -tulnp | grep 8080
```

## 16.3 Common issues and fixes

### Issue: Jenkins not starting
- Check Java version
- Check disk space
```bash
df -h
```

### Issue: Port already in use
```bash
sudo ss -tulnp | grep 8080
```
Change port in:
`/etc/default/jenkins` (depending package version).

### Issue: OutOfMemory
- Increase Java opts in Jenkins service env
- Restart service and monitor logs

### Issue: Pipeline fails due to missing tool
- Install tool on agent
- Add tool config in Manage Jenkins → Tools

---

## 17) Hardening and Production Tips

1. Put Jenkins behind reverse proxy (Nginx/Apache)
2. Use HTTPS
3. Enable regular backups
4. Use least-privileged credentials
5. Restrict who can configure jobs
6. Use agents for builds (avoid heavy builds on controller)
7. Audit plugins regularly
8. Rotate tokens and secrets

---

## 18) Practice Lab Tasks

1. Install Jenkins and create admin user
2. Create Freestyle job that runs `echo hello`
3. Create pipeline job from Jenkinsfile
4. Add a build parameter and use it
5. Store token in credentials and consume safely
6. Build Docker image in pipeline
7. Backup and restore Jenkins home in lab VM
8. Simulate pipeline failure and debug using logs

---

## 19) Daily Jenkins Cheat Sheet

```bash
# Service
sudo systemctl status jenkins
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f

# Validate access
curl -I http://localhost:8080/login

# Backup quick
sudo tar -czf /tmp/jenkins_backup_$(date +%F).tar.gz /var/lib/jenkins
```

---

## Final Notes

- Prefer pipeline-as-code over click-based config.
- Keep secrets in Jenkins Credentials, never in Git.
- Backup `/var/lib/jenkins` regularly.
- Monitor plugin and Java compatibility on updates.