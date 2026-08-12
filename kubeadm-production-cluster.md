# Production Kubernetes Cluster with kubeadm

Step-by-step guide to bootstrap a production-ready Kubernetes cluster using **kubeadm**.

> **Assumptions**
>
> - Ubuntu 22.04/24.04 LTS (or compatible Debian)
> - At least **3 control-plane** nodes + **2+ worker** nodes (HA)
> - Static IPs, working DNS, and outbound internet (or a private package mirror)
> - Root or `sudo` on all nodes

---

## 0. Architecture & planning

### Recommended topology

| Role           | Count | Notes                                      |
|----------------|-------|--------------------------------------------|
| Control plane  | 3     | Odd number for etcd quorum (if stacked)    |
| Workers        | 2+    | Scale to workload needs                    |
| Load balancer  | 1     | VIP / HAProxy / cloud LB for API server    |
| etcd (optional external) | 3 | Prefer dedicated nodes; see [external-etcd-tarball-systemd.md](./external-etcd-tarball-systemd.md) |

### Decisions before you start

- [ ] Kubernetes version (pin it; e.g. `v1.31`)
- [ ] Pod CIDR (e.g. `10.244.0.0/16` for Flannel, or CNI-specific)
- [ ] Service CIDR (e.g. `10.96.0.0/12`)
- [ ] Control-plane endpoint (DNS name → load balancer VIP)
- [ ] Container runtime: **containerd** (recommended)
- [ ] CNI plugin: Cilium / Calico / Flannel
- [ ] Storage: CSI driver for your platform
- [ ] Ingress controller (optional but typical)

Example values used below:

```text
CONTROL_PLANE_ENDPOINT = k8s-api.example.com:6443
POD_CIDR               = 10.244.0.0/16
SERVICE_CIDR           = 10.96.0.0/12
K8S_VERSION            = 1.31.0
```

---

## 1. Prepare every node (control plane + workers)

Run on **all** nodes.

### 1.1 Hostname, time, and packages

```bash
# Set a unique hostname
sudo hostnamectl set-hostname <node-name>

# Sync time (critical for certificates and etcd)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg \
  software-properties-common jq chrony

sudo systemctl enable --now chrony
timedatectl status
```

Ensure `/etc/hosts` (or DNS) resolves all node names and the API endpoint.

### 1.2 Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 1.3 Kernel modules and sysctl

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 1.4 Firewall / security groups (minimum)

| Direction | Port(s)        | Purpose                         | Who               |
|-----------|----------------|---------------------------------|-------------------|
| TCP       | 6443           | Kubernetes API                  | LB → control plane |
| TCP       | 2379–2380      | etcd                            | control plane     |
| TCP       | 10250          | kubelet API                     | cluster nodes     |
| TCP       | 10257, 10259   | scheduler / controller-manager  | control plane     |
| UDP/TCP   | CNI-specific   | pod networking                  | all nodes         |
| TCP       | 30000–32767    | NodePort (if used)              | external as needed|

---

## 2. Install containerd (all nodes)

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Use systemd cgroup driver (required for kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

Optional: configure a private registry mirror in `/etc/containerd/config.toml` if nodes cannot pull from the public internet.

---

## 3. Install kubeadm, kubelet, kubectl (all nodes)

Choose **one** path: package install (3A) or downloaded binaries (3B).

### 3A. Package install (apt)

Pin versions so every node matches.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.31.0-* kubeadm=1.31.0-* kubectl=1.31.0-*
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

Verify:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

### 3B. Manual binaries + systemd (no packages)

Use this when you already downloaded `kubelet` / `kubeadm` / `kubectl` and must wire systemd yourself.

Packages normally give you:

| What | Path | Who creates it |
|------|------|----------------|
| kubelet binary | `/usr/bin/kubelet` | you / package |
| kubeadm binary | `/usr/bin/kubeadm` | you / package |
| kubectl binary | `/usr/bin/kubectl` | you / package |
| base unit | `/usr/lib/systemd/system/kubelet.service` | **you** (manual) |
| kubeadm drop-in | `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` | **you** (manual) |
| kubelet config | `/var/lib/kubelet/config.yaml` | **kubeadm init/join** |
| kubeadm flags | `/var/lib/kubelet/kubeadm-flags.env` | **kubeadm init/join** |
| bootstrap kubeconfig | `/etc/kubernetes/bootstrap-kubelet.conf` | **kubeadm** (temporary) |
| kubelet kubeconfig | `/etc/kubernetes/kubelet.conf` | **kubeadm** |
| extra args (optional) | `/etc/sysconfig/kubelet` or `/etc/default/kubelet` | **you** |

You only create the **binary placement + systemd unit + drop-in** before `kubeadm init`/`join`. Do **not** hand-write `config.yaml` or `kubeadm-flags.env` unless you know you need overrides—kubeadm generates them.

#### 3B.1 Place binaries

```bash
# Example: binaries already in ~/k8s-bins matching your cluster version
sudo install -m 0755 ~/k8s-bins/kubelet /usr/bin/kubelet
sudo install -m 0755 ~/k8s-bins/kubeadm /usr/bin/kubeadm
sudo install -m 0755 ~/k8s-bins/kubectl /usr/bin/kubectl

# Or download from dl.k8s.io:
# VER=v1.31.0
# ARCH=amd64
# curl -fsSLo kubelet https://dl.k8s.io/release/${VER}/bin/linux/${ARCH}/kubelet
# curl -fsSLo kubeadm https://dl.k8s.io/release/${VER}/bin/linux/${ARCH}/kubeadm
# curl -fsSLo kubectl https://dl.k8s.io/release/${VER}/bin/linux/${ARCH}/kubectl
# sudo install -m 0755 kubelet kubeadm kubectl /usr/bin/

kubelet --version
kubeadm version
kubectl version --client
```

Also install CNI plugins and `crictl` if not already present (packages usually do this):

```bash
# CNI plugins → /opt/cni/bin
# crictl → /usr/local/bin/crictl  (from cri-tools release matching your minor version)
sudo mkdir -p /opt/cni/bin
```

#### 3B.2 Base systemd unit: `kubelet.service`

Create `/usr/lib/systemd/system/kubelet.service` (or `/etc/systemd/system/kubelet.service`):

```ini
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo tee /usr/lib/systemd/system/kubelet.service >/dev/null <<'EOF'
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

If the binary is not in `/usr/bin`, change every `ExecStart=` path to match (unit **and** drop-in below).

#### 3B.3 kubeadm drop-in: `10-kubeadm.conf`

This is required for `kubeadm init` / `join`. Without it, kubelet starts with no kubeconfig/config and crash-loops until kubeadm writes files—or never picks them up correctly.

```bash
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d

sudo tee /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf >/dev/null <<'EOF'
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# Populated by kubeadm init / join at runtime
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# Optional user overrides (create the file yourself if needed)
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
EOF
```

On Debian/Ubuntu you may prefer `/etc/default/kubelet` instead of `/etc/sysconfig/kubelet`. Either works if the drop-in’s `EnvironmentFile=` points at it. Example optional overrides file:

```bash
# Debian/Ubuntu style
sudo tee /etc/default/kubelet >/dev/null <<'EOF'
KUBELET_EXTRA_ARGS=
EOF

# RHEL/Fedora style (matches upstream drop-in default)
sudo mkdir -p /etc/sysconfig
sudo tee /etc/sysconfig/kubelet >/dev/null <<'EOF'
KUBELET_EXTRA_ARGS=
EOF
```

Prefer configuring kubelet via the kubeadm config (`KubeletConfiguration` / `nodeRegistration.kubeletExtraArgs`) over stuffing flags into `KUBELET_EXTRA_ARGS`.

#### 3B.4 Directories kubeadm / kubelet expect

```bash
sudo mkdir -p /etc/kubernetes/manifests \
  /var/lib/kubelet \
  /var/lib/kubelet/pki \
  /etc/cni/net.d \
  /opt/cni/bin
```

#### 3B.5 Enable kubelet (expected crash-loop before init/join)

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kubelet
# kubelet will restart until kubeadm writes configs — that is normal
systemctl status kubelet --no-pager || true
journalctl -u kubelet -e --no-pager | tail -n 50
```

Then continue with load balancer + `kubeadm init` / `join` as usual. After init/join, kubeadm writes:

```text
/var/lib/kubelet/config.yaml          # KubeletConfiguration
/var/lib/kubelet/kubeadm-flags.env    # e.g. KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock ..."
/etc/kubernetes/kubelet.conf          # node kubeconfig
/etc/kubernetes/bootstrap-kubelet.conf  # used during bootstrap, then often removed
```

Example of what `kubeadm-flags.env` looks like after join (generated—do not invent casually):

```bash
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.10"
```

Inspect the effective unit after everything is up:

```bash
systemctl cat kubelet
systemctl show kubelet -p Environment -p EnvironmentFiles -p ExecStart
ls -l /var/lib/kubelet/config.yaml /var/lib/kubelet/kubeadm-flags.env
```

#### 3B.6 Download upstream templates instead of pasting

Official templates (paths assume `/usr/bin`):

```bash
# Pin RELEASE_VERSION to a known kubernetes/release tag if you need reproducibility
RELEASE_VERSION=v0.17.0
curl -fsSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" \
  | sudo tee /usr/lib/systemd/system/kubelet.service

sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d
curl -fsSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" \
  | sudo tee /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl daemon-reload
sudo systemctl enable --now kubelet
```

If binaries live elsewhere (e.g. `/usr/local/bin`), rewrite paths:

```bash
sed -i 's:/usr/bin:/usr/local/bin:g' \
  /usr/lib/systemd/system/kubelet.service \
  /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

---

## 4. Control-plane load balancer

Production clusters must not point clients at a single control-plane IP.

1. Create a TCP load balancer (or HAProxy/keepalived VIP) on port **6443**.
2. Backend targets: all control-plane nodes `:6443`.
3. Health-check the API (HTTPS or TCP).
4. Point DNS `k8s-api.example.com` at the VIP.

Test from a node:

```bash
nc -vz k8s-api.example.com 6443
```

---

## 5. Initialize the first control plane

Run on **control-plane-1 only**.

### 5.1 Create a kubeadm config (recommended)

```bash
cat <<EOF | tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.0
controlPlaneEndpoint: "k8s-api.example.com:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

> If your kubeadm version expects `v1beta3`, change the `apiVersion` accordingly (`kubeadm config print init-defaults` shows the correct schema).

### 5.2 Pull images and init

```bash
sudo kubeadm config images pull --config kubeadm-config.yaml
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

Save the printed:

- `kubeadm join ... --control-plane --certificate-key ...`
- `kubeadm join ...` (worker)

### 5.3 Configure kubectl for the admin user

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u):$(id -g)" $HOME/.kube/config

kubectl get nodes
kubectl get pods -A
```

At this point the node is `NotReady` until a CNI is installed.

---

## 6. Install a CNI plugin

Pick **one** CNI. Full guides:

- [Calico](./cni-calico.md) — Tigera Operator, BGP / IPIP / VXLAN, NetworkPolicy
- [Cilium](./cni-cilium.md) — eBPF, optional kube-proxy replacement, Hubble

### Option A — Flannel (minimal)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Ensure `podSubnet` matches Flannel’s network (default `10.244.0.0/16`).

### Option B — Calico

See [cni-calico.md](./cni-calico.md). Short path:

```bash
export CALICO_VERSION=v3.29.1
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml
# Edit custom-resources so ipPools.cidr matches networking.podSubnet, then:
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml
```

### Option C — Cilium

See [cni-cilium.md](./cni-cilium.md). Short path:

```bash
# Install Cilium CLI, then:
cilium install --version <cilium-version>
cilium status --wait
```

Wait until the control-plane node is `Ready`:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 7. Join additional control-plane nodes

On **control-plane-2** and **control-plane-3**:

```bash
# If join tokens expired, regenerate from control-plane-1:
# kubeadm token create --print-join-command
# kubeadm init phase upload-certs --upload-certs

sudo kubeadm join k8s-api.example.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

Verify:

```bash
kubectl get nodes
kubectl -n kube-system get pods | grep etcd
```

You should see 3 etcd members and 3 control-plane nodes.

---

## 8. Join worker nodes

On each worker:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Label workers (optional but useful):

```bash
kubectl label node <worker-name> node-role.kubernetes.io/worker=
```

---

## 9. Production post-install checklist

### 9.1 Smoke test

```bash
kubectl create deployment nginx --image=nginx:stable --replicas=3
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl run curl --rm -it --restart=Never --image=curlimages/curl -- curl -sS http://nginx
kubectl delete deployment nginx service nginx
```

### 9.2 Metrics Server (for HPA / `kubectl top`)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# If needed in lab/TLS-edge environments, patch args for insecure TLS to kubelets carefully.
kubectl -n kube-system get pods -l k8s-app=metrics-server
```

### 9.3 Core add-ons

- [ ] CSI storage driver for your cloud/on-prem storage
- [ ] Ingress controller (e.g. Ingress-NGINX or Traefik)
- [ ] Cert-manager for TLS certificates
- [ ] Cluster autoscaler / machine provisioning (if applicable)
- [ ] Backup for etcd (Velero and/or scheduled `etcdctl snapshot`)

### 9.4 etcd backup (critical)

On a control-plane node (or via a CronJob with proper certs):

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/snapshot-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Store snapshots off-box and test restore periodically.

---

## 10. Admin shell & utility practices

Do this on **admin workstations / bastion hosts** (not required on every worker). Goal: safer, faster day-2 ops once the cluster is up.

### 10.1 Packages worth having

```bash
sudo apt-get update
sudo apt-get install -y bash-completion curl jq fzf tmux tree unzip

# Optional but high value:
# - yq          → YAML queries (snap/binary)
# - helm        → charts
# - k9s         → terminal UI
# - stern       → multi-pod logs (also via krew)
```

### 10.2 kubectl / kubeadm / helm bash completion

```bash
# Ensure programmable completion is loaded (Ubuntu usually has this)
grep -q bash_completion ~/.bashrc || \
  echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc

# kubectl
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl >/dev/null

# Short alias + completion for the alias
grep -q "alias k=" ~/.bashrc || cat >> ~/.bashrc <<'EOF'
alias k=kubectl
complete -o default -F __start_kubectl k
EOF

# kubeadm (on nodes where kubeadm is installed)
kubeadm completion bash | sudo tee /etc/bash_completion.d/kubeadm >/dev/null

# helm (if installed)
# helm completion bash | sudo tee /etc/bash_completion.d/helm >/dev/null

# crictl (node debugging)
# crictl completion bash | sudo tee /etc/bash_completion.d/crictl >/dev/null

source ~/.bashrc
```

For **zsh**:

```bash
# kubectl
kubectl completion zsh > "${fpath[1]}/_kubectl"
# or: source <(kubectl completion zsh)
echo 'alias k=kubectl' >> ~/.zshrc
echo 'compdef __start_kubectl k' >> ~/.zshrc
```

### 10.3 Useful aliases

```bash
cat >> ~/.bashrc <<'EOF'
# kubectl shortcuts
alias k='kubectl'
alias kx='kubectl exec -it'
alias kl='kubectl logs -f'
alias kgp='kubectl get pods -o wide'
alias kgpa='kubectl get pods -A -o wide'
alias kgn='kubectl get nodes -o wide'
alias kgs='kubectl get svc -A'
alias kgd='kubectl get deploy -A'
alias kdesc='kubectl describe'
alias kctx='kubectl config get-contexts'
alias kns='kubectl config view --minify -o jsonpath={..namespace}'

# Safer apply habit: dry-run first when unsure
alias kapplydry='kubectl apply --dry-run=client -o yaml'

# Prefer server-side apply for owned manifests (optional)
# alias kapply='kubectl apply --server-side -f'
EOF
source ~/.bashrc
```

Set a default editor for `kubectl edit`:

```bash
echo 'export EDITOR=vim' >> ~/.bashrc   # or nano / nvim
```

### 10.4 PS1: show cluster context + namespace

Install [kube-ps1](https://github.com/jonmosco/kube-ps1) so the prompt shows context/namespace (reduces wrong-cluster mistakes).

```bash
git clone --depth 1 https://github.com/jonmosco/kube-ps1.git ~/.kube-ps1

cat >> ~/.bashrc <<'EOF'
source ~/.kube-ps1/kube-ps1.sh
# Compact prompt: user@host  (ctx:ns)  path
PS1='\[\e[32m\]\u@\h\[\e[0m\] $(kube_ps1) \[\e[34m\]\w\[\e[0m\]\$ '
KUBE_PS1_SYMBOL_ENABLE=false
KUBE_PS1_NS_ENABLE=true
EOF
source ~/.bashrc
```

Toggle helpers:

```bash
# Temporarily hide / show kube info in prompt
kubeoff
kubeon
```

**Alternative:** starship with a Kubernetes module, or Oh My Zsh `kubectl` plugin — same idea (always visible context).

### 10.5 Krew + useful kubectl plugins

[Krew](https://krew.sigs.k8s.io/) is the plugin manager for kubectl.

```bash
(
  set -euo pipefail
  cd "$(mktemp -d)"
  OS="$(uname | tr '[:upper:]' '[:lower:]')"
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')"
  KREW="krew-${OS}_${ARCH}"
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz"
  tar zxvf "${KREW}.tar.gz"
  ./"${KREW}" install krew
)

echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
kubectl krew version
```

High-value plugins:

```bash
kubectl krew update
kubectl krew install \
  ctx \
  ns \
  neat \
  tree \
  stern \
  resource-capacity \
  view-secret \
  outdated \
  node-shell \
  sniff \
  access-matrix \
  score
```

| Plugin | Use |
|--------|-----|
| `ctx` | Switch kubeconfig context quickly (`kubectl ctx`) |
| `ns` | Switch default namespace (`kubectl ns`) |
| `neat` | Strip noisy fields from `kubectl get -o yaml` |
| `tree` | Ownership tree (Deploy → RS → Pod) |
| `stern` | Tail logs across many pods |
| `resource-capacity` | CPU/mem requests vs capacity overview |
| `view-secret` | Decode Secret data safely in terminal |
| `outdated` | Find images with newer tags |
| `node-shell` | Debug shell on a node via privileged pod |
| `sniff` | Packet capture on a pod (tcpdump/wireshark workflow) |
| `access-matrix` | Who can do what (RBAC overview) |
| `score` | Static analysis of manifests / live objects |

Everyday patterns:

```bash
kubectl ctx                         # pick cluster
kubectl ns prod                     # pick namespace
kubectl get deploy web -o yaml | kubectl neat
kubectl tree deploy web
kubectl stern web -n prod --since=10m
kubectl resource-capacity --pods --util
kubectl view-secret my-secret -a
```

Keep plugins current:

```bash
kubectl krew upgrade
kubectl krew list
```

> Prefer `ctx` / `ns` (or standalone `kubectx` / `kubens`) over memorizing long `kubectl config` commands. Always glance at the PS1 context before destructive applies.

### 10.6 kubeconfig hygiene

```bash
# Default location
export KUBECONFIG=$HOME/.kube/config

# Merge multiple clusters without overwriting
KUBECONFIG=~/.kube/config:~/.kube/cluster-b.conf kubectl config view --flatten > ~/.kube/merged.conf
mv ~/.kube/merged.conf ~/.kube/config
chmod 600 ~/.kube/config

# Never commit kubeconfigs; use short-lived tokens / OIDC where possible
# Separate personal admin creds from break-glass cluster-admin
```

Useful built-ins:

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config set-context --current --namespace=prod
kubectl api-resources | less
kubectl explain pod.spec.containers --recursive | less
```

### 10.7 Optional terminal tools

| Tool | Why |
|------|-----|
| [k9s](https://k9scli.io/) | Fast TUI for pods/logs/resources |
| [kubectx / kubens](https://github.com/ahmetb/kubectx) | Standalone alternative to krew `ctx`/`ns` |
| [helm](https://helm.sh/) + completion | Package installs (ingress, metrics extras, etc.) |
| [stern](https://github.com/stern/stern) | Multi-pod logs (CLI or krew) |
| [dive](https://github.com/wagoodman/dive) | Inspect image layers before deploy |
| [kubent](https://github.com/doitintl/kube-no-trouble) | Deprecated API detection before upgrades |
| `jq` / `yq` | Scriptable JSON/YAML |
| `fzf` | Fuzzy history / resource picking in custom scripts |

Example: install k9s (pick current release asset for your arch):

```bash
# See https://github.com/derailed/k9s/releases
# sudo install -m 0755 k9s /usr/local/bin/k9s
```

Helm quick setup:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm completion bash | sudo tee /etc/bash_completion.d/helm >/dev/null
helm repo add stable https://charts.helm.sh/stable 2>/dev/null || true
helm repo update
```

### 10.8 Safer kubectl habits

```bash
# Confirm context before prod changes
kubectl config current-context
kubectl cluster-info

# Client-side dry-run
kubectl apply -f app.yaml --dry-run=client -o yaml

# Server-side dry-run (admission-aware)
kubectl apply -f app.yaml --dry-run=server

# Diff against live objects
kubectl diff -f app.yaml

# Wait instead of blind sleep
kubectl rollout status deploy/web -n prod --timeout=120s

# Events when something is stuck
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -n 30
```

Drain pattern for node work:

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# ... maintenance ...
kubectl uncordon <node>
```

### 10.9 tmux / session tips (bastion)

```bash
# ~/.tmux.conf (minimal)
set -g mouse on
set -g history-limit 50000
setw -g mode-keys vi
```

Keep long upgrades / `journalctl -f` / `stern` in dedicated panes; name sessions per cluster (`tmux new -s prod-k8s`).

---

## 11. Hardening for production

### 11.1 Access control

```bash
# Prefer OIDC / cloud IAM / LDAP via API server flags or an auth proxy
# Create least-privilege Roles/ClusterRoles + RoleBindings
kubectl create namespace prod
# Avoid sharing admin.conf; issue short-lived kubeconfigs or use OIDC
```

### 11.2 Network policies

Install a CNI that supports NetworkPolicy (Calico/Cilium), then default-deny in sensitive namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### 11.3 Admission & policy

- Enable / configure Pod Security Admission (`restricted` for prod namespaces)
- Consider Gatekeeper or Kyverno for org policy
- Restrict privileged pods, hostPath, and hostNetwork

### 11.4 Secrets & supply chain

- Encrypt secrets at rest (`EncryptionConfiguration` for etcd)
- Use sealed-secrets / external secrets / cloud KMS
- Pin image digests; scan images in CI
- Restrict who can create privileged workloads

### 11.5 Node & OS

- Keep OS patched; reboot policy for kernel updates
- Separate OS disk vs container/etcd data where possible
- Disable unused services; restrict SSH to bastion / break-glass accounts
- Configure log shipping (kubelet, container runtime, audit logs)

### 11.6 Kubernetes audit logging

Configure API server audit policy and ship logs to a SIEM.

---

## 12. Useful operations

### Regenerate join command

```bash
kubeadm token create --print-join-command
```

### Upload new certificate key (for joining control planes)

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

### Upgrade cluster (high level)

1. Upgrade first control plane: `kubeadm upgrade plan` → `kubeadm upgrade apply`
2. Upgrade kubelet/kubectl packages on that node; restart kubelet
3. Drain/uncordon remaining control planes one by one
4. Drain/upgrade workers one by one
5. Always follow the [official version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)

### Reset a node (destructive)

```bash
sudo kubeadm reset -f
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo ipvsadm -C 2>/dev/null || true
sudo rm -rf /etc/cni/net.d
```

---

## 13. Verification matrix

| Check                         | Command / expectation                          |
|------------------------------|-------------------------------------------------|
| Nodes Ready                  | `kubectl get nodes` → all Ready                 |
| System pods healthy          | `kubectl get pods -A` → Running/Completed       |
| etcd quorum                  | 3 etcd pods; API stays up if 1 CP fails         |
| DNS                          | `kubectl -n kube-system get pods -l k8s-app=kube-dns` |
| CNI                          | Pod-to-pod connectivity across nodes            |
| API via LB                   | `kubectl` works through `controlPlaneEndpoint`  |
| Certificates                 | `kubeadm certs check-expiration`                |
| Backup                       | Successful etcd snapshot + restore drill        |

---

## Quick command map

```text
All nodes     → swap off, sysctl, containerd, kubeadm/kubelet/kubectl
LB            → TCP 6443 → control planes
CP1           → kubeadm init --config ... --upload-certs
Cluster       → install CNI
CP2/CP3       → kubeadm join --control-plane --certificate-key ...
Workers       → kubeadm join ...
Then          → metrics, CSI, ingress, backups, hardening
Admin host    → completion, kube-ps1, krew plugins, aliases, k9s/helm
```

---

## References

- [kubeadm Creating a cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [kubeadm HA topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Certificate management](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [kubectl autocomplete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_completion/)
- [Krew – kubectl plugin manager](https://krew.sigs.k8s.io/)
- [kube-ps1](https://github.com/jonmosco/kube-ps1)
