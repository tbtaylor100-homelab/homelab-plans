---
jira-epic: MAHHAUS-43
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-03-26
context-refs: []
---

# Epic 1b: Observability Platform

## Why

The K3s cluster established in Epic 1 is running but invisible. There is no way to know whether pods are healthy, whether ArgoCD is in sync, whether a service has gone down, or what a failing pod's logs say. This epic adds a full observability layer: metrics, logs, dashboards, and alerting.

A secondary goal is a family-visible status page. Uptime Kuma runs on its own dedicated VM so it stays up even when K3s is down — it is the watchdog for the whole platform, not a tenant of it.

**Current state after Epic 1:**
- VM 110 (`k3s-control-01`, 192.168.1.60) running K3s with ArgoCD + MetalLB
- ArgoCD UI accessible at 192.168.1.200; no other services deployed
- No metrics, no logs, no alerting, no status page

## What We're Building

Five deployment units, delivered in phases:

1. **Uptime Kuma** on a new dedicated VM 111 (`uptime-kuma-01`, 192.168.1.61) — family-facing status page, independent of K3s
2. **kube-prometheus-stack** in K3s — Prometheus, Grafana, and Alertmanager as a single Helm chart, deployed via ArgoCD
3. **Loki + Promtail** in K3s — log aggregation for all K3s pods, wired into Grafana as a second datasource, deployed via ArgoCD
4. **Alertmanager → Discord** — critical event notifications (K3s node down, service unreachable)
5. **Node Exporter on Proxmox host** — host-level metrics for the hypervisor

Each K3s component lives in its own Forgejo repo with its own Helm chart. ArgoCD watches each repo via a pointer CRD in `homelab-infra/argocd/apps/`. This is the first real end-to-end test of the GitOps pipeline established in Epic 1.

## Tool Decisions

| Tool | Role | Why |
|------|------|-----|
| Prometheus | Metrics scraping and storage | Industry standard; native K8s service discovery |
| Grafana | Dashboards | Unified UI for both Prometheus metrics and Loki logs |
| Loki | Log aggregation | Lightweight; indexes labels not content; integrates natively with Grafana |
| Promtail | Log shipper | DaemonSet that tails K3s pod logs and forwards to Loki |
| Alertmanager | Alert routing | Bundled in kube-prometheus-stack; routes to Discord via webhook |
| kube-state-metrics | K8s object metrics | Exposes pod/deployment/ArgoCD sync state to Prometheus; bundled in kube-prometheus-stack |
| Node Exporter | Host metrics | Exposes CPU, memory, disk, network for K3s node and Proxmox host |
| Uptime Kuma | Status page + uptime checks | Simple self-hosted; HTTP/TCP checks with public status page; runs outside K3s |
| kube-prometheus-stack | Helm umbrella chart | Packages Prometheus + Grafana + Alertmanager + kube-state-metrics + Node Exporter for K8s in one chart |
| loki-stack | Helm umbrella chart | Packages Loki + Promtail in one chart |

**Why Uptime Kuma is not in K3s:** If K3s is down, anything running inside it is also down, including the status page. Uptime Kuma must be outside the thing it monitors. VM 111 is small and cheap; it exists solely for this purpose.

**Why kube-prometheus-stack over individual charts:** It wires Prometheus, Grafana, Alertmanager, and kube-state-metrics together with sane defaults and pre-built dashboards. Installing them separately means hand-wiring all the datasource and scrape configuration.

**Why Loki over Elasticsearch/OpenSearch:** Loki indexes only labels (pod name, namespace, container), not the full log content. It is dramatically cheaper on RAM and disk for a single-node homelab. Full-text search is still available via LogQL.

## Infrastructure Decisions

| Decision | Choice |
|----------|--------|
| Uptime Kuma VM ID / hostname | 111 / `uptime-kuma-01` |
| Uptime Kuma VM IP | 192.168.1.61 (static, reserved in router) |
| Uptime Kuma VM spec | 1 vCPU / 2 GB RAM / 20 GB disk (Ubuntu 24.04 cloud-init) |
| Uptime Kuma VM storage | `Intel660P` LVM (same pool as k3s-control-01) |
| Uptime Kuma deployment | Docker Compose via Ansible role (not K3s) |
| Grafana LoadBalancer IP | 192.168.1.201 (MetalLB, reserved) |
| Prometheus storage | PersistentVolumeClaim inside K3s, local-path provisioner |
| Loki storage | PersistentVolumeClaim inside K3s, local-path provisioner |
| Grafana admin credentials | Kubernetes Secret (managed in app repo, not committed in plaintext) |
| Discord webhook | Kubernetes Secret (managed in app repo) |
| Node Exporter on Proxmox host | Runs as a systemd service directly on the Proxmox host (192.168.1.12) |
| Prometheus scrape for bare-metal exporters | Static scrape config in kube-prometheus-stack values |

## Repo Structure

Each K3s component lives in its own Forgejo repo. `homelab-infra` gains only the ArgoCD pointer CRDs and the new VM OpenTofu stack.

```
homelab-infra/
├── opentofu/
│   └── stacks/uptime-kuma/     # VM 111 stack (reuses proxmox-vm module)
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── ansible/
│   ├── inventory/hosts.yml     # add uptime-kuma-01
│   └── roles/uptime-kuma/      # installs Docker + Compose, deploys Uptime Kuma
│       ├── tasks/main.yml
│       ├── templates/docker-compose.yml.j2
│       └── defaults/main.yml
└── argocd/
    └── apps/
        ├── kube-prometheus-stack.yaml   # points at homelab-kube-prometheus-stack repo
        └── loki-stack.yaml              # points at homelab-loki-stack repo

homelab-kube-prometheus-stack/           # separate Forgejo repo
├── Chart.yaml                           # wraps kube-prometheus-stack as a dependency
├── values.yaml                          # Grafana datasources, Alertmanager Discord config, scrape targets
└── templates/
    ├── grafana-secret.yaml              # admin credentials (SOPS-encrypted or external secret)
    └── alertmanager-discord-secret.yaml # Discord webhook (SOPS-encrypted or external secret)

homelab-loki-stack/                      # separate Forgejo repo
├── Chart.yaml                           # wraps loki-stack as a dependency
├── values.yaml                          # retention, storage size, Promtail config
└── templates/
    └── (empty initially)
```

**Secret management note:** Secrets are not committed in plaintext. For this epic, the simplest path is to create the Kubernetes Secrets manually once (or via a sealed-secret / SOPS approach). A dedicated secret management epic is out of scope here; placeholder comments in `values.yaml` will document what secrets are expected.

## Implementation Order

```
Phase 1 (MAHHAUS-43a): VM 111 + Uptime Kuma
  Provision uptime-kuma-01 via OpenTofu proxmox-vm module (new stack)
  Run tofu apply locally (same bootstrap pattern as Epic 1)
  Write Ansible role: installs Docker, deploys Uptime Kuma via Docker Compose
  Configure Uptime Kuma checks: Forgejo (192.168.1.50), ArgoCD (192.168.1.200),
    Grafana (192.168.1.201 — placeholder until Phase 2)
  Status page publicly accessible at http://192.168.1.61:3001/status/homelab

Phase 2 (MAHHAUS-43b): kube-prometheus-stack via ArgoCD
  Create homelab-kube-prometheus-stack repo on Forgejo
  Write Chart.yaml (depends on kube-prometheus-stack upstream chart)
  Write values.yaml: enable Grafana, set LoadBalancer IP 192.168.1.201,
    add static scrape targets for Node Exporter on 192.168.1.12 and 192.168.1.50
  Add argocd/apps/kube-prometheus-stack.yaml to homelab-infra (PR → merge → ArgoCD deploys)
  Verify Grafana accessible at http://192.168.1.201, K3s node dashboard loads

Phase 3 (MAHHAUS-43c): Loki + Promtail via ArgoCD + promtail base role
  Create homelab-loki-stack repo on Forgejo
  Write Chart.yaml + values.yaml (Promtail DaemonSet tails /var/log/pods on k3s-control-01)
  Add argocd/apps/loki-stack.yaml to homelab-infra (PR → merge → ArgoCD deploys)
  Add Loki datasource to Grafana values.yaml in homelab-kube-prometheus-stack
  Add roles/promtail Ansible role to homelab-infra:
    - Installs Promtail as a systemd service on any VM
    - Ships systemd journal + syslog to Loki
    - Included in every VM playbook alongside base role (systematic, not ad-hoc)
  Apply retroactively to k3s-control-01 and uptime-kuma-01
  Verify K3s pod logs and VM systemd logs both searchable in Grafana Explore

Phase 4 (MAHHAUS-43d): Alertmanager → Discord + Uptime Kuma alerts
  Create Discord webhook; store as Kubernetes Secret
  Configure Alertmanager in kube-prometheus-stack values.yaml: route critical alerts to Discord
  Enable default K3s/node alerting rules (bundled in kube-prometheus-stack)
  Configure Uptime Kuma: add Discord notification channel, assign to all checks
  Verify: take down a test service, confirm Discord receives alert within 1 minute

Phase 5 (MAHHAUS-43e): Node Exporter on Proxmox host
  Install Node Exporter as systemd service on Proxmox host (192.168.1.12)
  Update kube-prometheus-stack values.yaml scrape target to include it
  Verify Proxmox host metrics visible in Grafana
  (VM 102 excluded — being decommissioned, not worth instrumenting)
```

## Bootstrapping Notes

**VM 111 first apply is local.** Same pattern as Epic 1: `tofu apply` from the operator Mac before CI is involved. Subsequent changes flow through PRs.

**ArgoCD deploys Phase 2 and 3.** Once `argocd/apps/kube-prometheus-stack.yaml` is merged to `homelab-infra`, ArgoCD detects the new Application CRD and reconciles it — pulling the chart from `homelab-kube-prometheus-stack` and deploying it. No `helm install` commands needed from the operator.

**Uptime Kuma check for Grafana in Phase 1.** The Grafana IP (192.168.1.201) does not exist yet when Phase 1 runs. Add the check in Uptime Kuma immediately, but expect it to show as down until Phase 2 completes. This is intentional — it validates the alerting path.

**MetalLB IP reservation.** 192.168.1.201 must be reserved in the router (no DHCP) before deploying the kube-prometheus-stack chart. MetalLB assigns it to the Grafana LoadBalancer Service.

**kube-prometheus-stack includes a Node Exporter DaemonSet** for K8s nodes. The bare-metal Node Exporters in Phase 5 are additive — they are scraped as static targets, not as K8s service discovery targets.

**Loki retention.** Default to 7 days for this epic. Adjust based on disk usage after a week of operation.

## Explicitly Out of Scope

- Migrating VM 102 services to K3s (separate epic)
- Ingress / TLS for Grafana or Uptime Kuma (plain HTTP on local IPs for now)
- SOPS or Sealed Secrets for automated secret management (manual secret creation for this epic)
- Multi-node K3s (single control plane node only)
- Alerting thresholds tuning (enable default rules now; tune thresholds in a follow-up)
- VLAN segmentation (MAHHAUS-49)
- OpenTofu state migration to MinIO (MAHHAUS-50)
- Tracing (Tempo/Jaeger) — metrics and logs are sufficient for this stage

## Done When

1. VM 111 (`uptime-kuma-01`) exists in Proxmox, provisioned from `homelab-infra`
2. Uptime Kuma status page is accessible at `http://192.168.1.61:3001/status/homelab` and shows checks for Forgejo, ArgoCD, and Grafana
3. Grafana is accessible at `http://192.168.1.201` and shows a K3s node health dashboard with live CPU, memory, and pod status data
4. Grafana Explore shows logs from all K3s pods via the Loki datasource, searchable by namespace and pod name
5. Alertmanager delivers a Discord message when a test service is taken offline
6. Uptime Kuma delivers a Discord message when a monitored service goes offline
7. Proxmox host (192.168.1.12) CPU and memory metrics are visible in Grafana
