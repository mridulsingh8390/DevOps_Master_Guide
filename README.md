# DevOps V5 Ultra-Long Command Encyclopedia (Ubuntu 24.04, Local/On-Prem Only)

> Single-file, step-by-step, command-first handbook for:
- Git
- Jenkins
- Ansible (Playbook, Roles, Vault)
- Docker (DCA scope + Dockerfiles)
- Terraform
- Kubernetes (kubeadm + kubectl catalog)
- Prometheus
- Grafana
- Troubleshooting + interview + validation checklist

---

## 0. Base Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
curl wget git vim nano jq yq unzip zip tree htop btop net-tools dnsutils telnet nmap \
ca-certificates gnupg lsb-release software-properties-common apt-transport-https \
build-essential python3 python3-pip python3-venv rsync tmux
```

```bash
mkdir -p ~/devops-lab/{git,jenkins,ansible,docker,terraform,k8s,monitoring,project,notes}
cd ~/devops-lab
```

---

## 1. Git — Ultra Command Catalog

## Config
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
git config --global color.ui auto
git config --global fetch.prune true
git config --global rerere.enabled true
git config --list
```

## SSH
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com
```

## Repo lifecycle
```bash
git init
git clone <repo_url>
git clone -b <branch> <repo_url>
git remote -v
git remote add origin <repo_url>
git remote set-url origin <new_url>
git fetch --all --prune
git pull
git pull --rebase
git push -u origin main
```

## Branching
```bash
git branch
git branch -a
git switch -c feature/x
git checkout -b feature/x
git switch main
git merge feature/x
git branch -d feature/x
git branch -D feature/x
git push origin --delete feature/x
```

## Commits/staging
```bash
git status
git add .
git add -p
git commit -m "feat: message"
git commit --amend
```

## Rebase/cherry-pick
```bash
git rebase main
git rebase -i HEAD~5
git rebase --continue
git rebase --abort
git cherry-pick <sha>
git cherry-pick --abort
```

## Undo/recovery
```bash
git restore <file>
git restore --staged <file>
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
git revert <sha>
git reflog
```

## Stash
```bash
git stash
git stash push -m "wip"
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}
git stash pop
git stash drop stash@{0}
```

## Logs/diff/tags
```bash
git log --oneline --graph --decorate --all
git show <sha>
git diff
git diff --staged
git blame <file>
git tag -a v1.0.0 -m "release"
git push --tags
```

---

## 2. Jenkins — Installation + Admin + Pipeline + Backup

## Install
```bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

Get initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Service ops:
```bash
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f
```

Backup/restore:
```bash
sudo systemctl stop jenkins
sudo tar -czf /tmp/jenkins_home_$(date +%F).tar.gz /var/lib/jenkins
sudo systemctl start jenkins
```

```bash
sudo systemctl stop jenkins
sudo tar -xzf /tmp/jenkins_home_YYYY-MM-DD.tar.gz -C /
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo systemctl start jenkins
```

Jenkinsfile:
```groovy
pipeline {
  agent any
  options { timestamps() }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build')    { steps { sh 'docker build -t demo:${BUILD_NUMBER} .' } }
    stage('Test')     { steps { sh 'echo test' } }
    stage('Deploy')   { steps { sh 'echo deploy' } }
  }
}
```

---

## 3. Ansible — Adhoc + Playbooks + Roles + Vault

## Install
```bash
sudo apt install -y ansible
ansible --version
```

Inventory (`inventory.ini`)
```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[db]
192.168.1.20 ansible_user=ubuntu
```

Adhoc:
```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m shell -a "df -h"
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
ansible all -i inventory.ini -m service -a "name=nginx state=started enabled=yes" --become
ansible all -i inventory.ini -m user -a "name=devops state=present groups=sudo append=yes" --become
ansible all -i inventory.ini -m setup
```

Playbook (`site.yml`)
```yaml
- hosts: web
  become: yes
  tasks:
    - apt:
        name: nginx
        state: present
        update_cache: yes
    - service:
        name: nginx
        state: started
        enabled: yes
```

Run:
```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml --limit web -vvv
```

Roles:
```bash
ansible-galaxy init roles/nginx
ansible-galaxy list
```

Vault:
```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey secrets.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

---

## 4. Docker — DCA Full + Dockerfile Encyclopedia

## Install Docker
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

Core:
```bash
docker version
docker info
docker ps
docker ps -a
docker run -d --name web -p 8080:80 nginx
docker logs -f web
docker exec -it web sh
docker inspect web
docker stats
docker events
docker stop web && docker rm web
```

Images:
```bash
docker images
docker pull nginx:latest
docker build -t myapp:v1 .
docker tag myapp:v1 myrepo/myapp:v1
docker push myrepo/myapp:v1
docker rmi myapp:v1
docker image prune -f
docker system prune -a
```

Volumes/networks:
```bash
docker volume create app-vol
docker volume ls
docker network create app-net
docker network ls
docker run -d --name app --network app-net nginx
docker network inspect app-net
```

Compose:
```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
docker compose down -v
```

Swarm:
```bash
docker swarm init
docker service create --name web --replicas 2 -p 8080:80 nginx
docker service ls
docker service scale web=4
docker service update --image nginx:alpine web
docker service rollback web
docker service rm web
docker swarm leave --force
```

Secrets/config:
```bash
echo "mypassword" | docker secret create db_pass -
docker secret ls
docker secret rm db_pass
```

Dockerfile examples:

Python:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python","app.py"]
```

Node:
```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
```

Java multi-stage:
```dockerfile
FROM maven:3.9.8-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

Go multi-stage:
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app main.go

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /src/app /app
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Nginx static:
```dockerfile
FROM nginx:alpine
COPY ./site /usr/share/nginx/html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

`.dockerignore`
```gitignore
.git
.gitignore
node_modules/
__pycache__/
*.log
.env
dist/
build/
```

---

## 5. Terraform — Full Local Command Workflow

## Install
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform version
```

Core:
```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform plan -out=tfplan
terraform apply tfplan
terraform apply -auto-approve
terraform destroy -auto-approve
```

State/workspace:
```bash
terraform state list
terraform state show <resource>
terraform state pull
terraform import <resource_address> <id>
terraform force-unlock <LOCK_ID>
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
terraform console
terraform graph
```

Local sample (`main.tf`):
```hcl
terraform {
  required_version = ">= 1.5.0"
}
provider "local" {}

resource "local_file" "demo" {
  filename = "demo.txt"
  content  = "Terraform Local Works"
}
```

---

## 6. Kubernetes — kubeadm + kubectl Catalog

## Install
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

sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Init cluster:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Core kubectl:
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
kubectl create ns dev
kubectl get all -A
kubectl api-resources
kubectl config get-contexts
kubectl config current-context
```

Workloads/services:
```bash
kubectl create deployment nginx --image=nginx
kubectl scale deploy nginx --replicas=3
kubectl set image deploy/nginx nginx=nginx:alpine
kubectl rollout status deploy/nginx
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx
kubectl expose deploy nginx --port=80 --type=NodePort
kubectl get svc
kubectl port-forward svc/nginx 8080:80
```

Debug:
```bash
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs -f <pod> -n <ns>
kubectl logs <pod> --previous -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl get events -A --sort-by=.lastTimestamp
kubectl top nodes
kubectl top pods -A
```

Config/secret:
```bash
kubectl create configmap app-config --from-literal=ENV=prod
kubectl create secret generic app-secret --from-literal=password=Pass@123
kubectl get configmap,secret
```

Node ops:
```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

Lab reset:
```bash
sudo kubeadm reset -f
rm -rf ~/.kube
```

---

## 7. Prometheus — Install + Config + PromQL

Install:
```bash
sudo useradd --no-create-home --shell /bin/false prometheus || true
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
tar -xvf prometheus-2.54.1.linux-amd64.tar.gz
cd prometheus-2.54.1.linux-amd64
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Service:
```bash
cat <<EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=network-online.target
Wants=network-online.target
[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus --web.enable-lifecycle
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

Node exporter:
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter || true
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
```

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

Prometheus config:
```bash
cat <<EOF | sudo tee /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
- job_name: prometheus
  static_configs:
  - targets: ['localhost:9090']
- job_name: node
  static_configs:
  - targets: ['localhost:9100']
EOF
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

Ops:
```bash
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets | jq
curl -X POST http://localhost:9090/-/reload
```

PromQL:
```promql
up
up{job="node"}
rate(node_cpu_seconds_total[5m])
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
(node_memory_MemTotal_bytes-node_memory_MemAvailable_bytes)/node_memory_MemTotal_bytes*100
rate(node_network_receive_bytes_total[5m])
```

---

## 8. Grafana — Install + Configure

Install:
```bash
cd /tmp
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo apt install -y adduser libfontconfig1 musl
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
```

Ops:
```bash
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -f
sudo grafana-cli admin reset-admin-password 'StrongPassword@123'
```

Access:
- `http://<ip>:3000`
- Default: `admin/admin`

Datasource:
- Prometheus URL: `http://localhost:9090`

---

## 9. End-to-End Local Mini Project

Structure:
```bash
mkdir -p ~/devops-lab/project/{app,k8s,jenkins,ansible,terraform}
```

`app/app.py`
```python
from flask import Flask
app=Flask(__name__)
@app.route("/")
def home():
    return "Hello DevOps"
if __name__=="__main__":
    app.run(host="0.0.0.0",port=5000)
```

`app/requirements.txt`
```txt
flask==3.0.3
```

`app/Dockerfile`
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python","app.py"]
```

Build/run:
```bash
cd ~/devops-lab/project/app
docker build -t flask-demo:v1 .
docker run -d --name flask-demo -p 5000:5000 flask-demo:v1
curl http://localhost:5000
```

K8s deploy:
`k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: flask-demo}
spec:
  replicas: 2
  selector: {matchLabels: {app: flask-demo}}
  template:
    metadata: {labels: {app: flask-demo}}
    spec:
      containers:
      - name: flask-demo
        image: flask-demo:v1
        imagePullPolicy: IfNotPresent
        ports: [{containerPort: 5000}]
```

`k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata: {name: flask-demo-svc}
spec:
  type: NodePort
  selector: {app: flask-demo}
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30080
```

```bash
kubectl apply -f ~/devops-lab/project/k8s/deployment.yaml
kubectl apply -f ~/devops-lab/project/k8s/service.yaml
kubectl get all
```

---

## 10. Troubleshooting Matrix

## OS/Network
```bash
ip a
ip r
ss -tulnp
sudo ufw status
ping -c 4 8.8.8.8
nslookup google.com
```

## Jenkins
```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
sudo ss -tulnp | grep 8080
```

## Docker
```bash
docker info
docker ps -a
docker logs <container>
sudo journalctl -u docker -f
```

## Kubernetes
```bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node <node>
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
sudo journalctl -u kubelet -xe
```

## Prometheus/Grafana
```bash
sudo systemctl status prometheus
sudo systemctl status grafana-server
curl -s http://localhost:9090/-/healthy
curl -I http://localhost:3000/login
```

## Ansible
```bash
ansible all -i inventory.ini -m ping -vvv
ansible-inventory -i inventory.ini --graph
```

## Terraform
```bash
terraform validate
terraform plan
terraform state list
```

---

## 11. Interview Drill (Quick)

- Git: merge vs rebase, reset vs revert, reflog recovery  
- Jenkins: declarative/scripted, credentials, agent strategies  
- Ansible: idempotency, var precedence, vault usage  
- Docker: CMD vs ENTRYPOINT, volume vs bind, networking modes  
- Terraform: state importance, workspaces, import  
- Kubernetes: Deployment vs StatefulSet, probes, service types  
- Prometheus: pull model, labels/cardinality  
- Grafana: datasource + alert/dashboard basics

---

## 12. Self-Validation Tracker (Nothing Missed)

### Git
- [ ] Config
- [ ] SSH
- [ ] Branching
- [ ] Rebase/cherry-pick
- [ ] Recovery
- [ ] Stash
- [ ] Tags

### Jenkins
- [ ] Install
- [ ] Initial unlock
- [ ] Pipeline
- [ ] Backup/restore
- [ ] Troubleshooting

### Ansible
- [ ] Inventory
- [ ] Adhoc
- [ ] Playbooks
- [ ] Roles
- [ ] Vault

### Docker
- [ ] Engine install
- [ ] Core commands
- [ ] Compose
- [ ] Swarm
- [ ] Secrets
- [ ] Dockerfile examples
- [ ] .dockerignore

### Terraform
- [ ] init/fmt/validate/plan/apply/destroy
- [ ] state/import/workspace/console

### Kubernetes
- [ ] kubeadm install/init
- [ ] CNI
- [ ] kubectl commands
- [ ] debug/reset

### Prometheus
- [ ] install/service
- [ ] node exporter
- [ ] scrape config
- [ ] promtool/promql

### Grafana
- [ ] install/service
- [ ] password reset
- [ ] datasource/dashboard


