---
jira-epic: MAHHAUS-42, MAHHAUS-43, MAHHAUS-44, MAHHAUS-45, MAHHAUS-46, MAHHAUS-47, MAHHAUS-48
status: active
authored-by: claude-sonnet-4-6
created-date: 2026-03-22
context-refs: []
---

# Plan of Plans: Living Knowledge & Agent Workforce System

## Context

**Problem being solved:** Context is lost every time you switch between LLM providers (Gemini → research, Perplexity → discovery, Claude Code → planning/dev). Each session starts cold. Research doesn't translate into action. Action doesn't update the underlying knowledge about the world.

**Desired outcome:** A system where a workforce of agents can help improve your home and homelab — where any LLM can pull the right context at any time, take action, and feed results back into a knowledge base that evolves over time.

**Key insight:** There is no single source of truth. Context is *distributed* (home specs, infrastructure state, goals, active JIRAs) and must be *aggregated* on demand. The goal is coherent aggregation, not centralization.

---

## System Layers (bottom-up)

Each layer is a prerequisite for the one above it. These become the epics.

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 4: Agent Workforce                               │
│  LLMs using MCPs to plan, act, hand off tasks           │
├─────────────────────────────────────────────────────────┤
│  LAYER 3: Work / Orchestration                          │
│  JIRA epics + issues as the canonical work record       │
│  Handoff contracts between LLMs (research → action)     │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: Knowledge / Context                           │
│  Versioned docs: home specs, infra state, diagrams      │
│  Living documents that update when JIRAs close          │
├─────────────────────────────────────────────────────────┤
│  LAYER 1: Infrastructure (IaC + CICD)                   │
│  All services version-controlled, Forgejo as source     │
└─────────────────────────────────────────────────────────┘
```

---

## Epics

### Epic 1: Homelab Platform — IaC Foundation
**JIRA:** MAHHAUS-42

**Why first:** Every other layer depends on being able to spin up, version, and reproduce services reliably. Currently infrastructure is implicit (manually configured). Agents cannot safely modify what isn't tracked.

**Important framing:** This epic is *not* scoped to the agent workforce system. The goal is a general-purpose homelab IaC platform that can host *anything*: a NAS, game servers, Plex, the agent system, and services not yet imagined. The agent workforce system is the first validated use case — the one that drives us to build the platform right — but the platform itself is broader.

**Scope:**
- Version-control all Proxmox provisioning, node configuration, and service definitions
- Establish a GitOps deployment model: changes to infra repo trigger automated deploys — no manual SSH
- Containerized services use Kubernetes-native packaging to expose a programmable API surface for future agent management
- Initial service inventory to migrate: NAS, game server(s), Plex, existing homelab services

**Key technology decisions** *(detailed comparisons in the Epic 1 plan):*
- **Container orchestration**: K3s (lightweight Kubernetes) — chosen over Docker Compose because agents need a programmable API to manage services, not a CLI tool
- **Service packaging**: Helm charts (templated, versioned, large public ecosystem)
- **GitOps/deployment**: ArgoCD (watches infra repo, reconciles cluster state automatically)
- Provisioning and configuration tooling TBD in Epic 1 planning session

**Done when:** Any service in the homelab can be reproduced from the infra repo with no manual steps.

---

### Epic 1b: Observability Platform
**JIRA:** MAHHAUS-43

**Why alongside Epic 1:** If higher layers depend on lower layers, you need visibility into failures *before* your family does. Observability must be deployed as part of the foundation, not retrofitted after things break. It is a platform concern, not an application concern.

**Scope:**
- **Metrics:** Prometheus + Grafana — node health, container resource usage, service availability
- **Logs:** Loki — centralized logs from all containers and VMs
- **Uptime:** Uptime Kuma — simple family-visible status page
- **Alerting:** Alertmanager → notification channel (Discord or similar) for critical failures
- **Host metrics:** Node Exporter on all Proxmox nodes
- All observability services deployed via IaC (dogfooding Epic 1)

**Done when:** A failing service generates an alert before anyone manually notices. All Proxmox nodes and running containers have metrics and logs visible in Grafana.

---

### Epic 2: Knowledge Base — Living Context Documents
**JIRA:** MAHHAUS-44

**Why second:** Before agents can act intelligently, the world they're acting on must be describable. This is the "base context" layer.

**Key principle:** Plans are also documentation and context. A plan captures the *why* behind decisions — information that future agents need just as much as current infrastructure state. Plans are not ephemeral session artifacts; they are first-class knowledge artifacts that must be versioned, reviewed, and accessible.

**Repo structure:**

| Repo | Purpose | Lifecycle |
|------|---------|-----------|
| `homelab-knowledge` | Living context: home, homelab state, ADRs | Updated when facts change or JIRAs close |
| `homelab-plans` | Plans: one file per initiative | Draft → PR review → merge = approved → closed when JIRA closes |

**`homelab-knowledge` scope:**
- `home/` — electrical, plumbing, HVAC, floorplan, appliance inventory
- `homelab/` — network topology, Proxmox node inventory, service registry
- `decisions/` — architectural decision records (ADRs); the *why* behind significant choices
- Each doc has YAML frontmatter: `last-updated`, `related-jira`, `confidence-level`

**`homelab-plans` scope:**
- One markdown file per initiative/epic
- Frontmatter per plan: `jira-epic`, `status`, `authored-by`, `created-date`, `context-refs`
- Plans are created as PRs — the PR review IS the plan review

**Done when:** Any LLM can open a PR in `homelab-plans`, have it reviewed, and merge to approve. Active plans are discoverable by any agent via the MCP context server.

---

### Epic 3: Context Aggregation Layer (MCP)
**JIRA:** MAHHAUS-45

**Why third:** Raw files in a repo are readable, but agents need *intelligent access* — "give me all context relevant to the kitchen electrical panel" not "read every file."

**Scope:**
- A custom MCP server (`context-server`) deployed via IaC, registered with ToolHive
- Capabilities:
  - `get_domain_context(domain)` — returns relevant docs for a domain
  - `search_context(query)` — semantic or keyword search across knowledge base
  - `update_document(path, content, jira_ref)` — write back updates when JIRAs close
  - `list_active_work(domain?)` — returns open JIRAs filtered by domain

**Done when:** Claude Code (and any MCP-capable LLM) can call `get_domain_context("home/electrical")` and receive an accurate, up-to-date summary without reading raw files manually.

---

### Epic 4: Work Orchestration — JIRA as the Action Layer
**JIRA:** MAHHAUS-46

**Why fourth:** With context accessible, the work tracking layer needs to be *connected* to context, not siloed.

**Scope:**
- JIRA issue templates per work type:
  - `research` — Gemini/Perplexity output, links to source material, "ready for planning" flag
  - `plan` — Claude output, links to research JIRA, executable steps
  - `execution` — Claude Code output, links to plan JIRA, commits/PRs attached
- Custom field: `context_refs` — which knowledge base docs this JIRA reads/modifies
- On JIRA close: trigger to update `context_refs` documents in `homelab-knowledge`

**Done when:** A Gemini research session produces a `research` JIRA. Claude Code reads that JIRA, produces a `plan` JIRA. Execution closes the loop and updates knowledge docs.

---

### Epic 5: Domain Architect Framework
**JIRA:** MAHHAUS-47

**Why before general workforce patterns:** Establishing specialized reviewing agents before general agent patterns ensures the workforce is built on the right model from the start.

**Model:** Agents create PRs → architects review → human approves (final authority always with human).

**Iteration 1 — Repo-scoped architects (MVP):**

| Architect | Repo | Responsibilities |
|-----------|------|-----------------|
| `architect-plans` | `homelab-plans` | Frontmatter completeness, JIRA ref validity, context-refs accuracy, ADR alignment |
| `architect-knowledge` | `homelab-knowledge` | Doc completeness, factual consistency, no conflicting state, ADR alignment |

**How they work:**
- Triggered by Forgejo Actions on PR open/update
- Receives: PR diff + all ADRs for the repo + repo standards doc
- Posts structured review: `✅ passes` or `⚠️ issues found` with specific callouts
- Advisory only — human can override

**Iteration 2 (future):** Domain-scoped architects for networking, electrical, Proxmox, etc.

**Done when:** Every PR triggers an architect review automatically. Human sees the assessment alongside the diff.

---

### Epic 6: Agent Workforce Patterns
**JIRA:** MAHHAUS-48

**Why last:** Once all layers exist, define the repeatable patterns that tie them together.

**Scope:**
- Agent playbooks in `homelab-knowledge/decisions/`:
  - Research session → JIRA capture
  - Research JIRA → Claude Code planning → PR in `homelab-plans`
  - Plan → architect review → human approval → execution → context update
- `CLAUDE.md` files per domain for context injection at Claude Code session start
- LLM routing matrix: which model/tool is appropriate for which task type

**Done when:** Starting a new improvement initiative follows a documented, repeatable flow from discovery to completion without losing context at any handoff.

---

## Sequencing & Dependencies

```
Epic 1 (IaC Platform) ─────────────────────────────────► must be first
     │
     ├──► Epic 1b (Observability) ─────────────────────► deploy via IaC immediately; validates platform
     │
     ▼
Epic 2 (Knowledge Base) ──────────────────────────────► can start parallel to Epic 1
     │
     ▼
Epic 3 (MCP Context Server) ──────────────────────────► requires Epic 1 (deploy) + Epic 2 (data)
     │
     ├──► Epic 4 (JIRA Work Orchestration) ────────────► can start parallel to Epic 3
     │
     ▼
Epic 5 (Domain Architect Framework) ──────────────────► requires Epic 2 (context) + Epic 3 (MCP)
     │
     ▼
Epic 6 (Agent Workforce Patterns) ────────────────────► requires all above
```

**Note:** Epics 1 and 1b are *platform* epics — they serve all homelab workloads. Epics 2–6 are specific to the agent workforce system. Future use cases can be added as new application epics on the same platform.

---

## Open Questions (for future planning sessions)

1. **Home physical data capture**: How do you get electrical diagrams, plumbing, floorplan *into* the knowledge base? (Scan/photograph → LLM extraction → structured doc is one path)
2. **Confidence decay**: Physical facts age. How do you track when a knowledge doc is "stale"?
3. **Multi-LLM handoff format**: What does a well-formed research handoff from Gemini → Claude look like?
4. **Access control**: Some home data (security systems, entry codes) shouldn't be in a plaintext repo. How is sensitive context handled?
