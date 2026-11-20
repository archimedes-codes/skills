---
name: check-pr
description: Use when checking a pull request for review comments, reviewer status, CI/approval status, or resolving code review feedback.
---

# Check PR

Orchestrate pull request inspection, resolution, and monitoring by delegating all execution tasks to **general subagents**. The main agent acts strictly as an orchestrator: launching subagents for investigation, immediately dispatching fixer subagents to automatically apply all suggested fixes without prompting the user, and launching subagents to watch PR and CI status until green.

---

## ⚠️ STRICT RULE: ZERO BASH IN MAIN AGENT

> **CRITICAL DISPATCH GUARD:**
> - **The main agent MUST NOT run any `gh`, `git`, or bash inspection/polling commands directly in the main session.**
> - **DO NOT run `gh pr view`, `gh api graphql`, `git commit`, `sleep 20`, or any shell commands in the main agent context.**
> - **All execution, querying, file reading, code editing, test running, and status polling MUST occur inside a `general` subagent.**
> - The main agent's sole responsibilities are: (1) dispatching subagents, (2) reading their reports, and (3) reporting the final outcome.

```
Main Agent (Strict Orchestrator - ZERO BASH, NO UNNECESSARY ASKS)
  │
  ├── 1. Launch [General Subagent: PR Checker]
  │        ├── Detects PR, queries GraphQL, inspects code context
  │        └── Returns structured inspection report with all suggested fixes
  │
  ├── 2. Launch [General Subagent: PR Fixer] (AUTONOMOUS - DO NOT ASK)
  │        ├── If actionable comments / suggested fixes exist: immediately apply them
  │        ├── Runs linters/tests, commits, pushes
  │        ├── Posts reply comments & resolves GraphQL review threads
  │        └── Returns fix summary
  │
  └── 3. Launch [General Subagent: PR Watcher]
           ├── Loops/polls internally until ALL CI & bot checks reach terminal completion
           └── Returns final PR readiness report
```

---

## Autonomous Resolution Rule: Do Not Ask

> ⚡ **AUTONOMOUS FIX POLICY:**
> If a review thread has suggested fixes, bot recommendations (e.g. CodeRabbit, Sonar), or clear reviewer requests:
> **DO NOT prompt the user with `ask`. Just apply the fixes immediately.**
> The user wants fixes executed and threads resolved autonomously without being asked for approval on each comment.

---

## Workflow Phases

### Phase 1: Launch Checking Subagent

The main agent immediately dispatches a `general` subagent. If the PR number is not already known, the subagent detects it from the current branch.

#### Dispatch Call:
```typescript
subagent({
  agent: "general",
  task: `Check the pull request for the current branch (or PR #${PR_NUMBER} if known):
1. Detect owner, repo, and PR number if not provided ('gh pr view --json number,title,url,headRefName').
2. Fetch review decision, reviewer statuses (approved/changes requested/pending), CI check rollup, and all unresolved review threads using GitHub GraphQL:
   gh api graphql -f query='query($owner: String!, $repo: String!, $pr: Int!) { repository(owner: $owner, name: $repo) { pullRequest(number: $pr) { id title url reviewDecision reviewRequests(first: 20) { nodes { requestedReviewer { ... on User { login } ... on Team { name } } } } latestReviews(first: 20) { nodes { author { login } state submittedAt } } reviewThreads(first: 50) { nodes { id isResolved isOutdated path line originalLine comments(first: 20) { nodes { id databaseId author { login } body createdAt } } } } } } }' -F owner="$OWNER" -F repo="$REPO" -F pr="$PR_NUMBER"
3. For each unresolved review thread (where isResolved == false):
   - Read the referenced file around the specified line to understand the context.
   - Extract suggested fixes (including GitHub suggestion blocks, reviewer advice, bot findings).
   - Formulate concrete code modifications to address the feedback.
4. Output a comprehensive structured report containing:
   - PR Metadata (number, title, branch, URL)
   - Reviewer Status Table (reviewers, decision, pending human requests)
   - CI / Check Runs Status Table
   - Unresolved Comments & Suggested Fixes List (thread ID, comment database ID, file, line, author, comment snippet, concrete code fix to apply)
   - Actionable item count`
})
```

---

### Phase 2: Autonomous Fix Dispatch (DO NOT ASK)

Parse the checking subagent's report:

1. **If actionable review comments or suggested fixes exist:**
   - **DO NOT call `ask()`.**
   - Proceed immediately to **Phase 3 (Launch Fixing Subagent)** with the list of fixes to apply.

2. **If zero actionable comments exist:**
   - If checks are pending/running or status needs verification, proceed directly to **Phase 4 (Launch Watching Subagent)**.
   - If PR is approved and all CI checks passed, present the status summary to the user.

---

### Phase 3: Launch Fixing Subagent

The main agent dispatches a `general` subagent to execute all suggested fixes, verify them, commit, push, reply, and resolve the threads autonomously.

#### Dispatch Call:
```typescript
subagent({
  agent: "general",
  task: `Apply all suggested review comment fixes and resolve the threads on PR #${PR_NUMBER}:

Findings to fix:
${ACTIONABLE_FINDINGS_AND_SUGGESTED_FIXES}

Instructions:
1. Apply the specified code modifications to the corresponding files.
2. Run project linters, type checks, and tests (e.g. npm test, make lint, pytest, cargo test, etc.) to verify fixes.
3. Stage modified files and commit: 'git commit -m "<ticket/scope>: address review comments"'.
4. Push changes: 'git push origin <branch>'.
5. For each fixed comment:
   - Reply via REST API: gh api "repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/comments/${COMMENT_DATABASE_ID}/replies" -f body="Fixed in commit \`\$(git rev-parse --short HEAD)\`: ${FIX_SUMMARY}"
   - Resolve thread via GraphQL mutation: gh api graphql -f query='mutation($threadId: ID!) { resolveReviewThread(input: { threadId: $threadId }) { thread { id isResolved } } }' -F threadId="${THREAD_ID}"
6. For any comments that were already resolved or clearly invalid false positives:
   - Reply with clear technical explanation and resolve the review thread.
7. Return a summary of applied fixes, commit SHA, test results, and resolved threads.`
})
```

---

### Phase 4: Launch Watching Subagent

Once fixes are pushed (or if checking revealed pending checks), dispatch a `general` subagent to monitor CI runs, bot re-reviews, and mergeability until terminal status.

> ⚠️ **WATCHER REQUIREMENT:** The watcher subagent MUST NOT return prematurely while checks are in progress. It must actively poll until all checks reach terminal status (`COMPLETED`, `SUCCESS`, `FAILURE`).

#### Dispatch Call:
```typescript
subagent({
  agent: "general",
  task: `Watch the status of PR #${PR_NUMBER} on repo ${OWNER}/${REPO} until ALL checks and reviews complete:
1. Poll the PR status checks every 20 seconds until NO checks are in progress or queued:
   while true; do
     PENDING=$(gh pr view ${PR_NUMBER} --json statusCheckRollup --jq '[.statusCheckRollup[]? | select(.status == "IN_PROGRESS" or .status == "QUEUED" or .status == "PENDING" or .state == "PENDING")] | length')
     if [ "$PENDING" -eq 0 ]; then
       break
     fi
     echo "Waiting for $PENDING pending check(s)..."
     sleep 20
   done
2. Once checks complete, inspect if any checks failed. If failed, fetch logs with 'gh run view <run-id> --log-failed'.
3. Check for any newly submitted review comments or threads (e.g. from review bots or human reviewers) using GraphQL.
4. Fetch final review decision and mergeability ('gh pr view ${PR_NUMBER} --json reviewDecision,latestReviews,mergeable').
5. Return a complete status report:
   - Final CI check status (all passed vs specific failures)
   - Reviewer approval status (APPROVED / CHANGES_REQUESTED / REVIEW_REQUIRED)
   - Any new unresolved comments created by bots/reviewers
   - Final ready-to-merge readiness assessment`
})
```

---

## Summary & Reporting

After the subagents finish:
- The main agent presents the final consolidated status to the user (list of fixes applied, commits pushed, resolved threads, and CI status).
- If the PR is ready to merge, recommend or invoke `finish`.
- If new CI failures or new review comments appeared during the watch:
  - If new suggested fixes exist, loop back to Phase 3 autonomously.
  - If unexpected test/build errors require developer architectural decisions, present the failure logs and diagnosis.
