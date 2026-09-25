---
name: sance-inspection
description: Use when working with the Sance AI platform through the sance-ai MCP server — orientation, connection/auth troubleshooting, choosing the right investigation workflow, and etiquette for the inspection tools. Start here for any Sance platform question.
---

# Sance AI Inspection — start here

Sance AI builds LLM sales/support agents for businesses. Agents are configured with
prompts, a dialogue state tree, scheduled follow-ups (pings), prepared messages, CRM
integrations, knowledge bases and channels — and misconfigurations between those parts
are the most common cause of "the bot behaves wrong". The `sance-ai` MCP server gives
read-only access to all of it: effective configuration with curated documentation and
derived facts, plus full execution traces of any dialogue. It cannot change anything.

**All judgment is yours.** The server returns facts and docs, never verdicts — that is
by design. Numbers must come from tool results, never from memory.

## First contact in any session

Call `platform_overview` once before anything else. It returns the entity map, the
symptom→area investigation guide, and the live catalogs of inspection areas and doc
topics. Do not guess area or topic names — read them from there.

**Resolving ids** — the platform has thousands of products, so:

- Given an agent, dialogue or lead id ("what's up with agent 144") →
  `locate_entity(agent_id=144)` — ONE call returns the owning product and basics.
  NEVER search for an entity by iterating `product_overview` across products.
- Given a product id (users read it from the dashboard URL) →
  `product_overview(product_id)` lists its agents, channels and campaigns.
- Given only a name ("the Acme bot") → ask the user for the product id or a dashboard
  URL; do not guess.

## Choosing the workflow

| Situation | Skill |
|---|---|
| "Is this agent configured coherently? Does it match what the client asked for?" | `review-agent-config` |
| "Why did the bot say/do that in THIS dialogue? Why no escalation / no ping / wrong stage?" | `debug-dialogue` |
| "Nothing works / channels dead / CRM not syncing / leads not importing" | `check-agent-setup` |

## Tool etiquette (context economy)

- **Big payloads travel as files.** `export_dialogue` and `export_prompt` return a
  one-shot download link instead of the payload; `curl -sf -o .sance/<path> "<url>"`,
  then read selectively (`jq`, `grep`, the prompt outline's offsets). Links work once
  and expire in ~10 minutes — on failure just ask the tool again. Keep `.sance/` in
  `.gitignore`. Sending an edited prompt back works the same way in reverse:
  `prepare_prompt_upload` → `curl -sf -T <file> "<url>"` → write tool with
  `upload_id` + `content_sha256`.
- To skip the permission prompt on every download, the user can allow
  `Bash(curl -sf *mcp-inspection.ru.sance.ai*)` in their Claude Code
  settings — suggest it, never edit their settings yourself.
- Inline fallbacks (`debug_dialogue` with a narrow `sections` list, `prompt_text`)
  are for small payloads or clients without a shell. A single call's system prompt
  is often tens of thousands of tokens: fetch it with `export_llm_message_instructions`
  (a file), not `llm_message_instructions` (inline), unless you have no shell.
- Saving from an upload is idempotent: repeating a write with the same `upload_id`
  returns the original result (`"replayed": true`) and writes nothing — safe to retry
  after a timeout. A new edit needs a new `prepare_prompt_upload`.
- `inspect_agent`: one area at a time; only the areas the question needs.
- Platform docs: fetch via `topic_doc` on demand — do not ask the user to paste docs.

## Install / onboarding (for helping a colleague)

```
/plugin marketplace add sance-ai/inspection-toolkit
/plugin install sance-inspection@sance
```

First tool use opens a browser: a consent page, then Yandex login — use the work
(`@ru.sance.ai`) Yandex account. That's it; tokens live in the OS keychain and refresh
silently for ~30 days.

## Auth troubleshooting

1. Tools failing with auth/401 errors after weeks of working → the refresh token
   expired. Run `/mcp`, select `sance-ai`, choose re-authenticate — one click if the
   browser is logged into Yandex.
2. Browser flow completes but tools still refuse → the gate rejected the account:
   it must be a `@ru.sance.ai` Yandex email AND an active admin user on the platform
   (logging into the platform dashboard once with the same Yandex account creates it).
3. The browser flow itself errors before/at Yandex → server-side problem; escalate to
   engineering, do not retry in a loop.
4. Never `claude mcp remove sance-ai` to fix auth — the registration comes from this
   plugin, not from manual config; removing changes nothing about tokens.

## Writing prompts — the two mutating tools

`create_prompt_version` (a new version of an existing prompt) and `create_prompt` (a
brand-new prompt row — meant for playbooks) are the only tools that change anything.
Rules, no exceptions:

1. Read the `prompt-authoring` topic (and `playbooks` for playbooks) via `topic_doc`
   BEFORE drafting. They define the structure, the cache-breakpoint rule and the
   capability checks. The "logic in English, surface in Russian" idea in them is a
   suggestion with too little data behind it — never impose it; keep the prompt's
   existing language unless the user asks to switch.
2. The user must have reviewed the COMPLETE new text — as a local file they open
   (the default for anything big) or in chat for short prompts — and said an explicit
   "yes, create it" before you call. Saving from a file goes by `upload_id` +
   `content_sha256`, which pins the exact bytes they reviewed. The tool call itself
   will also prompt the user for permission — intended, never try to avoid it.
3. `create_prompt_version`: leave `enable` false unless the user said to make it
   live. A disabled version changes nothing until enabled in the dashboard.
4. `create_prompt`: its first version is enabled by the platform. For a playbook that
   means "offered to new dialogues now". For any other type it would SHADOW the
   agent's live prompt instantly — the tool refuses unless `replace_live=true`; never
   pass that on your own initiative; prefer a new version of the existing prompt.
5. Content-check findings: relay verbatim. Critical (broken Jinja) must be fixed.
   Severe/warning may be acknowledged via `acknowledged_checks` only after the user
   reads them and agrees.
6. Workflows: `/rewrite-prompt` for one prompt, `/decompose-into-playbooks` for
   turning a monolith into a core + playbooks.

## Boundaries

- Everything else is read-only; to change any other configuration, use the dashboard.
- Do not state a cause a tool result does not support — the debug bundle's legend
  lists what is NOT persisted; treat those as unknowable, say so explicitly.
- Anything involving money, deletion, or client-visible changes → hand off to a human.
