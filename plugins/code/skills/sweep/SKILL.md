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

## Gather Decision Context

Before spawning any reviewer, assemble a **Context block** that every reviewer AND fixer prompt will carry verbatim:

1. **Design/plan docs**: if the changes came from a plan (`docs/plans/*.md` with a `Design:` header) or a design doc, list the path(s) and extract the explicitly decided behaviors (e.g. "hard close on cascade — deliberate, soft-close does not cascade").
2. **User decisions from this session**: choices made via AskUserQuestion, corrections the user issued, changes the user explicitly requested mid-session.
3. **Environment constraints**: anything learned this session a fresh subagent can't know (user's shell, deployment mode, etc.).

Format:
```
CONTEXT (decisions already made — do not re-litigate; report conflicts as design conflicts, not defects):
- Design doc: <path> (key decisions: ...)
- User decisions this session: ...
- Constraints: ...
```

If there is genuinely no context (standalone sweep, no plan, no session history), say so in the block ("No prior design context") rather than omitting it. Evidence from past runs: reviewers without this block re-litigated user-approved decisions at MAJOR severity, and a fixer reversed a design-mandated behavior it had no way to know about.

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

   Same prompt to each (use the phase 1 diff command, and include the Context block assembled above):
   ```
   Review code changes.

   <CONTEXT block>

   Run `<phase-1-diff-command>` to see all changes.
   Read source files for full context — do not review from diff alone.
   Do NOT run the project's full build or test suite — the fixers own validation. Never run interactive/UI-driving tests.

   Tag each finding with severity:
   - [CRITICAL] — bugs, security vulnerabilities, data loss, broken functionality, incorrect logic, missing critical error handling
   - [MAJOR]    — issues that change runtime behavior or correctness and should be fixed
   - [MINOR]    — missing tests, doc/comment drift, style, suggestions, small improvements

   Severity discipline: MAJOR requires a behavior/correctness impact. Missing tests and documentation staleness are MINOR — never inflate them.

   Report ALL findings as: [SEVERITY] file:line — description
   Findings that conflict with a decision in the CONTEXT block: report separately as [DESIGN-CONFLICT], not as defects.
   Pre-existing issues (not introduced by this diff): separate PRE-EXISTING section, not mixed into findings.
   ```

2. **Collect findings** — gather ALL output from all 4 agents. Deduplicate (same file:line + same issue = merge; if severities differ on a duplicate, keep the highest). Do NOT filter, dismiss, or summarize. Set aside `[DESIGN-CONFLICT]` and `PRE-EXISTING` items — they do NOT go to fixers:
   - **Design conflicts**: after printing findings, ask the user via AskUserQuestion whether to change the approved behavior or dismiss the finding. Never let a fixer resolve a design conflict.
   - **Pre-existing**: list them informationally at the end of the sweep report; fix only if the user asks.

3. **Print findings to user** — this MUST be a visible text message in your reply (not thinking, not only task metadata). Show the full deduplicated list grouped by severity, so the user can see what was found BEFORE any fixing happens:
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
   - **Safety snapshot** — before spawning the fixer, snapshot the working tree so agent mistakes can't destroy uncommitted work (this records a dangling stash object WITHOUT touching the working tree or history):
     ```bash
     git add -A && SNAP=$(git stash create "sweep safety snapshot") && git reset -q
     echo "snapshot: ${SNAP:-tree clean}"
     ```
     Print the snapshot SHA to the user once per batch ("Safety snapshot: <sha> — recover with `git stash apply <sha>`"). The `git add -A` is only there so untracked new files are captured; `git reset -q` restores the index without touching files.
   - `TaskUpdate` the batch's task to `in_progress`
   - Spawn fixer with `subagent_type: "code:fixer"`. Pass the batch's findings verbatim, the Context block, and instruct the fixer NOT to commit:
     ```
     Fix these review findings. For each one, report whether you fixed it or dismissed it as a false positive.

     <CONTEXT block>

     IMPORTANT: Do NOT commit and do NOT stage. The orchestrator will commit all phase fixes once at the end of the phase. You MUST still run the build and tests to validate your fixes before returning — never leave broken code behind. Bound long validation: never run interactive/UI tests (compile-only validation for those targets), and if a build/test run risks exceeding ~10 minutes, use a narrower target and report the limitation instead of hanging.

     <findings for this severity, verbatim>
     ```
   - When fixer returns, print its FIXES report to the user (visible text, not thinking)
   - **Verify state** — run `git status --porcelain` to confirm working tree is sane, and re-read the fixer's report. If the fixer reports a failed build/tests OR returned with findings neither fixed nor dismissed, treat the batch as failed:
     - Retry ONCE with a fresh fixer subagent, passing only the unresolved findings
     - If the retry also fails, mark the batch task `completed` with a note ("partial: K unresolved"), print the unresolved findings to the user, and continue to the next batch — do NOT investigate yourself
   - `TaskUpdate` the batch's task to `completed`
   - Only then start the next batch

7. **Do NOT commit** — leave all phase 1 fixer edits in the working tree. Phase 2 will use a working-tree-inclusive diff so it can still see them. The single end-of-sweep commit prompt (see Final Report) covers everything.

   Mark phase 1 task `completed`.

## Phase 2: Verification (4 agents)

Mark phase 2 as `in_progress`. Report: "--- Phase 2: verification (4 agents) ---"

1. **Spawn 4 agents in parallel** — same 4 agents with the verification prompt (use the phase 2 diff command so reviewers see uncommitted phase 1 fixes; only critical and major). Collect the list of files phase 1 fixers actually modified (from their FIXES reports / `git status`) and pass it — evidence from past runs shows phase 2's real findings are overwhelmingly regressions introduced by phase 1 fixes, so that's where verification effort goes:
   ```
   Verification review after a round of fixes.

   <CONTEXT block>

   Run `<phase-2-diff-command>` to see all changes (this includes uncommitted fixes from phase 1).
   PRIORITY: the phase 1 fixers modified these files: <list>. Scrutinize those changes first — fixer-introduced regressions are what this phase exists to catch. Widen to the rest of the diff only after covering them.
   Read source files for full context — do not review from diff alone.
   Do NOT run the project's full build or test suite. Never run interactive/UI-driving tests.

   Report ONLY critical and major issues — bugs, security vulnerabilities, data loss risks, broken functionality, incorrect logic, missing critical error handling.
   Ignore style, minor improvements, suggestions, missing tests, doc drift.

   Tag each finding with severity:
   - [CRITICAL]
   - [MAJOR]

   Report findings as: [SEVERITY] file:line — description
   Findings that conflict with a decision in the CONTEXT block: report as [DESIGN-CONFLICT], not as defects.
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

Counts come from the printed findings and per-batch tasks — recount from the actual printed lists, don't trust running totals.

Print (visible text message, not thinking):
```
Sweep complete!

- Phase 1 (comprehensive): N findings (C critical, M major, m minor) → all fixes uncommitted
- Phase 2 (verification):  N findings (C critical, M major)          → all fixes uncommitted
- Total findings: N
- Pre-existing issues noted (not fixed): K — <one line each>
- Runtime verification: <what was actually run/exercised, or "NOT performed — all gates were static (build/tests). For UI or user-facing changes, run the app before merging.">
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

- NEVER summarize or filter agent findings — print the full deduplicated list to the user AND pass each batch's findings verbatim to its fixer. Reviewer-tagged `PRE-EXISTING` and `[DESIGN-CONFLICT]` items are the only exceptions: pre-existing is reported informationally, design conflicts go to the user via AskUserQuestion — neither goes to a fixer
- All mandated reports (phase banners, findings lists, FIXES reports, final summary) MUST be visible text messages to the user — content that exists only in thinking blocks was never reported
- Fixers must NOT commit or stage — no commits happen during sweep at all; the user is prompted exactly once at the very end
- Take a safety snapshot before every fixer batch (see phase 1 step 6) — the uncommitted tree may be the only copy of days of work
- NEVER let any agent run working-tree-reverting git commands (`checkout -- <paths>`, `restore`, `stash`, `reset`, `clean`) or interactive/UI-driving tests
- Process batches serially in priority order (critical → major → minor), one fixer at a time
- One task per severity batch — mark `in_progress` when spawning the batch's fixer, `completed` when the fixer returns
- You are the ORCHESTRATOR — never read code, debug, or fix issues yourself
- If a subagent fails or stalls, retry with a fresh subagent scoped to the unresolved findings — do NOT investigate yourself
- A failing test at the end of the sweep is a finding, not an annoyance — never frame "drop the failing test" as the default resolution; a sweep-written test that keeps failing has caught a real bug before
