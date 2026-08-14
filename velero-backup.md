# Velero Backup from Zero to Hero

Install and operate **Velero** for Kubernetes backup/restore/migration, how to **move backups to another cluster**, production practices, and **alternatives**.

Related: [kubeadm-production-cluster.md](./kubeadm-production-cluster.md) · [concepts/argocd-production.md](./concepts/argocd-production.md) · [concepts/certificate-renewal.md](./concepts/certificate-renewal.md) · [external-etcd-tarball-systemd.md](./external-etcd-tarball-systemd.md)

Official: [Velero docs](https://velero.io/docs/) · [GitHub](https://github.com/vmware-tanzu/velero)

---

## 0. What Velero is (and is not)

| Velero does | Velero does not replace |
|-------------|-------------------------|
| Backup / restore **Kubernetes API objects** (YAML in object storage) | Application-consistent DB strategy by itself (use hooks / native DB backup) |
| Backup **persistent volumes** via CSI snapshots or restic/kopia file-level | etcd-only control-plane DR ([certificate-renewal](./concepts/certificate-renewal.md) / etcd snapshots still needed) |
| Scheduled backups, namespaces filters, restore to same or **another** cluster | GitOps as source of truth for desired config (pair with Argo CD) |
| Migrate workloads between clusters (with planning) | Magic for every StorageClass / cross-cloud disk type |

```text
Cluster A                          Object storage (S3 / MinIO / Azure / GCS / …)
  Velero backup  →  backup tarball + volume snapshots/data  →  bucket
Cluster B
  Velero (same bucket)  →  restore
```

**Two volume strategies:**

| Method | How | Good for |
|--------|-----|----------|
| **CSI snapshot** | VolumeSnapshot → snapshot in cloud/storage | Cloud disks, CSI with snapshot support; fast |
| **File-level (Kopia / Restic)** | Pod mounts FS, uploads files to object storage | NFS, local PVs, cross-cluster when snapshots aren’t portable |

Modern Velero defaults toward **Kopia** for file-level uploads (restic still appears in older docs).

---

## 1. Architecture

```text
velero CLI (laptop/CI)
    ↓
Velero Deployment + node-agent DaemonSet (optional, for FS backup)
    ↓
BackupStorageLocation (BSL)  →  S3-compatible bucket
VolumeSnapshotLocation (VSL) →  CSI / cloud snapshot API
```

Objects stored per backup typically include:

- Cluster/namespace resources (serialized)
- Backup metadata / logs
- Volume snapshot refs **or** Kopia/restic repository data

---

## 2. Prerequisites

- [ ] Kubernetes cluster with Velero-supported version  
- [ ] Object storage bucket + credentials  
- [ ] For CSI: VolumeSnapshot CRDs + CSI driver with snapshot support  
- [ ] For FS backup: enough node resources; DaemonSet allowed  
- [ ] Network: Velero pods → object storage  

```bash
kubectl get nodes
kubectl get volumesnapshotclass 2>/dev/null || echo "No VolumeSnapshotClass yet"
kubectl api-resources | grep -i volumesnapshot || true
```

---

## 3. Install from zero

### 3.1 Install Velero CLI

```bash
# Pin a release: https://github.com/vmware-tanzu/velero/releases
VER=v1.15.0
OS=linux
ARCH=amd64

curl -fsSL -o /tmp/velero.tgz \
  https://github.com/vmware-tanzu/velero/releases/download/${VER}/velero-${VER}-${OS}-${ARCH}.tar.gz
tar xz -C /tmp -f /tmp/velero.tgz
sudo install /tmp/velero-${VER}-${OS}-${ARCH}/velero /usr/local/bin/velero
velero version --client-only
```

### 3.2 Prepare object storage credentials (S3 example)

```bash
# credentials-velero file (AWS / MinIO style)
cat > credentials-velero <<'EOF'
[default]
aws_access_key_id=YOUR_ACCESS_KEY
aws_secret_access_key=YOUR_SECRET_KEY
EOF
```

MinIO / on-prem S3: same format; you’ll set `s3Url` below.

### 3.3 Install Velero server (AWS S3 example)

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:${VER} \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=eu-central-1 \
  --snapshot-location-config region=eu-central-1 \
  --use-node-agent \
  --uploader-type kopia
```

### 3.4 Install for MinIO / generic S3

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:${VER} \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config \
region=minio,s3ForcePathStyle="true",s3Url=https://minio.example.com \
  --use-node-agent \
  --uploader-type kopia
```

`--use-volume-snapshots=false` when you rely on **file-level** backup only (common for cross-cluster / NFS).

### 3.5 CSI snapshots (when available)

1. Install snapshot controller + CRDs if missing.  
2. Create `VolumeSnapshotClass` for your CSI driver.  
3. Install Velero CSI plugin and label the VolumeSnapshotClass for Velero (per current Velero CSI docs).  

```bash
velero install ... \
  --plugins velero/velero-plugin-for-aws:${VER},velero/velero-plugin-for-csi:${VER} \
  --features=EnableCSI \
  ...
```

### 3.6 Verify install

```bash
kubectl -n velero get pods
velero backup-location get
velero snapshot-location get
velero version
```

All BSL should be `Available` / `Phase: Available`.

---

## 4. Backup basics (hero path)

### 4.1 Backup one namespace

```bash
velero backup create payments-ns \
  --include-namespaces payments-prod \
  --wait

velero backup describe payments-ns --details
velero backup logs payments-ns
```

### 4.2 Backup with volume data (FS / Kopia)

Annotate pods (or use backup with default opt-in/out policies per version):

```bash
# Opt-in style (common pattern — confirm for your Velero version)
kubectl -n payments-prod annotate pod/<pod-name> \
  backup.velero.io/backup-volumes=data --overwrite

# Or annotate the pod template on the Deployment/StatefulSet so new pods get it
```

Many setups use:

```bash
velero backup create payments-full \
  --include-namespaces payments-prod \
  --default-volumes-to-fs-backup \
  --wait
```

(`--default-volumes-to-fs-backup` backs up all volumes via node-agent unless excluded.)

Exclude volumes:

```bash
kubectl -n payments-prod annotate pod/<pod> \
  backup.velero.io/backup-volumes-excludes=tmp --overwrite
```

### 4.3 Backup whole cluster (careful)

```bash
velero backup create full-cluster-$(date +%F) \
  --exclude-namespaces kube-system,velero,kube-public,kube-node-lease \
  --wait
```

Often better: **namespace-scoped** or label-selected backups for app teams; separate platform backups.

```bash
velero backup create by-label \
  --selector app=payments \
  --include-namespaces payments-prod \
  --wait
```

### 4.4 Schedules

```bash
velero schedule create payments-daily \
  --schedule="0 2 * * *" \
  --include-namespaces payments-prod \
  --ttl 720h0m0s \
  --default-volumes-to-fs-backup

velero schedule get
velero schedule describe payments-daily
```

`--ttl` = how long backup is retained in storage / CR.

### 4.5 Hooks (app-consistent DB)

```yaml
# Example annotations on a pod template
pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "pg_dump ... > /data/dump.sql"]'
pre.hook.backup.velero.io/container: postgres
post.hook.backup.velero.io/command: '["/bin/bash", "-c", "rm -f /data/dump.sql"]'
```

Prefer **native DB backups** (WAL, snapshots) for databases; Velero hooks are a complement, not a full DBA strategy.

### 4.6 Restore on the **same** cluster

```bash
# Disaster in one namespace
kubectl delete ns payments-prod   # only if intentional!

velero restore create payments-restore-1 \
  --from-backup payments-full \
  --wait

velero restore describe payments-restore-1
kubectl get all -n payments-prod
```

Partial restore:

```bash
velero restore create restore-deploys \
  --from-backup payments-full \
  --include-resources deployments,services,configmaps \
  --namespace-mappings payments-prod:payments-prod
```

---

## 5. Move backup → restore on **another** cluster (detailed)

This is the common DR / migration path.

### 5.1 Mental model

```text
Cluster A                          Shared bucket
  backup "payments-full"  ──────►  s3://velero-backups/...
                                      │
Cluster B  (Velero installed)  ◄──────┘
  BSL points to SAME bucket
  velero restore --from-backup payments-full
```

Velero does **not** require you to `scp` files if both clusters share the object store. You only “move” the backup if the destination uses a **different** bucket (then replicate objects).

### 5.2 Checklist before cross-cluster restore

| Item | Why |
|------|-----|
| Same (or compatible) Velero major version | Metadata format |
| Destination Velero BSL → **same bucket/prefix** (or replicated copy) | Must see the backup object |
| Credentials on B work | `velero backup-location get` Available |
| Matching / mapped StorageClasses | PVC restore fails if SC missing |
| CSI snapshots **rarely portable** across clouds/clusters | Prefer **FS backup (Kopia)** for migration |
| Ingress / LB / DNS / OIDC differ | Expect to re-point after restore |
| Nodes / CNI / CSI drivers ready on B | Workloads must schedule |
| GitOps (Argo CD) | May fight restored objects — pause sync or restore into unmanaged NS first |

### 5.3 Step-by-step (shared bucket — recommended)

**On cluster A (source)**

```bash
# Prefer FS volume backup for portability
velero backup create migrate-payments-$(date +%F) \
  --include-namespaces payments-prod \
  --default-volumes-to-fs-backup \
  --wait

velero backup describe migrate-payments-2026-08-14 --details
# Confirm phase Completed; volumes backed up as expected
```

**On cluster B (destination)**

```bash
# 1) Install Velero with the SAME bucket/config (different cluster kubeconfig)
export KUBECONFIG=~/.kube/cluster-b.conf

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:${VER} \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=eu-central-1 \
  --use-volume-snapshots=false \
  --use-node-agent \
  --uploader-type kopia

# 2) Wait until BSL available — Velero syncs backup list from object storage
kubectl -n velero get backupstoragelocations
velero backup-location get

# 3) Force sync / wait — backups from A should appear
velero backup get
# Should list migrate-payments-2026-08-14 created on A

# 4) (Optional) Read-only BSL on B so B doesn't write accidental backups to shared bucket
kubectl -n velero patch backupstoragelocation default --type merge \
  -p '{"spec":{"accessMode":"ReadOnly"}}'
```

**Restore on B**

```bash
# Same namespace name
velero restore create migrate-payments-restore \
  --from-backup migrate-payments-2026-08-14 \
  --wait

# OR map into a new namespace
velero restore create migrate-payments-restore \
  --from-backup migrate-payments-2026-08-14 \
  --namespace-mappings payments-prod:payments-dr \
  --wait

velero restore describe migrate-payments-restore --details
kubectl get all,pvc -n payments-dr
```

### 5.4 If buckets differ — replicate objects

```text
Bucket A (source)  --[mc mirror / aws s3 sync / native replication]-->  Bucket B
Then point cluster B Velero BSL at Bucket B and restore.
```

```bash
# Example MinIO client
mc alias set src https://minio-a.example.com KEY SECRET
mc alias set dst https://minio-b.example.com KEY SECRET
mc mirror --preserve src/velero-backups dst/velero-backups
```

Preserve prefixes Velero expects (`backups/`, `kopia/`, `restics/`, … — names vary by version/uploader). After sync:

```bash
velero backup-location get
velero backup get
```

### 5.5 CSI snapshot backups across clusters

Usually **do not** work unless:

- Same cloud account/region and snapshot is visible, **and**  
- CSI on B can create volume from that snapshot ID, **and**  
- Velero CSI plugin can map it  

For **cluster A → cluster B** or **cloud → on-prem**, use **`--default-volumes-to-fs-backup`** (or equivalent opt-in) so data lives in the **object store**, not as a non-portable disk snapshot.

### 5.6 Resource transforms (advanced)

When restoring to a different environment, rename StorageClasses, ingress classes, etc.:

```bash
# Velero restore resource modifiers / policies — see current docs for
# "resource modifier" ConfigMap or --restore-resource-modifiers
# Example ideas:
#  - storageClassName: premium-ssd → standard-csi
#  - ingressClassName: nginx → traefik
```

Check your Velero version for **Restore resource modifiers** / JSON patches.

### 5.7 After restore on the new cluster

```bash
kubectl get pods,pvc,ing -n payments-dr
kubectl describe pvc -n payments-dr
# Fix: secrets that referenced old KMS, image pull secrets, Ingress DNS, cert-manager certs
# Re-enable Argo CD Application with corrected destination
# Run app smoke tests / DB migration checks
```

Update:

- DNS / LoadBalancer / MetalLB IPs  
- OIDC redirect URLs  
- External Secrets / vault paths  
- Monitoring ServiceMonitors (cluster label)  

### 5.8 Export metadata only (optional)

If you only need YAML (no PVs):

```bash
velero backup download migrate-payments-2026-08-14 -o payments-backup.tar.gz
tar tzf payments-backup.tar.gz | head
# Contains resource JSON; useful for forensics — restore still goes through Velero + bucket for full PV data
```

---

## 6. Day-2 operations

```bash
velero backup get
velero restore get
velero schedule get
velero backup delete BACKUP_NAME
velero uninstall   # destructive — remove server components

# Logs
velero backup logs BACKUP
velero restore logs RESTORE
kubectl -n velero logs deploy/velero
kubectl -n velero logs -l name=node-agent   # DS name may vary
```

Backup failure debug:

```bash
velero backup describe BACKUP --details
kubectl -n velero get podvolumesbackups -o yaml
kubectl -n velero get events --sort-by='.lastTimestamp' | tail
```

---

## 7. Production best practices

| Practice | Detail |
|----------|--------|
| **Scope** | Prefer namespace / app backups over “everything” |
| **Schedules** | Daily + weekly; TTL aligned with compliance |
| **Test restores** | Quarterly restore to a **DR cluster** or empty NS |
| **Object lock / IAM** | Bucket versioning; least-privilege keys; optional immutability |
| **Encryption** | Server-side encryption on bucket; consider client-side if required |
| **FS vs CSI** | CSI for same-cluster/cloud speed; **Kopia/FS for migration/DR portability** |
| **DB data** | Native backups (RDS snapshots, pgBackRest, etc.) + Velero for K8s objects |
| **etcd** | Still backup etcd / control plane separately for cluster rebuild |
| **GitOps** | Restore into paused apps or use Velero for PV/state; let Argo own desired YAML |
| **Hooks** | Quiesce writes briefly for consistency when needed |
| **Monitoring** | Alert on Backup `Failed`, schedule miss, BSL unavailable |
| **Version skew** | Keep Velero versions close across DR clusters |
| **RBAC** | Limit who can create restores (destructive) |
| **Exclude** | Tokens, ephemeral caches, `kube-system` unless you know why |

Example exclude:

```bash
velero backup create app-only \
  --include-namespaces payments-prod \
  --exclude-resources events,events.events.k8s.io \
  --default-volumes-to-fs-backup
```

---

## 8. Limitations to expect

- Not every CRD/controller restores cleanly (order, webhooks, operators must be installed first on target)  
- Install Operators / CRDs on destination **before** restoring CRs  
- Large FS backups are slow and storage-heavy  
- Cross-cloud networking and pull secrets often break first  
- Cluster-scoped resources (ClusterRoles, CRDs, PriorityClasses) need explicit include and care  

Order for restoring a complex app on cluster B:

```text
1. Platform: CSI, Ingress, cert-manager, external-secrets, operators
2. Velero restore namespace (or restore CRDs first if included)
3. Fix StorageClass / Ingress / secrets
4. Verify pods + data
5. Point DNS / enable GitOps
```

---

## 9. Alternatives (and when they’re better)

| Solution | Type | Strengths | When better than Velero |
|----------|------|-----------|-------------------------|
| **[Kasten K10](https://www.kasten.io/)** | Commercial K8s backup | UI, policy, app-centric, transforms | Enterprise support, complex DR workflows |
| **[CloudCasa](https://cloudcasa.io/) / [Trilio](https://trilio.io/) / [Portworx Backup](https://portworx.com/)** | Commercial | Managed UX, multi-cluster | Want vendor-supported SaaS/appliance |
| **CSI volume snapshots only** | Storage | Fast disk-level restore | Same cluster/cloud; no need for full object backup |
| **Cloud disk snapshots (EBS, PD, Azure Disk)** + IaC | Cloud native | Durable, scheduled by cloud | Cluster rebuild from Terraform + restore disks; less K8s-object aware |
| **etcd snapshot** + node images | Control plane | Recovers API state | Cluster-level DR, not app mobility ([external-etcd](./external-etcd-tarball-systemd.md)) |
| **GitOps (Argo CD) + external DB/object storage** | Design | Desired state always in Git; data outside cluster | Stateless apps; PV data in S3/RDS — “backup” = Git + data-plane snapshots |
| **Database-native tools** | Data | Consistent, PITR | Postgres/MySQL/Mongo — use these **with** Velero for manifests |
| **[VolSync](https://github.com/backube/volsync)** / replication | PVC sync | Continuous PVC replication | Active-active / warm DR of volumes |
| **Restic/Kopia alone** | Files | Simple FS backup | Non-Kubernetes file shares |
| **Manual `kubectl get -o yaml` + PVC copy** | DIY | No agents | Tiny labs only — not production DR |

### Practical recommendation

```text
Stateless + GitOps
  → Argo CD rebuild; Velero optional

Stateful apps on K8s PVs
  → Velero (FS/Kopia for DR mobility) OR Kasten
  → plus native DB backups

Control plane disaster
  → etcd snapshots + documented rebuild (kubeadm)
  → Velero does not replace etcd backup

Multi-cluster enterprise DR with GUI/compliance
  → Evaluate Kasten / CloudCasa / similar
```

---

## 10. Quick flows

### Same-cluster namespace recovery

```text
velero backup create … → disaster → velero restore --from-backup …
```

### Cluster A → Cluster B migration

```text
A: backup with --default-volumes-to-fs-backup to shared bucket
B: velero install → same BSL bucket (ReadOnly optional)
B: velero backup get (see A's backup)
B: install Operators/CSI/Ingress first
B: velero restore --from-backup … [--namespace-mappings]
B: fix DNS/secrets/StorageClass; smoke test; enable GitOps
```

### Zero → hero learning path

```text
1. CLI + velero install with MinIO/S3
2. Backup one NS without volumes; restore
3. FS backup with node-agent; restore PVC data
4. Schedule + TTL
5. Hooks for a demo DB
6. Second cluster + shared bucket restore
7. Document StorageClass transforms + quarterly DR drill
```

---

## 11. Cleanup lab

```bash
velero backup delete migrate-payments-2026-08-14 --confirm
velero uninstall
kubectl delete ns velero --ignore-not-found
# Empty bucket objects carefully if no longer needed
```

---

## References

- [Velero documentation](https://velero.io/docs/)
- [Backup / restore](https://velero.io/docs/main/backup-reference/)
- [File system backup (Kopia/Restic)](https://velero.io/docs/main/file-system-backup/)
- [CSI snapshot support](https://velero.io/docs/main/csi/)
- [Disaster recovery](https://velero.io/docs/main/disaster-case/)
- [BackupStorageLocation](https://velero.io/docs/main/api-types/backupstoragelocation/)
- [Kasten K10](https://docs.kasten.io/) · [VolSync](https://volsync.readthedocs.io/)
