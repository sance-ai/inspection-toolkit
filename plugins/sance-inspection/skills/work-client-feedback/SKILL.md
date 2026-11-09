---
name: work-client-feedback
description: "Use when asked to work through client feedback on Sance dialogues — the comments and ratings a company's people leave in the markup interface (frontend-client) and its kanban board: review the open remarks, find what caused each one, fix it, reply to the client, move the card, resolve the remark. Every reply, move or resolve is visible to the client and needs the user's go-ahead."
---

# Work through client feedback

Goal: take the client's open remarks about dialogues, find the cause of each with evidence, get
it fixed, and close the loop with the client — without ever writing to the client on your own.

The markup interface is where a company's own people rate dialogues (1..5 per rating option) and
comment on them: on the whole dialogue, on one message, on a fragment of a message (with its
quote), on a rating; with reply threads. Every top-level comment gets a card on the company's
kanban board (columns `new` → `grooming` → `in_progress` → `done`). Read `topic_doc("feedback")`
once per session before the first write.

## The method

1. **The queue**: `list_feedback(product_id)` — open comments of the product, in board order, then
   the comments that have no card (`card_id` null: deleted on the board, or older than automatic
   cards). Narrow with `agent_id`, `columns`, `since`; page with `page` / `next_page` until
   `has_more` is false when the user asks for "all". Each item: the card (column, notes), the
   comment (scope, text, quote / message_text, rating, author with `side` client / team) and its
   replies. If the product id is unknown, use `locate_entity` with a known dialogue / agent / lead id;
   otherwise ask for the product id or dashboard URL — never iterate products.
2. **Group before digging.** Several remarks often share one cause (the same prompt rule, the same
   tree node). Group them by what they complain about and by agent; present the groups to the user
   and agree which to take first.
3. **One remark, one dialogue**: `debug_dialogue(product_id, dialogue_id,
   sections=["events", "feedback"])` (or `export_dialogue` + `jq` for big dialogues). The
   `feedback` section ties each comment to its `message_id` in the transcript and lists every
   author's ratings. Then follow the `debug-dialogue` method for the turn the comment is about:
   what the model saw, which prompt version, which tree node, what was retrieved.
4. **Name the cause with evidence** (run id, prompt version, node, rule quoted) — or say plainly
   that the data cannot show it (see the bundle legend's `not_persisted`). A remark can also be
   a client preference, not a bug: say so instead of inventing a fix.
5. **Fix through the usual workflows**, each with its own approvals: `rewrite-prompt` (a new prompt
   version), `decompose-into-playbooks` / `create_prompt` (a playbook), dashboard changes for
   anything the MCP cannot write (tree, pings, settings) — describe them for the user. Where
   useful, check the fix with `test-with-synthetic-customers` (the client's dialogue makes a good
   persona: `draft_sim_persona`).
6. **Close the loop with the client** — only after the user approves each step:
   - **Reply**: draft the reply in chat (short, in the client's language, what was wrong and what
     changed — no internal ids, model names or prompt text), get an explicit "yes, send it", then
     `reply_feedback_comment(product_id, comment_id, text)` with exactly that text.
   - **Take into work / park**: `move_feedback_card(product_id, card_id, column, notes?)` —
     usually `in_progress` while a fix is pending; notes are visible on the board.
   - **Resolve**: `set_feedback_comment_state(product_id, comment_id, "resolved")` once the fix is
     live (or the user decides it needs none). It moves the card to `done` itself.

## Hard rules

- **The client sees every write.** A reply appears in the client's interface under the name of
  the CSM signed in to the MCP; moves and resolves change the client's board. Never call a write
  tool without the user's explicit go-ahead on the exact reply text or change. The tool's own
  permission prompt is not that go-ahead — some clients skip it.
- **State, not column.** Whether a remark is handled is the COMMENT's `state` (`open` /
  `resolved`), never the card's column: a card dragged to `done` by hand can still have an open
  comment. `list_feedback` filters by state (`open` by default); `columns` is only a view.
- **Repeats are safe, rewordings are not.** The same reply text under the same comment within
  10 minutes returns the existing reply (`created: false`); a reworded text is a NEW reply. A
  repeated move or state change returns `changed: false` and does nothing.
- **"WAS SAVED" errors**: a write tool that says the change WAS SAVED but its result could not
  be shown (masking down) — the write happened. Do not repeat it; check later with
  `list_feedback` / `debug_dialogue`.
- **Pseudonyms**: a `pii_note` in a result means the company masks personal data — names and
  contacts in comments, quotes and notes are consistent stand-ins, never the real person. Never
  copy a stand-in into a reply to the client as if it were real; refer to people by role ("the
  customer", "the manager") in replies.
- The queue includes remarks from both clients and our team: keep both, and use `author.side`
  to distinguish them. Our own replies have `author.side = "team"`; do not mistake them for
  a new client complaint.

## Report format

Per remark (or group): what the client said (quote), where (dialogue id, message id), the cause
with evidence, the fix (done / proposed / not needed, with ids), and the client-facing step taken
(reply sent / card moved / resolved) or still waiting for the user's approval. End with what is
left open in the queue.
