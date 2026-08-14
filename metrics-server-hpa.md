# Metrics Server & Horizontal Pod Autoscaler (HPA)

Install **Metrics Server** on a kubeadm (or similar) cluster, then use **HPA** to scale workloads from CPU/memory (and optionally custom metrics). Includes cluster-wide sample usage.

Related: [kubeadm-production-cluster.md](./kubeadm-production-cluster.md) · [concepts/helm-zero-to-hero.md](./concepts/helm-zero-to-hero.md) · [concepts/argocd-production.md](./concepts/argocd-production.md)

Official: [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) · [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

---

## 0. How the pieces fit

```text
Kubelet (per node)  →  resource metrics
        ↓
 Metrics Server     →  metrics.k8s.io API (pods/nodes)
        ↓
 kubectl top  |  HPA controller  →  scales Deployment/StatefulSet/ReplicaSet/…
```

| Component | Role |
|-----------|------|
| **Metrics Server** | Aggregates CPU/memory usage; serves `metrics.k8s.io` |
| **HPA** | Adjusts replica count from metrics vs target |
| **Requests** | HPA % targets are relative to **resource requests** (not limits) |

Without Metrics Server: `kubectl top` fails and resource-based HPA stays on `<unknown>` / does not scale.

For **custom / external metrics** (QPS, queue depth, …) you need a metrics stack such as Prometheus Adapter — see §8.

---

## 1. Prerequisites

```bash
kubectl get nodes
kubectl get apiservices | grep metrics || true

# Aggregation layer is on by default with kubeadm
# Pods must set resources.requests for meaningful CPU/memory HPA
```

Checklist:

- [ ] Cluster healthy; CNI working  
- [ ] Nodes can be scraped by Metrics Server (API → node kubelet :10250)  
- [ ] Workloads define `resources.requests.cpu` / `memory` before using `%` targets  

---

## 2. Install Metrics Server

### 2.1 Pin a release (recommended)

```bash
# List releases: https://github.com/kubernetes-sigs/metrics-server/releases
export MS_VERSION=v0.7.2

kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/download/${MS_VERSION}/components.yaml
```

Or “latest” (labs only):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 2.2 kubeadm / bare metal — TLS to kubelet

Kubelet serving certs are often not trusted by Metrics Server. Common lab/on-prem fix:

```bash
kubectl -n kube-system patch deployment metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Also helpful when Hostname resolution fails:

```bash
kubectl -n kube-system patch deployment metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP"}]'
```

**Production preference:** enable kubelet **serverTLSBootstrap** and approve CSRs so Metrics Server can use `--kubelet-certificate-authority` instead of insecure TLS (see [Metrics Server FAQ](https://github.com/kubernetes-sigs/metrics-server) / kubeadm cert docs).

### 2.3 Install with Helm (alternative)

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm upgrade --install metrics-server metrics-server/metrics-server \
  -n kube-system \
  --version 3.12.2 \
  --set args[0]=--kubelet-insecure-tls \
  --set args[1]=--kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP \
  --atomic --wait
```

### 2.4 Verify Metrics Server

```bash
kubectl -n kube-system get deploy,pods -l k8s-app=metrics-server
kubectl -n kube-system logs -l k8s-app=metrics-server --tail=50

kubectl get apiservice v1beta1.metrics.k8s.io -o yaml | grep -E 'Available|True'

# Wait ~15–60s after pods Ready, then:
kubectl top nodes
kubectl top pods -A
```

Expect node CPU/memory columns and pod usage. If `Metrics not available`:

```bash
kubectl -n kube-system describe pod -l k8s-app=metrics-server
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | head
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | head
```

---

## 3. HPA basics

### 3.1 API versions

Prefer **`autoscaling/v2`** (multiple metrics, behavior, scale-up/down policies):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
```

### 3.2 Critical: resource requests

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: "1"
    memory: 512Mi
```

HPA target `averageUtilization: 70` means “aim for ~70% of **request**”, not of limit.

### 3.3 Algorithm (simplified)

```text
desiredReplicas = ceil( currentReplicas × (currentMetric / desiredMetric) )
```

Clamped between `minReplicas` and `maxReplicas`. Metrics are averaged across ready pods (with nuances for missing metrics).

---

## 4. Sample usage across the cluster

### 4.1 Namespace for demos

```bash
kubectl create namespace scale-demo
```

### 4.2 Example A — CPU-based HPA (web Deployment)

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: scale-demo
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: scale-demo
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: scale-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
EOF
```

Generate load:

```bash
kubectl -n scale-demo run load --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://web; done"

kubectl -n scale-demo get hpa web -w
kubectl -n scale-demo get pods -l app=web -w
kubectl -n scale-demo top pods
```

Stop load:

```bash
kubectl -n scale-demo delete pod load --force --grace-period=0
# Watch replicas fall back toward minReplicas (may take several minutes)
```

### 4.3 Example B — CPU + memory HPA

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: scale-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: "1"
              memory: 256Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
  namespace: scale-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  # HPA picks the highest desired replica count among metrics
EOF

kubectl -n scale-demo get hpa api
kubectl -n scale-demo describe hpa api
```

### 4.4 Example C — Absolute resource targets (not %)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-abs
  namespace: scale-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 1
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: AverageValue
          averageValue: 200m      # aim ~200 millicores per pod average
```

### 4.5 Example D — Scale behavior (stabilize prod)

Avoid flapping under bursty traffic:

```bash
kubectl apply -f - <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-stable
  namespace: scale-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
      selectPolicy: Min
EOF
```

| Knob | Meaning |
|------|---------|
| `stabilizationWindowSeconds` | Look-back before applying scale down/up |
| `policies` | Max pods or % change per period |
| `selectPolicy` | `Max` / `Min` / `Disabled` among policies |

### 4.6 Example E — Imperative HPA (quick)

```bash
kubectl -n scale-demo autoscale deployment web --cpu-percent=50 --min=1 --max=10
kubectl -n scale-demo get hpa
# Prefer declarative YAML in Git for production
```

### 4.7 Example F — Cluster-wide pattern (many teams)

Convention for every app namespace:

```text
1. Every Deployment/StatefulSet sets requests (+ limits)
2. One HPA per scalable workload (same name as Deployment)
3. Defaults: min=2 (HA), max= negotiated, CPU 60–70%
4. behavior.scaleDown.stabilizationWindowSeconds >= 300 in prod
5. PDB so HPA + drains don’t drop below quorum
```

Sample PDB alongside HPA:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api
  namespace: scale-demo
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: api
```

Platform checklist across namespaces:

```bash
# Find Deployments missing CPU requests (rough audit)
kubectl get deploy -A -o json | jq -r '
  .items[]
  | select([.spec.template.spec.containers[].resources.requests.cpu] | any(.==null or .==""))
  | "\(.metadata.namespace)/\(.metadata.name)"'

# List all HPAs cluster-wide
kubectl get hpa -A
kubectl get hpa -A -o wide

# HPAs that cannot compute (often missing metrics/requests)
kubectl get hpa -A | grep -E 'unknown|<|>'
```

### 4.8 Example G — StatefulSet note

HPA can scale StatefulSets, but order/storage semantics differ. Prefer HPA on **stateless Deployments**; scale stateful data planes with operators/manual care.

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: StatefulSet
  name: redis-followers   # only if safe for your app
```

---

## 5. Observe & debug

```bash
kubectl top nodes
kubectl top pods -n scale-demo
kubectl top pods -A --sort-by=cpu | head

kubectl get hpa -n scale-demo
kubectl describe hpa web -n scale-demo
# Events: SuccessfulRescale, FailedGetResourceMetric, …

kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/scale-demo/pods | jq .
```

| Symptom | Cause / fix |
|---------|-------------|
| `kubectl top` error | Metrics Server not Ready / TLS / APIService |
| HPA `cpu: <unknown>` | No requests, or metrics not flowing |
| Never scales up | Load too low, wrong target, maxReplicas=min |
| Scales up slowly | Stabilization / policies; metrics lag (~15s scrape) |
| Scales down too fast | Increase scaleDown stabilization window |
| Fight with Argo/Kustomize replicas | Ignore Deployment `/spec/replicas` in GitOps; let HPA own replicas |

GitOps tip ([argocd-production.md](./concepts/argocd-production.md)):

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

Or set `replicas` in the Deployment only as initial value and document HPA ownership.

---

## 6. Production practices

| Topic | Guidance |
|-------|----------|
| Install | Pin Metrics Server version; prefer Helm/GitOps over floating `latest` |
| TLS | Prefer kubelet cert bootstrap over long-term `--kubelet-insecure-tls` |
| HA | One Metrics Server Deployment is usual; ensure PDB if you add replicas |
| Requests | Enforce via LimitRange / policy (Kyverno) so HPA always works |
| minReplicas | ≥ 2 for HA frontends in prod |
| maxReplicas | Cap to node capacity / quota; alert near max |
| Testing | Load-test in staging; tune utilization & behavior |
| Custom metrics | Prometheus + adapter for business SLIs |
| VPA | Vertical Pod Autoscaler is separate (right-size requests); can combine carefully with HPA |

ResourceQuota interaction: HPA cannot exceed namespace quota—set quota headroom for `maxReplicas`.

```yaml
# LimitRange example — default requests if developers omit them
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: payments-prod
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: "1"
        memory: 512Mi
```

---

## 7. Cleanup demo

```bash
kubectl delete ns scale-demo

# Optional: remove Metrics Server (if you installed only for the lab)
# kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/download/${MS_VERSION}/components.yaml
# helm uninstall metrics-server -n kube-system
```

---

## 8. Beyond resource metrics (optional)

| Need | Tooling |
|------|---------|
| Scale on HTTP QPS / queue length | Prometheus + [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) → `custom.metrics.k8s.io` |
| Scale on external SaaS metric | External metrics API + adapter |
| Scale nodes (cluster capacity) | Cluster Autoscaler / Karpenter — **not** HPA |
| Right-size CPU/memory requests | VPA |

HPA `type: Pods` / `Object` / `External` metrics require those APIs to be installed and configured—Metrics Server alone is not enough.

Example shape (only works with custom metrics API):

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

---

## 9. Quick flow

```text
Install Metrics Server (pin version)
  → patch kubelet TLS flags if needed (kubeadm)
  → kubectl top nodes/pods works
  → ensure Deployments have resources.requests
  → create HPA (autoscaling/v2) per workload
  → load-test; tune utilization + behavior
  → GitOps: ignore Deployment replicas when HPA owns them
  → audit: kubectl get hpa -A
```

---

## References

- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Metrics Server installation](https://kubernetes-sigs.github.io/metrics-server/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Resource metrics pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [kubectl top](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/)
