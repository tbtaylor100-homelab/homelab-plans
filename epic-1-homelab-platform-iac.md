---
jira-epic: MAHHAUS-42
jira-stories: MAHHAUS-51, MAHHAUS-52, MAHHAUS-53, MAHHAUS-54, MAHHAUS-55, MAHHAUS-56, MAHHAUS-57, MAHHAUS-58
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-03-23
context-refs: []
---

# Epic 1: Homelab Platform — IaC Foundation

## Why

The homelab currently has no version-controlled infrastructure. Services are manually configured, VM provisioning is ad-hoc, and there is no safe way for agents to interact with the platform. This epic establishes the IaC foundation that every subsequent epic depends on.

**Current state:**
- Single Proxmox node (`pve`) with manually configured VMs
- VM 102 (`homelab`): Docker, running Forgejo + Proxmox/JIRA MCP servers
- VM 101 (`OMV5`): NAS — do not touch
- No GitOps model, no reproducible provisioning, no CI

## What We're Building

A new `homelab-infra` repo on Forgejo that version-controls all infrastructure. Changes flow through PRs. Merging a PR applies the change automatically. No more SSH and manual configuration.

**First milestone (this epic):** One K3s VM provisioned and managed entirely through the repo. ArgoCD running inside it. CI validating every change.

## Tool Decisions

| Tool | Role | Why |
|------|------|-----|
| OpenTofu | Provision Proxmox VMs | Declarative IaC; replaces manual Proxmox UI clicks |
| `bpg/proxmox` provider | OpenTofu ↔ Proxmox API | Best-maintained community Proxmox provider |
| Ansible | Configure VMs over SSH | Agentless; installs K3s and hardens OS |
| K3s | Container orchestration | Lightweight Kubernetes; exposes programmable API for future agent management |
| Helm | Package manager for K8s | Each app carries its own Helm chart in its own repo — not centralised here |
| ArgoCD | GitOps operator in K3s | Watches each app's repo; continuously reconciles cluster state |
| Forgejo Actions | CI pipelines | Built into Forgejo; `tofu plan` on PRs, `tofu apply` on merge |
| MetalLB | K8s LoadBalancer for bare metal | Assigns local IPs to K8s services (no cloud provider needed) |

**Helm lives in app repos, not here.** Each service (Forgejo, Prometheus, etc.) has its own repo containing its own Helm chart. A broken chart change only affects that app — no shared blast radius. `homelab-infra` is not aware of app-level Helm content.

**What ArgoCD does here:** `homelab-infra` contains thin ArgoCD `Application` CRDs — small YAML pointer files that say "watch *this* app repo at *this* path." ArgoCD reads these and then watches each app repo independently. The charts themselves never touch this repo.

**ArgoCD vs Forgejo Actions:** Actions runs one-time pipelines triggered by events. ArgoCD runs permanently inside K3s, watches git, and fixes any drift automatically. Both are needed: Actions for `tofu apply`, ArgoCD for K8s service reconciliation.

## Infrastructure Decisions

| Decision | Choice |
|----------|--------|
| K3s VM storage | `Intel660P` LVM (~600 GB free) |
| K3s VM OS | Ubuntu 24.04 LTS (cloud-init) |
| K3s VM spec | 4 vCPU / 8 GB RAM / 60 GB disk |
| VM ID / hostname | 110 / `k3s-control-01` |
| Network | Flat `192.168.1.x` (VLAN design deferred → MAHHAUS-49) |
| MetalLB pool | `192.168.1.200–220` (reserved in router, no DHCP) |
| OpenTofu state | Local file on operator Mac, gitignored (migrate to MinIO → MAHHAUS-50) |
| Proxmox auth | API token `root@pam!opentofu` (not root password) |

## Repo Structure (`homelab-infra`)

```
homelab-infra/
├── .forgejo/workflows/
│   ├── tofu-plan.yml       # PR: fmt + validate + plan (output as PR comment)
│   ├── tofu-apply.yml      # merge to main: apply + verify k3s Ready
│   └── ansible-lint.yml    # PR: lint check
├── opentofu/
│   ├── modules/proxmox-vm/ # reusable VM blueprint (main.tf, variables.tf, outputs.tf)
│   └── stacks/k3s/         # K3s VM (main.tf, variables.tf, terraform.tfvars)
├── ansible/
│   ├── inventory/hosts.yml
│   ├── group_vars/all.yml
│   ├── roles/base/         # apt, SSH hardening, qemu-guest-agent
│   ├── roles/k3s/          # k3s install, disable Traefik, kubeconfig fetch
│   ├── roles/argocd/       # helm install argocd (one-time bootstrap)
│   └── playbooks/k3s.yml
└── argocd/
    └── apps/               # ArgoCD Application pointer CRDs (one file per registered app)
        └── example-app.yaml  # points ArgoCD at an app repo; charts live there, not here
```

## Implementation Order

```
Phase 0 (manual, one-time)     → MAHHAUS-51
  Create Proxmox API token
  Download Ubuntu 24.04 cloud image to pve
  Reserve 192.168.1.50 (static IP) + MetalLB range in router

Phase 1: Scaffold homelab-infra → MAHHAUS-52
  Create repo on Forgejo, CODEOWNERS, branch protection

Phase 2: OpenTofu module        → MAHHAUS-53
  proxmox-vm module (reusable for any future VM)

Phase 3: OpenTofu K3s stack     → MAHHAUS-54
  k3s stack using the module; run tofu apply locally (first and only manual run)

Phase 4: Ansible                → MAHHAUS-55
  base + k3s roles; run playbook to configure VM and install K3s

Phase 5: ArgoCD bootstrap       → MAHHAUS-56
  Ansible role installs ArgoCD via Helm (one-time); ArgoCD then watches argocd/apps/ for app registrations

Phase 6: CI + runner            → MAHHAUS-57
  Deploy act_runner on VM 102; wire up Forgejo Actions workflows

ADRs (alongside implementation) → MAHHAUS-58
  adr-001-container-orchestration.md
  adr-002-iac-tooling.md
  adr-003-gitops-model.md
```

## Bootstrapping Note

CI can't automate OpenTofu until the runner exists, and the runner lives on VM 102 which we're not migrating yet. The first `tofu apply` and `ansible-playbook` runs are local. After Phase 6, all future changes flow through PRs.

## Explicitly Out of Scope

- Migrating VM 102 (Forgejo/MCP servers) to K3s — can't cut over before K3s exists
- Migrating OMV5 (NAS) or Pi-Hole
- Observability stack (Epic 1b, MAHHAUS-43)
- VLAN segmentation (MAHHAUS-49)
- OpenTofu state migration to MinIO (MAHHAUS-50)

## Done When

1. VM 110 (`k3s-control-01`) exists in Proxmox, provisioned from `homelab-infra`
2. `kubectl get nodes` returns `k3s-control-01` in `Ready` state
3. ArgoCD UI is accessible at `192.168.1.200`
4. A PR to `homelab-infra` triggers `tofu plan` automatically
5. Merging that PR triggers `tofu apply` and verifies K3s health
