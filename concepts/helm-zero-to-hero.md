# Helm from Zero to Hero (+ Helmfile)

Install, use, author, and operate **Helm** charts in production, then manage many releases with **Helmfile**.

Related: [kubeadm-production-cluster.md](../kubeadm-production-cluster.md) · [crd-operators.md](./crd-operators.md) · [developer-access-rbac.md](./developer-access-rbac.md)

Official: [Helm docs](https://helm.sh/docs/) · [Helmfile](https://helmfile.readthedocs.io/)

---

## 0. What Helm is

Helm is the **package manager for Kubernetes**.

| Concept | Meaning |
|---------|---------|
| **Chart** | Package of templates + default values (`Chart.yaml`, `templates/`, `values.yaml`) |
| **Release** | A chart **installed** into a cluster with a name (e.g. `ingress-nginx`) |
| **Repository** | HTTP index of charts (`helm repo add`) |
| **Values** | Config that fills templates (`--set`, `-f values.yaml`) |
| **Revision** | Each upgrade/rollback creates a new revision you can roll back to |

```text
Chart + Values  →  helm install/upgrade  →  Kubernetes objects (Deployments, Services, …)
                                          →  release secret/configmap in namespace
```

---

## 1. Install Helm (and friends)

### 1.1 Helm CLI

```bash
# Linux (script)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Or package managers:
# sudo snap install helm --classic
# brew install helm
```

Bash completion:

```bash
helm completion bash | sudo tee /etc/bash_completion.d/helm >/dev/null
source ~/.bashrc
```

### 1.2 Optional but useful

```bash
# Plugin examples
helm plugin install https://github.com/databus23/helm-diff
helm plugin install https://github.com/aslafy-z/helm-git

# Helmfile (see §9)
# Linux amd64 example — pin a release from https://github.com/helmfile/helmfile/releases
# curl -fsSL -o helmfile.tar.gz <release-url>
# sudo tar xz -C /usr/local/bin -f helmfile.tar.gz helmfile
```

Requires working `kubectl` context:

```bash
kubectl config current-context
kubectl get ns
```

---

## 2. Zero → first release (30 minutes)

### 2.1 Add a chart repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm repo list
helm search repo ingress-nginx
helm show chart ingress-nginx/ingress-nginx
helm show values ingress-nginx/ingress-nginx | less
```

### 2.2 Install with defaults (lab only)

```bash
kubectl create namespace demo-helm

helm install my-nginx bitnami/nginx -n demo-helm
helm list -n demo-helm
kubectl get all -n demo-helm
```

### 2.3 Install with your values (normal path)

```bash
cat > nginx-values.yaml <<'EOF'
service:
  type: ClusterIP
replicaCount: 2
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
EOF

helm upgrade --install my-nginx bitnami/nginx \
  -n demo-helm \
  -f nginx-values.yaml \
  --wait --timeout 5m

helm status my-nginx -n demo-helm
helm get values my-nginx -n demo-helm
helm get manifest my-nginx -n demo-helm | less
```

`upgrade --install` (aka **upsert**) is the production default: create or update in one command.

### 2.4 Upgrade, history, rollback

```bash
# Change values and upgrade
helm upgrade my-nginx bitnami/nginx -n demo-helm -f nginx-values.yaml --set replicaCount=3

helm history my-nginx -n demo-helm
helm rollback my-nginx 1 -n demo-helm    # back to revision 1
helm history my-nginx -n demo-helm
```

### 2.5 Uninstall

```bash
helm uninstall my-nginx -n demo-helm
# Some charts leave PVCs / CRDs behind — check docs and clean intentionally
kubectl get pvc -n demo-helm
```

### 2.6 Dry-run & debug (always before prod)

```bash
helm upgrade --install my-nginx bitnami/nginx \
  -n demo-helm -f nginx-values.yaml \
  --dry-run --debug

# Render templates locally without touching the cluster
helm template my-nginx bitnami/nginx -n demo-helm -f nginx-values.yaml > /tmp/out.yaml
kubectl apply --dry-run=server -f /tmp/out.yaml
```

With **helm-diff** plugin:

```bash
helm diff upgrade my-nginx bitnami/nginx -n demo-helm -f nginx-values.yaml
```

---

## 3. Charts, values, and layering

### 3.1 Chart layout

```text
mychart/
  Chart.yaml          # name, version, appVersion, dependencies
  values.yaml         # defaults
  values.schema.json  # optional JSON schema validation
  templates/          # K8s manifests with Go templating
    deployment.yaml
    service.yaml
    _helpers.tpl      # named templates
  charts/             # vendored dependency charts
  crds/               # CRDs (special install rules)
  README.md
```

### 3.2 Values precedence (low → high)

1. Chart `values.yaml`  
2. Parent chart values / dependency values  
3. `-f values.yaml` (later files override earlier)  
4. `--set` / `--set-string` / `--set-file`  

```bash
helm upgrade --install app ./mychart -n app \
  -f values/base.yaml \
  -f values/prod.yaml \
  --set image.tag=1.2.3
```

**Production tip:** prefer files over long `--set` chains; use `--set` only for CI-injected tags/digests.

### 3.3 Common value patterns

```yaml
# values/prod.yaml
replicaCount: 3
image:
  repository: ghcr.io/acme/api
  tag: "1.2.3"
  pullPolicy: IfNotPresent
imagePullSecrets:
  - name: ghcr-creds
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: "1", memory: 512Mi }
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.example.com
      paths: [{ path: /, pathType: Prefix }]
  tls:
    - secretName: api-tls
      hosts: [api.example.com]
```

---

## 4. Create your own chart (hero path)

### 4.1 Scaffold

```bash
helm create myapp
cd myapp
# Edit Chart.yaml, values.yaml, templates/*
```

`Chart.yaml` essentials:

```yaml
apiVersion: v2
name: myapp
description: My application
type: application
version: 0.1.0      # chart version (SemVer) — bump when chart packaging changes
appVersion: "1.0.0" # application version (informational)
```

### 4.2 Template basics

```yaml
# templates/deployment.yaml (excerpt)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

Useful commands while developing:

```bash
helm lint .
helm template myapp . -f values.yaml | less
helm template myapp . --debug
helm upgrade --install myapp . -n demo-helm -f values.yaml --dry-run
```

### 4.3 Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

```bash
helm dependency update   # downloads into charts/
helm dependency build
```

### 4.4 Hooks (use sparingly)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-migrate"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Prefer Operators or external CI migrate jobs for complex lifecycle; hooks are easy to get wrong on failed upgrades.

### 4.5 Package & share

```bash
helm package myapp
helm registry login ghcr.io
helm push myapp-0.1.0.tgz oci://ghcr.io/acme/charts
# Install from OCI:
helm upgrade --install myapp oci://ghcr.io/acme/charts/myapp --version 0.1.0 -n app
```

---

## 5. Day-2 operations cheat sheet

```bash
helm list -A
helm list -n payments-prod
helm status RELEASE -n NS
helm history RELEASE -n NS
helm get values RELEASE -n NS --all
helm get manifest RELEASE -n NS
helm get notes RELEASE -n NS

helm upgrade --install RELEASE CHART -n NS -f values.yaml --wait --timeout 10m --atomic
helm rollback RELEASE REVISION -n NS --wait
helm uninstall RELEASE -n NS --keep-history   # optional audit trail

helm repo update
helm search repo KEYWORD --versions
helm show values CHART --version X.Y.Z
```

**Flags that matter in prod**

| Flag | Purpose |
|------|---------|
| `--wait` | Wait for resources ready |
| `--timeout` | Fail if not ready in time |
| `--atomic` | Rollback automatically on failed upgrade |
| `--cleanup-on-fail` | Clean new resources on failure |
| `--create-namespace` | Create NS if missing (use carefully) |
| `--version` | Pin chart version |
| `--dry-run=server` | Validate against API (Helm 3.13+) when supported |

---

## 6. Production best practices

### 6.1 Version pinning & supply chain

- [ ] **Pin chart versions** (`--version 4.11.1`), never float `latest` in prod  
- [ ] **Pin image tags/digests** in values; prefer digests for critical apps  
- [ ] Use **OCI registries** or an internal chart museum / Harbor  
- [ ] **helm pull** + review manifests in CI before apply  
- [ ] Sign/verify if your org requires (cosign, chart provenance)  
- [ ] Run `helm dependency update` in CI from locked `Chart.lock`  

```bash
helm pull ingress-nginx/ingress-nginx --version 4.11.1 --untar
# review, then install that exact version
```

### 6.2 Values & secrets

- [ ] Split values: `values/base.yaml`, `values/staging.yaml`, `values/prod.yaml`  
- [ ] **Never commit raw secrets** to Git  
- [ ] Use SOPS, Sealed Secrets, External Secrets, or Vault — inject at deploy time  
- [ ] Keep a **non-secret** values file + secret overlays in CI  

```bash
# Example pattern: decrypt to temp file in CI
sops -d values/prod.secrets.yaml > /tmp/prod.secrets.yaml
helm upgrade --install api ./charts/api -n payments-prod \
  -f values/base.yaml -f values/prod.yaml -f /tmp/prod.secrets.yaml \
  --atomic --wait
rm -f /tmp/prod.secrets.yaml
```

### 6.3 Namespaces & RBAC

- [ ] One release (or clear set) per namespace/environment  
- [ ] CI deploy SA has **least privilege** (not cluster-admin)  
- [ ] Disable `helm` creating namespaces in prod if NS are GitOps-owned  

### 6.4 Safe upgrades

- [ ] Always `helm diff upgrade` or `helm template` + `kubectl diff` in CI  
- [ ] Use `--atomic --wait` for prod upgrades  
- [ ] Read chart **upgrade notes** / breaking changes  
- [ ] CRDs: Helm does **not** upgrade CRDs by default the same way as other resources — plan CRD upgrades explicitly  
- [ ] Test in staging with **same chart version + comparable values**  

### 6.5 Resource & runtime hygiene

- [ ] Set **requests/limits** in values  
- [ ] PodDisruptionBudgets, probes, securityContext (non-root, readOnlyRootFilesystem where possible)  
- [ ] `imagePullPolicy: IfNotPresent` + pinned tags (Avoid `:latest`)  
- [ ] Topology spread / anti-affinity for HA services  

### 6.6 Release naming & ownership

- [ ] Stable release names (`ingress-nginx`, `cert-manager`)  
- [ ] Label releases: `app.kubernetes.io/managed-by=Helm` (automatic) + team labels via values  
- [ ] Document **who owns** each release in a catalog / Helmfile comments  

### 6.7 Drift & GitOps

- [ ] Prefer **Git as source of truth** (values + chart version in repo)  
- [ ] Avoid manual `helm upgrade` on prod without PR  
- [ ] If using Argo CD / Flux: either Helmfile→PR→GitOps or native Helm of those tools — pick one control plane  
- [ ] Periodic `helm list -A` vs inventory audit  

### 6.8 Hooks & Jobs

- [ ] Avoid long, fragile pre-upgrade hooks for DB migrate when an Operator/CI job is clearer  
- [ ] Set hook delete policies; don’t leave failed Jobs blocking upgrades  

### 6.9 Multi-env layout (recommended repo)

```text
deploy/
  charts/api/                 # or reference OCI chart version
  helmfile.yaml               # see §9
  environments/
    staging/
      values.yaml
    prod/
      values.yaml
      values-secrets.sops.yaml
  README.md
```

### 6.10 Observability of deploys

- [ ] CI logs: chart version, app version, values digest, revision  
- [ ] Alert on failed releases / CrashLoop after deploy  
- [ ] `helm history` retained (`--history-max` on install/upgrade if needed)  

```bash
helm upgrade --install api oci://ghcr.io/acme/charts/api \
  --version 1.4.2 -n payments-prod \
  -f values/prod.yaml \
  --history-max 20 \
  --atomic --wait --timeout 15m
```

---

## 7. Three-way merge & common failures

Helm 3 uses **three-way strategic merge** (old live, old manifest, new manifest).

| Symptom | Likely cause |
|---------|----------------|
| Upgrade blocked by immutable field | Change needs delete/recreate (e.g. some selectors) |
| Hooks fail → stuck | Delete failed hook Job/Pod; fix hook; re-run |
| CRD not updated | Manual CRD apply / chart docs |
| “another operation in progress” | Previous secret locked; check pending release secrets |
| Values ignored | Wrong `-f` order; nested key typo; dependency `condition: false` |

```bash
kubectl get secrets -n NS -l owner=helm
# Pending-upgrade secrets may need careful cleanup — prefer helm rollback/unlock docs for your version
helm status RELEASE -n NS
```

---

## 8. CI pipeline sketch

```bash
set -euo pipefail
helm repo update
helm dependency update ./charts/api   # if applicable

helm lint ./charts/api
helm template api ./charts/api -f values/prod.yaml > /tmp/render.yaml
kubeconform -summary /tmp/render.yaml   # or kubectl apply --dry-run=server

helm diff upgrade api ./charts/api -n payments-prod -f values/prod.yaml || true

helm upgrade --install api ./charts/api \
  -n payments-prod \
  -f values/base.yaml \
  -f values/prod.yaml \
  --version 1.4.2 \
  --atomic --wait --timeout 15m
```

---

## 9. Helmfile — manage many releases

[Helmfile](https://github.com/helmfile/helmfile) is a declarative wrapper: **one YAML** describes many Helm releases, repos, and environments. Ideal when you have ingress + cert-manager + apps + monitoring.

### 9.1 Install Helmfile

```bash
# Pin version from GitHub releases — example pattern
VER=0.169.0
OS=linux
ARCH=amd64
curl -fsSL -o /tmp/helmfile.tar.gz \
  "https://github.com/helmfile/helmfile/releases/download/v${VER}/helmfile_${VER}_${OS}_${ARCH}.tar.gz"
sudo tar xz -C /usr/local/bin -f /tmp/helmfile.tar.gz helmfile
helmfile --version

# Needs helm + kubectl; helm-diff strongly recommended
helm plugin install https://github.com/databus23/helm-diff
```

### 9.2 Minimal `helmfile.yaml`

```yaml
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: jetstack
    url: https://charts.jetstack.io

releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    createNamespace: true
    chart: ingress-nginx/ingress-nginx
    version: 4.11.1
    values:
      - values/ingress-nginx.yaml

  - name: cert-manager
    namespace: cert-manager
    createNamespace: true
    chart: jetstack/cert-manager
    version: v1.15.3
    values:
      - values/cert-manager.yaml
    set:
      - name: crds.enabled
        value: true
```

### 9.3 Environments (staging / prod)

```yaml
environments:
  staging:
    values:
      - environments/staging/values.yaml
  prod:
    values:
      - environments/prod/values.yaml

---
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: api
    namespace: payments-{{ .Environment.Name }}
    createNamespace: true
    chart: oci://ghcr.io/acme/charts/api
    version: {{ .Values.api.chartVersion }}
    values:
      - values/api/base.yaml
      - values/api/{{ .Environment.Name }}.yaml
    secrets:
      - environments/{{ .Environment.Name }}/api.secrets.yaml   # with helm-secrets / sops integration
```

`environments/*/values.yaml` example:

```yaml
# environments/prod/values.yaml
api:
  chartVersion: "1.4.2"
```

### 9.4 Essential Helmfile commands

```bash
# See rendered releases
helmfile -e prod list
helmfile -e prod deps          # fetch charts
helmfile -e prod template      # render all
helmfile -e prod diff          # needs helm-diff — show drift/changes
helmfile -e prod lint

# Apply (idempotent)
helmfile -e prod apply         # diff then upgrade --install
# or
helmfile -e prod sync          # sync to desired state without interactive diff flow

# One release only
helmfile -e prod -l name=api apply
helmfile -e prod -l name=ingress-nginx diff

# Destroy (careful)
helmfile -e staging -l name=api destroy
```

### 9.5 `helmfile.d/` and layers

```text
helmfile.yaml          # includes
helmfile.d/
  00-repos.yaml
  10-ingress.yaml
  20-cert-manager.yaml
  30-apps.yaml
```

```yaml
# helmfile.yaml
helmfiles:
  - path: helmfile.d/*.yaml
```

Or:

```yaml
bases:
  - bases/repos.yaml
  - bases/environments.yaml
```

### 9.6 Ordering & dependencies between releases

```yaml
releases:
  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: v1.15.3
    needs: []          # no deps

  - name: cluster-issuer
    namespace: cert-manager
    chart: ./charts/cluster-issuer   # local chart of Issuer CRs
    needs:
      - cert-manager/cert-manager    # namespace/name
```

### 9.7 Production practices with Helmfile

| Practice | How |
|----------|-----|
| Pin everything | Chart `version:` on every release |
| Env separation | `helmfile -e staging\|prod` |
| Secrets | `secrets:` + [helm-secrets](https://github.com/jkroepke/helm-secrets) / SOPS |
| Review before apply | `helmfile diff` in PR CI; `apply` on merge |
| Selectivity | `--selector name=api` / labels on releases |
| Lockfile | Commit Helmfile lock if you use `deps` locking workflow |
| Don’t mix ad-hoc helm | Once Helmfile owns a release, change it only via Helmfile/Git |
| CRDs | Follow chart docs; may need one-time raw apply |
| Timeouts | `wait: true`, `timeout: 600` per release |

```yaml
releases:
  - name: api
    namespace: payments-prod
    chart: oci://ghcr.io/acme/charts/api
    version: 1.4.2
    wait: true
    atomic: true
    timeout: 600
    values:
      - values/api/prod.yaml
    labels:
      tier: app
      team: payments
```

```bash
helmfile -e prod -l tier=app diff
helmfile -e prod -l tier=app apply
```

### 9.8 Helmfile + GitOps

Common patterns:

1. **CI applies Helmfile** to the cluster (simple; needs strong RBAC + branch protection).  
2. **CI runs `helmfile template`** and commits rendered YAML for Argo/Flux (plain manifests).  
3. **Argo CD** manages Helm/Helmfile via plugins — keep one source of truth.

Avoid: Helmfile apply **and** Argo syncing the same release independently.

### 9.9 Quick Helmfile flow

```text
Write helmfile.yaml (repos + releases + versions + values)
  → helmfile -e staging deps
  → helmfile -e staging diff
  → helmfile -e staging apply
  → promote same chart versions to prod values
  → helmfile -e prod diff && apply
```

---

## 10. Helm vs Operator vs Helmfile

| Tool | Use when |
|------|----------|
| **Helm** | Package & configure apps; templated installs |
| **Helmfile** | Many Helm releases / envs as one declarative ops repo |
| **Operator** | Continuous reconcile + domain runbooks (see [crd-operators.md](./crd-operators.md)) |

```text
Helmfile  →  drives Helm  →  creates Deployments/… 
Operator  →  watches CRs   →  creates Deployments/… continuously
```

---

## 11. From zero to hero — learning path

```text
1. Install helm; repo add; install bitnami/nginx with -f values
2. upgrade / history / rollback / uninstall
3. helm template & diff before every change
4. helm create; lint; package; push OCI
5. Pin versions; split values; secrets via SOPS
6. CI: lint + template + diff + atomic upgrade
7. Install helmfile; describe ingress+apps; diff/apply per env
8. Lock chart versions; selectors; needs:; prod checklist §6
```

---

## 12. Cleanup lab

```bash
helm uninstall my-nginx -n demo-helm || true
helmfile -e staging destroy || true
kubectl delete ns demo-helm ingress-nginx cert-manager 2>/dev/null || true
```

---

## References

- [Helm documentation](https://helm.sh/docs/)
- [Helm best practices](https://helm.sh/docs/chart_best_practices/)
- [Chart hooks](https://helm.sh/docs/topics/charts_hooks/)
- [OCI registries](https://helm.sh/docs/topics/registries/)
- [Helmfile docs](https://helmfile.readthedocs.io/)
- [Helmfile GitHub](https://github.com/helmfile/helmfile)
- [helm-diff](https://github.com/databus23/helm-diff)
- [helm-secrets](https://github.com/jkroepke/helm-secrets)
