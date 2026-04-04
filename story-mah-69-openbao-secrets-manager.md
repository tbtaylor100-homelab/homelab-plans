---
jira-story: MAH-69
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-04-04
context-refs:
  - epic-1-homelab-platform-iac.md
---

# Story MAH-69: Self-Hosted Secrets Manager — OpenBao on K3s

## Why

Credentials (Proxmox API token, SSH public key, MinIO access keys) are managed as
gitignored files on the operator's Mac. This creates two blocking problems:

1. **CI is broken** — `tofu-plan.yml` references `${{ secrets.PROXMOX_ENDPOINT }}`
   etc. but no values exist in Forgejo. Every PR touching `opentofu/` fails immediately.
2. **No per-environment isolation** — no rotation mechanism, no audit trail, no way
   to scope credentials to specific consumers (CI vs operator vs future collaborators).

## What We're Building

OpenBao (open-source HashiCorp Vault fork) deployed to K3s via ArgoCD, with:

- **AppRole auth** for the CI runner (on VM .50) — two Forgejo secrets replace five,
  and the actual infra credentials never touch Forgejo at all
- **Token auth** for the local operator — root token from `bao operator init`, stored
  in a password manager
- **KV v2 secret engine** at `secret/homelab/ci` — structured, versioned secret storage

## Candidate Evaluation

| Option | Self-hosted | K8s-native | Open source | Verdict |
|---|---|---|---|---|
| **OpenBao** | ✅ | ✅ | ✅ (MPL-2.0) | **Chosen** |
| HashiCorp Vault | ✅ | ✅ | ❌ (BUSL-1.1) | Rejected — licence changed 2023 |
| Infisical | ✅ | ✅ | ✅ | Good, but adds significant complexity; better fit for multi-team |
| Doppler | ❌ (SaaS) | — | ❌ | Rejected — external dependency |
| Forgejo repo secrets | ✅ | — | ✅ | No per-path ACL, no rotation, no audit log |

OpenBao was already referenced in `infra/ansible/inventory/group_vars/nas/secrets.yml.example`,
indicating prior intent. It is the natural choice.

## Architecture

```
Operator Mac                    VM .50 (Forgejo)              K3s .60
────────────                    ────────────────              ──────────────────────────
bao CLI                         act-runner process            ┌─ openbao namespace ─────┐
  └─ token auth ──────────────────────────────────────────────┤  OpenBao pod            │
                                                              │  Service: LoadBalancer  │
                                AppRole login ───────────────►│  IP: 192.168.1.210:8200 │
                                (role_id + secret_id          │  Storage: file (1 Gi)   │
                                 from Forgejo secrets)        └─────────────────────────┘
                                     │
                                     ▼
                                secret/homelab/ci
                                  proxmox_endpoint
                                  proxmox_api_token
                                  ssh_public_key
                                  minio_access_key
                                  minio_secret_key
```

### Key deployment decisions

| Decision | Choice | Reason |
|---|---|---|
| Deployment method | Helm chart via ArgoCD | Avoids hand-writing a StatefulSet; consistent GitOps pattern |
| Node count | Single (standalone mode) | Homelab; HA adds operational overhead with no benefit |
| Storage backend | `file` at `/vault/data` | Simplest; no external dependency; upgrade to Raft if HA needed |
| TLS | Disabled | Internal network only (homelab LAN); add TLS if exposure grows |
| Service type | `LoadBalancer` (MetalLB) | Runner on .50 needs network access to K3s; fixed IP 192.168.1.210 |
| Vault Agent Injector | Disabled | Not needed; CLI injection in workflow is sufficient for now |
| Unseal | Manual | Most secure; pods restart rarely in homelab; auto-unseal is a follow-on |

### Auth strategy

| Consumer | Method | Credentials in Forgejo | Rotation |
|---|---|---|---|
| CI runner (on .50) | AppRole | `OPENBAO_ROLE_ID`, `OPENBAO_SECRET_ID` | Rotate `secret_id` via `bao write` |
| Local operator | Token | None (root token in password manager) | `bao token renew` or new token |
| Future: K3s runner | Kubernetes auth | None | Pod identity — no rotation needed |

**Why AppRole over Kubernetes auth for CI:** The Forgejo Actions runner runs as a
process on VM .50, not as a K3s pod. Kubernetes auth requires a pod service account
token. AppRole is specifically designed for this: machine-to-machine auth without
a cloud identity provider. When the runner moves to K3s (follow-on ticket), switch
to Kubernetes auth and remove both Forgejo secrets entirely.

### Secret injection in `tofu-plan.yml` and `tofu-apply.yml`

```
1. AppRole login → short-lived BAO_TOKEN (TTL 1h)
2. Single GET to /v1/secret/data/homelab/ci → all 5 secrets in one call
3. ::add-mask:: each sensitive value (suppresses from runner logs)
4. Write to $GITHUB_ENV → available as env vars in all subsequent steps
5. BAO_TOKEN discarded after job; never stored anywhere
```

Only `curl` and `jq` are used — both available in the standard `ubuntu-latest`
runner image. No binary installation required.

## IaC Changes

### `infra/argocd/apps/openbao.yaml` (new)

ArgoCD Application pointing to the OpenBao Helm chart repo
(`https://openbao.github.io/openbao`). Inline Helm values configure single-node
standalone deployment with `local-path` PVC and MetalLB IP `192.168.1.210`.

### `infra/.forgejo/workflows/tofu-plan.yml` (updated)

New `Fetch secrets from OpenBao` step added before `tofu fmt check`. Previous
`${{ secrets.PROXMOX_* }}` and `${{ secrets.MINIO_* }}` references replaced with
env vars populated from OpenBao. Only `OPENBAO_ROLE_ID` and `OPENBAO_SECRET_ID`
remain as Forgejo secrets.

### `infra/.forgejo/workflows/tofu-apply.yml` (updated)

Same `Fetch secrets from OpenBao` step added. Stale `-backend=false` flag in
`tofu init` removed (MinIO backend is live per ADR-003). `TF_VAR_*` env vars
now sourced from OpenBao.

## Bootstrap Procedure (one-time, post-ArgoCD-sync)

After ArgoCD syncs `openbao.yaml` and the pod is Running:

```bash
export BAO_ADDR=http://192.168.1.210:8200

# Step 1: Initialise — 1-of-1 for homelab simplicity
bao operator init -key-shares=1 -key-threshold=1
# Save the Unseal Key and Root Token in your password manager immediately.

# Step 2: Unseal
bao operator unseal <unseal-key>

# Step 3: Authenticate
export BAO_TOKEN=<root-token>

# Step 4: Enable secrets engine
bao secrets enable -path=secret kv-v2

# Step 5: Enable AppRole auth
bao auth enable approle

# Step 6: Write CI policy (read-only access to homelab/ci path)
bao policy write ci-read - <<'EOF'
path "secret/data/homelab/ci" {
  capabilities = ["read"]
}
EOF

# Step 7: Create CI role
bao write auth/approle/role/ci-runner \
  token_policies="ci-read" \
  token_ttl=1h \
  token_max_ttl=2h \
  secret_id_ttl=0

# Step 8: Retrieve role credentials (add both to Forgejo repo secrets)
bao read auth/approle/role/ci-runner/role-id         # → OPENBAO_ROLE_ID
bao write -f auth/approle/role/ci-runner/secret-id   # → OPENBAO_SECRET_ID

# Step 9: Write the infra credentials
bao kv put secret/homelab/ci \
  proxmox_endpoint="https://192.168.1.12:8006/" \
  proxmox_api_token="root@pam!opentofu=<uuid>" \
  ssh_public_key="ssh-ed25519 AAAA... operator@mac" \
  minio_access_key="<opentofu-access-key>" \
  minio_secret_key="<opentofu-secret-key>"

# Verify
bao kv get secret/homelab/ci
```

**Unseal after restarts:** If the OpenBao pod restarts, run:
```bash
bao operator unseal <unseal-key>
```
The unseal key is in your password manager. CI jobs will fail until OpenBao is
unsealed — this is expected and intentional (sealed = credentials locked).

## Follow-on Tickets

| Area | Description |
|---|---|
| CI runner → K3s | Move act-runner to K3s pod; switch OpenBao auth to Kubernetes backend; remove both Forgejo secrets |
| Auto-unseal | Investigate auto-unseal options (cloud KMS or K8s secret init container) to survive pod restarts without operator intervention |
| Operator credentials | Rotate operator from root token to a named userpass account with least-privilege policy |
| Ansible secrets | Migrate `group_vars/*/secrets.yml` files to OpenBao; replace gitignored local files |
| MinIO root credentials | `secrets.yml` contains plaintext MinIO root password — migrate to OpenBao |
| Secret rotation | Implement `secret_id` rotation schedule for the CI AppRole |
