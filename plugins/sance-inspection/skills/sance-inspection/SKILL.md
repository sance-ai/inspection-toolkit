---
name: sance-inspection
description: Use when working with the Sance AI platform through the sance-ai MCP server — orientation, connection/auth troubleshooting, choosing the right investigation workflow, and etiquette for the inspection tools. Start here for any Sance platform question.
---

# Sance AI Inspection — start here

Sance AI builds LLM sales/support agents for businesses. Agents are configured with
prompts, a dialogue state tree, scheduled follow-ups (pings), prepared messages, CRM
integrations, knowledge bases and channels — and misconfigurations between those parts
are the most common cause of "the bot behaves wrong". The `sance-ai` MCP server gives
read access to all of it: effective configuration with curated documentation and
derived facts, plus full execution traces of any dialogue. It changes only three things:
prompts (new versions, new playbooks), synthetic-customer tests (personas and runs) and client
feedback in the markup interface (replies to the client's comments, kanban card moves, resolving
a remark — all visible to the client). All of them need the user's go-ahead.

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
- Given a question about MANY dialogues ("find ones stuck on node X last week", "muted
  dialogues with refusal reason Y", research across a sample) → `find_dialogues(product_id,
  …filters)`: the dashboard's dialogue list with its filters, returning dialogue ids
  plus non-personal facts (stage, node, mute, A/B bucket, ticket, counts) — never names,
  contacts or texts. Get filter ids from `dialogue_filter_options`. Narrow by agent and
  dates; `reply_type` defaults to dialogues where the customer replied (pass `"all"`
  for unanswered outreach); synthetic customers are excluded unless
  `include_simulated=true`. Then export only the dialogues worth reading.
- Given a question about where some TEXT is used ("which prompts mention the client's
  job title", "where do we promise a discount") → `search_agent_text`, see Completeness.

## Completeness — every item, and say what you checked

A question about "which / where / all / is X used anywhere" is about EVERY item, never a
sample. Answering from part of the data is acceptable only when you say so first.

- **Find text with one call.** `search_agent_text(product_id, query)` scans every prompt
  of the product (live versions: enabled + pinned by active A/B buckets; `versions="all"`
  for history), every playbook catalog line, every prepared message and every
  knowledge-base unit. Never read prompts one by one to find a word. Quote its `summary`
  in your answer: it states what was searched and what was not (prompts without an
  enabled version, documents with no units yet). Search a stem without `whole_word`
  («должност» finds «должность», «должности»); a CRM field in a template is found by its
  key as written («Должность» in `{{ crm_fields['Должность'] }}`). Results are paged:
  if `matched_items` is larger than what you got, fetch the next `page`.
- **Read many prompts with one download.** `export_prompts(product_id)` returns every live
  prompt as `prompt_<id>_v<version>.md` in one zip, plus a manifest (version, sha256,
  whether it runs — a shadowed prompt does not).
- **Local copies are not the truth.** A file in `.sance/` may be days old. Before you
  answer from it, check it against the `export_prompts` manifest (it comes back inline
  in the tool result, no download needed): a file whose name or sha256 is not in the
  manifest is not the live text — export it again.
- **Count before you conclude.** Before the final answer, compare what you checked with
  the inventory (the search `coverage`, or `inspect_agent` area `prompts`). If anything
  was left out — a document not processed yet, an agent you skipped, a source the search
  does not cover (tree-node texts, ping texts, outreach templates) — say so at the START
  of the answer, not when asked.
- **Never trade coverage for economy.** Save context with files, `grep` and `jq` — never
  by looking at less than the question asks.

## Choosing the workflow

| Situation | Skill |
|---|---|
| "Is this agent configured coherently? Does it match what the client asked for?" | `review-agent-config` |
| "Why did the bot say/do that in THIS dialogue? Why no escalation / no ping / wrong stage?" | `debug-dialogue` |
| "Nothing works / channels dead / CRM not syncing / leads not importing" | `check-agent-setup` |
| "Test the agent with simulated customers / did this prompt change break anything / compare the draft with the live version / replay these real dialogues" | `test-with-synthetic-customers` |
| "What did the client complain about / work through the feedback queue / answer the client's comments" | `work-client-feedback` |

## Tool etiquette (save context, never coverage)

- Client feedback: `list_feedback` is the queue (open comments, board order, paged); a dialogue's
  comments and ratings come with `debug_dialogue` / `export_dialogue` section `feedback`. Read
  `topic_doc("feedback")` before the first write.
- **Knowledge base is readable, not writable.** `export_knowledge_document` downloads a
  document as uploaded (`format="original"`: PDF/XLSX/MD… — use your own file-reading
  tools on it), as the extracted text the bot's knowledge is built from
  (`format="text"`), or as its retrievable units (`format="units"`). Find document ids
  via `inspect_agent` area `knowledge_base`. Changes happen in the dashboard.
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
- `inspect_agent`: one area at a time; only the areas the question needs — and all of
  them when the question is about all of something.
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
brand-new prompt row — meant for playbooks) are the only tools that change the agent.
The synthetic-customer tools (`create_sim_personas`, `update_sim_persona`,
`draft_sim_persona`, `run_sim_personas`) change only test data, but spend the company's
tokens: they follow the `test-with-synthetic-customers` workflow.

The client-feedback tools (`reply_feedback_comment`, `move_feedback_card`,
`set_feedback_comment_state`) change nothing in the agent, but the CLIENT sees them in the
markup interface: they follow the `work-client-feedback` workflow, and each needs the user's
explicit go-ahead on the exact reply text or change.

Prompt rules, no exceptions:

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
  Client feedback writes are the one client-visible exception — see `work-client-feedback`.
- Do not state a cause a tool result does not support — the debug bundle's legend
  lists what is NOT persisted; treat those as unknowable, say so explicitly.
- Anything involving money or deletion → hand off to a human. Client-visible feedback changes
  happen only through the client-feedback tools, only with the user's explicit approval of the
  exact text or change — never on your own initiative.
