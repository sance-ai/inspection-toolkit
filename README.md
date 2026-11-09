# Sance Inspection Toolkit

Claude Code plugin for Sance AI CSMs: connects the platform inspection MCP server
and installs workflows for configuration review, dialogue debugging, prompt authoring,
synthetic-customer tests and client feedback.

## Install (once per laptop)

In Claude Code:

```
/plugin marketplace add sance-ai/inspection-toolkit
/plugin install sance-inspection@sance
```

Restart the session. The first tool call opens a browser — a consent page, then
Yandex login. Use your work (`@ru.sance.ai`) Yandex account. Done: tokens refresh
silently, no secrets to store.

## What you get

- MCP server `sance-ai` (https://mcp-inspection.ru.sance.ai): platform/product overviews,
  per-area agent inspection, prompt texts and authoring, dialogue debug bundles,
  historical LLM system prompts, platform docs, synthetic-customer tests and client feedback.
- Skills:
  - `sance-inspection` — orientation, auth troubleshooting, workflow routing;
  - `review-agent-config` — config coherence review methodology;
  - `debug-dialogue` — "why did the bot do that" investigation methodology;
  - `check-agent-setup` — integration/readiness sweep methodology.

  - `rewrite-prompt` — draft a new prompt version, get the user's explicit approval of
    the exact text, save it (disabled by default);
  - `decompose-into-playbooks` — split a monolith prompt into a core + playbooks and
    create them, one approval per item;
  - `test-with-synthetic-customers` — write simulated customers (by hand or from real
    dialogues), run them against the live agent or a draft version, triage the results
    and report findings;
  - `work-client-feedback` — read the feedback queue, investigate remarks, then reply,
    move cards and resolve comments after approval of the exact text or change.

Authoring knowledge (prompt anatomy, the English/Russian split, playbook rules) is
served by the MCP as the `prompt-authoring` and `playbooks` topics, and how
simulations work as the `simulations` topic. The skills tell Claude to read them first.

Everything is read-only except `create_prompt_version` and `create_prompt` (prompts)
and the synthetic-customer tools (`create_sim_personas`, `update_sim_persona`,
`draft_sim_persona`, `run_sim_personas` — test data and token spend), and client-feedback
writes (`reply_feedback_comment`, `move_feedback_card`, `set_feedback_comment_state`).
Writes require the user's approval; feedback changes are visible to the client under
the signed-in CSM's identity. All other configuration changes happen in the dashboard.

## Updating

Maintainers: bump `version` in `plugins/sance-inspection/.claude-plugin/plugin.json`
and push. Users: `/plugin marketplace update sance` (or wait for the periodic
refresh).

## Design principle

Skills carry *methodology only*. Platform knowledge (entity semantics, pipeline
order, field meanings) is served live by the MCP server from the backend repo's
canonical docs — so it is always current and never duplicated here. If a skill and a
tool result disagree, the tool result wins.
