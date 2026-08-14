# Ingress from Zero to Hero

Expose HTTP/HTTPS apps on Kubernetes: **Ingress**, **Ingress Controllers**, **TLS**, **Gateway API**, and production patterns on kubeadm (with Calico/Cilium + MetalLB).

Related:

- [domain-dns-ingress-ha.md](./domain-dns-ingress-ha.md) — **registrar DNS → Ingress without SPOF**
- [cni-calico.md](../cni-calico.md) — CNI + MetalLB for LoadBalancer  
- [cni-cilium.md](../cni-cilium.md) — CNI; optional Cilium Ingress / Gateway API  
- [concepts/helm-zero-to-hero.md](./helm-zero-to-hero.md)  
- [concepts/argocd-production.md](./argocd-production.md)  
- [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

Official: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) · [Gateway API](https://gateway-api.sigs.k8s.io/)

---

## 0. Mental model

Kubernetes does **not** implement Ingress by itself. You need:

```text
Internet / LAN
    →  LoadBalancer / NodePort / hostNetwork
    →  Ingress Controller (nginx, Traefik, HAProxy, Cilium, …)
    →  reads Ingress (or Gateway API) objects
    →  Service ClusterIP
    →  Pods
```

| Object | Role |
|--------|------|
| **Service** | Stable in-cluster VIP to pods (`ClusterIP` / `NodePort` / `LoadBalancer`) |
| **Ingress** | L7 rules: host + path → backend Service (HTTP/HTTPS) |
| **IngressClass** | Which controller should handle an Ingress |
| **Ingress Controller** | The software that watches Ingress and configures proxy/LB |
| **Gateway API** | Newer, more expressive L4/L7 API (Gateway, HTTPRoute, …) |

```text
Without controller:  kubectl apply Ingress  →  object stored  →  nothing listens
With controller:     same apply            →  controller programs routes + TLS
```

---

## 1. Choose how traffic enters the cluster

| Front-door | How | When |
|------------|-----|------|
| **LoadBalancer Service** for the controller | Cloud LB or **MetalLB** VIP → controller pods | Production bare metal / cloud (preferred) |
| **NodePort** | `http://nodeIP:3xxxx` | Labs; not pretty for prod HTTPS |
| **hostNetwork: true** on controller | Binds :80/:443 on nodes | Some edge setups; careful with port conflicts |
| **DaemonSet + host ports** | One proxy per node | High traffic edge |

On kubeadm **bare metal**, install **MetalLB** (see Calico guide §7) **or** an external HAProxy/VIP in front of NodePorts.

```text
Production bare-metal typical path:

  DNS api.example.com → MetalLB IP
    → Service type=LoadBalancer (ingress-nginx controller)
      → Ingress rules → app Services → Pods
```

---

## 2. Zero → first working Ingress (Ingress-NGINX)

### 2.1 Prerequisites

```bash
kubectl get nodes
# CNI Ready (Calico/Cilium)
# Optional but recommended on bare metal:
kubectl -n metallb-system get pods   # MetalLB installed
```

### 2.2 Install Ingress-NGINX (Helm)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Pin chart version in prod — example:
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --version 4.11.1 \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.metrics.enabled=true \
  --atomic --wait

kubectl -n ingress-nginx get pods,svc
```

Bare metal without MetalLB yet — temporary NodePort:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --wait
```

### 2.3 Note the external address

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
# EXTERNAL-IP = MetalLB address or cloud LB
# Point DNS (or /etc/hosts) demo.example.com → that IP
```

### 2.4 Deploy a sample app + Ingress

```bash
kubectl create namespace demo

kubectl -n demo create deployment web --image=nginx:1.25 --replicas=2
kubectl -n demo expose deployment web --port=80

kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    # Optional controller-specific tweaks:
    # nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF

kubectl -n demo get ingress
curl -sS -H "Host: demo.example.com" http://<EXTERNAL-IP>/
```

`ingressClassName: nginx` must match the controller’s IngressClass:

```bash
kubectl get ingressclass
```

---

## 3. Ingress API details

### 3.1 pathType

| pathType | Meaning |
|----------|---------|
| `Prefix` | Match path prefix (`/api` matches `/api/v1`) |
| `Exact` | Exact path only |
| `ImplementationSpecific` | Controller-defined (avoid for portability) |

### 3.2 Multiple hosts & paths

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 8080
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### 3.3 TLS on Ingress

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - demo.example.com
      secretName: demo-tls
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Secret must be `kubernetes.io/tls` with `tls.crt` / `tls.key`.

**cert-manager** (production):

```bash
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --version v1.15.3 \
  --set crds.enabled=true \
  --wait
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [demo.example.com]
      secretName: demo-tls          # cert-manager fills this
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

DNS must resolve publicly (for Let’s Encrypt HTTP-01) to your LoadBalancer IP.

### 3.4 IngressClass

```bash
kubectl get ingressclass
kubectl describe ingressclass nginx
```

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # optional
spec:
  controller: k8s.io/ingress-nginx
```

Multiple controllers (nginx + traefik): set `ingressClassName` explicitly on every Ingress — do not rely on default ambiguously.

---

## 4. Controllers compared

| Controller | Notes |
|------------|-------|
| **Ingress-NGINX** | Very common; rich annotations; large community |
| **Traefik** | CRDs + Ingress; built-in dashboard; middlewares |
| **HAProxy Ingress** | HAProxy performance/features |
| **Contour** | Envoy-based |
| **Cilium Ingress / Gateway** | eBPF datapath; tight Cilium integration — see [cni-cilium.md](../cni-cilium.md) |
| **Cloud controller Ingress** | GKE/ALB/AGIC — cloud-specific |

On **Calico** clusters, use any of the above + MetalLB. Calico does not provide an Ingress controller; it provides **NetworkPolicy** that can protect backends.

On **Cilium**, you may use Ingress-NGINX **or** Cilium’s own Ingress/Gateway API instead of (or beside) nginx — avoid two controllers fighting for the same IngressClass.

---

## 5. Gateway API (hero path / future default)

Gateway API splits **infrastructure** (Gateway/GatewayClass) from **app routes** (HTTPRoute).

```bash
# Install CRDs (pin version from gateway-api releases)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml

# Then install a controller that implements Gateway API
# e.g. Ingress-NGINX experimental, Cilium, Istio, Contour, …
```

Conceptual shape:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: ingress-nginx
spec:
  gatewayClassName: cilium   # or nginx / istio / …
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
  namespace: demo
spec:
  parentRefs:
    - name: public
      namespace: ingress-nginx
  hostnames: ["demo.example.com"]
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web
          port: 80
```

For greenfield platforms, evaluate Gateway API; for existing apps, Ingress remains fully valid.

---

## 6. Production best practices

### 6.1 Controllers & capacity

- [ ] ≥ **2 replicas** of the controller; PDB `minAvailable: 1`  
- [ ] `LoadBalancer` + MetalLB/cloud VIP (not single NodePort)  
- [ ] Resource requests/limits; HPA on controller if needed ([metrics-server-hpa.md](../metrics-server-hpa.md))  
- [ ] Spread across nodes/zones (`topologySpreadConstraints` / anti-affinity)  

### 6.2 TLS & certificates

- [ ] Terminate TLS at ingress (or gateway); use cert-manager  
- [ ] Redirect HTTP→HTTPS  
- [ ] Modern TLS only; disable weak ciphers via controller config  
- [ ] Wildcard vs per-host certs — document DNS challenge if HTTP-01 impossible  

### 6.3 Security

- [ ] Default-deny NetworkPolicy; allow Ingress controller → app pods only ([Calico](../cni-calico.md) / [Cilium](../cni-cilium.md))  
- [ ] WAF / rate-limit / auth annotations (or external auth) where required  
- [ ] Don’t expose admin dashboards without SSO  
- [ ] Separate internal vs public IngressClasses / Gateways  

Example: allow only ingress-nginx to reach app pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: demo
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
```

(Adjust selectors to your controller namespace/labels.)

### 6.4 Operations

- [ ] Pin Helm chart versions; GitOps the install  
- [ ] Access logs + metrics (Prometheus ServiceMonitor)  
- [ ] Alert on 5xx rate, upstream failures, cert expiry  
- [ ] Canary / blue-green via annotations, Flagger, or Argo Rollouts  
- [ ] One team owns the controller; app teams own Ingress/HTTPRoute objects  

### 6.5 DNS & naming

Full HA guide (registrar → DNS → VIP with no SPOF): **[domain-dns-ingress-ha.md](./domain-dns-ingress-ha.md)**.

- [ ] Delegate the domain at the **registrar** to an HA DNS provider (Cloudflare, Route53, …)—do not rely on weak registrar-only DNS  
- [ ] Point A/AAAA at a **HA VIP** (MetalLB BGP / keepalived pair / cloud multi-AZ LB), not a single node IP  
- [ ] Automate DNS (ExternalDNS) from Ingress/Gateway hosts  
- [ ] Consistent host naming (`svc.team.env.example.com`)  
- [ ] Split public vs private DNS zones  

### 6.6 Common nginx annotations (useful, controller-specific)

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "32m"
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  nginx.ingress.kubernetes.io/limit-rps: "50"
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

Prefer portable Ingress fields; keep annotations documented — they don’t transfer to Traefik/Cilium unchanged.

---

## 7. End-to-end bare-metal production sketch

```text
1. CNI (Calico or Cilium) Ready
2. MetalLB pool + L2/BGP advertisement
3. ingress-nginx (or Cilium Gateway) Service type=LoadBalancer
4. cert-manager + ClusterIssuer
5. ExternalDNS (optional)
6. App: Deployment + Service + Ingress/HTTPRoute
7. NetworkPolicy: allow controller → app
8. Grafana dashboards for ingress controller metrics
```

```bash
# Smoke
kubectl -n ingress-nginx get svc
kubectl get ingress -A
curl -vk https://demo.example.com/
```

---

## 8. Troubleshooting

| Symptom | Checks |
|---------|--------|
| Ingress `ADDRESS` empty | Controller not running; wrong IngressClass; no LB/MetalLB |
| 404 from controller | Host/path mismatch; wrong Service/port; default backend |
| 502 / 503 | Upstream pods not Ready; NetworkPolicy blocking; wrong port |
| TLS errors | Secret missing/wrong; cert-manager Order failed; DNS not pointing here |
| Only works via Node IP | Using NodePort; configure LoadBalancer/MetalLB + DNS |
| Two controllers conflict | Same IngressClass / default class |

```bash
kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100
kubectl -n demo describe ingress web
kubectl -n demo get endpoints web
kubectl get ingressclass
kubectl -n metallb-system get ipaddresspools,l2advertisements
```

---

## 9. Ingress vs alternatives

| Need | Prefer |
|------|--------|
| HTTP(S) routing by host/path | **Ingress** or **Gateway API** |
| Non-HTTP (TCP/UDP/gRPC L4) | Gateway API TCPRoute / LB Service / controller TCP config |
| East-west service mesh | Istio/Linkerd (still may use Ingress/Gateway at edge) |
| Single app, no routing | Service `LoadBalancer` alone |

---

## 10. Learning path (zero → hero)

```text
1. Install ingress-nginx with NodePort; Host header curl
2. Add MetalLB; Service type=LoadBalancer; real DNS
3. Multi-path / multi-host Ingress
4. TLS Secret manually; then cert-manager HTTP-01
5. NetworkPolicy allow-from-ingress
6. Second IngressClass or internal vs public
7. Metrics + dashboards; PDB; 2 replicas
8. Evaluate Gateway API / Cilium Ingress on Cilium clusters
9. GitOps Applications for controller + app Ingress objects
```

---

## 11. Quick flow

```text
CNI Ready → MetalLB (bare metal) → Ingress Controller (LB)
  → IngressClass → app Service → Ingress (host/path)
  → cert-manager TLS → NetworkPolicy → monitor
```

---

## References

- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Ingress-NGINX](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager](https://cert-manager.io/docs/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [MetalLB](https://metallb.universe.tf/) (via [cni-calico.md](../cni-calico.md))
- [Connect domains without SPOF](./domain-dns-ingress-ha.md)
- [Cilium Ingress / Gateway](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/)
