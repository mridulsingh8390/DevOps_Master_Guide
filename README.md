# DevOps Complete Learning Guide (Ubuntu 24.04, Local/On-Prem)
## Git • Jenkins • Ansible • Docker (DCA) • Terraform • Kubernetes (Core + Advanced Objects) • Prometheus • Grafana

> Beginner-friendly + command explanations.  
> Each section includes: **Why**, **Commands**, **Verify**, **Common issues**.

---

## 0) Prerequisites

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim nano jq yq unzip zip tree htop net-tools dnsutils \
ca-certificates gnupg lsb-release software-properties-common apt-transport-https
```

```bash
mkdir -p ~/devops-lab/{git,jenkins,ansible,docker,terraform,k8s,monitoring,project}
cd ~/devops-lab
```

---

## 1) Git

## Why Git?
Track changes, collaborate safely, recover work.

## Setup
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
git config --list
```

## SSH setup
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

## Core workflow
```bash
git init
git clone <repo_url>
git checkout -b feature/login
git add .
git commit -m "feat: login feature"
git pull --rebase
git push -u origin feature/login
```

## Recovery and advanced
```bash
git status
git stash
git stash list
git stash pop
git rebase main
git cherry-pick <sha>
git reset --soft HEAD~1
git revert <sha>
git reflog
git log --oneline --graph --decorate --all
```

---

## 2) Jenkins

## Why Jenkins?
Automates build/test/deploy pipeline.

## Install
```bash
sudo apt install -y fontconfig openjdk-21-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update && sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

Get unlock password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Service management
```bash
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f
```

## Pipeline example
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
```

## Backup/Restore
```bash
sudo systemctl stop jenkins
sudo tar -czf /tmp/jenkins_backup_$(date +%F).tar.gz /var/lib/jenkins
sudo systemctl start jenkins
```

---

## 3) Ansible (Playbook + Roles + Vault)

## Why Ansible?
Agentless configuration management and automation.

## Install
```bash
sudo apt install -y ansible
ansible --version
```

## Inventory (`inventory.ini`)
```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu
```

## Adhoc
```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
ansible all -i inventory.ini -m service -a "name=nginx state=started enabled=yes" --become
```

## Playbook (`site.yml`)
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

## Roles
```bash
ansible-galaxy init roles/nginx
```

## Vault
```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

---

## 4) Docker (DCA-focused + Dockerfiles)

## Why Docker?
Portable, reproducible runtime for applications.

## Install
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

## Core commands
```bash
docker version
docker info
docker ps -a
docker run -d --name web -p 8080:80 nginx
docker logs -f web
docker exec -it web sh
docker inspect web
docker stop web && docker rm web
```

## Images
```bash
docker build -t myapp:v1 .
docker images
docker tag myapp:v1 myrepo/myapp:v1
docker push myrepo/myapp:v1
docker image prune -f
```

## Volumes/Networks
```bash
docker volume create app-vol
docker network create app-net
docker run -d --name app --network app-net nginx
```

## Compose
```bash
docker compose up -d
docker compose logs -f
docker compose down
```

## Swarm (DCA)
```bash
docker swarm init
docker service create --name web --replicas 2 -p 8080:80 nginx
docker service ls
docker service scale web=4
docker service rollback web
```

## Dockerfile examples

### Python
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python","app.py"]
```

### Node
```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
```

### Java multi-stage
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

### Go multi-stage
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app main.go
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /src/app /app
ENTRYPOINT ["/app"]
```

### `.dockerignore`
```gitignore
.git
node_modules/
__pycache__/
*.log
.env
```

---

## 5) Terraform

## Why Terraform?
Infrastructure as code with reproducible state.

## Install
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform version
```

## Workflow
```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy
```

## State/workspace
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
  content  = "Terraform local demo"
}
```

---

# 6) Kubernetes (Core + Advanced Objects)

## 6.1 Install Kubernetes (kubeadm)

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

---

## 6.2 Kubernetes Objects (All Important)

### 1. Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### 2. Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.27
```

### 3. ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: dev
spec:
  replicas: 2
  selector: {matchLabels: {app: nginx-rs}}
  template:
    metadata: {labels: {app: nginx-rs}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

### 4. Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: dev
spec:
  replicas: 3
  selector: {matchLabels: {app: nginx-deploy}}
  template:
    metadata: {labels: {app: nginx-deploy}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

### 5. StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: dev
spec:
  serviceName: web
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

### 6. DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-agent
  namespace: dev
spec:
  selector: {matchLabels: {app: ds-agent}}
  template:
    metadata: {labels: {app: ds-agent}}
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sh","-c","sleep 3600"]
```

### 7. Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
  namespace: dev
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["sh","-c","echo hello"]
      restartPolicy: Never
```

### 8. CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
  namespace: dev
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["sh","-c","date; echo hi"]
          restartPolicy: OnFailure
```

### 9. Service (ClusterIP/NodePort)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: dev
spec:
  selector: {app: nginx-deploy}
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### 10. Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ing
  namespace: dev
spec:
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

### 11. ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  APP_ENV: "dev"
```

### 12. Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
stringData:
  DB_PASSWORD: "Strong123!"
```

### 13. PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity: {storage: 5Gi}
  accessModes: ["ReadWriteOnce"]
  hostPath: {path: /mnt/data}
```

### 14. PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: dev
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
```

### 15. StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### 16. ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
```

### 17. Role + RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 18. ClusterRole + ClusterRoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list","watch"]
```

### 19. NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: dev
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

### 20. ResourceQuota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "20"
    requests.cpu: "2"
    requests.memory: 2Gi
```

### 21. LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default: {cpu: "500m", memory: "512Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
```

### 22. HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

### 23. PodDisruptionBudget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
  namespace: dev
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx-deploy
```

### 24. PriorityClass
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority"
```

### 25. RuntimeClass
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: runc
handler: runc
```

### 26. CRD (basic example)
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  names:
    kind: Widget
    plural: widgets
    singular: widget
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
```

---

## 6.3 Advanced Pod Patterns

### Probes
- readiness: ready for traffic
- liveness: container health restart
- startup: long startup protection

### Init containers
Run setup before main container.

### Sidecar
Helper container in same pod.

### Ephemeral debug container
```bash
kubectl debug -it <pod> -n dev --image=busybox --target=<container>
```

---

## 6.4 Scheduling & Security Advanced

- nodeSelector
- node affinity / pod anti-affinity
- taints and tolerations
- topology spread constraints
- securityContext (runAsNonRoot, capabilities drop)
- Pod Security Admission labels
- RBAC `kubectl auth can-i`

---

## 6.5 kubectl Must-Know Commands

```bash
kubectl get all -A
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs -f <pod> -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl get events -A --sort-by=.lastTimestamp
kubectl rollout status deploy/<name> -n <ns>
kubectl rollout undo deploy/<name> -n <ns>
kubectl top nodes
kubectl top pods -A
kubectl auth can-i create pods -n dev
kubectl explain deployment.spec.template.spec.containers
```

---

## 6.6 etcd backup (important)

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-snapshot.db
```

---

## 7) Prometheus

Install binaries + service + node exporter (same as earlier).  
Verify:
```bash
curl -s http://localhost:9090/-/healthy
```

PromQL:
```promql
up
rate(node_cpu_seconds_total[5m])
(node_memory_MemTotal_bytes-node_memory_MemAvailable_bytes)/node_memory_MemTotal_bytes*100
```

---

## 8) Grafana

Install:
```bash
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo systemctl enable --now grafana-server
```

Reset password:
```bash
sudo grafana-cli admin reset-admin-password 'StrongPassword@123'
```

Add Prometheus datasource: `http://localhost:9090`

---

## 9) Troubleshooting Quick Matrix

### Kubernetes
```bash
kubectl get nodes
kubectl describe node <node>
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
sudo journalctl -u kubelet -xe
```

### Docker
```bash
docker info
docker logs <container>
sudo journalctl -u docker -f
```

### Jenkins
```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
```

### Ansible
```bash
ansible all -i inventory.ini -m ping -vvv
```

### Terraform
```bash
terraform validate
terraform plan
terraform state list
```

---

## 10) Final Validation Checklist

- [ ] Git commands + recovery
- [ ] Jenkins install + pipeline + backup
- [ ] Ansible inventory + playbook + role + vault
- [ ] Docker core + compose + swarm + Dockerfile examples
- [ ] Terraform init→destroy + state/workspace
- [ ] Kubernetes core objects done
- [ ] Kubernetes advanced objects done
- [ ] Prometheus + node exporter working
- [ ] Grafana datasource/dashboard configured
- [ ] Troubleshooting commands practiced

---
