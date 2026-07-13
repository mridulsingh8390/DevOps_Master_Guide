# DevOps V3 Command Bible (Local/On-Prem, Ubuntu 24.04 Only)
## Git + Jenkins + Ansible (Playbook/Roles/Vault) + Docker (DCA Full) + Terraform + Prometheus + Grafana + Kubernetes (kubeadm)

> **Goal:** Single-file, command-heavy, step-by-step learning and execution guide.  
> **Scope:** Ubuntu 24.04 local/on-prem only (no cloud-specific commands).  
> **Note:** Some commands are destructive. Practice in lab VM first.

---

## Table of Contents

1. [Base System Preparation](#1-base-system-preparation)
2. [Git - Complete Command Reference](#2-git---complete-command-reference)
3. [Jenkins - Install, Configure, Build, Backup, Troubleshooting](#3-jenkins---install-configure-build-backup-troubleshooting)
4. [Ansible - Inventory, Adhoc, Playbooks, Roles, Vault, Debug](#4-ansible---inventory-adhoc-playbooks-roles-vault-debug)
5. [Docker - DCA Full Practical Command Set](#5-docker---dca-full-practical-command-set)
6. [Terraform - Full Local Workflow Command Set](#6-terraform---full-local-workflow-command-set)
7. [Kubernetes - kubeadm Install + All Core kubectl Commands](#7-kubernetes---kubeadm-install--all-core-kubectl-commands)
8. [Prometheus - Install + Configure + PromQL + Ops](#8-prometheus---install--configure--promql--ops)
9. [Grafana - Install + Datasource + Dashboards + Admin Ops](#9-grafana---install--datasource--dashboards--admin-ops)
10. [Integrated Mini Project (Local)](#10-integrated-mini-project-local)
11. [Massive Troubleshooting Section](#11-massive-troubleshooting-section)
12. [Interview Drill (Tool-wise)](#12-interview-drill-tool-wise)
13. [Daily/Weekly Practice Checklist](#13-dailyweekly-practice-checklist)

---

## 1) Base System Preparation

## 1.1 Update OS and install essentials
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  curl wget git vim nano jq yq unzip zip tree htop net-tools telnet nmap \
  ca-certificates gnupg lsb-release software-properties-common apt-transport-https \
  build-essential python3 python3-pip rsync
```

## 1.2 Time sync + hostname
```bash
timedatectl
sudo timedatectl set-timezone Asia/Kolkata   # change as required
hostnamectl
sudo hostnamectl set-hostname devops-lab
```

## 1.3 Useful folders
```bash
mkdir -p ~/devops-lab/{git,jenkins,ansible,docker,terraform,k8s,monitoring,scripts}
cd ~/devops-lab
```

## 1.4 Optional aliases
```bash
cat <<'EOF' >> ~/.bashrc
alias k='kubectl'
alias d='docker'
alias tf='terraform'
alias a='ansible'
alias ap='ansible-playbook'
EOF
source ~/.bashrc
```

---

## 2) Git - Complete Command Reference

## 2.1 Global config
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"
git config --global pull.rebase false
git config --global color.ui auto
git config --global credential.helper store
git config --list --show-origin
```

## 2.2 SSH setup
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com
```

## 2.3 Repository setup
```bash
git init
git clone <repo_url>
git clone -b <branch> <repo_url>
git remote -v
git remote add origin <repo_url>
git remote set-url origin <new_repo_url>
```

## 2.4 File lifecycle
```bash
git status
git add .
git add <file>
git add -p
git commit -m "feat: message"
git commit --amend -m "new message"
git mv old.txt new.txt
git rm file.txt
```

## 2.5 Branching
```bash
git branch
git branch -a
git switch -c feature/auth
git checkout -b feature/auth
git switch main
git merge feature/auth
git branch -d feature/auth
git branch -D feature/auth
git push -u origin feature/auth
git push origin --delete feature/auth
```

## 2.6 Syncing
```bash
git fetch
git fetch --all --prune
git pull
git pull --rebase
git push
```

## 2.7 Rebase/cherry-pick
```bash
git rebase main
git rebase -i HEAD~5
git rebase --continue
git rebase --abort
git cherry-pick <sha>
git cherry-pick --continue
git cherry-pick --abort
```

## 2.8 Undo/recovery
```bash
git restore <file>
git restore --staged <file>
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
git revert <sha>
git reflog
git checkout <sha>
git switch -
```

## 2.9 Stash
```bash
git stash
git stash push -m "WIP login"
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}
git stash pop
git stash drop stash@{0}
git stash clear
```

## 2.10 Logs/diff/blame
```bash
git log --oneline --graph --decorate --all
git log -p
git show <sha>
git diff
git diff --staged
git diff main..feature/auth
git blame <file>
```

## 2.11 Tags
```bash
git tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
git push --tags
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0
```

## 2.12 Cleanup and maintenance
```bash
git clean -n
git clean -fd
git gc
git fsck
```

---

## 3) Jenkins - Install, Configure, Build, Backup, Troubleshooting

## 3.1 Install Java + Jenkins
```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version

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

## 3.2 Access + initial password
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Open `http://<server-ip>:8080`

## 3.3 Service operations
```bash
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins
sudo systemctl reload jenkins
sudo journalctl -u jenkins -f
```

## 3.4 Jenkins directories
```bash
ls -lah /var/lib/jenkins
ls -lah /var/log/jenkins
```

## 3.5 Install Docker for Jenkins jobs (local)
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

## 3.6 Sample Jenkinsfile (build/test/package)
```groovy
pipeline {
  agent any
  options { timestamps() }
  environment {
    APP = "flask-demo"
    TAG = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build') {
      steps { sh 'docker build -t $APP:$TAG .' }
    }
    stage('Test') {
      steps { sh 'docker run --rm $APP:$TAG python -c "print(\"ok\")"' }
    }
    stage('List Image') {
      steps { sh 'docker images | head' }
    }
  }
  post {
    always { sh 'echo Pipeline finished' }
    failure { sh 'echo Pipeline failed' }
  }
}
```

## 3.7 Jenkins backup/restore
```bash
sudo systemctl stop jenkins
sudo tar -czf /tmp/jenkins_backup_$(date +%F).tar.gz /var/lib/jenkins
sudo systemctl start jenkins
```

Restore:
```bash
sudo systemctl stop jenkins
sudo tar -xzf /tmp/jenkins_backup_YYYY-MM-DD.tar.gz -C /
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo systemctl start jenkins
```

---

## 4) Ansible - Inventory, Adhoc, Playbooks, Roles, Vault, Debug

## 4.1 Install
```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

## 4.2 Inventory examples
`inventory.ini`
```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[db]
192.168.1.20 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

## 4.3 Ad-hoc commands (very useful)
```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m command -a "whoami"
ansible all -i inventory.ini -m shell -a "df -h"
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
ansible all -i inventory.ini -m service -a "name=nginx state=started enabled=yes" --become
ansible all -i inventory.ini -m copy -a "src=/etc/hosts dest=/tmp/hosts.bak"
ansible all -i inventory.ini -m fetch -a "src=/etc/hosts dest=./fetched flat=yes"
ansible all -i inventory.ini -m file -a "path=/tmp/testdir state=directory mode=0755"
ansible all -i inventory.ini -m user -a "name=devops state=present groups=sudo append=yes" --become
ansible all -i inventory.ini -m group -a "name=devops state=present" --become
ansible all -i inventory.ini -m setup
ansible all -i inventory.ini -m setup -a "filter=ansible_distribution*"
```

## 4.4 Playbook example
`site.yml`
```yaml
- name: Configure web
  hosts: web
  become: yes
  vars:
    pkg_name: nginx
  tasks:
    - name: Install package
      apt:
        name: "{{ pkg_name }}"
        state: present
        update_cache: yes

    - name: Deploy index
      copy:
        content: "Hello from Ansible on {{ inventory_hostname }}"
        dest: /var/www/html/index.html
      notify: restart nginx

    - name: Ensure service started
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

Run options:
```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --check
ansible-playbook -i inventory.ini site.yml --diff
ansible-playbook -i inventory.ini site.yml --limit web
ansible-playbook -i inventory.ini site.yml --tags nginx
ansible-playbook -i inventory.ini site.yml --skip-tags debug
ansible-playbook -i inventory.ini site.yml -vvv
```

## 4.5 Roles
```bash
ansible-galaxy init roles/nginx
tree roles/nginx
```

`roles/nginx/tasks/main.yml`
```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Template config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx
```

`roles/nginx/handlers/main.yml`
```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

`role-site.yml`
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

## 4.6 Vault commands
```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey secrets.yml
```

Playbook with vault:
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
ansible-playbook -i inventory.ini site.yml --vault-password-file .vault_pass.txt
```

---

## 5) Docker - DCA Full Practical Command Set

## 5.1 Install Docker Engine + Compose plugin
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
```

## 5.2 Core info
```bash
docker version
docker info
docker system df
docker context ls
```

## 5.3 Container commands
```bash
docker run hello-world
docker run -d --name web -p 8080:80 nginx
docker ps
docker ps -a
docker stop web
docker start web
docker restart web
docker rm web
docker rm -f web
docker rename web web01
```

## 5.4 Image commands
```bash
docker images
docker pull ubuntu:24.04
docker build -t myapp:v1 .
docker tag myapp:v1 myrepo/myapp:v1
docker push myrepo/myapp:v1
docker image inspect myapp:v1
docker history myapp:v1
docker rmi myapp:v1
docker image prune -f
```

## 5.5 Logs/exec/inspect
```bash
docker logs web
docker logs -f web
docker exec -it web sh
docker inspect web
docker top web
docker stats
docker events
```

## 5.6 Volume commands
```bash
docker volume ls
docker volume create data-vol
docker run -d --name db -v data-vol:/var/lib/mysql mysql:8
docker volume inspect data-vol
docker volume rm data-vol
docker volume prune -f
```

## 5.7 Bind mounts/tmpfs
```bash
docker run -it --rm -v $PWD:/app ubuntu:24.04 ls /app
docker run -d --name tmp --tmpfs /tmp nginx
```

## 5.8 Network commands
```bash
docker network ls
docker network create app-net
docker network inspect app-net
docker run -d --name app1 --network app-net nginx
docker network connect app-net app1
docker network disconnect app-net app1
docker network rm app-net
```

## 5.9 Docker Compose
`compose.yaml`
```yaml
services:
  app:
    image: nginx:latest
    ports:
      - "8081:80"
```

Commands:
```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose exec app sh
docker compose down
docker compose down -v
```

## 5.10 Swarm (DCA)
```bash
docker swarm init
docker node ls
docker service create --name web --replicas 2 -p 8080:80 nginx
docker service ls
docker service ps web
docker service scale web=4
docker service update --image nginx:alpine web
docker service rollback web
docker service rm web
docker stack ls
docker swarm leave --force
```

## 5.11 Secrets/configs
```bash
echo "mypassword" | docker secret create db_password -
docker secret ls
docker secret rm db_password

echo "ENV=prod" > app.env
docker config create app_env app.env
docker config ls
docker config rm app_env
```

## 5.12 Save/load/export/import
```bash
docker save myapp:v1 -o myapp_v1.tar
docker load -i myapp_v1.tar
docker export web -o web_container.tar
docker import web_container.tar web_imported:latest
```

---

## 6) Terraform - Full Local Workflow Command Set

## 6.1 Install
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y terraform
terraform version
```

## 6.2 Basic commands
```bash
terraform init
terraform fmt
terraform fmt -recursive
terraform validate
terraform plan
terraform plan -out=tfplan
terraform apply
terraform apply -auto-approve
terraform destroy
terraform destroy -auto-approve
```

## 6.3 Example local terraform (`main.tf`)
```hcl
terraform {
  required_version = ">= 1.5.0"
}
provider "local" {}

resource "local_file" "a" {
  filename = "${path.module}/output.txt"
  content  = "Hello Terraform Local"
}
```

Run:
```bash
terraform init
terraform plan
terraform apply -auto-approve
cat output.txt
terraform destroy -auto-approve
```

## 6.4 Variable/outputs
```bash
terraform output
terraform output -json
terraform console
```

## 6.5 State commands
```bash
terraform state list
terraform state show local_file.a
terraform state pull > state.json
terraform state rm local_file.a
terraform import local_file.a /absolute/path/to/output.txt
terraform force-unlock <LOCK_ID>
```

## 6.6 Workspaces
```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
terraform workspace select default
terraform workspace delete dev
```

## 6.7 Graph and providers
```bash
terraform providers
terraform graph
terraform graph | dot -Tpng > graph.png
```

---

## 7) Kubernetes - kubeadm Install + All Core kubectl Commands

## 7.1 System prep
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
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## 7.2 containerd
```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 7.3 Install kubeadm/kubelet/kubectl
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 7.4 Init cluster
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

## 7.5 kubeconfig
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 7.6 Install CNI (Calico)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

Single-node schedule:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## 7.7 kubectl Command Catalog (Major)

## Cluster & API
```bash
kubectl version --client
kubectl cluster-info
kubectl api-resources
kubectl api-versions
kubectl config view
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>
```

## Nodes/Namespaces
```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl top nodes
kubectl get ns
kubectl create ns dev
kubectl delete ns dev
```

## Pods
```bash
kubectl get pods -A
kubectl get pods -n default -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs -f <pod> -n <ns>
kubectl logs <pod> -c <container> -n <ns>
kubectl logs <pod> --previous -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl delete pod <pod> -n <ns>
```

## Deployments/ReplicaSets
```bash
kubectl create deployment nginx --image=nginx
kubectl get deploy
kubectl describe deploy nginx
kubectl scale deploy nginx --replicas=3
kubectl set image deploy/nginx nginx=nginx:alpine
kubectl rollout status deploy/nginx
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx
kubectl delete deploy nginx
```

## Services
```bash
kubectl expose deploy nginx --port=80 --type=NodePort
kubectl get svc
kubectl describe svc nginx
kubectl delete svc nginx
kubectl port-forward svc/nginx 8080:80
```

## ConfigMaps/Secrets
```bash
kubectl create configmap app-config --from-literal=ENV=prod
kubectl create configmap app-file --from-file=app.properties
kubectl get configmap
kubectl describe configmap app-config
kubectl delete configmap app-config

kubectl create secret generic db-secret --from-literal=password=Passw0rd!
kubectl get secret
kubectl describe secret db-secret
kubectl delete secret db-secret
```

## Jobs/CronJobs
```bash
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'
kubectl get jobs
kubectl describe job pi
kubectl delete job pi
```

CronJob (yaml apply recommended):
```bash
kubectl get cronjobs
kubectl delete cronjob <name>
```

## DaemonSet/StatefulSet
```bash
kubectl get ds -A
kubectl get sts -A
kubectl describe ds <name> -n <ns>
kubectl describe sts <name> -n <ns>
```

## Label/Annotate
```bash
kubectl label pod <pod> app=demo --overwrite
kubectl annotate pod <pod> owner=devops --overwrite
```

## Taint/Toleration Node Ops
```bash
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:NoSchedule-
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

## Manifest operations
```bash
kubectl apply -f file.yaml
kubectl apply -f dir/
kubectl create -f file.yaml
kubectl delete -f file.yaml
kubectl replace -f file.yaml
kubectl diff -f file.yaml
kubectl explain deployment.spec.template.spec.containers
```

## Output formatting
```bash
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP
```

## Debug/events
```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe pod <pod> -n <ns>
kubectl auth can-i create pods
kubectl auth can-i '*' '*' --all-namespaces
```

## 7.8 kubeadm reset (lab only)
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo systemctl restart containerd
rm -rf $HOME/.kube
```

---

## 8) Prometheus - Install + Configure + PromQL + Ops

## 8.1 Install Prometheus
```bash
sudo useradd --no-create-home --shell /bin/false prometheus || true
cd /tmp
PROM_VERSION="2.54.1"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar -xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

## 8.2 Prometheus service
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
  --storage.tsdb.path=/var/lib/prometheus \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

## 8.3 Node exporter
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter || true
cd /tmp
NODE_VERSION="1.8.2"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_VERSION}/node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
tar -xvf node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_VERSION}.linux-amd64/node_exporter /usr/local/bin/
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
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

## 8.4 Prometheus scrape config
```bash
cat <<EOF | sudo tee /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node_exporter
    static_configs:
      - targets: ["localhost:9100"]
EOF

sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

## 8.5 Prometheus operations
```bash
sudo systemctl status prometheus
sudo journalctl -u prometheus -f
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets | jq
curl -X POST http://localhost:9090/-/reload
```

## 8.6 PromQL quick list
```promql
up
up{job="node_exporter"}
node_cpu_seconds_total
rate(node_cpu_seconds_total[5m])
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
node_memory_MemAvailable_bytes
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
node_filesystem_avail_bytes
```

---

## 9) Grafana - Install + Datasource + Dashboards + Admin Ops

## 9.1 Install
```bash
cd /tmp
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo apt install -y adduser libfontconfig1 musl
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

## 9.2 Service ops
```bash
sudo systemctl start grafana-server
sudo systemctl stop grafana-server
sudo systemctl restart grafana-server
sudo journalctl -u grafana-server -f
```

## 9.3 Admin reset
```bash
sudo grafana-cli admin reset-admin-password 'StrongNewPassword!'
```

## 9.4 Access
- URL: `http://<server-ip>:3000`
- Default: admin/admin

## 9.5 Configure Prometheus datasource
- Connections → Data sources → Add data source → Prometheus  
- URL: `http://localhost:9090`  
- Save & Test

## 9.6 Import dashboard (Node Exporter)
- Dashboard ID commonly used: `1860` (Node Exporter Full)

---

## 10) Integrated Mini Project (Local)

## 10.1 Project structure
```bash
mkdir -p ~/devops-lab/project/{app,k8s,jenkins,ansible,terraform}
cd ~/devops-lab/project
```

## 10.2 Simple app
`app/app.py`
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello DevOps V3"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
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

Build and run:
```bash
cd app
docker build -t flask-demo:v1 .
docker run -d --name flask-demo -p 5000:5000 flask-demo:v1
curl http://localhost:5000
```

## 10.3 Kubernetes deploy
`k8s/deployment.yaml`
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
```

`k8s/service.yaml`
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
      nodePort: 30080
```

Apply:
```bash
cd ~/devops-lab/project
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get all
curl http://<node-ip>:30080
```

---

## 11) Massive Troubleshooting Section

## 11.1 OS/Port/Network
```bash
ip a
ip r
ss -tulnp
sudo ufw status
sudo ufw allow 22,80,443,3000,8080,9090,6443/tcp
ping -c 4 8.8.8.8
curl -I http://localhost
```

## 11.2 DNS
```bash
cat /etc/resolv.conf
nslookup google.com
dig google.com
```

## 11.3 Service health
```bash
systemctl status docker
systemctl status jenkins
systemctl status kubelet
systemctl status prometheus
systemctl status grafana-server
journalctl -xe
```

## 11.4 Jenkins fail
```bash
sudo journalctl -u jenkins -f
sudo cat /var/log/jenkins/jenkins.log
```

## 11.5 Docker fail
```bash
docker info
sudo journalctl -u docker -f
sudo usermod -aG docker $USER
newgrp docker
```

## 11.6 Kubernetes fail
```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node <node>
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
sudo journalctl -u kubelet -xe
crictl ps -a
crictl logs <container_id>
```

Common symptoms:
- `NotReady`: CNI not up / containerd issue / ip_forward disabled
- `ImagePullBackOff`: bad image/tag or no registry access
- `CrashLoopBackOff`: app startup failure, env/config mismatch
- `Pending`: insufficient resources, taints, missing PVC

## 11.7 Prometheus fail
```bash
sudo journalctl -u prometheus -f
sudo promtool check config /etc/prometheus/prometheus.yml
curl -s http://localhost:9090/api/v1/targets | jq
```

## 11.8 Grafana fail
```bash
sudo journalctl -u grafana-server -f
curl -I http://localhost:3000/login
sudo grafana-cli admin reset-admin-password 'Admin@12345'
```

## 11.9 Ansible fail
```bash
ansible all -i inventory.ini -m ping -vvv
ssh -i <key> ubuntu@<host>
```

---

## 12) Interview Drill (Tool-wise)

## Git
- Difference between merge and rebase?
- Difference between revert and reset?
- How reflog helps in recovery?

## Jenkins
- Declarative vs scripted pipeline?
- How to secure secrets?
- How to trigger pipeline on Git webhook?

## Ansible
- What is idempotency?
- Role vs playbook?
- Variable precedence order?
- Vault in CI/CD?

## Docker (DCA)
- Image layers?
- CMD vs ENTRYPOINT?
- Bridge vs host network?
- Swarm rolling update and rollback?

## Terraform
- Why state file?
- Local vs remote backend?
- Module reuse?
- `taint`, `import`, `workspace` use cases?

## Kubernetes
- Pod vs Deployment vs StatefulSet?
- Service types?
- ConfigMap vs Secret?
- Liveness/readiness/startup probes?
- etcd snapshot recovery basics?

## Monitoring
- Pull vs push model?
- Why labels are important?
- What is cardinality problem?
- Golden signals?

---

## 13) Daily/Weekly Practice Checklist

## Daily (30-60 min)
- 10 Git commands
- 10 kubectl commands
- 5 Docker commands
- 1 Ansible adhoc + 1 playbook run
- 1 Terraform plan/apply/destroy cycle (local)

## Weekly
- Rebuild k8s cluster once from scratch
- Deploy one app end-to-end
- Break/fix one component intentionally
- Create one Grafana dashboard
- Backup and restore Jenkins home

---

## Final Note

This V3 is command-heavy by design.  
If you want next, I can create a **V4 exam-style edition** with:
- 300+ scenario-based tasks,
- expected outputs,
- “broken environment” troubleshooting labs,
- and answer key sections.
