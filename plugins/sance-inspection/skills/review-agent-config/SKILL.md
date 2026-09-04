---
name: review-agent-config
description: Use when reviewing a Sance agent's configuration for coherence — prompts vs dialogue tree vs pings vs prepared messages vs settings — or checking it against a client's requirements document. The full config-review methodology for the sance-ai MCP tools.
---

# Review an agent's configuration

Goal: find places where the agent's parts contradict each other or promise things the
configuration cannot deliver. The tools hand you facts; the incoherences are yours to
spot. Work through the phases in order — later phases need the earlier context.

## Phase 1 — orient

1. `platform_overview` (once per session).
2. `product_overview(product_id)` → confirm which agent is meant, note its channels
   and campaigns.

## Phase 2 — collect the five core areas

Run `inspect_agent` for: `agent_config`, `prompts`, `dialogue_tree`,
`schedule_and_pings`, `prepared_messages`. Read each result's `docs` before its
`state`/`facts` — the docs explain what the fields mean and their silent-failure
traps. Add `crm_integrations` when the review touches stages/CRM, `knowledge_base`
when it touches RAG.

## Phase 3 — read the prompts that matter

From the prompts area result, identify: the main prompt for the default language, and
every prompt referenced by tree conditions, ping steps and campaigns (`referenced_by`
maps). Fetch bodies with `prompt_text` — only the ones you need.

## Phase 4 — the coherence checklist

Structural (the facts usually flag these directly):

- Dangling references: tree `llm_db` conditions pointing at missing prompts, message
  sources pointing at missing prepared messages/files, prompts with no enabled version.
- Shadowed prompts: a newer prompt of the same type/lang/modality silently wins —
  is the one the client edits the one that actually runs?
- Unreachable tree nodes (no working conditions, no root), nodes without
  `llm_description` (the arbiter can't reason about them).
- Template variables used in prepared messages/prompts that don't exist among the
  lead CRM fields (renders as a hole, no error).
- Ping steps whose timing or `dialogue_init_msg_mode` can never fire for this
  agent's traffic shape.

Semantic (read the texts, think):

- The prompt instructs behavior the config disables: booking meetings with
  `calendar_enabled=false`, escalation phrasing with no human-clarification config,
  sending files not in `files_to_send`, "consult document X" where no such
  knowledge/tooling exists. A prompt cannot "go to another prompt" — any such
  instruction is dead text.
- Tree states that overlap semantically — two nodes whose conditions both match the
  same user intent; the arbiter will pick inconsistently (canvas proximity is a real
  tie-breaker input).
- Double-messaging: a ping and a tree action (or stage-triggered message) covering
  the same trigger window.
- Stage names in prompt text diverging from the actual platform/CRM stage mapping in
  `agent_config.pipeline_stages`.
- Language mismatches: prompt language vs agent default vs the client's audience.

## Phase 5 — against the client's requirements

If the user supplies a requirements document, map each requirement to the config
element that implements it and mark: implemented / partially / contradicted / absent.
Quote the config evidence (ids, field values) for every verdict.

## Report format

Lead with a one-paragraph verdict. Then findings grouped by severity — "breaks now" /
"will break under condition X" / "cosmetic" — each with the evidence (tool + field)
and the concrete fix in dashboard terms. Skip the checklist items that passed;
mention only what the reader should act on.
