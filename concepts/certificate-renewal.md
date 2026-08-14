# Certificate Renewal in Production (kubeadm)

How to **monitor, renew, and recover** TLS certificates for a kubeadm-built Kubernetes cluster in production.

Related:

- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)
- [kubeadm-upgrade-skew-policy.md](./kubeadm-upgrade-skew-policy.md)
- [external-etcd-tarball-systemd.md](../external-etcd-tarball-systemd.md)

Official: [Certificate management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)

---

## 0. Why this matters

kubeadm issues control-plane leaf certificates with a default lifetime of **~1 year**. Cluster CAs default to **~10 years**. If leaf certs expire:

- `kubectl` fails with `x509: certificate has expired`
- API server / etcd / controller-manager may refuse connections
- Nodes can go `NotReady`

Production goal: **never discover expiry in an outage**—monitor early, renew on a schedule (or via upgrades), verify HA.

---

## 1. What certificates exist

### 1.1 kubeadm-managed PKI (control plane)

Typical layout on each control-plane node under `/etc/kubernetes/pki/`:

| Certificate / kubeconfig | Role |
|--------------------------|------|
| `ca.crt` / `ca.key` | Cluster CA (signs API + client certs) |
| `apiserver.crt` | API server serving cert |
| `apiserver-kubelet-client.crt` | API server → kubelet client |
| `front-proxy-ca.crt` | Front-proxy CA |
| `front-proxy-client.crt` | Aggregator / front-proxy client |
| `sa.pub` / `sa.key` | Service-account signing key pair (not an X.509 leaf renew via same path, but part of PKI) |
| `etcd/ca.crt` | etcd CA (stacked etcd) |
| `etcd/server.crt`, `peer.crt`, `healthcheck-client.crt`, `apiserver-etcd-client.crt` | etcd + API↔etcd |

Embedded client certs in kubeconfigs:

| File | Used by |
|------|---------|
| `/etc/kubernetes/admin.conf` | Cluster-admin kubectl on the node |
| `/etc/kubernetes/controller-manager.conf` | kube-controller-manager |
| `/etc/kubernetes/scheduler.conf` | kube-scheduler |
| `/etc/kubernetes/kubelet.conf` | kubelet bootstrap / auth (see §1.2) |

**Defaults (kubeadm):**

| Material | Typical lifetime |
|----------|------------------|
| Cluster / etcd CAs | ~10 years |
| Leaf certs + kubeconfig client certs | ~1 year |

`kubeadm certs renew` renews **leaves** (and kubeconfig client certs). It does **not** rotate the CA. CA expiry is a separate, rare, high-impact project.

### 1.2 Kubelet certificates (per node)

| Material | Path | Who renews |
|----------|------|------------|
| Kubelet **client** cert | `/var/lib/kubelet/pki/kubelet-client-current.pem` (symlink) | **Kubelet** (CSR to API), not `kubeadm certs renew` |
| Kubelet **serving** cert | `/var/lib/kubelet/pki/` | Kubelet + CSR approval when rotation enabled |

kubeadm sets `rotateCertificates: true` in kubelet config so client certs rotate automatically while the API is reachable.

### 1.3 External etcd / ingress / app TLS

- **External etcd:** your own CA/client certs (see external-etcd guide)—kubeadm does not renew them.
- **ingress / cert-manager:** separate lifecycle (Let’s Encrypt, company PKI).
- **apiserver-etcd-client** on CPs must stay in sync with whatever etcd trusts.

---

## 2. Production operating model

| Practice | Detail |
|----------|--------|
| **Inventory** | Know stacked vs external etcd; where admin kubeconfigs live |
| **Monitor** | Alert when any kubeadm leaf or kubelet client cert &lt; **30–60 days** |
| **Renew window** | Schedule renewal at **~60–90 days** before expiry (or rely on timely `kubeadm upgrade`, which renews certs) |
| **HA procedure** | Renew **one control plane at a time**; keep API LB healthy |
| **Backup first** | Copy `/etc/kubernetes/pki` (+ etcd snapshot) before renewal |
| **Distribute admin.conf** | Refresh bastion/CI kubeconfigs after renewing `admin.conf` |
| **CA watch** | Calendar CA expiry years ahead; plan CA rotation as its own program |
| **Never ignore** | Expired certs → restore from backup / renew offline; prefer prevention |

Recommended calendar:

```text
Monthly:   kubeadm certs check-expiration on each CP (+ metrics scrape)
Quarterly: verify kubelet client cert dates on a sample of nodes
Yearly:    planned leaf renewal (if no upgrade already renewed them)
~Year 8–9: plan CA rotation project
```

---

## 3. Check expiration

### 3.1 Control plane (kubeadm)

On **each** control-plane node:

```bash
sudo kubeadm certs check-expiration
```

Example columns: `CERTIFICATE`, `EXPIRES`, `RESIDUAL TIME`, `CERTIFICATE AUTHORITY`, `EXTERNALLY MANAGED`.

Also inspect a single file:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates -subject
sudo openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -dates
```

### 3.2 Kubeconfigs

```bash
# Client cert embedded in admin.conf
sudo grep client-certificate-data /etc/kubernetes/admin.conf | awk '{print $2}' \
  | base64 -d | openssl x509 -noout -dates

# Or with kubectl (if still working)
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d | openssl x509 -noout -dates
```

### 3.3 Kubelet client cert

On any node:

```bash
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
# or newest kubelet-client-*.pem if symlink missing
ls -l /var/lib/kubelet/pki/
```

Confirm rotation is enabled:

```bash
grep -E 'rotateCertificates|serverTLSBootstrap' /var/lib/kubelet/config.yaml
# rotateCertificates: true
```

### 3.4 Monitoring (production)

Export metrics or run a CronJob/exporter that fails if residual time &lt; threshold:

- Prefer a dedicated check that runs `kubeadm certs check-expiration` or openssl against PKI paths on each CP.
- Alert on kubelet CSR backlog / node NotReady spikes near known expiry dates.
- Track **CA** residual years, not only leaves.

Example one-liner for residual days on apiserver cert (for scripts):

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate \
  | cut -d= -f2 \
  | xargs -I{} date -d {} +%s
# compare to $(date +%s); alert if days_left < 30
```

---

## 4. Automatic renewal paths

### 4.1 During `kubeadm upgrade`

Control-plane upgrades renew kubeadm-managed certificates as part of the process (when applicable). Keeping a regular upgrade cadence often prevents surprise 1-year leaf expiry—but **do not rely on upgrades alone**; still monitor.

### 4.2 Kubelet client auto-rotation

While API server is up:

1. Kubelet creates a CSR before expiry (roughly in the last portion of cert lifetime).  
2. controller-manager auto-approves (default kubeadm RBAC).  
3. Kubelet writes a new `kubelet-client-*.pem` and updates the `current` symlink.

**Failure mode:** node or API was down during the rotation window → client cert expires → node cannot authenticate → must repair manually (§7).

### 4.3 Kubelet serving cert rotation

If `serverTLSBootstrap: true`, serving certs also go through CSR approval. Approve pending CSRs if your policy requires manual approval:

```bash
kubectl get csr
kubectl certificate approve <csr-name>
```

---

## 5. Manual renewal — control plane (production procedure)

Run during a maintenance window. **Backup first.** Renew **one CP at a time** in HA.

### 5.1 Backup

```bash
NODE=$(hostname)
TS=$(date +%F-%H%M)
sudo tar czf /root/k8s-pki-backup-${NODE}-${TS}.tgz -C /etc/kubernetes pki admin.conf controller-manager.conf scheduler.conf
# Also take an etcd snapshot (stacked or external)
```

Store the tarball off-box.

### 5.2 Renew all leaves on this CP

```bash
sudo kubeadm certs renew all
sudo kubeadm certs check-expiration
```

Renew a single cert if needed:

```bash
sudo kubeadm certs renew apiserver
sudo kubeadm certs renew admin.conf
sudo kubeadm certs renew apiserver-etcd-client
# see: kubeadm certs renew --help
```

### 5.3 Restart static control-plane pods

Renewal writes new files; processes must reload. On kubeadm, move manifests out/in (or restart the node carefully):

```bash
# Causes kubelet to stop then recreate static pods
sudo mkdir -p /tmp/k8s-manifests-backup
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/k8s-manifests-backup/ 2>/dev/null || true
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/k8s-manifests-backup/
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/k8s-manifests-backup/
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/k8s-manifests-backup/

sleep 20
sudo mv /tmp/k8s-manifests-backup/*.yaml /etc/kubernetes/manifests/

# Wait until healthy
kubectl get pods -n kube-system -o wide
sudo crictl ps | grep -E 'kube-apiserver|etcd|scheduler|controller'
```

For **external etcd**, skip moving `etcd.yaml` (it may not exist); only restart API server / controller / scheduler manifests. Ensure `apiserver-etcd-client` matches what external etcd trusts.

### 5.4 Refresh local kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u):$(id -g)" $HOME/.kube/config
kubectl get nodes
```

Copy updated `admin.conf` to bastion/CI secrets stores as needed.

### 5.5 Repeat on other control planes

Same backup → `kubeadm certs renew all` → restart static pods → verify.  
Do not renew all CPs simultaneously in a way that drops API quorum.

### 5.6 Verify cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl get --raw='/readyz?verbose' | head
sudo kubeadm certs check-expiration
```

---

## 6. External etcd certificates

If etcd runs outside kubeadm static pods:

1. Renew etcd **server/peer/client** certs with your PKI process (cfssl/openssl/step-ca).  
2. Reload/restart etcd members rolling (maintain quorum).  
3. Renew `/etc/kubernetes/pki/apiserver-etcd-client.crt` (+ key) on **every** control plane (or run `kubeadm certs renew apiserver-etcd-client` if that pair is kubeadm-issued against the same CA).  
4. Restart kube-apiserver static pods so they pick up the client cert.  

Keep etcd CA and Kubernetes CA lifecycles documented separately.

---

## 7. Emergency: expired certificates

### 7.1 Control-plane leaves expired

If the API is already down, use **local** kubeadm on a CP (still works with local PKI files):

```bash
sudo kubeadm certs renew all
# restart static pods as in §5.3
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u):$(id -g)" $HOME/.kube/config
kubectl get nodes
```

If CA itself is expired, leaf renewal is not enough—you need **CA rotation / cluster recovery** from backup (out of scope for routine renewal; treat as DR).

### 7.2 Kubelet client cert expired

Symptoms: node `NotReady`; kubelet logs `x509` / `unauthorized` for `system:node`.

With API healthy again (or from a working admin host):

```bash
# On the broken node — back up, then clear expired client certs and re-bootstrap
sudo mv /var/lib/kubelet/pki /var/lib/kubelet/pki.bak-$(date +%F)
sudo mkdir -p /var/lib/kubelet/pki

# Ensure kubelet.conf uses bootstrap mechanism expected by your kubeadm version.
# If kubelet.conf embeds expired long-lived client certs (older clusters), regenerate:
sudo kubeadm init phase kubeconfig kubelet --config /etc/kubernetes/kubeadm-config.yaml
# or copy a working kubelet.conf pattern from kubeadm docs for your version

sudo systemctl restart kubelet
# Watch CSR approval and new kubelet-client-current.pem
kubectl get csr
sudo ls -l /var/lib/kubelet/pki/
journalctl -u kubelet -e --no-pager | tail -n 50
```

Exact recovery differs slightly by Kubernetes version; follow [kubeadm certs – kubelet client certificate rotation fails](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-client-certificate) for your release.

---

## 8. CA rotation (planning only)

Rotating `ca.crt` / `ca.key` means re-issuing **all** trust material: every leaf, every kubeconfig, every node kubelet trust, sometimes service-account keys if you change those too. In production:

1. Schedule a dedicated project (not a Friday hotfix).  
2. Prefer [manual CA rotation guides](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#signing-your-own-certificates) / kubeadm alpha phases documented for your version.  
3. Full etcd + PKI backup; staged soak in non-prod.  
4. Update all trust bundles (aggregated API, front-proxy, etcd).  

Leaf renewal (§5) is the **routine** yearly task; CA rotation is **exceptional**.

---

## 9. Automation patterns

| Approach | Pros | Cons |
|----------|------|------|
| Alert + manual runbook (§5) | Simple, controlled | Needs human window |
| Renew during planned upgrades | Fits release train | Gaps if upgrades slip past 1 year |
| Cron/Ansible on each CP | Predictable | Must serialize HA; restart pods carefully |
| Long-lived custom `--certificate-validity-period` at init | Fewer renewals | Longer compromise window; still need CA plan |

kubeadm init flag (set only at cluster creation / advanced recreate flows):

```bash
# Example concept — validity in days for leaf certs when generated
kubeadm init --config=...   # certificateValidityPeriod in ClusterConfiguration (version-dependent)
```

Check your kubeadm API (`ClusterConfiguration` certificate duration fields) before relying on longer lifetimes; many teams keep 1-year leaves and automate renewal instead.

**Safe automation checklist**

1. Fail if residual time already &lt; 7 days without human approval (possible outage path).  
2. Backup PKI → renew one CP → health check → next CP.  
3. Sync renewed `admin.conf` to secret storage.  
4. Page on failure; never ignore half-renewed HA.

---

## 10. Quick runbook

```text
Healthy cluster, certs within 90 days of expiry
  → backup /etc/kubernetes/pki (+ etcd snapshot)
  → for each control plane (one at a time):
       kubeadm certs renew all
       restart static pods (move manifests)
       kubectl get nodes
  → refresh bastion kubeconfigs
  → kubeadm certs check-expiration (all CPs green)

Monthly
  → check-expiration + kubelet-client dates sample
  → alert if < 30–60 days

Expired already
  → renew all on CP locally → restart static pods
  → fix kubelet client certs on NotReady nodes (§7.2)

External etcd
  → renew etcd PKI + apiserver-etcd-client → restart API

CA near expiry
  → start dedicated CA rotation project (not leaf renew alone)
```

---

## 11. Verification matrix

| Check | Command / expectation |
|-------|------------------------|
| Leaf residual time | `kubeadm certs check-expiration` → months left |
| API healthy | `kubectl get nodes`, `/readyz` |
| Static pods up | `kubectl -n kube-system get pods` |
| admin kubeconfig | `kubectl auth can-i get nodes` |
| Kubelet client | `openssl x509 … kubelet-client-current.pem` dates OK |
| etcd (stacked) | etcd pods Ready; `etcdctl endpoint health` |
| No pending bad CSRs | `kubectl get csr` |

---

## References

- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [kubeadm certs CLI](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
- [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [Kubelet serving certificate rotation](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubelet-tls-bootstrapping/)
- [Upgrading kubeadm (certs renew on upgrade)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
