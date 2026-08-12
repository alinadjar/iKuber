# Calico CNI on a kubeadm Cluster

Configure and install **Project Calico** (Tigera Operator) as the pod network for a kubeadm cluster.

Companion docs: [kubeadm-production-cluster.md](./kubeadm-production-cluster.md) · [cni-cilium.md](./cni-cilium.md)

> Install **one** CNI only. Nodes stay `NotReady` until a CNI is running.

---

## 0. When to choose Calico

| Strength | Notes |
|----------|--------|
| NetworkPolicy + Calico policy | Richer policy than stock NetworkPolicy alone |
| BGP or encapsulation | Native BGP peering, or IPIP/VXLAN overlays |
| Mature on bare metal | Common choice for on-prem kubeadm |
| Optional eBPF dataplane | Can use iptables (default) or eBPF |

---

## 1. Prerequisites (before / during kubeadm init)

### 1.1 Pod CIDR

Calico’s default IP pool is often `192.168.0.0/16`. Whatever you use must match kubeadm’s `networking.podSubnet`.

Example (recommended to set explicitly in both places):

```text
POD_CIDR = 192.168.0.0/16
```

In `kubeadm-config.yaml`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

Or:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 ...
```

If you already initialized with another CIDR (e.g. `10.244.0.0/16`), set Calico’s `ipPools[].cidr` to **that** value—do not change only one side.

### 1.2 Node requirements

- Linux with `ip_forward` / `br_netfilter` already set (see main guide)
- Ports between nodes (typical):

| Protocol | Port | Use |
|----------|------|-----|
| TCP | 179 | BGP (if BGP enabled) |
| UDP | 4789 | VXLAN (if VXLAN encapsulation) |
| IP proto 4 | — | IP-in-IP (if IPIP encapsulation) |
| TCP | 5473 | Typha (if used at scale) |

### 1.3 Cluster state

```bash
kubectl get nodes          # expect NotReady until CNI
kubectl get pods -A
# No other CNI DaemonSet should be installed
```

Pin a Calico version that supports your Kubernetes minor version. Examples below use `v3.29.1`—replace with a current compatible release from [Calico releases](https://github.com/projectcalico/calico/releases).

```bash
export CALICO_VERSION=v3.29.1
```

---

## 2. Install the Tigera Operator

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml

kubectl -n tigera-operator get pods
kubectl -n tigera-operator wait --for=condition=Available deploy/tigera-operator --timeout=120s
```

Newer Calico releases may also ship CRDs separately (e.g. `v1_crd_projectcalico_org.yaml`). If the operator docs for your version require it, apply that **before** the operator manifest.

---

## 3. Configure Calico (`Installation` CR)

Download and edit custom resources so the IP pool matches your cluster.

```bash
curl -fsSL -o custom-resources.yaml \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml

# Inspect / edit before apply
grep -A20 'kind: Installation' custom-resources.yaml
```

Minimal production-oriented example (BGP + IPIP, CIDR = kubeadm podSubnet):

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # registry: quay.io/   # optional: private mirror prefix
  calicoNetwork:
    bgp: Enabled
    ipPools:
      - name: default-ipv4-ippool
        cidr: 192.168.0.0/16          # MUST match networking.podSubnet
        encapsulation: IPIP           # or VXLAN, or VXLANCrossSubnet, or None (BGP-only)
        natOutgoing: Enabled
        nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

### Common configuration choices

| Goal | Settings |
|------|----------|
| Overlay on L2/L3 that doesn’t route pod CIDRs | `encapsulation: IPIP` or `VXLAN` |
| Underlay / BGP to ToR | `bgp: Enabled`, `encapsulation: None` (network must route pod CIDR) |
| Cloud / no BGP | Often `bgp: Disabled` + `encapsulation: VXLAN` |
| Dual-stack | Add an IPv6 pool and enable IPv6 in kubeadm + Calico |

VXLAN-only example (no BGP):

```yaml
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - cidr: 192.168.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
```

Apply:

```bash
kubectl create -f custom-resources.yaml
```

---

## 4. Wait until nodes are Ready

```bash
kubectl get pods -n calico-system -o wide
kubectl get tigerastatus
kubectl get nodes -o wide

# Expect calico-node Running on every node, apiserver/typha healthy
kubectl -n calico-system wait --for=condition=Ready pod -l k8s-app=calico-node --timeout=300s
```

Optional CLI (`calicoctl` / `kubectl calico`):

```bash
# If calicoctl is installed and configured:
calicoctl get ippool -o yaml
calicoctl node status
```

---

## 5. Verify connectivity

```bash
kubectl create namespace netcheck
kubectl -n netcheck create deployment ping --image=busybox:1.36 --replicas=3 -- sleep 3600
kubectl -n netcheck wait --for=condition=Available deploy/ping --timeout=120s

POD_A=$(kubectl -n netcheck get pod -l app=ping -o jsonpath='{.items[0].metadata.name}')
POD_B_IP=$(kubectl -n netcheck get pod -l app=ping -o jsonpath='{.items[1].status.podIP}')

kubectl -n netcheck exec "$POD_A" -- ping -c 3 "$POD_B_IP"
kubectl -n netcheck delete ns netcheck --wait=false
```

---

## 6. NetworkPolicy example

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

Calico also supports `NetworkPolicy` / `GlobalNetworkPolicy` CRDs (`projectcalico.org`) for host endpoints and richer matchers—use those when you need more than stock Kubernetes NetworkPolicy.

---

## 7. Add MetalLB (LoadBalancer IPs on bare metal)

On bare metal / kubeadm, `Service` type `LoadBalancer` stays `<pending>` unless a controller assigns external IPs. **MetalLB** fills that role. Install it **after** Calico is healthy and nodes are `Ready`.

MetalLB and Calico solve different problems:

| Component | Owns |
|-----------|------|
| Calico | Pod networking + NetworkPolicy |
| MetalLB | External IPs for `LoadBalancer` Services |

### 7.1 Choose advertisement mode

| Mode | How it works | Best when |
|------|----------------|-----------|
| **L2** (ARP/NDP) | One node answers ARP for the Service IP | Same L2 LAN; simplest; recommended start |
| **BGP** | Speakers peer with ToR / router and advertise Service IPs | Routed DC fabric; need ECMP |

**Calico note:** Calico BGP (pod routes) and MetalLB BGP (Service IPs) are separate. For most Calico + kubeadm clusters, use **MetalLB L2** even if Calico uses BGP or IPIP/VXLAN. Only use MetalLB BGP if your network team peers speakers to routers for LB VIP ranges. Do not put MetalLB pool ranges inside the pod CIDR or service CIDR.

Example planning values (must be free addresses on your LAN / routed to the cluster):

```text
METALLB_POOL = 10.0.0.200-10.0.0.250
# or a CIDR: 10.0.0.200/29
```

### 7.2 Prerequisites

```bash
# Calico up, nodes Ready
kubectl get nodes
kubectl -n calico-system get pods

# If kube-proxy runs in IPVS mode, enable strictARP (required by MetalLB)
kubectl -n kube-system get cm kube-proxy -o yaml | grep mode
```

If IPVS is enabled:

```bash
kubectl -n kube-system get cm kube-proxy -o yaml | \
  sed 's/strictARP: false/strictARP: true/' | \
  kubectl apply -f -

kubectl -n kube-system rollout restart ds/kube-proxy
```

(iptables mode needs no `strictARP` change.)

Allow speaker traffic as needed:

| Mode | Allow |
|------|--------|
| L2 | ARP/NDP on the node LAN (usually already works) |
| BGP | TCP **179** between speaker nodes and BGP peers |

### 7.3 Install MetalLB

Pin a release (example `v0.14.9`—use a current [MetalLB release](https://github.com/metallb/metallb/releases)):

```bash
export METALLB_VERSION=v0.14.9

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml

kubectl -n metallb-system get pods
kubectl -n metallb-system wait --for=condition=Available deploy/controller --timeout=120s
kubectl -n metallb-system wait --for=condition=Ready pod -l app=metallb --timeout=180s
```

Components:

- `controller` — assigns IPs from pools  
- `speaker` (DaemonSet) — advertises those IPs (L2 and/or BGP)

Pods stay idle until you create an `IPAddressPool` plus an advertisement CR.

For FRR-based BGP features, some releases also ship `metallb-frr.yaml` instead of `metallb-native.yaml`—use that only if you need those BGP options.

### 7.4 Configure L2 mode (typical with Calico)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.200-10.0.0.250
  # avoidBuggyIPs: true   # skip .0 / .255 in CIDR-style pools
  # autoAssign: true      # default; set false for pools used only via annotation
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
  # Optional: limit which nodes announce (e.g. only workers)
  # nodeSelectors:
  #   - matchLabels:
  #       node-role.kubernetes.io/worker: ""
  # Optional: bind ARP to a specific NIC
  # interfaces:
  #   - eth0
EOF
```

Verify:

```bash
kubectl -n metallb-system get ipaddresspools.metallb.io
kubectl -n metallb-system get l2advertisements.metallb.io
```

### 7.5 Configure BGP mode (optional)

Use when routers should learn Service VIPs via BGP. Speakers peer with your ToR (example ASN/peer—replace with real values).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
    - 203.0.113.16/28
---
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: tor-peer
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.0.0.1
  # nodeSelectors:   # optional: which nodes open the session
  #   - matchLabels:
  #       metallb-bgp: "true"
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: default-bgp
  namespace: metallb-system
spec:
  ipAddressPools:
    - bgp-pool
EOF
```

Keep MetalLB’s ASN/peers distinct from Calico’s pod-route BGP peering plan so troubleshooting stays clear (different address families/purposes even if both use TCP/179).

### 7.6 Multiple pools / pin a Service to a pool

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.240/29
  autoAssign: false
```

Request an IP from that pool:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  annotations:
    metallb.universe.tf/address-pool: ingress-pool
    # or request a specific IP:
    # metallb.universe.tf/loadBalancerIPs: 10.0.0.241
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: ingress-nginx
```

### 7.7 Test a LoadBalancer Service

```bash
kubectl create deployment nginx-lb --image=nginx:stable --replicas=2
kubectl expose deployment nginx-lb --port=80 --type=LoadBalancer

kubectl get svc nginx-lb -w
# EXTERNAL-IP should become an address from your pool (not <pending>)

LB_IP=$(kubectl get svc nginx-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -sS -o /dev/null -w "%{http_code}\n" http://${LB_IP}/

kubectl delete svc nginx-lb deployment nginx-lb
```

From another host on the same L2 segment (L2 mode), `curl`/`ping` to `LB_IP` should reach the Service. In BGP mode, test from a network that has the route via your peer router.

### 7.8 Calico NetworkPolicy and MetalLB

If you use default-deny policies, do not lock down `metallb-system` in a way that breaks speaker ↔ controller or speaker ↔ API server. Allow DNS and API access for MetalLB pods. Client traffic to a LoadBalancer IP is handled on the node (kube-proxy / datapath) toward endpoints—namespace policies on the **workload** still apply to pod ingress as usual.

Optional: restrict which nodes run speakers with the speaker DaemonSet nodeSelector / L2 `nodeSelectors` so control-plane nodes are not the ARP owner if you prefer workers only.

### 7.9 MetalLB troubleshooting

| Symptom | Check |
|---------|--------|
| `EXTERNAL-IP` stuck `<pending>` | Pool + `L2Advertisement`/`BGPAdvertisement` exist; controller logs |
| IP assigned but not reachable (L2) | Client on same L2? Wrong interface? Another host using the IP? |
| Flapping ARP owner | Node/speaker restarts; check speaker logs |
| BGP session down | TCP/179, ASN/IP, `BGPPeer` status, speaker logs |
| Only works on some nodes | `nodeSelectors` / interface filters too strict |

```bash
kubectl -n metallb-system logs deploy/controller --tail=100
kubectl -n metallb-system logs ds/speaker --tail=100
kubectl -n metallb-system get ipaddresspools,l2advertisements,bgppeers,bgpadvertisements
```

---

## 8. Useful post-install tweaks

### 8.1 MTU

If tunnels drop large packets, set MTU in the Installation CR (value depends on underlay; VXLAN often needs ~50 bytes less than interface MTU):

```yaml
spec:
  calicoNetwork:
    mtu: 1450
```

Then re-apply / patch the `Installation` named `default`.

### 8.2 Private registries

Mirror Calico images and set `spec.registry` (and/or `imagePath` / `imagePrefix`) on the `Installation` resource per [Calico install reference](https://docs.tigera.io/calico/latest/reference/installation/api).

### 8.3 Upgrade (high level)

1. Read release notes for your target Calico version  
2. Upgrade operator manifest for the new version  
3. Operator reconciles Calico components  
4. Watch `calico-system` and `tigera-operator` pods / `TigeraStatus`

---

## 9. Troubleshooting

| Symptom | Check |
|---------|--------|
| Nodes stay NotReady | `kubectl -n calico-system get pods`; operator logs |
| IP pool mismatch | Compare `podSubnet` vs `Installation.spec.calicoNetwork.ipPools` |
| Cross-node ping fails | Encapsulation vs underlay routing; firewall for BGP/VXLAN/IPIP |
| BGP not establishing | TCP/179; `calicoctl node status`; ToR ASN/peer config |
| Operator CrashLoop | CRD/version skew; `kubectl -n tigera-operator logs deploy/tigera-operator` |
| LoadBalancer pending | See [§7 MetalLB](#7-add-metallb-loadbalancer-ips-on-bare-metal) |

```bash
kubectl -n tigera-operator logs deploy/tigera-operator --tail=100
kubectl -n calico-system logs ds/calico-node --tail=100
kubectl get installation default -o yaml
```

---

## 10. Quick flow

```text
kubeadm init with podSubnet=192.168.0.0/16
  → apply tigera-operator.yaml
  → edit custom-resources (cidr + encapsulation/bgp)
  → apply custom-resources.yaml
  → wait calico-system Ready → nodes Ready
  → smoke-test pod networking → NetworkPolicies
  → install MetalLB → IPAddressPool + L2Advertisement (or BGP)
  → test Service type=LoadBalancer
```

---

## References

- [Calico quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [Install on kubeadm / self-managed](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Installation API reference](https://docs.tigera.io/calico/latest/reference/installation/api)
- [Determine IP pool / encapsulation](https://docs.tigera.io/calico/latest/networking/configuring/determine-best-networking)
- [MetalLB installation](https://metallb.universe.tf/installation/)
- [MetalLB configuration](https://metallb.universe.tf/configuration/)
- [MetalLB Layer 2](https://metallb.universe.tf/concepts/layer2/)
- [MetalLB BGP](https://metallb.universe.tf/concepts/bgp/)
