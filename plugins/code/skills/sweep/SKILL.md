---
name: sweep
description: "Use when the user asks to 'sweep my code', 'deep review and fix', 'thorough code review with fixes', 'review and fix everything', 'clean up code', 'sweep changes', or wants a multi-phase code review that finds AND fixes issues. Runs 2 review phases with specialized agents and a fixer agent."
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Agent", "TaskCreate", "TaskUpdate", "TaskList", "AskUserQuestion"]
argument-hint: "optional: scope, files, or branch to review"
---

# Code Sweep

Run a thorough 2-phase code review using specialized reviewer agents, then fix all confirmed findings with batched fixer agents (one batch per severity, serial). Unlike `/code:review` (which only reports), sweep finds AND fixes issues. Sweep does NOT commit during the run — all fixes accumulate in the working tree and the user is prompted exactly once at the very end whether to commit.

## Determine Scope

1. If `$ARGUMENTS` specifies files or scope, use that
2. Otherwise, detect the default branch:
   ```bash
   git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo main
   ```
3. Check for branch changes: `git diff <default-branch>...HEAD --stat`
4. If branch has changes, review those. If no branch changes, fall back to unstaged changes (`git diff --stat`)
5. If no changes at all, ask user what to review using AskUserQuestion

Store TWO diff commands for reviewer prompts. Phase 1 uses the original scope; phase 2 uses a working-tree-inclusive form so reviewers can see uncommitted phase 1 fixes (since the orchestrator does not commit between phases).

- Branch changes:
  - Phase 1 diff: `git diff <default-branch>...HEAD`
  - Phase 2 diff: `git diff <default-branch>` (no `...HEAD` — includes the working tree, so uncommitted phase 1 fixes show up)
- Unstaged changes:
  - Phase 1 diff: `git diff`
  - Phase 2 diff: `git diff` (same command — `git diff` already reflects the working tree, so phase 1 fixes are picked up without needing a different form)

## Create Phase Tracking Tasks

Two top-level tasks, one per phase. Per-batch tasks are created later, after each review pass returns findings and they are bucketed by severity.

```
TaskCreate({ subject: "Sweep phase 1: comprehensive", activeForm: "Running comprehensive review (4 agents)..." })
TaskCreate({ subject: "Sweep phase 2: verification", activeForm: "Running verification review (4 agents)..." })
```

## Phase 1: Comprehensive (4 agents)

Mark phase 1 as `in_progress`. Report: "--- Phase 1: comprehensive (4 agents) ---"

1. **Spawn 4 reviewer agents in parallel** — send ALL 4 Agent tool calls in a SINGLE message:
   - `code:reviewer-correctness`
   - `code:reviewer-structure`
   - `code:reviewer-testing`
   - `code:reviewer-documentation`

   Same prompt to each (use the phase 1 diff command):
   ```
   Review code changes.

   Run `<phase-1-diff-command>` to see all changes.
   Read source files for full context — do not review from diff alone.

   Tag each finding with severity:
   - [CRITICAL] — bugs, security vulnerabilities, data loss, broken functionality, incorrect logic, missing critical error handling
   - [MAJOR]    — important issues affecting correctness, maintainability, or test coverage that should be fixed
   - [MINOR]    — style, suggestions, small improvements

   Report ALL findings as: [SEVERITY] file:line — description
   ```

2. **Collect findings** — gather ALL output from all 4 agents. Deduplicate (same file:line + same issue = merge; if severities differ on a duplicate, keep the highest). Do NOT filter, dismiss, or summarize.

3. **Print findings to user** — show the full deduplicated list grouped by severity, so the user can see what was found BEFORE any fixing happens:
   ```
   Phase 1 findings: N total (C critical, M major, m minor)

   CRITICAL (C):
   - file:line — description
   ...

   MAJOR (M):
   - file:line — description
   ...

   MINOR (m):
   - file:line — description
   ...
   ```

4. **If zero findings** → report "Phase 1: clean (0 findings)". Mark phase 1 task `completed` and proceed to phase 2.

5. **Create one task per non-empty severity batch** — skip empties. Order: critical, major, minor. Keep the returned task IDs.
   ```
   TaskCreate({ subject: "[P1] Fix C critical findings", activeForm: "Fixing C critical findings..." })
   TaskCreate({ subject: "[P1] Fix M major findings",    activeForm: "Fixing M major findings..."    })
   TaskCreate({ subject: "[P1] Fix m minor findings",    activeForm: "Fixing m minor findings..."    })
   ```

6. **Process batches serially** — one fixer at a time, in priority order (critical → major → minor). For each batch:
   - `TaskUpdate` the batch's task to `in_progress`
   - Spawn fixer with `subagent_type: "code:fixer"`. Pass the batch's findings verbatim and instruct the fixer NOT to commit:
     ```
     Fix these review findings. For each one, report whether you fixed it or dismissed it as a false positive.

     IMPORTANT: Do NOT commit and do NOT stage. The orchestrator will commit all phase fixes once at the end of the phase. You MUST still run the build and tests to validate your fixes before returning — never leave broken code behind.

     <findings for this severity, verbatim>
     ```
   - When fixer returns, print its FIXES report to the user
   - **Verify state** — run `git status --porcelain` to confirm working tree is sane, and re-read the fixer's report. If the fixer reports a failed build/tests OR returned with findings neither fixed nor dismissed, treat the batch as failed:
     - Retry ONCE with a fresh fixer subagent, passing only the unresolved findings
     - If the retry also fails, mark the batch task `completed` with a note ("partial: K unresolved"), print the unresolved findings to the user, and continue to the next batch — do NOT investigate yourself
   - `TaskUpdate` the batch's task to `completed`
   - Only then start the next batch

7. **Do NOT commit** — leave all phase 1 fixer edits in the working tree. Phase 2 will use a working-tree-inclusive diff so it can still see them. The single end-of-sweep commit prompt (see Final Report) covers everything.

   Mark phase 1 task `completed`.

## Phase 2: Verification (4 agents)

Mark phase 2 as `in_progress`. Report: "--- Phase 2: verification (4 agents) ---"

1. **Spawn 4 agents in parallel** — same 4 agents with the verification prompt (use the phase 2 diff command so reviewers see uncommitted phase 1 fixes; only critical and major):
   ```
   Review code changes.

   Run `<phase-2-diff-command>` to see all changes (this includes uncommitted fixes from phase 1).
   Read source files for full context — do not review from diff alone.

   Report ONLY critical and major issues — bugs, security vulnerabilities, data loss risks, broken functionality, incorrect logic, missing critical error handling.
   Ignore style, minor improvements, suggestions.

   Tag each finding with severity:
   - [CRITICAL]
   - [MAJOR]

   Report findings as: [SEVERITY] file:line — description
   ```

2. **Collect findings** — deduplicate same as phase 1.

3. **Print findings to user** — same format as phase 1, but only critical and major buckets.

4. **If zero findings** → report "Verification: clean (0 findings)". Mark phase 2 task `completed` and produce the final report.

5. **Create one task per non-empty severity batch** — order: critical, major.
   ```
   TaskCreate({ subject: "[P2] Fix C critical findings", activeForm: "Fixing C critical findings..." })
   TaskCreate({ subject: "[P2] Fix M major findings",    activeForm: "Fixing M major findings..."    })
   ```

6. **Process batches serially** — identical to phase 1 step 6.

7. **Do NOT commit** — leave all phase 2 fixer edits in the working tree. The end-of-sweep commit prompt below handles committing.

   Mark phase 2 task `completed`.

## Final Report and Commit Prompt

Counts come from the printed findings and per-batch tasks.

Print:
```
Sweep complete!

- Phase 1 (comprehensive): N findings (C critical, M major, m minor) → all fixes uncommitted
- Phase 2 (verification):  N findings (C critical, M major)          → all fixes uncommitted
- Total findings: N
```

Then show the current working-tree state so the user knows what `Yes` will commit:
```
git status --short
```

Then call `AskUserQuestion` with a single yes/no question:
```json
{
  "questions": [{
    "question": "Sweep finished. Commit all changes shown above now?",
    "header": "Commit",
    "multiSelect": false,
    "options": [
      {"label": "Yes", "description": "Stage and commit everything in the working tree with message 'fix: address code review findings'"},
      {"label": "No",  "description": "Leave changes uncommitted; user will commit manually"}
    ]
  }]
}
```

- If **Yes** and `git status --porcelain` is non-empty: `git add -A && git commit -m "fix: address code review findings"`
- If **Yes** and the working tree is clean (everything was dismissed): report "Nothing to commit" and skip
- If **No**: report "Changes left uncommitted"

## Key Rules

- NEVER dismiss findings as "pre-existing" or "architectural" — ALL findings are actionable
- NEVER summarize or filter agent findings — print the full deduplicated list to the user AND pass each batch's findings verbatim to its fixer
- Fixers must NOT commit or stage — no commits happen during sweep at all; the user is prompted exactly once at the very end
- Process batches serially in priority order (critical → major → minor), one fixer at a time
- One task per severity batch — mark `in_progress` when spawning the batch's fixer, `completed` when the fixer returns
- You are the ORCHESTRATOR — never read code, debug, or fix issues yourself
- If a subagent fails, retry with a fresh subagent — do NOT investigate yourself
