# Kubernetes Secrets — Types, Examples & Production Practice

Clear catalog of **built-in Secret types**, how to create and use each, plus **best practices** and hard-won **production lessons**.

Related: [developer-access-rbac.md](./developer-access-rbac.md) · [oidc-sso-setup.md](./oidc-sso-setup.md) · [argocd-production.md](./argocd-production.md) · [velero-backup.md](../velero-backup.md) · [certificate-renewal.md](./certificate-renewal.md)

Official: [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Good practices for Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#good-practices)

---

## 0. What a Secret is (and is not)

A **Secret** stores sensitive small blobs (passwords, tokens, keys, certs) as API objects—usually base64 in `data`, or plain in `stringData` (API encodes for you).

| Myth | Reality |
|------|---------|
| “Secrets are encrypted” | By default they are **base64-encoded**, not encrypted. Enable **encryption at rest** for etcd. |
| “If it’s a Secret, it’s safe in Git” | **Never** commit raw Secrets to Git. |
| “Only Opaque matters” | Typed Secrets document intent and unlock helpers (`kubectl create secret tls`, image pulls, …). |
| “Env vars are fine everywhere” | Env vars are easy to leak via logs/`kubectl describe`/crash dumps—prefer **files** + restricted RBAC. |

```bash
# Decode for debugging (careful on shared screens)
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## 1. Common fields

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example
  namespace: demo
type: Opaque                    # see types below
immutable: true                 # optional: lock data after create
stringData:                     # write plain text (convenience)
  password: "s3cr3t"
data:                           # or already base64
  # password: czNjcjN0
```

| Field | Use |
|-------|-----|
| `type` | Documents purpose; some controllers/kubelet treat types specially |
| `data` | map[string][]byte as **base64** |
| `stringData` | Write plaintext; merged/encoded into `data` on write; **not** returned on read |
| `immutable: true` | Blocks updates to `data`/`stringData` (must delete/recreate)—reduces accidental mutation |

---

## 2. All built-in Secret types (with examples)

### 2.1 `Opaque` (generic / default)

**Use for:** arbitrary key-value secrets (DB passwords, API keys, app config).

```bash
kubectl -n demo create secret generic db-creds \
  --from-literal=username=app \
  --from-literal=password='hunter2-change-me'

# From files
kubectl -n demo create secret generic app-keys \
  --from-file=api-key=./api.key \
  --from-file=./config/secret.env
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  namespace: demo
type: Opaque
stringData:
  username: app
  password: "hunter2-change-me"
  DATABASE_URL: "postgres://app:hunter2-change-me@db:5432/app"
```

**Consume as env:**

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password
envFrom:
  - secretRef:
      name: db-creds
```

**Consume as files (preferred):**

```yaml
volumes:
  - name: db
    secret:
      secretName: db-creds
      defaultMode: 0400
containers:
  - name: app
    volumeMounts:
      - name: db
        mountPath: /var/run/secrets/db
        readOnly: true
# files: /var/run/secrets/db/username, password, …
```

---

### 2.2 `kubernetes.io/service-account-token`

**Use for:** ServiceAccount identity tokens for the API (legacy auto-mounted Secrets; modern clusters prefer **bound TokenRequest** projections).

Legacy (still seen):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  namespace: demo
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token
# Controller fills data: token, ca.crt, namespace
```

**Preferred (1.24+):** don’t create long-lived SA Secrets; use projected volume:

```yaml
volumes:
  - name: api-token
    projected:
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600
            audience: https://kubernetes.default.svc
```

```bash
# Short-lived token for CI debugging
kubectl -n demo create token my-sa --duration=1h
```

**Production:** disable automount where unused (`automountServiceAccountToken: false`); never mount default SA into untrusted pods.

---

### 2.3 `kubernetes.io/dockerconfigjson` (and legacy `dockercfg`)

**Use for:** private registry image pulls (`imagePullSecrets`).

```bash
kubectl -n demo create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=USER \
  --docker-password=TOKEN \
  --docker-email=you@example.com
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: demo
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "ghcr.io": {
          "username": "USER",
          "password": "TOKEN",
          "email": "you@example.com",
          "auth": "BASE64(USER:TOKEN)"
        }
      }
    }
```

Legacy type `kubernetes.io/dockercfg` uses key `.dockercfg` (older Docker format)—prefer **`dockerconfigjson`**.

**Attach to ServiceAccount (recommended) or Pod:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: demo
imagePullSecrets:
  - name: regcred
---
# Pod/Deployment:
spec:
  serviceAccountName: app
  # or:
  # imagePullSecrets:
  #   - name: regcred
```

Full end-to-end flow: **[§9 Deploy from a private registry](#9-deploy-an-app-from-a-private-docker-registry)**.

---

### 2.4 `kubernetes.io/basic-auth`

**Use for:** username/password pairs (documented type; still Opaque-like storage).

```bash
kubectl -n demo create secret generic basic-auth \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=admin \
  --from-literal=password='change-me'
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
  namespace: demo
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: "change-me"
```

Keys **must** be `username` and `password`. Useful with Ingress basic-auth annotations (controller-specific) or app libraries that expect this convention.

---

### 2.5 `kubernetes.io/ssh-auth`

**Use for:** SSH private keys (e.g. Git clone, remote host).

```bash
kubectl -n demo create secret generic git-ssh \
  --type=kubernetes.io/ssh-auth \
  --from-file=ssh-privatekey=$HOME/.ssh/id_ecdsa
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh
  namespace: demo
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

Key name **must** be `ssh-privatekey`. Mount as a file with mode `0400`; pair with known_hosts ConfigMap/Secret.

---

### 2.6 `kubernetes.io/tls`

**Use for:** TLS cert + key (Ingress, webhook servers, mTLS).

```bash
kubectl -n demo create secret tls demo-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-tls
  namespace: demo
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    ...
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    ...
```

Keys **must** be `tls.crt` and `tls.key`. Optional `ca.crt` is sometimes added for chains (not all controllers require it inside the Secret).

**Ingress example:**

```yaml
spec:
  tls:
    - hosts: [demo.example.com]
      secretName: demo-tls
```

In production, prefer **cert-manager** to create/rotate `kubernetes.io/tls` Secrets ([ingress-zero-to-hero.md](./ingress-zero-to-hero.md)).

---

### 2.7 `bootstrap.kubernetes.io/token`

**Use for:** kubeadm **bootstrap tokens** (node join)—not general app secrets.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcde1
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "Development token"
  token-id: abcde1
  token-secret: 0123456789abcdef
  expiration: "2026-12-01T00:00:00Z"
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: system:bootstrappers:kubeadm:default-node-token
```

```bash
kubeadm token create --print-join-command
kubeadm token list
```

**Production:** short TTL; don’t leave bootstrap tokens forever; treat like cluster join credentials.

---

## 3. Type quick reference

| Type | Typical keys | Created by / for |
|------|--------------|------------------|
| `Opaque` | any | Generic app secrets |
| `kubernetes.io/service-account-token` | `token`, `ca.crt`, `namespace` | Legacy SA tokens |
| `kubernetes.io/dockerconfigjson` | `.dockerconfigjson` | Image pull |
| `kubernetes.io/dockercfg` | `.dockercfg` | Legacy image pull |
| `kubernetes.io/basic-auth` | `username`, `password` | Basic auth |
| `kubernetes.io/ssh-auth` | `ssh-privatekey` | SSH |
| `kubernetes.io/tls` | `tls.crt`, `tls.key` | TLS |
| `bootstrap.kubernetes.io/token` | `token-id`, `token-secret`, … | kubeadm join |

Custom `type` strings are allowed (e.g. `example.com/my-type`) for your own conventions—controllers must know to honor them.

---

## 4. Best practices

### 4.1 Security

- [ ] Enable **EncryptionConfiguration** for Secrets at rest in etcd  
- [ ] RBAC: separate who can `get`/`list` Secrets vs deploy workloads  
- [ ] Prefer **file mounts** over env vars for high-value material  
- [ ] `defaultMode: 0400` or `0440` with correct fsGroup  
- [ ] `automountServiceAccountToken: false` unless needed  
- [ ] Short-lived tokens (projected SA tokens, OIDC) over static Secrets when possible  
- [ ] Rotate on schedule and on personnel change  
- [ ] Don’t log Secret values; redact CI output  

### 4.2 GitOps & delivery

- [ ] **Never** commit plaintext Secret YAML  
- [ ] Use one of: **Sealed Secrets**, **SOPS** (+ KSOPS/helm-secrets), **External Secrets Operator** (Vault/AWS SM/GCP SM/Azure KV), Vault Agent/CSI  
- [ ] App Git repos hold **references** (`ExternalSecret`, `SealedSecret`) not raw values  
- [ ] Different secrets per env (staging ≠ prod passwords)  

External Secrets sketch:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
  namespace: demo
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-creds
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/demo/db
        property: password
```

### 4.3 Operational

- [ ] `immutable: true` for tokens that should not change in place  
- [ ] Name Secrets clearly: `payments-db-app`, `ingress-demo-tls`  
- [ ] One concern per Secret (don’t mega-blob unrelated keys)—eases rotation  
- [ ] Document ownership and rotation owner in annotations  

```yaml
metadata:
  annotations:
    app.kubernetes.io/owner: team-payments
    secrets.example.com/rotation: "90d"
```

### 4.4 Workload wiring

- [ ] Use `secretKeyRef` optional flags carefully (`optional: true` only when truly optional)  
- [ ] For TLS: let cert-manager own the Secret; don’t hand-edit  
- [ ] For pulls: attach `imagePullSecrets` on **ServiceAccount**, not every Pod template copy  

### 4.5 Backup / DR

- [ ] Velero restores Secrets—ensure bucket encryption and access control ([velero-backup.md](../velero-backup.md))  
- [ ] Prefer re-sync from Vault/ESO after DR rather than trusting old etcd-only copies forever  
- [ ] Bootstrap tokens and join credentials: regenerate, don’t restore blindly  

---

## 5. Production experiences (what bites teams)

### 5.1 “We put Secrets in Git (base64)”

Base64 is not encryption. Leaks via public repos, forks, and CI artifacts are common. **Move to ESO/SOPS/Sealed Secrets** before the audit does it for you.

### 5.2 Env vars in crash reports and `kubectl describe`

Many frameworks dump env on panic. Mount secrets as files and read at runtime. Disable debug endpoints that echo config.

### 5.3 RBAC `list` on Secrets

`get` on a named Secret is narrower than `list` + `get` in a namespace (list exposes names; combined with weak controls aids theft). Tighten Role rules; avoid `edit` ClusterRole for humans in prod ([developer-access-rbac.md](./developer-access-rbac.md)).

### 5.4 Forgotten default ServiceAccount tokens

Pods silently got API power. Set:

```yaml
automountServiceAccountToken: false
```

and use a dedicated SA with minimal Role per app.

### 5.5 Image pull Secret sprawl

Copying `regcred` into every namespace by hand drifts. Use:

- Kyverno/policy to copy or generate  
- ESO templating per namespace  
- Or workload identity / node IAM (EKS/GKE/AKS) to **avoid** pull Secrets where possible  

### 5.6 TLS Secret vs cert-manager races

Manual `kubectl create secret tls` fights cert-manager renewals. Pick one owner. Monitor certificate expiry separately from Opaque app secrets.

### 5.7 Rotation without restart

Apps that read env only at process start **won’t** see Secret updates. Use:

- File mounts + app reload, or  
- Reloader / stamped annotations to roll Deployments on Secret change, or  
- External Secrets refresh + restart policy  

### 5.8 etcd theft = secret theft

If disk/etcd backups leak, Secrets leak. Encryption at rest + encrypted backup buckets + IAM are mandatory for production.

### 5.9 Multi-cluster DR

Restored Secrets may reference wrong KMS keys or old registry tokens. After Velero restore, **reconcile from the secret manager**, then delete stale copies.

### 5.10 Bootstrap token left forever

A join token in `kube-system` with no expiry is a cluster-add backdoor. List and expire:

```bash
kubeadm token list
kubectl -n kube-system get secrets | grep bootstrap-token
```

---

## 6. Commands cheat sheet

```bash
# Create
kubectl create secret generic NAME --from-literal=k=v -n NS
kubectl create secret tls NAME --cert=c.crt --key=c.key -n NS
kubectl create secret docker-registry NAME --docker-server=... -n NS
kubectl create secret generic NAME --type=kubernetes.io/basic-auth \
  --from-literal=username=u --from-literal=password=p -n NS

# Inspect (careful)
kubectl get secrets -n NS
kubectl describe secret NAME -n NS          # values hidden
kubectl get secret NAME -n NS -o yaml       # data base64 visible if allowed

# Consume test
kubectl -n NS run curl --rm -it --restart=Never \
  --image=curlimages/curl --command -- sleep 1   # prefer a debug Pod with volumeMount

# Delete / rotate
kubectl delete secret NAME -n NS
# recreate + rollout
kubectl -n NS rollout restart deploy/APP
```

---

## 7. Decision guide

```text
App password / API key     → Opaque (+ ESO/SOPS in GitOps)
Registry pull              → dockerconfigjson on SA (or cloud node identity)
Ingress / webhook cert     → tls (cert-manager)
SSH deploy key             → ssh-auth
Ingress basic auth file    → Opaque htpasswd or basic-auth (controller docs)
Pod → API auth             → projected serviceAccountToken (not long-lived SA Secret)
Node join                  → bootstrap token (short TTL)
```

---

## 8. Quick flow (production)

```text
Enable Secret encryption at rest
  → Choose secret manager (Vault / cloud SM)
  → External Secrets / SOPS into cluster
  → Typed Secrets only where needed (tls, dockerconfigjson)
  → Mount as files; tight RBAC; no Git plaintext
  → Rotate + rollout; audit list/get access
  → DR: restore from manager, not only etcd
```

---

## 9. Deploy an app from a private Docker registry

Use a Secret of type **`kubernetes.io/dockerconfigjson`** so kubelet can authenticate to the registry when pulling the image. Without it, pods stay `ErrImagePull` / `ImagePullBackOff`.

```text
kubectl create secret docker-registry
  → reference as imagePullSecrets (Pod) or on ServiceAccount
  → Deployment image: registry.example.com/team/app:1.2.3
  → kubelet pulls with those credentials
```

### 9.1 Create the pull Secret

**Interactive / CI-friendly (recommended):**

```bash
export NS=demo
export REGISTRY=ghcr.io                 # or docker.io, registry.gitlab.com, 123.dkr.ecr….amazonaws.com, …
export REG_USER=my-bot-user
export REG_PASS=ghp_xxxxxxxx            # PAT / robot token / password (not your login password if 2FA)

kubectl create namespace ${NS} --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ${NS} create secret docker-registry regcred \
  --docker-server="${REGISTRY}" \
  --docker-username="${REG_USER}" \
  --docker-password="${REG_PASS}" \
  --docker-email="ci@example.com" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ${NS} get secret regcred -o yaml
# type should be kubernetes.io/dockerconfigjson
```

**From an existing local Docker login:**

```bash
# After: docker login ghcr.io
kubectl -n ${NS} create secret generic regcred \
  --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  --dry-run=client -o yaml | kubectl apply -f -
```

**YAML (prefer External Secrets / SOPS in Git — do not commit real passwords):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: demo
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "ghcr.io": {
          "username": "my-bot-user",
          "password": "ghp_xxxxxxxx",
          "email": "ci@example.com",
          "auth": "BASE64(my-bot-user:ghp_xxxxxxxx)"
        }
      }
    }
```

Generate `auth` if needed:

```bash
echo -n 'my-bot-user:ghp_xxxxxxxx' | base64 -w0; echo
```

**Registry-specific notes**

| Registry | `--docker-server` | Auth tip |
|----------|-------------------|----------|
| Docker Hub | `https://index.docker.io/v1/` | Username + password or PAT |
| GHCR | `ghcr.io` | PAT with `read:packages` |
| GitLab | `registry.gitlab.com` | Deploy token / PAT |
| Quay | `quay.io` | Robot account |
| Harbor | `harbor.example.com` | Robot account |
| ECR | `<account>.dkr.ecr.<region>.amazonaws.com` | Password from `aws ecr get-login-password` (expires ~12h) — prefer **IRSA/node IAM** instead of static Secret |
| GCR/Artifact Registry | `REGION-docker.pkg.dev` | Prefer **Workload Identity** over JSON key Secrets |

### 9.2 Attach credentials (prefer ServiceAccount)

Attaching on the **ServiceAccount** applies to every pod using that SA—cleaner than repeating `imagePullSecrets` on each Deployment.

```bash
kubectl -n ${NS} create serviceaccount app \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ${NS} patch serviceaccount app -p '{"imagePullSecrets":[{"name":"regcred"}]}'
# Or edit:
# kubectl -n ${NS} edit sa app
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: demo
imagePullSecrets:
  - name: regcred
automountServiceAccountToken: false
```

### 9.3 Deploy the app

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
  namespace: ${NS}
  labels:
    app: private-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: private-app
  template:
    metadata:
      labels:
        app: private-app
    spec:
      serviceAccountName: app
      # If you did not put imagePullSecrets on the SA, uncomment:
      # imagePullSecrets:
      #   - name: regcred
      containers:
        - name: app
          image: ghcr.io/my-org/private-app:1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: private-app
  namespace: ${NS}
spec:
  selector:
    app: private-app
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl -n ${NS} rollout status deploy/private-app --timeout=120s
kubectl -n ${NS} get pods -o wide
```

**Inline Pod-level pull secret (when you cannot change the SA):**

```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: ghcr.io/my-org/private-app:1.2.3
```

### 9.4 Verify the pull worked

```bash
kubectl -n ${NS} get pods
kubectl -n ${NS} describe pod -l app=private-app | sed -n '/Events:/,$p'

# Healthy: Pulling → Pulled → Created → Started
# Broken:  ErrImagePull / ImagePullBackOff — read the event message
```

### 9.5 Troubleshooting

| Symptom / message | Fix |
|-------------------|-----|
| `pull access denied` / `unauthorized` | Wrong user/password/PAT scopes; wrong `--docker-server` host |
| `manifest unknown` | Bad tag/digest; image not pushed to that registry/project |
| Secret exists but still denied | Secret not in **same namespace**; SA not referenced; typo in `imagePullSecrets` name |
| Works on one node only | Node-local mirror/cache oddity; check that node’s kubelet logs |
| ECR auth worked yesterday | Static ECR password expired — automate refresh or use IAM |
| `http: server gave HTTP response to HTTPS client` | Insecure registry — need registry mirror / HTTP config on containerd (prefer TLS) |

```bash
# Confirm secret type & keys
kubectl -n ${NS} get secret regcred -o jsonpath='{.type}{"\n"}'
kubectl -n ${NS} get secret regcred -o jsonpath='{.data}' | jq 'keys'

# Confirm SA wiring
kubectl -n ${NS} get sa app -o yaml | grep -A3 imagePullSecrets
kubectl -n ${NS} get deploy private-app -o yaml | grep -A2 serviceAccountName

# Decode docker config (careful — shows auth material)
kubectl -n ${NS} get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' \
  | base64 -d | jq .
```

### 9.6 Production practices for private pulls

- [ ] Use a **robot / deploy token** with **read-only** pull scope — not a human password  
- [ ] Store `regcred` via **External Secrets / SOPS**, not plaintext Git  
- [ ] Put `imagePullSecrets` on the **ServiceAccount** used by the app  
- [ ] Prefer **cloud node/workload identity** (EKS IRSA, GKE WI, AKS) so you can drop pull Secrets for cloud registries  
- [ ] Pin images by **digest** in prod (`image@sha256:…`)  
- [ ] One pull Secret per registry (or one dockerconfigjson with multiple `auths` entries)  
- [ ] Rotate tokens on a schedule; `kubectl rollout restart` after replacing the Secret  
- [ ] Restrict RBAC: who can `get` `regcred` (it is as powerful as registry read)  

**Rotate pull credentials:**

```bash
kubectl -n ${NS} delete secret regcred
kubectl -n ${NS} create secret docker-registry regcred \
  --docker-server="${REGISTRY}" \
  --docker-username="${REG_USER}" \
  --docker-password="${NEW_PASS}" \
  --docker-email="ci@example.com"
kubectl -n ${NS} rollout restart deploy/private-app
```

### 9.7 Multi-namespace pattern

Each namespace needs its **own** copy of the Secret (Secrets are namespaced). Options:

```bash
# Manual copy (lab)
for ns in demo payments-prod payments-staging; do
  kubectl get secret regcred -n demo -o yaml \
    | sed "s/namespace: demo/namespace: ${ns}/" \
    | kubectl apply -f -
done
```

In production: External Secrets per namespace, Cluster reflector tools, or policy (Kyverno) to sync — don’t hand-copy forever.

### 9.8 Quick flow

```text
Create docker-registry Secret (regcred) in app namespace
  → Attach to ServiceAccount imagePullSecrets
  → Deployment uses that SA + private image URL
  → Pods Running; describe Events show Pulled
  → GitOps: ExternalSecret → regcred; never commit password
```

---

## References

- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Secret good practices](https://kubernetes.io/docs/concepts/configuration/secret/#good-practices)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [ServiceAccount tokens](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [ImagePullSecrets](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [cert-manager](https://cert-manager.io/docs/)
