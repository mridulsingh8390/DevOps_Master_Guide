# DevOps Master Guide (Ubuntu 24.04) — Git, Jenkins, Ansible, Vault, Docker (DCA), Terraform, Prometheus, Grafana, Kubernetes

> A practical, step-by-step single-file learning guide.

---

## 0) Prerequisites

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim unzip gnupg lsb-release ca-certificates software-properties-common apt-transport-https
```

Optional useful tools:

```bash
sudo apt install -y jq tree net-tools htop
```

---

## 1) Git Commands (Daily + Advanced)

### Initial setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"
```

### SSH setup for GitHub/GitLab

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
# add key to Git provider
ssh -T git@github.com
```

### Repo lifecycle

```bash
git clone <repo_url>
cd <repo>
git status
git add .
git commit -m "feat: initial commit"
git push origin main
```

### Branching + merge/rebase

```bash
git checkout -b feature/login
git add .
git commit -m "feat: add login API"
git push -u origin feature/login

git checkout main
git pull origin main
git checkout feature/login
git rebase main
git push --force-with-lease
```

### Undo / recovery

```bash
git log --oneline --graph --decorate
git restore <file>
git restore --staged <file>
git reset --soft HEAD~1
git reset --hard HEAD~1
git reflog
git cherry-pick <commit_sha>
```

### Tags + release

```bash
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

---

## 2) Jenkins Installation (Ubuntu 24.04)

## Install Java + Jenkins

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

### Access Jenkins

- URL: `http://<server-ip>:8080`
- Initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Basic Jenkins pipeline (Declarative)

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'echo Building...' }
    }
    stage('Test') {
      steps { sh 'echo Testing...' }
    }
    stage('Deploy') {
      steps { sh 'echo Deploying...' }
    }
  }
}
```

---

## 3) Ansible Installation + Commands + Playbook + Roles + Vault

## Install Ansible

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

### Inventory example (`inventory.ini`)

```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[db]
192.168.1.20 ansible_user=ubuntu
```

### Ad-hoc commands

```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m apt -a "name=nginx state=present" --become
```

### Playbook example (`site.yml`)

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

    - name: Ensure nginx started
      service:
        name: nginx
        state: started
        enabled: yes
```

Run:

```bash
ansible-playbook -i inventory.ini site.yml
```

### Roles example

```bash
mkdir -p ansible-demo && cd ansible-demo
ansible-galaxy init roles/nginx
```

`roles/nginx/tasks/main.yml`:

```yaml
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

`role-site.yml`:

```yaml
- hosts: web
  become: yes
  roles:
    - nginx
```

Run:

```bash
ansible-playbook -i inventory.ini role-site.yml
```

### Ansible Vault

Encrypt variable file:

```bash
ansible-vault create group_vars/all/vault.yml
```

Example content:

```yaml
db_password: "SuperSecret123!"
```

Use in playbook:

```yaml
- debug:
    msg: "{{ db_password }}"
```

Run with vault password prompt:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

Encrypt existing file:

```bash
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault view secrets.yml
```

---

## 4) Docker (DCA-oriented full checklist)

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
docker --version
```

Add user to docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

### Core DCA domains practice

- **Image creation/management**: `docker build`, multi-stage builds, `docker image prune`
- **Registry**: `docker login`, `docker push`, `docker pull`
- **Orchestration (Swarm basics)**:
  ```bash
  docker swarm init
  docker service create --name web -p 8080:80 nginx
  docker service ls
  docker service scale web=3
  ```
- **Networking**:
  ```bash
  docker network create app-net
  docker run -d --name redis --network app-net redis
  docker run -d --name app --network app-net nginx
  ```
- **Storage**:
  ```bash
  docker volume create app-data
  docker run -d -v app-data:/data --name busybox busybox sleep 3600
  ```
- **Security**:
  - Least privilege
  - Image scanning (e.g., Trivy)
  - Secrets in Swarm: `docker secret create`
- **Troubleshooting**:
  ```bash
  docker ps -a
  docker logs <container>
  docker inspect <container>
  docker exec -it <container> sh
  docker events
  ```

---

## 5) Terraform Installation + Commands

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

### Terraform workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy
```

### Minimal example (`main.tf`)

```hcl
terraform {
  required_version = ">= 1.5.0"
}

provider "local" {}

resource "local_file" "example" {
  filename = "demo.txt"
  content  = "Terraform is working!"
}
```

Run:

```bash
terraform init
terraform apply -auto-approve
cat demo.txt
terraform destroy -auto-approve
```

---

## 6) Prometheus + Grafana Installation and Configuration

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

Systemd service:

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
sudo systemctl status prometheus
```

## Grafana install

```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

- Grafana URL: `http://<ip>:3000`
- Default login: `admin/admin` (change immediately)

### Configure Grafana datasource

- Go to **Connections → Data Sources → Add data source → Prometheus**
- URL: `http://localhost:9090`
- Save & test

---

## 7) Kubernetes Installation (kubeadm single-node lab)

## Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Kernel modules + sysctl

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
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

## Install kubeadm/kubelet/kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Init cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl for user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install CNI (Calico):

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

Allow scheduling on control-plane (lab only):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Verify:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 8) Kubernetes Troubleshooting Quick Guide

### Cluster not ready

```bash
kubectl get nodes
kubectl describe node <node-name>
journalctl -u kubelet -xe
```

### Pod crash

```bash
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
```

### Network/CNI issues

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <cni-pod>
ip a
sudo sysctl net.ipv4.ip_forward
```

### DNS issues

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl exec -it <pod> -- nslookup kubernetes.default
```

### etcd / control plane issues (single node)

```bash
sudo crictl ps -a | grep -E "etcd|kube-apiserver|kube-controller|kube-scheduler"
sudo journalctl -u kubelet -f
```

### Reset cluster (lab only)

```bash
sudo kubeadm reset -f
sudo rm -rf ~/.kube
```

---

## 9) Suggested Learning Order (30-day path)

1. Git fundamentals + branching
2. Docker core + compose + swarm basics
3. Jenkins pipelines
4. Ansible playbooks + roles + vault
5. Terraform workflow + state basics
6. Kubernetes install + workloads + troubleshooting
7. Prometheus + Grafana monitoring stack

---

## 10) Security Best Practices (All Tools)

- Never hardcode secrets in Git
- Use Ansible Vault / secret manager
- Rotate credentials regularly
- Restrict firewall ports (8080, 3000, 9090, 6443, etc.)
- Enable least privilege (sudo, cloud IAM, RBAC)
- Keep systems patched

---

## 11) Common Port Reference

- Jenkins: `8080`
- Prometheus: `9090`
- Grafana: `3000`
- Kubernetes API: `6443`
- Node Exporter: `9100` (if installed)
- Docker Registry (self-hosted): `5000`

---

## 12) Next Step (Recommended)

After this guide, build one mini project:
- Jenkins pipeline builds Docker image
- Push to registry
- Deploy to Kubernetes
- Monitor with Prometheus + Grafana
- Infra managed by Terraform
- Config managed by Ansible

That project will solidify all concepts together.