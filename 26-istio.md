# Istio Complete Guide (Beginner to Advanced)
## Service Mesh for Kubernetes (Traffic, Security, Observability)

> Practical guide for DevOps engineers.
> Covers install, sidecar injection, routing, mTLS, authz, telemetry, troubleshooting.

---

## 1) What is Istio and Why it Matters

Istio is a service mesh that adds traffic control, security, and observability to Kubernetes services without changing application code.

## Why use it?
- Fine-grained traffic routing (canary, blue/green, A/B)
- Mutual TLS (mTLS) between services
- Centralized auth policies
- Rich telemetry (metrics, traces, logs)

---

## 2) Core Istio Concepts

- **Data plane**: Envoy sidecar proxies attached to pods
- **Control plane**: `istiod` manages config/certs/policy distribution
- **Gateway**: inbound/egress traffic entry/exit to mesh
- **VirtualService**: traffic routing rules
- **DestinationRule**: policies for traffic to a service (subsets, TLS, LB)
- **PeerAuthentication**: mTLS mode controls
- **AuthorizationPolicy**: allow/deny rules at workload/service level

---

## 3) Prerequisites

- Kubernetes cluster running (kubeadm/minikube/EKS/AKS/GKE)
- `kubectl` configured and working
- Enough cluster resources for sidecars

Verify:
```bash
kubectl get nodes
```

---

## 4) Install Istio CLI and Control Plane

## 4.1 Download Istio
```bash
cd /tmp
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.23.2 sh -
cd istio-1.23.2
export PATH=$PWD/bin:$PATH
istioctl version
```

## 4.2 Install Istio (demo profile for lab)
```bash
istioctl install --set profile=demo -y
```

Verify:
```bash
kubectl get pods -n istio-system
```

---

## 5) Enable Sidecar Injection

Label namespace:
```bash
kubectl create ns app
kubectl label namespace app istio-injection=enabled
kubectl get ns --show-labels
```

Deploy sample app in `app` namespace, then verify sidecar:
```bash
kubectl get pods -n app
kubectl get pod <pod-name> -n app -o jsonpath='{.spec.containers[*].name}'
```

You should see app container + `istio-proxy`.

---

## 6) Deploy Sample Application (Bookinfo)

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n app
kubectl get svc,pod -n app
```

Check app health:
```bash
kubectl exec -n app deploy/productpage-v1 -c productpage -- curl -sS http://details:9080/details/1
```

---

## 7) Expose Traffic via Istio Gateway

Apply gateway + virtual service:
```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n app
```

Get ingress address:
```bash
kubectl get svc istio-ingressgateway -n istio-system
```

For local clusters (NodePort), use node IP + nodePort.
For LoadBalancer-enabled environments, use external IP.

---

## 8) Traffic Management Basics

## 8.1 Define subsets (v1, v2, v3) with DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: app
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## 8.2 Route traffic with VirtualService

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: app
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

Use this for canary rollouts.

---

## 9) Advanced Traffic Features

## 9.1 Retry and timeout
```yaml
http:
- route:
  - destination:
      host: ratings
  retries:
    attempts: 3
    perTryTimeout: 2s
  timeout: 5s
```

## 9.2 Fault injection (testing resilience)
```yaml
http:
- fault:
    delay:
      percentage:
        value: 50
      fixedDelay: 2s
  route:
  - destination:
      host: ratings
```

## 9.3 Circuit breaker (DestinationRule)
- control connection pools/outlier detection to isolate unhealthy instances

---

## 10) Security: mTLS

## 10.1 Enable strict mTLS namespace-wide
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: app
spec:
  mtls:
    mode: STRICT
```

Apply:
```bash
kubectl apply -f peerauth-strict.yaml
```

Verify:
```bash
istioctl authn tls-check <pod>.<namespace>
```

---

## 11) Security: AuthorizationPolicy

Allow only specific service account:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-to-details
  namespace: app
spec:
  selector:
    matchLabels:
      app: details
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/app/sa/bookinfo-productpage"]
```

Use deny-by-default + explicit allow model for stronger security.

---

## 12) Ingress and Egress Patterns

## Ingress
- external traffic enters via `istio-ingressgateway`

## Egress
- control outbound traffic to external services
- use ServiceEntry for explicit external dependencies
- can route egress via egress gateway for monitoring/policy

---

## 13) Observability (Kiali, Prometheus, Grafana, Jaeger)

Install addons (lab):
```bash
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```

Port-forward examples:
```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
kubectl port-forward svc/prometheus -n istio-system 9090:9090
kubectl port-forward svc/grafana -n istio-system 3000:3000
```

Why this matters:
- visualize service graph
- request success/error rates
- latency percentiles
- trace request path across services

---

## 14) Istioctl Useful Commands

```bash
istioctl proxy-status
istioctl analyze -A
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard prometheus
```

Proxy config inspect:
```bash
istioctl proxy-config clusters <pod> -n app
istioctl proxy-config routes <pod> -n app
istioctl proxy-config listeners <pod> -n app
```

---

## 15) Performance and Resource Considerations

- Sidecars add CPU/memory overhead
- Tune proxy resources via sidecar settings
- Avoid unnecessary mesh enrollment for non-critical namespaces
- Watch p95/p99 latency impact after mesh enablement

---

## 16) Upgrade and Revision-Based Install (Concept)

For safer upgrades:
- use revision-based control planes
- migrate namespaces gradually to new revision label
- validate and roll forward/rollback cleanly

---

## 17) Troubleshooting Matrix

## 17.1 Sidecar not injected
- namespace label missing
- pod created before label
- webhook issue

Check:
```bash
kubectl get ns app --show-labels
kubectl describe pod <pod> -n app
```

## 17.2 503 upstream errors
- service endpoints missing
- DestinationRule/VirtualService mismatch
- mTLS mode mismatch

Check:
```bash
kubectl get endpoints -n app
istioctl analyze -A
```

## 17.3 Gateway not reachable
- ingress service type issue
- firewall/security group
- wrong host/path routing rules

## 17.4 Policy blocking traffic
- AuthorizationPolicy too strict
- missing principal/service account mapping

Inspect:
```bash
kubectl get authorizationpolicy -n app
kubectl describe authorizationpolicy <name> -n app
```

---

## 18) Best Practices

1. Start with non-prod namespace rollout  
2. Enable observability before strict security policies  
3. Adopt mTLS progressively, then strict  
4. Use canary traffic shifting for safe releases  
5. Keep routing/policy manifests in GitOps flow  
6. Validate with `istioctl analyze` in CI  

---

## 19) Practice Tasks

1. Install Istio demo profile  
2. Enable injection and deploy Bookinfo  
3. Expose app through ingress gateway  
4. Route 90/10 traffic between v1/v2 reviews  
5. Enable strict mTLS in namespace  
6. Add AuthorizationPolicy for one service path  
7. Trigger faults/timeouts and observe metrics/traces  

---

## 20) Daily Cheat Sheet

```bash
istioctl install --set profile=demo -y
kubectl label ns app istio-injection=enabled
istioctl analyze -A
istioctl proxy-status
kubectl get virtualservice,destinationrule,gateway -A
kubectl get peerauthentication,authorizationpolicy -A
```

---

## Final Notes

- Istio is powerful but should be introduced gradually.
- Master traffic routing + observability first, then enforce security policies.
- Pair Istio with GitOps (Argo CD) for reliable, auditable mesh config lifecycle.