---
name: research
description: |
  Use to investigate before deciding — comparing libraries or frameworks, evaluating
  approaches or architectures, investigating how something works, finding best
  practices, competitive analysis, researching academic papers — OR to talk through an
  idea, design a feature, or brainstorm before coding ("let's discuss", "lets
  brainstorm", "I want to think this through"). Triggers on: "research", "compare",
  "evaluate", "deep-dive", "investigate", "best practices for", "discuss", "brainstorm",
  "let's talk through". Not for: simple file lookups (use explore directly), quick fact
  checks, or tasks requiring code changes.
---

# Research

Turn an unclear question into a decision you can defend — either by gathering evidence
(**Path A**) or by talking through the design with the person (**Path B**). Both start the
same way: ask what the user actually means before doing any work.

Neither path writes code, neither loads another skill, and neither hands off to implementation. Each
one ends by writing its document, attaching it to the card, and calling `mark_done`.

## Architecture

```
Main agent (YOU)
  │
  ├── Which path?  (evidence gap → A · design gap → B)
  │
  ├── Path A — RESEARCH
  │     ├── Phase 0: Clarify the question FIRST (ask, don't guess)
  │     ├── Phase 1: Classify + plan all research angles
  │     ├── Phase 2: Dispatch N parallel researcher subagents   ← pure leaves
  │     ├── Phase 3: Synthesise all findings
  │     ├── Phase 4: HARD STOP — present draft + ask            ← required
  │     └── Phase 5: Deliver (report → card → tracker → mark_done)
  │
  └── Path B — DESIGN
        ├── B0: Pre-check the existing doc (don't re-run a finished design)
        ├── B1: Settle the domain language (glossary, fuzzy terms, code cross-check)
        ├── B2: Explore context (AGENTS.md, files, commits)
        ├── B3: Clarifying questions — one at a time via ask
        ├── B4: 2-3 approaches with trade-offs + recommendation
        ├── B5: Present the design section by section, approval per section
        └── B6: Design approved → write the design doc + card → mark_done
```

**You own all orchestration.** Researcher subagents are leaves — one targeted angle each, they never
plan, dispatch, or synthesise. No nesting.

**This skill ends where its own document ends.** It writes its output — the research report, or the
approved design — to the repo and attaches it to the Task card, then calls `mark_done`. It loads no
other skill. `specify` is the writer of the **plan**, and it is started by a fresh session from the
card, not by this one. That division is what stops two skills from writing one document differently,
and it is what keeps a research session from rolling on into planning nobody approved yet.

## Which path?

| | Path A — RESEARCH | Path B — DESIGN |
|---|---|---|
| **The gap is** | Facts you don't have | Intent you don't have |
| **Typical ask** | "Compare X and Y", "how does X work", "is this library viable", "what do others do" | "Let's discuss X", "design a feature for Y", "brainstorm Z" |
| **Answer comes from** | Sources — web, code, papers, dispatched researchers | The user — through questions and trade-offs |
| **Ends with** | A cited report on disk + its path on the card, then `mark_done` | An approved design doc on disk + its path on the card, then `mark_done` |

**Choose on what's missing, not on the verb.** "Should we use Postgres or Postgres + pgvector?" is
Path A even though the user said "should we" — the answer is in the world, not in their head.
"We haven't decided whether notes belong to a Space or a Session" is Path B even after research — the
answer is their intent, and no source can supply it.

**Both signals present → A first, then B.** Research the options, present the findings, then design
with them. Do not design on an unexamined option set, and do not research options the user hasn't
agreed are candidates.

**Ambiguous → ask. Never guess.** This is the question the merge exists to make routine:

```
ask({
  questions: [{
    id: "path",
    question: "<the thing they said> — do you want evidence about the options, or do you want to talk through what you actually want built?",
    options: [
      { label: "Research it — go find out, then bring back options with sources" },
      { label: "Design it — ask me questions and let's decide together" },
      { label: "Both — research first, then design from what you find" }
    ]
  }]
})
```

A session that picks a path and runs 40 tool calls on it is expensive to undo, and the user cannot
course-correct a report they never asked for. One question is cheaper than either failure.

## Shared foundations (both paths, before anything else)

1. **Read `AGENTS.md` / `CLAUDE.md`** — it carries the project's conventions, its validation bar, and
   **which issue tracker it uses**. It is also the tiebreaker when the conversation and the repo
   disagree about a convention.
2. **Read `CONTEXT.md`** at the repo root (or per-context, if `CONTEXT-MAP.md` exists). Use the
   established terms from your very first question. A glossary is how terminology compounds across
   sessions — starting a conversation on a term the project already settled wastes it.
3. **Note the Task card id** if a `<task id="tsk_…" stage="…">` block is in your input. See
   [Durable Capture](guides/guide-durable-capture.md) — it explains where the id comes from, why it
   must be copied into the document's front-matter, and what silently breaks when it isn't. No block →
   not a card session; skip every board step. **A plan with no card has no durable home** — say so
   before you finish, and do not write it into the repo to compensate.
4. **Resolve the tracker once** — order of precedence in
   [Durable Capture](guides/guide-durable-capture.md). Never derive it from `git remote`.

---

# Path A — RESEARCH

## Phase 0: Clarify the question (before dispatching anything)

Research is expensive — N subagents, each with its own budget — and the expensive failure is not a bad
answer, it is a **precise answer to the wrong question**. Ask first.

**A. One round, up to three questions, via `ask`.** Batch them in a single call; do not drip-feed.
Ask only what changes the angles you would dispatch:

```
ask({
  questions: [
    { id: "decision", question: "What decision will this feed? (Choosing a library? A one-off design check? Writing a doc?)",
      options: [ { label: "Pick one to adopt now" }, { label: "Understand the landscape" }, { label: "Validate a choice I've already made" } ] },
    { id: "constraint", question: "Any hard constraint I should research inside?",
      options: [ { label: "Must stay in the existing stack" }, { label: "License / self-hosting matters" }, { label: "No constraints — best answer wins" } ] },
    { id: "depth", question: "How much depth?",
      options: [ { label: "Quick — one page of facts" }, { label: "Standard — 2-3 angles" }, { label: "Deep — 4-6 angles, cited" } ] }
  ]
})
```

**Skip the ask when the question is already answerable.** If it names the candidates, the decision,
and the stack, asking is friction — say what you inferred in one line ("researching X vs Y for the
storage layer, staying in Postgres, standard depth — say so if that's wrong") and go. The rule targets
*guessing*, not *reading*.

**B. Challenge fuzzy language the same way Path B does.** If the question contains an overloaded term,
that term is the ambiguity — not the missing detail. "Make it faster" — faster to build, or faster to
run? "Our account system" — the Customer or the User? Resolve it in the same `ask` round. A researcher
dispatched on an ambiguous term returns confident research about the wrong thing.

**C. Check whether the code already answers it.** Before dispatching a web researcher for "how does our
X work", grep. Local facts the tree already contains do not need a subagent, and
`specify`'s grounding rule is the same one: never write from memory what you can read.

**D. Record the settled question in one sentence** and keep it visible in the report. It is what makes
a later "this answered the wrong thing" cheap to spot.

> **Do not ask how to dispatch the angles.** That is not a user decision — see Phase 2.

## Phase 1: Classify (1 min)

1. **Research type:** web, code, academic, video, or hybrid
2. **Depth:** quick / standard / deep → determines number of angles (from Phase 0, or inferred)
3. **Design angles:** 1-6 specific, non-overlapping angles
4. **Task card:** the id noted in *Shared foundations* — Phase 5 needs it

Consult as needed: [Web Research](strategies/strategy-web-research.md) ·
[Code Research](strategies/strategy-code-research.md) · [Academic](strategies/strategy-academic.md) ·
[Video](strategies/strategy-video-research.md)

## Phase 2: Dispatch (main agent dispatches, never delegated)

**Always parallel.** Call `subagent` once per angle, **all in the same response** — N angles means N
tool calls in one message. Do NOT use the `tasks` array. Parallel calls issued from one response
execute simultaneously and each gets its own visible UI block. Quick depth is simply one call.

Never ask the user how to run the angles. There is no case where serial beats parallel here: the
angles are independent by construction (Phase 1), and the one legitimate want behind serial — steer
between steps — is what the Phase 4 hard stop is for, one round later and with every angle's findings
in hand rather than one.

**The one exception is an explicit user instruction.** The rule forbids *polling* the user for a
decision you can make, not overriding your default at their direction:

- The user says nothing, or anything short of a dispatch instruction → **parallel. No ask.** Never
  treat a hint, a deadline, or your own guess about their budget as an instruction — that is the same
  offloading, inverted.
- The user explicitly asks for serial / one-at-a-time → **comply for this run.** Name the trade-off in
  one line, then run it their way. Do not re-ask, do not silently switch back, and do not decide an
  informed user is wrong.

"Never ask" buys a default, not the authority to overrule.

```typescript
// Emit all of these as SEPARATE tool calls in ONE response:
subagent({ agent: "researcher", task: `Angle 1: [specific angle]\n\nFind: [exactly what]\nSources: [web / local / specific URLs]\nReport: key facts with citations, gaps if any.` })
subagent({ agent: "researcher", task: `Angle 2: [specific angle]\n\nFind: [exactly what]\nSources: [web / local / specific URLs]\nReport: key facts with citations, gaps if any.` })
// Quick — just one call
subagent({ agent: "researcher", task: "Find: [specific thing]. Report findings with citations." })
```

If an angle cannot be written until another has returned, that is **one angle with N steps**, not two
angles — give it to a single researcher, who sequences its own steps.

**Rules:**
- One `subagent` call per angle — never bundle angles into one call
- Never use `tasks: [...]` for research — that hides all children inside one UI block
- Be explicit about what sources to use (web, local files, specific URLs)
- Do NOT tell researchers to plan, synthesise, or dispatch — they are leaves
- **DO NOT add a `model` parameter to any subagent call.** The agent definition controls its own
  model; adding `model` causes hallucinated model names that break the call.

**Failure modes:** subagent returns nothing → note as gap, retry if critical. Times out → note the
timeout, skip the angle, record the gap.

## Phase 3: Synthesise (main agent only)

1. **Organise by research question**, not by which subagent found it
2. **Resolve contradictions** — present both sides with credibility assessment; never silently discard
   (see [Resolving Contradictions](guides/guide-resolving-contradictions.md))
3. **Build the evidence chain** — every claim linked to its source
4. **Identify gaps** — what remains unanswered

## Phase 4: HARD STOP — present the draft, then ask (REQUIRED)

> 🛑 **STOP HERE. Do not proceed to Phase 5 until `ask()` returns.**

```markdown
## Research Draft

### Executive Summary
### Findings            (organised by question, every claim cited)
### Unresolved Contradictions
### Gaps
```

```typescript
ask({
  questions: [{
    id: "research-next",
    question: "Draft ready. What next?",
    options: [
      { label: "Write the report and attach it to the card — done (recommended)" },
      { label: "Just talk — stop and let me react to the findings" },
      { label: "Move to Approved — deliver, then hand the card to a fresh Session" },
      { label: "Targeted deep-dive on specific gaps" },
      { label: "Add a research dimension" },
      { label: "Reframe the question" }
    ],
    multi: true,
    description: "**Write the report:** the report is the deliverable — it goes to docs/research/, its path goes on the card, and the session is handed to you for review. Planning is a later session's job.\n**Just talk:** stop. The findings stay in this session and you drive — nothing is written.\n**Move to Approved:** deliver, then hand the card to a fresh Session. Only offered when a `<task>` block was found.\n**Deep-dive / Add dimension / Reframe:** more research."
  }]
})
```

**Compose rules:**
- **"Move to Approved" only when a `<task id="tsk_…">` block was found** — an option that cannot run is
  a dead control. It composes with "Write the report" (delivering is a prerequisite, not an alternative).
- **No card → drop "to the card" from the first option's label** ("Write the report — done"), since the
  attach step cannot run; Phase 5's no-card branch covers it.
- **"Move to Approved" does NOT compose with "Just talk"** — that continues work in *this* session; the
  move ends it. If both come back selected, ask once which was meant. Do not resolve it by picking one
  silently.
- **"Just talk" is a real answer, not a stall.** It writes nothing. Do not quietly write the report, and
  do not keep going into planning because the findings "obviously" imply one.
- **No option here loads another skill.** If the user asks for a spec or a plan, say the report path is
  the handoff and that a session started from the card picks it up; offer to name the next step in the
  report's Gaps section. Do not load `specify` or `implement`.

## Phase 5: Deliver (after `ask()` responds)

Every path below ends the same way once its writes are done: **call `mark_done`** — it takes no
arguments, flags this Session as finished, and hands it to a person for review. It is the last call of
the session. Nothing is loaded after it.

| Choice | Action |
|---|---|
| **Write the report and attach it** (default) | 1. Write the report to `docs/research/<descriptive-name>.md` (front-matter below). **Do NOT re-print it — the draft was already shown.** 2. Make sure git tracks it (`git ls-files --error-unmatch <path>`; add it on the working branch if not). 3. Attach its **path** to the card (location attachment: the body is the repo path, not the report's contents) and set `spec_path` to the same path — [Durable Capture](guides/guide-durable-capture.md), key `research`. 4. Offer the tracker mirror — the ask in [Durable Capture](guides/guide-durable-capture.md); skip when no tracker was found. 5. **Clear the todo list**, state the report path in one line, then `mark_done`. |
| **Just talk** | **Stop.** Say one line — findings are above, I'm here. Write nothing. The user drives from here. No `mark_done`: the session is not finished, the user is still reacting. |
| **Move to Approved** | Deliver first (steps 1–4 above). Then load `move-task`: attach artifacts, read the Approved slug off `list_board`, `move_task`, verify with `get_task`, tell the user to **archive this session** and click the card's **arrow pill**. **Then `mark_done` and stop — do not load `specify` or `implement`.** |
| **Targeted deep-dive** | Dispatch more researchers on the named gaps → re-synthesise → Phase 4 |
| **Add a dimension** | Dispatch a new angle → re-synthesise → Phase 4 |
| **Reframe** | Back to Phase 0 with the revised question |

Report front-matter (always):

```yaml
---
status: current          # current | superseded
last-verified: YYYY-MM-DD
verified-by: web research — [N] sources, [date]
task: <task_id>          # omit only if no card; see Durable Capture
---
```

Sections: Executive Summary · Findings (by question, every claim cited) · Evidence (source details,
credibility) · Unresolved Contradictions · Gaps. If this supersedes an older report, set the old
file's `status: superseded` and link forward.

**The Gaps section is the handoff.** When the findings point at work to do, name the next step there in
one line ("plan the migration from this report — start with the lock-ordering finding"). A later session
starts from the card, reads `spec_path`, and finds it. That replaces any mid-session handoff: the card is
the bridge, not a skill loaded on top of this one.

**No card** (no `<task>` block): write the report, then say in one line that it has no card, so nothing is
attached and no `spec_path` was set — per [Durable Capture](guides/guide-durable-capture.md), offer to
create a card for it and only create one on the user's say-so. Either way `mark_done` afterwards: the file
on disk is still the deliverable. Never fall back to the repo for a *plan*; the report already lives in
the repo.

---

# Path B — DESIGN

Turn an idea into an approved design through dialogue. No code and no **plan** here — the plan (the
numbered HOW-IN-ORDER tasks) belongs to `specify`, in a session started from the card. What this path
*does* write is the approved design itself: a design doc in `docs/research/`, its path on the card, then
`mark_done`. That doc is the handoff `specify` reads.

## B0. Pre-check the existing doc (before the design flow)

Look for the existing plan and branch on its front-matter `status`. With a card, `get_artifact(task_id, "plan")` is the source; restore it to `<scratch>/plan.md` (see [Durable Capture](guides/guide-durable-capture.md) for `<scratch>`). With no card, read `<scratch>/plan.md` if it exists. The card is checked first, because a plan from an earlier session is on the card, not on this machine:

- **No plan found** → run the design flow (B1–B6).
- **`status: approved`** (a spec, no plan yet) → the design is done; **do not re-run the flow.**
  **Wording rule — spec ≠ plan:** a *spec* is the WHAT/WHY (behavior + design); a *plan* is the
  HOW-IN-ORDER (numbered tasks with TDD steps and verification commands). The task-level plan is
  `specify`'s work in its own session, so never offer it from here — offer only what this skill can do.
  ```
  ask({ questions: [{ id: "next-step",
    question: "The approved spec is attached to the card as its `plan` artifact — design done, no plan yet. Planning happens in a session started from the card. What should this session do?",
    options: [
      { label: "Mirror the spec to the tracker and verify the artifact — then done" },
      { label: "Revise the design" },
      { label: "No action for now" }
    ] }] })
  ```
  Mirror or revise → do it, then `mark_done`. No action → stop without writing.
- **`status: committed`** (a plan exists) → do not re-run the design flow, do not offer to plan, and do
  not approve anything: approval is `specify`'s HARD BREAK, in its own session. Offer `Revise the design`
  / `Mirror to the tracker` / `No action for now`, say where the plan lives, and end on `mark_done`.

## B1. Settle the domain language first

Prevents the whole conversation from drifting on fuzzy terms.

- **A. Read the glossary** — `CONTEXT.md` (or `CONTEXT-MAP.md` per context). Use established terms from
  the first question. If none exists, you'll build one.
- **B. Challenge fuzzy language** — when the user uses a vague or overloaded term, call it out
  immediately and propose a precise canonical term: *"You're saying 'account' — the Customer or the
  User? Those are different things."*
- **C. Cross-reference with code** — when the user states how something works, check whether the code
  agrees, and surface a contradiction: *"The code cancels entire Orders, but you said partial is
  possible — which is right?"*
- **D. Challenge against the glossary** — *"Your glossary defines 'cancellation' as X, but you seem to
  mean Y — which is it?"*
- **E. Stress-test with scenarios** — invent edge cases that force precision about the boundaries
  between concepts.
- **F. Capture terms inline** — when a term resolves, offer to add it to `CONTEXT.md` right then
  ([format](context-format.md)). Don't batch to the end.

> Do not proceed to B2 until terminology is settled. New ambiguities may loop back at any point.

## B2. Explore context

Files, docs, recent commits. `AGENTS.md` was already read (*Shared foundations*).

## B3. Ask clarifying questions — one at a time, via `ask`

Multiple-choice options. One question per `ask` call.

## B4. Propose 2-3 approaches with trade-offs and a recommendation, via `ask`

## B5. Present the design section by section, `ask` for approval after each

> **Always use `ask`** for B3–B5. Never present a section and continue to the next — wait for explicit
> approval. After each section, run the **ADR check** and the **terminology check** in
> [Durable Capture](guides/guide-durable-capture.md) — capture inline, don't batch, and don't force it
> when neither applies.

## B6. Design approved → write it, attach it, `mark_done`

Present the full design once, in chat. **Do not write the plan here** — the plan is the numbered
task-level breakdown, and `specify` is its only writer, in its own session. What you write now is the
*design*: the WHAT/WHY the next session plans from.

1. **Write the design doc** to `docs/research/<topic>-design.md` with the same front-matter as a research
   report (`status: current`, `last-verified`, `task: <task_id>` when a card exists). Sections: the settled
   terminology, the approach chosen and why, the rejected options with their trade-offs, the boundaries
   and interfaces, and an **Open questions / next step** section naming what planning must decide.
   Name the `<topic>` in one line so the next session names its scratch dir and card key the same way.
2. **Track it** — `git ls-files --error-unmatch <path>`, add it on the working branch if untracked.
3. **Attach its path** to the card as a location attachment and set `spec_path` to the same path —
   [Durable Capture](guides/guide-durable-capture.md). Key: `research` when this run produced no report
   of its own; if a Path A report is already attached under `research`, use the `<topic>` key instead and
   leave `research` pointing at the report — never collapse two documents into one key.
4. **Offer the tracker mirror** — the ask in [Durable Capture](guides/guide-durable-capture.md); skip when
   no tracker was found.
5. **Clear the todo list**, say the doc path in one line, then **`mark_done`**. Planning is the next
   session's job: it starts from the card, reads `spec_path`, and finds the design there.

Do not load `specify` or `implement` from here, whatever the scope. If the user asks for a plan or waves
now, say the design is on the card and the card's arrow pill starts a session that plans it; if they want
a card move, offer `move-task` (read the destination slug off `list_board`, `move_task`, verify with
`get_task`, tell them to archive this session and click the pill), then `mark_done`.

**No card:** write the design doc (steps 1–2), say in one line that nothing is attached because no card
exists, and `mark_done`.

**If the user asks to revise:** go back to the section that needs rework, re-approve, then re-run the
steps above — the doc is the same file, and `write_artifact` upserts the path.

**If the user wants to stop before approval:** stop. Nothing is written — an unapproved design is not a
deliverable. If this run produced an ADR or glossary terms, those were already captured (B5). No
`mark_done` on this branch: the design is unfinished and the user still owns it.

## The hard gate (both paths)

> 🚫 **No code, no `implement`, no implementation skill — until a human has approved, and approval
> does not decay into a continuation.**

After approval this skill still does not implement, and it does not plan. It writes its own document,
attaches it, and calls `mark_done`. The next decision point belongs to another skill in another session —
`specify`'s HARD BREAK, or `implement`'s one-time consent question (autonomous vs check-in) — reached by
starting that session from the card. Only an explicit confirmation from the user — or one of those — ever
leads to implementation work. This is the one rule that survives every restructuring of this skill,
because every failure mode it prevents is a session that got confident and kept going.

`mark_done` is where this session ends. It takes no arguments, flags the Session as done, and hands it to
a person for review. It is called **after** the writes, never instead of them, and never on a branch that
wrote nothing ("just talk", a design abandoned before approval).

---

## Evidence standards (Path A)

**Every claim carries a source** — URL, file path, or video timestamp. Distinguish **verified facts**
(directly observable in a cited source), **reasonable inferences** (from multiple sources), and
**speculation** (educated guesses with no direct evidence).

| Source type | Citation format |
|---|---|
| Open-source code | GitHub permalink with commit SHA |
| Local code | `repo:path:line` |
| Web | Full URL + title |
| Video | URL + timestamp |

**Credibility hierarchy:** official source code / docs → peer-reviewed papers / specs → well-known
experts / maintainers → community discussions (SO, GitHub issues) → blogs / tutorials. See
[Source Credibility](sources/source-credibility.md) and [Citation](sources/source-citation.md).

**Depth matches the question:** quick = 1 angle (a simple lookup) · standard = 2-3 ("how does X work?",
an approach comparison) · deep = 4-6 ("should we use X or Y?", architecture).

**Angle design** — each angle is narrow enough for one pass, different enough not to overlap,
**independent** (fully writable before any other returns), and together they cover the whole question.

| Question | Angles |
|---|---|
| "Compare A vs B" | A docs/features / B docs/features / benchmarks / adoption |
| "How does X work?" | Official docs / source internals / usage examples / known gotchas |
| "Best approach for X?" | Option A trade-offs / Option B trade-offs / real-world usage / codebase patterns |
| Hybrid | Local codebase patterns / web docs / web best practices |

## Failure modes & recovery

| Failure | Recovery |
|---|---|
| Question too vague | Ask in Phase 0 — before dispatching, not after |
| Path guessed wrong | Ask (see *Which path?*) — one question beats a 40-call wrong path |
| Subagent returns nothing | Note as gap; retry with a broader angle if critical |
| Sources conflict | Present both with citations, mark unresolved |
| Source paywalled / stale | Note it; try an archive; flag as a gap ([Stale Sources](guides/guide-stale-sources.md)) |
| Depth misjudged | User catches it at the Phase 4 hard stop and chooses deep-dive |
| Kept going after the answer | Phase 4 is a hard stop. Present, ask, **wait** |
| Loaded `specify` / `implement` at the end | Wrong. Write the doc, attach the path, `mark_done`. The card is the bridge to the next session |
| Finished with no report on disk and no card attachment | The findings die with the session. Deliver (Phase 5 steps 1–4), then `mark_done` |
| Design ran, nothing written | Wrong on both counts — an approved design is written (B6); a design that never got approval writes nothing and stops |

**When to stop** researching is a judgement, not a budget: [When to Stop](guides/guide-when-to-stop.md).
Before publishing, self-check: [Research Quality Checklist](checklists/checklist-research-quality.md).
Report skeleton: [Report Template](assets/research-report-template.md).

## Principles

- **Ask before working.** Phase 0 (evidence) and B3 (intent) both exist because guessing is the
  expensive failure, not slowness.
- **`ask` at every decision point** — one question per call, and never present-then-continue.
- **YAGNI** — remove unnecessary features; design for clear boundaries and single responsibilities.
- **Follow established patterns** in existing codebases.
- **Scale detail to complexity** — a few sentences if simple, more if nuanced.
- **Capture decisions and terminology inline** — don't batch, don't skip, don't force.
- **One writer for the plan.** This skill never writes the plan, to scratch, to the card, or to the repo. `specify` does, in its own session.
- **Repo for reports, card for plans.** A research report or design doc goes in `docs/research/` and the card holds its path. A plan goes on the card only.
- **End the session, don't chain it.** Deliver, attach, then `mark_done`. Loading the next skill mid-session is how one run turns into three documents nobody approved in order.
