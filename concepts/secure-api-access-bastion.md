# Secure External Access to the API (Bastion, VPN, Jump Host)

How to use `kubectl` from your laptop when **control-plane nodes and `:6443` are not on the public internet**.

Related: [kubeadm-production-cluster.md](../kubeadm-production-cluster.md) · [developer-access-rbac.md](./developer-access-rbac.md) · [oidc-sso-setup.md](./oidc-sso-setup.md) · [domain-dns-ingress-ha.md](./domain-dns-ingress-ha.md)

Official: [Controlling access](https://kubernetes.io/docs/concepts/security/controlling-access/) · [API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

---

## 0. Why keep the control plane private

| Risk if `:6443` is public | Mitigation |
|---------------------------|------------|
| Internet-wide authn/authz attacks, CVE scanning | Private API + controlled path in |
| Credential stuffing / stolen kubeconfig abuse from anywhere | Path requires VPN/bastion + short-lived creds |
| Confusion between **Ingress (apps)** and **API (cluster)** | Public HTTPS for apps ≠ public Kubernetes API |

**Recommended default:**

```text
Public Internet
  → Ingress / LoadBalancer   (apps only: 80/443)

Private network / VPN / bastion
  → Kubernetes API VIP :6443  (control plane)
```

Workers’ NodePorts and the API should not be casually exposed. Apps use Ingress ([ingress-zero-to-hero.md](./ingress-zero-to-hero.md)).

---

## 1. Access patterns (pick one primary)

| Pattern | How laptop reaches API | Best for |
|---------|------------------------|----------|
| **A. VPN into private network** | WireGuard / Tailscale / OpenVPN / cloud VPN → then `kubectl` to private `k8s-api` VIP | Teams; simplest UX |
| **B. Bastion / jump host (SSH)** | SSH to bastion in private net → LocalForward `6443` or `kubectl` on bastion | Small teams; break-glass |
| **C. SSH SOCKS / ProxyCommand** | Dynamic tunnel; kubectl via HTTPS_PROXY or `proxy-url` | No VPN client install |
| **D. Identity-aware proxy** | Teleport / Cloudflare Access / Azure Bastion+AAD / AWS SSM+tunnel | Enterprise SSO + audit |
| **E. Cloud private endpoint** | Private Link / PSC + corp network | Managed cloud clusters |

**Production preference:** **VPN or identity-aware access (A/D)** for daily use; **bastion (B)** as break-glass and for automation runners that must stay off the public API.

Never: open security group / firewall so the world can hit `https://<public-ip>:6443`.

---

## 2. Target address on the private side

Your kubeconfig `server:` should be the **private** control-plane endpoint (same as kubeadm `controlPlaneEndpoint`):

```text
https://k8s-api.internal.example.com:6443
# or https://10.0.0.10:6443  (HA VIP / internal LB)
```

```bash
# From bastion or VPN, must resolve and connect:
curl -vk https://k8s-api.internal.example.com:6443/readyz
nc -vz k8s-api.internal.example.com 6443
```

Use an **internal** DNS name and TLS cert SANs that match (kubeadm `apiServer.certSANs`). Avoid pointing laptop kubeconfigs at a single control-plane node IP.

---

## 3. Pattern A — VPN (best daily UX)

### 3.1 Idea

```text
Laptop --(WireGuard/Tailscale/corp VPN)--> private CIDR
                                            → k8s-api:6443
                                            → (optional) SSH to nodes via bastion only
```

### 3.2 Practices

- [ ] Split-tunnel or full-tunnel per security policy  
- [ ] VPN MFA; device posture if available  
- [ ] API VIP only routable on VPN/private routes — not 0.0.0.0/0 on a public LB  
- [ ] Still use OIDC / short-lived creds ([oidc-sso-setup.md](./oidc-sso-setup.md))  
- [ ] Do **not** expose node SSH on the VPN without MFA + CA keys  

### 3.3 Laptop kubeconfig

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: prod
    cluster:
      server: https://k8s-api.internal.example.com:6443
      certificate-authority-data: <base64>
users:
  - name: oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args: ["oidc-login", "get-token", ...]
contexts:
  - name: prod
    context:
      cluster: prod
      user: oidc
current-context: prod
```

Connect VPN → `kubectl get nodes`.

---

## 4. Pattern B — Bastion / jump host (detailed)

### 4.1 What a bastion is

A **hardened VM** (or small instance) that:

- Has a **controlled** public SSH (or only via VPN / IAP / SSM)  
- Can reach the **private** API VIP and (optionally) private node SSH  
- Is **not** a control-plane node and **not** a general-purpose desktop  

```text
Laptop --SSH:22--> Bastion --private--> k8s-api:6443
                      └-------------→ worker SSH (optional, break-glass)
```

### 4.2 Bastion sizing & placement

| Item | Guidance |
|------|----------|
| Count | **≥2** bastions behind an internal pattern, or one + documented break-glass; avoid one forever-unpatched box |
| Network | Public subnet with tiny SG **or** no public IP (SSM/IAP only) |
| Reachability | Route to API VIP / pod CIDR **not** required for kubectl (only API); node SSH optional |
| OS | Minimal image; auto-updates; no Docker “playground” |
| Identity | SSO + SSH CA (Teleport) or short-lived certs; **no shared ubuntu password** |

### 4.3 Hardening checklist

- [ ] SSH key or cert only; `PasswordAuthentication no`  
- [ ] MFA (pam, hardware key, or Teleport)  
- [ ] Allowlist CIDRs for SSH if public (corp egress IPs) — still prefer VPN/IAP  
- [ ] `AllowUsers` / groups; disable root login  
- [ ] fail2ban or equivalent; auditd  
- [ ] No workload kubeconfig with `cluster-admin` left world-readable  
- [ ] Bastion role: forward only — don’t run random apps  
- [ ] Separate **prod** and **nonprod** bastions  
- [ ] Session recording if compliance requires (Teleport, etc.)  
- [ ] Patch monthly; immutable ASG/recreate preferred  

Example `sshd_config` fragments:

```text
PasswordAuthentication no
PermitRootLogin no
AllowTcpForwarding yes          # needed for -L tunnels
X11Forwarding no
AllowAgentForwarding no         # prefer no unless required
ClientAliveInterval 30
MaxAuthTries 3
```

Firewall (conceptual):

```text
Inbound SSH:   corp VPN CIDR or IAP only → bastion:22
Outbound:      bastion → k8s-api VIP:6443
Outbound:      bastion → worker:22 (optional break-glass)
Deny:          Internet → :6443 on masters
```

### 4.4 Option B1 — SSH local port forward (kubectl stays on laptop)

```bash
# Terminal 1: map local 6443 to private API via bastion
ssh -N -L 6443:k8s-api.internal.example.com:6443 bastion.example.com

# If API cert is for k8s-api.internal.example.com, use:
# ssh -N -L 6443:k8s-api.internal.example.com:6443 ...
```

Laptop kubeconfig pointing at **localhost** (TLS name mismatch risk):

**Problem:** API server cert says `k8s-api.internal.example.com`, but you connect to `https://127.0.0.1:6443`.

**Fixes (pick one):**

1. **Better:** use `ProxyCommand` / SOCKS (B2) or VPN so SNI/hostname stay correct.  
2. Add `tls-server-name` in kubeconfig (kubectl):

```yaml
clusters:
  - name: prod
    cluster:
      server: https://127.0.0.1:6443
      certificate-authority-data: <base64>
      tls-server-name: k8s-api.internal.example.com
```

```bash
# Or:
kubectl config set-cluster prod \
  --server=https://127.0.0.1:6443 \
  --tls-server-name=k8s-api.internal.example.com
```

3. Use `/etc/hosts` on laptop: `127.0.0.1 k8s-api.internal.example.com` + forward `6443:k8s-api.internal.example.com:6443`, kubeconfig `server: https://k8s-api.internal.example.com:6443`.

```bash
# Terminal 2
export KUBECONFIG=~/.kube/config-prod
kubectl get nodes
```

Helper alias:

```bash
alias k8s-tunnel='ssh -N -L 6443:k8s-api.internal.example.com:6443 bastion.example.com'
```

### 4.5 Option B2 — `ProxyCommand` (no separate forward terminal)

```bash
# ~/.ssh/config
Host bastion
  HostName bastion.example.com
  User ubuntu
  IdentityFile ~/.ssh/id_bastion
  ForwardAgent no

Host k8s-api-tunnel
  HostName k8s-api.internal.example.com
  Port 6443
  ProxyJump bastion
  # Note: ProxyJump is for SSH; for HTTPS API use LocalForward or corkscrew-style
```

For HTTPS API, common pattern is **SSH dynamic SOCKS**:

```bash
ssh -N -D 1080 bastion.example.com
```

```yaml
# kubeconfig
clusters:
  - name: prod
    cluster:
      server: https://k8s-api.internal.example.com:6443
      certificate-authority-data: <base64>
      proxy-url: socks5://127.0.0.1:1080
```

Requires kubectl with `proxy-url` support (recent kubectl). Alternatively:

```bash
export HTTPS_PROXY=socks5://127.0.0.1:1080
kubectl get nodes
```

### 4.6 Option B3 — run kubectl **on** the bastion

```bash
ssh bastion.example.com
export KUBECONFIG=~/.kube/config   # OIDC or short-lived; not admin.conf world-readable
kubectl get nodes
```

Practices:

- [ ] Install kubectl + oidc-login on bastion **or** use SSO only  
- [ ] Home directories encrypted; kubeconfigs `chmod 600`  
- [ ] Prefer ephemeral sessions; don’t leave `cluster-admin` files forever  
- [ ] Audit who SSH’d and which kubeconfig was used  

Good for break-glass; daily UX is worse than VPN.

### 4.7 Option B4 — `kubectl` via SSH exec (quick one-offs)

```bash
ssh bastion.example.com kubectl --kubeconfig=/home/ops/.kube/config get nodes
```

Fine for scripts; still harden bastion credentials.

---

## 5. Pattern D — Identity-aware access (enterprise best practice)

Examples: **Teleport**, **Boundary**, **Cloudflare Access + tunnel**, **AWS SSM Session Manager**, **Google IAP**, **Azure Bastion + AAD**.

Benefits:

- SSO / MFA to reach the API or SSH  
- Session recording & short-lived certs  
- No standing public SSH on bastion  

Sketch (Teleport Kubernetes access):

```text
Laptop → Teleport (SSO) → signed kubeconfig → private API
```

Bastion becomes Teleport node or disappears behind SSM/IAP with **no** inbound :22 from Internet.

---

## 6. CI / automation access

| Runner location | Approach |
|-----------------|----------|
| Private runners on same network | Talk to private API VIP directly; OIDC/SA tokens |
| GitHub-hosted public runners | **Do not** open :6443; use self-hosted runners in VPC, or deploy via GitOps agent **pull** (Argo CD) |
| Argo CD on hub | Hub in private net; laptop uses bastion/VPN to reach Argo UI only |

Prefer **GitOps pull** ([argocd-production.md](./argocd-production.md)) over giving cloud CI a public API endpoint.

---

## 7. What **not** to do

| Anti-pattern | Why |
|--------------|-----|
| Public NLB/SG to `:6443` “just for me” | Entire Internet can probe; stolen creds = instant access |
| `admin.conf` on laptop + public API | Highest blast radius |
| Bastion = control-plane node | Compromised bastion = etcd/API host |
| Shared SSH user + shared kubeconfig | No attribution |
| Long-lived client certs on bastion world-readable | Lateral movement gold |
| Expose `kubectl proxy` or dashboard publicly | Same as public API |

---

## 8. Production recommendation (summary)

```text
Daily developer access
  → Corp VPN or Tailscale/WireGuard to private API VIP
  → OIDC kubeconfig (no admin.conf)
  → RBAC least privilege

Break-glass / platform
  → Hardened bastion (preferably no public SSH: SSM/IAP/Teleport)
  → SSH tunnel or SOCKS with tls-server-name
  → Short-lived elevated RoleBinding

Apps for end users
  → Ingress 80/443 only (separate from API)

Automation
  → In-VPC runners or Argo CD pull; never public :6443 for CI
```

### Minimal bastion tunnel cheat sheet

```bash
# ~/.ssh/config
Host k8s-bastion
  HostName bastion.example.com
  User ops
  IdentityFile ~/.ssh/id_bastion

# Start tunnel
ssh -N -L 6443:k8s-api.internal.example.com:6443 k8s-bastion

# kubeconfig fragment
# server: https://127.0.0.1:6443
# tls-server-name: k8s-api.internal.example.com
```

### Verify privacy

```bash
# From Internet (should FAIL)
nc -vz <public-ip-of-master> 6443

# From VPN/bastion (should WORK)
nc -vz k8s-api.internal.example.com 6443
kubectl get --raw='/readyz'
```

---

## 9. Quick flow

```text
1. Keep API LB / controlPlaneEndpoint internal only
2. Expose apps via Ingress, not the API
3. Choose VPN (daily) + bastion/IAP (break-glass)
4. Point kubeconfig at private DNS name
5. Use OIDC + RBAC; never distribute admin.conf
6. Harden bastion; prefer 2 + session audit
7. Test: public :6443 closed; VPN/tunnel works
```

---

## References

- [Controlling access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [kubeadm HA / controlPlaneEndpoint](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [OIDC SSO setup](./oidc-sso-setup.md)
- [Developer access / RBAC](./developer-access-rbac.md)
- [Teleport Kubernetes access](https://goteleport.com/docs/enroll-resources/kubernetes-access/)
- [AWS SSM port forwarding](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started.html)
- [Google IAP](https://cloud.google.com/iap/docs)
