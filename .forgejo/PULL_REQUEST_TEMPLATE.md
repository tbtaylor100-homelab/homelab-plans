## Plan Frontmatter Checklist

Verify the plan file includes complete and valid frontmatter:

- [ ] `jira-epic` — references a valid MAHHAUS epic key (e.g. `MAHHAUS-42`)
- [ ] `status` — set to `draft` (new plans should not be merged as `active`)
- [ ] `authored-by` — identifies the LLM or agent that produced this plan
- [ ] `created-date` — ISO 8601 date (e.g. `2026-03-22`)
- [ ] `context-refs` — lists knowledge docs this plan reads or modifies (empty list `[]` is valid if none)

## Plan Content Checklist

- [ ] **Why** — explains why this plan exists and what problem it solves
- [ ] **Scope** — clearly bounded; does not drift into Epic 1 planning inside a plan-of-plans
- [ ] **Done when** — includes a concrete, verifiable definition of done
- [ ] **Decision rationale** — any technology or approach choices include a brief *why*, not just *what*
- [ ] **ADR alignment** — does not contradict any existing ADR in `homelab-knowledge/decisions/`

## Scope Check

- [ ] This plan operates at the correct abstraction level (not too detailed, not too vague)
- [ ] Implementation details are deferred to per-epic plans, not included here

## Summary

<!-- Briefly describe what this plan covers and why it is being proposed now -->

## JIRA Reference

<!-- Link to the JIRA epic or issue this plan belongs to -->
