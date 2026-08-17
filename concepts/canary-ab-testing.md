# A/B Testing & Canary Releases on Kubernetes

Best practices to roll out changes safely: **canary**, **A/B (experiments)**, and how they differ from **blue/green**—with concrete patterns on Ingress, Gateway API, Flagger, and Argo Rollouts.

Related: [ingress-zero-to-hero.md](./ingress-zero-to-hero.md) · [argocd-production.md](./argocd-production.md) · [metrics-server-hpa.md](../metrics-server-hpa.md) · [prometheus-grafana.md](../prometheus-grafana.md) · [helm-zero-to-hero.md](./helm-zero-to-hero.md)

Official / tools: [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) · [Flagger](https://docs.flagger.app/) · [Gateway API](https://gateway-api.sigs.k8s.io/) · [Ingress-NGINX canary](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)

---

## 0. Choose the right strategy

| Strategy | Goal | Traffic split | Success signal | Typical use |
|----------|------|---------------|----------------|-------------|
| **Canary** | Reduce blast radius of a **new version** | % of traffic (1% → 10% → 50% → 100%) | SLOs: error rate, latency, saturation | Almost every production deploy |
| **A/B test** | Compare **product variants** (UX, pricing, algorithm) | By **user/cookie/header** (sticky), often 50/50 | Business metrics + guardrail SLOs | Experiments; may keep both versions long |
| **Blue/Green** | Instant cutover / instant rollback | 100% flip between two full environments | Smoke + health after switch | Regulated cutovers; costly double capacity |
| **Rolling update** | Default Deployment update | Gradual pod replace, **no** traffic intelligence | Pod readiness only | Fine for low risk; weak for bad code that still “Ready” |

```text
Canary  → “Is v2 safe enough for everyone?”
A/B     → “Which experience is better for the business?”
```

You can combine them: canary for **safety**, then A/B for **experimentation** on a stable baseline—or run experiments only after canary reaches 100%.

---

## 1. Building blocks you always need

Regardless of tool:

| Piece | Role |
|-------|------|
| **Two (or more) versions runnable** | Separate Deployments, or Rollout with pod template versions |
| **Traffic splitting** | Ingress weights, Gateway HTTPRoute weights, mesh VirtualService, or service mesh |
| **Sticky identity (A/B)** | Cookie / header / user-id hash so a user doesn’t flip every request |
| **Observability** | Golden signals (latency, errors, traffic, saturation) + app KPIs |
| **Automated analysis** | Compare canary vs stable; promote or rollback |
| **Fast rollback** | Shift weight to 0% or revert image in one step |
| **Compatible data** | Migrations expand/contract; dual-write if needed |
| **Feature flags (optional)** | Decouple deploy from release; great with A/B |

**Do not** rely only on `kubectl rollout status`—Ready pods can still serve 5xx or wrong business results.

---

## 2. Canary — detailed patterns

### 2.1 Mental model

```text
stable (v1)  ←── most traffic
canary (v2)  ←── small % 

Metrics: canary error/latency vs stable (or vs SLO threshold)
  → OK: increase %
  → Bad: send 0% to canary; keep/revert stable
```

### 2.2 Pattern A — Ingress-NGINX canary annotations (simple)

Two Deployments + Services; two Ingresses sharing the host. Canary Ingress carries nginx annotations.

```yaml
# Stable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-stable
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels: { app: api, track: stable }
  template:
    metadata:
      labels: { app: api, track: stable }
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api:1.0.0
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: api-stable
  namespace: demo
spec:
  selector: { app: api, track: stable }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: api-stable, port: { number: 80 } }
```

```yaml
# Canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-canary
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: api, track: canary }
  template:
    metadata:
      labels: { app: api, track: canary }
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api:1.1.0
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
---
apiVersion: v1
kind: Service
metadata:
  name: api-canary
  namespace: demo
spec:
  selector: { app: api, track: canary }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-canary
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% of traffic
    # Optional sticky / header based:
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-header-value: "always"
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: api-canary, port: { number: 80 } }
```

**Progression (manual):**

```bash
# 5% → 25% → 50% → 100% (then make canary the new stable)
kubectl -n demo annotate ingress api-canary \
  nginx.ingress.kubernetes.io/canary-weight=25 --overwrite

# Rollback
kubectl -n demo annotate ingress api-canary \
  nginx.ingress.kubernetes.io/canary-weight=0 --overwrite
# or delete canary Ingress/Deployment
```

**Pros:** no mesh; works with existing Ingress-NGINX.  
**Cons:** nginx-specific; analysis/promotion mostly **manual** unless you script it; limited vs mesh.

### 2.3 Pattern B — Gateway API weighted backends

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
  namespace: demo
spec:
  parentRefs:
    - name: public-gateway
  hostnames: ["api.example.com"]
  rules:
    - backendRefs:
        - name: api-stable
          port: 80
          weight: 90
        - name: api-canary
          port: 80
          weight: 10
```

Portable across Gateway implementations (Cilium, Istio, Contour, …). Still need automation for stepwise weight + metrics.

### 2.4 Pattern C — Argo Rollouts (recommended for canary automation)

Replace `Deployment` with `Rollout`; controller shifts traffic and runs analysis.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
  namespace: demo
spec:
  replicas: 4
  strategy:
    canary:
      maxSurge: "25%"
      maxUnavailable: 0
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: api-success-rate
            args:
              - name: service-name
                value: api-canary
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      # Traffic routing via Ingress/Gateway/Istio plugin — see Rollouts docs for your provider
      trafficRouting:
        nginx:
          stableIngress: api
          additionalIngressAnnotations:
            canary-by-header: X-Canary
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api:1.1.0
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: api-success-rate
  namespace: demo
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 2
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

```bash
kubectl argo rollouts get rollout api -n demo -w
kubectl argo rollouts promote api -n demo      # skip pause if allowed
kubectl argo rollouts abort api -n demo        # rollback
```

**Pros:** first-class canary + analysis + abort; fits Argo CD.  
**Cons:** new CRDs; learn traffic-routing plugins.

### 2.5 Pattern D — Flagger + progressive delivery

Flagger watches a Deployment/App and creates canary objects, automating weight steps against Prometheus/Datadog/etc. metrics. Good with Istio, Linkerd, Contour, nginx, Gateway API, App Mesh.

Typical loop: install Flagger → define `Canary` CR with metrics thresholds → Flagger promotes or rolls back.

### 2.6 Pattern E — Service mesh (Istio example)

```yaml
# Conceptual VirtualService
http:
  - route:
      - destination: { host: api-stable, subset: v1 }
        weight: 90
      - destination: { host: api-canary, subset: v2 }
        weight: 10
```

Powerful (retries, fault injection, observability) but higher ops cost—use when you already run a mesh.

---

## 3. A/B testing — detailed patterns

### 3.1 Mental model

```text
Users hashed or cookied into cohort A or B
  → sticky routing to variant Deployment
  → measure conversion / revenue / engagement
  → keep guardrail SLOs (errors/latency) on both
  → declare winner; decommission loser
```

A/B is **not** primarily about gradual % for safety (though you may ramp exposure). Sticky assignment matters.

### 3.2 Routing dimensions

| Mechanism | Example | Notes |
|-----------|---------|-------|
| **Header** | `X-Experiment: B` | Great for QA / internal force |
| **Cookie** | `ab=variant_b` | Sticky browsers; set at edge or app |
| **User id hash** | Consistent hash in mesh/app | Best for logged-in users |
| **Query param** | `?ab=b` | Easy demos; easy to leak/cache wrong |
| **Geo / device** | Match rules | Product-specific |

### 3.3 Ingress-NGINX header / cookie canary (acts as A/B)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Experiment"
    nginx.ingress.kubernetes.io/canary-by-header-value: "B"
    # OR cookie:
    # nginx.ingress.kubernetes.io/canary-by-cookie: "ab_variant"
```

Traffic with header/cookie → canary Service; everyone else → stable. For **true 50/50 A/B**, use:

- App or edge sets cookie randomly once, **or**
- Mesh/Gateway match + weights with session affinity, **or**
- Experiment platform (LaunchDarkly, Unleash, GrowthBook, Flagsmith, custom)

### 3.4 Feature flags (often better for product A/B)

Deploy **one** binary; flag SDK chooses behavior.

```text
Deploy once (safe rolling/canary of code)
  → Flag “checkout_v2” 5% users → 50% → 100%
  → Instant disable without redeploy
```

**Best practice:** use flags for **product experiments**; use canary releases for **binary/platform risk**. Many orgs do both.

### 3.5 Experiment hygiene

- [ ] Pre-register hypothesis and primary metric  
- [ ] Guardrail metrics (p99 latency, 5xx, CPU) can auto-stop experiment  
- [ ] Same data schema for A and B; avoid incompatible API responses mid-test  
- [ ] Segment carefully (new users only, etc.)  
- [ ] Don’t peek endlessly (inflation of false winners)  
- [ ] Document freeze windows (marketing campaigns skew results)  

---

## 4. Shared production best practices

### 4.1 Observability first

Before the first canary:

- [ ] RED/USE metrics on **both** versions (label by `version` / `track` / `rollouts_pod_template_hash`)  
- [ ] Dashboards: stable vs canary side-by-side  
- [ ] Alerts on burn rate / error budget  
- [ ] Trace sampling to compare latency breakdowns  
- [ ] Log `version` / `x-release-header` for support  

Prometheus metric labeling example (app side):

```text
http_requests_total{app="api",version="1.1.0",track="canary",status="500"}
```

### 4.2 Analysis gates (canary)

Define **objective** promote/abort rules:

| Metric | Example gate |
|--------|----------------|
| Availability | success rate ≥ 99% (or ≥ stable − 0.5%) |
| Latency | canary p99 ≤ stable p99 × 1.1 |
| Saturation | CPU/memory not thrashing |
| Business (optional) | checkout conversion not down > X% |

Run analysis long enough for **statistical confidence** (low traffic = longer pauses).

### 4.3 Capacity & scheduling

- [ ] Canary pods must get **real** traffic and similar resources as stable  
- [ ] Don’t run canary on a unique node type that masks issues  
- [ ] HPA: prefer separate HPAs per track or use Rollouts-aware scaling  
- [ ] PDB on stable so canary experiments don’t drain quorum  

### 4.4 Data & migrations

```text
Expand  → deploy code that reads old+new schema
Migrate → backfill
Contract → remove old paths
```

Never ship a destructive migration only on canary pods that stable still needs.

### 4.5 APIs & contracts

- [ ] Backward-compatible APIs during canary  
- [ ] Client retries / timeouts tuned before shifting weight  
- [ ] gRPC/WebSocket: sticky sessions; weight shifts harder—test explicitly  

### 4.6 Security & compliance

- [ ] Same NetworkPolicy for canary and stable  
- [ ] Same image signing / admission policy  
- [ ] PII in experiment logs scrubbed  
- [ ] Change windows / approvals for prod weight bumps (GitOps PR)  

### 4.7 GitOps

- [ ] Desired weight/steps in Git (ApplicationSet / PR)  
- [ ] Argo CD ignore Deployment replicas if HPA/Rollouts owns them  
- [ ] Record promote/abort in PR or Rollout history  

### 4.8 Rollback criteria (write them down)

Abort if:

- Error rate exceeds threshold for N minutes  
- p99 latency exceeds budget  
- CrashLoop / failed readiness on canary  
- Critical business KPI drops  
- Manual `abort` by on-call  

Rollback must be **one command / one PR**, practiced in staging.

---

## 5. End-to-end recommended stacks

| Maturity | Canary | A/B |
|----------|--------|-----|
| **Starting** | Ingress-NGINX weight + manual steps + Grafana | Header/cookie canary + product analytics |
| **Solid** | **Argo Rollouts** or **Flagger** + Prometheus Analysis | Feature flags + analytics; guardrail SLOs |
| **Advanced** | Mesh + progressive delivery + automated freeze | Experimentation platform + metric pipelines |

**Practical recommendation for most kubeadm platforms:**

```text
1. Ingress / Gateway for north-south traffic
2. Argo Rollouts (or Flagger) for automated canary
3. Prometheus metrics with version labels
4. Feature-flag system for product A/B
5. GitOps (Argo CD) for auditability
```

---

## 6. Worked canary day checklist

```text
□ New image in registry (scanned, signed)
□ Staging full test + canary dry-run
□ Dashboards & AnalysisTemplate ready
□ DB expand migration already applied
□ Start canary at 1–5%
□ Watch SLOs for agreed soak time
□ Step weights 5 → 25 → 50 → 100
□ Promote: canary becomes stable; remove old ReplicaSet
□ Tag release; update runbook if anything failed
```

Abort path:

```text
□ Abort rollout / set weight 0
□ Confirm stable healthy
□ Keep failed image blocked; file incident
□ Fix forward or roll image pin in Git
```

---

## 7. Example: promote canary to stable (manual nginx)

```bash
# After canary at 100% and healthy:
# 1) Update stable Deployment to canary image
kubectl -n demo set image deploy/api-stable api=ghcr.io/acme/api:1.1.0
kubectl -n demo rollout status deploy/api-stable

# 2) Remove canary Ingress weight / delete canary objects
kubectl -n demo delete ingress api-canary
kubectl -n demo delete deploy api-canary svc api-canary
```

With Rollouts, promotion is automatic at final step or `kubectl argo rollouts promote`.

---

## 8. Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Canary with 0 meaningful traffic | Minimum weight + soak time |
| Same pods for “canary” without labels/metrics | Distinct version labels + split metrics |
| Only CPU health for promote | Error/latency/business gates |
| Breaking schema on canary | Expand/contract migrations |
| A/B without stickiness | Cookie/user-hash assignment |
| Flipping weights via click-ops only | GitOps + audited automation |
| Canary in prod without staging rehearsal | Same pipeline in staging first |

---

## 9. Quick comparison of implementations

| Approach | Automation | Stickiness | Complexity | Best fit |
|----------|------------|------------|------------|----------|
| NGINX canary annotations | Low | Header/cookie | Low | Simple HTTP canary |
| Gateway API weights | Low–med | Implementation-dependent | Low–med | Portable HTTP |
| Argo Rollouts | High | Via routing plugin | Med | Recommended canary |
| Flagger | High | Via provider | Med | Progressive delivery |
| Istio/Linkerd | High | Excellent | High | Existing mesh |
| Feature flags | High (product) | Excellent | Med | Product A/B |

---

## 10. Quick flow

```text
Define SLO + abort rules
  → Instrument metrics by version
  → Pick router (Ingress/Gateway/Mesh) + controller (Rollouts/Flagger)
  → Canary: 5% → analyze → step → 100% → promote
  → A/B: sticky cohorts + business metrics + guardrails
  → Always: one-step rollback, practiced
```

---

## References

- [Argo Rollouts – Canary](https://argoproj.github.io/argo-rollouts/features/canary/)
- [Argo Rollouts – Analysis](https://argoproj.github.io/argo-rollouts/features/analysis/)
- [Flagger](https://docs.flagger.app/)
- [Ingress-NGINX canary annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)
- [Gateway API HTTPRoute weight](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/)
- [Kubernetes Deployments (rolling)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Google – canary releases](https://cloud.google.com/architecture/application-deployment-and-testing-strategies)
