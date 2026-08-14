# Kustomize from Zero to Hero

Declarative Kubernetes config without a templating language: **bases, overlays, patches, generators**, and production patterns (including GitOps with Argo CD).

Related:

- [helm-zero-to-hero.md](./helm-zero-to-hero.md)
- [argocd-production.md](./argocd-production.md)
- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

Official: [Kustomize](https://kustomize.io/) · [kubectl kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) · [kustomize docs](https://kubectl.docs.kubernetes.io/references/kustomize/)

---

## 0. What Kustomize is

Kustomize builds Kubernetes YAML by **composing and patching** plain manifests. No `{{ }}` templates required.

| Concept | Meaning |
|---------|---------|
| **kustomization.yaml** | Instructions: which resources, patches, images, namespaces, … |
| **Base** | Shared, env-agnostic manifests |
| **Overlay** | Env-specific changes (staging/prod) on top of a base |
| **Patch** | Strategic merge or JSON6902 edit of a resource |
| **Generator** | Create ConfigMaps/Secrets from files/literals |
| **Component** | Reusable snippet included by many overlays |
| **Build** | `kustomize build` / `kubectl kustomize` → final YAML |

```text
base/ + overlays/prod/  →  kustomize build  →  kubectl apply / Argo CD sync
```

**Built into kubectl** (`kubectl apply -k`, `kubectl kustomize`). Standalone `kustomize` CLI is useful for newer features before they land in your kubectl version.

---

## 1. Install & versions

```bash
# Usually enough (kubectl embeds kustomize)
kubectl version --client
kubectl kustomize --help

# Standalone (pin a release for CI)
# https://github.com/kubernetes-sigs/kustomize/releases
VER=v5.5.0
curl -fsSL -o /tmp/kustomize.tgz \
  "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2F${VER}/kustomize_${VER}_linux_amd64.tar.gz"
sudo tar xz -C /usr/local/bin -f /tmp/kustomize.tgz kustomize
kustomize version
```

```bash
# Prefer same major in CI and laptops
kustomize version
kubectl version --client -o yaml | grep -i kustomize || true
```

---

## 2. Zero → first build (30 minutes)

### 2.1 Tiny app layout

```bash
mkdir -p myapp/{base,overlays/staging,overlays/prod}
```

**Base resources**

```bash
cat > myapp/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 1
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
          image: ghcr.io/acme/api:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF

cat > myapp/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
EOF

cat > myapp/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app.kubernetes.io/name: api
EOF
```

**Build the base**

```bash
kubectl kustomize myapp/base
# or: kustomize build myapp/base
```

### 2.2 Staging overlay

```bash
cat > myapp/overlays/staging/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-staging
resources:
  - ../../base
namePrefix: staging-
images:
  - name: ghcr.io/acme/api
    newTag: "1.1.0-rc.1"
replicas:
  - name: api
    count: 1
EOF

kubectl kustomize myapp/overlays/staging
kubectl apply -k myapp/overlays/staging
# kubectl delete -k myapp/overlays/staging
```

### 2.3 Prod overlay (patch + more replicas)

```bash
cat > myapp/overlays/prod/replicas.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
EOF

cat > myapp/overlays/prod/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-prod
resources:
  - ../../base
images:
  - name: ghcr.io/acme/api
    newName: ghcr.io/acme/api
    newTag: "1.0.0"
replicas:
  - name: api
    count: 3
patches:
  - path: replicas.yaml
    target:
      kind: Deployment
      name: api
commonAnnotations:
  env: production
EOF

kubectl kustomize myapp/overlays/prod | less
kubectl diff -k myapp/overlays/prod || true
kubectl apply -k myapp/overlays/prod
```

> Tip: use either `replicas:` **or** a patch for replica count—don’t fight yourself. `replicas:` transformer is enough for simple cases.

---

## 3. Core building blocks

### 3.1 `resources`

List files, directories, or HTTP(S)/Git URLs (careful in prod—pin refs).

```yaml
resources:
  - deployment.yaml
  - ../shared/namespace.yaml
  - github.com/kubernetes-sigs/kustomize/examples/helloWorld?ref=v5.0.0
```

### 3.2 `namespace`, `namePrefix`, `nameSuffix`

```yaml
namespace: payments-prod
namePrefix: prod-
nameSuffix: -v2
```

Affects resource names **and** references Kustomize knows how to rewrite (labels/selectors need care—prefer stable names + namespace overlays).

### 3.3 `commonLabels` / `labels` / `commonAnnotations`

```yaml
labels:
  - pairs:
      app.kubernetes.io/part-of: payments
    includeSelectors: true   # older commonLabels always affected selectors—be careful
```

Modern `labels` gives control over whether selectors are modified. Mis-labeling selectors can force Deployment recreation.

### 3.4 `images` (most common overlay change)

```yaml
images:
  - name: ghcr.io/acme/api          # must match image name in base (without tag digest quirks)
    newTag: "1.2.3"
    # newName: registry.example.com/acme/api
    # digest: sha256:abcd...
```

Prefer **digest** in production:

```yaml
images:
  - name: ghcr.io/acme/api
    newName: ghcr.io/acme/api
    digest: sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

### 3.5 `replicas`

```yaml
replicas:
  - name: api
    count: 3
```

### 3.6 Patches

**Strategic merge patch** (YAML fragment):

```yaml
# overlays/prod/resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
        - name: api
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: "1"
              memory: 512Mi
```

```yaml
patches:
  - path: resources-patch.yaml
```

**JSON 6902 patch** (precise ops):

```yaml
# overlays/prod/json6902.yaml
- op: replace
  path: /spec/template/spec/containers/0/imagePullPolicy
  value: IfNotPresent
```

```yaml
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: api
    path: json6902.yaml
```

**Inline patch:**

```yaml
patches:
  - target:
      kind: Deployment
      name: api
    patch: |-
      - op: add
        path: /spec/template/metadata/annotations
        value:
          prometheus.io/scrape: "true"
```

### 3.7 ConfigMap / Secret generators

```yaml
configMapGenerator:
  - name: api-config
    literals:
      - LOG_LEVEL=info
      - FEATURE_X=true
    files:
      - config/app.properties
    options:
      disableNameSuffixHash: false   # default: hash suffix → roll pods on change

secretGenerator:
  - name: api-secrets
    envs:
      - secrets/prod.env             # KEY=VALUE file — do not commit plaintext in real prod
    type: Opaque
```

Reference in Deployment:

```yaml
envFrom:
  - configMapRef:
      name: api-config        # Kustomize rewrites to api-config-<hash> unless hash disabled
```

**Production:** generate Secrets via SOPS/KSOPS, Sealed Secrets, or External Secrets—not raw `secretGenerator` from committed plaintext.

### 3.8 `replacements` (wire one field into another)

Useful for injecting a Service name or annotation into another object (Kustomize v4.5.something+ / v5).

```yaml
replacements:
  - source:
      kind: ConfigMap
      name: api-config
      fieldPath: data.LOG_LEVEL
    targets:
      - select:
          kind: Deployment
          name: api
        fieldPaths:
          - spec.template.spec.containers.[name=api].env.[name=LOG_LEVEL].value
```

(Exact paths depend on your manifests—validate with `kustomize build`.)

### 3.9 `components` (reusable bundles)

```text
components/
  observability/
    kustomization.yaml    # kind: Component
    servicemonitor.yaml
overlays/prod/
  kustomization.yaml      # components: [../../components/observability]
```

```yaml
# components/observability/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - servicemonitor.yaml
```

```yaml
# overlays/prod/kustomization.yaml
components:
  - ../../components/observability
```

Use components for cross-cutting concerns: metrics, PDB, NetworkPolicy defaults.

### 3.10 `helmCharts` (optional)

Kustomize can render Helm charts as a generator (feature maturity / flags vary by version):

```yaml
helmCharts:
  - name: redis
    repo: https://charts.bitnami.com/bitnami
    version: 18.6.1
    releaseName: redis
    namespace: payments-prod
    valuesFile: redis-values.yaml
```

May need `kustomize build --enable-helm`. In many orgs: **Helm for third-party**, **Kustomize for first-party apps**—or render Helm in CI and commit output. See [helm-zero-to-hero.md](./helm-zero-to-hero.md).

---

## 4. Recommended directory layouts

### 4.1 Classic base / overlays

```text
payments-api/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    serviceaccount.yaml
  overlays/
    staging/
      kustomization.yaml
      ingress.yaml
    prod/
      kustomization.yaml
      ingress.yaml
      pdb.yaml
      patches/
        resources.yaml
```

### 4.2 Multi-app monorepo (GitOps-friendly)

```text
gitops/
  platform/
    base/
    overlays/prod/
  teams/
    payments/
      api/
        base/
        overlays/{staging,prod}/
      worker/
        base/
        overlays/{staging,prod}/
  components/
    security-defaults/
    observability/
```

Argo CD `path:` points at an **overlay** directory (see [argocd-production.md](./argocd-production.md)).

### 4.3 Remote bases

```yaml
resources:
  - git::https://git.example.com/platform/kustomize-bases.git//web-app?ref=v1.4.0
```

**Pin `ref=` to a tag/commit** in production—never floating `main` without protection.

---

## 5. Day-2 commands cheat sheet

```bash
# Build
kubectl kustomize path/to/overlay
kustomize build path/to/overlay
kustomize build path/to/overlay --enable-helm

# Apply / delete / diff
kubectl apply -k path/to/overlay
kubectl delete -k path/to/overlay
kubectl diff -k path/to/overlay

# CI validation
kustomize build overlays/prod > /tmp/out.yaml
kubeconform -summary /tmp/out.yaml
kubectl apply --dry-run=server -f /tmp/out.yaml

# Show effective images / names
kustomize build overlays/prod | grep -E 'image:|name:'
```

---

## 6. From zero to hero — worked example

Goal: base API + staging/prod overlays with ConfigMap, resources patch, PDB in prod only.

```bash
mkdir -p hero/{base,overlays/staging,overlays/prod/patches,components/pdb}

# --- base ---
cat > hero/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop
  template:
    metadata:
      labels:
        app: shop
    spec:
      containers:
        - name: shop
          image: ghcr.io/acme/shop:0.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: shop-config
EOF

cat > hero/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: shop
spec:
  selector:
    app: shop
  ports:
    - port: 80
      targetPort: 8080
EOF

cat > hero/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: shop-config
    literals:
      - LOG_LEVEL=info
labels:
  - pairs:
      app.kubernetes.io/name: shop
    includeSelectors: false
EOF

# --- component: PDB ---
cat > hero/components/pdb/pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shop
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: shop
EOF

cat > hero/components/pdb/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - pdb.yaml
EOF

# --- staging ---
cat > hero/overlays/staging/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shop-staging
resources:
  - ../../base
images:
  - name: ghcr.io/acme/shop
    newTag: "1.2.0-rc.5"
replicas:
  - name: shop
    count: 1
configMapGenerator:
  - name: shop-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - ENV=staging
EOF

# --- prod ---
cat > hero/overlays/prod/patches/resources.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
spec:
  template:
    spec:
      containers:
        - name: shop
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
EOF

cat > hero/overlays/prod/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shop-prod
resources:
  - ../../base
components:
  - ../../components/pdb
images:
  - name: ghcr.io/acme/shop
    newTag: "1.1.8"
replicas:
  - name: shop
    count: 3
configMapGenerator:
  - name: shop-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - ENV=prod
patches:
  - path: patches/resources.yaml
EOF

kustomize build hero/overlays/staging | head -n 80
kustomize build hero/overlays/prod | head -n 100
```

Promote to prod = PR that bumps `images.newTag` / digest in `overlays/prod` only.

---

## 7. Production best practices

### 7.1 Structure & ownership

- [ ] One **base** per app; overlays per environment (or cluster)  
- [ ] No copy-paste of full Deployments between envs—**patch** differences  
- [ ] Pin remote bases to **tags/SHAs**  
- [ ] Document which overlay Argo CD / CI deploys  

### 7.2 Images & supply chain

- [ ] Prod overlays use **digests** when possible  
- [ ] CI updates only overlay image fields (or use Image Updater → Git)  
- [ ] Never `:latest` in prod  

### 7.3 Secrets

- [ ] Avoid plaintext `secretGenerator` in Git  
- [ ] Sealed Secrets manifests as resources, External Secrets CRs, or SOPS+KSOPS plugin  
- [ ] Separate secret management from kustomization layout  

### 7.4 Safe changes to labels/names

- [ ] Prefer `includeSelectors: false` unless you intend selector changes  
- [ ] Changing Deployment selectors is a **replace** — plan downtime/migration  
- [ ] Stable resource `metadata.name` across envs; differ by **namespace**  

### 7.5 Validation in CI

```bash
kustomize build overlays/prod > /tmp/prod.yaml
kubeconform -ignore-missing-schemas -summary /tmp/prod.yaml
# optional: kustomize build | kyverno apply / policy checks
kubectl apply --dry-run=server -f /tmp/prod.yaml
```

- [ ] Fail PR if `kustomize build` breaks  
- [ ] `kubectl diff -k` against the cluster (read-only SA) on prod PRs  

### 7.6 GitOps

- [ ] Argo CD `path: teams/payments/api/overlays/prod`  
- [ ] Auto-sync staging; gated prod  
- [ ] Don’t also `kubectl apply -k` manually on the same objects  
- [ ] Sync-wave annotations allowed on raw resources in base  

### 7.7 Compatibility

- [ ] Pin `kustomize` version in CI  
- [ ] Avoid bleeding-edge fields your cluster kubectl can’t build if builders differ  
- [ ] Commit `kustomization.yaml` `apiVersion` consistently (`v1beta1` / Component `v1alpha1`)  

### 7.8 Keep overlays thin

| Put in base | Put in overlay |
|-------------|----------------|
| Deployment shape, ports, probes defaults | replica counts |
| Service | Ingress host/TLS |
| SA, NetworkPolicy skeleton | image tag/digest |
| | resources limits, env-specific ConfigMap |

### 7.9 Helm vs Kustomize vs both

| Use case | Prefer |
|----------|--------|
| First-party apps you own | **Kustomize** |
| Third-party complex charts | **Helm** (or Helm→render→Kustomize) |
| Many releases / envs as one ops entry | Helmfile or Argo ApplicationSet |
| Need continuous domain logic | Operator ([crd-operators.md](./crd-operators.md)) |

Pattern **Helm + Kustomize:**

```text
helm template ... > charts/render/   # or use helmCharts generator
kustomize overlays patch image/NS/labels
```

---

## 8. Common pitfalls

| Pitfall | Fix |
|---------|-----|
| ConfigMap name hash → Deployment not updated | Reference without hardcoding hash; let Kustomize rewrite; or `options.disableNameSuffixHash` + explicit rollout |
| Patch doesn’t apply | Wrong `name`/`kind`/`namespace` in target; build and inspect |
| `commonLabels` broke upgrade | Use modern `labels` with `includeSelectors: false` |
| Overlay too fat | Move shared bits back to base/components |
| Secret in Git | ESO / SOPS / Sealed Secrets |
| Floating remote `ref=main` | Pin version |
| Two tools manage same app | One source of truth |

```bash
# Debug: see if name hash is the issue
kustomize build overlays/prod | grep -A2 'kind: ConfigMap'
kustomize build overlays/prod | grep -A5 'configMapRef'
```

---

## 9. Kustomize + Argo CD (minimal)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-prod
  namespace: argocd
spec:
  project: payments
  source:
    repoURL: https://git.example.com/gitops.git
    targetRevision: main
    path: teams/shop/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: shop-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Argo CD runs `kustomize build` on that path (version configurable in Argo CD).

---

## 10. Learning path (zero → hero)

```text
1. kubectl kustomize on a folder with resources:
2. Add namespace + images + replicas in an overlay
3. Strategic merge patch for resources/probes
4. configMapGenerator + envFrom (understand name hash)
5. components/ for PDB / observability
6. JSON6902 for surgical edits
7. Pin digests; CI build + kubeconform + diff
8. Wire overlay path to Argo CD
9. Remote bases with pinned refs; replacements if needed
10. Decide Helm vs Kustomize boundaries for third-party
```

---

## 11. Cleanup

```bash
kubectl delete -k myapp/overlays/staging --ignore-not-found
kubectl delete -k myapp/overlays/prod --ignore-not-found
kubectl delete -k hero/overlays/staging --ignore-not-found
kubectl delete -k hero/overlays/prod --ignore-not-found
rm -rf myapp hero
```

---

## References

- [Kustomize](https://kustomize.io/)
- [kubectl kustomize / apply -k](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomize references](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Kustomize GitHub](https://github.com/kubernetes-sigs/kustomize)
- [Good practices (SIG docs)](https://kubectl.docs.kubernetes.io/guides/config_management/)
- [Argo CD — Kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/)
