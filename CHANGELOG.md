# Changelog

This repo ships independent Claude Code plugins. Version headings use values from `plugins/<name>/.claude-plugin/plugin.json`; they are not git tags.

Entries are sorted by plugin version date, newest first.

## planning v4.5.0 - 2026-07-27

### Improvements

- make: no longer asks "TDD or regular?" — Claude picks the testing approach per task type (TDD for behavior specifiable up front: reproducible bug fixes, pure logic, parsers, contracts; regular for work whose shape emerges: UI, wiring, exploratory refactors) and records the choice with a one-line rationale in the plan. Mixing formats across tasks is allowed; a user-stated preference always overrides. With the question gone, the design-doc path often has nothing left to ask and skips the interview entirely

## brainstorm v4.3.0 - 2026-07-27

### Improvements

- do: prompt slimmed for Claude 5-generation models (Opus 5 / Fable 5), which follow instructions literally and lose quality under over-prescriptive scaffolding — per [Anthropic's context-engineering guidance](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models). 179 → 97 lines with no behavioral constraint removed:
  - dropped the `<SELF-CHECK>` per-response checklist (a verification-scaffolding pattern that now causes over-verification); its two load-bearing items — visible content before any approval, free-text corrections are final — are promoted to top-level rules
  - dropped the WRONG vs RIGHT example gallery; the one transcript-earned failure it guarded (approving a section the user never saw) is folded into the visible-approval rule
  - deduplicated: the AskUserQuestion-not-plain-text rule was stated four times, now once; Key Principles keeps only items not already covered by rules or phases
  - replaced "MANDATORY, NO EXCEPTIONS" caps-emphasis with plain declarative rules

## planning v4.4.0 - 2026-07-24

### Improvements

- execute: the auto plan-review's NEEDS REVISION path no longer asks apply / execute as-is / stop — the priority fixes are applied to the plan automatically and execution continues (interrupt if a fix is wrong)

## planning v4.3.0 - 2026-07-23

### Improvements

- plan-review: drops its hardcoded `opus` pin for `inherit` — the alias resolves to Opus 4.8, which now sits below the Claude 5 family, so the pin silently downgraded plan review in sessions running newer models
- task-executor: moves from `inherit` to `opus` for consistent implementation quality regardless of the session model

## code v2.6.0 - 2026-07-23

### Improvements

- architect + reviewer-correctness: switch to `inherit` so the highest-judgment work (design blueprints, bug/security review) rides the session model instead of a hardcoded pin that goes stale as new model families ship
- explorer, reviewer-structure, reviewer-testing, reviewer-documentation: move from `sonnet` to `opus` (`fixer` already ran on `opus`)

The three releases below (planning v4.2.0, code v2.5.0, brainstorm v4.2.0) come from a transcript audit of the last 31 real sessions using these plugins — every change in them traces to an observed failure or friction point.

## planning v4.2.0 - 2026-07-23

### Bug Fixes

- make: the step-3 "Plan created. What's next?" menu listed 5 options, but AskUserQuestion caps options at 4 — every run threw a visible `InputValidationError` and the ad-hoc retry silently dropped a different option each time (usually "Auto review", which made the plan-review agent nearly unreachable). Reproduced in 8+ audited sessions. The menu is now 4 options with plan review folded into the execute path

### New Features

- execute: auto-runs the `plan-review` agent before the first task (it delivered verified, low-noise findings in all 6 audited runs). APPROVE → one-line note and continue; NEEDS REVISION → priority fixes are shown and the user chooses apply / execute as-is / stop
- make + execute: plans record a `Base:` header (`branch@sha`), and execute verifies that commit is an ancestor of HEAD before running — catches worktrees created from a stale `origin/<default>` (EnterWorktree's `fresh` default), which previously cost a mid-pipeline rebase once and ~250K tokens of discarded work another time

### Improvements

- make: interview questions are now batched into one AskUserQuestion call and topics already settled by a design doc or earlier session answers are skipped; the plan title is derived from the design filename instead of asked; codebase facts asserted in questions/plans must be verified first (a plan once claimed "no test framework" in a repo with a gtest harness)
- make: bug-fix plans must attempt reproduction first when feasible, or mark the diagnosis UNVERIFIED; the verify-acceptance template task gains a runtime-verification item (reproduce the original symptom / run the app) — an audited pipeline shipped a 1301-line "fix" for a hang it never reproduced, and the hang remained
- execute: every task-executor prompt carries a session-constraints block (environment quirks like fish shell, user decisions, mid-session corrections) — an executor once reintroduced a bash-arithmetic-under-fish bug the user had already corrected in the same session
- execute: manual/interactive tasks (visual checks, tests that synthesize keyboard/mouse events) are negotiated with the user instead of blindly delegated; adjacent trivial doc/one-line tasks may share one executor spawn instead of paying ~30K tokens of spin-up each
- task-executor: stops and reports when the codebase materially contradicts the plan's assumptions instead of improvising a refactor; treats prompt constraints as binding

## code v2.5.0 - 2026-07-23

### Bug Fixes

- review command: `commands/review.md` shadowed `skills/review/SKILL.md` for the name `code:review`, so invoking the skill loaded only the 3-line wrapper and the real workflow (hunk sync, confidence filter, prompt template) never entered context. The command now instructs reading the SKILL.md file directly
- fixer: agent definition ordered a commit (Step 4) while sweep forbids fixer commits — every sweep prompt had to override the agent's own spec. The fixer now never commits or stages unless explicitly instructed

### New Features

- sweep: decision-context block — the orchestrator collects design-doc decisions, user choices from the session, and environment constraints, and passes them to every reviewer and fixer. Findings that contradict an approved decision are reported as `[DESIGN-CONFLICT]` and escalated to the user, never auto-fixed (a fixer once reversed a design-mandated close behavior it had no way to know about; reviewers re-litigated user-approved decisions at MAJOR severity)
- sweep: safety snapshot before every fixer batch (`git add -A && git stash create && git reset -q` — dangling object, working tree untouched, SHA printed) so agent mistakes can't destroy uncommitted work — a fixer once ran `git checkout HEAD -- <files>` chasing a rustfmt nit and permanently destroyed uncommitted tests
- sweep: final report discloses runtime verification status — when all gates were static (build/tests), it says so and recommends running the app before merging UI changes; audited sweeps shipped a keyboard-unreachable panel and a click-closes-the-app bug past 8 reviewer runs

### Improvements

- all reviewer agents: hardened rules — read-only now explicitly covers "temporary" edits; interactive/UI-driving tests are banned (one hijacked the user's screen mid-typing); full build/test-suite runs are banned (up to 11 redundant full validation cycles per sweep observed — fixers own the gates); provenance must be verified so pre-existing issues land in a separate `PRE-EXISTING` section instead of being attributed to the branch
- sweep: severity discipline — MAJOR requires behavior/correctness impact; missing tests and doc drift are MINOR (an audited sweep had 7 "MAJOR" findings and zero behavior bugs, inflating fixer batches)
- sweep: phase 2 reviewers prioritize the files phase 1 fixers modified — audited phase-2 findings were overwhelmingly fixer-introduced regressions (2/4, 2/2, 2/2 in three sessions)
- sweep: mandated reports (findings lists, FIXES reports, final summary) must be visible text messages — on some models they sank into thinking blocks and the user saw nothing for 54 minutes; final counts are recomputed from the printed lists after two sessions reported wrong arithmetic
- sweep: a failing test at sweep end is a finding, not an annoyance — "drop the failing test" framing once nearly buried a real product bug the user had to rescue
- fixer: destructive-git ban (`checkout -- <paths>`, `restore`, `stash`, `reset`, `clean`), bounded validation (no interactive tests, ~10-minute ceiling with narrower targets instead of hangs — one fixer stalled 18 minutes on an XCUITest build), no speculative "preventive" fixes, no unverified enumerative doc claims (a wrong version bound written by a fixer became the next phase's CRITICAL), full repo-relative paths in reports, and a `design conflict` report category
- reviewer-testing: checks that demanded tests are actually observable/testable in the project's harness before flagging (dismissal-rate driver in audited sweeps)
- reviewer-documentation: stays in its lane (no code-provenance/merge-regression claims), stale counts are MINOR, and "corrected" enumerations must be derived from the source of truth (a reviewer's proposed count fix was itself arithmetically wrong)
- review: triggers extended ("review current branch", "make a code review"); orchestrator must independently verify each CRITICAL finding before reporting; per-specialty focus lines allowed on the shared prompt; reviewer `PRE-EXISTING` items get their own report section
- explorer: gained Bash (read-only usage — git diff/show/log/blame) — it previously had to caveat reports as "best-effort inference" because it couldn't run git at all

## brainstorm v4.2.0 - 2026-07-23

### Bug Fixes

- do: Phase 4 design sections must be presented as visible message text before the approval question — on some models sections landed only in thinking blocks, and users blind-approved designs they never saw ("you showed nothing", observed 4 rounds in a row). Rule 1 now states the tool call follows visible text, never replaces it; the section must ALSO be mirrored into the "Looks good" option's `preview` field as a backup; the self-check enforces both, and the failure is a named WRONG/RIGHT example

### Improvements

- do: anchor on the user's words — Phase 1 restates the request in the user's own terms mapped to concrete code elements, and a free-text correction is final (never re-ask a question built on rejected framing); anchoring on code over user intent was the #1 source of pushback in audited sessions
- do: bug reports get reproduced before designing (or the user is asked for repro steps), and static-only diagnoses prefer runtime verification before committing to a fix — two audited sessions designed confident fixes for wrong root causes
- do: investigation requests gather measurements before scoping/tooling questions (asking first caused a full abandonment)
- do: concrete facts in options (key chords, command names) must be grep-verified — a hallucinated "⌘K palette" once propagated into five doc surfaces
- do: Phase 3 is explicitly optional — skip honestly with a "settled in Phase 2" note instead of staging fake comparisons (it was skipped-but-marked-completed in most audited sessions; when forced, it produced over-engineered machinery); approaches are sanity-checked against Phase 1 findings
- do: load-bearing UX choices (keybindings, removals, narrow-width behavior) must be surfaced as their own questions, never buried in bulk section approvals — a buried width-guard decision shipped three visible regressions; designs that depend on the user's runtime environment (theme, install mode) ask about it
- do: `code:explorer` subagents are for large/unfamiliar areas; small or well-understood areas are explored directly

## code v2.4.0 - 2026-07-22

### New Features

- review: mirror consolidated findings into a live Hunk session as inline comments when the `hunk` CLI is installed and a session is open for the reviewed repo. The session is reloaded to the reviewed diff so line numbers match, stale agent comments from previous runs are cleared, and all findings are pushed in one `comment apply` batch (summary = severity + description, rationale = suggested fix + confidence). The screen report stays canonical; the step skips silently when hunk or a session is absent. Hunk's bundled session-CLI skill is vendored at `skills/review/references/hunk-review.md` (from hunk v0.17.3)

### Improvements

- review: reviewer prompt now requires severity tags (`[CRITICAL]`/`[IMPORTANT]`), a suggested fix, and a calibrated 0-100 confidence in each finding — consolidation and the Hunk comment summaries previously consumed severity that no agent was asked to produce
- review: Hunk mirroring step hardened — bail-out paths use named targets instead of ambiguous step numbers, the batch JSON temp file must live outside the repo and is deleted after apply (a report-only skill never pollutes the reviewed diff), batch rejection falls back to per-comment `comment add`, the vendored CLI reference is read only on errors and marked command-reference-only, and a custom `$ARGUMENTS` scope now gets a stored diff command
- review: added `argument-hint`, dropped unused task tools from `allowed-tools` (skill and command), renamed the command's subagent tool `Task` → `Agent` to match the skills
- reviewer-structure: over-engineering section rebuilt around ponytail's ladder, rules, and constraints — a finding is code sitting lower on the ladder than it could (should it exist at all → stdlib → native platform feature → already-installed dependency → shorter form). Rules flag unrequested abstractions, pass-through layers, and unnecessary fallbacks, and treat `ponytail:` comments as deliberate intent rather than findings. Constraints keep smoke tests, trust-boundary validation, security/accessibility code, explicitly requested features, and hardware calibration knobs off the delete list. Every finding names its concrete replacement and net line change. Benefits both `/code:review` and `/code:sweep`, which share the agent

## planning v3.8.1 - 2026-06-29

### Bug Fixes

- plan review: add `PLANNING_DISABLE_REVDIFF=1` to skip interactive plan review entirely on both routes (the `ExitPlanMode` hook and `/planning:make`). Under `claude /remote-control` the overlay opened on the host terminal the remote client cannot see, blocking the session indefinitely; the flag bypasses both revdiff and the `$EDITOR` fallback and falls through to the normal `ExitPlanMode` confirmation #32

## planning v3.8.0 - 2026-06-28

### New Features

- plan-review overlay: add a `herdr` terminal backend to `launch-plan-review.sh`. Opens revdiff in a new fullscreen tab via the herdr CLI (`tab create` / `pane run` / `tab close`), blocking on a sentinel file until the overlay closes, so `/planning:make` interactive review and the `ExitPlanMode` hook work inside herdr sessions. #31
- plan-review overlay: add an `agterm` terminal backend to `launch-plan-review.sh`. Opens revdiff in a full-pane overlay via `agtermctl session overlay open --block` and toggles the session status indicator to blocked while the overlay is up, restoring active on exit.

## planning v3.7.8 - 2026-06-23

### Bug Fixes

- make: instruct the plan template to renumber the two trailing tasks (verify acceptance criteria, update documentation) with concrete sequential integers. They were shown as literal "Task N-1" and "Task N" placeholders with no substitution rule, so generated plans transcribed the letter `N` verbatim instead of continuing the task numbering

## thinking-tools v1.2.2 - 2026-06-22

### Improvements

- ask-codex: add a memory-load preamble so Codex reads Claude's memory files (`CLAUDE.md`, `CLAUDE.local.md`, `.claude/rules/`, `~/.claude/CLAUDE.md`); Codex only auto-loads `AGENTS.md` #30 @alexkart
- ask-codex: raise default `model_reasoning_effort` to `xhigh` and align the intro wording with `gpt-5.5` #30 @alexkart

### Bug Fixes

- ask-codex: drop the dead `-c project_doc=...` overrides. `project_doc` is not a valid Codex config key, so they loaded nothing #30 @alexkart
- ask-codex: redirect stdin from `/dev/null` so `codex exec` no longer hangs on "Reading additional input from stdin…" on fresh installs (#26) #30 @alexkart

## planning v3.7.7 - 2026-06-22

### Bug Fixes

- exec: drop the dead `-c project_doc=...` overrides from `run-codex.sh`. `project_doc` is not a valid Codex config key, so the codex review pass loaded nothing #30 @alexkart

## planning v3.7.6 - 2026-06-09

### Bug Fixes

- exec: report the plan move honestly. Step 13 hardcoded "plan moved to completed/" in the final line even though the move is best-effort, so a no-op (plan already under `completed/` or missing) or a failed move would print a false claim. The suffix is now appended only when `move-plan.sh` actually moved the file.
- exec: `move-plan.sh` refuses to overwrite an existing destination instead of clobbering it. A same-named plan already under `completed/` now causes a non-zero exit (reported, non-blocking) rather than a silent `mv` over the existing file.

## planning v3.7.5 - 2026-06-09

### Bug Fixes

- exec: move the finished plan into `docs/plans/completed/` at completion. The plan's final "move to completed/" checkbox was marked `[x]` by a task subagent but the file never moved (the orchestrator explicitly refused, and a mid-run move would break every later phase's `PLAN_FILE_PATH`). Step 13 now performs the move via a VCS-aware `move-plan.sh` (git/hg), committing without pushing, so finished plans leave `docs/plans/` and stop re-appearing as `/planning:exec` candidates.
- exec: forbid task subagents from moving/renaming the plan file. A subagent could interpret the "move to completed/" checkbox as an automatable `git mv` and abort the run when the orchestrator's `PLAN_FILE_PATH` re-read failed; the task prompt now marks such a checkbox `[x]` and leaves the move to the harness.

## planning v3.7.4 - 2026-06-02

### Improvements

- make the plan-review overlay popup size configurable via `REVDIFF_POPUP_WIDTH` / `REVDIFF_POPUP_HEIGHT` env vars, defaulting to 90% #27 @aldobrynin

### Bug Fixes

- pass `90%` (not 90 cells) to zellij for the plan-review overlay #27 @aldobrynin

## planning v3.7.3 - 2026-06-01

### Bug Fixes

- exec: enforce one-task-at-a-time in the task loop. Step 6 described a sequential loop but never forbade batch-spawning, so an autonomous run could fan out all remaining tasks in parallel — corrupting the shared plan file and working tree. Added an explicit guard that the parallel-fanout instruction applies only to the review phases.

## planning v3.7.2 - 2026-05-30

### Bug Fixes

- redirect codex stdin from `/dev/null` in `run-codex.sh` so the external review step does not hang when launched with an inherited open stdin (e.g. background tasks); `codex exec` reads stdin to append a `<stdin>` block even when a prompt arg is given

## thinking-tools v1.2.1 - 2026-05-18

### Improvements

- bump `ask-codex` default Codex model to `gpt-5.5` #22 @fitz123

## release-tools v2.0.2 - 2026-05-18

### Improvements

- replace Git-specific wording with generic repository wording #11 @paskal

## workflow v1.1.0 - 2026-05-16

### New Features

- route learn discoveries to `CLAUDE.local.md` when they are per-developer or per-checkout and the file exists #25 @alexkart
- defer to project memory placement rules before using workflow defaults #25 @alexkart

### Improvements

- show inferred memory destinations in the selection prompt #25 @alexkart
- clarify that `Other` selects discoveries only, not arbitrary output paths #25 @alexkart

### Bug Fixes

- read user memory while checking for duplicate discoveries #25 @alexkart

## workflow v1.0.1 - 2026-05-14

### Bug Fixes

- align learn skill wording with Claude Code memory docs #24 @alexkart

## planning v3.7.1 - 2026-05-13

### Bug Fixes

- keep the worktree choice mandatory and reframe the prompt by current branch state 74789cc

## planning v3.7.0 - 2026-05-13

### New Features

- add stats summary phase to `/planning:exec` with wall-clock time, tokens, tool use, agent count, diff stats, commits, and final state 72faf91

## planning v3.6.8 - 2026-05-13

### Improvements

- change default Codex model to `gpt-5.5` and reasoning effort to `xhigh` 0d6ad06

## planning v3.6.7 - 2026-05-13

### Bug Fixes

- make the worktree question mandatory in exec step 2 bcc9a22

## planning v3.6.6 - 2026-05-13

### Bug Fixes

- require structured review findings grouped by severity and preserve agent attribution 0b4e71f

## planning v3.6.5 - 2026-05-13

### Bug Fixes

- trigger review agents in one parallel batch and require severity tags 7db9756

## planning v3.6.4 - 2026-05-13

### Improvements

- document prompt customization patterns and the subagent fanout constraint d665eab

## planning v3.6.3 - 2026-05-13

### Bug Fixes

- run review fanout from the main orchestrator because subagents cannot spawn agents 957b0ad

## planning v3.6.2 - 2026-05-13

### New Features

- pass the plan file to Codex so review has intent context 1379f32

## planning v3.6.1 - 2026-05-13

### New Features

- stop the Codex review loop after an iteration has no critical or major findings 0917ff4

## brainstorm v2.2.2 - 2026-05-04

### Bug Fixes

- align brainstorm-generated plan filenames with `/planning:make` 5f947a7

## planning v3.6.0 - 2026-04-25

### New Features

- add `CODEX_NO_OVERRIDES=1` for Codex wrappers that reject `-c` overrides #20 @paskal

## planning v3.5.1 - 2026-04-25

### Improvements

- modernize Mercurial dispatch for newer `hg` behavior #19 @paskal

## planning v3.5.0 - 2026-04-23

### Bug Fixes

- add zellij, kaku, cmux, ghostty, iTerm2, and emacs vterm backends to the plan review launcher #18 @umputun
- list kaku in the no-overlay error message #18 @umputun

## planning v3.4.0 - 2026-04-17

### New Features

- add Mercurial support to `/planning:exec` helper scripts #15 @paskal
- add VCS dispatch for branch detection, branch creation, commit staging, and Codex review #15 @paskal

### Improvements

- skip git-only finalize and external review phases in Mercurial repositories #15 @paskal

## planning v3.3.0 - 2026-04-16

### New Features

- narrow phase 1 re-check loop to critical review agents d7a1f65

## thinking-tools v1.2.0 - 2026-04-13

### New Features

- add stuck-detection triggers to `ask-codex` c5091a7
- add adversarial code review template with structured JSON output c5091a7

### Improvements

- split `ask-codex` presentation formats for investigation and review c5091a7
- update default Codex model to `gpt-5.4` c5091a7

## planning v3.2.1 - 2026-04-13

### Bug Fixes

- pass plugin data directory as an argument to custom-rule resolve scripts 8aaa38b

## brainstorm v2.2.1 - 2026-04-13

### Bug Fixes

- pass plugin data directory as an argument to custom-rule resolve scripts 8aaa38b

## planning v3.2.0 - 2026-04-12

### New Features

- add custom rules injection to `/planning:make`, `/planning:exec`, and plan-review #13 @umputun
- add `custom-rules.md` and `usage.md` references for planning #13 @umputun
- add tests for custom-rule resolution #13 @umputun

### Bug Fixes

- fix README manual install copy paths for planning references #13 @umputun
- add `$CLAUDE_PLUGIN_DATA` guard to rules management instructions #13 @umputun

## brainstorm v2.2.0 - 2026-04-12

### New Features

- add custom rules injection to the brainstorm skill #13 @umputun
- add `custom-rules.md` and `usage.md` references for brainstorm #13 @umputun
- add tests for custom-rule resolution #13 @umputun

## planning v3.1.2 - 2026-04-04

### Bug Fixes

- use `window_id` instead of `id` for kitty overlay targeting 33a6b57

## review v2.2.1 - 2026-04-04

### Bug Fixes

- use `window_id` instead of `id` for kitty overlay targeting 33a6b57

## planning v3.1.1 - 2026-04-04

### Bug Fixes

- fix `AskUserQuestion` option limit and script path resolution 3635dc5

## planning v3.1.0 - 2026-04-04

### New Features

- add revdiff support for plan review with editor fallback 1fcf4d4

### Improvements

- replace unnecessary Git-specific prose with generic repository wording #11 @paskal
- add Solution Overview and TodoWrite guidance to `/planning:make` 1fcf4d4

## brainstorm v2.1.0 - 2026-04-04

### Improvements

- rename direct skill invocation from `/brainstorm:do` to `/brainstorm:brainstorm` 1ee00db

## planning v3.0.3 - 2026-03-31

### Bug Fixes

- fix YAML frontmatter parsing and shellcheck warnings #10 @paskal

### Other

- add CI checks for YAML frontmatter and shell scripts #10 @paskal

## release-tools v2.0.1 - 2026-03-31

### Bug Fixes

- fix shellcheck warnings in release note generation #10 @paskal

## planning v3.0.2 - 2026-03-31

### New Features

- add `Execute autonomously` option to `/planning:make` 76132b3

### Bug Fixes

- stop the exec orchestrator from doing subagent work directly 44bf46d
- move `plan-annotate.py` to plugin-level `scripts/` for reliable cross-plugin path resolution f7b3a6b

## planning v3.0.1 - 2026-03-31

### Bug Fixes

- make `create-branch.sh` usage mandatory in `/planning:exec` f7fc577

## planning v3.0.0 - 2026-03-30

### New Features

- add `/planning:exec` for autonomous plan execution #8 @umputun
- add task loop, multi-phase review, fixer agent, optional finalize, and override chain #8 @umputun
- add bundled exec prompts, agents, and helper scripts #8 @umputun

## review v2.2.0 - 2026-03-30

### Improvements

- remove personal preferences from the writing-style skill #9 @umputun

## planning v2.1.2 - 2026-03-27

### Bug Fixes

- correct plan-review references to `/planning:make` #6 @bronislav
- resolve `$EDITOR` to an absolute path in overlay shells #7 @bronislav
- replace stale `/action:plan` reference with `/planning:make` 2dfcf67

## planning v2.1.1 - 2026-03-16

### Bug Fixes

- use the focused window for file-mode kitty overlay c9078b7

## review v2.1.1 - 2026-03-13

### New Features

- add `--branch` flag to git-review for remote branch review f9403c8

## thinking-tools v1.1.0 - 2026-03-06

### New Features

- add `ask-codex` skill for OpenAI Codex consultation c0715a3

## planning v2.1.0 - 2026-03-01

### New Features

- add plan-review agent for automated plan quality review 8bf680a

## review v2.1.0 - 2026-02-28

### New Features

- add git-review skill for interactive diff annotation #2 @umputun

### Bug Fixes

- handle copied files like renamed files in git-review #2 @umputun
- remove dead diff argument assignment in uncommitted mode #2 @umputun
- add early git repository guard in git-review #2 @umputun

## planning v2.0.1 - 2026-02-26

### New Features

- add wezterm support to the plan annotation hook #1 @tdragon

### Bug Fixes

- target kitty overlay to the originating window ee53808
- use explicit kitty socket for the plan annotation hook 82bc6a4

## brainstorm v2.0.0 - 2026-02-17

### Improvements

- rename skill invocations to remove repeated plugin names ebd1cfb

## planning v2.0.0 - 2026-02-17

### Improvements

- rename skill invocations to remove repeated plugin names ebd1cfb

## release-tools v2.0.0 - 2026-02-17

### Improvements

- rename skill invocations to remove repeated plugin names ebd1cfb

## review v2.0.0 - 2026-02-17

### Improvements

- rename skill invocations to remove repeated plugin names ebd1cfb

## brainstorm v1.0.0 - 2026-02-17

Initial marketplace release.

### New Features

- add brainstorm skill for collaborative design dialogue 70b947f

## planning v1.0.0 - 2026-02-17

Initial marketplace release.

### New Features

- add planning plugin with `/planning:make` and plan annotation support 70b947f

## release-tools v1.0.0 - 2026-02-17

Initial release.

### New Features

- add release workflow skill and last-tag helper a59bb1f

## review v1.0.0 - 2026-02-17

Initial marketplace release.

### New Features

- add PR review and writing-style skills 70b947f

## skill-eval v1.0.0 - 2026-02-17

Initial marketplace release.

### New Features

- add skill evaluation hook 70b947f

## thinking-tools v1.0.0 - 2026-02-17

Initial release.

### New Features

- add dialectic and root-cause-investigator skills d627b3f

## workflow v1.0.0 - 2026-02-17

Initial release.

### New Features

- add learn, clarify, wrong, md-copy, and txt-copy skills 782e0e3
