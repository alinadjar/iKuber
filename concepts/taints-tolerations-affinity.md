# Taints, Tolerations, Affinity & Anti-Affinity

How Kubernetes decides **which nodes can run which pods**, and how to steer placement for HA, isolation, and special hardware.

Related: [kubeadm-production-cluster.md](../kubeadm-production-cluster.md)

---

## Mental model

| Mechanism | Direction | Question it answers |
|-----------|-----------|---------------------|
| **Taint** (on node) | Node → pods | “Who is *not* welcome here unless they opt in?” |
| **Toleration** (on pod) | Pod → taint | “I accept this node’s restriction.” |
| **Node affinity** (on pod) | Pod → nodes | “I *prefer* / *require* nodes with these labels.” |
| **Pod affinity** (on pod) | Pod → other pods | “Schedule me *near* pods that match …” |
| **Pod anti-affinity** (on pod) | Pod → other pods | “Schedule me *away from* pods that match …” |

```text
Taints/tolerations  →  repel by default (opt-in to tainted nodes)
Affinity            →  attract (hard or soft rules)
Anti-affinity       →  spread / separate (hard or soft rules)
```

They compose: a pod must **tolerate** a node’s taints **and** satisfy affinity rules to land there.

---

## 1. Taints and tolerations

### 1.1 What a taint is

A **taint** marks a node so the scheduler **avoids** placing pods there unless the pod has a matching **toleration**.

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>
# Example:
kubectl taint nodes worker-gpu gpu=true:NoSchedule

# Remove the same taint (note the trailing "-")
kubectl taint nodes worker-gpu gpu=true:NoSchedule-
```

Built-in example (control plane from kubeadm):

```bash
kubectl describe node <control-plane> | grep -A5 Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

### 1.2 Effects

| Effect | Meaning |
|--------|---------|
| `NoSchedule` | New pods without a matching toleration are **not scheduled**. Existing pods stay. |
| `PreferNoSchedule` | Scheduler **tries** to avoid the node; may still place pods if needed. |
| `NoExecute` | No new pods without toleration; **evicts** existing pods that don’t tolerate (unless they already have a matching toleration, optionally with `tolerationSeconds`). |

### 1.3 Toleration matching

A toleration matches a taint when:

- `key` is equal (or toleration omits `key` and sets `operator: Exists` to match all keys), and  
- `effect` is equal (or toleration omits `effect` to match all effects), and  
- `operator` is `Equal` (default: `value` must match) or `Exists` (value ignored).

```yaml
tolerations:
  # Exact match
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  # Tolerate any value for key "dedicated"
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"

  # Stay up to 60s after NoExecute taint appears (e.g. not-ready / unreachable)
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 60
```

### 1.4 Clear examples

#### Dedicated GPU node

```bash
kubectl label nodes gpu-1 accelerator=nvidia
kubectl taint nodes gpu-1 dedicated=gpu:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: train
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  nodeSelector:
    accelerator: nvidia
  containers:
    - name: train
      image: my/trainer:1.0
```

Only pods that **tolerate** `dedicated=gpu:NoSchedule` (and usually select the GPU label) land on `gpu-1`. Normal workloads stay off it.

#### Soft preference (PreferNoSchedule)

```bash
kubectl taint nodes edge-1 workload=batch:PreferNoSchedule
```

Interactive / latency-sensitive pods without a toleration are steered elsewhere when possible; batch pods that tolerate can fill `edge-1`.

#### NoExecute + grace period

```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
```

Useful for apps that should fail over quickly instead of hanging on a dead node. DaemonSets and many system pods already ship with broader tolerations.

### 1.5 Best practices (taints / tolerations)

1. **Taint special nodes**, don’t only rely on `nodeSelector` — without a taint, any pod that matches labels (or none) can still land there when capacity is free elsewhere is exhausted.
2. **Pair taint + label + node affinity/selector** for dedicated pools (`gpu`, `ingress`, `database`).
3. Prefer **`NoSchedule`** for dedicated hardware; use **`PreferNoSchedule`** for soft isolation; use **`NoExecute`** carefully (eviction impact).
4. Document taint keys in a team convention (`dedicated=…`, `team=…`) — avoid ad-hoc keys per person.
5. Don’t remove the control-plane taint in production unless you intentionally run workloads there.
6. Remember: **toleration ≠ preference**. Tolerating a taint only *allows* the node; use affinity if the pod should *prefer* it.
7. For spot/preemptible nodes, combine a taint with tolerations only on workloads that can die and retry.

---

## 2. Node affinity (and node anti-patterns)

Node affinity expresses **hard** (`required`) or **soft** (`preferred`) rules against **node labels**.

### 2.1 Operators you’ll use

| Operator | Meaning |
|----------|---------|
| `In` | Label value in the list |
| `NotIn` | Label value not in the list |
| `Exists` | Label key present |
| `DoesNotExist` | Label key absent |
| `Gt` / `Lt` | Numeric compare (string-encoded integers) |

### 2.2 Required vs preferred

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - zone-a
                - zone-b
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
            - key: node.kubernetes.io/instance-type
              operator: In
              values:
                - m6i.large
```

| Rule | Behavior |
|------|----------|
| `requiredDuringSchedulingIgnoredDuringExecution` | Pod **must** match or it stays `Pending`. |
| `preferredDuringSchedulingIgnoredDuringExecution` | Scheduler scores matching nodes higher (`weight` 1–100). |

`IgnoredDuringExecution` means if labels change later, the pod is **not** evicted solely for that (unlike some taint `NoExecute` cases).

### 2.3 Example: require zone, prefer SSD nodes

```bash
kubectl label nodes worker-1 topology.kubernetes.io/zone=zone-a disk=ssd
kubectl label nodes worker-2 topology.kubernetes.io/zone=zone-a disk=hdd
kubectl label nodes worker-3 topology.kubernetes.io/zone=zone-b disk=ssd
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["zone-a", "zone-b"]
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disk
                    operator: In
                    values: ["ssd"]
      containers:
        - name: api
          image: my/api:1.2
```

### 2.4 `nodeSelector` vs node affinity

```yaml
# Simple equality only
nodeSelector:
  disk: ssd
```

Use **`nodeSelector`** for one/few exact labels; use **node affinity** for `In`/`NotIn`, ORs via multiple `nodeSelectorTerms`, and soft preferences.

### 2.5 Best practices (node affinity)

1. Standardize labels early (`topology.kubernetes.io/zone`, `node.kubernetes.io/instance-type`, custom `workload=…`).
2. Prefer **required** only when correctness depends on it (GPU, zone-bound volumes, license).
3. Prefer **preferred** for performance hints so the cluster still schedules under pressure.
4. Avoid over-constraining: too many `required` rules → widespread `Pending`.
5. Combine with **taints** for exclusive node pools (affinity alone does not keep others off the node).

---

## 3. Pod affinity and pod anti-affinity

These rules look at **already-running pods** (by label) relative to a **topology key** (usually a node or zone label).

### 3.1 Topology key

`topologyKey` is a **node label key**. The rule applies within groups of nodes that share the same value for that key.

| `topologyKey` | Typical intent |
|---------------|----------------|
| `kubernetes.io/hostname` | Same / different **node** |
| `topology.kubernetes.io/zone` | Same / different **zone** |
| `topology.kubernetes.io/region` | Same / different **region** |

### 3.2 Pod affinity — run *near* something

Co-locate a cache pod with the app that uses it (same node):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-sidecar-logic
  labels:
    app: store-cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - store-api
          topologyKey: kubernetes.io/hostname
  containers:
    - name: redis
      image: redis:7
```

This pod only schedules onto a node that already runs a pod labeled `app=store-api`.

Soft co-location (prefer same zone):

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: store-api
          topologyKey: topology.kubernetes.io/zone
```

### 3.3 Pod anti-affinity — spread / separate

Keep replicas of the same app off the same node (common HA pattern):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - payment
              topologyKey: kubernetes.io/hostname
      containers:
        - name: payment
          image: my/payment:2.0
```

With 3 replicas and `required` + `hostname`, you need **at least 3 schedulable nodes** or pods stay `Pending`.

Prefer spread but allow packing if the cluster is small:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: payment
          topologyKey: kubernetes.io/hostname
```

Zone-level anti-affinity (survive AZ loss):

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: payment
          topologyKey: topology.kubernetes.io/zone
```

### 3.4 Affinity + anti-affinity together

Example: API pods prefer to sit with a local cache, but not with another API replica on the same node:

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 40
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: store-cache
          topologyKey: kubernetes.io/hostname
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: store-api
          topologyKey: kubernetes.io/hostname
```

### 3.5 Best practices (pod affinity / anti-affinity)

1. Prefer **`preferredDuringScheduling…`** for anti-affinity in prod unless you truly must never co-locate (and have enough nodes).
2. Hard anti-affinity on `hostname` with high replica counts needs **≥ replicas** eligible nodes.
3. Use **zone** topology for multi-AZ HA; **hostname** for single-node failure isolation.
4. Ensure **labels on the pod template** match what selectors expect (`app`, `component`, etc.).
5. Inter-pod affinity can slow scheduling at large scale (scheduler must evaluate many pods)—keep rules purposeful.
6. For “spread evenly”, also consider **Topology Spread Constraints** (`topologySpreadConstraints`) — often clearer than stacking many anti-affinity rules.
7. Combine anti-affinity with **PodDisruptionBudgets** so voluntary drains don’t take all replicas down at once.

---

## 4. Topology spread constraints (related tool)

Often cleaner than heavy anti-affinity for even spread:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
    labelSelector:
      matchLabels:
        app: payment
```

| Field | Role |
|-------|------|
| `maxSkew` | Max imbalance across topology domains |
| `whenUnsatisfiable: DoNotSchedule` | Hard spread |
| `whenUnsatisfiable: ScheduleAnyway` | Soft spread |

Use spread constraints for balanced replicas; use anti-affinity when you need “never next to X”; use taints for “this node is reserved.”

---

## 5. Putting it together

### Dedicated ingress nodes

```bash
kubectl label nodes node-a node.role/ingress=true
kubectl taint nodes node-a dedicated=ingress:NoSchedule
```

```yaml
# ingress-nginx controller pod spec (excerpt)
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ingress"
    effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node.role/ingress
              operator: In
              values: ["true"]
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
          topologyKey: kubernetes.io/hostname
```

### Stateful DB: zone pin + no co-location of members

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["zone-a"]
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: postgres
        topologyKey: kubernetes.io/hostname
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

---

## 6. Quick decision guide

| Goal | Use |
|------|-----|
| Keep random pods off GPU / ingress / CP nodes | **Taint** + **toleration** on allowed pods |
| Must run only in zone-a / on SSD | **Node affinity** `required…` |
| Prefer big machines but allow fallback | **Node affinity** `preferred…` |
| Co-locate with another pod | **Pod affinity** |
| Don’t share a node/zone with my replicas | **Pod anti-affinity** or **topologySpreadConstraints** |
| Soft HA spread | `preferred` anti-affinity or spread `ScheduleAnyway` |
| Hard HA spread | `required` anti-affinity or spread `DoNotSchedule` |

---

## 7. Verify placement

```bash
kubectl get nodes --show-labels
kubectl describe node <node> | grep -E 'Taints|Labels'

kubectl get pods -o wide
kubectl describe pod <pod>    # Events: FailedScheduling, affinity/taint messages

# Why Pending?
kubectl get events --field-selector reason=FailedScheduling --sort-by='.lastTimestamp'
```

---

## References

- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Assigning Pods to Nodes (affinity)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)
