---
name: update-task-with-findings
description: Use when a session produced durable output — an approved spec, a design decision, a measurement, a research finding — and an existing task-board card must carry it — "update the task with what we decided", "attach the spec", "record this on the card", or after research / specify / implement finishes. Not for brand-new ideas (use add-idea-to-backlog) or for cards you are creating.
---

# Update a task with new information

## Overview

Every board write here is a **full replace**, not an append, and reading the card does
not tell you what you are about to overwrite. So the rule is: **read the specific thing
you are about to replace, then prove the write landed.**

**Violating the letter of this rule is violating the spirit.** "I called `get_task` so I
read before writing" does not detect an artifact clobber — see step 2.

## Why this needs a skill

Two facts that read opposite to intuition:

- `write_artifact` upserts with `ON CONFLICT(task_id, key) DO UPDATE SET markdown =
  excluded.markdown` — it **replaces the whole artifact** and exits happily.
- `get_task` returns `artifact_keys` — **names only, never content**. So the card can
  tell you a key exists while leaving you blind to what is in it.

And a durable-copy hazard: a plan is a card artifact, not a repo file. Plans used to live in `docs/roadmap/`,
which is gitignored in repos with a board, so the file had no commit, no stash, and no dangling blob. The card's
`plan` artifact is now the only durable copy, and a plan overwritten here is gone. The scratch copy under
`~/.cache/agent-plans/` is a working copy only.

## The sequence

1. **`list_board(space_path)`** for the current Space → get the real `task_id`. Never
   invent or reuse an id from memory.
2. **`get_task(task_id)`** → current title, body, `spec_path`, artifact key names.
3. **`get_artifact(task_id, key)` for the key you intend to write** — this is the only
   call that detects a clobber. If it has content, merge deliberately or pick a distinct
   key; never replace approved text you have not read.
4. **`write_artifact`** with the content, or for a repo document (research, ADR, closeout) with **only its
   repo path** as the body. Repo documents are location attachments: the file on disk is the record, and
   copying it into the card bloats the database with a second copy.
5. **Prove it**: `get_artifact` again. For content, compare against the source; a silent drop during a
   paraphrase is the failure this step exists to catch. For a location, confirm the returned path exists.
6. **`edit_task`** for `body` / `spec_path` last.

## `edit_task` is per-field replace

`title` / `body` / `spec_path` are patched one field at a time: **omit a field to preserve it, pass
it to replace it entirely.** Body is never appended. So a body rewrite must carry forward
anything in the old body that still holds — re-read it in step 2, don't recall it.

Three states, not two, for a **desktop** caller:

| You pass | What happens |
|---|---|
| the key **omitted** | the column is kept exactly as it is |
| an explicit **`null`** | the column is cleared to NULL |
| a **string** | the column is written with it |

So `spec_path` can be **cleared**: `spec_path: null` writes NULL, and the card stops naming a file that was
deleted under it. Omitting the field is a different act and preserves the old value — the two are not
interchangeable, and "normalizing" an omitted argument to `null` would turn every keep-it into a clear-it.

`""` still writes an empty string, not NULL — and it is now worse than useless, because the render sites treat
an empty pointer as no pointer at all. So `""` leaves a row holding a value that every surface displays as
absent. **`""` is never an answer to "clear this"; `null` is.**

**This is desktop-only.** A skill whose `edit_task` reaches the board through the harness **tool** is still on
the published crate's two-state shape: there `null` and absent are one value and both mean "leave it alone", so
a `null` clears nothing while the call still reports success. On that path, prove the clear with `get_task`
before believing it, and when it did not land say the stale pointer remains rather than quietly passing `""`.
The third state reaches the tool once the verb moves desktop-side.

## `spec_path` must be a path that survives

Pointing `spec_path` at a gitignored or scratch file is worse than leaving it null: it
reads like a pointer to a durable record and resolves to nothing on ship. Set it only to
a tracked path. A location attachment must point at a tracked file too. Content lives on the card only for
artifacts with no repo file (plans, status, follow-ups). For a gitignored or scratch source, the **artifact holds
the content** and `spec_path` stays null.

## Artifact keys: self-enforced

Grammar `[a-z0-9][a-z0-9-]{0,39}`, ≤ 256 KiB. `require_artifact_key` sits in the command
layer and **rejects rather than cleans** — but the model's path goes through the router
straight to `write_artifact` and **never touches it**, so nothing validates your key for
you. A key is interpolated into the `<task>` launch block, so a bad key is load-bearing,
not cosmetic. Conventional names: `spec`, `research`, `notes`, `design`.

## Do not move the card unless asked

Settling a design is not a stage transition you get to invent. And know the real
semantics before refusing or performing one for the wrong reason: a **model** `move_task`
fires **no** `on_enter` hook and no compaction. A human **drag** does the compaction boost
and nothing else — its live half is that boost alone. The full verb (move, then start a
Session on the card) lives behind the card's **arrow pill**, its detail-dialog buttons, and
the chain runtime that acts on a `mark_done` advance. So a `move_task` is indeed inert, but
"moving it triggers the launch skill" is false for a drag as well as for a model, and the
pill is the door — not the drag.

Also: `move_task` is **not** the door for a forward move. A model's forward `move_task` is
refused — on a default, never-configured board too, because the rule is graph reachability
with a `sort_order + 1` fallback — and the refusal names `mark_done`, the only forward door
(see `move-task`). "Don't move the card unless asked" therefore still stands, and it stands
for `mark_done` as well: if the user did not ask for the hand-off, write the artifact, say
where it is, and stop.

## Subagents cannot touch the board

Task tools **park** in a worker — the daemon holds no `TaskStore` and answers
`TaskRequest` with a `task-unsupported` refusal, so the call hangs until the turn is
cancelled. Emit the exact calls as your deliverable; the parent executes them.

## Common mistakes

| Mistake | Reality |
|---|---|
| Writing `key: "spec"` because `get_task` showed no `spec` | Names without content still hide nothing here — but if the key DOES exist you just replaced it blind. `get_artifact` first. |
| "I stored it verbatim" after paraphrasing | Paraphrase silently drops sections — caveats and load-bearing guard clauses first. Re-read and diff. |
| Setting `spec_path` to the scratch/gitignored file | A pointer to a deleted file looks like a record. The artifact is the record. |
| Replacing body and losing prior content | Body is a full replace. Carry forward what still holds. |
| Moving the card to `approved`/`done` unprompted | Not your transition to make. |
| Believing a move will run the launch skill | A model move fires no `on_enter`. |
| Trusting your artifact key to be validated | The model path bypasses `require_artifact_key`. |
