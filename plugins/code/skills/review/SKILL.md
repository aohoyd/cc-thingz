---
name: review
description: "Use when the user asks to 'review my code', 'check this code', 'review changes', 'code review', 'review what I wrote', 'check for bugs', or wants quality feedback on recent code changes. Launches 4 parallel reviewer agents and reports consolidated findings with confidence-based filtering."
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Agent", "TaskCreate", "TaskUpdate", "TaskList"]
---

# Code Review

Launch 4 parallel reviewer agents to review code changes, consolidate findings, and report issues. This skill identifies and reports problems — it does NOT fix them. The caller decides what to do with the results.

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

## Process

1. **Spawn 4 reviewer agents in parallel** — send ALL 4 Agent tool calls in a SINGLE message:
   - `code:reviewer-correctness`
   - `code:reviewer-structure`
   - `code:reviewer-testing`
   - `code:reviewer-documentation`

   Give each the same task prompt:
   ```
   Review code changes.

   Run `<diff-command>` to see all changes.
   Read source files for full context — do not review from diff alone.

   Report ALL findings as: file:line — description (confidence: N)
   ```

2. **Consolidate findings** — gather ALL output from all 4 agents:
   - Deduplicate: same file:line + same issue = merge
   - Filter: only keep issues with confidence >= 80
   - Sort by severity: Critical first, then Important

3. **Report results**:
   ```
   Review Complete

   Issues found: N (X critical, Y important)

   Critical:
   - [description] in file.py:42 (confidence: 95) — suggested fix: [fix]

   Important:
   - [description] in file.py:100 (confidence: 85) — suggested fix: [fix]
   ```

4. If no issues found: "Review complete: code looks good."

## Key Principles

- **Report only, never fix** — this skill identifies issues and suggests fixes but never applies them
- **Quality over quantity** — only surface issues with confidence >= 80
- **Deduplicate** — when multiple agents report the same issue, consolidate into one
- **Actionable output** — every issue includes file:line, confidence score, and a concrete fix suggestion
