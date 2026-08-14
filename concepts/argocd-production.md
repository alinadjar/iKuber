# Argo CD Production Setup — Best Practices

How to install and operate **Argo CD** for GitOps in production: HA, security, repo design, sync policy, multi-cluster, and day-2 ops.

Related:

- [helm-zero-to-hero.md](./helm-zero-to-hero.md)
- [developer-access-rbac.md](./developer-access-rbac.md)
- [oidc-sso-setup.md](./oidc-sso-setup.md)
- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

Official: [Argo CD docs](https://argo-cd.readthedocs.io/) · [Operator manual](https://argo-cd.readthedocs.io/en/stable/operator-manual/)

---

## 0. What Argo CD is (and is not)

| Argo CD does | Argo CD does not (by itself) |
|--------------|------------------------------|
| Sync cluster state **from Git** (desired → live) | Replace CI (build/test/scan images) |
| Continuous drift detection & heal | Store secrets in Git unencrypted |
| Multi-cluster deploy from one control plane | Replace Helm/Kustomize (it *uses* them) |
| RBAC + SSO for who can sync/apps | Replace Kubernetes RBAC on target clusters |

```text
CI builds & pushes images + updates Git (tag/digest)
  → Argo CD detects Git commit
  → syncs manifests/Helm/Kustomize to cluster(s)
  → Kubernetes runs workloads
```

**Pick one deployer for a given release:** Argo CD **or** Helmfile-apply-from-CI — not both fighting over the same objects.

---

## 1. Architecture decisions (make these first)

### 1.1 Topology

| Pattern | When |
|---------|------|
| **Management / hub cluster** runs Argo CD; deploys to many workload clusters | Production standard |
| Argo CD **on each** cluster | Small / isolated blast radius; more to upgrade |
| Argo CD on the **same** prod cluster it manages | OK for single-cluster; protect `argocd` NS heavily |

### 1.2 HA vs single replica

**Production:** run Argo CD in **HA** mode (multiple `argocd-server`, `repo-server`, `application-controller` shards as documented for your version). Use the [HA install manifests](https://argo-cd.readthedocs.io/en/stable/operator-manual/high-availability/) or Helm chart with HA values.

### 1.3 Git layout

Prefer:

```text
gitops/
  apps/                     # Application / ApplicationSet YAMLs (bootstrapped)
  platform/                 # ingress, cert-manager, policy…
    base/
    overlays/prod/
  teams/
    payments/
      base/
      overlays/staging/
      overlays/prod/
  charts/                   # optional local charts
```

**App of Apps** or **ApplicationSet** owns team apps; humans rarely click “New App” in the UI for prod.

---

## 2. Install (production-oriented)

### 2.1 Prerequisites

- [ ] Dedicated namespace (usually `argocd`)  
- [ ] Ingress / LB + TLS for UI/API (`argocd.example.com`)  
- [ ] OIDC/SSO ready ([oidc-sso-setup.md](./oidc-sso-setup.md))  
- [ ] Git repos (platform + app) with branch protection  
- [ ] Network path: Argo CD → Git, OCI Helm repos, target cluster API servers  

### 2.2 Install via official HA manifests (example)

Pin a **specific Argo CD version** (do not float `stable` in prod long-term without pinning the commit/tag you tested):

```bash
kubectl create namespace argocd

# Example — replace VERSION with a release you have reviewed
export ARGOCD_VERSION=v2.13.0
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/ha/install.yaml

kubectl -n argocd get pods
kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
```

### 2.3 Install via Helm (often easier for values/HA)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Pull and review values for your chart version
helm show values argo/argo-cd --version 7.7.0 > /tmp/argocd-values-default.yaml
```

Sketch of production-leaning values (adjust to current chart keys):

```yaml
# values-argocd-prod.yaml (illustrative — verify against chart version)
crds:
  install: true

redis-ha:
  enabled: true

controller:
  replicas: 1          # or HA sharding per current docs
  # resources: set requests/limits

server:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-server-tls
        hosts: [argocd.example.com]
  # Behind TLS-terminating ingress, often:
  # extraArgs:
  #   - --insecure   # ONLY if ingress does TLS and you understand the tradeoff

repoServer:
  replicas: 2

configs:
  params:
    server.insecure: true   # common with external TLS; prefer documented pattern for your chart
  cm:
    # url: https://argocd.example.com
    # oidc.config: |
    #   name: SSO
    #   issuer: https://dex.example.com
    #   clientID: argocd
    #   ...
  rbac:
    create: true
    # policy.csv set via chart or ConfigMap — see §4
```

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 7.7.0 \
  -f values-argocd-prod.yaml \
  --atomic --wait --timeout 15m
```

Bootstrap Argo CD **with Argo CD** after the first install (app-of-apps pointing at your `gitops` repo) so the install itself becomes declarative.

### 2.4 CLI

```bash
# Install argocd CLI matching server minor version
curl -fsSL -o argocd https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

argocd login argocd.example.com --sso
# or: argocd login argocd.example.com --username admin
```

Initial admin password (manifest install):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
# Disable / rotate local admin after SSO works — see §4
```

---

## 3. Production best practices (checklist)

### 3.1 Availability & sizing

- [ ] HA install (multiple server/repo-server; Redis HA if required by mode)  
- [ ] Set **CPU/memory requests/limits** for all components  
- [ ] PodDisruptionBudgets for server/repo-server  
- [ ] Prefer dedicated nodes or priority for controllers on busy hubs  
- [ ] Monitor queue depth / reconcile time; scale controller shards when needed  

### 3.2 Git as source of truth

- [ ] **No manual kubectl apply** on managed objects in prod  
- [ ] Protected branches; required PR reviews; signed commits if required  
- [ ] Separate repos or paths: `platform` vs `teams/*` (blast radius)  
- [ ] Environment promotion: `staging` → `prod` via PR / ApplicationSet (not sync from `main` unchecked)  
- [ ] Pin **chart versions**, **image digests**, and **manifest revisions** (Git SHA)  

### 3.3 Sync policy

| Setting | Production guidance |
|---------|---------------------|
| `automated.prune` | Enable for apps you fully own; **disable** until confident for shared NS |
| `automated.selfHeal` | Enable to fight drift; ensure Git is correct first |
| `syncOptions.CreateNamespace=true` | OK if NS lifecycle is Git-owned |
| `RespectIgnoreDifferences` | Use for known noisy fields (e.g. HPA replicas) |
| Manual sync | Prefer for first prod cutovers / data stores |

Example Application snippet:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: payments
  source:
    repoURL: https://git.example.com/gitops.git
    targetRevision: main          # or release branch / tag for prod
    path: teams/payments/overlays/prod
  destination:
    server: https://kubernetes.default.svc   # or external cluster API
    namespace: payments-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Server-Side Apply** reduces field-ownership fights with other controllers—prefer it on modern clusters.

### 3.4 Ignore differences (stop sync storms)

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # when HPA owns replicas
```

Or use `RespectIgnoreDifferences=true` sync option with managed fields carefully documented.

---

## 4. Security (non-negotiable in prod)

### 4.1 SSO + disable local admin

- [ ] Configure **OIDC/SAML** (Dex, Keycloak, Okta, Azure AD, …)  
- [ ] Map IdP groups → Argo CD RBAC roles  
- [ ] Remove or disable `admin` after SSO verified  
- [ ] MFA at IdP  

Argo CD RBAC (`argocd-rbac-cm`) sketch:

```csv
p, role:payments-dev, applications, get, payments/*, allow
p, role:payments-dev, applications, sync, payments/*, allow
p, role:payments-dev, logs, get, payments/*, allow
p, role:payments-admin, applications, *, payments/*, allow
g, oidc:payments-dev, role:payments-dev
g, oidc:platform, role:admin
```

```bash
# After SSO works — rotate/delete initial admin secret per current Argo CD docs
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### 4.2 AppProjects (mandatory multi-tenant boundary)

Never leave everything in the default project unrestricted.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  description: Payments team apps
  sourceRepos:
    - https://git.example.com/gitops.git
  destinations:
    - namespace: payments-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: "apps"
      kind: StatefulSet
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: "networking.k8s.io"
      kind: Ingress
    - group: "networking.k8s.io"
      kind: NetworkPolicy
  # Deny cluster-scoped surprises:
  clusterResourceBlacklist:
    - group: "*"
      kind: "*"
  # Better: explicit whitelist only (above). Tighten over time.
```

- [ ] Per-team **AppProject** with repo + destination + resource allowlists  
- [ ] Platform project separate from product teams  
- [ ] Deny `*` cluster resources for app teams  

### 4.3 Secrets

- [ ] **Do not** commit plaintext secrets  
- [ ] Use Sealed Secrets, SOPS + KSOPS/helm-secrets, External Secrets Operator, or Vault sidecar/CSI  
- [ ] Prefer External Secrets that sync from a vault; Git holds only references  
- [ ] Restrict who can `argocd` decrypt / view secrets in UI (RBAC)  

### 4.4 Network & exposure

- [ ] UI only on internal ingress / VPN / SSO-aware proxy  
- [ ] NetworkPolicies: limit egress from `repo-server` to Git/OCI/DNS; limit who can reach Redis  
- [ ] Keep `argocd-server` off the public internet when possible  
- [ ] TLS everywhere (ingress cert-manager)  

### 4.5 Repo and cluster credentials

- [ ] Deploy keys / GitHub Apps with **read-only** Git access where possible  
- [ ] Fine-grained tokens; rotate on schedule  
- [ ] Cluster registration credentials: least privilege (avoid full cluster-admin if your model allows a deploy SA)  
- [ ] Store repo/cluster secrets as Kubernetes Secrets; enable encryption at rest  

```bash
argocd repo add https://git.example.com/gitops.git \
  --ssh-private-key-path ~/.ssh/argocd-gitops-ro

argocd cluster add <context> --name prod-workload
```

Prefer declaring repos/clusters via **Secrets** in Git (Sealed) or the declarative config management approach in Argo CD docs—not one-off laptop CLI for prod.

### 4.6 Image & supply chain

- [ ] CI updates **digests** in Git; Argo only syncs  
- [ ] Optional: [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/) with strict allowlists (write-back to Git)  
- [ ] Scan images in CI before Git bump  
- [ ] Quay/ECR/GHCR pull secrets for `repo-server` if private Helm OCI  

---

## 5. Application bootstrap patterns

### 5.1 Root app (App of Apps)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://git.example.com/gitops.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`apps/` contains one Application YAML per system (or ApplicationSets).

### 5.2 ApplicationSet (many clusters / tenants)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: staging
            cluster: https://kubernetes.default.svc
            namespace: payments-staging
          - env: prod
            cluster: https://kubernetes.default.svc
            namespace: payments-prod
  template:
    metadata:
      name: 'payments-{{env}}'
    spec:
      project: payments
      source:
        repoURL: https://git.example.com/gitops.git
        targetRevision: main
        path: 'teams/payments/overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

### 5.3 Helm apps from Argo CD

```yaml
spec:
  source:
    repoURL: https://git.example.com/gitops.git
    path: teams/payments/chart
    targetRevision: main
    helm:
      releaseName: payments-api
      valueFiles:
        - values.yaml
        - values-prod.yaml
      # or:
      # values: |
      #   replicaCount: 3
```

Pin chart deps in `Chart.lock`. For remote charts, use declarative Helm repos and **version pins**.

---

## 6. Progressive delivery & change safety

- [ ] Sync **staging** automatically; **prod** via PR to prod overlay / manual sync gate for critical systems  
- [ ] Use [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) for canary/blue-green when needed  
- [ ] Sync waves / hooks for ordered platform deps (`argocd.argoproj.io/sync-wave`)  
- [ ] Pre-prod `argocd app diff` in CI (read-only token)  

```bash
argocd app diff payments-api-prod
argocd app sync payments-api-prod --dry-run
argocd app sync payments-api-prod --prune
argocd app history payments-api-prod
argocd app rollback payments-api-prod <id>
```

Sync waves example:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "10"   # lower first: -1 CRDs, 0 NS, 10 operators, 20 apps
```

---

## 7. Multi-cluster

- [ ] Register workload clusters with unique names  
- [ ] Network: hub → `target:6443` allowed; least privilege kubeconfig  
- [ ] Destination restrictions in AppProject per cluster  
- [ ] Failover: document what happens if hub is down (apps keep running; GitOps pauses)  
- [ ] Avoid Argo CD managing itself **and** critical data planes without backup access (break-glass kubeconfig)  

```bash
argocd cluster list
argocd cluster add workload-prod --kubeconfig /path/to/workload.kubeconfig
```

---

## 8. Observability, backup, upgrades

### 8.1 Metrics & alerts

- [ ] Scrape Argo CD metrics (Prometheus)  
- [ ] Alert: `OutOfSync` prolonged, sync failures, repo-server errors, controller workqueue depth  
- [ ] Log aggregation for `argocd-server` / `application-controller` / `repo-server`  

### 8.2 Backup

Back up at least:

- Argo CD namespace Secrets (repo, cluster, redis, notifications)  
- ConfigMaps (`argocd-cm`, `argocd-rbac-cm`, `argocd-cmd-params-cm`)  
- Application / AppProject / ApplicationSet CRs (ideally already in Git)  

If Applications are fully in Git, disaster recovery = reinstall Argo CD + reconnect repos + sync root app.

### 8.3 Upgrades

- [ ] Read release notes; upgrade **minor by minor** when required  
- [ ] Staging hub first  
- [ ] Pin chart/manifest version; PR the bump  
- [ ] Ensure CRDs upgrade per chart instructions  

```bash
helm upgrade argocd argo/argo-cd -n argocd --version <new> -f values-argocd-prod.yaml --atomic --wait
```

---

## 9. Notifications (optional but useful)

Use [Argo CD Notifications](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/) for Slack/Teams/email on sync fail / health degrade. Keep tokens in Secrets; don’t spam on every healthy sync.

---

## 10. Anti-patterns

| Avoid | Prefer |
|-------|--------|
| ClickOps apps in UI for prod | Declarative Application / ApplicationSet in Git |
| `project: default` with `*` destinations | Tight AppProjects |
| Auto-prune everywhere on day one | Enable after ownership is clear |
| Helmfile CI **and** Argo managing same release | One controller of record |
| Long-lived admin password | SSO + disable local admin |
| Floating `targetRevision: HEAD` on prod with force-push | Protected branch / tags / release branches |
| Cluster-admin for every team deploy key | Least privilege + project allowlists |
| Secrets in plain Git | ESO / SOPS / Sealed Secrets |

---

## 11. Day-0 → day-2 runbook

```text
1. Decide hub topology + Git layout
2. Install HA Argo CD (pin version) + Ingress TLS
3. Configure SSO + AppProjects + RBAC; disable local admin
4. Register Git repos (read-only) + clusters
5. Commit root Application + platform apps
6. Add team ApplicationSets with tight projects
7. Enable automated sync on staging; gated prod
8. Metrics, alerts, backup of argocd secrets/config
9. Document break-glass kubeconfig + upgrade SOP
```

### Useful commands

```bash
kubectl -n argocd get applications,appprojects,applicationsets
argocd app list
argocd app get payments-api-prod
argocd app sync payments-api-prod
argocd app wait payments-api-prod --health --timeout 300
argocd proj list
argocd repo list
argocd cluster list
```

---

## 12. Minimal production acceptance tests

- [ ] SSO login works; local admin disabled/rotated  
- [ ] Team user can sync only their project  
- [ ] Team user **cannot** deploy to `kube-system` / other teams  
- [ ] Kill a pod → selfHeal restores from Git  
- [ ] Edit live Deployment → selfHeal reverts (or OutOfSync if manual)  
- [ ] Force bad Git → sync fails safely; alert fires  
- [ ] Root app recovers Argo CD config after reinstall drill (staging)  

---

## References

- [Argo CD documentation](https://argo-cd.readthedocs.io/)
- [High availability](https://argo-cd.readthedocs.io/en/stable/operator-manual/high-availability/)
- [Security](https://argo-cd.readthedocs.io/en/stable/operator-manual/security/)
- [RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [AppProject](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Best practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Declarative setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [Helm chart (argo-helm)](https://github.com/argoproj/argo-helm)
