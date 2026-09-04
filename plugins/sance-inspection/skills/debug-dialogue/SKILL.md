---
name: debug-dialogue
description: Use when investigating a specific Sance dialogue — why the bot replied the way it did, why it didn't escalate to a human, why a ping/follow-up never fired, why the dialogue is on the wrong tree node or stage. Timeline-first debugging with the sance-ai debug_dialogue tool.
---

# Debug one dialogue

Goal: reconstruct what actually happened in one dialogue and name the cause with
evidence. The dialogue id comes from the dashboard; if the product id is unknown,
resolve it with `locate_entity(dialogue_id=...)` — one call, never a product search.

## The method: timeline first, prompts last

1. **First fetch — cheap and wide**: `debug_dialogue` with
   `include_instructions=false` and `sections=["lead_events", "escalation"]`.
   Read the embedded `legend` — it defines every event type, the link fields, and
   (critically) `not_persisted`: the list of things that are unknowable after the
   fact. Never assert a cause from that list.
2. **Build the timeline**: lead_events give you mutes (with reasons), stage changes,
   tree moves, ping skips, campaign assignment. Each event has a `scope` — events
   from sibling dialogues of the same lead often explain "mysterious" state.
3. **Narrow to the suspicious turn**, then fetch what that question needs:

| Question | Sections / tools |
|---|---|
| why did it SAY that | `events` (the llm_run for the turn), then `llm_message_instructions` for that one run — the historical ground truth of what the model saw |
| why no escalation | `escalation`: gate_decisions rows (mode/label/armed/reason). No rows + mode=off means the gate never ran. Budget exhaustion is never persisted — treat as possible, not provable |
| why no ping / follow-up | `pings`: the `eligibility_replay` evaluates every production clause against CURRENT state and names `first_failing_clause`. Runtime-only gates (debounce, chain timing, throttling) are listed in the legend as non-replayable |
| why this tree node | `dialogue_tree`: `movements` are the historical record; `arbiter_turns` carry the per-node condition verdicts and the arbiter's reason, recovered from stored LLM runs |
| what data did extraction write | `lead_snapshot`: CRM field values with per-row source/actor provenance, stage movements, refusal assignments (these store reason + confidence) |
| did retrieval fire, with what | `rag`: per-run gate decision, rewritten query, selected units |
| what WOULD the prompt be now | `prompt_dry_run` — current config only; never evidence about the past |

4. **Compare past vs present deliberately.** `instructions` from an llm_run = what
   ran THEN. `prompt_dry_run` = what would run NOW. Agent config, tree nodes and A/B
   bucket configs are unversioned — only prompt versions are pinned per message.

## Hard rules

- A cache-hit turn (`llm_cache_hit`) has NO stored prompt — say "served from cache,
  prompt unrecoverable", don't reconstruct it.
- A missing llm_run is not proof nothing ran (best-effort writes) — phrase as
  "no record", not "did not happen".
- Detected language, sanitizer verdicts, router picks, delivery-failure text: not
  persisted. If the cause seems to live there, say exactly that and stop.
- `eligible_now=true` in the ping replay does NOT mean it was eligible yesterday —
  the replay runs against current state; combine with lead_events for history.
- One section erroring (`section_error`) degrades that section only — report the
  gap, use the rest.

## Report format

Chronological story of the incident with event timestamps, then the cause (or the
shortest list of candidate causes with what would distinguish them), then the fix or
escalation. Quote event ids / run ids so an engineer can pick up exactly where you
stopped.
