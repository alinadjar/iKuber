# Connect Domains to Ingress without a Single Point of Failure

How to point **registrar domains** at your Kubernetes Ingress edge and design the path so DNS, VIP, and controllers are not single points of failure.

Related: [ingress-zero-to-hero.md](./ingress-zero-to-hero.md) · [cni-calico.md](../cni-calico.md) (MetalLB) · [cni-cilium.md](../cni-cilium.md) · [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

---

## 0. End-to-end path (and where SPOFs hide)

```text
User
  → Recursive DNS
  → Authoritative DNS (often NOT your registrar’s “parking” DNS)
  → VIP / anycast / multi-A addresses
  → Ingress controller pods (≥2)
  → Service → app pods
```

| Layer | SPOF if… | HA approach |
|-------|----------|-------------|
| **DNS** | One broken NS / one DNS vendor outage | 2+ nameservers; secondary DNS or dual providers; short TTL only when changing |
| **VIP / LB IP** | One MetalLB L2 node dies and ARP is slow; one VM LB | BGP ECMP, keepalived VIP pair, cloud LB, or dual-region DNS |
| **Ingress controller** | One replica | ≥2 replicas + PDB + anti-affinity |
| **Node / rack** | All controllers on one node | Spread across workers/AZs |
| **Cluster** | Whole cluster down | Warm DR / multi-cluster + DNS failover (advanced) |
| **Certificate** | HTTP-01 only via one broken path | DNS-01; dual issuers; monitor expiry |

**Registrar ≠ DNS.** Buying `example.com` at a registrar only owns the name. For production, delegate to a **serious DNS host** (Cloudflare, Route53, Google Cloud DNS, Azure DNS, NS1, …) or run dual DNS. Registrar “basic DNS” is often the weakest link.

---

## 1. Recommended production topologies

### 1.A Bare metal — MetalLB BGP (best in-cluster VIP HA)

```text
DNS A/AAAA record(s)
  → Anycast or ECMP VIP(s) advertised by MetalLB BGP speakers
  → Multiple nodes accept traffic
  → ingress-nginx Service type=LoadBalancer
```

- Prefer **MetalLB BGP** over L2 when routers can peer — failing a node withdraws the route; ECMP spreads load.  
- L2 mode: one speaker “owns” ARP; failover works but is a softer HA story. Still run **≥2 ingress pods**.  
- Details: [cni-calico.md §7](../cni-calico.md).

### 1.B Bare metal — External HA pair in front of Kubernetes (classic SPOF-resistant)

```text
DNS → VIP (keepalived/VRRP) on two Linux LB VMs
         → HAProxy/Nginx upstreams = worker NodePorts or ingress NodePorts
              → Ingress controller pods
```

- Two VMs in different racks/power.  
- VIP floats with keepalived; no dependency on MetalLB.  
- Ingress Service can stay `NodePort` or `ClusterIP` + hostPorts depending on design.  
- Slightly more ops; very clear blast-radius control.

### 1.C Cloud

```text
DNS → Cloud Load Balancer (managed, multi-AZ)
        → Service type=LoadBalancer (cloud controller)
             → Ingress controller
```

Use the cloud’s **multi-AZ LB**; don’t point DNS at a single instance public IP.

### 1.D Multi-cluster / multi-site (no cluster SPOF)

```text
DNS (health-checked failover or weighted / geo)
  → Cluster A ingress VIP
  → Cluster B ingress VIP (standby or active-active)
```

Requires replicated apps (GitOps + data strategy). DNS failover is **not** instant; combine with low TTL **during** cutover only.

---

## 2. Step-by-step: registrar → DNS → Ingress

### 2.1 At the registrar — delegate nameservers

1. Create a zone at your DNS provider (Cloudflare / Route53 / …).  
2. Provider shows nameservers, e.g.:

```text
ns1.exampledns.net
ns2.exampledns.net
```

3. At the **registrar**, set the domain’s **nameservers** to those values (not “A record at registrar” only).  
4. Wait for delegation (hours possible; check with dig):

```bash
dig NS example.com +short
dig NS example.com @a.root-servers.net   # follow delegation if needed
```

**Avoid SPOF:** pick a DNS host with **anycast NS** and SLA. Optionally configure a **secondary** DNS provider that AXFRs the zone (dual-DNS).

### 2.2 Create records pointing at the Ingress edge

**Single VIP (simplest, VIP layer must be HA):**

```text
Type  Name              Value              TTL
A     demo.example.com  203.0.113.10       300
AAAA  demo.example.com  2001:db8::10       300
```

`203.0.113.10` = MetalLB address **or** keepalived VIP **or** cloud LB.

```bash
# Discover the address Kubernetes expects
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

**Multiple A records (DNS round-robin):**

```text
A  demo.example.com  203.0.113.10
A  demo.example.com  203.0.113.11
```

Use only if **both** IPs are healthy front-doors (two LB VIPs or two clusters). Clients cache and don’t health-check — pair with provider **health checks** / failover when possible (Route53 health checks, Cloudflare load balancing, etc.).

**Apex / root domain (`example.com`):**

- Prefer provider **ALIAS/ANAME/CNAME flattening** to the LB hostname if offered.  
- Or A/AAAA to VIP(s).  
- Don’t CNAME the apex unless the DNS product supports flattening.

**www → apex:**

```text
CNAME  www.example.com  example.com
# or both www and apex as A → same VIP
```

### 2.3 TTL strategy

| Phase | TTL |
|-------|-----|
| Steady state | 300–3600s (balance cache vs agility) |
| Before migration / failover test | Lower to 60–120s **ahead of time** |
| After stable | Raise again |

Low TTL does not remove SPOF; it only speeds **planned** DNS changes.

### 2.4 Wire Kubernetes Ingress to the hostname

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [demo.example.com]
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

```bash
dig +short demo.example.com
curl -vk https://demo.example.com/
```

---

## 3. Automate DNS updates (ExternalDNS)

Avoid hand-editing records for every Ingress.

```bash
# Example: Cloudflare (pin chart version in prod)
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
# Create Secret with API token; grant Zone.DNS edit only

helm upgrade --install external-dns external-dns/external-dns \
  -n external-dns --create-namespace \
  --set provider=cloudflare \
  --set env[0].name=CF_API_TOKEN \
  --set env[0].valueFrom.secretKeyRef.name=cloudflare-api \
  --set env[0].valueFrom.secretKeyRef.key=token \
  --set domainFilters[0]=example.com \
  --set policy=sync \
  --set sources[0]=ingress \
  --set txtOwnerId=prod-cluster-a \
  --wait
```

ExternalDNS creates A/AAAA/CNAME from Ingress `host` + LB address.

**HA notes:**

- `txtOwnerId` unique per cluster so two clusters don’t fight.  
- For active-active dual cluster, prefer DNS load-balancing products over two ExternalDNS writing the same names.  
- Restrict RBAC; use least-privilege API tokens.

---

## 4. Removing SPOFs layer by layer (checklist)

### 4.1 DNS layer

- [ ] Domain NS delegated to HA DNS provider (not only registrar UI “hosting”)  
- [ ] ≥2 nameserver hostnames; ideally anycast  
- [ ] Optional: secondary DNS provider  
- [ ] Registrar account MFA + registry lock / transfer lock  
- [ ] Document who can change NS (social-engineering risk is a SPOF too)  

```bash
dig NS example.com +short
dig demo.example.com +short
dig demo.example.com @ns1.your-dns-provider.net +short
```

### 4.2 Edge IP / LB layer

**MetalLB L2**

- [ ] ≥2 worker nodes that can speak  
- [ ] Ingress controller replicas ≥2 on different nodes  
- [ ] Accept short ARP failover; test `kubectl drain` on speaker node  

**MetalLB BGP**

- [ ] Peer with ≥2 ToRs / routers if possible  
- [ ] ECMP so multiple next-hops exist  
- [ ] Monitor BGP session flaps  

**External keepalived + HAProxy**

- [ ] Two VMs, different hypervisors/racks  
- [ ] VRRP/unicast keepalived; fence carefully  
- [ ] HAProxy health checks to ingress NodePorts  

**Cloud LB**

- [ ] Multi-AZ enabled  
- [ ] Health check = ingress /healthz  

### 4.3 Ingress controller layer

```bash
kubectl -n ingress-nginx get deploy,pdb
kubectl -n ingress-nginx get pods -o wide   # different nodes?
```

```yaml
# Sketch — ensure chart values include:
controller:
  replicaCount: 2
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
          topologyKey: kubernetes.io/hostname
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
```

- [ ] PDB `minAvailable: 1` (or `maxUnavailable: 1` with 3+ replicas)  
- [ ] Requests/limits; HPA optional  
- [ ] Don’t schedule all replicas on one rack if topology labels exist  

### 4.4 Certificate layer

- [ ] cert-manager with DNS-01 if HTTP-01 path can fail during incidents  
- [ ] Alert on cert expiry (`cert-manager` metrics / blackbox)  
- [ ] Staging issuer tested  

### 4.5 App layer

- [ ] App pods ≥2; PDB  
- [ ] NetworkPolicy allow from ingress only  
- [ ] Readiness probes so bad pods leave Service  

---

## 5. Failure drills (prove there’s no SPOF)

| Drill | Expectation |
|-------|-------------|
| `kubectl drain` node running an ingress pod | Other replica serves; DNS unchanged |
| `kubectl drain` MetalLB L2 leader | VIP moves; brief blip possible |
| Stop one keepalived node | VIP on peer; traffic continues |
| Break one BGP peer | Alternate path remains |
| Quiesce cluster A (multi-cluster DNS) | Health check fails over to B (measure RTO) |
| Revoke/replace TLS | Renew without DNS change |

```bash
# While draining, from outside:
while true; do curl -fsS -o /dev/null -w "%{http_code}\n" https://demo.example.com/; sleep 1; done
```

---

## 6. Patterns to avoid

| Pattern | Why it’s a SPOF / risk |
|---------|------------------------|
| DNS A → single node ExternalIP / hostNetwork one node | Node death = outage |
| One ingress controller replica | Pod/node death = outage |
| Registrar-only DNS with one NS | Vendor/registrar outage |
| CNAME to a hostname you don’t control without monitoring | Silent dependency |
| Ultra-low TTL forever | Extra resolver load; little HA benefit |
| Two ExternalDNS controllers same `txtOwnerId` | Flapping records |
| Pointing prod domain at lab MetalLB IP | Accidental cutover |

---

## 7. Minimal “good enough” vs “strong HA”

### Good enough (single cluster)

```text
Registrar → Cloudflare/Route53 NS
  A record → MetalLB IP (BGP if possible, else L2)
  ingress-nginx × 2 + PDB + anti-affinity
  cert-manager
```

### Stronger

```text
Above, plus:
  Dual DNS or DNS LB health checks
  External keepalived pair OR cloud multi-AZ LB
  DNS-01 certs
  Quarterly drain/failover drills
```

### Strongest (site-level)

```text
Active-active or warm standby cluster
  DNS failover / traffic manager
  Replicated data plane (DB, object storage)
  Independent ingress VIPs per site
```

---

## 8. Quick verification commands

```bash
# Delegation
dig NS example.com +short

# Edge IP
dig +short demo.example.com
kubectl -n ingress-nginx get svc ingress-nginx-controller

# HTTPS
curl -vk https://demo.example.com/

# Controllers spread
kubectl -n ingress-nginx get pods -o wide

# MetalLB
kubectl -n metallb-system get ipaddresspools,l2advertisements,bgppeers 2>/dev/null
```

---

## 9. Quick flow

```text
1. Delegate example.com NS at registrar → HA DNS provider
2. Make Ingress VIP HA (MetalLB BGP / keepalived pair / cloud LB)
3. Run ≥2 ingress controller pods across nodes
4. A/AAAA (or ExternalDNS) → VIP
5. Ingress host = demo.example.com + cert-manager
6. NetworkPolicy + PDB on apps
7. Drain tests prove failover
```

---

## References

- [ingress-zero-to-hero.md](./ingress-zero-to-hero.md)
- [ExternalDNS](https://kubernetes-sigs.github.io/external-dns/)
- [MetalLB](https://metallb.universe.tf/)
- [cert-manager DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/)
- [Ingress-NGINX](https://kubernetes.github.io/ingress-nginx/)
