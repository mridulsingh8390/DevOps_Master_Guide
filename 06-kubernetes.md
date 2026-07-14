# Kubernetes Complete Guide (Core + Advanced Objects)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem, kubeadm-based)

> This file is intentionally detailed.
> Each section includes:
> - What/Why
> - YAML/Commands
> - Explanation
> - Verification
> - Common issues and fixes
> - Practice tasks

---

## Table of Contents

1. What is Kubernetes and why it matters  
2. Cluster setup using kubeadm  
3. kubectl essentials  
4. Core workload objects  
5. Networking objects  
6. Config and secret objects  
7. Storage objects  
8. Security and access control objects  
9. Policy and governance objects  
10. Scaling and availability objects  
11. Scheduling and runtime objects  
12. Advanced pod patterns  
13. CRD and extensibility  
14. Cluster operations (backup, upgrade, drain)  
15. Troubleshooting matrix  
16. Practice lab tasks  
17. Daily kubectl cheat sheet

---

## 1) What is Kubernetes and Why it Matters

## What
Kubernetes (K8s) is a container orchestration platform for deploying, scaling, and managing containerized applications.

## Why
- Self-healing workloads
- Declarative desired state
- Service discovery and load balancing
- Rolling updates and rollback
- Better reliability at scale

---

## 2) Cluster Setup using kubeadm (Single control-plane lab)

## 2.1 Prepare host OS

### Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Kernel modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Sysctl settings
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

---

## 2.2 Install container runtime (containerd)

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify:
```bash
sudo systemctl status containerd
```

---

## 2.3 Install kubeadm, kubelet, kubectl

```bash
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

Verify:
```bash
kubeadm version
kubectl version --client
```

---

## 2.4 Initialize cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Set kubeconfig:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install CNI (Calico):
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```

(For single-node lab) remove control-plane taint:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Verify:
```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 3) kubectl Essentials

```bash
kubectl cluster-info
kubectl get nodes
kubectl get ns
kubectl create ns dev
kubectl config get-contexts
kubectl config current-context
kubectl api-resources
kubectl api-versions
kubectl explain deployment.spec.template.spec.containers
```

---

## 4) Core Workload Objects

## 4.1 Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Apply:
```bash
kubectl apply -f namespace.yaml
kubectl get ns
```

---

## 4.2 Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: dev
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
```

Verify:
```bash
kubectl get pod -n dev -o wide
kubectl describe pod nginx-pod -n dev
kubectl logs nginx-pod -n dev
```

---

## 4.3 ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

---

## 4.4 Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Commands:
```bash
kubectl apply -f deploy.yaml
kubectl rollout status deploy/nginx-deploy -n dev
kubectl rollout history deploy/nginx-deploy -n dev
kubectl set image deploy/nginx-deploy nginx=nginx:alpine -n dev
kubectl rollout undo deploy/nginx-deploy -n dev
```

---

## 4.5 StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: dev
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

Why:
- Stable pod identity (`web-0`, `web-1`)
- Ordered rollout
- Strong stateful workload fit (DBs, queues)

---

## 4.6 DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: dev
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sh","-c","sleep 3600"]
```

Why:
Runs one pod per node (log collector, node exporter, security agent).

---

## 4.7 Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: one-time-job
  namespace: dev
spec:
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["sh","-c","echo hello && sleep 5"]
      restartPolicy: Never
```

---

## 4.8 CronJob
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
            command: ["sh","-c","date; echo hello"]
          restartPolicy: OnFailure
```

---

## 5) Networking Objects

## 5.1 Service (ClusterIP)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-deploy
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

## 5.2 Service (NodePort)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-deploy
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

## 5.3 Headless Service (for StatefulSet)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: dev
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
```

## 5.4 Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
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
            name: nginx-clusterip
            port:
              number: 80
```

---

## 6) Config and Secret Objects

## 6.1 ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
```

## 6.2 Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
stringData:
  DB_PASSWORD: "StrongPass123!"
```

Create via CLI:
```bash
kubectl create configmap app-config-cli --from-literal=ENV=dev -n dev
kubectl create secret generic app-secret-cli --from-literal=password='StrongPass123!' -n dev
```

---

## 7) Storage Objects

## 7.1 PersistentVolume (PV)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
```

## 7.2 PersistentVolumeClaim (PVC)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: dev
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## 7.3 StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Check:
```bash
kubectl get pv,pvc,storageclass -A
```

---

## 8) Security and Access Control Objects (RBAC)

## 8.1 ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
```

## 8.2 Role + RoleBinding
Role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
```

RoleBinding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
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

## 8.3 ClusterRole + ClusterRoleBinding
ClusterRole:
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

ClusterRoleBinding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: dev
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

Verify permissions:
```bash
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:app-sa
```

---

## 9) Policy and Governance Objects

## 9.1 NetworkPolicy (default deny ingress)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

## 9.2 ResourceQuota
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
    limits.cpu: "4"
    limits.memory: 4Gi
```

## 9.3 LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

---

## 10) Scaling and Availability Objects

## 10.1 HorizontalPodAutoscaler (HPA)
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

Note:
Needs metrics-server.

## 10.2 PodDisruptionBudget (PDB)
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

---

## 11) Scheduling and Runtime Objects

## 11.1 PriorityClass
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority workloads"
```

## 11.2 RuntimeClass
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: runc
handler: runc
```

## 11.3 Node selector / affinity / toleration example snippet
```yaml
spec:
  nodeSelector:
    disktype: ssd
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "batch"
    effect: "NoSchedule"
```

---

## 12) Advanced Pod Patterns

## 12.1 Probes
- readinessProbe: ready to serve?
- livenessProbe: still healthy?
- startupProbe: slow-start apps

## 12.2 Init container example
```yaml
initContainers:
- name: init-task
  image: busybox
  command: ["sh","-c","echo init && sleep 5"]
```

## 12.3 Sidecar pattern
Two containers in same pod sharing volume/logs/network namespace.

## 12.4 Ephemeral debug container
```bash
kubectl debug -it <pod> -n dev --image=busybox --target=<container>
```

---

## 13) CRD and Extensibility

## 13.1 CRD example
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

Check CRDs:
```bash
kubectl get crd
```

---

## 14) Cluster Operations

## 14.1 Node maintenance
```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

## 14.2 etcd backup (control-plane node)
```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-snapshot.db
```

Verify snapshot:
```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-snapshot.db -w table
```

## 14.3 Reset cluster (lab only)
```bash
sudo kubeadm reset -f
rm -rf ~/.kube
```

---

## 15) Troubleshooting Matrix

## 15.1 Cluster-level
```bash
kubectl get nodes
kubectl describe node <node>
sudo journalctl -u kubelet -xe
systemctl status containerd
```

## 15.2 Pod-level
```bash
kubectl get pods -A
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
```

## Common pod errors
- `ImagePullBackOff`: wrong image/tag/registry auth
- `CrashLoopBackOff`: app crashes at startup
- `Pending`: insufficient resources / PVC issue / taints

## 15.3 Networking
```bash
kubectl get svc,ep,endpoints,endpointslices -A
kubectl get ingress -A
kubectl get networkpolicy -A
```

## 15.4 DNS checks
```bash
kubectl get pods -n kube-system | grep coredns
kubectl exec -it <pod> -n dev -- nslookup kubernetes.default
```

---

## 16) Practice Lab Tasks

1. Install cluster via kubeadm  
2. Create `dev` namespace  
3. Deploy nginx deployment + service  
4. Perform rolling update and rollback  
5. Create ConfigMap + Secret and consume in pod  
6. Create PVC and mount in pod  
7. Create Role/RoleBinding and test `can-i`  
8. Apply NetworkPolicy and verify traffic behavior  
9. Configure HPA (with metrics-server)  
10. Take etcd snapshot backup  

---

## 17) Daily kubectl Cheat Sheet

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
```

---

## 18) Multi-Node Cluster (Joining Worker Nodes)

## Why
The single-node lab in Section 2 taints removed only cover a single control-plane. Real clusters add worker nodes.

## 18.1 On the control-plane, generate/print join command
```bash
kubeadm token create --print-join-command
```

## 18.2 On each worker node (after installing containerd + kubeadm/kubelet as in 2.2/2.3)
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

## 18.3 Verify from control-plane
```bash
kubectl get nodes -o wide
```

## 18.4 Rejoining after token expiry
```bash
kubeadm token list
kubeadm token create --ttl 2h --print-join-command
```

---

## 19) Ingress Controller Installation

## Why
The `Ingress` object (Section 5.4) is just a spec — you need a controller actually running to implement it. Nginx Ingress Controller is the most common choice.

## 19.1 Install via manifest
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
```

## 19.2 Or via Helm (preferred for upgrades)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace
```

## 19.3 Verify
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## 19.4 Full Ingress example referencing the controller's class
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: dev
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
```

Check ingress class:
```bash
kubectl get ingressclass
```

---

## 20) Metrics Server (Required for HPA and `kubectl top`)

## Install
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Lab-only fix (self-signed kubelet certs)
Metrics-server often fails TLS verification on kubeadm labs. Patch it:
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

## Verify
```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

---

## 21) Kustomize (Native Config Overlays)

## Why
Manage environment-specific variants (dev/staging/prod) of the same manifests without templating — built into `kubectl`.

## 21.1 Base (`base/kustomization.yaml`)
```yaml
resources:
  - deployment.yaml
  - service.yaml
```

## 21.2 Overlay (`overlays/prod/kustomization.yaml`)
```yaml
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
namespace: prod
```

`replica-patch.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 5
```

## 21.3 Build and apply
```bash
kubectl kustomize overlays/prod
kubectl apply -k overlays/prod
```

---

## 22) Admission Control / Policy Engines (OPA Gatekeeper, Kyverno)

## Why
NetworkPolicy and RBAC control traffic and access; policy engines enforce broader governance ("no `:latest` tags", "must have resource limits", "no privileged containers") at admission time.

## 22.1 Kyverno (simpler YAML-native option)
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
```

Example policy — require resource limits:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-resources
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "CPU/memory limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                cpu: "?*"
                memory: "?*"
```

## 22.2 OPA Gatekeeper (Rego-based, more powerful/complex)
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

Both approaches intercept `kubectl apply` via admission webhooks and can block or audit non-compliant resources.

---

## Final Notes

- Focus first on Deployment + Service + ConfigMap + Secret + PVC + RBAC.
- Then move to policies, autoscaling, and advanced scheduling.
- In production, add strong security controls, backups, and monitoring from day one.
- Add an Ingress controller and metrics-server early — most HPA and routing tasks depend on them.
- Use Kustomize for environment overlays and a policy engine (Kyverno/Gatekeeper) for governance at scale.
