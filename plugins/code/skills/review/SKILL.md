---
name: review
description: "Use when the user asks to 'review my code', 'check this code', 'review changes', 'code review', 'review what I wrote', 'check for bugs', or wants quality feedback on recent code changes. Launches 4 parallel reviewer agents and reports consolidated findings with confidence-based filtering."
allowed-tools: ["Read", "Write", "Glob", "Grep", "Bash", "Agent", "AskUserQuestion"]
argument-hint: "optional scope: files, directories, or a branch/commit range"
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
- Custom scope from `$ARGUMENTS`: the equivalent `git diff <target> [-- <paths>]` — reuse it verbatim for both the reviewer prompts and the hunk reload

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

   Tag each finding with severity:
   - [CRITICAL] — bugs, security vulnerabilities, data loss, broken functionality
   - [IMPORTANT] — issues affecting correctness, maintainability, or test coverage

   Report ALL findings as: [SEVERITY] file:line — description — suggested fix: <fix> (confidence: N)
   Confidence is 0-100: how certain you are this is a real issue you verified in the source, not just the diff.
   ```

2. **Consolidate findings** — gather ALL output from all 4 agents:
   - Deduplicate: same file:line + same issue = merge
   - Filter: only keep issues with confidence >= 80
   - Sort by severity: Critical first, then Important

3. **Sync a live Hunk session** (optional — skip silently when hunk or a session is unavailable; if a command errors mid-way, continue and note it). Run this even with zero findings, so the session never shows stale results:
   1. Detect: run `command -v hunk && hunk session get --repo "$(git rev-parse --show-toplevel)" --json`. If either fails (hunk not installed, no live session for this repo), skip the rest of this step and go to **Report results**.
   2. Reload the session to show the reviewed diff so comment line numbers line up:
      - Branch scope: `hunk session reload --repo <root> -- diff <default-branch>...HEAD`
      - Unstaged scope: `hunk session reload --repo <root> -- diff`
      - Custom scope from `$ARGUMENTS`: append the pathspec, e.g. `hunk session reload --repo <root> -- diff <target> -- <paths>`
   3. Clear agent comments from previous review runs: `hunk session comment clear --repo <root> --yes`. This removes agent notes only — NEVER pass `--include-user` or `--all`, which would destroy the user's own notes.
   4. If there are findings, run `hunk session review --repo <root> --json` and keep only findings whose file appears in the loaded diff (the **diff filter**). `comment apply` validates the whole batch before mutating, so one out-of-view comment rejects all of them. Count the dropped findings for the report.
   5. Write the batch JSON to a temp file OUTSIDE the repo (e.g. `/tmp/code-review-hunk-batch.json`) with the Write tool — never inside the working tree, where it would pollute the reviewed diff. Finding text often contains quotes — do not inline JSON in shell. Apply as ONE batch, then delete the temp file:
      ```bash
      hunk session comment apply --repo <root> --stdin < /tmp/code-review-hunk-batch.json
      ```
      Map each finding to: `{"filePath": "<repo-relative path>", "newLine": <line>, "summary": "[SEVERITY] <description>", "rationale": "Suggested fix: <fix> (confidence: N)", "author": "code:review"}`. Use `"oldLine"` instead of `"newLine"` for findings anchored to removed lines.
   6. If `comment apply` rejects the batch (a finding can target a line outside the loaded hunks even in a file the diff filter kept), fall back to adding comments one at a time with `hunk session comment add`, skipping individual failures. Count the skipped ones for the report.
   7. If any hunk command errors or a flag/mapping is unclear, read `${CLAUDE_PLUGIN_ROOT}/skills/review/references/hunk-review.md` for the full CLI reference and error catalog — use it as a command reference only; this skill's steps override its review-guiding workflow. The whole step is best-effort: on unrecoverable errors, continue to **Report results** and note that mirroring was skipped and why. Never fail or block the review over hunk.

4. **Report results**:
   ```
   Review Complete

   Issues found: N (X critical, Y important)

   Critical:
   - [description] in file.py:42 (confidence: 95) — suggested fix: [fix]

   Important:
   - [description] in file.py:100 (confidence: 85) — suggested fix: [fix]
   ```
   If findings were mirrored to Hunk, append: `Mirrored N inline comments to the live Hunk session.` — add `(M skipped: outside the loaded diff)` when the diff filter or the per-comment fallback dropped any.

5. If no issues found: "Review complete: code looks good."

## Key Principles

- **Report only, never fix** — this skill identifies issues and suggests fixes but never applies them
- **Hunk is additive** — the screen report is the canonical output; inline Hunk comments are a mirror for users with a live session, and their absence never changes the review
- **Quality over quantity** — only surface issues with confidence >= 80
- **Deduplicate** — when multiple agents report the same issue, consolidate into one
- **Actionable output** — every issue includes file:line, confidence score, and a concrete fix suggestion
