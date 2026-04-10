---
jira-story: MAH-76
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-04-10
context-refs:
  - living-knowledge-system.md
---

# Story MAH-76: RepoWise as a K8s Workload

## Why

The homelab has three MCP servers running in K3s (Forgejo, Proxmox, Atlassian) but no
codebase intelligence layer. RepoWise indexes Git repos and exposes a RAG-based Q&A
interface, dependency graphs, hotspot analysis, and architectural decision search — all
consumable via MCP tools by Claude Code and other agents. Deploying it as a long-lived
K8s workload means the index is always up to date and available without requiring a local
`repowise serve` on the operator's machine.

## What We're Building

A custom Docker image (`python:3.11-slim` + `repowise==0.2.1` + `mcp-proxy`) deployed to
the `mcp-servers` namespace. At startup the container clones configured repos from Forgejo,
runs a one-time `repowise init` (~25 min), then serves:

- HTTP MCP endpoint on `:8080` via `mcp-proxy` (matches existing proxmox-mcp pattern)
- Web dashboard on `:7337` via `repowise serve` (subprocess under mcp-proxy)

A background loop runs `repowise update` on a configurable interval for incremental refresh.
A PVC at `/data` preserves the index across pod restarts so init only runs once.

## Architecture

```
Operator Mac / Claude Code
──────────────────────────
  claude mcp add repowise --transport http http://192.168.1.2xx:8080/mcp

                          K3s .60 — mcp-servers namespace
                          ──────────────────────────────────────────
                          ┌─ repowise-mcp (Deployment) ───────────┐
                          │  mcp-proxy (PID 1, :8080)             │
                          │    └─ repowise serve (:7337)          │
                          │  background: init + update loop       │
                          │  PVC: repowise-data (10Gi, /data)     │
                          └───────────────────────────────────────┘
                                     │ clones from
                          Forgejo 192.168.1.50:3000
                            └─ root/infra
                            └─ root/homelab-plans
                            └─ ... (REPOWISE_REPOS env)

  Forgejo CI (192.168.1.50)
    └─ .forgejo/workflows/repowise-image.yml
         └─ builds + pushes → 192.168.1.50:3000/root/repowise:0.2.1
```

## Key Decisions

| Decision | Choice | Reason |
|---|---|---|
| MCP transport | `mcp-proxy` wrapping `repowise serve` stdio | repowise has no native HTTP MCP; matches proxmox-mcp pattern |
| Base image | `python:3.11-slim` (Debian) | C-extension deps (numpy, sentence-transformers) require glibc |
| mcp-proxy install | `pip install mcp-proxy` | Homogeneous pip env; no Node.js runtime needed |
| Init timing | Background subshell; `mcp-proxy` starts as PID 1 immediately | Pod stays alive during 25-min init; avoids crash-loop |
| Index persistence | PVC `local-path` 10Gi at `/data` | Init only runs once; survives pod restarts |
| Repos config | `REPOWISE_REPOS` secret key (comma-separated `owner/repo`) | Keeps repo list out of manifests; same secret as Forgejo token |
| Image registry | Forgejo container registry (`192.168.1.50:3000`) | Local registry; CI/CD via Forgejo Actions |
| Image tag | Pinned `:0.2.1` (no `:latest`) | Prevents ArgoCD stale-cache issues; version bump = new image build |

## IaC Changes

### `infra/docker/repowise/Dockerfile` (new)

`python:3.11-slim` image with `repowise==0.2.1` and `mcp-proxy` installed via pip. Installs
`git` for repo cloning. Copies `entrypoint.sh` as the container entrypoint.

### `infra/docker/repowise/entrypoint.sh` (new)

Bash script that:
1. Clones each repo from `REPOWISE_REPOS` into `/data/<owner>/<name>/`
2. Runs `repowise init --path <dest>` per repo (skips if `.repowise/` already exists)
3. Starts a background update loop: `git pull && repowise update` on `REPOWISE_UPDATE_INTERVAL` (default 3600s)
4. Execs `mcp-proxy --port 8080 -- repowise serve` as PID 1

### `infra/kubernetes/mcp-servers/repowise-mcp.yaml` (new)

Three K8s resources in one file:
- **PVC** `repowise-data`: `ReadWriteOnce`, `local-path`, 10Gi — preserves index and cloned repos
- **Deployment** `repowise-mcp`: single replica, image `192.168.1.50:3000/root/repowise:0.2.1`, mounts PVC at `/data`, refs `repowise-mcp-secret` + `mcp-servers-config`
- **Service** `repowise-mcp`: LoadBalancer, exposes port `8080` (MCP) and `7337` (web UI)

Readiness probe: `tcpSocket :8080`, `failureThreshold: 60` × `periodSeconds: 30` = 30-min grace window for init.

### `infra/.forgejo/workflows/repowise-image.yml` (new)

Forgejo Actions workflow that builds and pushes the Docker image on pushes to `main`
affecting `docker/repowise/**`. Requires Actions secret `FORGEJO_REGISTRY_TOKEN`
(PAT with `write:package` scope).

### `infra/ansible/roles/mcp-servers/tasks/main.yml` (modified)

Add `Create repowise-mcp secret` task after the proxmox-mcp task. Creates a K8s secret
`repowise-mcp-secret` with keys `token` (Forgejo PAT) and `repos` (comma-separated repo list).
Vars: `repowise_forgejo_token`, `repowise_repos` (add to `inventory/group_vars/all/secrets.yml`).

## Deployment Workflow

```
1. One-time manual prerequisites (see below)
2. Branch: feat/mah-76-repowise-k8s in infra repo
3. Create all 4 new files + modify Ansible tasks
4. Push branch; Forgejo Actions builds + pushes image to local registry
5. Run Ansible playbook locally against K3s node to create repowise-mcp-secret
6. ArgoCD auto-syncs repowise-mcp.yaml; pod starts
7. Watch logs: sudo k3s kubectl logs -n mcp-servers -l app=repowise-mcp -f
8. After ~25 min, logs show "[repowise] Init complete"
9. Verify endpoints; add MCP to Claude Code
10. Merge PR
```

### One-Time Manual Prerequisites

These must exist in the cluster before ArgoCD can deploy:

1. **K3s insecure registry** — verify `/etc/rancher/k3s/registries.yaml` on `.60` has a mirror
   entry for `http://192.168.1.50:3000`; add + restart k3s if missing

2. **imagePullSecret** (if Forgejo requires auth for registry pulls):
   `sudo k3s kubectl create secret docker-registry repowise-registry-secret --namespace mcp-servers --docker-server=192.168.1.50:3000 --docker-username=root --docker-password=<token>`

3. **`FORGEJO_REGISTRY_TOKEN` Actions secret** — add to Forgejo infra repo settings
   (PAT with `write:package` scope)

4. **Run Ansible** — after adding `repowise_forgejo_token` and `repowise_repos` to vault,
   run `ansible-playbook ansible/playbooks/mcp-servers.yml` to create the K8s secret

## Verification

| Check | Command / URL |
|---|---|
| Image built | Forgejo Actions run green; image visible in Forgejo packages |
| Pod running | `sudo k3s kubectl get pods -n mcp-servers` → `repowise-mcp-*` Running |
| Init complete | Pod logs show `[repowise] Init complete` |
| MCP endpoint | `curl http://<LB-IP>:8080/mcp` → MCP capabilities JSON |
| Web UI | `http://<LB-IP>:7337` → RepoWise dashboard |
| Claude Code | `claude mcp add repowise --transport http http://<LB-IP>:8080/mcp` → tools available |

## Risks

| Risk | Mitigation |
|---|---|
| `repowise serve` exits before init completes | mcp-proxy restarts subprocess; pod stays alive (log noise only) |
| `.repowise` is not the correct init-complete marker | Verify after first `docker run`; adjust entrypoint check if needed |
| `mcp-proxy` binary name differs post-install | Run `pip show mcp-proxy && which mcp-proxy` inside built image to confirm |
| OOMKill during init (embedding model load) | 2Gi memory limit; raise if pod dies during init |
| HuggingFace model re-download on every restart | Mount a second PVC at `/root/.cache` if restart latency becomes a problem |

## Follow-on Tickets

| Area | Description |
|---|---|
| Multi-repo MCP routing | One mcp-proxy per repo, each on its own LoadBalancer IP, for per-repo MCP endpoints |
| HuggingFace cache PVC | Add `/root/.cache` PVC to avoid ~500MB model re-download on pod restart |
| repowise upgrade path | Bump `repowise==x.y.z` in Dockerfile + K8s image tag together via single PR |
| K3s registries.yaml automation | Add Ansible task to manage `/etc/rancher/k3s/registries.yaml` for local registry |
