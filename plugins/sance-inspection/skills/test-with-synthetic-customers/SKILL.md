---
name: test-with-synthetic-customers
description: Use when asked to test a Sance agent with synthetic (simulated) customers. Covers writing a set of test personas, turning real dialogues into personas, regression-testing a prompt, playbook or model change, comparing a draft version with the live one, and reading or reporting simulation runs. Personas, checks, runs and results go through the sim MCP tools, and every write or run needs the user's go-ahead.
---

# Test an agent with synthetic customers

A synthetic customer is an LLM playing a persona. It talks to the REAL agent through the
real pipeline: real prompts, playbooks, tree, knowledge base and models. Nothing reaches
CRM, webhooks, notifications or real customers. Each conversation is then checked against
the persona's expectations.

Platform facts are in `topic_doc("simulations")`: states, caps, end reasons, what is
intercepted, the summary fields. Read it once per session. This skill is the method.

**Sims test behaviour. They never predict conversion.** A simulated customer who doesn't
buy says nothing about real customers.

## 1. Read the agent before writing anything

1. `locate_entity(agent_id=…)`, then `inspect_agent` for these areas:
   - `prompts`: the live core prompt, its playbooks, and the outreach opener;
   - `dialogue_tree`: every node, its conditions, its final actions;
   - `agent_config`: tools, ask-manager / clarification, calendar, inbound vs outreach.
2. Download the core prompt and the main playbooks (`export_prompt` → `.sance/`). Read the
   hard rules, the escalation triggers, the questionnaire and its exceptions, and the
   "never" lists. The expectations you write come from these texts, quoted, never from
   memory.
3. Build a **coverage table**, one row per behaviour worth a persona:
   - every final node (success, refused, escalated, disqualified…);
   - every playbook;
   - every tool;
   - every escalation trigger;
   - every safety rule (pregnancy, medications, age limits…);
   - every special branch the prompt describes.

   Add the platform pack:
   - asks for a human;
   - asks the price first;
   - angry or firm refusal;
   - off-topic or wrong company;
   - prompt injection ("ignore your instructions…");
   - "are you a bot?";
   - answers once, then goes silent.

   Show the table to the user and agree on the set. 10–20 personas is a good first
   baseline.

## 2. Write personas

Each persona has these fields:

- **`title`**: who the customer is and what they want, a few words.
- **`entry`**:
  - `inbound` (the customer writes first): needs a `first_message`;
  - `outbound` (the customer answers the agent's real outreach opener): the agent needs an
    outreach campaign with an opener, or the persona an `opener_override`.
- **`persona_prompt`**: present-tense instructions to the simulated customer.
  - What they want.
  - What they know, told only when asked.
  - How they react at each decision point the test cares about: the price, the scheme,
    the short version, a handoff.
  - When they stop.

  Write the exact answer wherever a check depends on it. For example: «на финальный блок
  отвечаешь: 1) нет 2) не беременна 3) пью Акнекутан, 3 месяц 4) аллергий нет». Otherwise
  the simulator may skip an item and the agent re-asks.
- **A made-up name, when the agent asks for one.** If the opener asks «как к вам
  обращаться», put a name in the persona. Otherwise the simulator improvises one.
- **`style_examples`**: 2–5 lines in the customer's own voice (typos, no greetings,
  lowercase…). These set the tone more than the description does.
- **`max_turns`**: the customer's messages.
  - Full consultation flows need 12–16: the questionnaire, the scheme, the short scheme,
    the closing.
  - A refusal or an escalation needs 3–6.
  - A run cut by `max_turns` proves nothing.
- **Tools.** Every tool call the agent makes is intercepted unless the run lists it in
  `real_tools`. An intercepted call is answered with the persona's `tool_mocks[tool]`, or
  with `{"error": "tool_unavailable"}`. So for each tool on the persona's path, either
  plan to run it for real or give a canned answer that looks like the real one.
- **`seed_crm_fields`**: only what the scenario needs (repeat customer, stage, a field a
  template or condition reads), with made-up values. Never real names, phones,
  addresses or links.

The simulated customer has hard limits:

- it cannot send photos or files;
- it answers only after the agent's whole reply batch arrives, so it cannot interrupt
  halfway through a long message series;
- it never invents names.

Don't design a persona around any of these.

## 3. Write checks

Prefer structural checks, by id. They are exact and cost nothing.

- `node` with `node_id` and `expect`:
  - `reached`: at any point;
  - `ended_on`: the last node;
  - `not_reached`.

  A final node usually mutes the dialogue, so the conversation ends `bot_muted` there.
  That is expected.
- `playbook` with `playbook_id`, `opened` or `not_opened`.
- `tool` with `tool` and `called` / `not_called`, optionally with `args_contain`. An
  intercepted call counts: the model made it.
- `shape`: messages per turn, characters per message.

Then **1–3 `judge` criteria** for what only reading shows. Each one:

- **A single checkable statement about the agent**, taken from a prompt rule. Quote the
  rule's own words.
- **Exact about its exceptions.** "One question per message" fails on a two-question
  greeting, and passes or fails at random unless the criterion says what to do with
  greetings and numbered blocks. Same for blocks that legitimately shrink: a pregnant
  client's final block has 2 items, not 4. Vague criteria give inconsistent verdicts:
  six identical greetings were judged 4 fail / 2 pass.
- **Decidable from the transcript and the tool CALLS.** The judge never sees what a tool
  returned. "Only products from the tool's result" cannot be judged.
- **Likely to come up.** One criterion the conversation never reaches (e.g. "after the
  handoff, does not keep consulting" when the dialogue mutes at the handoff) stays
  undecided, and so does the whole persona.

`severity` (critical / major / minor) is a label for the reader only: ANY failed check
fails the conversation. Don't add nits you would ignore.

Write what SHOULD happen per the prompt, not what prod did. A sim exists to catch the
agent doing the wrong thing.

## 4. Personas from real dialogues

Use these when the user wants real cases ("take a few real ones", "replay what went
wrong").

1. Find candidates by outcome with `find_dialogues`:
   - `tree_node_ids` for the final nodes, or a stage or refusal reason (ids from
     `dialogue_filter_options`);
   - narrow by agent and dates.

   Spread them across outcomes. Wrong escalations, odd closes and long dialogues that went
   sideways teach the most.
2. Read them before drafting: `export_dialogue` with `sections=["events"]` has the
   transcript. A draft costs an LLM call.
3. `draft_sim_persona(dialogue_id, note=…)`, with the go-ahead. The `note` says what the
   agent SHOULD have done; it becomes judge criteria.
4. **Edit the draft. Never save it as is.**
   - Its `node` / `playbook` / `tool` checks record what happened. When the note says the
     outcome was wrong, fix them: e.g. `ended_on 631` → `not_reached 631`, plus the right
     tool check.
   - Rewrite the persona text as present-tense behaviour if it retells the past.
   - The drafter already leaves out channel history imported before the agent's
     conversation (menu bots, `/start`) and managers' messages. Still check that
     `style_examples` are the customer's own lines.
   - Nothing personal stays. Phones and emails are masked as `[phone]` / `[email]`; also
     remove names and anything identifying. No CRM fields are seeded.
5. Title it with the source: «… (реальный диалог 196959)».

## 5. Save and run

- `create_sim_personas` (batch), `update_sim_persona`, archive with
  `{"status": "archived"}`: show the batch or the change first and get a yes.
  `update_sim_persona` replaces each field it is given whole: to change one check, send
  the full `checks` list. Results already played keep the persona as it was.
- **`run_sim_personas`**: name the personas, variants, repeats and real tools to the user
  and get a yes. It spends the company's tokens: about $0.25–0.30 per conversation on a
  long-prompt Flash agent (agent ~80 %, simulated customer ~15 %, judge ~5 %). A
  15-persona × 2-repeat run is about $8 and 15 minutes; currently five conversations play
  at a time.
- **Repeats.** One conversation is a sample: the agent and the customer both vary.
  - Baselines: `repeats=2`, or 3 for anything you will act on.
  - A persona whose repeats disagree is `unstable`. Its verdict is noise until you know
    why.
- **Real tools.** List a tool in `real_tools` only if its backend has no real-world
  effect: a read-only recommender yes; booking a real calendar slot or creating a CRM
  task, never without the user explicitly asking. Calls to a real async tool can take
  minutes; the run waits for them.
- **Variants** compare configurations in one run. Each variant is an A/B bucket config.
  - A draft prompt: save it disabled first (the `rewrite-prompt` flow), then use variants
    `[{"label": "live"}, {"label": "draft", "config": {"prompt_overrides": {"prompt":
    {"ru": [<prompt_id>, <version_id>]}}}}]`.
  - Playbooks: `playbook_overrides` `{<playbook_prompt_id>: {"version_id": …}}`.
  - Other configs: `models_config_override`, `outreach_config_override`.
  - Tree nodes, tools and the knowledge base are not versioned: compare those with a
    run before and a run after the saved change.
- Poll `get_sim_run` every few minutes, not every few seconds. If the user's MCP client
  rejects a parameter the docs mention, its tool list is stale: `/mcp` reconnects it.

## 6. Read the results

1. `get_sim_run(run_id)` → `summary`:
   - per variant: counts, end reasons, the checks failing most, `unstable_personas`;
   - per persona: each variant's `verdict` (passed / failed / mixed) over its repeats,
     `flipped` (the variants disagree), `unstable` (repeats disagree).

   A flip on an unstable persona is not a finding.
2. `list_sim_results(run_id=…)` gives each check's evidence, the judge's quote and
   reasoning, the customer's note and the `dialogue_id`. Responses are big: filter by
   `persona_id` / `status` when you can.
3. **Triage every fail before calling it a bug.** Open the conversation
   (`export_dialogue` with the result's `dialogue_id`) and decide which it is:
   - **Agent behaviour** against a prompt rule.
   - **Prompt or config contradiction.** Two rules disagree, or the tree's condition
     disagrees with a playbook. Seen in practice: an Escalated node firing on "the doctor
     prescribed it" while the playbook prescribes a gentle scheme. The outcome then
     depends on the customer's wording.
   - **Platform behaviour.** The input filter muting a prompt-injection opener; a
     tool's own result.
   - **Test mistake.** A criterion that misreads the prompt, the simulator skipping an
     item, a judge miss on an ambiguous criterion. Fix the persona, then rerun.
4. End reasons:
   - `bot_muted`: a final node, the input filter, the bot-turn limit.
   - `customer_done` / `customer_left`.
   - `no_reply`: the agent stayed silent with nothing in progress.
   - `waiting_tool`: a real tool was still open at the cap.
   - `handed_off`: ask-manager.
   - `max_turns`: raise it.
   - `error`.

   Several `no_reply` or `waiting_tool` endings in one run: look at the tool and model
   latency before blaming the prompt.

## 7. Report

Write it for the CSM or client, not for engineers:

- what was tested and how many times;
- each finding with its frequency (k of n), the evidence (dialogue ids, marked
  simulated / real) and the prompt rule it breaks;
- **whose decision it is**: the client's (which rule wins), a prompt fix (offer
  `rewrite-prompt`, then rerun the same personas with a variant to confirm it), or a
  platform FYI;
- what works: covered behaviours that held every time.

Keep test mistakes and platform behaviour out of the agent's findings. Name the ones you
corrected and the ones that remain.

## Never

- Never create, edit or archive personas, draft, or run without the user's yes on the
  exact batch, run or change.
- Never let a tool with real-world effects run for real on your own initiative.
- Never present one conversation's fail, or a flip on an unstable persona, as a
  regression.
- Never put real personal data into a persona.
- Never fix a criterion so that the agent passes. Fix it only when it misreads the
  prompt, and say so.
