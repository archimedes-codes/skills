---
name: specify
description: Use when an approved design or a researched decision needs to become a written spec plus an implementation plan with concrete tasks, file paths, and test steps before coding begins. This skill is the ONLY writer of a plan; the plan is attached to the Task card, never written into the repo.
---

# Specify

Write the spec and the plan in one pass, in one file: the WHAT/WHY, then the HOW-IN-ORDER that
delivers it, as independent, commitable tasks.

**You are the only writer of the plan.** `research` produces evidence and an approved
design in chat; it deliberately writes no plan so that one skill owns one document. That is why
the spec and the plan arrive together from you rather than being authored by two skills that would
each phrase the same decisions differently.

**Write for a dumb agent.** The executing agent has no memory of your conversation, no knowledge of intent, and no ability to infer what you meant. Every task must be self-contained and complete enough that a capable but context-free agent can execute it without guessing. If something is ambiguous, spell it out. If something could be done multiple ways, specify which way. Leave nothing to interpretation.

## Where the plan lives

**The plan is a Task card artifact, not a repo file.** Never write it to `docs/roadmap/` or anywhere else in the repository, gitignored or not.

- **Working copy:** `<scratch>/plan.md`, where `<scratch>` is `~/.cache/agent-plans/<topic>/`. It is outside every repo, so nothing in it is committed or ignored by accident. Write here first.
- **Durable copy:** the card's `plan` artifact. Attach it with `write_artifact` after every write and every reviewer round, then confirm with `get_artifact`. The card copy is the only copy that survives, so the plan is not finished until the attach is verified.
- **No card** (no `<task>` block and no `task:` key): the plan lives only in scratch on this machine. Say so once in your report. Do not present it as attached.
- The attach procedure, the key table and the no-card rule are in `research`'s Durable Capture guide.

## When to Use

- After `research` hands off — either an approved design (its Path B) or a delivered report whose
  answer the user chose (its Path A)
- When a user asks directly for a spec and a plan
- When work is large enough to benefit from task breakdown (2+ tasks)

**Don't use when:**
- **The design is not settled** — the gap is intent, not wording. Load `research` (Path B) instead.
  `specify` writes down a design; it does not have the conversation that produces one. If you find
  yourself inventing answers to questions nobody has asked, that is the signal to stop and route back.
- Change is small enough to implement directly (single file, obvious fix)

## Invocation Modes

This skill runs in one of two modes. The mode is chosen by an **explicit marker in the invoking text** — never inferred from the filesystem.

### Standalone mode (default)

The behavior described in the rest of this document, unchanged: read the input → **ground (`researcher`, or inline if dispatched)** → **write the spec section** → write the plan → reviewer vet (max 3 rounds) → **HARD BREAK** → `ask` → optionally load the `implement` skill. Use when a user or entry skill loads `specify` directly.

There is **one** approval gate — the HARD BREAK — and it covers the whole document, spec and plan
together. Section-by-section design approval already happened upstream, in `research` Path B; asking
again here is the same question twice.

### Consumed mode

Entered **only** when the invoking task text contains the exact marker `SPECIFY CONSUMED MODE`. Used by the
`implement` skill's COARSE path, which runs `specify` once per task. Consumed mode is normally run **as a dispatched child**
(`researcher`, `agentName: "researcher"`), which has **no `subagent` tool** — so consumed mode never dispatches anything itself; the caller
owns every sub-agent. In consumed mode:

- Plan exactly **ONE task** (described by the task text), NOT a whole feature. The invocation also supplies a
  **grounding report path** and the task's **owned file set** — consume both (see *Ground the Plan First*); a
  missing report path is the only grounding you should ever have to do yourself.
- **`Builds on:`** — the verbatim signatures this task consumes from other tasks in the same run, each with its
  producing task id, or `none`. Plan against these signatures. Other tasks' plan files do not exist yet (all plans
  are written in one response), so do not read them. Do not invent a signature that is not listed.
- **Ground the plan inline** — read the files, grep the real signatures, confirm each path exists (see *Ground the Plan First*). If you genuinely have a `subagent` tool, dispatch `researcher` instead and say so.
- The target path is the `target:` stated in the invocation text (e.g. `<scratch>/plan-a1.md`). If no `target:` is present, **stop and report NEEDS_CONTEXT** — never guess a path.
- Write the plan to the target path with front-matter `status: committed`, a `parent: <feature>` key (the run's topic; its parent doc is the card's `parent` artifact), and `done-when`. The target file is the **only** file you write — never a grounding report, never source, never the status file.
- **The reviewer vet is the caller's, not yours.** Do not dispatch a reviewer — a child cannot, and promising one silently produces an un-vetted plan. `implement`'s COARSE path vets the plan you write (see its Step 4) and re-dispatches you to refine it if the vet finds anything. If you *are* running with a `subagent` tool, run the vet as described in *After Plan is Written*.
- Perform **neither** the card attach **nor** the issue-tracker mirror — the caller (`implement` COARSE) owns both. Its Step 4 attaches your target to the card as `plan-<id>`; subagents cannot touch the board.
- Do **NOT** perform the HARD BREAK, do **NOT** call `ask`, do **NOT** load the `implement` skill. Return control with a one-line "plan written to <target>" report, plus anything you could not verify so the caller's vet can target it.

## Input

Three things can arrive. **A missing plan is the normal case, not an error** — `research` deliberately
writes no plan, so a handoff from it has nothing to read. The input is the handoff text.

**In consumed mode ignore this section** — the target path is the `target:` given in the invocation text, no `ask` is permitted, and the 'no plan yet' branch never applies. If a `status: committed` plan already exists at the target (a re-run after a BLOCKED task), REFINE it in place (as the standalone committed-branch already describes) rather than re-planning from scratch.

Read the existing plan first: with a card, `get_artifact(task_id, "plan")` and restore it to `<scratch>/plan.md`; with no card, read `<scratch>/plan.md`.

- **No plan yet** (the common handoff from `research`, or a direct ask) → **you write the spec section
  yourself, then the plan.** Take the design from the handoff text or the research report it names.
  If the design is genuinely absent — no approved design, no chosen answer, nothing to write the WHAT
  from — **do not invent one.** Route back: tell the user the design isn't settled and load `research`
  (Path B). The failure this guards is the confident invented requirement, which the executing agent
  will follow exactly as written.
- **Plan exists, `status: approved`** (a spec, no plan yet — written by an earlier `specify` run, or a
  doc predating this merge) → keep the spec content, add the plan below it, and move `status` to
  `committed`. Read the existing spec first; do not rewrite decisions the user already approved.
- **Plan exists, `status: committed`** (a plan already exists) → **refine the existing plan; do NOT
  re-plan from scratch.** Read the existing plan and update it in place: fill gaps, add the
  information and rules this skill carries (concrete file paths, exact signatures/types, test steps,
  acceptance criteria, task breakdown), and keep what is already correct. This is the path taken when
  `research`'s pre-check routes an existing plan back here. The file stays `status: committed` — you
  are improving the plan, not regenerating it.

If the topic isn't already known (no handoff named it, no path given) and more than one plan is in scratch with no card naming one,
ask the user which plan to work on. **A topic you invent will not match the scratch directory and filenames the run uses** — when a handoff names a topic, use it verbatim.

## Ground the Plan First

**Never write a plan from memory of the codebase.** Ground it — write the plan from what the tree actually
contains. The plan's entire value is that it names *exact* paths, signatures and commands; an invented path is
worse than no plan, because the executing agent follows it confidently and nothing downstream can tell a
verified path from a plausible one.

**How you ground depends on whether you can dispatch — and only you know that.** Do not assume, and do not
stall looking for a tool you were never given:

- **You have a `subagent` tool** (you are the main agent, running standalone `specify`) → **dispatch
  `researcher`** and write the plan from its report.
- **You do not** (you were dispatched as a child, which is how `implement`'s COARSE path runs consumed `specify` via `researcher`) → **read the
  grounding report the caller hands you.** An `implement` COARSE dispatch supplies a path like
  `<scratch>/ground-<area>.md`; open it and plan from it. It already contains the file inventory,
  the verbatim signatures, and the test commands — that is why it exists.
  - **Read the report first, then read only what it does not answer.** Do not re-walk the tree to re-confirm
    what the report already states. A planning child that ignored its report and re-derived the codebase from
    scratch cost **359.7k tokens / 88 tool calls / 281% of its context budget**, compacting and re-reading as it
    went — the report is the fix for exactly that, so ignoring it is the expensive choice, not the careful one.
  - Stay inside the **owned file set** the caller gave you: read your set and the report, and treat files
    outside it as out of scope.
  - No report was supplied? Then ground yourself — read the files, grep the real signatures, confirm each path
    exists — and **say in your report back that no grounding was supplied**, so the caller fixes it for the rest
    of the run. Say once that you are grounding inline; never report it as a blocker.
- Either way the plan must be grounded. The mechanism changes; the requirement does not.

When dispatching, one `researcher` per independent area of the change — all calls in **one response** so they
run concurrently:

```typescript
subagent({ agent: "researcher", task: `Grounding for plan: <feature or task name>

Find and report, with file paths and line references:
1. Every existing file this work will touch or depend on — confirm each path exists on disk.
2. The real signatures / types / field names of anything this plan will call or modify. Quote them verbatim.
3. Conventions this codebase already uses for that kind of change (error handling, module layout, naming).
4. The exact test, format and build commands, and where test files for this area live.
5. Integration points — who calls this code, and what would break.
6. Anything already implemented that the plan might duplicate.

Sources: local codebase. Report findings with paths; flag anything you could NOT confirm.` })
```

- **`researcher` for planning, `general` for implementation** — `general` is the *implementation* child; it writes code and commits.
  Grounding and consumed planning are read-only to code and touch only markdown files; they run via `researcher`. (When you are the consumed-mode child doing inline
  grounding, you are already read-only for the duration — do not edit source while grounding.)
- **Not the `reviewer`** — the reviewer runs *after* the plan exists (see *After Plan is Written*) and checks it
  against the tree. Grounding runs *before*, so the plan starts from facts instead of being corrected afterwards.
- **Report anything that could NOT be confirmed** — by the researcher, or by your own reading. Mark those items
  explicitly `UNVERIFIED: <reason>` in the plan rather than inventing a plausible path. An honest gap is
  actionable; a confident guess is not. The caller's reviewer uses these to target its vet.
- **Scale to the work** — a one-file change needs one quick researcher; a multi-area feature needs one per area.
  Two angles that depend on each other are one angle with N steps: give them to a single researcher.
- **Consumed mode:** the caller supplies a **grounding report path** and the task's **owned file set**. Read the
  report; do not re-derive the tree. Scope everything to the owned set — the caller uses it to prove the wave's
  tasks are disjoint, so a plan that reaches outside it breaks the wave. No `ask` is permitted, so never pause to
  query the user about a gap; record it as `UNVERIFIED: <reason>` in the plan instead.

Never skip grounding silently. If you could not ground part of the plan, say so in the plan itself rather
than writing a confident path you never opened.

## Parallelisation

**Seek waves. SERIAL is the finding, not the default.** `implement` runs a PARALLEL plan with one
child per task in its own worktree, so a plan that could have been two waves and is written as one
long serial chain costs real wall-clock time — and the cost is invisible, because a serial plan
completes correctly and nothing reports the parallelism that was left on the table.

So write the tasks first, then **actively look for the largest set that clears all six gates**, before
reaching for the SERIAL verdict. Three places a hidden wave usually is:

- **Split a shared file's edits into a prep task.** If four tasks all touch `routes.py`, one task that
  adds the routes/scaffolding first can leave four disjoint tasks behind it — that is a wave of four
  behind a serial one, not a serial run of five.
- **Tests are usually disjoint even when sources are not.** Two features that both register a route
  collide in the source; their test files don't. If the collision is only in registration, that
  registration is the prep task.
- **Independent areas of the tree** (a new module, a new command, a new component) almost always
  parallelise; the collision is in whatever barrel file, router, or `mod.rs` wires them in. Extract
  that wiring.

**But the six gates below are absolute, and a gate is never traded for speed.** A wrong PARALLEL is
strictly worse than a right SERIAL: it wastes a whole child's work on a fan-in conflict, or worse, it
lets a child invent an interface that exists only in a sibling's branch — green alone, broken at
integration, unattributable. "It feels independent" is not a gate. If after looking you genuinely
cannot find a wave, write SERIAL **with the reason** and stop — do not manufacture a split to look
parallel.

Decide this while writing the plan, not after. Emit a `## Parallelisation` verdict and a
`**Parallel:**` line on **every** task — those two are all `implement` reads.

**Consumed mode exception:** a consumed-mode plan contains exactly one task, so it has nothing to parallelise
with. Write `## Parallelisation` with a `SERIAL` verdict (one task cannot be a wave) and emit no `**Parallel:**`
line. Parallelism in a COARSE run happens **between tasks**, in the wave grouping `implement` owns.

Test each candidate wave against ALL SIX gates. A task joins a wave only if it clears every one; a task that clears none runs alone (which does not disqualify the tasks that do clear them).

1. **Disjoint files** — `Files:` lists (Create + Modify + **Test**) share ZERO paths, pairwise.
2. **No build-graph dependency** — A cannot `import`/`use` a symbol B creates in the same wave.
3. **No shared mutable artifact** — lockfiles, migrations, generated code, shared fixtures, `mod.rs`/`__init__.py`/barrel files, route or middleware registration, config. **Check the commands, not just the file lists**: a task verified through `uv run` / `npm test` / `cargo test` can rewrite a lockfile or write snapshots without naming them in `Files:`.
4. **Provable from the base alone** — acceptance criteria provable in a checkout holding the base plus only this task's work. Each child runs its own test files in its worktree, and the full suite runs once after fan-in. A criterion can be checked by the full suite; needing a *sibling's* work present to pass is still disqualifying.
5. **No tree-wide mutation** — whole-tree formatters, lockfile regeneration, codegen. Harmless inside one worktree, but two children who both rewrote everything collide across every file at fan-in.
6. **Wave size ≤ 4** — beyond four, each new worktree pays its own dependency install and the global caches (`~/.cache/go-build`, module cache) are still shared, so the win flattens.

**Why this is not just "are they independent?":** every wave child works in **its own worktree on its own branch** (see the `trees-and-branches` skill), so siblings cannot corrupt each other's tree or index — but they also cannot *see* each other. A child's suite result proves its own task and says nothing about the combination, and two children touching one file each produce a clean commit that collides at fan-in. Getting gates 1–3 wrong no longer crashes; it wastes a whole child's work on a conflict, and gates 2 and 4 let a child invent an interface for work that exists only in a sibling's branch — green alone, broken at integration, unattributable.

Consult **[Parallelisation Guide](guides/guide-parallelisation.md)** for worked wave/serial examples, the near-miss checklist, and how to word each `**Parallel:**` line.

**State the verdict even when it is SERIAL — with the reason.** A plan with no verdict is read by `implement` as SERIAL anyway; the difference is that a reason tells the next reader whether serial was decided or merely assumed.

## Document Format

Write **one document** to `<scratch>/plan.md`, then attach it to the card as `plan`, containing **both** sections, in this order: the spec (WHAT/WHY), then the
plan (HOW-IN-ORDER). The spec section **survives** — it is not replaced by the plan. One file, one
writer, both documents, so a later session can read the design decision without reconstructing it from
a task list.

If a `status: approved` spec already exists, keep its body verbatim and append the plan; only revise
spec text the user asks you to change. …unless consumed mode names a different `target:` — then write
there instead, with the extra `parent:` front-matter key, and skip the spec section (the parent doc
holds it).

```markdown
---
status: committed
task: <task_id>            # the card's id; omit only when there is no card
done-when: <observable exit condition — what does "shipped" look like for a real user?>
---

# [Feature]

## Spec

**Problem:** What hurts, or what is missing. One or two sentences.
**Solution:** The chosen approach, and the shape of it. 2-4 sentences.
**Out of scope:** What this deliberately does NOT do. (This line is what stops a plan from quietly
growing — name the adjacent thing you rejected and why.)
**Decisions:** The non-obvious choices, each with its reason. Link any ADR in `docs/decisions/`.
**Evidence:** If this came from a research report, name it (`docs/research/<file>.md`) and the
findings the design rests on, with their sources. A plan with no provenance cannot be re-checked when
the library releases v3.

## Plan

**Goal:** One sentence
**Architecture:** 2-3 sentences
**Tech Stack:** Key technologies

---

## Parallelisation

**Verdict:** PARALLEL — 2 waves
- Wave 1: Task 1, Task 2, Task 3 (disjoint file sets, verified pairwise)
- Wave 2: Task 4 (needs Task 1's `AuthError`, so it cannot join Wave 1)

<!-- Exactly one verdict. The SERIAL form:
**Verdict:** SERIAL — every task shares `src/routes.py`, so no two tasks have disjoint file sets.
-->

---

### Task N: [Name]

**Parallel:** wave 1 — disjoint from Task 2 (`src/a.py`, `tests/test_a.py`) and Task 3 (`src/b.py`, `tests/test_b.py`)
<!-- or: **Parallel:** none — shares `src/routes.py` with Task 2. Every task carries this line; it is the field `implement` dispatches on. -->

**Context:**
A short paragraph explaining *why* this task exists, what problem it solves, and how it fits into the overall feature. Include any decisions already made and why. Do not assume the agent knows anything about the conversation.

**Files:**
<!-- In consumed mode these MUST fall inside the owned file set the caller gave you. A file outside it is not
     yours to change: mark it `UNVERIFIED: outside owned set` and report it, never silently add it. -->
- Create: `exact/path/file.ext`
- Modify: `exact/path/existing.ext`
- Test: `tests/path/test.ext`

**What to implement:**
A precise description of the change. Include:
- Exact function/struct/module names to add or modify
- Exact signatures, types, or field names where relevant
- Any specific logic, conditions, or edge cases to handle
- What NOT to change (if there's a risk of over-engineering)

**Steps:**
- [ ] Write failing test for [specific behavior] in `tests/path/test.ext`
- [ ] Run `[exact test command]`
  - Did it fail with [expected error]? If it passed unexpectedly, stop and investigate why.
- [ ] Implement [specific thing] in `exact/path/file.ext`
- [ ] Run `[exact test command]`
  - Did all tests pass? If not, fix the failures and re-run before continuing.
- [ ] Run `[exact format command]` (e.g. `cargo fmt`)
  - Did it succeed? If not, fix and re-run before continuing.
- [ ] Run `[exact build command]` (e.g. `cargo build`)
  - Did it succeed? If not, fix and re-run before continuing.
- [ ] Commit with message: "[suggested commit message]"
  - **Wave tasks differ:** a task declared `**Parallel:** wave N` commits on its own task branch, per *Parallel wave rules* below; a serial task commits on the feature branch. Write the commit message here; the child uses it verbatim.

**Acceptance criteria:**
- [ ] [Specific, verifiable outcome]
- [ ] [Another specific outcome]
```


Each task must be independently commitable. The agent executing it has no context beyond what is written here — be exhaustive.

## Parallel wave rules

The invariants the `**Parallel:**` lines encode — `implement` executes these, so a plan claiming PARALLEL must hold all four:

- **Each wave child gets its own worktree and branch** (`feature/<topic>-taskN`, all from one recorded base) and **commits there** with the plan's message. It must never touch the feature branch or `main`. A wave that ends up width 1 runs as a plain serial task, and serial tasks work in the primary checkout and commit as usual.
- **In-wave verification is complete but local** — the child runs its own test files in its worktree, where that result measures only its own work. It cannot observe a sibling, so it cannot certify the combination.
- **The orchestrator fans the branches in, in plan order, then runs the full suite on the integrated tree** — so acceptance criteria are provable by per-child suite + integrated suite together. Each task stays its own commit, which is what makes a post-wave failure bisectable.
- **A wave is a barrier** — dispatch all of wave N together, wait for all, fan in, then wave N+1. A BLOCKED sibling holds the following waves; its partial work stays isolated in its own worktree.

Full protocol, failure handling, and the child dispatch wording live in the **implement** skill's Parallel Execution Guide — that is `implement`'s to read at execution time, not yours.

## After the Doc is Written

**Attach first.** The moment the plan is written to `<scratch>/plan.md`, attach it to the card as `plan` (see *Standalone mode* below), *before* the reviewer runs. A reviewer or session that dies after this point then leaves the card holding the plan, not only a scratch file. Re-attach after each reviewer round.

Dispatch the **reviewer subagent** to review the whole document — the spec section and the plan:

```
subagent({
  agent: "reviewer",
  task: "Review type: spec + plan review. Review the document at `<resolved doc path>`. It has two sections and BOTH need vetting.

SPEC section: is the stated design actually implementable in this codebase? Are the decisions consistent with each other and with the repo's conventions and existing ADRs? Does `Out of scope` conflict with anything the plan does? Name any requirement that is impossible, contradicts the code, or silently assumes something unverified.

PLAN section: this is NOT a review of the file as a document — it is a review of the plan against the actual codebase that will be modified. Read every file the plan references (creates, modifies, tests). Read any additional files needed to verify the plan is correct: existing interfaces, types, imports, dependencies, build configs, and related modules. Report anything the plan should do differently: missing files, wrong paths, incorrect signatures, overlooked edge cases, integration gaps, or anything the executing agent would get wrong. Also check the `## Parallelisation` verdict: for every task you are told is in a wave, verify its `Files:` list is genuinely disjoint from each sibling's — including shared registration points, fixtures, and lockfiles that a task's test command may touch without naming.

If you need to read the whole codebase to be sure, do it. The plan will be executed verbatim by an agent with no context — leave no stone unturned."
})
```

`<resolved doc path>` is the doc's scratch file in standalone mode, or the invocation's `target:` in consumed mode.

**Fix issues (max 3 rounds).** Round 1 is a full review. A round *after* a fix pass is a different
job, and saying so is what makes it useful: scope it to **verify each fix against the tree** and
**check the fold-in did not introduce a new error** — an edit made to resolve a finding is itself an
unverified edit until a round has seen it. In practice a second round finds errors the fixes created
(a sentence left behind now contradicting the step it describes, a count corrected in one place and not
its twin, a claim that was true of the old text). Do not re-litigate findings the round already
resolved, and do not spend the final round on fixes that then go unvetted: if you edit after the last
round, say in your report that those edits were not reviewed.

**Standalone mode only — attach the plan first.** Plans are not committed; they are card artifacts. Attach the whole document before anything else: `write_artifact(task_id, key: "plan", markdown: <the whole doc as written, spec section included>)`, then `get_artifact` to confirm it landed. The card id is the doc's `task:` key, or the `<task>` block forwarded to you. Re-attach after every reviewer round. The upsert replaces the whole artifact, so if the card already holds a `plan` from an earlier run, read it with `get_artifact` first and refine it rather than replacing it. Because you write spec and plan into one file, the `plan` artifact is the copy that carries both: attach the whole document, not the plan section alone. If an earlier `research` run attached a `spec` or `research` artifact, leave those keys alone: they are the evidence trail, and `plan` is the deliverable. Do not collapse two keys into one. Do not set `spec_path` to the scratch file; if the card's `spec_path` already names a `docs/roadmap/` path, clear it with `edit_task(task_id, spec_path: null)`. No `write_artifact` tool, or no card id → say once that the plan exists only at `<scratch>/plan.md` on this machine, and continue.

**Standalone mode only — mirror to the issue tracker.** In consumed mode the caller owns the mirror. Resolve the tracker per `research`'s Durable Capture guide (AGENTS.md / CLAUDE.md first, never `git remote`). The card copy is the durable one; the tracker is the review mirror. Reuse the issue already tied to this topic (the doc's `issue:` front-matter key). If GitHub, attach the scratch file with `gh issue artifact create <issue-number> <scratch>/plan.md`, or update it with `gh issue artifact edit <issue-number> <artifact-number> -F <scratch>/plan.md`, falling back to an issue comment if artifact support is unavailable. If Jira, post an update comment via Atlassian MCP. Carry the key forward if no earlier step posted. No tracker found → say so once and continue; this never gates the HARD BREAK.

**CRITICAL: Do NOT begin implementing any tasks in the plan. The `specify` skill ends once the plan is vetted and presented to the user. The plan is handed off to the `implement` skill, which will ask the user to confirm plan selection before executing.**

**In consumed mode, skip the HARD BREAK below** — do not ask, do not load `implement` (its COARSE path is what dispatched you — loading it recurses); return control to the caller. **The reviewer dispatch above runs only if you actually have a `subagent` tool** (true standalone, false when you are the consumed-mode child) — when you do not, the caller (`implement`'s COARSE path) vets the plan you wrote; never report a vet you did not run as one you did.

**HARD BREAK — STOP here and ask. Do NOT continue into implementation on your own.**

1. **Clear the todo list** — use `manage_todo_list` to remove all entries now that planning is complete.
2. Tell the user the plan is ready (card and artifact key, or the scratch path if there is no card) and give a one-line summary.
3. Get explicit confirmation with the `ask` tool. **The question depends on whether this
   run knows a Task card** — the plan's front-matter `task:` key, or a `<task id="tsk_…">`
   block forwarded by the calling skill.

   **With a Task card** — the handoff replaces "implement now", so load
   `move-task` and ask. **Read `list_board` and name the columns the way that board labels
   them** — do not paste a column word from this skill. A card arriving here is usually *already*
   in the parking lane its `<task stage="…">` names, so "move the card to Approved" is a no-op
   question, and a Space owner can rename that column to anything (`Ready for Build`, `Approved`,
   `Groomed` — all the same `slug`). Name the lane the card is in and the destination the arrows lead
   to, both from the rows. Keep the `id` spelling the board matches on:

   ```
   ask({
     questions: [{
       id: "move-to-approved",
       question: "The plan is attached to the card as its `plan` artifact. The card is in \"<real label of its current stage>\". Claim this Session done so the card advances to \"<real label of the destination>\" and a fresh Session picks it up?",
       options: [
         { label: "Claim done — advance to \"<real destination label>\" (recommended)" },
         { label: "No — keep the card where it is" },
         { label: "Revise the plan first" }
       ]
     }]
   })
   ```

   Never promise what the claim will do. The rows give you `slug` and `label`; the `auto` flag and
   the destination's `on_enter_skill` decide whether the card actually moves and what launches, and
   `move-task` forbids naming a Skill you have not read there. Say "hand it forward", not "it will
   start implementing".

   On the **advance** option: that skill attaches the artifacts and verifies with `get_task`.
   The destination is **forward**, and forward is not `move_task`'s door — a model's forward
   `move_task` is refused (even on a default, unconfigured board) and its refusal names
   `mark_done`. So the hand-off is **`mark_done`**: it flags this Session as done and, if the
   column it points at is automatic, advances the card and launches the next Session itself.
   Read its answer and report what it said: either the card advanced (then say so, and that a
   fresh Session holds the card's artifacts), or the column is not automatic and a person
   moves the card or clicks its arrow pill on the Tasks board. **Then stop — do not
   load `implement`.** The whole point is that implementation happens in a fresh Session
   with the card's artifacts, not in this session's leftover context.

   **Without a Task card** — nothing to move, so keep the old question:

   ```
   ask({
     questions: [{
       id: "proceed-to-implementation",
       question: "The plan is ready at <scratch>/plan.md, but this session has no Task card, so it exists only on this machine. Would you like to proceed with implementation now?",
       options: [
         { label: "Yes — start implementation (load implement skill)" },
         { label: "No — hold the plan for later" },
         { label: "Revise the plan first" }
       ]
     }]
   })
   ```

4. **Only if the user explicitly confirms implementation**, load the `implement` skill.
   Otherwise stop and wait for the user to decide. Do NOT load `implement` or do any
   implementation work without that explicit confirmation — and note that on the
   card-backed path no answer routes here, because the board launch does instead.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Tasks that depend on each other's uncommitted work | Make each task independently commitable |
| Vague file paths ("update the config") | Use exact paths: `src/config/auth.ts` |
| Skipping TDD steps in task template | Every task needs failing test → implement → pass |
| Planning a design that was never agreed | Route back to `research` Path B and settle the design with the user. If you are inventing requirements nobody stated, that is the signal — do not write them down as if they were |
| Too many tasks (10+) | Group related changes; aim for 3-7 tasks |
| Sliding into implementation right after the plan | HARD BREAK — stop, ask via `ask`, and only load `implement` on explicit user confirmation |
| Writing the plan from memory — invented paths, signatures, or test commands | Ground first: dispatch `researcher` if you have a subagent tool, otherwise read the files yourself. Mark anything unconfirmed `UNVERIFIED` |
| Performing the HARD BREAK, attaching to the card, or mirroring to the tracker in consumed mode | Consumed mode writes the plan to its scratch target and returns; the caller owns the card attach + tracker |
| Creating a plan index or done/ folder | No index, no graveyard — plans are self-contained; history lives on the card and in git |
| Writing the plan into the repo (`docs/roadmap/`, gitignored or not) | Plans are card artifacts. Scratch is the working copy; the repo is not a plan location |
| Omitting the `## Parallelisation` verdict | `implement` reads a missing verdict as SERIAL. Declare SERIAL with a reason — a missed parallel opportunity is invisible; a wrong one is not |
| Declaring a wave because the tasks are *logically* independent | Independence of features is not disjointness of paths. `routes.py`, a shared fixture, or a `mod.rs` collide regardless of how separate the features feel |
| Letting a wave child certify the combination | Its own worktree proves its own task and cannot see a sibling's. The integrated suite after fan-in is the only measurement of the wave together — and prove it ran tests: a package with no discovered test files still exits 0 |
| Asking the user to "move the card to Approved" | Read `list_board` and name the real labels. The card is usually already in that lane, and the column may not be called Approved. Forward is `mark_done`; name the destination, promise nothing about what launches |
| Treating a fix to a plan as verified because a reviewer asked for it | The fix is a new edit. Send it back through a round scoped to "verify these fixes + did the fold-in break anything" |

More parallelisation mistakes and worked examples: **[Parallelisation Guide](guides/guide-parallelisation.md)**.