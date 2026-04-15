---
name: sweep
description: "Use when the user asks to 'sweep my code', 'deep review and fix', 'thorough code review with fixes', 'review and fix everything', 'clean up code', 'sweep changes', or wants a multi-phase code review that finds AND fixes issues. Runs 3 review phases with specialized agents and a fixer agent."
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Agent", "TaskCreate", "TaskUpdate", "TaskList"]
argument-hint: "optional: scope, files, or branch to review"
---

# Code Sweep

Run a thorough 3-phase code review using specialized reviewer agents, then fix all confirmed findings with the fixer agent. Unlike `/code:review` (which only reports), sweep finds AND fixes issues.

## Determine Scope

1. If `$ARGUMENTS` specifies files or scope, use that
2. Otherwise, detect the default branch:
   ```bash
   git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo main
   ```
3. Check for branch changes: `git diff <default-branch>...HEAD --stat`
4. If branch has changes, review those. If no branch changes, fall back to unstaged changes (`git diff --stat`)
5. If no changes at all, ask user what to review using AskUserQuestion

Store the diff command for reviewer prompts:
- Branch changes: `git diff <default-branch>...HEAD`
- Unstaged changes: `git diff`

## Create Tracking Tasks

```
TaskCreate({ subject: "Sweep phase 1: comprehensive", activeForm: "Running comprehensive review (5 agents)..." })
TaskCreate({ subject: "Sweep phase 2: smells", activeForm: "Running smells review..." })
TaskCreate({ subject: "Sweep phase 3: critical", activeForm: "Running critical-only review..." })
```

## Phase 1: Comprehensive (5 agents)

Mark phase 1 as `in_progress`. Report: "--- Phase 1: comprehensive (5 agents) ---"

Loop up to 3 iterations:

1. **Spawn 5 reviewer agents in parallel** — send ALL 5 Agent tool calls in a SINGLE message:
   - `code:reviewer-quality`
   - `code:reviewer-implementation`
   - `code:reviewer-testing`
   - `code:reviewer-simplification`
   - `code:reviewer-documentation`

   Give each the same task prompt:
   ```
   Review code changes.

   Run `<diff-command>` to see all changes.
   Read source files for full context — do not review from diff alone.

   Report ALL findings as: file:line — description
   ```

2. **Collect findings** — gather ALL output from all 5 agents. Deduplicate (same file:line + same issue = merge). Do NOT filter, dismiss, or summarize.

3. **If ALL agents reported zero issues** → report "Phase 1: clean" and proceed to phase 2. Mark as `completed`.

4. **Spawn fixer** — use Agent tool with `subagent_type: "code:fixer"`. Pass the FULL unedited findings:
   ```
   Fix these review findings:

   <all findings verbatim>
   ```

5. **After fixer returns** → show the FIXES section to user. Loop back to step 1.

If 3 iterations reached with issues still found, report "Phase 1: max iterations reached, moving on". Mark as `completed`.

## Phase 2: Smells (1 agent)

Mark phase 2 as `in_progress`. Report: "--- Phase 2: code smells ---"

Single pass (no loop):

1. **Spawn** `code:reviewer-smells` with the same review prompt as phase 1
2. **If no issues** → report "Smells: clean" and proceed
3. **If issues found** → spawn `code:fixer` with findings. Report fixes.

Mark phase 2 as `completed`.

## Phase 3: Critical Only (2 agents)

Mark phase 3 as `in_progress`. Report: "--- Phase 3: critical only (2 agents) ---"

Single pass (no loop):

1. **Spawn 2 agents in parallel** — `code:reviewer-quality` and `code:reviewer-implementation` with modified prompt:
   ```
   Review code changes.

   Run `<diff-command>` to see all changes.
   Read source files for full context — do not review from diff alone.

   Report ONLY critical and major issues — bugs, security vulnerabilities, data loss risks, broken functionality, incorrect logic, missing critical error handling.
   Ignore style, minor improvements, suggestions.

   Report findings as: file:line — description
   ```

2. **If no issues** → report "Critical: clean"
3. **If issues found** → spawn `code:fixer` with findings. Report fixes.

Mark phase 3 as `completed`.

## Final Report

```
Sweep complete!

- Phase 1 (comprehensive): N iterations, N fixes
- Phase 2 (smells): N fixes
- Phase 3 (critical): N fixes
- Total fixes applied: N
```

## Key Rules

- NEVER dismiss findings as "pre-existing" or "architectural" — ALL findings are actionable
- NEVER summarize or filter agent findings — pass full output to fixer verbatim
- You are the ORCHESTRATOR — never read code, debug, or fix issues yourself
- If a subagent fails, retry with fresh subagent — do NOT investigate yourself
