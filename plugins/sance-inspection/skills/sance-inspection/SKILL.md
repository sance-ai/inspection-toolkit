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

- `debug_dialogue`: always start with `include_instructions=false` and a narrow
  `sections` list — a full bundle with prompts can be enormous. Fetch a single turn's
  system prompt with `llm_message_instructions` when you actually need it.
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

## Boundaries

- Everything is read-only; to change configuration, use the dashboard.
- Do not state a cause a tool result does not support — the debug bundle's legend
  lists what is NOT persisted; treat those as unknowable, say so explicitly.
- Anything involving money, deletion, or client-visible changes → hand off to a human.
