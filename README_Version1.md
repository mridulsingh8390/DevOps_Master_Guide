# DevOps Zero-to-Project Guide (Ubuntu 24.04) — Git, Jenkins, Ansible, Vault, Docker (DCA), Terraform, Prometheus, Grafana, Kubernetes

> One-file, step-by-step learning + implementation guide.  
> Target OS: **Ubuntu 24.04**  
> Audience: Beginner to Intermediate DevOps engineer

---

## Table of Contents

1. [Learning Path](#learning-path)
2. [Base Setup](#base-setup)
3. [Git — Full Practical Commands](#git--full-practical-commands)
4. [Docker — DCA-Oriented Hands-on](#docker--dca-oriented-hands-on)
5. [Jenkins — Installation + Pipeline](#jenkins--installation--pipeline)
6. [Ansible + Roles + Vault](#ansible--roles--vault)
7. [Terraform — Commands + Example](#terraform--commands--example)
8. [Kubernetes (kubeadm) — Install + Configure](#kubernetes-kubeadm--install--configure)
9. [Prometheus + Grafana — Install + Configure](#prometheus--grafana--install--configure)
10. [End-to-End Mini Project](#end-to-end-mini-project)
11. [Troubleshooting Cookbook](#troubleshooting-cookbook)
12. [Interview Q&A Practice](#interview-qa-practice)
13. [Cheat Sheets](#cheat-sheets)

---

## Learning Path

Recommended order:
1. Git
2. Docker
3. Jenkins
4. Ansible + Vault
5. Terraform
6. Kubernetes
7. Prometheus + Grafana
8. End-to-End project

---

## Base Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim jq unzip tree net-tools htop ca-certificates gnupg lsb-release software-properties-common apt-transport-https
```

Create workspace:

```bash
mkdir -p ~/devops-lab && cd ~/devops-lab
```

---

## Git — Full Practical Commands

## Configure identity

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"
```

## SSH setup

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
# Add to GitHub/GitLab
ssh -T git@github.com
```

## Everyday workflow

```bash
git clone <repo_url>
cd <repo>
git checkout -b feature/login
git add .
git commit -m "feat: add login endpoint"
git push -u origin feature/login
```

## Merge/rebase

```bash
git checkout main
git pull origin main
git checkout feature/login
git rebase main
git push --force-with-lease
```

## Undo and recovery

```bash
git status
git restore <file>
git restore --staged <file>
git reset --soft HEAD~1
git reset --hard HEAD~1
git reflog
git cherry-pick <sha>
```

## Useful logs

```bash
git log --oneline --graph --decorate --all
```

---

## Docker — DCA-Oriented Hands-on

## Install Docker Engine

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

## Build image example

Create app:

```bash
mkdir -p ~/devops-lab/docker-demo && cd ~/devops-lab/docker-demo
cat > app.py <<'EOF'
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello from Docker Demo!"
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF

cat > requirements.txt <<'EOF'
flask==3.0.3
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python","app.py"]
EOF
```

Build/run:

```bash
docker build -t flask-demo:v1 .
docker run -d --name flask-demo -p 5000:5000 flask-demo:v1
curl http://localhost:5000
```

## DCA topics checklist (must practice)

- Image lifecycle: build/tag/push/pull/prune
- Volumes and bind mounts
- Bridge/host/overlay networking
- Compose
- Swarm init/service scale/update/rollback
- Security: least privilege, image scanning, secrets
- Troubleshooting: logs, inspect, exec, events, stats

---

## Jenkins — Installation + Pipeline

## Install Jenkins

```bash
sudo apt update
sudo apt install -y openjdk-21-jre fontconfig

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

Get admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open: `http://<server-ip>:8080`

## Install recommended Jenkins plugins

- Git
- Pipeline
- Docker Pipeline
- Credentials Binding
- SSH Agent
- Kubernetes CLI (optional)
- Blue Ocean (optional)

## Jenkinsfile example (build + push + deploy placeholder)

```groovy
pipeline {
  agent any
  environment {
    IMAGE_NAME = "flask-demo"
    IMAGE_TAG  = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build Docker Image') {
      steps { sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .' }
    }
    stage('Test') {
      steps { sh 'docker run --rm $IMAGE_NAME:$IMAGE_TAG python -c "print(\"test ok\")"' }
    }
    stage('Deploy (placeholder)') {
      steps { sh 'echo "Deploy to Kubernetes here"' }
    }
  }
}
```

---

## Ansible + Roles + Vault

## Install Ansible

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

## Inventory (`inventory.ini`)

```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[monitoring]
192.168.1.20 ansible_user=ubuntu
```

## Ad-hoc commands

```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
```

## Playbook (`site.yml`)

```yaml
- name: Configure web servers
  hosts: web
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

Run:

```bash
ansible-playbook -i inventory.ini site.yml
```

## Roles structure

```bash
mkdir -p ~/devops-lab/ansible && cd ~/devops-lab/ansible
ansible-galaxy init roles/nginx
```

`roles/nginx/tasks/main.yml`:

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Deploy index page
  copy:
    content: "Hello from Ansible role!"
    dest: /var/www/html/index.html
  notify: restart nginx
```

`roles/nginx/handlers/main.yml`:

```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

`role-site.yml`:

```yaml
- hosts: web
  become: yes
  roles:
    - nginx
```

## Vault example

Create encrypted file:

```bash
mkdir -p group_vars/all
ansible-vault create group_vars/all/vault.yml
```

Inside:

```yaml
db_password: "SuperSecretPassword!"
api_key: "abc123"
```

Use in playbook:

```yaml
- hosts: web
  tasks:
    - debug:
        msg: "DB Password is {{ db_password }}"
```

Run:

```bash
ansible-playbook -i inventory.ini role-site.yml --ask-vault-pass
```

---

## Terraform — Commands + Example

## Install Terraform

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y terraform
terraform -version
```

## Core commands

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform destroy
```

## Local demo (`main.tf`)

```hcl
terraform {
  required_version = ">= 1.5.0"
}
provider "local" {}

resource "local_file" "demo" {
  filename = "terraform-demo.txt"
  content  = "Terraform works!"
}
```

Run:

```bash
mkdir -p ~/devops-lab/terraform-demo && cd ~/devops-lab/terraform-demo
terraform init
terraform apply -auto-approve
cat terraform-demo.txt
terraform destroy -auto-approve
```

---

## Kubernetes (kubeadm) — Install + Configure

## System prep

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

## Install containerd

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install kubelet/kubeadm/kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initialize cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Set kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Calico CNI:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

Single-node scheduling (lab only):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Test app:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods,svc -o wide
```

---

## Prometheus + Grafana — Install + Configure

## Prometheus install

```bash
sudo useradd --no-create-home --shell /bin/false prometheus || true
cd /tmp
PROM_VER="2.54.1"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz
tar -xvf prometheus-${PROM_VER}.linux-amd64.tar.gz
cd prometheus-${PROM_VER}.linux-amd64

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Service file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

## Node Exporter

```bash
cd /tmp
NODE_VER="1.8.2"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_VER}/node_exporter-${NODE_VER}.linux-amd64.tar.gz
tar -xvf node_exporter-${NODE_VER}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_VER}.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter || true
```

Service:

```bash
cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Add scrape job:

```bash
cat <<EOF | sudo tee /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
EOF

sudo systemctl restart prometheus
```

## Grafana install

```bash
cd /tmp
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo apt install -y adduser libfontconfig1 musl
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
```

Open:
- Prometheus: `http://<ip>:9090`
- Grafana: `http://<ip>:3000` (admin/admin)

Grafana datasource:
- Connections → Data Sources → Prometheus → URL `http://localhost:9090`

---

## End-to-End Mini Project

Goal:
1. Developer pushes code to Git repo
2. Jenkins builds Docker image + runs tests
3. Jenkins deploys app to Kubernetes
4. Prometheus scrapes metrics
5. Grafana visualizes dashboard
6. Infra/config managed by Terraform + Ansible

## Suggested project structure

```text
devops-project/
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── jenkins/
│   └── Jenkinsfile
├── ansible/
│   ├── inventory.ini
│   ├── site.yml
│   └── roles/
├── terraform/
│   └── main.tf
└── README.md
```

## Kubernetes manifests example

`k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-demo
  template:
    metadata:
      labels:
        app: flask-demo
    spec:
      containers:
      - name: flask-demo
        image: flask-demo:v1
        ports:
        - containerPort: 5000
```

`k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-demo-svc
spec:
  type: NodePort
  selector:
    app: flask-demo
  ports:
    - port: 80
      targetPort: 5000
```

Deploy:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get all
```

---

## Troubleshooting Cookbook

## Git

- Auth failed → verify SSH key and remote URL
- Detached HEAD → `git checkout main`

## Docker

```bash
docker ps -a
docker logs <container>
docker inspect <container>
docker exec -it <container> sh
docker system df
```

Common issue: permission denied on Docker socket  
Fix:
```bash
sudo usermod -aG docker $USER && newgrp docker
```

## Jenkins

```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
```

- 8080 blocked → open firewall/security group
- Plugin dependency issue → restart Jenkins and re-install plugin

## Ansible

```bash
ansible all -i inventory.ini -m ping -vvv
```

- SSH unreachable → check key/user/firewall
- sudo failed → ensure `--become` and sudo permissions

## Terraform

```bash
terraform validate
terraform plan
```

- Provider download issue → check internet/proxy
- State lock issue → unlock carefully (`terraform force-unlock`)

## Kubernetes

```bash
kubectl get nodes
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
sudo journalctl -u kubelet -xe
```

- `NotReady` node → CNI not installed or container runtime issue
- `ImagePullBackOff` → wrong image/tag or registry auth
- `CrashLoopBackOff` → app crash, check logs/env vars

## Prometheus/Grafana

```bash
sudo systemctl status prometheus
sudo systemctl status grafana-server
curl -s http://localhost:9090/-/healthy
```

- No data in Grafana → datasource URL wrong or target down
- Target down in Prometheus → check job config and service status

---

## Interview Q&A Practice

### Git
- Rebase vs Merge?
- `reset` vs `revert`?
- How to recover deleted commit?

### Docker/DCA
- Difference: image vs container?
- Bridge vs host network?
- CMD vs ENTRYPOINT?
- Volumes vs bind mounts?
- Swarm service rolling update steps?

### Jenkins
- Declarative vs Scripted pipeline?
- How to secure credentials?
- How to run parallel stages?

### Ansible
- Idempotency meaning?
- Variable precedence?
- Role vs playbook?
- Vault workflow?

### Terraform
- `plan` vs `apply`?
- State file purpose?
- Remote backend benefits?
- Taint/import/use cases?

### Kubernetes
- Deployment vs StatefulSet?
- Service types?
- ConfigMap vs Secret?
- Liveness vs readiness probe?
- Why etcd backup is critical?

### Monitoring
- Pull vs push monitoring?
- What is scrape interval?
- Alert fatigue handling?
- Golden signals (latency, traffic, errors, saturation)?

---

## Cheat Sheets

## Kubernetes quick

```bash
kubectl get nodes,pods,svc -A
kubectl describe pod <pod> -n <ns>
kubectl logs -f <pod> -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl rollout status deploy/<name>
kubectl rollout undo deploy/<name>
```

## Docker quick

```bash
docker build -t app:v1 .
docker run -d -p 8080:80 app:v1
docker logs -f <container>
docker exec -it <container> sh
docker image prune -f
```

## Terraform quick

```bash
terraform init && terraform fmt && terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform destroy
```

## Ansible quick

```bash
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml
ansible-vault create secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

---

## Final Notes

- Practice each section in isolated VM/sandbox
- Never hardcode production secrets
- Build one complete project and document it in Git
- Repeat troubleshooting commands until they become muscle memory

**You now have one complete DevOps learning + implementation file.**