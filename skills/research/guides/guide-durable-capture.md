# Durable Capture — where documents live, the Task card, tracker

Shared by `research`, `specify` and `implement`. Everything in this file is a **write that outlives the
session**, and each one has a failure mode where it silently does not happen. Read the matching section
before doing the write, not after discovering it missing.

## Where each document lives

The **Task card** is the durable home for everything a later session needs in order to continue the work.
The **repo** holds the documents that belong in version control for their own sake: research reports, ADRs,
feature closeouts, the glossary. The card does not copy them. It stores a **pointer** to them.

**Two kinds of attachment, and they are not interchangeable:**

- **Content** (plan, plan-<id>, parent, status, follow-ups): the card holds the full markdown, because nothing
  else holds it. These are the only attachments whose bodies are documents.
- **Location** (research, adr-NNNN, <topic>): the card holds one line naming the repo path. The file on disk is
  the record, so its contents are not copied into the database. Bloating the card with a second copy of a file
  the repo already versions is the failure this rule exists to prevent.

| Document | Durable home | Card key |
|---|---|---|
| Plan — spec and plan in one document (`specify`) | **Card only** | `plan` |
| Per-task plan of a COARSE run (`specify`, consumed) | **Card only** | `plan-<id>` |
| Parent doc of a COARSE run (`implement`) | **Card only** | `parent` |
| Status file (`implement`) | **Card only** | `status` |
| Grounding report per area (`implement`) | **Scratch only** — re-derivable from the tree | — |
| Follow-ups (`implement`) | **Card only**, folded into a repo doc before ship | `follow-ups` |
| Research report (`research`) | **Repo** `docs/research/<name>.md`. Card holds its **path** | `research` |
| ADR (`research`) | **Repo** `docs/decisions/NNNN-slug.md`. Card holds its **path** | `adr-NNNN` |
| Feature closeout | **Repo** `docs/features/<topic>.md`. Card holds its **path** | `<topic>` |
| Glossary terms | **Repo** `CONTEXT.md` | — |

**Plans never go in the repo.** Not `docs/roadmap/`, not any other path in a repository, gitignored or
not. A plan in the repo is a file `git status` does not show and that a later cleanup may delete.

Card keys follow `[a-z0-9][a-z0-9-]{0,39}`. Lowercase the task id (`A1` → `plan-a1`).

The rule for the split: **the card holds what cannot be rebuilt.** A grounding report can be re-derived by
re-reading the tree, at the cost of re-grounding, so it stays in scratch. A plan, a status file and a follow-ups
file cannot be re-derived, so they go on the card.

## Scratch — the working copy

Skills and wave children read and write files, so every card artifact has a working copy on disk at
**`<scratch>/<key>.md`**, where **`<scratch>` = `~/.cache/agent-plans/<feature>/`** and `<key>` is the card key
(`plan.md`, `plan-a1.md`, `status.md`, `parent.md`, `follow-ups.md`):

- outside every repository, so nothing in it is committed or ignored by accident;
- under `~/.cache`, not `/tmp`, so it survives a reboot in the middle of a multi-session run.

`mkdir -p` it, and pass **absolute** paths to every child. A child in a git worktree cannot see a
repo-relative path to a gitignored file, and gitignored files are not in a worktree at all.

Grounding reports have no card key. They are `<scratch>/ground-<area>.md`, are never attached, and are
re-derived from the tree if lost.

The card is the durable copy; the scratch file is the working copy. Write to scratch, then attach. If a
scratch file is missing, `get_artifact` restores it. **The card copy is the only copy of a plan, status or
follow-ups file, so it must be kept current.**

## Attach in the turn you write it

Writing a file to scratch is not saving it. A document counts when `get_artifact` on the card returns it.
Attach before any question that presumes the document exists, before any card move, and before the session
ends. Re-attach after every revision.

`write_artifact` replaces the whole artifact, so `get_artifact` first, to confirm you are replacing your own
earlier version and not text you have not read.

## No card

A plan with no card has no durable home. It exists in scratch and in this session only. **Never fall back to
the repo.** Say so to the user in one sentence, and ask (each skill has its own question) whether to attach
it to a card. Do not create a card to hold it without the user's say-so.

## Which tracker?

Resolve in this order and **stop at the first answer** — do not infer a tracker from a remote URL when a
stated one exists.

1. **`AGENTS.md` / `CLAUDE.md`** — an explicit tracker statement wins. Check here first:
   - **Jira**: if it specifies Jira (mentions Jira, a project key such as `PROJ`, or Jira issue
     formatting), the tracker is **Jira**.
   - **GitHub**: if it specifies GitHub Issues (or mentions `gh issue`), the tracker is **GitHub**.
2. **The tracker's own CLI or MCP tools**, when present and authenticated:
   - `gh repo view --json url,nameWithOwner` succeeds → **GitHub**.
   - Atlassian MCP tool `atlassian_getAccessibleAtlassianResources` is available → **Jira**.
3. **An existing `issue:` front-matter key** in the card's `plan` artifact (`get_artifact`), or in the
   research report under `docs/research/`:
   - A Jira key/URL (`https://*.atlassian.net/...` or `PROJ-123`) → **Jira**.
   - A GitHub issue URL (`https://github.com/.../issues/...`) → **GitHub**.
4. **Nothing found** → no tracker. Skip the publish ask, mention it once, and carry on.

**Do not derive the URL from `git remote`.** A renamed repo keeps answering on its old name forever —
GitHub redirects so clones and pushes keep working — so the remote is the one signal that survives being
wrong, and publishing to it files against a moved repo. `gh repo view` resolves the redirect; `git remote -v`
does not.

## The Task card id — where it comes from and where it must go

A card session carries a `<task id="tsk_…" stage="…">` block, and **that block is the only place the card's
id appears.** It normally arrives in your first user message (the board's launch button emits it); when
another skill loads this one mid-session it must forward the block, so also accept one pasted into the
context you were handed. Note the id the moment you see it.

**No block anywhere in your input → this is not a card session; skip every Task-board step in this skill.**
If you were clearly asked to work a named card but hold no block, ask for its id rather than guessing —
`write_artifact` refuses a dangling id and there is no fuzzy lookup to fall back on.

The id has to survive the session, because `specify`, `implement` and `finish` are loaded mid-session,
**never see the block**, and read the id from the document's front-matter instead. So every document you
attach carries it, the plan and the research report included:

```yaml
task: <task_id>
```

Write it even when the tracker step was skipped. Without it the chain dead-ends exactly where the documents
are most destructive — a plan overwriting a spec, a status file overwriting a run's memory.

## Attaching a document to the card

**Content attachments** (plan, plan-<id>, parent, status, follow-ups):

```
write_artifact(task_id: "<id from the <task> block>", key: "<key>", markdown: "<full file contents>")
```

**Location attachments** (research, adr-NNNN, <topic>): the body is one line, the repo-relative path, and nothing else. Do not paste the file in.

```
write_artifact(task_id: "<id from the <task> block>", key: "research", markdown: "docs/research/<name>.md\n")
```

The file must exist **and be tracked by git** before you attach its path. Confirm with `git ls-files --error-unmatch <path>`. A path to a missing or untracked file is a false record, the same as a dangling `spec_path`. If the file is untracked, track it on the working branch first, or say it is untracked and do not attach the path as durable.

Keys: `plan`, `plan-<id>`, `parent`, `status`, `follow-ups` (content); `research`, `adr-<number>` (e.g.
`adr-0101`), `<topic>` for a `docs/features/` closeout (location). A refused write stores nothing (256 KiB cap, key
grammar above). The call **UPSERTS**, so re-running after a revision is safe and is the correct response to
a reviewer's fix.

**Read before you overwrite.** `get_task` returns artifact **key names only, never content**, so the card can
tell you a key exists while leaving you blind to what is in it. `get_artifact(task_id, key)` is the only call
that detects a clobber. Never replace approved text you have not read. And `plan` and `research` are two
keys for two documents — do not overwrite one with the other.

Then point the card at what it should name:

- `edit_task(task_id, spec_path: "docs/research/<name>.md")` for a research report — a tracked file. The `research` artifact and `spec_path` then both name the same path, and neither holds the contents.
- A **plan's `spec_path` stays null.** The plan is not in the repo, and a `spec_path` that names a file which
  is not tracked reads like a record. If the card's `spec_path` already names `docs/roadmap/...`, clear it
  with `spec_path: null` and check it with `get_task`.

**Why the card:** plans used to be written to `docs/roadmap/`. In repos with a Task board that directory is
gitignored, so a plan there had no commit, no stash and no copy at all, and a tracked plan was deleted on
ship. The card keeps the plan either way.

**If there is no `write_artifact` tool** (this skill is global — plenty of projects have no Task board), say
so in one sentence and continue. The plan then has no durable copy, and the question you ask next must say
so. Do not emulate an artifact by writing a file into the repo.

## Mirroring a document to the tracker

Only after the document is attached. If no tracker was found, skip — this never gates anything.

Ask first. The file is `<scratch>/plan.md` for a plan and `docs/research/<name>.md` for a report:

```
ask({
  questions: [{
    id: "mirror-issue",
    question: "The <plan|report> is attached to the card and saved at <path>. Create or mirror to a GitHub issue?",
    options: [
      { label: "Create or update issue and attach the artifact (recommended)" },
      { label: "Skip — keep it on the card only" }
    ]
  }]
})
```

**GitHub:**
- Reuse the issue already tied to this topic (an `issue:` key in the front-matter, or an open issue whose
  title names it); otherwise open one titled the topic:
  `gh issue create --title "<topic>" --body "<summary> — artifact attached."`
- Attach the markdown: `gh issue artifact create <issue-number> <path>`
- Write the URL into the front-matter as `issue: <url>`. For a plan, edit the scratch copy and re-attach it to
  the card. For a repo report, amend its commit if it is tracked; do not force-commit a gitignored file.
- If `gh issue artifact` is unavailable or fails, fall back to posting the body as a comment with the
  front-matter stripped (split across consecutive comments if over the 65 KB limit).
- After a later revision: `gh issue artifact edit <issue-number> <artifact-number> -F <path>`, or post a
  comment saying it supersedes the earlier one.

**Jira:**
- Reuse the ticket the `issue:` key names. Otherwise create one with the Jira tooling this environment
  provides (for example an Atlassian MCP create operation, `contentFormat: "markdown"`), routing the
  project from `AGENTS.md` / `CLAUDE.md`. No Jira tooling → say so once and skip the mirror.
- Post or link the description, and write the key or URL into the front-matter as `issue: <key-or-url>`.

The durable copy is the card for a plan, and the repo for a report. The tracker is the review mirror.

## When to write an ADR

All three must be true:

1. **Hard to reverse** — the cost of changing our mind later is meaningful
2. **Surprising without context** — a future reader would look at the code and wonder "why?"
3. **A real trade-off** — there were genuine alternatives and we picked one for specific reasons

If any is missing, skip it. The obvious choice doesn't need documenting. If the user agrees to capture one,
write `docs/decisions/NNNN-slug.md` using [adr-format.md](../adr-format.md), make sure git tracks it, and attach its **path** as `adr-NNNN` (a location attachment, not its contents). Create `docs/decisions/` lazily — only when the first decision is needed.

Offer it, don't announce it: *"That decision to [summary] feels worth recording — it's [hard to reverse /
surprising / a real trade-off]. Want me to capture it?"*

## When to add a glossary term

Did a fuzzy or overloaded term get resolved? Typical cases:
- The user said "account" but meant the billing entity, not the user profile
- One concept has several names in conversation ("materialize", "publish", "go live")
- A new term was coined that a future session should know

If the user agrees, add it to `CONTEXT.md` using [context-format.md](../context-format.md). Create `CONTEXT.md`
lazily — only when the first term is resolved. Capture inline, as it happens; don't batch to the end, and
don't force it when neither an ADR nor a term applies.
