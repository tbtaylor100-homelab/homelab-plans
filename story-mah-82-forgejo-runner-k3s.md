---
jira-story: MAH-82
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-04-13
context-refs:
  - living-knowledge-system.md
---

# Story MAH-82: Forgejo Actions Runner on K3s

## Why

Forgejo CI workflows (e.g. `repowise-image.yml`) build and push Docker images to the
Forgejo container registry at `192.168.1.50:3000`. Without a runner registered, no CI
jobs execute. A runner was previously co-located with Forgejo on the same host (192.168.1.50),
but that host is inaccessible — no SSH, no Proxmox console — making it impossible to
configure or maintain.

Moving the runner to K3s (192.168.1.60, a fully controlled node) gives us a managed,
GitOps-driven runner where we can configure Docker-in-Docker (DinD) with
`insecure-registries: ["192.168.1.50:3000"]`. This avoids the need to enable TLS on
the inaccessible Forgejo host while still making CI fully operational.

## What We're Building

A Forgejo Actions runner deployed as a K8s Deployment in the `forgejo-runner` namespace.
The pod has two containers:

- **runner**: `code.forgejo.org/forgejo/runner` — polls Forgejo for queued jobs and
  executes workflow steps
- **dind**: `docker:27-dind` (privileged) — provides a Docker daemon for `docker build`,
  `docker login`, and `docker push` steps; configured with
  `--insecure-registry=192.168.1.50:3000`

An initContainer handles one-time runner registration with Forgejo using a token stored
in a K8s secret (created by Ansible). ArgoCD manages the deployment via GitOps.

## Architecture

```
Forgejo (192.168.1.50:3000)
  ↑ polls for jobs
forgejo-runner pod (K3s .60 — forgejo-runner namespace)
  ├── runner container (forgejo-runner daemon)
  └── dind sidecar (Docker daemon, insecure-registry configured)
       ↓ docker push
Forgejo registry (192.168.1.50:3000)
```

## Implementation

### K8s Manifests — `infra/kubernetes/forgejo-runner/deployment.yaml`

- Namespace: `forgejo-runner`
- ConfigMap: runner `config.yml` (capacity 1, 3h timeout)
- Deployment: runner + DinD sidecar with shared emptyDir for `.runner` registration file
- initContainer: registers runner with `--name k3s-runner --labels ubuntu-latest --no-interactive`
- No Service needed (runner dials out to Forgejo)

### ArgoCD App — `infra/argocd/apps/forgejo-runner.yaml`

Follows the same pattern as `mcp-servers.yaml`:
- repoURL: `http://forgejo.local:3000/root/infra.git`
- path: `kubernetes/forgejo-runner`
- Automated sync with prune + selfHeal + CreateNamespace

Applied manually after merge (same as all ArgoCD apps).

### Ansible Secret Creation

`forgejo_runner_token` added to `group_vars/all/secrets.yml` (vault-encrypted).
New task in `ansible/roles/mcp-servers/tasks/main.yml` creates `forgejo-runner-secret`
in the `forgejo-runner` namespace using the idempotent `--dry-run=client | apply` pattern.

### Registration Token

Obtained from Forgejo Admin UI: **Site Administration → Actions → Runners → Create new Runner**.

## Acceptance Criteria

- `k3s-runner` shows as online in Forgejo admin → Actions → Runners
- `repowise-image.yml` CI workflow completes successfully (docker login + build + push)
- Runner pod is stable (no CrashLoopBackOff), managed by ArgoCD
