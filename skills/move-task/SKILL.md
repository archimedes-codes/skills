---
name: move-task
description: Use when moving a Task board card to another stage or column — "move it to approved", "hand it off", "move to review", "mark it done", "move the card to X", "we're done here then what" — or at the end of research, specify, or implement when the session came from a `<task id="tsk_…">` block. Also triggered by the terminal ask of those skills.
---

# Move a task card

## Overview

A Task card is the handoff medium: the card carries the artifacts, and whatever session or
person picks it up next reads them from the card. Moving a card is a transfer of context, not
just a status change. So the move comes **last**, and only after the card holds every durable
document this session produced.

**Decide the direction first, because the two directions are two different verbs.** A **forward**
move — the column the board's arrows lead to — goes through **`mark_done`**, which flags this
Session as finished and advances the card. `move_task` moves a card **backward or sideways** only;
a forward `move_task` is refused outright. The artifact checkpoint below gates both doors: attach
everything first, then take the door that matches the direction. A person's drag is never gated by the
forward rule (a `board_strict` graph still binds it).

Run this **only when this session knows a task id**, from a `<task id="tsk_…">` block or a
plan's `task:` front-matter key. No id means no card: skip entirely, or ask the user which
card to use. Do not create a card just to have something to move.

## Order is load-bearing

1. **Inventory the durable documents this session produced.** Use the transcript, then cross-check
   the scratch directory: `ls ~/.cache/agent-plans/<feature>/`. Plans and run records are not in the repo,
   so `git status` will not show them. Scratch is the only place a plan exists if the card copy was never
   attached, so it is the first thing to check.
2. **Map each document to its key** (table below). Read the front-matter `task:` key and confirm it
   matches the card id.
3. **Attach each one.** `get_artifact` first, because the upsert replaces the whole artifact and
   `get_task` shows key names only. If the existing content is from an earlier revision of this
   same document, overwrite. If it is a different document, stop and pick a distinct key. Then
   `write_artifact`, then `get_artifact` again to prove it landed (see *Verifying a write*).
4. **Checkpoint, in your reply, before the move (either door):** answer the question *"Did I attach every
   durable document this session produced?"* from evidence, using the template below. Any "no" means
   you do not move. Go back to step 3.
5. **`list_board(space_path)`** and take the destination's **real slug** from the rows. Never assume
   a slug; a wrong one errors naming the valid set. You need the slug for `move_task`, and the row's
   `label` to name the column to the user; a forward `mark_done` needs neither (the graph picks the
   destination), but read the row anyway so you can report where the card landed.
6. **Take the door that matches the direction.**
   - **Forward** (the column the board's arrows lead to, e.g. Research → Approved): **`mark_done`**.
     Its one optional argument `outcome` names which transition to follow when the column has more
     than one wired out of it — one of `done`, `failed`, `error`, `success`, `blocked`, `partial`,
     `finished`, `complete`, `handoff`. Leave it out to take the first. Whether the card actually
     moves is the destination column's `auto` flag, not your choice: `auto` ON advances it and
     launches that column's Skill; `auto` OFF leaves the card where it is and flags it for a person.
     Read the answer — it names the column the card advanced to, or says plainly why it stayed.
     `mark_done` claims **this Session** and takes no card id: a Session not linked to a card marks
     itself done and moves nothing on the board, so this door only carries a card when the session
     came from one.
   - **Backward or sideways:** `move_task(space_path, task_id, <slug>)`. It is inert — it starts
     nothing (see *Facts*).
7. **Verify**: `get_task` for a `move_task`, or the `mark_done` answer plus `get_task` for a forward
   claim — `stage_slug` changed, and `artifact_keys` still contains every key from the checkpoint.
8. **Tell the user, then stop.** Do not load the destination's launch Skill, and do not keep working on
   the card's content.

## Which document gets which key

| Document this session produced | Key | Notes |
|---|---|---|
| `<scratch>/plan.md` (from `specify`) | `plan` | Spec and plan live in this one file. Attach the whole file. Do not add a separate `spec` key for it. |
| `<scratch>/plan-<id>.md` (a COARSE task's plan) | `plan-<id>` | One per task. Do not put it in `plan`. |
| `<scratch>/parent.md` (a COARSE run's parent doc) | `parent` | Its own key. |
| `<scratch>/status.md` (from `implement`) | `status` | Its own key. Do not put it in `plan`. |
| `<scratch>/follow-ups.md` (from `implement`) | `follow-ups` | Its own key. |
| `docs/research/<name>.md` (from `research`) | `research` | **Location attachment:** the body is the repo path, not the report. Leave an existing one alone unless the path changed. |
| A spec written on its own, with no plan | `spec` | Only when no `plan` exists for the same topic. |
| `docs/decisions/NNNN-slug.md` (ADR) | `adr-NNNN` | **Location attachment:** the body is the repo path. One key per ADR written this session. |
| `docs/features/<topic>.md` closeout | `<topic>` | **Location attachment:** the body is the repo path. Keys are `[a-z0-9][a-z0-9-]{0,39}`: lowercase, digits, hyphens, no path, no dot. |

Keys are `[a-z0-9][a-z0-9-]{0,39}`. Markdown is capped at 256 KiB, and a refused write stores nothing.

## Checkpoint template

Fill this in from evidence, not memory, and put it in your reply before the move (whichever door you
take):

```
Durable documents this session produced:
- <scratch>/plan.md               → key: plan      attached: yes  verified: yes
- docs/research/<name>.md        → key: research  attached: yes  verified: yes  (path only)
- (none beyond the above)
Card artifact_keys now include: plan, research
Missing: none
```

If the session produced **no** durable documents (a talk-only session, or a card with no written output),
write `Durable documents this session produced: none` and say so. The gate passes with nothing to attach.

## Verifying a write

**Content attachments** (plan, plan-<id>, parent, status, follow-ups): `get_artifact` returns the markdown. Compare it with the source file:

- every heading in the source (`grep '^#'`) appears in the returned content;
- the returned content ends with the same last lines as the source;
- it is not shorter than the source.

A silent truncation or paraphrase fails this check. Re-attach from the source file.

**Location attachments** (research, adr-NNNN, <topic>): `get_artifact` must return exactly the repo path, and that path must exist and be tracked: `git ls-files --error-unmatch <path>` succeeds. Do not compare contents; the file is the record.

## Reporting depends on the destination

### Approved (or any parking lane that starts work later)

Use this wording when the card reached Approved **without** a `mark_done` advance — a person moved it,
or a `mark_done` answer said the column is not automatic and a person still has to move it. Say all
three parts, in this order, and name the controls the way the UI names them:

> The card is now in **<real Approved label>**, and its artifacts are: `plan`, `research` (verified). To implement it:
> 1. **Archive this session**: hover its row in the sidebar and click **Archive** (it is
>    a hover-revealed action). This session's context is done; the card now holds the
>    artifacts.
> 2. **Open the Tasks board** for this Space.
> 3. **Click the arrow pill on the card**. It is labelled with the next column's name, which
>    you read from `list_board` in step 5 (quote the real label; never guess it). That click
>    moves the card and starts a **fresh Session** on it, which loads that column's launch
>    Skill and reads the card's artifacts.

**If the forward move went through `mark_done` instead** (the usual case for a card-borne session),
report what the answer actually said rather than describing a move you did not make:

- It named a destination column → say the card advanced there and that the column's launch Skill was
  started (or that a Session is already running on the card, if the answer said so). Do not tell the
  user to click the pill for a move that already happened.
- It said the column is not automatic → the card stayed put and the card carries an **amber bell**;
  a person moves it, or clicks the pill to move *and* launch. Say that, and say the card is flagged for
  review.
- It said the column leads nowhere, or the chain is at its step bound or cost ceiling → nothing moved.
  Say so in the same words the answer used.

### Any other stage

Say which column the card is now in (quote the real label), list its artifact keys, and say that the
move itself starts nothing. If the user wants the work to start, they click the card's arrow pill on
the board. Then stop.

### Done

Only move to a completion stage when the user asked for it or the work is actually complete. A card
in an approval or review lane is not finished.

## Facts that read opposite to intuition

- **`move_task` takes only a slug, so nothing in the call looks at artifacts.** Assume it will not stop
  a card with missing documents. The checkpoint is this skill's job, not the tool's.
- **A model `move_task` fires no `on_enter` hook and no launch — and neither does a drag.** The full
  verb (move, then start a Session on the card, then send it its prompt) lives in `launchTaskSession`,
  which has exactly three callers: the card's **arrow pill**, the buttons in the card's detail dialog,
  and the chain runtime that acts on a `mark_done` advance. A board **drag** goes through a different
  path whose live half is the compaction boost, so it launches nothing either. So the move is safe but
  inert — "safe but inert because only a human drag runs the full verb" is wrong twice over: a drag is
  inert too, and the **pill** is the real door. Nothing starts until a person clicks the pill or
  `mark_done` advances an `auto` column.
- **`mark_done` is the only forward door, and the column's `auto` flag decides what the claim does.**
  `auto` ON: the card advances to the column the edge names and that column's `on_enter_skill` launches.
  `auto` OFF: the card does **not** move; an amber bell tells a person it is ready to be moved. A
  Session that settles **without ever calling `mark_done`** moves nothing, closes the chain run with a
  stated reason, and says so in the transcript. Claiming done is therefore always safe; it is never a
  way to force a card somewhere.
- **A stage's launch behaviour is data, not a word you can guess.** Which Skill a new Session loads
  comes from the destination Stage's `on_enter_skill` (NULL on a parking lane such as `approved` by
  default). It varies per Space and changes without warning. Read the stage rows; never name a Skill
  you have not read there.
- **The pill's label is the DESTINATION column's name, read from `list_board`.** A Space owner can
  rename columns, so `Doing →`, `Implement →` and `WIP →` are each correct on some board. Renaming a
  column does **not** change which Skill it launches: `slug` and `label` are separate columns.
- **There are now TWO refusals on `move_task`, and they ask different questions.** Under
  `board_strict = 1`, a move that is not a **declared edge** refuses, writing nothing — relay the
  refusal's allowed list and do not retry with a guessed slug. And **unconditionally**, a forward move
  by a model refuses with a sentence that is the whole lesson — it arrives as an "invalid arguments"
  result, so you will not be tempted to retry it:

  > cannot move {task_id} forward from {from} to {to}: forward moves go through `mark_done`, which
  > flags the Session and advances the card. Call `mark_done` when the work is finished and a person
  > should look at it; move backward or sideways with this verb.

  The second refusal is a
  **graph-reachability** rule: it walks the column graph, and a column with no drawn edges still has a
  next step at `sort_order + 1`. **So it fires on a default, never-configured board** — the plain
  backlog → research → approved → doing → done shape is a chain to this rule. Nothing about it says
  "only if you configured a workflow". Backward and sideways moves stay unaffected on every board.

## Do not

- **Do not move past a missing artifact.** If you cannot attach a document (no `write_artifact` tool,
  a refused write, or no card id), stop and tell the user which document would be lost. Do not move
  the card so it looks finished.
- Do not load a destination's launch Skill after the move. The board starts it when the user clicks,
  in a fresh Session.
- **Do not call `move_task` for a forward move.** It is refused, and the refusal is not a bug to work
  around: `mark_done` is the forward door. Do not try to dodge it by moving backward first, by
  re-reading the slug, or by retrying — the rule is graph reachability, not a spelling check.
- Do not move a card with no task id, and do not create one to have something to move.
- Do not move to a completion stage on your own initiative.
- Do not assume a slug. Read it from `list_board`.
