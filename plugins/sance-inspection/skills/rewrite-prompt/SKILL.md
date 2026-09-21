---
name: rewrite-prompt
description: Use when asked to rewrite, edit, improve, fix or update a Sance agent's prompt (main prompt, ping prompt, tree condition prompt, any prompt type) and save it to the platform. The write workflow around the create_prompt_version tool — the only mutating tool; requires explicit user approval of the exact text before saving.
---

# Rewrite a prompt and save it as a new version

Goal: produce a better version of an existing prompt and save it — safely, with the
user in control of every byte that goes live.

## Workflow

1. **Locate**: `locate_entity(agent_id=…)` if only an agent id is known; then
   `inspect_agent(…, area="prompts")` to see the inventory: which prompt is the live
   one for the language/modality, what references it (tree conditions, pings,
   campaigns), whether a newer prompt shadows it.
2. **Read the current text**: `prompt_text(product_id, prompt_id)`. Then read the
   `prompt-authoring` topic (`topic_doc`) once per session — it defines the core
   prompt anatomy, the cache-breakpoint rule, the capability-alignment table and the
   pre-save checklist. For a playbook, also read the `playbooks` topic (body
   structure, description-as-trigger, no Jinja). The English-instructions /
   Russian-surface split described there is a suggestion, not a rule: keep the
   prompt's current language unless the user asks to change it.
3. **Understand the constraints before writing**: what the agent can actually do
   (`agent_config` area: tools, calendar, files, clarification), tree node names the
   prompt refers to (`dialogue_tree` area), and the client's requirements if the user
   gave them. A prompt must never instruct the agent to do something the config
   disables (see `review-agent-config`).
4. **Draft** the new version following the anatomy and checklist from the
   `prompt-authoring` topic. Preserve: Jinja variables that exist in the lead's CRM
   fields (below the marker only), the cache-breakpoint marker, the output language,
   the tone the client asked for, every client-facing Russian string verbatim. Do
   not "consult file X" or reference other prompts — the agent cannot. If the prompt
   is a monolith (scripts for every situation, >40k chars), propose
   `/decompose-into-playbooks` instead of polishing it.
5. **Present the COMPLETE new text** to the user, plus a short list of what changed
   and why. Ask explicitly: create as a disabled version, or create and enable?
6. **Only after an explicit yes**: `create_prompt_version(product_id, prompt_id,
   content, description="<what changed>", enable=<as the user said>)`. Claude Code
   will ask the user to permit the call — expected.
7. **Handle findings**: if the tool blocks on checks, show them verbatim. Fix critical
   ones and re-present the text. For warnings/severe, ask whether to proceed; pass
   the codes in `acknowledged_checks` only with consent.
8. **Report**: version number, enabled or not, and where to review it in the
   dashboard (the prompt's version history). If disabled, remind that nothing changed
   live until it is enabled.

## Never

- Never call `create_prompt_version` without showing the full text and getting a yes.
- Never set `enable=true` on your own initiative.
- Never rewrite a prompt you have not read in full via `prompt_text`.
- Never touch prompts the user did not name — a "rewrite the prompts" request means
  list candidates first and confirm which ones.
