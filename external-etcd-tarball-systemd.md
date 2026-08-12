# External etcd Cluster from Tarball (systemd on Ubuntu) + kubeadm

Guide to install a **production-style external etcd cluster** from the official GitHub tarball, run it under **systemd** on Ubuntu, then point **`kubeadm init`** at it (external etcd topology).

Companion doc: [kubeadm-production-cluster.md](./kubeadm-production-cluster.md).

> **Why external etcd?**
>
> Stacked etcd (etcd as a static pod on each control plane) is simpler. External etcd separates the datastore from the control plane so you can scale/upgrade them independently and avoid etcd and API server sharing the same failure domain as tightly.

---

## 0. Topology & assumptions

| Role              | Count | Example hosts / IPs              |
|-------------------|-------|----------------------------------|
| etcd              | 3     | `etcd0` `10.0.0.10`, `etcd1` `10.0.0.11`, `etcd2` `10.0.0.12` |
| Control plane     | 3     | separate nodes (or same machine only for labs) |
| API load balancer | 1     | `k8s-api.example.com:6443`       |

**Assumptions**

- Ubuntu 22.04/24.04 LTS
- Static IPs and DNS (or `/etc/hosts`) for all etcd members
- Root or `sudo`
- Odd number of etcd members (3 or 5)
- etcd client port **2379**, peer port **2380** open between members; **2379** reachable from control-plane nodes

Example values used below:

```text
ETCD_VER     = v3.5.16
ETCD0_IP     = 10.0.0.10
ETCD1_IP     = 10.0.0.11
ETCD2_IP     = 10.0.0.12
ETCD0_NAME   = etcd0
ETCD1_NAME   = etcd1
ETCD2_NAME   = etcd2
```

Pick an etcd version compatible with your Kubernetes minor version (see [Kubernetes version skew / etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)).

---

## 1. Prepare every etcd node

```bash
sudo apt-get update
sudo apt-get install -y curl tar chrony

sudo hostnamectl set-hostname <etcd0|etcd1|etcd2>
sudo systemctl enable --now chrony

# Optional: map names if DNS is not ready
# sudo tee -a /etc/hosts <<EOF
# 10.0.0.10 etcd0
# 10.0.0.11 etcd1
# 10.0.0.12 etcd2
# EOF
```

Firewall (UFW example — adapt to your setup):

```bash
# Peer + client among etcd nodes; client from control planes
sudo ufw allow from 10.0.0.0/24 to any port 2379 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 2380 proto tcp
```

Create dedicated user and directories:

```bash
sudo useradd --system --home /var/lib/etcd --shell /usr/sbin/nologin etcd || true
sudo mkdir -p /etc/etcd/pki /var/lib/etcd /var/log/etcd
sudo chown -R etcd:etcd /var/lib/etcd /var/log/etcd
sudo chmod 700 /var/lib/etcd
```

---

## 2. Download and install the etcd tarball

Run on **each** etcd node (same version everywhere).

```bash
ETCD_VER=v3.5.16
ARCH=amd64   # or arm64
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download

curl -fsSL -o /tmp/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz \
  "${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz"

# Optional: verify checksum from the GitHub release page
# sha256sum /tmp/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz

sudo tar xzf /tmp/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz -C /tmp
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcd /usr/local/bin/etcd
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcdctl /usr/local/bin/etcdctl
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcdutl /usr/local/bin/etcdutl

etcd --version
etcdctl version
```

---

## 3. TLS certificates (required for production / kubeadm)

kubeadm expects the API server to talk to etcd over **HTTPS** with client certs. Generate a small PKI once, then distribute.

Recommended layout on etcd nodes:

```text
/etc/etcd/pki/
  ca.crt
  ca.key          # keep offline / on CA host only after distribute
  server.crt      # peer + client serving (per member)
  server.key
  peer.crt        # can be same as server if SANs cover both uses
  peer.key
```

On **control-plane** nodes (for kubeadm / kube-apiserver):

```text
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/apiserver-etcd-client.key
```

### 3.1 Generate CA + member + client certs (cfssl)

Install cfssl on a secure bootstrap host (can be `etcd0`):

```bash
CFSSL_VER=1.6.5
curl -fsSL -o /tmp/cfssl https://github.com/cloudflare/cfssl/releases/download/v${CFSSL_VER}/cfssl_${CFSSL_VER}_linux_amd64
curl -fsSL -o /tmp/cfssljson https://github.com/cloudflare/cfssl/releases/download/v${CFSSL_VER}/cfssljson_${CFSSL_VER}_linux_amd64
sudo install -m 0755 /tmp/cfssl /tmp/cfssljson /usr/local/bin/
```

Create CSRs:

```bash
mkdir -p ~/etcd-pki && cd ~/etcd-pki

cat > ca-csr.json <<'EOF'
{
  "CN": "etcd-ca",
  "key": { "algo": "rsa", "size": 2048 },
  "names": [ { "O": "etcd", "OU": "etcd-ca" } ]
}
EOF

cat > ca-config.json <<'EOF'
{
  "signing": {
    "default": { "expiry": "87600h" },
    "profiles": {
      "server": {
        "expiry": "87600h",
        "usages": ["signing", "key encipherment", "server auth", "client auth"]
      },
      "peer": {
        "expiry": "87600h",
        "usages": ["signing", "key encipherment", "server auth", "client auth"]
      },
      "client": {
        "expiry": "87600h",
        "usages": ["signing", "key encipherment", "client auth"]
      }
    }
  }
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Per-member server/peer cert (repeat for each IP/name; example for `etcd0`):

```bash
MEMBER_NAME=etcd0
MEMBER_IP=10.0.0.10

cat > ${MEMBER_NAME}.json <<EOF
{
  "CN": "${MEMBER_NAME}",
  "hosts": [
    "${MEMBER_NAME}",
    "${MEMBER_IP}",
    "localhost",
    "127.0.0.1"
  ],
  "key": { "algo": "rsa", "size": 2048 },
  "names": [ { "O": "etcd" } ]
}
EOF

cfssl gencert \
  -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server \
  ${MEMBER_NAME}.json | cfssljson -bare ${MEMBER_NAME}
```

API server client cert (used by kube-apiserver → etcd):

```bash
cat > apiserver-etcd-client.json <<'EOF'
{
  "CN": "kube-apiserver-etcd-client",
  "hosts": [],
  "key": { "algo": "rsa", "size": 2048 },
  "names": [ { "O": "system:masters" } ]
}
EOF

cfssl gencert \
  -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client \
  apiserver-etcd-client.json | cfssljson -bare apiserver-etcd-client
```

### 3.2 Install certs on each etcd node

```bash
# On etcd0 (adjust source filenames)
sudo install -m 0644 ca.pem /etc/etcd/pki/ca.crt
sudo install -m 0644 etcd0.pem /etc/etcd/pki/server.crt
sudo install -m 0600 etcd0-key.pem /etc/etcd/pki/server.key
# Using the same cert for peer is fine when SANs include the peer identity
sudo install -m 0644 etcd0.pem /etc/etcd/pki/peer.crt
sudo install -m 0600 etcd0-key.pem /etc/etcd/pki/peer.key
sudo chown -R etcd:etcd /etc/etcd/pki
sudo chmod 755 /etc/etcd /etc/etcd/pki
```

Repeat for `etcd1` / `etcd2` with their own `server`/`peer` certs and the **same** `ca.crt`.

### 3.3 Stage client certs for kubeadm (control planes)

On each control-plane node **before** `kubeadm init` / `join --control-plane`:

```bash
sudo mkdir -p /etc/kubernetes/pki/etcd
sudo install -m 0644 ca.pem /etc/kubernetes/pki/etcd/ca.crt
sudo install -m 0644 apiserver-etcd-client.pem /etc/kubernetes/pki/apiserver-etcd-client.crt
sudo install -m 0600 apiserver-etcd-client-key.pem /etc/kubernetes/pki/apiserver-etcd-client.key
```

> Do **not** place `ca.key` on every machine. Keep the CA key offline or on a secure PKI host.

---

## 4. etcd config file (per member)

Create `/etc/etcd/etcd.conf` on each node. Example for **etcd0** (`10.0.0.10`):

```bash
sudo tee /etc/etcd/etcd.conf >/dev/null <<'EOF'
# Member
ETCD_NAME="etcd0"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.0.10:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.0.10:2379,https://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.0.10:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.0.10:2380"

# Cluster
ETCD_INITIAL_CLUSTER="etcd0=https://10.0.0.10:2380,etcd1=https://10.0.0.11:2380,etcd2=https://10.0.0.12:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"

# TLS: client
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_CERT_FILE="/etc/etcd/pki/server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/server.key"

# TLS: peer
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"

# Ops
ETCD_AUTO_COMPACTION_RETENTION="8"
ETCD_QUOTA_BACKEND_BYTES="8589934592"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
EOF

sudo chown root:etcd /etc/etcd/etcd.conf
sudo chmod 640 /etc/etcd/etcd.conf
```

Change `ETCD_NAME`, listen/advertise URLs for **etcd1** and **etcd2**. Keep `ETCD_INITIAL_CLUSTER` identical on all three when bootstrapping a **new** cluster.

After the cluster is healthy, for future restarts leave `ETCD_INITIAL_CLUSTER_STATE=new` only for the first bootstrap. For rebuilt members joining an existing cluster use `existing` (and follow etcd member add procedures). Once the cluster has formed, many operators remove or ignore initial-cluster bootstrap flags and rely on the data directory; if you recreate a wiped member, follow [etcd runtime reconfiguration](https://etcd.io/docs/latest/op-guide/runtime-configuration/).

---

## 5. systemd unit

Create `/etc/systemd/system/etcd.service` on each etcd node:

```bash
sudo tee /etc/systemd/system/etcd.service >/dev/null <<'EOF'
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10
LimitNOFILE=40000
Nice=-10

# Harden a bit (optional; relax if it breaks your setup)
ProtectSystem=full
ProtectHome=true
NoNewPrivileges=true
PrivateTmp=true
ReadWritePaths=/var/lib/etcd /var/log/etcd

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start on **all three members** (close together so they can form the cluster):

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
sudo systemctl status etcd --no-pager
journalctl -u etcd -e --no-pager
```

---

## 6. Verify the etcd cluster

From any etcd node (using local client URL + client certs; you can reuse the apiserver client cert or issue an `etcdctl` client cert):

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379
export ETCDCTL_CACERT=/etc/etcd/pki/ca.crt
export ETCDCTL_CERT=/etc/etcd/pki/server.crt
export ETCDCTL_KEY=/etc/etcd/pki/server.key

etcdctl endpoint health --cluster
etcdctl endpoint status --cluster -w table
etcdctl member list -w table
```

Expect all members healthy and one leader. Quick write test:

```bash
etcdctl put k8s/setup ok
etcdctl get k8s/setup
```

Do **not** run `kubeadm init` until this is green.

---

## 7. Referencing external etcd in `kubeadm init`

With external etcd, kubeadm **does not** create local etcd static pods. You must supply endpoints and the API server client TLS files in `ClusterConfiguration.etcd.external`.

### 7.1 Prerequisites on the first control plane

1. containerd + kubeadm/kubelet/kubectl installed ([main guide](./kubeadm-production-cluster.md))
2. API load balancer DNS ready
3. Client TLS already installed:

```text
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/apiserver-etcd-client.key
```

### 7.2 kubeadm config with `etcd.external`

```bash
cat <<'EOF' | tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.0
controlPlaneEndpoint: "k8s-api.example.com:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
etcd:
  external:
    endpoints:
      - https://10.0.0.10:2379
      - https://10.0.0.11:2379
      - https://10.0.0.12:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

Field reference:

| Field | Meaning |
|-------|---------|
| `etcd.external.endpoints` | Client HTTPS URLs of all etcd members |
| `etcd.external.caFile` | etcd CA that signed member + client certs |
| `etcd.external.certFile` | Client cert kube-apiserver presents to etcd |
| `etcd.external.keyFile` | Matching private key |

`local` and `external` are mutually exclusive. If `external` is set, stacked etcd is skipped.

If your kubeadm version still expects `v1beta3`, change `apiVersion` accordingly (`kubeadm config print init-defaults`).

### 7.3 Initialize the first control plane

```bash
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u):$(id -g)" $HOME/.kube/config
```

Sanity checks:

```bash
kubectl get nodes
kubectl -n kube-system get pods
# There should be NO etcd-* static pods on control planes
kubectl -n kube-system get pods -l component=etcd
```

Install your CNI next (see main guide), then join more control planes / workers.

### 7.4 Join additional control planes

For each extra control plane:

1. Copy the same etcd client material **before** join:

```bash
sudo mkdir -p /etc/kubernetes/pki/etcd
# copy ca.crt, apiserver-etcd-client.crt, apiserver-etcd-client.key into place
```

2. Also copy the usual Kubernetes PKI pieces kubeadm requires for external-etcd HA joins (from the first CP), typically:

```text
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/sa.pub
/etc/kubernetes/pki/sa.key
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-ca.key
/etc/kubernetes/pki/etcd/ca.crt          # etcd CA cert (not necessarily ca.key)
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/apiserver-etcd-client.key
```

Or use the join command printed by `kubeadm init` with `--control-plane --certificate-key ...` (certificate key path still applies for the Kubernetes PKI upload). Ensure etcd client files exist on the joining node either via `--upload-certs` distribution or manual copy—**kubeadm does not install your external etcd PKI for you**.

3. Join:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

---

## 8. Operations cheat sheet

### etcd environment for etcdctl

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379
export ETCDCTL_CACERT=/etc/etcd/pki/ca.crt
export ETCDCTL_CERT=/path/to/client.crt
export ETCDCTL_KEY=/path/to/client.key
```

### Snapshot backup (run regularly; store off-box)

```bash
sudo -u etcd mkdir -p /var/lib/etcd/backups
etcdctl snapshot save /var/lib/etcd/backups/snapshot-$(date +%F-%H%M).db
etcdutl --write-out=table snapshot status /var/lib/etcd/backups/snapshot-*.db
```

### Upgrade etcd (rolling)

1. Snapshot
2. Stop one member: `sudo systemctl stop etcd`
3. Replace `/usr/local/bin/etcd{,ctl,utl}` from the new tarball
4. Start: `sudo systemctl start etcd`
5. Wait for healthy, then next member

### Common failures

| Symptom | Likely cause |
|---------|----------------|
| etcd crash-loop on start | Wrong cert SANs / IP in listen URLs |
| `etcdctl endpoint health` fails | Firewall, wrong CA, clock skew |
| `kubeadm init` can’t reach etcd | CP can’t reach `:2379`, or client cert CN/key mismatch |
| API server logs etcd TLS errors | Paths in `etcd.external` wrong or certs not readable by kube-apiserver |
| Split brain / no leader | Even member count, or peer `2380` blocked |

---

## 9. Quick flow

```text
etcd nodes:  download tarball → install binaries → TLS → /etc/etcd/etcd.conf → systemd → verify quorum
CP nodes:    copy etcd CA + apiserver-etcd-client cert/key into /etc/kubernetes/pki/...
CP1:         kubeadm init --config kubeadm-config.yaml  (etcd.external.endpoints + ca/cert/key)
Cluster:     install CNI → join more CPs (with etcd client PKI) → join workers
Ongoing:     etcdctl snapshots, rolling etcd upgrades
```

---

## References

- [etcd download / releases](https://github.com/etcd-io/etcd/releases)
- [etcd clustering guide](https://etcd.io/docs/latest/op-guide/clustering/)
- [Creating HA clusters with kubeadm (external etcd)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [kubeadm config – etcd.external](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
