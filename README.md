# Skills and agents

A set of agent skills and subagents for a coding harness. The skills form a pipeline that turns an
idea into a reviewed, merged change. Plans and task state live on a **Task board** when one is
available, so the handoff between sessions does not depend on any one machine.

## The pipeline

```
Small scope:  research → specify → implement → review → finish
Large scope:  research → implement (COARSE: waves of tasks) → review → finish
```

- **research** — investigates before deciding. Path A gathers evidence; Path B designs with you one
  section at a time. Writes no plan.
- **specify** — the only writer of a plan: the spec (what and why) and the plan (tasks, file paths,
  test steps) in one document. Attaches the plan to the Task card.
- **implement** — executes a plan. LIGHT path for one plan; COARSE path for a block of tasks split into
  waves that run in parallel worktrees.
- **review** — a structured review of a change, with fixes applied only after you choose.
- **finish** — checks CI and reviews, merges, and syncs main.

Supporting skills: **move-task** (moves a card and checks its documents are attached first),
**update-task-with-findings** (writes durable output onto a card), **add-idea-to-backlog** (records an
idea as a card instead of designing it on the spot), **check-issue** and **check-pr** (read-only
triage of an issue, and inspection and resolution of a pull request), **codebase-improvement** (audits
a codebase and produces a dated report), **systematic-debugging**, **test-driven-development** and
**trees-and-branches** (process skills the others rely on).

## Where things are kept

| Document | Kept in | Why |
|---|---|---|
| Plan (spec + tasks) | Task card artifact `plan`, scratch copy under `~/.cache/agent-plans/<feature>/` | A plan is not repo history. Nothing in git needs to hold it. |
| Status, follow-ups | Task card artifacts `status`, `follow-ups` | They are run memory, not code. |
| Research report, ADR, feature closeout | The repo (`docs/research/`, `docs/decisions/`, `docs/features/`) | These are records that belong in version control. The card stores only the path. |

**Content and location attachments.** Plans and run records are attached as full markdown, since
nothing else holds them. Repo documents are attached as a single line naming their path, so the
database does not carry a second copy of a file git already versions.

## Agents

Subagents the skills dispatch by name:

- **explore** — fast, cheap local file lookup.
- **researcher** — reads the codebase and the web to ground a plan or report. Read-only except for
  the markdown file it is told to write.
- **general** — implements one task from a plan, with its own worktree and commit.
- **reviewer** — checks specs, plans and code against the repo. Never changes anything.

## Requirements

- An agent harness that loads `SKILL.md` skills and `agents/*.md` subagents.
- For card-backed work: a Task board that exposes `list_board`, `get_task`, `get_artifact`,
  `write_artifact`, `edit_task`, `move_task` and `mark_done` (`mark_done` is the only door a card
  moves **forward** through; `move_task` is backward and sideways). Without one, a plan lives only in
  scratch on the machine where it was written, and the skills say so.
- `git` and, for PR steps, the `gh` CLI.

## Install

Link the skills and agents where your harness reads them:

```bash
git clone git@github.com:archimedes-codes/skills.git ~/.agents/kit
ln -s ~/.agents/kit/skills ~/.claude/skills        # or link individual skills
ln -s ~/.agents/kit/agents ~/.config/opencode/agents
```

## Layout

```
agents/   subagent definitions (explore, general, researcher, reviewer)
skills/   one directory per skill: SKILL.md plus its guides, checklists and templates
```

## Contributing

A skill is a directory with a `SKILL.md`. Its frontmatter has only `name` and `description`, the
name matches the directory, and the description says when to use the skill, not what it does.
