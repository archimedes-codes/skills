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
4. **Checkpoint, in your reply, before any `move_task`:** answer the question *"Did I attach every
   durable document this session produced?"* from evidence, using the template below. Any "no" means
   you do not move. Go back to step 3.
5. **`list_board(space_path)`** and take the destination's **real slug** from the rows. Never assume
   a slug; a wrong one errors naming the valid set.
6. **`move_task(space_path, task_id, <slug>)`**.
7. **Verify** with `get_task`: `stage_slug` changed, and `artifact_keys` still contains every key from
   the checkpoint.
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

Fill this in from evidence, not memory, and put it in your reply before `move_task`:

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

Say all three parts, in this order, and name the controls the way the UI names them:

> The card is now in **<real Approved label>**, and its artifacts are: `plan`, `research` (verified). To implement it:
> 1. **Archive this session**: hover its row in the sidebar and click **Archive** (it is
>    a hover-revealed action). This session's context is done; the card now holds the
>    artifacts.
> 2. **Open the Tasks board** for this Space.
> 3. **Click the arrow pill on the card**. It is labelled with the next column's name, which
>    you read from `list_board` in step 5 (quote the real label; never guess it). That click
>    moves the card and starts a **fresh Session** on it, which loads that column's launch
>    Skill and reads the card's artifacts.

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
- **A model `move_task` fires no `on_enter` hook and no launch.** Only a human drag or pill click runs
  the full verb. The move is safe but inert; nothing starts until the user acts.
- **A stage's launch behaviour is data, not a word you can guess.** Which Skill a new Session loads
  comes from the destination Stage's `on_enter_skill` (NULL on a parking lane such as `approved` by
  default). It varies per Space and changes without warning. Read the stage rows; never name a Skill
  you have not read there.
- **The pill's label is the DESTINATION column's name, read from `list_board`.** A Space owner can
  rename columns, so `Doing →`, `Implement →` and `WIP →` are each correct on some board. Renaming a
  column does **not** change which Skill it launches: `slug` and `label` are separate columns.
- **Under `board_strict = 1`, a move that is not a declared edge refuses**, writing nothing. Relay
  the refusal's allowed list; do not retry with a guessed slug.

## Do not

- **Do not move past a missing artifact.** If you cannot attach a document (no `write_artifact` tool,
  a refused write, or no card id), stop and tell the user which document would be lost. Do not move
  the card so it looks finished.
- Do not load a destination's launch Skill after the move. The board starts it when the user clicks,
  in a fresh Session.
- Do not move a card with no task id, and do not create one to have something to move.
- Do not move to a completion stage on your own initiative.
- Do not assume a slug. Read it from `list_board`.
