---
jira-story: MAH-65
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-04-10
context-refs:
  - epic-1b-observability-platform.md
---

# Story MAH-65: Prometheus + Grafana on K3s via ArgoCD

## Why

The K3s cluster is running ArgoCD, OpenBao, and three MCP servers but has no
visibility into its own health. There is no way to know whether pods are
healthy, whether node CPU or memory is under pressure, or whether a service
has silently gone down. This story adds the metrics foundation:
Prometheus for scraping and storage, Grafana for dashboards, plus static
scrape targets for the two bare-metal hosts in the homelab.

## What We're Building

kube-prometheus-stack deployed to K3s via a single ArgoCD Application CRD
in the infra repo. ArgoCD sources the chart directly from the upstream
Prometheus community Helm repo; no separate Forgejo repo is needed for this
story. All values are inline in the Application CRD — same pattern as
`openbao.yaml`.

**Corrections from the epic plan:**

| Epic plan said | Actual |
|---|---|
| Grafana LoadBalancer at `192.168.1.201` | `.201` is owned by `proxmox-mcp`. Using `192.168.1.204` (next free IP) |
| Separate `homelab-kube-prometheus-stack` Forgejo repo | Inline values in infra ArgoCD CRD — simpler, no umbrella chart overhead |
| ArgoCD auto-detects new Application CRDs on merge | No App-of-Apps exists; one-time manual `kubectl apply` required |

## Architecture

```
Operator Mac                     K3s .60 — monitoring namespace
────────────                     ─────────────────────────────────────────────
                                 ┌─ kube-prometheus-stack ───────────────────┐
                                 │  Prometheus      (scrape + TSDB)          │
                                 │  Grafana         LoadBalancer → .204       │
                                 │  Alertmanager    ClusterIP (config later)  │
                                 │  kube-state-metrics                        │
                                 │  node-exporter   DaemonSet (K3s node)     │
                                 └───────────────────────────────────────────┘
                                          │ additionalScrapeConfigs
                                          ├─ 192.168.1.12:9100  (Proxmox host)
                                          └─ 192.168.1.50:9100  (VM 102 "homelab")
ArgoCD (192.168.1.200)
  └─ Application: kube-prometheus-stack
       └─ source: https://prometheus-community.github.io/helm-charts
```

## Key Decisions

| Decision | Choice | Reason |
|---|---|---|
| Deployment method | Helm chart via ArgoCD (inline values) | Matches openbao pattern; no extra repo to manage |
| Chart version | Pinned to specific patch (e.g. `83.4.0`) | Reproducible deployments; bump via PR when upgrading |
| Grafana IP | `192.168.1.204` (MetalLB pinned) | `.201`–`.203` already allocated to MCP servers |
| `ServerSideApply=true` | Required | kube-prometheus-stack CRDs exceed 262KB client-side annotation limit |
| Bare-metal scrape | `additionalScrapeConfigs` (static targets) | Proxmox host and VM 102 are not K8s nodes; static is correct |
| Separate Forgejo repo | Not created | Umbrella chart adds helm dep update + tarball commit overhead for no benefit at this scale |
| Grafana admin password | Chart default (`prom-operator`) | Homelab LAN only; change manually post-deploy if desired |
| Prometheus persistence | Ephemeral (chart default) | Acceptable for initial deploy; add PVC in follow-on if TSDB retention matters |

## IaC Changes

### `infra/argocd/apps/kube-prometheus-stack.yaml` (new)

ArgoCD Application pointing to the upstream `kube-prometheus-stack` Helm chart
from `https://prometheus-community.github.io/helm-charts`. Inline Helm values
configure:

- Grafana `LoadBalancer` service with MetalLB IP pin `192.168.1.204`
- `additionalScrapeConfigs` with two static jobs:
  - `node-exporter-proxmox` → `192.168.1.12:9100`
  - `node-exporter-homelab-vm` → `192.168.1.50:9100`
- `ServerSideApply=true` sync option (required for CRD size)
- `CreateNamespace=true` (creates `monitoring` namespace on first sync)

## Deployment Workflow

This story has a one-time manual step because `argocd/apps/` is not watched
by any ArgoCD Application (no App-of-Apps):

```
1. Branch: feat/mah-65-kube-prometheus-stack in infra repo
2. Create argocd/apps/kube-prometheus-stack.yaml
3. PR → test by applying the CRD from the branch (no merge needed to test)
4. One-time apply: ssh ubuntu@192.168.1.60 "sudo kubectl apply -f -" < argocd/apps/kube-prometheus-stack.yaml
5. Verify ArgoCD syncs and Grafana is reachable at http://192.168.1.204
6. Merge PR
```

Future edits to the inline values (e.g. bumping chart version) also require a
re-apply after merge. This is the current limitation of not having an
App-of-Apps — a follow-on ticket should address this.

## Verification

| Check | Command / URL |
|---|---|
| ArgoCD sync | `kubectl -n argocd get application kube-prometheus-stack` → `Synced / Healthy` |
| Grafana reachable | `http://192.168.1.204` → login `admin` / `prom-operator` |
| MetalLB IP assigned | `kubectl -n monitoring get svc kube-prometheus-stack-grafana` → EXTERNAL-IP = `.204` |
| Prometheus targets | `http://localhost:9090/targets` (port-forward) → both static jobs show `UP` |

## Follow-on Tickets

| Area | Description |
|---|---|
| App-of-Apps | Create a root ArgoCD Application watching `argocd/apps/` so new CRDs apply automatically on merge |
| Prometheus persistence | Add a PVC (`local-path`, ~10 Gi) if TSDB data needs to survive pod restarts |
| Loki + Promtail | MAH-66 (or equivalent) — add log aggregation, wire into Grafana as second datasource |
| Alertmanager → Discord | Configure alert routing for critical events |
| Node Exporter on Proxmox | Install node_exporter as a systemd service on `192.168.1.12` (currently assumed running; Phase 5 of epic) |
| Grafana admin credentials | Store admin password as a K8s Secret rather than using the chart default |
