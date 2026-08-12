# Cilium CNI on a kubeadm Cluster

Configure and install **Cilium** as the pod network (and optionally kube-proxy replacement) for a kubeadm cluster.

Companion docs: [kubeadm-production-cluster.md](./kubeadm-production-cluster.md) · [cni-calico.md](./cni-calico.md)

> Install **one** CNI only. Nodes stay `NotReady` until a CNI is running.

---

## 0. When to choose Cilium

| Strength | Notes |
|----------|--------|
| eBPF datapath | High performance networking & observability |
| NetworkPolicy + CiliumNetworkPolicy | L3–L7 aware policies (HTTP, Kafka, etc.) |
| Hubble | Built-in flow visibility |
| kube-proxy replacement | Optional full Service LB in eBPF |

**Kernel:** prefer Linux **5.10+** (or distro backports) for full feature support.

---

## 1. Prerequisites

### 1.1 Pod / cluster CIDRs

Cilium works with the CIDR you passed to kubeadm. Common choice:

```text
POD_CIDR     = 10.244.0.0/16
SERVICE_CIDR = 10.96.0.0/12
```

In `kubeadm-config.yaml`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
controlPlaneEndpoint: "k8s-api.example.com:6443"
```

### 1.2 Decide: keep kube-proxy or replace it

| Mode | kubeadm | Cilium install |
|------|---------|----------------|
| **A. With kube-proxy** (simpler) | Normal `kubeadm init` | Default Cilium install |
| **B. kube-proxy replacement** | `kubeadm init --skip-phases=addon/kube-proxy` (or skip in config) | Set `kubeProxyReplacement=true` + API host/port |

For mode B, also skip kube-proxy in the kubeadm config:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
skipPhases:
  - addon/kube-proxy
```

### 1.3 Node / firewall notes

Cilium primarily uses eBPF; VXLAN/Geneve overlays (if used) need UDP between nodes (commonly **8472** for VXLAN or **6081** for Geneve—confirm for your Cilium version/config). Direct routing mode needs the underlay to route the pod CIDR.

### 1.4 Cluster state

```bash
kubectl get nodes
# No Flannel/Calico/other CNI already installed
```

Pin versions. Examples use Cilium `1.16.5`—replace with a release compatible with your Kubernetes version from [Cilium stable docs](https://docs.cilium.io/).

```bash
export CILIUM_VERSION=1.16.5
export CILIUM_CLI_VERSION=v0.16.23   # pick a CLI release that matches your workflow
```

---

## 2. Install the Cilium CLI (recommended)

On an admin machine with `kubectl` access to the cluster:

```bash
# Linux amd64 example — see https://github.com/cilium/cilium-cli/releases
curl -fsSL -o cilium-linux-amd64.tar.gz \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
cilium version --client
```

Alternatively use **Helm** (section 4) without the CLI.

---

## 3. Install Cilium with the CLI

### 3.A Default (kube-proxy kept)

```bash
cilium install --version ${CILIUM_VERSION}
cilium status --wait
```

### 3.B kube-proxy replacement (production-oriented)

Use the **control plane endpoint** (load balancer DNS/IP), not a single node IP, when you have HA:

```bash
API_SERVER_HOST=k8s-api.example.com   # or LB VIP
API_SERVER_PORT=6443

cilium install --version ${CILIUM_VERSION} \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=${API_SERVER_HOST} \
  --set k8sServicePort=${API_SERVER_PORT}

cilium status --wait
```

`k8sServiceHost` / `k8sServicePort` are required when kube-proxy is absent so agents can reach the API server without relying on the `kubernetes` ClusterIP service via kube-proxy.

If you already installed kube-proxy and want to switch later:

```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy   # avoid kubeadm upgrade recreating it
# then cilium upgrade with kubeProxyReplacement=true + k8sServiceHost/Port
# clean leftover iptables KUBE-* rules on nodes if needed (carefully)
```

---

## 4. Install with Helm (alternative)

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

# Mode A — with kube-proxy
helm install cilium cilium/cilium --version ${CILIUM_VERSION} \
  --namespace kube-system

# Mode B — kube-proxy replacement
helm install cilium cilium/cilium --version ${CILIUM_VERSION} \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=k8s-api.example.com \
  --set k8sServicePort=6443
```

Useful Helm values (examples):

```bash
helm install cilium cilium/cilium --version ${CILIUM_VERSION} \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

| Value | Meaning |
|-------|---------|
| `routingMode=tunnel` | Overlay (default-friendly on mixed L2/L3) |
| `routingMode=native` | Direct routing; underlay must route pod CIDR |
| `ipam.mode=cluster-pool` | Cilium allocates from its own pools (set `clusterPoolIPv4PodCIDRList`) |
| `ipam.mode=kubernetes` | Use Kubernetes/`kube-controller-manager` node pod CIDRs |
| `hubble.*` | Observability UI / relay |

For `cluster-pool` IPAM, align the pool with planning (and often with kubeadm `podSubnet`):

```bash
--set ipam.mode=cluster-pool \
--set ipam.operator.clusterPoolIPv4PodCIDRList={10.244.0.0/16} \
--set ipam.operator.clusterPoolIPv4MaskSize=24
```

---

## 5. Wait until nodes are Ready

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
kubectl get nodes -o wide

cilium status --wait
# or:
kubectl -n kube-system wait --for=condition=Ready pod -l k8s-app=cilium --timeout=300s
```

Confirm kube-proxy mode if using replacement:

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- \
  cilium status 2>/dev/null | grep -i KubeProxyReplacement
```

---

## 6. Connectivity & Hubble checks

### 6.1 Built-in connectivity test

```bash
cilium connectivity test
```

### 6.2 Manual smoke test

```bash
kubectl create namespace netcheck
kubectl -n netcheck create deployment ping --image=busybox:1.36 --replicas=3 -- sleep 3600
kubectl -n netcheck wait --for=condition=Available deploy/ping --timeout=120s

POD_A=$(kubectl -n netcheck get pod -l app=ping -o jsonpath='{.items[0].metadata.name}')
POD_B_IP=$(kubectl -n netcheck get pod -l app=ping -o jsonpath='{.items[1].status.podIP}')

kubectl -n netcheck exec "$POD_A" -- ping -c 3 "$POD_B_IP"
kubectl -n netcheck delete ns netcheck --wait=false
```

### 6.3 Enable Hubble (if not already)

```bash
cilium hubble enable --ui
cilium status
# Port-forward UI when needed:
cilium hubble ui
```

---

## 7. Network policy examples

### Kubernetes NetworkPolicy

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

### CiliumNetworkPolicy (L7 HTTP example)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-http
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/api/.*"
```

---

## 8. Production configuration notes

### 8.1 HA API endpoint

Always set `k8sServiceHost` to the **load balancer / `controlPlaneEndpoint` host** when using kube-proxy replacement so agents survive a single control-plane outage.

### 8.2 Native routing vs tunnel

- **Tunnel (VXLAN/Geneve):** fewer underlay requirements; slight overhead  
- **Native routing:** better performance; configure node routes / BGP (e.g. Cilium BGP control plane) so pod CIDRs are reachable

### 8.3 kubeadm upgrades with kube-proxy skipped

Keep kube-proxy disabled across upgrades (`skipPhases` / delete ConfigMap) so kubeadm does not bring kube-proxy back.

### 8.4 Upgrade Cilium

```bash
# CLI
cilium upgrade --version <new-version>

# Helm
helm upgrade cilium cilium/cilium --version <new-version> \
  --namespace kube-system --reuse-values
```

Follow [Cilium upgrade notes](https://docs.cilium.io/en/stable/operations/upgrade/) for your minor version jump.

---

## 9. Troubleshooting

| Symptom | Check |
|---------|--------|
| Nodes NotReady | `cilium status`; `kubectl -n kube-system logs ds/cilium` |
| Agents can’t reach API | `k8sServiceHost`/`Port` wrong when kube-proxy is off |
| Cross-node fail | Tunnel UDP blocked, or native routing missing routes |
| BPF / kernel errors | Kernel too old; `cilium status` features; dmesg |
| Hubble UI empty | Relay/UI not enabled; `cilium hubble enable --ui` |

```bash
cilium status --verbose
kubectl -n kube-system logs ds/cilium -c cilium-agent --tail=100
kubectl -n kube-system get cm cilium-config -o yaml
```

---

## 10. Quick flow

```text
Mode A:
  kubeadm init (normal)
    → cilium install --version X
    → cilium status --wait → connectivity test

Mode B (kube-proxy replacement):
  kubeadm init --skip-phases=addon/kube-proxy
    → cilium install --set kubeProxyReplacement=true \
         --set k8sServiceHost=<LB> --set k8sServicePort=6443
    → cilium status --wait → hubble / policies
```

---

## References

- [Install Cilium with kubeadm](https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/)
- [Kubernetes without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium CLI getting started](https://docs.cilium.io/en/stable/gettingstarted/cilium-cli/)
- [Helm values reference](https://docs.cilium.io/en/stable/helm-reference/)
- [Network policy](https://docs.cilium.io/en/stable/security/policy/)
