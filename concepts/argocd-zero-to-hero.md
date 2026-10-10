# Argo CD from Zero to Hero

Learn **Argo CD** end-to-end: first install, first Application, App of Apps, Helm/Kustomize, sync/health, then **production practices from real operations**.

Deep production checklist (HA, AppProjects, SSO, multi-cluster): [argocd-production.md](./argocd-production.md)

Related: [helm-zero-to-hero.md](./helm-zero-to-hero.md) · [kustomize-zero-to-hero.md](./kustomize-zero-to-hero.md) · [canary-ab-testing.md](./canary-ab-testing.md) · [secure-api-access-bastion.md](./secure-api-access-bastion.md) · [kubernetes-secrets.md](./kubernetes-secrets.md)

Official: [Getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/) · [User guide](https://argo-cd.readthedocs.io/en/stable/user-guide/)

---

## 0. Mental model

Argo CD continuously compares **desired state in Git** to **live state in the cluster** and can sync (apply) to converge them.

```text
Git (desired)  ←→  Argo CD  ←→  Kubernetes (live)
     ↑ sync / self-heal / prune
```

| Term | Meaning |
|------|---------|
| **Application** | CR that points at a Git path (or Helm chart) + destination cluster/namespace |
| **AppProject** | Tenancy boundary: which repos, namespaces, and resource kinds an app may use |
| **Sync** | Apply Git → cluster |
| **OutOfSync** | Live differs from Git |
| **Healthy / Degraded** | Health of resources (Deployments Ready, etc.) |
| **Self-heal** | Auto-revert manual drift toward Git |
| **Prune** | Delete resources removed from Git |
| **App of Apps** | One root Application that syncs other Application manifests |
| **ApplicationSet** | Generator that creates many Applications (envs, clusters, teams) |

**CI vs CD:** CI builds/tests/pushes images and **commits digests to Git**. Argo CD only syncs. Don’t let Jenkins/`helm upgrade` fight Argo on the same objects.

---

## 1. Zero — install and log in

### 1.1 Lab install (non-HA)

```bash
kubectl create namespace argocd

# Pin a version in real use — example:
export ARGOCD_VERSION=v2.13.0
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
kubectl -n argocd get pods
```

### 1.2 Access the UI in a browser (lab)

Argo CD serves HTTPS on the `argocd-server` Service (port **443**). Until you add Ingress/SSO, the usual lab path is **port-forward** + the initial admin password.

#### Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Username is always **`admin`** for this secret. Change it after first login:

```bash
argocd account update-password
# or via UI: User Info → Update password
```

If the secret is missing, it was already deleted (common after SSO hardening)—use SSO or recover admin per Argo CD docs.

#### Port-forward (this machine only)

```bash
# Listen on localhost only (safest default)
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Browser: **https://localhost:8080** (accept the self-signed cert warning in lab).

CLI:

```bash
# Install CLI (if needed)
curl -fsSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

argocd login localhost:8080 --username admin --insecure
argocd account get-user-info
```

Leave the port-forward running in that terminal; stop with `Ctrl+C`.

#### Port-forward reachable from other hosts on your network

```bash
# Bind on all interfaces — anyone who can reach this machine:8080 can hit Argo CD
kubectl -n argocd port-forward svc/argocd-server 8080:443 --address 0.0.0.0
```

Then open `https://<your-machine-ip>:8080` from another laptop.

| Risk | Mitigation |
|------|------------|
| Binds on **all NICs** (LAN/Wi‑Fi) | Use only on trusted networks / short lab sessions |
| No SSO, self-signed TLS | Lab only — not a production exposure path |
| Forwards as your kube credentials | Treat like sharing admin console access |

Prefer SSH tunnel from a remote client instead of `0.0.0.0` when possible:

```bash
# On your laptop (Argo stays private on the jump/bastion host)
ssh -N -L 8080:localhost:8080 user@bastion
# On bastion: kubectl -n argocd port-forward svc/argocd-server 8080:443
# Browser on laptop: https://localhost:8080
```

See [secure-api-access-bastion.md](./secure-api-access-bastion.md).

### 1.3 Best practice: make the UI accessible (production)

**Do not** leave `port-forward --address 0.0.0.0` or a public NodePort as the long-term UI path. Treat the Argo CD UI like a **privileged control plane console**.

#### Recommended pattern

```text
Browser
  → HTTPS Ingress (argocd.example.com)
  → private / VPN / corp network only (or SSO-aware proxy)
  → argocd-server Service
  → OIDC/SSO login (not long-lived admin)
```

| Approach | When | Notes |
|----------|------|--------|
| **Ingress + TLS + SSO** on **private** DNS/VPN | Production default | Best balance of UX and control |
| **VPN / Tailscale** then ClusterIP Ingress or port-forward | Strong security, simple | No public UI at all |
| **Identity-aware proxy** (Cloudflare Access, IAP, Teleport) | Enterprise | MFA + audit in front of UI |
| **LoadBalancer + MetalLB** (internal pool) | Bare metal | Same as Ingress backend VIP; keep VIP off the public Internet |
| **port-forward / SSH tunnel** | Break-glass / lab | OK temporarily; document as emergency only |
| Public Ingress with only `admin` password | **Avoid** | Credential stuffing + stolen kubeconfig blast radius |

#### Ingress example (sketch)

```bash
# After CNI + MetalLB/Ingress controller exist — see ingress-zero-to-hero.md
# Point DNS argocd.example.com → internal VIP (VPN-only) or IAP-protected entry
```

Helm values sketch (`argo-cd` chart — verify keys for your chart version):

```yaml
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-server-tls
        hosts:
          - argocd.example.com
  # When TLS terminates at Ingress, charts often use:
  # extraArgs:
  #   - --insecure
configs:
  params:
    server.insecure: true   # only with TLS at Ingress — follow chart docs
  cm:
    url: https://argocd.example.com
    # oidc.config: |
    #   name: SSO
    #   issuer: https://dex.example.com
    #   clientID: argocd
    #   ...
```

cert-manager annotation on the Ingress for the TLS Secret; OIDC as in [oidc-sso-setup.md](./oidc-sso-setup.md).

#### Production checklist for UI access

- [ ] UI hostname on **internal DNS** or behind VPN/IAP — not world-open unless risk-accepted  
- [ ] **TLS** (cert-manager); HSTS at Ingress  
- [ ] **SSO/OIDC** + Argo RBAC groups; disable/delete local admin after SSO works  
- [ ] Delete `argocd-initial-admin-secret` once SSO is verified  
- [ ] MFA at the IdP  
- [ ] NetworkPolicy: only Ingress controller (or VPN CIDR) can reach `argocd-server`  
- [ ] Audit logins; alert on repeated auth failures  
- [ ] Break-glass: documented port-forward/SSH tunnel for platform on-call  
- [ ] Same rule as the Kubernetes API: **apps** are public via Ingress; **Argo UI** stays privileged  

```bash
# After SSO works — remove bootstrap admin secret
kubectl -n argocd delete secret argocd-initial-admin-secret --ignore-not-found

# Confirm who you are
argocd account get-user-info
```

**Experience:** a public Argo UI with only `admin` + initial password is a common breach path. Prefer “VPN + SSO” or “IAP + SSO”; keep port-forward for break-glass.

More detail: [argocd-production.md](./argocd-production.md) §2–4 · [domain-dns-ingress-ha.md](./domain-dns-ingress-ha.md) · [secure-api-access-bastion.md](./secure-api-access-bastion.md).

### 1.4 Helm install (closer to prod)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd \
  -n argocd --create-namespace \
  --version 7.7.0 \
  --atomic --wait
```

Use HA values before production traffic ([argocd-production.md](./argocd-production.md) §2).

---

## 2. First Application (hero step 1)

### 2.1 Tiny Git repo layout

```text
gitops-demo/
  apps/
    guestbook/
      deployment.yaml
      service.yaml
```

Or use the public example:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync guestbook
argocd app get guestbook
kubectl get deploy,svc -n default -l app.kubernetes.io/instance=guestbook
```

### 2.2 Declarative Application (prefer this over UI click-ops)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default          # replace with a tight AppProject ASAP
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application-guestbook.yaml
argocd app sync guestbook
argocd app wait guestbook --health --timeout 180
```

**Experience:** anything created only in the UI becomes invisible tribal knowledge. If the Argo CD DB is lost and it’s not in Git, the app definition is gone. **Declarative from day one.**

---

## 3. Sync, health, drift — day-to-day

```bash
argocd app list
argocd app get myapp
argocd app diff myapp          # Git vs live
argocd app sync myapp
argocd app sync myapp --prune
argocd app history myapp
argocd app rollback myapp <id>
argocd app logs myapp -f
```

| Sync status | Meaning |
|-------------|---------|
| Synced | Live matches Git |
| OutOfSync | Drift or new commit not applied |
| Unknown | Repo/cluster connectivity issue |

| Health | Meaning |
|--------|---------|
| Healthy | Controllers report OK |
| Progressing | Rollout in progress |
| Degraded | e.g. Deployment failing |
| Missing | Expected resource absent |

### Auto-sync (enable carefully)

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

**Experience:**

- Turn on **selfHeal** only when Git is trusted; otherwise you fight firefighters who patched live.  
- **Prune** deletes removed resources—great when you own the namespace; catastrophic if shared leftovers aren’t in Git.  
- Start with **manual sync** in prod for databases/operators; automate staging first.

### Ignore noisy fields (HPA / Rollouts)

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

Or let the controller own replicas and don’t sync-fight ([metrics-server-hpa.md](../metrics-server-hpa.md), [canary-ab-testing.md](./canary-ab-testing.md)).

---

## 4. Sources: plain YAML, Kustomize, Helm

### 4.1 Directory of manifests

```yaml
source:
  repoURL: https://git.example.com/gitops.git
  targetRevision: main
  path: teams/payments/raw
```

### 4.2 Kustomize overlay

```yaml
source:
  repoURL: https://git.example.com/gitops.git
  targetRevision: main
  path: teams/payments/overlays/prod
```

Argo runs `kustomize build` on that path. Pin Kustomize version in `argocd-cm` if builds differ from CI.

### 4.3 Helm (chart in Git or repo)

```yaml
source:
  repoURL: https://git.example.com/gitops.git
  path: charts/api
  targetRevision: main
  helm:
    releaseName: payments-api
    valueFiles:
      - values.yaml
      - values-prod.yaml
    parameters:
      - name: image.tag
        value: "1.4.2"
```

Remote chart:

```yaml
source:
  chart: ingress-nginx
  repoURL: https://kubernetes.github.io/ingress-nginx
  targetRevision: 4.11.1
  helm:
    values: |
      controller:
        replicaCount: 2
```

**Experience:** always **pin `targetRevision` chart versions**. Floating tags caused surprise breakages more than once. Prefer image **digests** written by CI into values.

### 4.4 Multiple sources (chart + values repo)

Supported in modern Argo CD—values from one repo, chart from another. Use when platform owns charts and app teams own values.

---

## 5. App of Apps (hero step 2)

Bootstrap the platform declaratively:

```text
gitops/
  apps/
    root.yaml              # Application → path apps/
    ingress-nginx.yaml
    cert-manager.yaml
    payments-api.yaml
```

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

Sync waves for order:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # CRDs / operators first
# apps use "0", "10", "20" …
```

**Experience:** root app + waves beat “remember to sync cert-manager before Certificates.” Put waves on Application manifests and on resources that need ordering.

---

## 6. ApplicationSet (hero step 3)

Generate apps per environment/cluster without copy-paste:

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
            namespace: payments-staging
          - env: prod
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
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

Other generators: Git folders, clusters list, matrix, SCM providers (PRs).

**Experience:** ApplicationSets are powerful; a bad generator template can spawn dozens of broken apps. Test with a dry generator in staging; use `syncPolicy.applicationsSync` carefully when deleting elements.

---

## 7. Projects, RBAC, SSO (leave default ASAP)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  sourceRepos:
    - https://git.example.com/gitops.git
  destinations:
    - namespace: payments-*
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
    - group: networking.k8s.io
      kind: Ingress
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

```csv
# argocd-rbac-cm policy.csv sketch
p, role:payments-dev, applications, get, payments/*, allow
p, role:payments-dev, applications, sync, payments/*, allow
g, oidc:payments-dev, role:payments-dev
g, oidc:platform, role:admin
```

**Experience:** the `default` project with `*` destinations is how juniors sync into `kube-system`. Create **AppProjects before** the second team arrives. SSO (OIDC) + disable local admin—see [oidc-sso-setup.md](./oidc-sso-setup.md) and [argocd-production.md](./argocd-production.md).

---

## 8. Secrets

Argo CD syncs Secret *manifests* if they appear in Git—**don’t put plaintext there**.

| Approach | Notes |
|----------|-------|
| External Secrets Operator | Git has `ExternalSecret`; vault has values |
| Sealed Secrets / SOPS | Encrypted blobs decrypt in-cluster |
| Ignore Secrets; inject at runtime | Vault agent / CSI |

**Experience:** “temporary” plaintext Secret in Git lives forever in history. Assume leaked; rotate.

---

## 9. Multi-cluster

```bash
argocd cluster add <context> --name workload-prod
argocd cluster list
```

Application destination:

```yaml
destination:
  name: workload-prod          # or server: https://...
  namespace: payments-prod
```

**Experience:** hub Argo CD in a **management cluster** scales better than one Argo per cluster—but protect the hub (private API, SSO, backups). If the hub dies, apps keep running; GitOps pauses—document break-glass kubectl via bastion ([secure-api-access-bastion.md](./secure-api-access-bastion.md)).

---

## 10. Production best practices (from experience)

### 10.1 Git as the only truth

- [ ] No production `kubectl apply` / `helm upgrade` on managed apps  
- [ ] Hotfixes: PR to Git (or break-glass then **commit the fix**)  
- [ ] Branch protection + required reviews on `main` / `prod`  

**Lived failure:** engineer “fixed” prod with kubectl; selfHeal reverted; outage loop until they understood Git was winning.

### 10.2 Environment promotion

```text
PR → staging overlay auto-sync
  → soak / tests
  → PR merge to prod overlay (image digest bump)
  → prod sync (auto or manual gate)
```

Avoid “same branch, same path” for staging and prod with only a manual sync hope.

### 10.3 Image updates

CI should write immutable refs:

```yaml
image: ghcr.io/acme/api@sha256:abc…
# or tag + digest digest pinned in values
```

Optional [Image Updater](https://argocd-image-updater.readthedocs.io/) with **write-back to Git** and allowlists—not silent prod latest.

### 10.4 Health ≠ Sync

Synced + Degraded = Git applied but app broken. Wire notifications on **health** and sync failures. Page on Degraded for tier-1 apps.

### 10.5 Repo-server and big monorepos

- [ ] Shallow clone / exclude unused paths when possible  
- [ ] Don’t put huge binaries in GitOps repos  
- [ ] Scale `repo-server` replicas under load  
- [ ] Cache / OCI Helm charts instead of re-fetching endlessly  

**Lived failure:** monorepo with generated bloat → sync timeouts; split platform vs team repos.

### 10.6 CRDs and operators

- [ ] Sync CRDs / operators in earlier **sync-wave** than CRs  
- [ ] Helm CRD hooks are tricky—know whether Argo + Helm will upgrade CRDs  
- [ ] Install operators before restoring apps (Velero)  

### 10.7 Finalizers & delete

`resources-finalizer.argocd.argoproj.io` deletes app resources when the Application is deleted. Great for ephemeral envs; dangerous if someone deletes the wrong Application in prod.

**Experience:** protect prod Applications with RBAC; consider omitting finalizers on tier-0 platform apps or requiring dual control.

### 10.8 Drift that is “allowed”

Document exceptions: HPA replicas, cert-manager Secret `tls.crt` data, ClusterIP allocations. Use `ignoreDifferences` / SSA field managers deliberately.

### 10.9 Upgrades

- [ ] Read release notes; staging hub first  
- [ ] Pin manifest/chart version in Git  
- [ ] Backup `argocd` Secrets/ConfigMaps before upgrade  

### 10.10 Observability of Argo itself

- [ ] Metrics + alerts: sync failures, OutOfSync age, repo errors, controller queue  
- [ ] Audit who synced (SSO username in logs)  

### 10.11 Don’t dual-drive

Pick **one**: Argo CD **or** Helmfile-from-CI for a given release. Dual controllers cause perpetual OutOfSync and blame games.

### 10.12 Progressive delivery

For safer prod deploys, pair with Argo Rollouts / Flagger ([canary-ab-testing.md](./canary-ab-testing.md))—Argo CD syncs the Rollout object; Rollouts shifts traffic.

---

## 11. Minimal GitOps repo that scales

```text
gitops/
  apps/                          # Application + ApplicationSet YAMLs
    root.yaml
    platform-appset.yaml
    team-payments-appset.yaml
  platform/
    ingress-nginx/overlays/prod/
    cert-manager/overlays/prod/
    monitoring/overlays/prod/
  teams/
    payments/
      api/base/
      api/overlays/staging/
      api/overlays/prod/
  README.md                      # who owns what; sync policy; break-glass
```

---

## 12. Day-2 command sheet

```bash
# Apps
argocd app list
argocd app sync APP [--prune] [--dry-run]
argocd app diff APP
argocd app wait APP --sync --health --timeout 300
argocd app history APP
argocd app rollback APP ID
argocd app set APP --sync-policy automated
argocd app set APP --sync-policy none

# Repos / clusters / projects
argocd repo list
argocd repo add REPO --ssh-private-key-path KEY
argocd cluster list
argocd proj list

# Kubernetes view
kubectl -n argocd get applications,appprojects,applicationsets
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

---

## 13. Learning path (zero → hero)

```text
1. Install Argo CD; login; sync guestbook
2. Declarative Application YAML in Git
3. Auto-sync on a staging app; practice diff/prune
4. Kustomize overlay path; Helm chart + pinned version
5. App of Apps root; sync waves
6. AppProject lockdown; SSO
7. ApplicationSet for staging/prod
8. IgnoreDifferences for HPA; ServerSideApply
9. Hub multi-cluster; notifications
10. HA install; backup; Image digests; canary via Rollouts
11. Live by Git—break-glass then commit
```

---

## 14. Acceptance checks before calling it “prod”

- [ ] SSO works; local admin disabled/rotated  
- [ ] Team cannot sync outside their AppProject  
- [ ] Staging auto-sync; prod gated or tightly reviewed  
- [ ] Self-heal demo: edit live Deployment → reverts (or OutOfSync if manual)  
- [ ] Root app recreates child Applications after delete (staging drill)  
- [ ] Secrets not plaintext in Git  
- [ ] Alerts on sync fail / Degraded  
- [ ] Documented break-glass without public `:6443`  

Full checklist: [argocd-production.md](./argocd-production.md).

---

## 15. Quick flow

```text
Install Argo CD → login
  → Application (declarative) → sync
  → GitOps repo + App of Apps
  → AppProjects + SSO
  → ApplicationSets for envs
  → Auto-sync staging; promote digests to prod
  → HA, metrics, backup, no dual controllers
```

---

## References

- [Argo CD getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Applications](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Best practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Sync options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [argocd-production.md](./argocd-production.md) (this repo)
- [argo-helm chart](https://github.com/argoproj/argo-helm)
