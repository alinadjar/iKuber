# kubeadm Cluster Upgrade & Version Skew Policy

Step-by-step minor/patch upgrades for clusters created with **kubeadm**, plus the Kubernetes **version skew policy** in detail.

Related:

- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)
- [cni-calico.md](../cni-calico.md) (upgrade Calico with the cluster)
- [cni-cilium.md](../cni-cilium.md) (upgrade Cilium with the cluster)
- [external-etcd-tarball-systemd.md](../external-etcd-tarball-systemd.md)

Official sources: [Version skew policy](https://kubernetes.io/releases/version-skew-policy/) · [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

---

## 0. Core rules (read first)

1. **Do not skip minor versions.** Upgrade `1.31 → 1.32 → 1.33`, never `1.31 → 1.33` in one jump.
2. **Patch first, then minor.** On the current minor, move to the latest patch; then upgrade to the latest patch of the next minor.
3. **Control plane before workers.** Always finish (or sufficiently progress) API servers before rolling kubelets hard onto the new minor.
4. **One control-plane node at a time.** Keep HA quorum.
5. **kubeadm drives static-pod upgrades;** you still upgrade **kubelet/kubectl packages** (or binaries) on each node and restart kubelet.
6. **CNI / CSI / ingress** have their own compatibility matrices—plan those upgrades around the Kubernetes bump (see Calico/Cilium sections below and linked docs).

Example target used below: **`1.31.x → 1.32.y`**. Replace with your versions.

```text
FROM_MINOR=1.31
TO_MINOR=1.32
TO_PATCH=1.32.2          # pick a real current patch
```

---

## 1. Version skew policy (detailed)

Kubernetes versions look like `x.y.z` (major.minor.patch). Skew is measured in **minor** versions unless noted. The **kube-apiserver** is the anchor everything else is compared against.

### 1.1 Supported project releases

- The project maintains patch branches for the **three most recent minors** (example at docs time: 1.36 / 1.35 / 1.34—check [kubernetes.io/releases](https://kubernetes.io/releases/) for current).
- ~1 year of patch support per minor (1.19+).
- Running an unsupported minor means no security backports—upgrade before you fall off the window.

### 1.2 Component skew limits

| Component | Rule vs kube-apiserver | Notes |
|-----------|------------------------|--------|
| **kube-apiserver** (HA) | Newest and oldest CP API servers within **1 minor** | During rolling upgrade only briefly mixed |
| **kube-controller-manager** | Not newer than API server; at most **1 minor older** | Expected to match API minor after upgrade |
| **kube-scheduler** | Same as controller-manager | |
| **cloud-controller-manager** | Same as controller-manager | If used |
| **kubelet** | **Never newer** than API server; may be up to **3 minors older** | Pre-1.25 kubelets: max **2** older |
| **kube-proxy** | **Never newer** than API server; may be up to **3 minors older** | Same 2-minor historical caveat; also limited skew vs local kubelet |
| **kubectl** | Within **±1 minor** of kube-apiserver | Admin laptop must stay close |

#### kubelet examples (API server = 1.36)

Supported kubelets: **1.36, 1.35, 1.34, 1.33**.  
Not supported: **1.37** (newer than API) or **1.32** (four minors older).

#### HA narrows the window

If API servers are temporarily mixed **1.36 and 1.35**:

- kubelet/kube-proxy cannot be **1.36** (would be newer than the 1.35 API server).
- Supported kubelets become **1.35, 1.34, 1.33**.

So during a rolling CP upgrade, **do not** jump workers to the new minor until **all** API servers are on that minor (kubeadm’s flow already assumes CP first).

#### Persistent max skew warning

If workers sit permanently **three** minors behind the API server, you **must** upgrade those kubelets **before** the next control-plane minor bump—or you will violate skew.

### 1.3 kubectl skew

| API server | Supported kubectl |
|------------|-------------------|
| 1.36 | 1.37, 1.36, 1.35 |
| Mixed 1.36 + 1.35 HA | effectively 1.36 and 1.35 only |

Keep bastion `kubectl` within one minor of the cluster.

### 1.4 kubeadm-specific skew

kubeadm adds practical constraints on top of the platform policy:

| Pair | Typical expectation |
|------|---------------------|
| kubeadm ↔ kubelet | Prefer **same version**; kubelet may be older within supported skew |
| kubeadm CLI used for upgrade | Install the **target** kubeadm version on the node before `upgrade plan/apply` |
| Minor jumps | kubeadm **does not support skipping minors** |

See also [kubeadm skew vs kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy).

### 1.5 Required upgrade order (platform view)

To go **1.35 → 1.36** (generalized):

```text
1. Latest patch on current minor (everywhere practical)
2. kube-apiserver(s)          → new minor (one at a time in HA)
3. controller-manager / scheduler / cloud-controller-manager → new minor
4. kubelet / kube-proxy       → new minor (optional immediately; must stay within skew)
```

kubeadm bundles steps 2–3 via `kubeadm upgrade apply` / `upgrade node` on control planes (static pods + addons), then you upgrade kubelet packages and restart.

### 1.6 What “IgnoredDuringExecution” does *not* change

Skew policy is about **component binaries/APIs**, not pod affinity. Still drain nodes on **minor** kubelet upgrades—in-place minor kubelet upgrades are **not** supported; drain first.

---

## 2. Pre-upgrade checklist

Do this in staging first when possible.

### 2.1 Inventory

```bash
kubectl get nodes -o wide
kubectl version
kubeadm version
kubectl get pods -A
kubectl get cs 2>/dev/null || true
kubectl -n kube-system get pods
```

Record CNI (Calico/Cilium), CSI, ingress, MetalLB, metrics-server versions.

### 2.2 Health & capacity

- [ ] All nodes `Ready`; system pods healthy  
- [ ] etcd healthy (stacked pods or external cluster)  
- [ ] Enough spare capacity to **drain one node** (or one failure domain)  
- [ ] PDBs won’t block drains forever (`kubectl get pdb -A`)  
- [ ] Maintenance window / communication done  

### 2.3 Backups

```bash
# Stacked etcd example (control plane) — or use your external etcd backup job
sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/pre-upgrade-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Also back up:

- `/etc/kubernetes/` (especially `admin.conf`, manifests, pki) on each CP  
- kubeadm ClusterConfiguration (`kubectl -n kube-system get cm kubeadm-config -o yaml`)  
- App/GitOps state; Velero if you use it  

### 2.4 Deprecated APIs

```bash
# Optional: kube-no-trouble / Pluto / similar
kubent
```

Fix manifests still using APIs removed in the **target** minor before upgrading.

### 2.5 CNI compatibility

Confirm target Calico/Cilium supports the **target Kubernetes** minor. If the CNI must bump first (or together), schedule that (see §7 and the CNI docs).

### 2.6 External etcd

If etcd is external, kubeadm will not upgrade etcd for you. Keep etcd on a version supported by the target Kubernetes; upgrade etcd in its own window if required ([configure/upgrade etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)).

---

## 3. Patch upgrade (same minor)

Example: `1.32.1 → 1.32.2`. Safer; still roll node by node for kubelet.

On **first control plane**:

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.2-*
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.2
```

Upgrade kubelet/kubectl on that node:

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.32.2-* kubectl=1.32.2-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Other control planes:

```bash
# install matching kubeadm, then:
sudo kubeadm upgrade node
# then kubelet/kubectl packages + restart kubelet
```

Workers: drain → kubeadm package → `kubeadm upgrade node` → kubelet/kubectl → uncordon (same pattern as §6).

---

## 4. Minor upgrade — control plane (step by step)

Example: all components currently on **1.31.x**; target **1.32.y**.

### 4.1 Point apt at the target minor repo

On nodes that use pkgs.k8s.io, the list file is **per minor**. Switching `1.31` → `1.32`:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

Binary installs: download matching `kubeadm`/`kubelet`/`kubectl` from `dl.k8s.io` instead.

### 4.2 First control-plane node

Pick a CP that has `/etc/kubernetes/admin.conf`.

```bash
# 1) Upgrade kubeadm only first
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.32.2-*
sudo apt-mark hold kubeadm
kubeadm version

# 2) See what will change (CoreDNS, kube-proxy, etcd image, etc.)
sudo kubeadm upgrade plan

# 3) Apply (stacked etcd + control plane static pods + addons)
sudo kubeadm upgrade apply v1.32.2
```

If the node runs workloads (unusual for tainted CP, but possible):

```bash
kubectl drain <cp-node> --ignore-daemonsets --delete-emptydir-data
```

Upgrade kubelet/kubectl and restart:

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.32.2-* kubectl=1.32.2-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <cp-node>   # if drained
```

Verify:

```bash
kubectl get nodes
kubectl get pods -n kube-system -o wide
kubectl version
```

### 4.3 Additional control-plane nodes

**One at a time:**

```bash
# Same apt repo + install kubeadm=1.32.2-*
sudo kubeadm upgrade node

# drain if needed
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.32.2-* kubectl=1.32.2-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <cp-node>
```

After all CPs:

- All API servers on **1.32.y**  
- `kubectl get nodes` shows control planes Ready  
- No crash-looping `kube-apiserver` / `etcd` / `scheduler` / `controller-manager`  

---

## 5. CNI pause point (recommended)

Before or while workers roll, ensure the CNI supports **1.32**:

| CNI | Action |
|-----|--------|
| Calico | Upgrade Tigera Operator / Calico per [cni-calico.md](../cni-calico.md) § upgrade |
| Cilium | `cilium upgrade` / Helm upgrade per [cni-cilium.md](../cni-cilium.md) |
| Others | Follow vendor matrix |

Many teams: **upgrade CNI to a K8s-compatible release → finish worker kubelet roll**.

---

## 6. Minor upgrade — worker nodes

Do **one worker (or a small surge group) at a time**.

```bash
# From an admin host
kubectl cordon <worker>
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data
```

On the worker:

```bash
# apt repo already on v1.32
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.2-*
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.32.2-* kubectl=1.32.2-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Back on admin host:

```bash
kubectl uncordon <worker>
kubectl get node <worker>
```

Repeat until all workers match. Confirm:

```bash
kubectl get nodes -o wide
# VERSION column should show v1.32.y everywhere (or acceptable skew during roll)
```

---

## 7. Upgrading Calico & Cilium with the cluster

### 7.1 Calico (summary)

1. Check [Calico ↔ Kubernetes support](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements).  
2. Upgrade **operator** manifest to the target Calico version.  
3. Let the operator reconcile `calico-system`.  
4. Verify `TigeraStatus` / dataplane; run a connectivity smoke test.  

Details and commands: **[cni-calico.md](../cni-calico.md)** (upgrade section).

### 7.2 Cilium (summary)

1. Check Cilium’s Kubernetes compatibility for the target version.  
2. `cilium upgrade --version …` or Helm `upgrade --reuse-values`.  
3. If using **kube-proxy replacement**, ensure kubeadm upgrades do **not** recreate kube-proxy (`skipPhases` / delete kube-proxy ConfigMap as documented).  
4. `cilium status --wait` + connectivity test.  

Details: **[cni-cilium.md](../cni-cilium.md)**.

### 7.3 MetalLB / ingress / CSI

Upgrade per their release notes after the API server is on the new minor (or per their tested matrix). IP pools and BGP peers usually survive MetalLB chart/manifest upgrades if CRDs remain compatible.

---

## 8. Post-upgrade verification

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get --raw='/readyz?verbose' | head
kubeadm certs check-expiration

# Workload smoke
kubectl create deploy upgrade-smoke --image=nginx:stable --replicas=2
kubectl expose deploy upgrade-smoke --port=80
kubectl run curl --rm -it --restart=Never --image=curlimages/curl -- curl -sS -o /dev/null -w "%{http_code}\n" http://upgrade-smoke
kubectl delete deploy upgrade-smoke svc upgrade-smoke

# CNI
# Calico: kubectl get tigerastatus ; kubectl -n calico-system get pods
# Cilium: cilium status --wait
```

Check dashboards/alerts, DNS, ingress, storage attach/detach, and certificate expiration.

---

## 9. Multi-step upgrades (behind by several minors)

If the cluster is on **1.29** and you need **1.32**:

```text
1.29.latest-patch → 1.30.latest-patch → 1.31.latest-patch → 1.32.latest-patch
```

At each step repeat: CP → (CNI if required) → workers → verify.  
Do **not** leave kubelets three minors behind and then attempt another CP minor bump without rolling workers.

Suggested cadence for large fleets:

1. CP minor N+1  
2. Critical add-ons (CNI/CSI)  
3. Workers to N+1  
4. Soak  
5. Next minor  

---

## 10. Failure / rollback notes

- kubeadm **does not** support downgrading a cluster in place.  
- Prefer **etcd snapshot restore** + control-plane recovery from pre-upgrade backup if an upgrade fails badly.  
- If `kubeadm upgrade apply` fails mid-way, read the error, fix mirrors/disk/CNI, re-run `upgrade plan` / `apply`; avoid manual partial static-pod edits unless you know the state.  
- Keep the previous package minor repo URL available until the roll finishes so you can reinstall matching kubelet builds if needed on a bad node (still not a full cluster downgrade).

---

## 11. Quick flow

```text
Backup etcd + /etc/kubernetes
  → latest patch on current minor
  → switch apt/binary to NEXT minor
  → CP1: kubeadm install → upgrade plan → upgrade apply → kubelet restart
  → CP2..n: upgrade node → kubelet restart
  → upgrade CNI (Calico/Cilium) if required for new K8s
  → workers: drain → kubeadm → upgrade node → kubelet → uncordon
  → verify nodes, system pods, CNI, smoke test
  → (repeat for each minor if jumping multiple)
```

---

## References

- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Upgrading Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [API deprecation guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
