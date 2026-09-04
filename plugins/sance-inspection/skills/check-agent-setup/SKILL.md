---
name: check-agent-setup
description: Use when a Sance agent seems broadly broken or freshly set up — channels not receiving/sending, CRM leads not importing or syncing, outreach not dispatching, knowledge base not answering. The readiness sweep across integration areas with the sance-ai MCP tools.
---

# Check an agent's setup (readiness sweep)

Goal: answer "is this agent wired up to actually run?" — connectivity and integration
health rather than conversational quality. Most "nothing happens" cases are one
silent gate failing early in a chain; the facts are built to expose exactly those.

## Phase 1 — orient and route

1. `platform_overview` → read the `investigation_guide`: it maps symptoms to areas.
   Trust it for routing — it is maintained with the backend.
2. `product_overview(product_id)` → the agent, its channels (active? inbound?),
   campaigns and their statuses. An agent with zero linked channels answers nothing,
   ever — check this before anything deeper.

## Phase 2 — sweep the integration areas

`inspect_agent` per area, guided by the symptom; for a full readiness audit run all:

- `agent_channels` — channel lifecycle flags, inbound routing, sender readiness.
  Watch for: no inbound-enabled channel (inbound leads can't be born), channels
  deactivated with cooldowns, archived channels still expected to work.
- `crm_integrations` — the three-layer stage mapping. Watch for: expired tokens
  (every CRM operation failing), leads on unmapped CRM stages (silently skipped on
  import), the all-null default `pipeline_stages` map (dialogue outcomes never move
  leads), growing unsynced-movement counts, refusal `movement_type_coverage` holes
  (technical closes assigning no reason).
- `outreach_campaigns` — campaign dispatchability facts (the intake gate evaluation
  uses the same function production uses; "accepts this campaign: false" is a fact,
  not a guess), entry-stage existence, contact-type distribution at the entry stage.
- `knowledge_base` — document readiness statuses, retrieval gating, budgets. Not
  ready ≠ broken: ingestion may still be running.
- `ab_tests`, `stage_triggered` — when the symptom involves experiments or
  stage-triggered messages.

Read each area's `docs` for the "why is nothing happening" ladders — they encode the
known failure chains in priority order.

## Phase 3 — cross-checks that span areas

- Outreach chain: campaign active → entry stage exists and receives leads (import or
  push) → contacts of the right type for the dispatch steps → channels with capacity.
  Any link broken = silent zero dispatches.
- Inbound chain: channel inbound-enabled → agent linked → main prompt exists for the
  default language (prompts area) → not muted by config gates (agent_config).
- CRM loop: import brings leads in → agent moves them (pipeline_stages) → push
  reflects moves back → sync doesn't fight external moves.

## When config looks fine

If every area is green but behavior is still wrong, the problem is per-dialogue, not
setup — switch to `debug-dialogue` with a concrete dialogue id. If a needed check
does not exist in any area's facts, say so explicitly ("not inspectable via MCP —
needs an engineer") instead of inferring.

## Report format

A readiness table (area → OK / broken → one-line evidence), then the broken chains in
dependency order — fix the earliest link first; downstream red often clears itself.
