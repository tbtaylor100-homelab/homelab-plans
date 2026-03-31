---
jira-story: MAHHAUS-62
jira-epic: MAHHAUS-42
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-03-30
context-refs:
  - epic-1b-observability-platform.md
---

# Story MAHHAUS-62: Persistent Volume Backup — Velero + MinIO

## Why

Stateful workloads on K3s write to NVMe-backed PersistentVolumes on `k3s-control-01`. If the VM is destroyed or corrupted, that data is lost. A backup strategy must be in place before any stateful workload (database, user data store) is onboarded to K3s.

This story establishes that baseline: nightly PV backups to MinIO on the NAS, with a verified restore procedure.

**Rule: No stateful workload (database, anything with user data) runs on K3s until this story is complete.**

**Current state:**
- K3s running on VM 110 (`k3s-control-01`, 192.168.1.60) with local-path provisioner for PVs
- MinIO already deployed on Alexandria (NAS, 192.168.1.132, OMV7/Debian 12) at port 9000
- No backup tooling in place

> Note: MAHHAUS-62 refers to "OMV5" — this is stale. The NAS runs OMV7/Debian 12.

## What We're Building

Velero running in K3s, configured for File System Backup (kopia) targeting a dedicated MinIO bucket on the NAS. Nightly schedule with 30-day retention. One verified test restore before the story is closed.

## Tool Decisions

| Tool | Role | Why |
|------|------|-----|
| Velero | Kubernetes-native backup operator | Runs as a K8s controller; integrates with PVCs natively; schedule CRD; restore CRD |
| Velero AWS plugin | S3-compatible object store driver | Speaks S3 API; works with any S3-compatible target including MinIO |
| File System Backup (kopia) | PV data backup mechanism | Required because `local-path` provisioner does not implement CSI VolumeSnapshot API; kopia does file-level backup of mounted PV directories |
| MinIO (existing) | Backup storage target | Already deployed on Alexandria; S3-compatible; suitable as archival store |
| Helm + ArgoCD | Deployment | Consistent with Epic 1 GitOps pattern |

**Why File System Backup over CSI snapshots:** K3s's default `local-path` provisioner does not support the CSI VolumeSnapshot API. Velero's CSI snapshot mode requires the storage driver to implement `VolumeSnapshotContent` creation. File System Backup bypasses this by mounting the PV and copying data at the file level — slower than a snapshot, but correct and complete.

**Why kopia over restic:** Velero deprecated restic as the FSB engine in favour of kopia in v1.12. Kopia provides cross-backup deduplication — nightly full backups do not consume N× the disk space because unchanged blocks are referenced, not re-uploaded.

**Why the same MinIO instance as MAHHAUS-50:** Operational simplicity — one MinIO to monitor, back up, and credential-manage. Velero uses a dedicated bucket (`velero`) isolated from other uses (e.g., OpenTofu state in `terraform-state`).

## Infrastructure Decisions

| Decision | Choice |
|----------|--------|
| MinIO endpoint | `http://192.168.1.132:9000` (existing instance) |
| Velero bucket | `velero` (create new bucket in existing MinIO) |
| MinIO credentials | New dedicated access key/secret for Velero (principle of least privilege) |
| Backup schedule | Nightly at 02:00 local time (low activity window) |
| Backup retention (TTL) | 30 days |
| Default backup scope | All namespaces, all PVCs with FSB annotation |
| Velero namespace | `velero` |
| Velero LoadBalancer | Not needed — no UI; purely operator-facing via CLI/CRDs |
| Credentials storage | Kubernetes Secret in `velero` namespace (created manually, not committed) |

## Repo Structure

Velero configuration lives directly in `homelab-infra`. There is no separate repo. Velero is cluster-wide infrastructure (like ArgoCD or MetalLB), not a tenant application — a dedicated repo would add overhead with no benefit.

ArgoCD Applications can reference a path within the same repo via `repoURL` + `path`. Cluster-level tooling lives under `homelab-infra/k3s/` as a natural home for anything that manages the cluster itself rather than running on it.

```
homelab-infra/
├── k3s/
│   └── velero/
│       ├── Chart.yaml               # wraps vmware-tanzu/velero Helm chart as dependency
│       ├── values.yaml              # provider config, FSB settings, schedule, credentials ref
│       └── templates/
│           └── backup-schedule.yaml # Velero Schedule CRD (nightly, all-namespaces)
└── argocd/
    └── apps/
        └── velero.yaml              # ArgoCD Application CRD (repoURL=homelab-infra, path=k3s/velero)
```

**Why a separate Schedule CRD template:** The upstream Velero Helm chart supports schedules via `values.yaml`, but expressing it as an explicit CRD template makes the schedule visible in ArgoCD and version-controllable with full clarity.

## Implementation Order

```
Phase 1 (prep): MinIO bucket + credentials
  Log in to MinIO Console at http://192.168.1.132:9001
  Create bucket: velero
    - Versioning: disabled (Velero manages its own object lifecycle)
    - Object locking: disabled
  Create access key: velero-k3s
    - Attach policy: ReadWrite on bucket velero only
  Store credentials: create K8s Secret manually on k3s-control-01
    kubectl create secret generic velero-minio-credentials \
      --namespace velero \
      --from-literal=cloud="\[default]\naws_access_key_id=<key>\naws_secret_access_key=<secret>"

Phase 2: Velero Helm chart + ArgoCD deployment
  Add k3s/velero/ directory to homelab-infra
  Write Chart.yaml:
    - dependency: vmware-tanzu/velero (pin to current stable, e.g. 6.x)
  Write values.yaml:
    - configuration.provider: aws
    - configuration.backupStorageLocation:
        name: default
        provider: aws
        bucket: velero
        config:
          region: minio
          s3ForcePathStyle: "true"
          s3Url: http://192.168.1.132:9000
          publicUrl: http://192.168.1.132:9000
    - credentials.existingSecret: velero-minio-credentials
    - deployNodeAgent: true           # required for File System Backup
    - configuration.features: EnableCSI  # leave off; CSI not available via local-path
    - defaultVolumeToFsBackup: true   # opt all PVCs into FSB by default
  Write templates/backup-schedule.yaml:
    apiVersion: velero.io/v1
    kind: Schedule
    metadata:
      name: nightly-all-namespaces
      namespace: velero
    spec:
      schedule: "0 2 * * *"
      template:
        ttl: 720h   # 30 days
        includedNamespaces: ["*"]
        defaultVolumesToFsBackup: true
  Add argocd/apps/velero.yaml to homelab-infra — set repoURL to homelab-infra, path to k3s/velero
  Commit all to homelab-infra in a single PR (chart + ArgoCD app pointer together) → merge → ArgoCD deploys
  Verify: kubectl get pods -n velero shows controller + node-agent DaemonSet running
  Verify: kubectl get backupstoragelocations -n velero shows phase=Available

Phase 3: Trigger manual backup + test restore
  Trigger one-off backup:
    velero backup create smoke-test --include-namespaces <a namespace with a PVC>
  Verify backup object appears in MinIO velero bucket
  Verify: velero backup describe smoke-test shows Completed
  Simulate data loss: delete the PVC content or create a test namespace with known data
  Restore:
    velero restore create --from-backup smoke-test
  Verify data is present after restore
  Document restore command in this file (below)

Phase 4 (optional, this story): Grafana alert for backup failures
  Velero exposes Prometheus metrics (velero_backup_failure_total, velero_backup_last_status)
  Add scrape target to kube-prometheus-stack values.yaml (serviceMonitor or static target)
  Add Alertmanager rule: alert if velero_backup_last_status != 1 for any schedule
  This provides observability parity with Epic 1b — backup failures notify Discord
```

## Restore Procedure

The step-by-step restore playbook belongs in `homelab-knowledge`, not here. Once Phase 3 (test restore) is complete and verified, write a runbook page there covering:

- How to list available backups
- How to restore a full namespace
- How to restore a single PVC
- How to monitor restore progress
- Worst-case data loss window (up to ~24 hours from last nightly backup)

This plan doc records the *design decision* to use Velero FSB + MinIO. Operational runbooks (what to do when something breaks) live in the knowledge base.

## Bootstrapping Notes

**MinIO bucket creation is manual (Phase 1).** The MinIO instance is not yet managed by OpenTofu (MAHHAUS-50 covers that). Bucket and credentials are created via the MinIO Console UI or `mc` CLI. This is consistent with the pattern of one-time manual setup for credentials that are referenced by K8s Secrets.

**Node agent DaemonSet is required.** Velero's File System Backup uses a `node-agent` pod running on each node to access the PV mount paths on the host. With a single-node K3s cluster (`k3s-control-01`) this is one pod. It must be running for FSB to work.

**ArgoCD will not manage the K8s Secret.** The `velero-minio-credentials` secret is created manually and is not tracked in Git. ArgoCD may show the `velero` namespace as "out of sync" — configure the Application in `homelab-infra/argocd/apps/velero.yaml` with `ignoreDifferences` for secrets, or use `syncOptions: [RespectIgnoreDifferences=true]`.

**Phase 4 (Grafana alert) depends on Epic 1b Phase 2.** Velero metrics are only scrapeable once kube-prometheus-stack is deployed. Phase 4 is sequenced after Epic 1b is complete, but it is included here because it closes the observability loop for backups.

## Explicitly Out of Scope

- Cross-site backup replication (single-site NAS is sufficient for homelab)
- Backup encryption at rest (MinIO on local LAN; encryption is a follow-up)
- Automated MinIO provisioning via OpenTofu (covered by MAHHAUS-50)
- Velero backup for the Proxmox host itself (VM snapshots via Proxmox are the host-level strategy)
- Restoring individual files from a backup (full namespace restore is the recovery unit for now)

## Done When

1. Velero controller and node-agent DaemonSet running in K3s `velero` namespace
2. BackupStorageLocation shows `phase: Available` (connectivity to MinIO confirmed)
3. Nightly schedule is active and at least one scheduled backup has completed successfully
4. One test restore has been completed and verified — data recovered matches data written before backup
5. (Stretch) Alertmanager fires a Discord notification when a test backup is deliberately broken
