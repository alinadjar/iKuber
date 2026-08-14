# Prometheus & Grafana on Kubernetes (Production)

How to add **Prometheus** and **Grafana** for cluster and app observability, whether they should live **on the workload cluster or separately**, and practical **install methods**.

Related: [metrics-server-hpa.md](./metrics-server-hpa.md) · [concepts/helm-zero-to-hero.md](./concepts/helm-zero-to-hero.md) · [concepts/argocd-production.md](./concepts/argocd-production.md) · [kubeadm-production-cluster.md](./kubeadm-production-cluster.md)

Official: [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) · [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) · [Grafana](https://grafana.com/docs/grafana/latest/)

---

## 0. Metrics Server vs Prometheus

| | **Metrics Server** | **Prometheus** |
|--|--------------------|----------------|
| Purpose | HPA / `kubectl top` (resource usage) | Long-term metrics, alerts, dashboards |
| Retention | Ephemeral (seconds–minutes) | Days–weeks (or longer with remote storage) |
| Query | `metrics.k8s.io` | PromQL |
| Replace each other? | **No** — keep both in production |

HPA continues to use Metrics Server (or custom metrics via adapters). Prometheus is for **SLOs, alerting, and Grafana**.

---

## 1. Should monitoring be separate from the operational cluster?

**Short answer:** For serious production, prefer a **dedicated observability path** so that when the workload cluster is sick, you can still see metrics and alerts. That can mean:

1. A **separate monitoring cluster** (or VM/Prometheus HA pair) that scrapes many clusters, **or**
2. In-cluster Prometheus with **remote_write** to durable storage (Thanos / Mimir / Cortex / vendor) **outside** the failing failure domain, **or**
3. A **managed** metrics backend (Grafana Cloud, Amazon Managed Prometheus, Google Managed Prometheus, …).

Running **only** an in-cluster Prometheus/Grafana with local PV and no remote storage is fine for **lab / small prod**, but it is a weak design when the cluster it lives on is the thing on fire.

### 1.1 Options compared

| Pattern | Description | Pros | Cons | Best for |
|---------|-------------|------|------|----------|
| **A. In-cluster (same as workloads)** | kube-prometheus-stack in `monitoring` NS on the prod cluster | Simple, low latency scrape, one kubeconfig | Cluster outage → blind; noisy neighbor; upgrade coupled | Labs, small single clusters, non-critical |
| **B. In-cluster + remote_write** | Local Prometheus (or Agent) scrapes; ships to Thanos/Mimir/SaaS | Local scrape + durable/global view; survives cluster loss for history | Extra components/cost; tune ship buffers | **Most production single/multi cluster** |
| **C. Dedicated observability cluster** | Hub cluster runs Prometheus/Grafana/Alertmanager; scrapes workload clusters | Blast radius isolation; central UI; shared rules | Hub HA required; network/firewall; more ops | Many clusters, regulated ops, platform teams |
| **D. Fully managed SaaS** | Agent/Grafana Alloy/Prometheus remote_write only in cluster | Least ops; HA/retention handled | Cost, data residency, vendor lock-in | Teams that want ops focus on apps |

```text
Recommended default for production:

  Workload cluster(s)
    → scrape with Prometheus / Grafana Alloy / Prometheus Agent
    → remote_write → durable backend (Mimir/Thanos/SaaS)
    → Grafana (on hub cluster OR SaaS) reads backend + optional live query

Keep a small local Prometheus only if you need short-term debug without depending on the WAN.
```

### 1.2 Decision guide

| Question | If yes → lean toward |
|----------|----------------------|
| Multiple clusters / regions? | **C** or **B** with global store |
| Must debug when API/CNI is broken? | **C** or out-of-cluster Grafana + remote store (**B/D**) |
| Single small cluster, tight budget? | **A**, plan migration to **B** |
| Strict data residency / air-gap? | **A** or **C** in your DC (not SaaS) |
| Already on Grafana Cloud / AMP? | **D** + in-cluster agents |

**Grafana placement:** UI can sit on the hub/observability cluster or SaaS even if scrapers run on every workload cluster. Avoid putting your **only** Grafana on a single prod cluster with no backup access.

---

## 2. What to install (components)

Typical production stack:

| Component | Role |
|-----------|------|
| **Prometheus Operator** | Manages Prometheus/Alertmanager via CRs (`Prometheus`, `ServiceMonitor`, …) |
| **Prometheus** | Scrapes & evaluates rules |
| **Alertmanager** | Routes alerts (Slack, PagerDuty, email) |
| **node-exporter** / **kube-state-metrics** | Node + Kubernetes object metrics |
| **Grafana** | Dashboards & Explore |
| **ServiceMonitor / PodMonitor** | How apps opt into scraping |
| **Optional:** Thanos sidecar / Mimir / Grafana Alloy | HA, long retention, multi-cluster |

The fastest production-grade bundle is **`kube-prometheus-stack`** (Helm): Operator + Prometheus + Alertmanager + Grafana + exporters + default rules/dashboards.

---

## 3. Method 1 — kube-prometheus-stack (Helm) on the cluster

Best starting point for **pattern A** or as the **local scraper** in **pattern B**.

### 3.1 Add repo & pin versions

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Pick chart version compatible with your Kubernetes — check Artifact Hub
helm search repo prometheus-community/kube-prometheus-stack --versions | head
```

### 3.2 Production-oriented values (sketch)

```yaml
# values-monitoring-prod.yaml — review against your chart version
fullnameOverride: "kube-prometheus-stack"

grafana:
  enabled: true
  admin:
    existingSecret: grafana-admin          # create Secret; don't put password in Git
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.example.com
    tls:
      - secretName: grafana-tls
        hosts: [grafana.example.com]
  persistence:
    enabled: true
    size: 10Gi
  # SSO later: grafana.ini auth.generic_oauth / Azure AD / etc.

prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: 40GB
    replicas: 2                  # needs anti-affinity + storage strategy
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        memory: 4Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: your-csi-storageclass
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    # Pattern B — ship off-cluster:
    # remoteWrite:
    #   - url: https://mimir.example.com/api/v1/push
    #     basicAuth:
    #       username:
    #         name: mimir-auth
    #         key: user
    #       password:
    #         name: mimir-auth
    #         key: password
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    # Empty selectors = discover all ServiceMonitors/PodMonitors/rules in the cluster

alertmanager:
  enabled: true
  alertmanagerSpec:
    replicas: 2
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: your-csi-storageclass
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

prometheusOperator:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

kube-state-metrics:
  enabled: true

nodeExporter:
  enabled: true
```

Create Grafana admin secret:

```bash
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='CHANGE_ME_LONG_RANDOM'
```

### 3.3 Install

```bash
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --version 65.0.0 \
  -f values-monitoring-prod.yaml \
  --atomic --wait --timeout 15m

kubectl -n monitoring get pods
kubectl -n monitoring get prometheus,alertmanager
```

### 3.4 Access UIs

```bash
# Port-forward (break-glass / first login)
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
# https://grafana.example.com via Ingress in prod

kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090
```

Default Grafana dashboards from the chart cover kubelet, API server, nodes, pods, etc.

### 3.5 Scrape your application (ServiceMonitor)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  namespace: payments-prod
  labels:
    app: payments-api
spec:
  selector:
    app: payments-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-api
  namespace: payments-prod
  labels:
    release: kps          # must match Prometheus serviceMonitorSelector if set
spec:
  selector:
    matchLabels:
      app: payments-api
  namespaceSelector:
    matchNames:
      - payments-prod
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

```bash
kubectl apply -f servicemonitor-payments-api.yaml
# In Prometheus UI → Status → Targets: payments-api should appear UP
```

App must expose Prometheus text format on `/metrics` (client libraries: prom-client, prometheus_client, micrometer, …).

---

## 4. Method 2 — Dedicated observability cluster (pattern C)

### 4.1 Layout

```text
observability-cluster (hub)
  ├── Grafana
  ├── Prometheus (or Mimir/Thanos query)
  ├── Alertmanager
  └── optionally: Loki, Tempo

workload-cluster-prod
  ├── metrics-server          (HPA — stays local)
  ├── kube-state-metrics / node-exporter / kubelet metrics
  └── Prometheus Agent or Grafana Alloy  →  remote_write → hub/store
```

### 4.2 How the hub gets metrics

| Approach | How |
|----------|-----|
| **remote_write** | Agent on each cluster pushes to hub Mimir/Prometheus receive |
| **federation** | Hub Prometheus scrapes `/federate` on leaf Prometheus (simpler, weaker at scale) |
| **Thanos sidecar** | Each Prometheus uploads blocks to object storage; hub Query reads all |
| **Grafana Alloy / Agent** | Lightweight collect + remote_write (often preferred vs full Prometheus per cluster) |

### 4.3 Cross-cluster scrape requirements

- Network allowlist: agent → remote_write URL; or hub → leaf Prometheus if federating  
- Auth: mTLS or token on remote_write  
- Labels: inject `cluster="prod-eu-1"` on every series (`externalLabels` / Alloy `cluster` label)  
- Tenancy: separate Grafana datasources or Mimir tenants per env  

### 4.4 Bootstrap sketch

```bash
# On HUB cluster
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-hub.yaml --version 65.0.0

# On WORKLOAD cluster — lighter agent example (Grafana Alloy chart or Prometheus Agent)
# Alloy remote_write to https://mimir.obs.example.com/api/v1/push
# Include kubelet, cAdvisor, kube-state-metrics, ServiceMonitors as needed
```

Keep **Metrics Server on each workload cluster** for HPA; do not rely on hub Prometheus for `kubectl top`.

---

## 5. Method 3 — kube-prometheus (jsonnet / raw manifests)

[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) is the upstream “opinionated” stack (used as a base for the Helm chart). Use when you want Git-managed manifests without Helm:

```bash
# Follow current repo quickstart — typically:
# git clone + make / jb install + ./build.sh + kubectl apply -f manifests/
```

Heavier customization curve; prefer **Helm chart** unless you already live in jsonnet.

---

## 6. Method 4 — GitOps (Argo CD)

Declare the Helm release in Git ([argocd-production.md](./concepts/argocd-production.md)):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.0.0
    helm:
      valueFiles:
        - $values/platform/monitoring/values-prod.yaml
      # Or use multi-source to combine chart + values repo
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Put ServiceMonitors next to each app in team repos; ensure Prometheus selectors allow them (see §3.2).

---

## 7. Method 5 — Managed / SaaS (pattern D)

```text
Cluster: Grafana Alloy or prometheus-agent
  → remote_write → Grafana Cloud / AMP / GMP
Grafana: cloud UI or local Grafana with cloud datasource
```

Still install **kube-state-metrics** / node exporters or Alloy Kubernetes integrations so you don’t miss cluster metrics.

---

## 8. Alerting & Grafana SSO (production)

### 8.1 Alertmanager

Configure receivers (Slack, PagerDuty, Opsgenie) via chart values or `AlertmanagerConfig` CRs. Start from default kube-prometheus **recording/alerting rules**, then tune noise.

```bash
kubectl -n monitoring get prometheusrule
```

### 8.2 Grafana SSO

Wire OIDC (same IdP as [oidc-sso-setup.md](./concepts/oidc-sso-setup.md)). Disable anonymous admin; use groups for Editor vs Viewer.

### 8.3 Dashboards

- Prefer chart/community dashboards as a baseline  
- App dashboards as **ConfigMaps** with sidecar (`grafana.sidecar.dashboards`) or Grafana folder provisioning in Git  
- Label dashboards with `cluster` variable for multi-cluster  

---

## 9. Production best practices

| Area | Practice |
|------|----------|
| **Failure domain** | Don’t keep the only metrics copy inside one prod cluster without remote_write/hub |
| **Retention** | Local 7–15d; long-term in object storage / Mimir |
| **Resources** | Set requests/limits; watch Prometheus RSS vs cardinality |
| **Cardinality** | Avoid high-cardinality labels (`user_id`, full URLs); use recording rules |
| **Storage** | CSI with backups or emptyDir+remote_write only |
| **HA** | 2+ Prometheus replicas **or** Agent+remote HA store; 2 Alertmanager |
| **RBAC** | Least privilege for Operator; Grafana SSO roles |
| **NetworkPolicy** | Limit who can reach Prometheus/Grafana; allow scrape ports from Prometheus pods |
| **TLS / Ingress** | Grafana and Alertmanager behind SSO-aware ingress; not public write paths |
| **Version pin** | Chart + app versions in Git; staging first |
| **App contract** | `/metrics` + ServiceMonitor; document RED/USE metrics |
| **HPA** | Keep Metrics Server; optional Prometheus Adapter for custom metrics HPA |
| **Cost** | Sample less frequently for huge fleets; drop unused metrics |

### Cardinality red flags

```promql
# Top metrics by series count (PromQL varies by stack)
topk(20, count by (__name__)({__name__=~".+"}))
```

---

## 10. Verify end-to-end

```bash
kubectl -n monitoring get pods
kubectl -n monitoring get servicemonitor -A

# Prometheus targets UP
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090
# Browser: http://localhost:9090/targets

# Sample queries
# up
# sum(rate(container_cpu_usage_seconds_total{namespace="payments-prod"}[5m]))
# kube_pod_status_phase{phase="Failed"}

# Grafana datasource = Prometheus (usually auto-provisioned by the chart)
```

Test an alert by firing a synthetic rule in staging, not by breaking prod.

---

## 11. Which method should you choose?

```text
Single cluster, getting started
  → Method 1: kube-prometheus-stack in "monitoring"
  → Add remote_write as soon as metrics matter for incidents (→ pattern B)

Multiple clusters / platform team
  → Method 2: hub Grafana + Mimir/Thanos + Alloy/Agent on leaves
  → Or Method 5: SaaS backend + in-cluster agents

GitOps already (Argo CD)
  → Method 4 wrapping Method 1 or 2 values

Custom metrics HPA
  → Prometheus (any pattern) + prometheus-adapter (see metrics-server-hpa.md §8)
```

---

## 12. Quick flow

```text
Decide: A / B / C / D (prefer B or C for production)
  → helm install kube-prometheus-stack (pin version) OR Alloy → remote_write
  → Ingress + SSO for Grafana
  → ServiceMonitors for platform + apps
  → Tune Alertmanager; silence noise
  → Optional: hub cluster / Thanos / Mimir / SaaS for durability
  → Keep Metrics Server for HPA
```

---

## References

- [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Prometheus remote_write](https://prometheus.io/docs/concepts/remote_write/)
- [Thanos](https://thanos.io/) · [Grafana Mimir](https://grafana.com/docs/mimir/latest/)
- [Grafana Alloy](https://grafana.com/docs/alloy/latest/)
- [ServiceMonitor](https://prometheus-operator.dev/docs/developer/getting-started/)
- [Kubernetes monitoring overview](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
