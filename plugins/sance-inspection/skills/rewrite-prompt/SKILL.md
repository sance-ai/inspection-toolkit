---
name: rewrite-prompt
description: Use when asked to rewrite, edit, improve, fix or update a Sance agent's prompt (main prompt, ping prompt, tree condition prompt, playbook, any prompt type) and save it to the platform. The file-based edit loop around export_prompt / prepare_prompt_upload / create_prompt_version; every save requires the user's explicit approval of the exact text.
---

# Rewrite a prompt and save it as a new version

Goal: produce a better version of an existing prompt and save it — safely, with the
user in control of every byte that goes live, and without pushing 80k-character bodies
through the conversation.

## The file loop (default)

Prompts travel as FILES, not tool results. Work in `.sance/` in the current directory;
make sure `.sance/` is listed in `.gitignore` (add it if not).

1. **Locate**: `locate_entity(agent_id=…)` if only an agent id is known; then
   `inspect_agent(…, area="prompts")` for the inventory: which prompt is live for the
   language/modality, what references it (tree conditions, pings, campaigns), whether
   a newer prompt shadows it.
2. **Read the rules** once per session: `topic_doc("prompt-authoring")` — core-prompt
   anatomy, the cache-breakpoint rule, the capability-alignment table, the pre-save
   checklist; for a playbook also `topic_doc("playbooks")`. The "instructions in
   English, client-facing text in Russian" split described there is a suggestion with
   too little data behind it: keep the prompt's current language unless the user asks.
3. **Download**: `export_prompt(product_id, prompt_id)` returns a one-shot link, the
   sha256 and an outline (headings with character offsets). Then:
   `curl -sf -o .sance/<suggested_path> "<url>"`. The link works once and expires in
   minutes — if curl fails, call export_prompt again. Read the file by section using
   the outline; don't load a huge prompt whole unless you need all of it.
4. **Understand the constraints**: what the agent can actually do (`agent_config`
   area: tools, calendar, files, clarification), node names the prompt refers to
   (`dialogue_tree` area), and the client's requirements if the user gave them. A
   prompt must never instruct the agent to do something the config disables.
5. **Edit the file in place** with your edit tool — targeted edits, not a rewrite from
   memory. Preserve: Jinja variables that exist (below the marker only), the
   `---cache breakpoint---` marker, the output language, the client's tone, every
   client-facing Russian string verbatim. No "consult file X" / references to other
   prompts — the agent cannot open them. A monolith (scripts for every situation,
   >40k chars) → propose `/decompose-into-playbooks` instead of polishing it.
6. **Review with the user**: point them at the file path, summarize what changed and
   why (a `diff` against a pristine copy is ideal — keep one, e.g.
   `.sance/<name>.orig.md`). Ask explicitly: create as a disabled version, or create
   and enable?
7. **Only after an explicit yes — upload and save**:
   - `prepare_prompt_upload(product_id, prompt_id=…)` → an `upload_id` and a PUT link;
   - `curl -sf -T .sance/<file> "<url>"` → the response carries the `sha256`;
   - `create_prompt_version(product_id, prompt_id, upload_id=…, content_sha256=…,
     description="<what changed>", enable=<as the user said>)`. Claude Code asks the
     user to permit this call — expected. The hash guarantees what is saved is exactly
     the file the user reviewed; if the file changed after review, re-confirm first.
8. **Handle findings**: if the tool blocks on content checks, show them verbatim. Fix
   critical ones in the file and repeat step 7. For warnings/severe, ask whether to
   proceed; pass the codes in `acknowledged_checks` only with consent.
9. **Report**: version number, enabled or not, where to review it in the dashboard
   (the prompt's version history). If disabled, remind that nothing changed live.

## Small prompts and shell-less clients

For a short prompt (a ping prompt, a tree condition — a few hundred characters) the
inline path is fine: `prompt_text` to read, show the full new text in chat, then
`create_prompt_version(…, content=…)`. Same approval rules.

## Never

- Never save without the user having reviewed the full new text and said yes.
- Never set `enable=true` on your own initiative.
- Never rewrite a prompt you have not read — via the downloaded file or `prompt_text`.
- Never touch prompts the user did not name — "rewrite the prompts" means list
  candidates first and confirm which ones.
