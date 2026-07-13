# DevOps Complete Guide (Ubuntu 24.04, Local/On-Prem)
## Git • Jenkins • Ansible • Docker • Terraform • Kubernetes • Prometheus • Grafana

## How to use this guide
Each section includes:
- What/Why
- Commands
- Verification
- Troubleshooting

---

## 1) Base Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim nano jq yq unzip zip tree htop net-tools dnsutils \
ca-certificates gnupg lsb-release software-properties-common apt-transport-https
mkdir -p ~/devops-lab/{git,jenkins,ansible,docker,terraform,k8s,monitoring,project}
cd ~/devops-lab
```

Verification:
```bash
git --version
curl --version
jq --version
```## 2) Git

### Why
Version control, collaboration, rollback.

### Setup
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --list
```

### SSH
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

### Daily commands
```bash
git init
git clone <repo_url>
git status
git add .
git commit -m "feat: message"
git fetch --all --prune
git pull --rebase
git push
```

### Branching
```bash
git branch
git switch -c feature/x
git switch main
git merge feature/x
git branch -d feature/x
```

### Recovery
```bash
git stash
git stash list
git stash pop
git reset --soft HEAD~1
git revert <sha>
git reflog
```## 3) Jenkins

### Install
```bash
sudo apt install -y fontconfig openjdk-21-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update && sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Service/logs
```bash
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f
```

### Jenkinsfile
```groovy
pipeline {
  agent any
  stages {
    stage('Checkout'){ steps{ checkout scm } }
    stage('Build'){ steps{ sh 'echo Build' } }
    stage('Test'){ steps{ sh 'echo Test' } }
    stage('Deploy'){ steps{ sh 'echo Deploy' } }
  }
}
```## 4) Ansible

### Install
```bash
sudo apt install -y ansible
ansible --version
```

### Inventory (`inventory.ini`)
```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu
```

### Adhoc
```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
```

### Playbook (`site.yml`)
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
```

### Roles + Vault
```bash
ansible-galaxy init roles/nginx
ansible-vault create secrets.yml
ansible-vault view secrets.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```## 5) Docker (DCA-oriented)

### Install
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

### Core commands
```bash
docker version
docker info
docker ps -a
docker run -d --name web -p 8080:80 nginx
docker logs -f web
docker exec -it web sh
docker inspect web
docker stop web && docker rm web
docker image prune -f
```

### Compose
```bash
docker compose up -d
docker compose logs -f
docker compose down
```

### Swarm
```bash
docker swarm init
docker service create --name web --replicas 2 -p 8080:80 nginx
docker service ls
docker service scale web=4
docker service rollback web
docker service rm web
docker swarm leave --force
```

### Dockerfile examples
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
ENTRYPOINT ["/app"]
```

`.dockerignore`
```gitignore
.git
node_modules/
__pycache__/
*.log
.env
dist/
```## 6) Terraform

### Install
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform version
```

### Workflow
```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy
```

### State/workspace
```bash
terraform state list
terraform state show <resource>
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform console
```

Example `main.tf`:
```hcl
terraform {
  required_version = ">= 1.5.0"
}
provider "local" {}
resource "local_file" "demo" {
  filename = "demo.txt"
  content  = "Terraform works"
}
```## 7) Kubernetes (Core + Advanced Objects)

### Install (kubeadm)
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Must-know kubectl
```bash
kubectl get all -A
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs -f <pod> -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl rollout status deploy/<name> -n <ns>
kubectl rollout undo deploy/<name> -n <ns>
kubectl get events -A --sort-by=.lastTimestamp
kubectl top nodes
kubectl top pods -A
kubectl auth can-i create pods -n dev
kubectl explain deployment.spec.template.spec.containers
```## 8) Kubernetes Objects Coverage

Covered objects:
- Namespace
- Pod
- ReplicaSet
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob
- Service
- Ingress
- ConfigMap
- Secret
- PV
- PVC
- StorageClass
- ServiceAccount
- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding
- NetworkPolicy
- ResourceQuota
- LimitRange
- HPA
- PodDisruptionBudget
- PriorityClass
- RuntimeClass
- CRD

(Use the object manifests from earlier responses or ask and I’ll provide full per-object YAML file pack.)## 9) Prometheus

Install and verify:
```bash
# install prometheus + node_exporter services
curl -s http://localhost:9090/-/healthy
```

PromQL examples:
```promql
up
rate(node_cpu_seconds_total[5m])
(node_memory_MemTotal_bytes-node_memory_MemAvailable_bytes)/node_memory_MemTotal_bytes*100
```

---

## 10) Grafana

Install:
```bash
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
```

Reset admin password:
```bash
sudo grafana-cli admin reset-admin-password 'StrongPassword@123'
```

Datasource:
- Prometheus URL: `http://localhost:9090`## 11) Troubleshooting Quick Reference

### Jenkins
```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
```

### Docker
```bash
docker info
docker logs <container>
sudo journalctl -u docker -f
```

### Kubernetes
```bash
kubectl get nodes
kubectl describe node <node>
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
sudo journalctl -u kubelet -xe
```

### Terraform
```bash
terraform validate
terraform plan
terraform state list
```

### Ansible
```bash
ansible all -i inventory.ini -m ping -vvv
```

---

## 12) Final Self-Validation

- [ ] Git workflow + recovery
- [ ] Jenkins pipeline
- [ ] Ansible playbook + vault
- [ ] Docker build/run/compose/swarm
- [ ] Terraform workflow
- [ ] Kubernetes objects and troubleshooting
- [ ] Prometheus metrics visible
- [ ] Grafana dashboards working
