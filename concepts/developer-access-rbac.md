# Managing Developer Access to a Kubernetes Cluster

Best practices for **least-privilege access**, how to **onboard developers**, **grant / verify** permissions, and **raise access levels** safely on a kubeadm (or similar) production cluster.

Related:

- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)
- [certificate-renewal.md](./certificate-renewal.md)
- [taints-tolerations-affinity.md](./taints-tolerations-affinity.md)
- [oidc-sso-setup.md](./oidc-sso-setup.md) — **OIDC/SSO install & login (step by step)**

Official: [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) · [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

---

## 0. Principles

| Principle | Practice |
|-----------|----------|
| **Least privilege** | Grant only what the role needs in the namespaces they own |
| **Namespace isolation** | One team / app / env per namespace (or clear folder of namespaces) |
| **No shared `admin.conf`** | Never hand out `/etc/kubernetes/admin.conf` or cluster-admin kubeconfigs to developers |
| **Short-lived credentials** | Prefer OIDC / cloud IAM / short tokens over long-lived client certs |
| **Human ≠ workload identity** | People use User/OIDC; apps use ServiceAccounts |
| **Break-glass separately** | Cluster-admin only for on-call / platform; audited and rare |
| **Verify before announce** | `kubectl auth can-i` (and reviews) before telling the user “you’re in” |
| **Change via Git** | RBAC manifests in GitOps/PR; avoid one-off `kubectl create` in prod without review |

---

## 1. Access model (what to design first)

### 1.1 Layers of control

```text
Authentication  →  who are you?     (OIDC, cert, token, cloud IAM)
Authorization   →  what can you do? (RBAC Roles / ClusterRoles)
Admission       →  what may exist?  (PSA, Kyverno/Gatekeeper, quotas)
Network         →  what can talk?   (NetworkPolicy)
```

RBAC alone is not enough: pair it with **namespaces**, **ResourceQuotas / LimitRanges**, **Pod Security**, and **NetworkPolicies**.

### 1.2 Suggested environment layout

| Namespace pattern | Who | Typical access |
|-------------------|-----|----------------|
| `app-dev` | Developers | edit / deploy in that NS |
| `app-staging` | Developers (limited) + CI | edit or deploy-only |
| `app-prod` | CI + platform; developers **read** (or break-glass) | get/list/watch; no delete secrets |
| `kube-system`, `tigera-operator`, … | Platform only | no developer access |

### 1.3 Role tiers (name them and stick to them)

| Tier | Intent | Example binding |
|------|--------|-----------------|
| **viewer** | Read workloads, logs (optional), no Secrets | Role + RoleBinding |
| **developer** | Deploy/manage own apps; no nodes/RBAC | Role + RoleBinding |
| **namespace-admin** | Full control **inside one NS** (still no cluster scope) | `admin` Role + RoleBinding |
| **ci-deployer** | Apply manifests / rollout; often no exec/secrets | Role + RoleBinding to SA |
| **platform / cluster-admin** | Cluster-wide | ClusterRoleBinding — **not for normal developers** |

Built-in ClusterRoles useful as starting points: `view`, `edit`, `admin` (namespace-scoped via RoleBinding). Prefer **custom Roles** when `edit` is too broad (e.g. it can read Secrets).

---

## 2. Limitations to consider before granting access

### 2.1 Security & data

| Risk | Mitigation |
|------|------------|
| Reading Secrets | Omit `secrets` from verbs; use Sealed Secrets / External Secrets; separate “secret-reader” rare role |
| `kubectl exec` / port-forward | Often omit in prod; allow in `*-dev` only |
| Creating privileged pods | Pod Security `restricted`; Kyverno/Gatekeeper deny `privileged`, hostNetwork, hostPath |
| Escaping to nodes | No `nodes/proxy`, no bind to privileged ClusterRoles; limit `create pods` with PSA |
| Token projection abuse | Short-lived SA tokens; no automount where unused |
| Long-lived kubeconfig leak | OIDC + short sessions; rotate; store in password manager / SSO only |

### 2.2 Stability & multi-tenancy

| Risk | Mitigation |
|------|------------|
| Noisy neighbor | ResourceQuota + LimitRange per namespace |
| Delete shared resources | No access to other teams’ namespaces; PDB awareness training |
| Cluster-scoped objects | Developers should not create CRDs, ClusterRoles, PriorityClasses, Nodes |
| LoadBalancer / NodePort sprawl | Limit who can create Services of those types (policy or restricted Role) |
| PV / storage cost | Limit PVC classes; quota on storage |

### 2.3 Compliance & ops

| Risk | Mitigation |
|------|------------|
| Unclear who did what | API audit logging; unique user identities (OIDC email/sub), not shared users |
| Prod change without review | Developers = view in prod; deploy via CI after PR |
| Orphaned access | Joiner-mover-leaver process; review RoleBindings quarterly |

### 2.4 What developers usually should **not** get

- `cluster-admin` or bind to `system:masters`
- Access to `kube-system` / CNI / etcd / cert namespaces
- `create`/`update` on `nodes`, `certificatesigningrequests` (approval), `mutatingwebhookconfigurations`
- Ability to create ClusterRoleBindings
- Permanent client certificates copied from a control plane

---

## 3. Authentication options (how they “log in”)

| Method | Production fit | Notes |
|--------|----------------|-------|
| **OIDC** (Dex, Keycloak, Okta, Azure AD, Google, …) | **Best default** | Maps groups → RoleBindings; short-lived tokens |
| Cloud IAM (EKS/GKE/AKS style) | Excellent on managed | Map IAM roles/groups to RBAC |
| **ServiceAccount + token** | CI/CD robots | Not for humans day-to-day |
| Client certificate (user) | Legacy / break-glass | Hard to revoke; prefer short validity |
| Static token / basic auth | Avoid | Removed/disabled in modern stacks |

### 3.1 OIDC (recommended pattern)

Full install guide: **[oidc-sso-setup.md](./oidc-sso-setup.md)**.

1. Deploy/configure OIDC issuer (Dex or corporate IdP).  
2. Configure API server OIDC flags on every control plane.  
3. Developers install `kubectl oidc-login` and use an exec-based kubeconfig.  
4. JWT includes `email` / `groups`.  
5. Bind Kubernetes groups to Roles:

```yaml
kind: RoleBinding
subjects:
  - kind: Group
    name: developers-app
    apiGroup: rbac.authorization.k8s.io
```

Adding a person = add them to the **IdP group**, not a new RoleBinding per human (when possible).

### 3.2 Temporary user certificate (lab / break-glass only)

```bash
# Example: issue a short-lived client cert signed by cluster CA (platform only)
openssl genrsa -out dev1.key 2048
openssl req -new -key dev1.key -out dev1.csr -subj "/CN=dev1/O=developers-app"
# sign with /etc/kubernetes/pki/ca.key (break-glass procedure; keep CA offline otherwise)
```

Prefer OIDC; if you must use certs, use **hours/days** validity and automate revocation lists / reissue.

---

## 4. Create developer access (step by step)

Example: team **payments** gets a namespace `payments-dev` with a **developer** role (deploy + exec, no secret list in this example—you can add secrets if policy allows).

### 4.1 Namespace + guardrails

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments-dev
  labels:
    team: payments
    env: dev
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-dev-quota
  namespace: payments-dev
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-dev-limits
  namespace: payments-dev
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

Optional default-deny NetworkPolicy (then allow DNS + needed peers):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: payments-dev
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

### 4.2 Custom Role (least privilege)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payments-developer
  namespace: payments-dev
rules:
  # Workloads
  - apiGroups: ["", "apps", "batch"]
    resources:
      - pods
      - pods/log
      - pods/exec          # remove in staging/prod if undesired
      - pods/portforward   # remove in prod if undesired
      - services
      - configmaps
      - persistentvolumeclaims
      - deployments
      - statefulsets
      - daemonsets
      - jobs
      - cronjobs
      - replicasets
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Often needed for apps
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Explicitly NO secrets in this example — add only if required:
  # - apiGroups: [""]
  #   resources: ["secrets"]
  #   verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Events help debugging
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

Using built-in `edit` is faster but **includes Secrets**—prefer a custom Role in regulated environments.

### 4.3 Bind users or groups

**Group (OIDC) — preferred:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-developers
  namespace: payments-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: payments-developer
subjects:
  - kind: Group
    name: oidc:payments-dev      # exact group string from your IdP / API server mapping
    apiGroup: rbac.authorization.k8s.io
```

**Individual user:**

```yaml
subjects:
  - kind: User
    name: alice@example.com     # must match authenticator username claim
    apiGroup: rbac.authorization.k8s.io
```

**CI ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-ci
  namespace: payments-dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-ci
  namespace: payments-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: payments-developer
subjects:
  - kind: ServiceAccount
    name: payments-ci
    namespace: payments-dev
```

Apply via Git/PR:

```bash
kubectl apply -f namespace-payments-dev.yaml
kubectl apply -f role-payments-developer.yaml
kubectl apply -f rolebinding-payments-developers.yaml
```

### 4.4 Read-only access to production (common pattern)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-prod-view
  namespace: payments-prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view          # built-in: no secrets
subjects:
  - kind: Group
    name: oidc:payments-dev
    apiGroup: rbac.authorization.k8s.io
```

---

## 5. Provide credentials to the developer

### 5.1 OIDC / SSO (preferred)

Full install (Dex or corporate IdP, API server flags, laptop plugin): **[oidc-sso-setup.md](./oidc-sso-setup.md)**.

#### What must be installed / configured

| Where | What | Who |
|-------|------|-----|
| Cluster | OIDC issuer (Dex **or** existing Keycloak/Okta/Azure AD/…) | Platform |
| Every control plane | kube-apiserver `--oidc-*` flags (or AuthenticationConfiguration) | Platform |
| Developer laptop | `kubectl` + **`kubectl oidc-login`** (krew plugin or kubelogin binary) | Developer / laptop mgmt |
| Cluster | Namespace Role + RoleBinding to IdP **User** or **Group** | Platform |
| IdP | User in the correct group | IdP / team lead |

Client certificates from `admin.conf` are **not** required for developers when OIDC works.

#### Platform: one-time cluster setup (summary)

```bash
# 1) Issuer discovery must work (Dex or IdP)
curl -fsS https://dex.example.com/.well-known/openid-configuration | jq .issuer

# 2) On each CP: trust private CA if needed
sudo mkdir -p /etc/kubernetes/pki/oidc
sudo cp oidc-ca.crt /etc/kubernetes/pki/oidc/oidc-ca.crt

# 3) Enable OIDC on kube-apiserver (persist in kubeadm-config; see oidc-sso-setup.md)
#    --oidc-issuer-url=https://dex.example.com
#    --oidc-client-id=kubernetes
#    --oidc-username-claim=email
#    --oidc-groups-claim=groups
#    --oidc-groups-prefix=oidc:
#    --oidc-ca-file=/etc/kubernetes/pki/oidc/oidc-ca.crt

sudo grep oidc /etc/kubernetes/manifests/kube-apiserver.yaml
kubectl -n kube-system get pods -l component=kube-apiserver
```

#### Developer laptop: install login plugin

```bash
# krew
kubectl krew update
kubectl krew install oidc-login
kubectl oidc-login --help

# First login test (opens browser)
kubectl oidc-login get-token \
  --oidc-issuer-url=https://dex.example.com \
  --oidc-client-id=kubernetes \
  --oidc-client-secret='change-me-long-random' \
  --oidc-extra-scope=email \
  --oidc-extra-scope=groups
```

#### Platform: build & hand out kubeconfig (no private keys)

```bash
export API_SERVER=https://k8s-api.example.com:6443
export OIDC_ISSUER=https://dex.example.com
export OIDC_CLIENT_ID=kubernetes
export OIDC_CLIENT_SECRET='change-me-long-random'
CA_B64=$(sudo base64 -w0 /etc/kubernetes/pki/ca.crt)   # macOS: base64 < ca.crt | tr -d '\n'

# Write a shareable kubeconfig template
cat > kubeconfig-payments-dev-oidc.yaml <<EOF
apiVersion: v1
kind: Config
clusters:
  - name: prod
    cluster:
      server: ${API_SERVER}
      certificate-authority-data: ${CA_B64}
users:
  - name: oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=${OIDC_ISSUER}
          - --oidc-client-id=${OIDC_CLIENT_ID}
          - --oidc-client-secret=${OIDC_CLIENT_SECRET}
          - --oidc-extra-scope=email
          - --oidc-extra-scope=groups
contexts:
  - name: payments-dev
    context:
      cluster: prod
      user: oidc
      namespace: payments-dev
current-context: payments-dev
EOF

# Deliver kubeconfig-payments-dev-oidc.yaml via Vault / 1Password / MDM — not Slack plaintext
```

Same thing with imperative commands:

```bash
kubectl config --kubeconfig=kubeconfig-payments-dev-oidc.yaml set-cluster prod \
  --server="${API_SERVER}" --certificate-authority-data="${CA_B64}"

kubectl config --kubeconfig=kubeconfig-payments-dev-oidc.yaml set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url="${OIDC_ISSUER}" \
  --exec-arg=--oidc-client-id="${OIDC_CLIENT_ID}" \
  --exec-arg=--oidc-client-secret="${OIDC_CLIENT_SECRET}" \
  --exec-arg=--oidc-extra-scope=email \
  --exec-arg=--oidc-extra-scope=groups

kubectl config --kubeconfig=kubeconfig-payments-dev-oidc.yaml set-context payments-dev \
  --cluster=prod --user=oidc --namespace=payments-dev

kubectl config --kubeconfig=kubeconfig-payments-dev-oidc.yaml use-context payments-dev
```

#### IdP: add the person

```text
1. Create/enable user in IdP (or Dex staticPasswords for lab only)
2. Add user to group that maps to RoleBinding (e.g. payments-dev → oidc:payments-dev)
3. No Kubernetes change needed if RoleBinding already targets that Group
```

#### Developer: first use

```bash
mkdir -p ~/.kube
cp kubeconfig-payments-dev-oidc.yaml ~/.kube/config-prod-oidc
export KUBECONFIG=~/.kube/config-prod-oidc

kubectl get ns                 # browser opens → SSO login
kubectl auth can-i --list -n payments-dev
kubectl get pods -n payments-dev
```

#### After SSO: ensure RBAC exists

```bash
# Group binding (preferred)
kubectl -n payments-dev create rolebinding payments-developers \
  --role=payments-developer \
  --group=oidc:payments-dev \
  --dry-run=client -o yaml | kubectl apply -f -

# Or single user (lab / exception)
kubectl -n payments-dev create rolebinding alice-dev \
  --role=payments-developer \
  --user=alice@example.com \
  --dry-run=client -o yaml | kubectl apply -f -
```

Username/group strings **must** match token claims (and `oidc-groups-prefix`). Debug claims:

```bash
kubectl oidc-login get-token \
  --oidc-issuer-url=https://dex.example.com \
  --oidc-client-id=kubernetes \
  --oidc-client-secret='change-me-long-random' \
  --oidc-extra-scope=email --oidc-extra-scope=groups -o json \
  | jq -r .status.token | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
```

### 5.2 Kubeconfig with client cert (discouraged for routine access)

1. Platform generates user cert/key.  
2. Build kubeconfig with `cluster` + `user` (client-certificate / client-key).  
3. Deliver via **secure channel** (1Password, Vault, encrypted ticket)—never Slack/email plaintext.  
4. Set expiry; calendar rotation.  
5. Document revocation (remove RoleBinding + rely on cert expiry).

```bash
# Break-glass style only — prefer OIDC
openssl genrsa -out alice.key 2048
openssl req -new -key alice.key -out alice.csr -subj "/CN=alice@example.com/O=payments-dev"
# Sign CSR with cluster CA (platform procedure), then:
kubectl config set-credentials alice \
  --client-certificate=alice.crt --client-key=alice.key
kubectl config set-context alice-payments \
  --cluster=prod --user=alice --namespace=payments-dev
```

### 5.3 What to send in the “access granted” message

- Cluster name / purpose (`nonprod` vs `prod`)  
- Default namespace  
- How to install `kubectl` + `kubectl oidc-login` ([oidc-sso-setup.md](./oidc-sso-setup.md) §5)  
- Attached/secure-link kubeconfig (OIDC exec — **no** admin key)  
- Allowed actions (tier name + link to Role YAML)  
- Forbidden actions (no `kube-system`, no cluster-admin)  
- Support channel / how to request more access  
- Self-check commands:

```bash
export KUBECONFIG=~/.kube/config-prod-oidc
kubectl config current-context
kubectl auth can-i --list -n payments-dev
kubectl auth can-i create deployments -n payments-dev
kubectl auth can-i get secrets -n payments-dev
```

---

## 6. Grant and verify access

### 6.1 Platform verification (before notifying the user)

```bash
# Impersonate the user or group (platform admin)
kubectl auth can-i --as=alice@example.com --as-group=oidc:payments-dev \
  create deployments -n payments-dev

kubectl auth can-i --as=alice@example.com --as-group=oidc:payments-dev \
  get secrets -n payments-dev
# Expect: no (if secrets omitted)

kubectl auth can-i --as=alice@example.com --as-group=oidc:payments-dev \
  get pods -n kube-system
# Expect: no

# Full list as that user in a namespace
kubectl auth can-i --list --as=alice@example.com --as-group=oidc:payments-dev -n payments-dev
```

Check bindings:

```bash
kubectl get rolebindings,clusterrolebindings -A | grep -i payments
kubectl describe rolebinding payments-developers -n payments-dev
kubectl get role payments-developer -n payments-dev -o yaml
```

### 6.2 Developer self-verification

```bash
kubectl config current-context
kubectl config view --minify
kubectl get ns
kubectl get pods -n payments-dev

kubectl auth can-i create deployments -n payments-dev
kubectl auth can-i get secrets -n payments-dev
kubectl auth can-i --list -n payments-dev
```

Smoke test (dev only):

```bash
kubectl -n payments-dev create deploy access-smoke --image=nginx:stable --replicas=1
kubectl -n payments-dev get deploy access-smoke
kubectl -n payments-dev delete deploy access-smoke
```

### 6.3 Audit trail

```bash
# Who is bound where
kubectl get rolebindings -n payments-dev -o wide
# API audit logs (SIEM): search for user alice@example.com
```

---

## 7. Raise access level (requested privilege increase)

Treat upgrades as **change requests**, not chat-only grants.

### 7.1 Request intake (what to ask)

- Requester + manager / team lead approval  
- Cluster + namespace(s)  
- **Why** (ticket / use case)  
- Duration (permanent vs time-boxed)  
- Tier requested: viewer → developer → namespace-admin → (rarely) broader  
- Prod or non-prod  

### 7.2 Decision guide

| Request | Prefer |
|---------|--------|
| “Need to see logs in prod” | `view` + `pods/log` in prod NS (or platform dashboards) |
| “Need to deploy to prod” | CI ServiceAccount + PR; human stays `view` |
| “Need secrets” | Prefer External Secrets / sealed; if human needs get, separate Role + approval |
| “Need exec in prod” | Break-glass RoleBinding with **TTL** (see below) |
| “Need cluster-admin” | Deny for developers; platform on-call only |

### 7.3 Implement the upgrade (commands)

```bash
# Prefer IdP: add user to higher group (e.g. payments-ns-admin) — no kubectl if binding exists

# Or apply additive Role + RoleBinding from Git
kubectl apply -f role-payments-secret-reader.yaml
kubectl apply -f rolebinding-alice-secrets.yaml

# Or imperative
kubectl -n payments-dev create role payments-secret-reader \
  --verb=get,list,watch --resource=secrets \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n payments-dev create rolebinding alice-secret-reader \
  --role=payments-secret-reader \
  --user=alice@example.com \
  --dry-run=client -o yaml | kubectl apply -f -

# Promote to namespace admin (broad — use sparingly)
kubectl -n payments-dev create rolebinding alice-ns-admin \
  --clusterrole=admin \
  --user=alice@example.com

# Verify new powers
kubectl auth can-i get secrets -n payments-dev --as=alice@example.com
kubectl auth can-i --list -n payments-dev --as=alice@example.com --as-group=oidc:payments-dev
```

Example: grant **secret read** without full admin:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payments-secret-reader
  namespace: payments-dev
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-secret-readers
  namespace: payments-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: payments-secret-reader
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
```

### 7.4 Time-boxed / break-glass access

```bash
# Example: temporary binding; delete after window (automate with Job/controller or calendar)
kubectl create rolebinding alice-breakglass-prod \
  --role=payments-developer \
  --user=alice@example.com \
  -n payments-prod

# After incident window
kubectl delete rolebinding alice-breakglass-prod -n payments-prod
```

Better: use a controller (e.g. rbac-sync, temporary AccessRequest CRDs) or IdP group membership with automatic removal.

### 7.5 Communicate the new level

Tell the user:

1. What changed (Role/RoleBinding/group).  
2. Where it applies (namespaces).  
3. How to re-auth if group-based (re-login).  
4. How to verify (`kubectl auth can-i …`).  
5. When it expires (if temporary).  
6. How to roll back / who to contact.

---

## 8. Revoke access (leaver / mover)

1. Remove IdP group membership (primary).  
2. Delete orphaned RoleBindings for that User.  
3. Revoke refresh tokens / disable IdP account.  
4. If client certs: rely on expiry; remove bindings immediately (cert without binding is useless for RBAC).  
5. Rotate any shared secrets they could have read.  
6. Check CI keys / personal access they owned.

```bash
# Find all bindings mentioning the user
kubectl get rolebinding,clusterrolebinding -A -o json \
  | jq -r '
    .items[]
    | select(.subjects[]? | .name=="alice@example.com")
    | "\(.kind)/\(.metadata.namespace // "-")/\(.metadata.name)"'

# Delete a known binding
kubectl -n payments-dev delete rolebinding alice-dev

# Cluster-scoped (rare for developers — remove if wrongly granted)
kubectl delete clusterrolebinding alice-something

# Force developer re-login after group removal
# (developer side)
rm -rf ~/.kube/cache/oidc-login 2>/dev/null || true
kubectl oidc-login clean 2>/dev/null || true
```

---

## 9. Ongoing best practices checklist

- [ ] No developer has `cluster-admin`  
- [ ] Prod deploy path is CI + review; humans mostly `view` in prod  
- [ ] RBAC in Git; changes via PR  
- [ ] Quotas + PSA on every team namespace  
- [ ] Secrets access minimized and audited  
- [ ] Quarterly access review (bindings ↔ active employees)  
- [ ] On-call break-glass documented and logged  
- [ ] API audit logs retained per compliance  
- [ ] kubectl + oidc-login install docs published ([oidc-sso-setup.md](./oidc-sso-setup.md))  

---

## 10. Command reference (copy/paste)

### Namespace & guardrails

```bash
kubectl create namespace payments-dev
kubectl label namespace payments-dev team=payments env=dev \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

kubectl -n payments-dev apply -f quota.yaml
kubectl -n payments-dev apply -f limitrange.yaml
kubectl -n payments-dev apply -f networkpolicy-default-deny.yaml
```

### Roles & bindings

```bash
kubectl -n payments-dev apply -f role-payments-developer.yaml
kubectl -n payments-dev apply -f rolebinding-payments-developers.yaml

kubectl -n payments-dev create rolebinding payments-developers \
  --role=payments-developer --group=oidc:payments-dev

kubectl -n payments-dev create rolebinding alice-dev \
  --role=payments-developer --user=alice@example.com

kubectl -n payments-prod create rolebinding payments-prod-view \
  --clusterrole=view --group=oidc:payments-dev

kubectl get roles,rolebindings -n payments-dev
kubectl describe role payments-developer -n payments-dev
kubectl describe rolebinding payments-developers -n payments-dev
kubectl get rolebinding,clusterrolebinding -A | grep -i payments
```

### ServiceAccount (CI)

```bash
kubectl -n payments-dev create sa payments-ci
kubectl -n payments-dev create rolebinding payments-ci \
  --role=payments-developer --serviceaccount=payments-dev:payments-ci

# Token for CI (K8s 1.24+ create token)
kubectl -n payments-dev create token payments-ci --duration=1h
```

### Verify / impersonate

```bash
kubectl auth can-i create deployments -n payments-dev \
  --as=alice@example.com --as-group=oidc:payments-dev

kubectl auth can-i get secrets -n payments-dev --as=alice@example.com
kubectl auth can-i get pods -n kube-system --as=alice@example.com
kubectl auth can-i --list -n payments-dev \
  --as=alice@example.com --as-group=oidc:payments-dev
```

### OIDC (see also [oidc-sso-setup.md](./oidc-sso-setup.md))

```bash
kubectl krew install oidc-login
curl -fsS https://dex.example.com/.well-known/openid-configuration | jq .issuer
sudo grep oidc /etc/kubernetes/manifests/kube-apiserver.yaml

export KUBECONFIG=~/.kube/config-prod-oidc
kubectl get ns
kubectl oidc-login get-token --oidc-issuer-url=... --oidc-client-id=... 
```

### Raise / revoke

```bash
kubectl -n payments-dev create rolebinding alice-secret-reader \
  --role=payments-secret-reader --user=alice@example.com

kubectl -n payments-prod create rolebinding alice-breakglass \
  --role=payments-developer --user=alice@example.com
kubectl -n payments-prod delete rolebinding alice-breakglass

kubectl -n payments-dev delete rolebinding alice-dev
```

---

## 11. Quick flow

```text
[Once] Deploy IdP/Dex + API server OIDC flags + document oidc-login install
Design tiers (viewer / developer / ns-admin / ci)
  → create namespace + quota + PSA (+ NetworkPolicy)
  → create Role (least privilege) + RoleBinding to IdP Group
  → add user to IdP group (or bind User)
  → verify with: kubectl auth can-i --as=... --as-group=...
  → send OIDC kubeconfig + install steps (not admin.conf)
  → raise level: ticket → PR/group → re-verify → notify
  → leave: remove group + bindings; rotate exposed secrets
```

---

## References

- [RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [OIDC / SSO setup (this repo)](./oidc-sso-setup.md)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [API Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/api-server-access/)
- [kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth_can-i/)
- [kubectl oidc-login](https://github.com/int128/kubelogin)
