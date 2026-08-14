# OIDC / SSO Access to Kubernetes (kubeadm) — Step by Step

Install and configure **OpenID Connect** so developers log in with SSO instead of sharing `admin.conf`. Works with **Dex** (in-cluster IdP broker) or an existing IdP (Keycloak, Okta, Azure AD, Google, …).

Related: [developer-access-rbac.md](./developer-access-rbac.md) · [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

Official:

- [Kubernetes authenticating — OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- [Dex + Kubernetes](https://dexidp.io/docs/guides/kubernetes/)
- [kubectl oidc-login (kubelogin)](https://github.com/int128/kubelogin)

---

## 0. What you will build

```text
Developer laptop
  → kubectl + oidc-login plugin (browser login)
  → short-lived ID token
  → Kubernetes API server (OIDC authenticator)
  → RBAC RoleBinding on User/Group claims
```

| Piece | Where | Install? |
|-------|--------|----------|
| OIDC issuer (Dex **or** corporate IdP) | Cluster / SaaS | Yes (Dex) or use existing IdP |
| API server OIDC flags / Auth config | Every control-plane | Configure (no extra package) |
| `kubectl` | Developer laptop | Already needed |
| `kubectl oidc-login` plugin | Developer laptop | **Yes** (krew or binary) |
| RBAC Roles / RoleBindings | Cluster | Yes (YAML) |
| Ingress / TLS / DNS for Dex | Cluster / LB | Yes if using Dex |

**Alternative IdPs (no Dex):** skip §2–3 Dex deploy; use your issuer URL/client and jump to §4 (API server) + §5 (kubectl plugin).

---

## 1. Prerequisites & decisions

```bash
# Platform admin: cluster reachable with break-glass kubeconfig
kubectl get nodes
kubectl version --short 2>/dev/null || kubectl version

# Decide values (example — replace everywhere)
export OIDC_ISSUER=https://dex.example.com
export OIDC_CLIENT_ID=kubernetes
export OIDC_CLIENT_SECRET='change-me-long-random'   # confidential client; for public+PKCE some setups omit secret
export OIDC_USERNAME_CLAIM=email
export OIDC_GROUPS_CLAIM=groups
export API_SERVER=https://k8s-api.example.com:6443
```

Checklist:

- [ ] DNS for Dex (e.g. `dex.example.com`) pointing at Ingress/LB  
- [ ] TLS cert for Dex (cert-manager or corporate cert) — API server must **trust** that CA  
- [ ] HA: plan to patch **each** kube-apiserver (one CP at a time)  
- [ ] IdP source of users/groups (Dex static passwords for lab; LDAP/GitHub/SAML connector for prod)  

---

## 2. Install tools on the **admin** workstation

```bash
# jq helps debugging JWT payloads
sudo apt-get update && sudo apt-get install -y jq curl

# krew (if not already) — see kubeadm-production-cluster.md §10.5
# Then install oidc-login for your own tests:
kubectl krew install oidc-login
kubectl oidc-login --help
```

Or install [kubelogin](https://github.com/int128/kubelogin) binary as `kubectl-oidc_login` on `PATH` (krew names the plugin `oidc-login`).

---

## 3. Deploy Dex (in-cluster OIDC issuer)

> Skip this section if you already have Keycloak / Okta / Azure AD / Google as issuer.

### 3.1 Namespace and TLS secret

```bash
kubectl create namespace auth

# Option A: cert-manager Certificate → Secret dex-tls
# Option B: manual TLS
kubectl -n auth create secret tls dex-tls \
  --cert=./dex-cert.pem \
  --key=./dex-key.pem
```

If Dex uses a **private CA**, copy the CA to every control plane for the API server:

```bash
# On each control-plane node:
sudo mkdir -p /etc/kubernetes/pki/oidc
sudo cp oidc-ca.crt /etc/kubernetes/pki/oidc/oidc-ca.crt
sudo chmod 644 /etc/kubernetes/pki/oidc/oidc-ca.crt
```

Public CA (Let’s Encrypt, public DigiCert, …): often **no** `oidc-ca-file` needed.

### 3.2 Dex ConfigMap (static users — lab only)

Production: replace `staticPasswords` / `staticClients` with connectors (LDAP, SAML, GitHub, OIDC to Okta, …). See [Dex connectors](https://dexidp.io/docs/connectors/).

```bash
# Generate bcrypt hash for lab user password "password" (change it!)
# docker run --rm httpd:2 apachectl -t -D DUMP_INCLUDES 2>/dev/null
# Or: python3 -c "import bcrypt; print(bcrypt.hashpw(b'password', bcrypt.gensalt()).decode())"
# Example hash below is illustrative — generate your own.

kubectl -n auth apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex
  namespace: auth
data:
  config.yaml: |
    issuer: https://dex.example.com
    storage:
      type: kubernetes
      config:
        inCluster: true
    web:
      https: 0.0.0.0:5556
      tlsCert: /etc/dex/tls/tls.crt
      tlsKey: /etc/dex/tls/tls.key
    oauth2:
      skipApprovalScreen: true
    staticClients:
      - id: kubernetes
        name: Kubernetes
        secret: change-me-long-random
        redirectURIs:
          - http://localhost:8000
          - http://localhost:18000
    enablePasswordDB: true
    staticPasswords:
      - email: "alice@example.com"
        hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
        username: "alice"
        userID: "1"
      - email: "bob@example.com"
        hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
        username: "bob"
        userID: "2"
    # Optional: static groups via connectors; for lab, use custom claims via connectors.
    # With password DB only, groups often come from a connector — for a minimal lab,
    # bind RBAC to User email first; add groups when LDAP/OIDC connector is configured.
EOF
```

> **Groups in production:** configure an LDAP/OIDC connector that returns a `groups` claim, or use Dex `connectors` + `userIDPools`. Static password DB alone often has **no groups** — bind to `User` (`alice@example.com`) until groups work.

### 3.3 Dex RBAC (store state in Kubernetes)

```bash
kubectl -n auth apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dex
  namespace: auth
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dex
rules:
  - apiGroups: ["dex.coreos.com"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["create", "get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dex
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
  - kind: ServiceAccount
    name: dex
    namespace: auth
EOF
```

### 3.4 Dex Deployment + Service + Ingress

Pin an image version from [Dex releases](https://github.com/dexidp/dex/releases):

```bash
export DEX_IMAGE=ghcr.io/dexidp/dex:v2.41.1

kubectl -n auth apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dex
  namespace: auth
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dex
  template:
    metadata:
      labels:
        app: dex
    spec:
      serviceAccountName: dex
      containers:
        - name: dex
          image: ${DEX_IMAGE}
          args: ["dex", "serve", "/etc/dex/cfg/config.yaml"]
          ports:
            - name: https
              containerPort: 5556
          volumeMounts:
            - name: config
              mountPath: /etc/dex/cfg
            - name: tls
              mountPath: /etc/dex/tls
          readinessProbe:
            httpGet:
              path: /healthz
              port: https
              scheme: HTTPS
      volumes:
        - name: config
          configMap:
            name: dex
            items:
              - key: config.yaml
                path: config.yaml
        - name: tls
          secret:
            secretName: dex-tls
---
apiVersion: v1
kind: Service
metadata:
  name: dex
  namespace: auth
spec:
  selector:
    app: dex
  ports:
    - name: https
      port: 443
      targetPort: https
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dex
  namespace: auth
  annotations:
    # adapt to your ingress controller
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["dex.example.com"]
      secretName: dex-tls
  rules:
    - host: dex.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dex
                port:
                  name: https
EOF
```

### 3.5 Verify Dex

```bash
kubectl -n auth get pods,svc,ingress
curl -fsS https://dex.example.com/home
# OIDC discovery document (API server uses this):
curl -fsS https://dex.example.com/.well-known/openid-configuration | jq .
```

Must show `"issuer": "https://dex.example.com"` matching `--oidc-issuer-url` **exactly** (no trailing slash mismatch).

---

## 4. Configure the Kubernetes API server (kubeadm)

Do this on **each control-plane node**, one at a time. Prefer persisting via kubeadm `ClusterConfiguration` so upgrades keep the flags.

### 4.1 Persist with kubeadm (recommended)

```bash
# On a control plane — edit the live config, then upload
kubectl -n kube-system get configmap kubeadm-config -o yaml > kubeadm-config-backup.yaml

# Patch ClusterConfiguration (example using a local file you then apply carefully)
# Or: kubeadm config print init-defaults > /tmp/kc.yaml and merge apiServer.extraArgs
```

Example `ClusterConfiguration` snippet:

```yaml
apiServer:
  extraArgs:
    - name: oidc-issuer-url
      value: "https://dex.example.com"
    - name: oidc-client-id
      value: "kubernetes"
    - name: oidc-username-claim
      value: "email"
    - name: oidc-groups-claim
      value: "groups"
    # optional prefixes (helps avoid clashing with system: users)
    - name: oidc-username-prefix
      value: ""
    - name: oidc-groups-prefix
      value: "oidc:"
    # required if issuer uses private CA:
    - name: oidc-ca-file
      value: "/etc/kubernetes/pki/oidc/oidc-ca.crt"
  extraVolumes:
    - name: oidc-ca
      hostPath: "/etc/kubernetes/pki/oidc/oidc-ca.crt"
      mountPath: "/etc/kubernetes/pki/oidc/oidc-ca.crt"
      readOnly: true
      pathType: File
```

> If your kubeadm version expects `extraArgs` as a **map** instead of a name/value list, use:
>
> ```yaml
> extraArgs:
>   oidc-issuer-url: "https://dex.example.com"
>   oidc-client-id: "kubernetes"
>   ...
> ```
>
> Check with `kubeadm config print init-defaults` / your installed API version.

Upload and re-sync apiserver on each CP (method depends on version; common approach):

```bash
# After updating the ClusterConfiguration in the kubeadm-config ConfigMap,
# re-generate / push apiserver manifest on each CP, OR edit static pod once then
# ensure kubeadm-config matches so future upgrades keep OIDC flags.
```

### 4.2 Immediate enable — edit static pod (each CP)

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak

sudo editable=/etc/kubernetes/manifests/kube-apiserver.yaml
# Add under command: (YAML list entries)
#   - --oidc-issuer-url=https://dex.example.com
#   - --oidc-client-id=kubernetes
#   - --oidc-username-claim=email
#   - --oidc-groups-claim=groups
#   - --oidc-groups-prefix=oidc:
#   - --oidc-ca-file=/etc/kubernetes/pki/oidc/oidc-ca.crt   # if private CA
#
# And under volumeMounts / volumes hostPath for oidc-ca if needed.
```

Or use a careful `sed`/editor. After save, kubelet recreates the pod:

```bash
# Watch API server come back (from another CP or when LB still has backends)
kubectl get pods -n kube-system -l component=kube-apiserver -o wide
kubectl get --raw='/readyz?verbose' | head
```

Repeat on **all** control planes. Client cert auth (`admin.conf`) continues to work alongside OIDC.

### 4.3 Using an existing IdP (no Dex)

| IdP | Typical issuer | Notes |
|-----|----------------|-------|
| Azure AD | `https://login.microsoftonline.com/<tenant-id>/v2.0` | Register app; expose groups claim |
| Google | `https://accounts.google.com` | Client ID from Google Cloud console |
| Okta | `https://<org>.okta.com/oauth2/default` | Or custom auth server |
| Keycloak | `https://keycloak.example.com/realms/<realm>` | Create client; mappers for groups |

Same API flags; `oidc-client-id` = that app’s client id. Create a public client (or confidential + secret) with redirect `http://localhost:8000` / `18000` for kubelogin.

```bash
# Discovery check
curl -fsS "${OIDC_ISSUER}/.well-known/openid-configuration" | jq .issuer,.jwks_uri
```

---

## 5. Install kubectl OIDC plugin on **developer** laptops

### 5.1 Install

```bash
# Option A: krew
kubectl krew update
kubectl krew install oidc-login
kubectl oidc-login --help

# Option B: binary from GitHub releases (int128/kubelogin)
# Place as kubectl-oidc_login on PATH → invoked as: kubectl oidc-login
```

### 5.2 Create developer kubeconfig (no admin keys)

```bash
# Cluster CA from platform (base64 of /etc/kubernetes/pki/ca.crt)
CA_B64=$(sudo cat /etc/kubernetes/pki/ca.crt | base64 -w0)   # macOS: base64 | tr -d '\n'

kubectl config set-cluster prod \
  --server="${API_SERVER}" \
  --certificate-authority-data="${CA_B64}"

kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-interactive-mode=IfAvailable \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url="${OIDC_ISSUER}" \
  --exec-arg=--oidc-client-id="${OIDC_CLIENT_ID}" \
  --exec-arg=--oidc-client-secret="${OIDC_CLIENT_SECRET}" \
  --exec-arg=--oidc-extra-scope=email \
  --exec-arg=--oidc-extra-scope=profile \
  --exec-arg=--oidc-extra-scope=groups

# If using PKCE public client without secret, omit --oidc-client-secret
# and add: --exec-arg=--oidc-pkce-method=S256  (per kubelogin docs for your version)

kubectl config set-context payments-dev \
  --cluster=prod \
  --user=oidc \
  --namespace=payments-dev

kubectl config use-context payments-dev
```

Distribute a ready-made file instead:

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: prod
    cluster:
      server: https://k8s-api.example.com:6443
      certificate-authority-data: <BASE64_CLUSTER_CA>
users:
  - name: oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://dex.example.com
          - --oidc-client-id=kubernetes
          - --oidc-client-secret=change-me-long-random
          - --oidc-extra-scope=email
          - --oidc-extra-scope=groups
contexts:
  - name: payments-dev
    context:
      cluster: prod
      user: oidc
      namespace: payments-dev
current-context: payments-dev
```

```bash
# Developer:
export KUBECONFIG=~/.kube/config-prod-oidc
kubectl get ns   # opens browser on first call
```

### 5.3 First login test

```bash
kubectl oidc-login get-token \
  --oidc-issuer-url="${OIDC_ISSUER}" \
  --oidc-client-id="${OIDC_CLIENT_ID}" \
  --oidc-client-secret="${OIDC_CLIENT_SECRET}" \
  --oidc-extra-scope=email \
  --oidc-extra-scope=groups

# Decode ID token payload (second JWT segment) for claims debugging:
# paste token into jwt.io OR:
kubectl oidc-login get-token ... -o json | jq -r .status.token \
  | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
```

Confirm claims include `email` (username) and `groups` (if configured). Username seen by Kubernetes must match RoleBinding `User` / `Group` names (with prefix if you set `oidc-groups-prefix`).

---

## 6. Grant RBAC after SSO works

Even with a valid token, **Unauthorized** until RoleBindings exist.

```bash
# Namespace + role (see developer-access-rbac.md for full manifests)
kubectl create namespace payments-dev --dry-run=client -o yaml | kubectl apply -f -

# Bind by user email (lab / no groups yet)
kubectl -n payments-dev create rolebinding alice-dev \
  --clusterrole=edit \
  --user=alice@example.com

# Or bind by group (production) — note oidc: prefix if you set oidc-groups-prefix=oidc:
kubectl -n payments-dev create rolebinding payments-devs \
  --clusterrole=edit \
  --group=oidc:payments-dev
```

Custom Role (preferred over `edit` if Secrets must be denied) — apply YAML from [developer-access-rbac.md](./developer-access-rbac.md).

### Verify as platform (impersonation)

```bash
kubectl auth can-i get pods -n payments-dev \
  --as=alice@example.com

kubectl auth can-i get pods -n payments-dev \
  --as=alice@example.com --as-group=oidc:payments-dev

kubectl auth can-i --list -n payments-dev \
  --as=alice@example.com --as-group=oidc:payments-dev

# Should be denied:
kubectl auth can-i get pods -n kube-system --as=alice@example.com
```

### Verify as developer

```bash
kubectl config current-context
kubectl auth can-i create deployments -n payments-dev
kubectl auth can-i get secrets -n payments-dev
kubectl get pods -n payments-dev
kubectl create deploy smoke --image=nginx:stable -n payments-dev
kubectl delete deploy smoke -n payments-dev
```

---

## 7. Command cheat sheet

### Discovery & health

```bash
curl -fsS https://dex.example.com/.well-known/openid-configuration | jq .
kubectl -n auth get pods,ingress
kubectl -n kube-system get pods -l component=kube-apiserver
kubectl get --raw=/readyz
```

### API server OIDC flags present?

```bash
# On a control plane:
sudo grep oidc /etc/kubernetes/manifests/kube-apiserver.yaml
```

### AuthN / AuthZ debug

```bash
kubectl auth can-i --list
kubectl auth can-i --list -n payments-dev
kubectl auth can-i create pods -n payments-dev --as=alice@example.com
kubectl api-resources | head

# Who am I? (if Impersonate-User not used — from audit or token claims)
kubectl oidc-login get-token ... | jq .
```

### Bindings

```bash
kubectl get rolebindings,clusterrolebindings -A | grep -i alice
kubectl describe rolebinding alice-dev -n payments-dev
kubectl get rolebinding -n payments-dev -o yaml
```

### Raise / revoke access

```bash
# Raise: add to IdP group OR create extra RoleBinding
kubectl -n payments-dev create rolebinding alice-secrets \
  --role=payments-secret-reader \
  --user=alice@example.com

# Revoke:
kubectl -n payments-dev delete rolebinding alice-dev
# Plus remove from IdP group; revoke IdP sessions
```

### Developer laptop

```bash
kubectl krew install oidc-login
export KUBECONFIG=~/.kube/config-prod-oidc
kubectl config use-context payments-dev
kubectl get ns
# Force re-login:
kubectl oidc-login clean   # if supported by your plugin version
rm -rf ~/.kube/cache/oidc-login 2>/dev/null || true
```

---

## 8. Production hardening

| Item | Guidance |
|------|----------|
| Dex HA | ≥2 replicas; durable storage if required by connector |
| TLS | Public or private CA; API server trusts issuer |
| Client type | Prefer PKCE public client for CLI; rotate secrets if confidential |
| Groups | Drive access via IdP groups + RoleBindings; avoid per-user bindings at scale |
| Prefixes | `oidc-groups-prefix` avoids colliding with Kubernetes built-in groups |
| Break-glass | Keep `admin.conf` offline for platform only |
| Auditing | Enable API audit logs; identity = email/sub from OIDC |
| Upgrades | Ensure OIDC `extraArgs` survive `kubeadm upgrade` (stored in kubeadm-config) |

---

## 9. Troubleshooting

| Symptom | Check |
|---------|--------|
| Browser login OK but `Unauthorized` | RoleBinding username/group mismatch (prefix, email vs sub) |
| `oidc: authenticator not initialized` / API crash | Issuer unreachable from API server; bad CA file; typo in issuer URL |
| Discovery fails from API | DNS from CP nodes; firewall to Dex; correct `oidc-ca-file` |
| Groups empty | Connector not sending `groups`; scope missing; claim name wrong |
| Redirect URI mismatch | Dex/IdP client must allow `http://localhost:8000` (and/or 18000) |
| Works with cert, not OIDC | OIDC flags missing on some HA API servers |

```bash
# From a control plane — can API host resolve/trust issuer?
curl -v --cacert /etc/kubernetes/pki/oidc/oidc-ca.crt \
  https://dex.example.com/.well-known/openid-configuration

kubectl -n kube-system logs -l component=kube-apiserver --tail=100 | grep -i oidc
```

---

## 10. Quick flow

```text
1. Deploy Dex (or pick existing IdP) + TLS + DNS
2. Confirm /.well-known/openid-configuration
3. Copy OIDC CA to CPs (if private)
4. Add oidc-* flags to every kube-apiserver (kubeadm-config + manifests)
5. Install kubectl oidc-login on laptops
6. Issue kubeconfig with exec oidc-login (no admin cert)
7. Create namespace Role + RoleBinding to User/Group
8. Verify: kubectl auth can-i --as=... ; developer login smoke test
9. Drive joiner/mover/leaver via IdP groups
```

---

## References

- [OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- [Dex Kubernetes guide](https://dexidp.io/docs/guides/kubernetes/)
- [kubelogin / oidc-login](https://github.com/int128/kubelogin)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [developer-access-rbac.md](./developer-access-rbac.md)
