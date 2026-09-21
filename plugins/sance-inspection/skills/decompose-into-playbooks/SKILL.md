---
name: decompose-into-playbooks
description: Use when a Sance agent's main prompt is a monolith (tens of thousands of characters, scripts and examples for every situation) and the user wants it split into a short core prompt plus playbooks, or wants new playbooks created for an agent. The pilot-proven decomposition workflow around the create_prompt / create_prompt_version tools; every write needs the user's explicit approval.
---

# Decompose a monolith into a core prompt + playbooks

Goal: the same behavior at a fraction of the tokens per turn — a short core that binds
on every turn, plus playbooks the agent loads only when their situation is on. This is
the workflow two production pilots used; the rules come from the `playbooks` and
`prompt-authoring` topics — read both first, every time.

## Phase 1 — read

1. `locate_entity` / `product_overview` to fix the agent; `inspect_agent(prompts)` to
   find the live main prompt; `prompt_text` for the full monolith.
2. `inspect_agent(agent_config)`, `inspect_agent(dialogue_tree)`,
   `inspect_agent(knowledge_base)`: what the agent can actually do, the stage names
   the core must use, what knowledge already lives in RAG (never becomes a playbook).
3. `topic_doc("playbooks")` and `topic_doc("prompt-authoring")`.

## Phase 2 — sort every rule into a bucket

Walk the monolith top to bottom. Each rule goes to exactly one place:

| Bucket | Goes to | Test |
|---|---|---|
| decision rule, safety gate, escalation trigger | core (one line, stated once) | "must the model know this BEFORE any playbook is loaded?" |
| procedure for one funnel stage | a **stage** playbook | "does this describe HOW to run one step of the funnel?" |
| procedure for a branch that can occur at any stage | a **situational** playbook | "objections, refusal, a special client type…" |
| one-line situation («это бот?», outside hours) | core one-liner | "would the playbook be under ~300 characters?" |
| facts: FAQ answers, prices, program lists | knowledge base — not a prompt | "is it knowledge rather than instruction?" |
| ping texts, stage moves, CRM mechanics | the platform's own config | "does the platform already do this?" |
| the same rule restated for emphasis | delete the copies, keep one | — |

Present this sort as a table to the user before writing anything. Expect 10–15
playbooks; two or three fat ones is a sign the sort is wrong.

## Phase 3 — write

1. **Playbooks first.** Body structure: `# Name` → `## Procedure` → `## FAQ` (short
   facts, full wording stays in RAG) → `## Calibration dialogue` (Russian, verbatim
   from the client's material where it exists) → `## Self-check`. Client-facing
   strings stay Russian and verbatim. The language of the procedure text is the
   user's call — the pilots used English for it, but that is an untested suggestion
   (too little data); default to the monolith's own language and offer the switch.
   **No `{{ }}` anywhere.** 1–16k characters each.
2. **Description = trigger**, saying WHEN to use it, never what it contains, with the
   client's own Russian words in parentheses if the description is in English. This
   is the single most important line: a vague one makes the agent enable everything.
3. **Conditions** only for facts the platform knows before the model runs (a field
   present in the lead's CRM fields — check the variable list in
   `inspect_agent(prepared_messages)`); `null` otherwise.
4. **Core last**: the anatomy from `prompt-authoring` (output language → role → turn
   protocol → most-broken rules → stage map → step 1 → hard gates → style contract →
   escalation → pre-send check → `---cache breakpoint---` → lead data). It names the
   playbooks it expects per stage but never pastes their content or the catalog.
   Target 18–25k characters.

## Phase 4 — save, in this exact order, one approval per item

1. Stage playbooks first, in funnel order, then situational ones — `create_prompt`
   with `type="playbook"`, `name`, `description`, `content`, `crm_fields_conditions`.
   Ids ascend in creation order and the enabled-playbooks block is sorted by id, so
   this order makes the common funnel a shared cache prefix. Show each playbook in
   full and get a yes before each call.
2. The core LAST, and as a **new version of the existing main prompt**
   (`create_prompt_version`, `enable=false`), never as a new prompt row: a new
   `type="prompt"` row would go live the instant it exists, before the user has
   reviewed anything. The user enables the version in the dashboard when ready —
   or asks you to pass `enable=true` explicitly.
3. Relay every content-check finding verbatim; `vars_in_playbook` means a variable
   leaked into a playbook — fix, don't acknowledge.

## Phase 5 — hand-off

Report: the bucket table, the list of created playbooks (id, name, trigger line), the
core version number and that it is disabled, and what to watch after enabling
(tokens-in-context and cost per turn; `enable_playbooks` calls per dialogue should
stay far below the catalog size). Recommend an A/B campaign bucket pinned to the old
version vs the new one rather than a hard switch.

## Never

- Never create the core as a new prompt row (`create_prompt` with `type="prompt"`).
- Never create anything without showing its full text and getting an explicit yes.
- Never turn knowledge-base material into playbooks, or restate a core gate inside a
  playbook.
- Never paste the playbook catalog or lead data into the core — the platform injects
  both.
