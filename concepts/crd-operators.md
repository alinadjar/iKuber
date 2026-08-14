# CRDs and Operators in Kubernetes

What **Custom Resource Definitions (CRDs)** and **Operators** are, when to use them, and a **simple end-to-end example** you can run on a kubeadm (or kind/minikube) cluster.

Related: [kubeadm-production-cluster.md](../kubeadm-production-cluster.md) · [developer-access-rbac.md](./developer-access-rbac.md)

Official: [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) · [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

---

## 0. Mental model

```text
Built-in API          →  Pod, Deployment, Service, …
Custom Resource (CR)  →  your API object (e.g. Backup, Website, Cache)
CRD                   →  registers that new API type with the apiserver
Controller / Operator →  watches CRs and makes cluster state match the spec
```

| Term | Meaning |
|------|---------|
| **CRD** | Schema that teaches Kubernetes a new kind (`kubectl get websites`) |
| **Custom Resource (CR)** | One instance of that kind (like one Deployment) |
| **Controller** | Loop: observe → diff → act until actual ≈ desired |
| **Operator** | Controller(s) + CRDs that encode **domain/operational knowledge** (install, upgrade, backup, failover…) |

A CRD **alone** only stores and validates objects. An **Operator** (or any controller) **does something** with them.

```text
Without controller:  kubectl apply Website  →  object sits in etcd  →  nothing runs
With operator:       kubectl apply Website  →  operator creates Deployment/Service/Ingress
```

---

## 1. What is a CRD?

A **CustomResourceDefinition** extends the Kubernetes API:

- New group / version / kind (e.g. `demo.example.com/v1`, kind `Website`)
- Optional OpenAPI schema (required fields, types)
- Objects stored in etcd like built-in resources
- Usable with `kubectl`, RBAC, admission, GitOps

Example shape:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.demo.example.com    # <plural>.<group>
spec:
  group: demo.example.com
  scope: Namespaced                  # or Cluster
  names:
    plural: websites
    singular: website
    kind: Website
    shortNames: ["web"]
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
```

After the CRD exists:

```bash
kubectl api-resources | grep website
kubectl explain website
kubectl get websites -A
```

---

## 2. What is an Operator?

An **Operator** is a (set of) controller(s) that:

1. **Watch** one or more CR types (and related objects).  
2. **Reconcile** desired state (`spec`) with live cluster state.  
3. Often update **`status`** on the CR (`Ready`, URLs, errors).  
4. Encode human runbooks: upgrade order, backups, users, TLS, scaling rules.

Famous examples: **Prometheus Operator**, **Tigera/Calico Operator**, **cert-manager**, **PostgreSQL / Kafka / Redis operators**, **Cilium** (agent + CRDs).

```text
You:     kubectl apply -f Website.yaml   (spec.image, spec.replicas)
Operator: creates Deployment + Service; sets status.readyReplicas
You:     kubectl apply  (replicas: 3)
Operator: scales Deployment to 3
You:     kubectl delete Website
Operator: deletes owned Deployment/Service (if using ownerReferences)
```

### Operator ≠ only “Go binary”

| Style | Description |
|-------|-------------|
| **Go + controller-runtime / Kubebuilder / Operator SDK** | Most common for production |
| **Ansible / Helm Operator** | Operator SDK can wrap playbooks/charts |
| **Shell / Python / any language** | Fine for learning or small tools |
| **CronJob + script** | Weak “operator”; usually not continuous reconcile |

---

## 3. When should you use a CRD / Operator?

### Use them when

| Situation | Why | Detailed example |
|-----------|-----|------------------|
| You repeat the same multi-step ops for an app | Encode once in reconcile logic | [§4.1 AppRelease](#41-situation-a--repeat-the-same-multi-step-ops--encode-in-reconcile) |
| Users need a **simple API** (“give me a cache”) not raw Deployments | CR hides complexity | [§4.2 Cache](#42-situation-b--simple-api-give-me-a-cache--cr-hides-complexity) |
| Lifecycle is non-trivial (ordered upgrade, backup, failover, users, CRDs of their own) | Operator owns the runbook | |
| You want GitOps of **intent** (`Website`) not 20 YAML files | One object drives many children | |
| Platform productizes a service (DB-as-a-Service inside the cluster) | Clear tenancy + status | [§4.2](#42-situation-b--simple-api-give-me-a-cache--cr-hides-complexity) similar |

### Prefer **not** to use them when

| Situation | Prefer instead |
|-----------|----------------|
| One static app deploy | Deployment + Service (+ Helm chart) |
| Simple config packaging | Helm / Kustomize |
| One-shot job | Job / CronJob |
| Only need validation of a ConfigMap | Validating admission / Kyverno policy |
| No continuous reconciliation needed | Plain manifests |

**Rule of thumb:** if the “product” is a **long-lived service with operational knowledge**, an Operator fits. If you only need to **ship YAML**, use Helm/Kustomize.

### CRD without Operator?

Valid for:

- Storing config that **another** tool reads (CI, external controller)
- Aggregating inventory for humans (`kubectl get`)

Usually you still want *some* controller eventually, or the CR is just documentation in etcd.

---

## 4. Detailed scenario examples

Two common reasons to build an Operator, worked through with concrete APIs and reconcile steps.

---

### 4.1 Situation A — Repeat the same multi-step ops → encode in reconcile

#### The pain (without an Operator)

Your team ships an internal app **payments-api** every week. The human runbook is always the same:

```text
1. Scale old Deployment to N (or leave it)
2. Apply new image tag
3. Wait for rollout
4. Run DB migrate Job (must finish before traffic shift)
5. Switch Service selector / Ingress to new ReplicaSet
6. Smoke-check /healthz
7. Scale down old version
8. On failure: roll back image + keep DB migrate notes
```

People forget step 4, run migrate twice, or skip the smoke check. Copy-pasted scripts drift per environment.

#### The Operator idea

Expose one CR: **`AppRelease`**. Engineers only declare *what* they want; the controller owns *how* every time.

```yaml
apiVersion: apps.example.com/v1
kind: AppRelease
metadata:
  name: payments-api
  namespace: payments-prod
spec:
  image: ghcr.io/acme/payments-api:1.14.2
  replicas: 3
  migrate:
    enabled: true
    command: ["python", "manage.py", "migrate"]
  healthCheckPath: /healthz
  rollbackOnFailure: true
```

#### What the reconcile loop encodes (the old runbook)

```text
Observe AppRelease spec
  │
  ├─ Ensure Deployment exists with spec.image / replicas
  ├─ If migrate.enabled and image changed since last success:
  │     create Job migrate-<hash> (owned by the CR)
  │     wait until Job Complete ──or── Failed
  │     if Failed && rollbackOnFailure → revert Deployment image; set status=Degraded; stop
  ├─ Wait Deployment Available (rollout)
  ├─ Probe healthCheckPath (Exec/HTTP via a probe Pod or hit Service)
  ├─ Update status:
  │     conditions: Progressing / Available / Degraded
  │     observedGeneration, currentImage, lastMigrateJob
  └─ Requeue until Available=True
```

Idempotent rules that humans get wrong:

| Rule | Encoded behavior |
|------|------------------|
| Migrate once per image | Annotation `apps.example.com/last-migrated-image` |
| Don’t start migrate while previous Job running | Check Job status first |
| Don’t mark Available if probe fails | Keep `Available=False`; optionally rollback |
| Delete CR | Cascade delete Job/Deployment via ownerReferences |

#### CRD sketch

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: appreleases.apps.example.com
spec:
  group: apps.example.com
  scope: Namespaced
  names:
    plural: appreleases
    singular: apprelease
    kind: AppRelease
    shortNames: ["apprel"]
  versions:
    - name: v1
      served: true
      storage: true
      subresources:
        status: {}
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["image"]
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  default: 2
                migrate:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    command:
                      type: array
                      items:
                        type: string
                healthCheckPath:
                  type: string
                  default: "/healthz"
                rollbackOnFailure:
                  type: boolean
                  default: true
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Migrating", "RollingOut", "Available", "Degraded"]
                currentImage:
                  type: string
                lastMigrateJob:
                  type: string
                message:
                  type: string
      additionalPrinterColumns:
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Image
          type: string
          jsonPath: .status.currentImage
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

#### Day-2 usage (what humans run)

```bash
# Install CRD + deploy your AppRelease operator once (platform)
kubectl apply -f crd-apprelease.yaml
kubectl apply -f operator-apprelease.yaml

# Every release: only change intent
kubectl apply -f - <<'EOF'
apiVersion: apps.example.com/v1
kind: AppRelease
metadata:
  name: payments-api
  namespace: payments-prod
spec:
  image: ghcr.io/acme/payments-api:1.14.2
  replicas: 3
  migrate:
    enabled: true
    command: ["python", "manage.py", "migrate"]
  healthCheckPath: /healthz
  rollbackOnFailure: true
EOF

# Watch the runbook execute itself
kubectl get apprelease payments-api -n payments-prod -w
kubectl get jobs,deploy -n payments-prod -l app=payments-api

# Next week: bump image only
kubectl patch apprelease payments-api -n payments-prod --type merge \
  -p '{"spec":{"image":"ghcr.io/acme/payments-api:1.15.0"}}'
```

#### Why this fits “multi-step ops”

- The **same** migrate → roll → probe → rollback path runs in every env.  
- GitOps stores one object per app, not a checklist wiki.  
- Failures become `status.phase=Degraded` + Events, not a half-finished SSH session.  
- New team members don’t need to memorize the runbook order.

Without an Operator you’d keep a brittle CI script that isn’t continuously reconciling (it won’t notice someone manually scaled the Deployment or deleted the migrate Job).

---

### 4.2 Situation B — Simple API (“give me a cache”) → CR hides complexity

#### The pain (without an Operator)

Developers ask for Redis. Platform pastes a pile of YAML every time:

```text
Deployment (image, resources, probes, anti-affinity)
Service (ClusterIP)
ConfigMap (redis.conf)
Secret (password)
PVC (size, StorageClass)
NetworkPolicy (only allow namespace X)
PodDisruptionBudget
optional: metrics ServiceMonitor, password rotation note
```

Developers shouldn’t need to know StorageClass names, anti-affinity, or redis.conf knobs. Mistakes: no persistence, open NetworkPolicy, weak probes, wrong memory limits.

#### The Operator idea

Expose one CR: **`Cache`**. User specifies **intent**; operator materializes the stack.

```yaml
apiVersion: cache.example.com/v1
kind: Cache
metadata:
  name: sessions
  namespace: payments-dev
spec:
  engine: redis
  size: small          # small|medium|large → CPU/memory/PVC map
  version: "7.2"
  persistence: true
  passwordSecretRef:
    name: sessions-redis-auth
    key: password
  allowFromNamespaces:
    - payments-dev
```

That single object is the “simple API.” Developers never touch raw Deployments.

#### What the operator creates (hidden complexity)

| Child object | How operator decides |
|--------------|----------------------|
| Secret | Generate password if `passwordSecretRef` missing; or use referenced Secret |
| ConfigMap | `redis.conf` from template (`maxmemory` from size profile) |
| PVC | `1Gi` / `10Gi` / `50Gi` from `size`; StorageClass from platform default |
| Deployment/StatefulSet | Image `redis:7.2`, resources from size profile, probes, securityContext |
| Service | ClusterIP `sessions-redis:6379` |
| NetworkPolicy | Ingress only from `allowFromNamespaces` |
| PDB | `minAvailable: 1` if replicas ≥ 2 |
| status | `endpoint`, `phase`, `passwordSecretName` |

Size profile example encoded in operator config (not in user YAML):

```text
small  → 250m CPU, 256Mi RAM, 1Gi PVC, 1 replica
medium → 500m CPU, 1Gi RAM,  10Gi PVC, 1 replica
large  → 1 CPU,    4Gi RAM,  50Gi PVC, 2 replicas + anti-affinity
```

#### CRD sketch

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: caches.cache.example.com
spec:
  group: cache.example.com
  scope: Namespaced
  names:
    plural: caches
    singular: cache
    kind: Cache
    shortNames: ["ch"]
  versions:
    - name: v1
      served: true
      storage: true
      subresources:
        status: {}
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["engine", "size"]
              properties:
                engine:
                  type: string
                  enum: ["redis"]
                size:
                  type: string
                  enum: ["small", "medium", "large"]
                version:
                  type: string
                  default: "7.2"
                persistence:
                  type: boolean
                  default: true
                passwordSecretRef:
                  type: object
                  properties:
                    name:
                      type: string
                    key:
                      type: string
                      default: password
                allowFromNamespaces:
                  type: array
                  items:
                    type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                endpoint:
                  type: string
                passwordSecretName:
                  type: string
                message:
                  type: string
      additionalPrinterColumns:
        - name: Size
          type: string
          jsonPath: .spec.size
        - name: Endpoint
          type: string
          jsonPath: .status.endpoint
        - name: Phase
          type: string
          jsonPath: .status.phase
```

#### Day-2 usage (developer experience)

```bash
# Platform installs Cache CRD + cache-operator once
kubectl apply -f crd-cache.yaml
kubectl apply -f operator-cache.yaml

# Developer (or GitOps) — only the simple API
kubectl apply -f - <<'EOF'
apiVersion: cache.example.com/v1
kind: Cache
metadata:
  name: sessions
  namespace: payments-dev
spec:
  engine: redis
  size: small
  version: "7.2"
  persistence: true
  allowFromNamespaces:
    - payments-dev
EOF

# Wait until ready
kubectl get cache sessions -n payments-dev -w
# NAME       SIZE    ENDPOINT                       PHASE
# sessions   small   sessions-redis.payments-dev:6379   Ready

# App consumes the endpoint from status (or fixed naming convention)
kubectl get cache sessions -n payments-dev -o jsonpath='{.status.endpoint}{"\n"}'
kubectl get cache sessions -n payments-dev -o jsonpath='{.status.passwordSecretName}{"\n"}'

# Grow later — still one field
kubectl patch cache sessions -n payments-dev --type merge -p '{"spec":{"size":"medium"}}'
# Operator resizes resources / PVC policy per its rules; user did not edit Deployment

# Leave
kubectl delete cache sessions -n payments-dev
# Operator deletes owned Deploy/SVC/PVC/NetworkPolicy (per retention policy)
```

Example app ConfigMap using the contract:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-api
  namespace: payments-dev
data:
  REDIS_URL: "redis://sessions-redis.payments-dev.svc:6379"
# password from Secret mounted via passwordSecretName from Cache status
```

#### Reconcile outline (what “hides complexity”)

```text
On Cache create/update:
  1. Resolve size → resources + PVC size + replica count
  2. Ensure auth Secret exists
  3. Ensure ConfigMap redis.conf
  4. Ensure PVC if persistence=true
  5. Ensure StatefulSet/Deployment + Service
  6. Ensure NetworkPolicy for allowFromNamespaces (+ DNS egress)
  7. Ensure PDB if needed
  8. Set status.endpoint = "<name>-redis.<ns>.svc:6379"
  9. Set status.phase = Ready when pods ready

On Cache delete:
  Delete owned objects (optionally keep PVC if spec says retain)
```

#### Why this fits “simple API”

| User thinks in | Operator thinks in |
|----------------|--------------------|
| `size: small` | requests/limits, PVC, redis.conf `maxmemory` |
| `allowFromNamespaces` | NetworkPolicy podSelectors / namespaceSelectors |
| `persistence: true` | PVC + volumeMount + rewrite/AOF settings |
| `kubectl get cache` | endpoint + phase for app wiring |

Developers never choose the Redis image digest, StorageClass, or anti-affinity rules—you change those **once** in the operator when platform standards evolve, and all `Cache` objects pick it up on reconcile.

Contrast: a Helm chart still exposes many values and doesn’t continuously fix drift if someone edits the Deployment by hand. An Operator **reverts/repairs** toward `spec`.

---

### 4.3 Side-by-side

| | **4.1 AppRelease** | **4.2 Cache** |
|--|--------------------|---------------|
| Primary goal | Encode a **runbook** (order, migrate, rollback) | Hide **infrastructure complexity** |
| User mental model | “Ship this image safely” | “Give me a cache” |
| Typical author | App platform / SRE | Internal developer platform |
| Spec focus | image, migrate, health, rollback | size, persistence, ACL namespaces |
| Failure UX | `phase=Degraded` + auto rollback | `phase=Failed` + message (e.g. PVC pending) |

Both use the same pattern: **CRD = API**, **Operator = brain**.

---

## 5. Simple example: “Website” Operator (learning)

Goal: a CR `Website` with `spec.image` and `spec.replicas`. A small controller ensures a **Deployment** and **ClusterIP Service** exist.

You will:

1. Install the CRD  
2. Apply a sample `Website`  
3. Run a tiny Python controller (easy to read)  
4. See Deployment/Service created and updated  

> Production operators are usually Go + Kubebuilder. Python here maximizes clarity.

### 5.1 Prerequisites

```bash
kubectl get nodes
# python3 + pip on an admin machine that can reach the API
pip3 install --user kubernetes
```

Use a kubeconfig with rights to create CRDs, Deployments, Services (e.g. platform admin in a lab namespace).

### 5.2 Create the CRD

```bash
kubectl apply -f - <<'EOF'
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.demo.example.com
spec:
  group: demo.example.com
  scope: Namespaced
  names:
    plural: websites
    singular: website
    kind: Website
    shortNames: ["web"]
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["image"]
              properties:
                image:
                  type: string
                  description: "Container image for the site"
                replicas:
                  type: integer
                  minimum: 1
                  default: 1
            status:
              type: object
              properties:
                readyReplicas:
                  type: integer
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
EOF

kubectl get crd websites.demo.example.com
kubectl explain website.spec
```

### 5.3 Create a Website (custom resource)

```bash
kubectl create namespace demo

kubectl apply -f - <<'EOF'
apiVersion: demo.example.com/v1
kind: Website
metadata:
  name: hello
  namespace: demo
spec:
  image: nginx:1.25
  replicas: 2
EOF

kubectl get websites -n demo
kubectl get website hello -n demo -o yaml
```

At this point **nothing else appears**—no controller is running yet:

```bash
kubectl get deploy,svc -n demo
# empty (or unrelated)
```

### 5.4 Minimal Operator (Python reconcile loop)

Save as `website-operator.py`:

```python
#!/usr/bin/env python3
"""Tiny Website operator — for learning only (not HA / not production)."""
import time
from kubernetes import client, config, watch

GROUP = "demo.example.com"
VERSION = "v1"
PLURAL = "websites"

def main():
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    api = client.CustomObjectsApi()
    apps = client.AppsV1Api()
    core = client.CoreV1Api()

    print("Website operator started; watching demo.example.com/v1 websites")
    while True:
        try:
            # List-and-reconcile (simple); production uses watch + workqueue
            websites = api.list_cluster_custom_object(GROUP, VERSION, PLURAL)
            for site in websites.get("items", []):
                reconcile(api, apps, core, site)
        except Exception as exc:
            print(f"reconcile error: {exc}")
        time.sleep(5)


def reconcile(api, apps, core, site):
    meta = site["metadata"]
    spec = site.get("spec") or {}
    name = meta["name"]
    ns = meta["namespace"]
    image = spec.get("image")
    replicas = int(spec.get("replicas") or 1)
    if not image:
        return

    labels = {"app": name, "managed-by": "website-operator"}

    deploy_body = client.V1Deployment(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=ns,
            labels=labels,
            owner_references=[
                client.V1OwnerReference(
                    api_version=f"{GROUP}/{VERSION}",
                    kind="Website",
                    name=name,
                    uid=meta["uid"],
                    controller=True,
                    block_owner_deletion=True,
                )
            ],
        ),
        spec=client.V1DeploymentSpec(
            replicas=replicas,
            selector=client.V1LabelSelector(match_labels={"app": name}),
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(labels={"app": name}),
                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name="web",
                            image=image,
                            ports=[client.V1ContainerPort(container_port=80)],
                        )
                    ]
                ),
            ),
        ),
    )

    try:
        apps.read_namespaced_deployment(name, ns)
        apps.patch_namespaced_deployment(name, ns, deploy_body)
    except client.exceptions.ApiException as e:
        if e.status == 404:
            apps.create_namespaced_deployment(ns, deploy_body)
        else:
            raise

    svc_body = client.V1Service(
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=ns,
            labels=labels,
            owner_references=deploy_body.metadata.owner_references,
        ),
        spec=client.V1ServiceSpec(
            selector={"app": name},
            ports=[client.V1ServicePort(port=80, target_port=80)],
        ),
    )
    try:
        core.read_namespaced_service(name, ns)
        # keep existing ClusterIP — skip patch for simplicity
    except client.exceptions.ApiException as e:
        if e.status == 404:
            core.create_namespaced_service(ns, svc_body)
        else:
            raise

    # Best-effort status
    try:
        dep = apps.read_namespaced_deployment(name, ns)
        ready = dep.status.ready_replicas or 0
        api.patch_namespaced_custom_object_status(
            GROUP, VERSION, ns, PLURAL, name,
            {"status": {"readyReplicas": ready}},
        )
    except Exception as exc:
        print(f"status update skipped for {ns}/{name}: {exc}")

    print(f"reconciled Website {ns}/{name} image={image} replicas={replicas}")


if __name__ == "__main__":
    main()
```

Run it (lab: from your laptop):

```bash
chmod +x website-operator.py
python3 website-operator.py
```

In another terminal:

```bash
kubectl get deploy,svc,pods -n demo -o wide
kubectl get website hello -n demo -o yaml
```

### 5.5 Use it — change desired state

```bash
# Scale via the CR (operator reconciles Deployment)
kubectl patch website hello -n demo --type merge -p '{"spec":{"replicas":3}}'
kubectl get deploy hello -n demo

# Change image
kubectl patch website hello -n demo --type merge -p '{"spec":{"image":"nginx:1.26"}}'
kubectl get pods -n demo -w
```

### 5.6 Delete

```bash
kubectl delete website hello -n demo
# OwnerReference → Deployment/Service garbage-collected
kubectl get deploy,svc -n demo
```

Stop the Python process when done. Remove the CRD only if you want a full cleanup:

```bash
kubectl delete crd websites.demo.example.com
kubectl delete ns demo
```

### 5.7 Optional: run the operator *inside* the cluster

For a real deployment you would:

1. Build an image containing the controller.  
2. Create a ServiceAccount + Role/RoleBinding (list/watch Websites; create/patch Deployments/Services).  
3. Deploy a Deployment of the operator.  
4. Use leader election for HA.

That packaging is what Operator SDK / Kubebuilder generate for you in Go.

---

## 6. Production operator checklist

| Topic | Practice |
|-------|----------|
| **API design** | Clear `spec` / `status`; version (`v1alpha1` → `v1`); don’t break fields casually |
| **Ownership** | `ownerReferences` so deletes cascade |
| **Idempotent reconcile** | Safe to run repeatedly |
| **Status & conditions** | `Ready`, `Progressing`, human-readable messages |
| **RBAC** | Least privilege for the operator SA |
| **HA** | Leader election; ≥1 replica |
| **Upgrade** | CRD conversion webhooks if changing versions |
| **Observability** | Metrics, logs, events on the CR |
| **Tests** | Unit + envtest / integration |
| **Don’t reinvent** | Prefer existing operators (cert-manager, DB operators) when they fit |

---

## 7. How this relates to tools you already use

| Component | Role |
|-----------|------|
| **Calico / Tigera Operator** | CRDs (`Installation`, …) + operator install CNI |
| **Cilium** | CRDs (`CiliumNetworkPolicy`, …) + agent/operator |
| **MetalLB** | CRDs (`IPAddressPool`, `L2Advertisement`) + controller/speaker |
| **cert-manager** | `Certificate` CRD + controllers request/renew TLS |
| **Prometheus Operator** | `ServiceMonitor` / `Prometheus` CRs manage monitoring stacks |

You already consume Operators when you `kubectl apply` those CRs—the pattern is the same as the Website example, at larger scale.

---

## 8. Kubebuilder / Operator SDK (next step)

When you outgrow a script:

```bash
# Example sketch — follow current Kubebuilder quickstart for your Go version
# go install sigs.k8s.io/kubebuilder/v4@latest
# kubebuilder init --domain example.com --repo example.com/website
# kubebuilder create api --group demo --version v1 --kind Website
# make manifests generate run
```

Operator SDK similarly scaffolds Go/Ansible/Helm operators and OLM bundles for lifecycle on Operator Lifecycle Manager.

---

## 9. Quick decision guide

```text
Need to package YAML once?          → Helm / Kustomize
Need continuous ops knowledge?      → Operator + CRD   (see §4.1 AppRelease)
Need a simple product API for users? → Operator + CRD   (see §4.2 Cache)
Need only a new API object store?   → CRD (+ maybe external consumer)
Need one-shot task?                 → Job
```

```text
kubectl apply CRD
  → kubectl apply CR
  → Operator reconciles children
  → kubectl get CR  (status)
  → update spec → reconcile again
  → delete CR → cleanup owned objects
```

---

## References

- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Extend the Kubernetes API with CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubebuilder](https://book.kubebuilder.io/)
- [Operator SDK](https://sdk.operatorframework.io/)
- [kubernetes Python client](https://github.com/kubernetes-client/python)
