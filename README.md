# cc-utils

Things to make [Claude Code](https://claude.ai/code) even better — hooks, skills, and commands, organized as a marketplace of independent plugins.

This is an unapologetically opinionated set. Every skill here is something I actually use — some multiple times a day (brainstorm, plan, review), others less often but worth having in the toolbox. There are plenty of plugin collections out there, from random grab-bags to well-organized catalogs. This one is mine, and it reflects how I work. Even if you don't need my particular toolbox, it might give you ideas for building your own and making Claude Code do what you want it to do.

## Install

Add the marketplace, then install the plugins you want:

    /plugin marketplace add aohoyd/cc-utils

    /plugin install brainstorm@aohoyd-cc-utils
    /plugin install code@aohoyd-cc-utils
    /plugin install review@aohoyd-cc-utils
    /plugin install planning@aohoyd-cc-utils
    /plugin install release-tools@aohoyd-cc-utils
    /plugin install thinking-tools@aohoyd-cc-utils
    /plugin install skill-eval@aohoyd-cc-utils
    /plugin install workflow@aohoyd-cc-utils

Test a plugin locally:

    claude --plugin-dir plugins/brainstorm

<details>
<summary>Manual install (alternative)</summary>

Copy the files you want to your Claude Code config directory manually.

**brainstorm** — skill + command:
```bash
cp -r plugins/brainstorm/skills/do ~/.claude/skills/
cp plugins/brainstorm/commands/do.md ~/.claude/commands/
```

**code** — skill + agents:
```bash
cp -r plugins/code/skills/review ~/.claude/skills/
cp plugins/code/commands/review.md ~/.claude/commands/
cp plugins/code/agents/*.md ~/.claude/agents/
```

Note: when installed manually, update `${CLAUDE_PLUGIN_ROOT}` references inside `brainstorm/SKILL.md` to use `~/.claude/skills/brainstorm` instead.

**review** — skills (review-pr + git-review + writing-style):
```bash
cp -r plugins/review/skills/pr ~/.claude/skills/
cp -r plugins/review/skills/git-review ~/.claude/skills/
cp -r plugins/review/skills/writing-style ~/.claude/skills/
chmod +x ~/.claude/skills/git-review/scripts/git-review.py
```

Note: update the `/review:writing-style` reference inside `pr/SKILL.md` to `/writing-style` when installed manually.

**planning** — commands + agents + hook:
```bash
cp plugins/planning/commands/make.md ~/.claude/commands/
cp plugins/planning/commands/execute.md ~/.claude/commands/
cp plugins/planning/agents/task-executor.md ~/.claude/agents/
cp plugins/planning/scripts/plan-annotate.py ~/.claude/scripts/
chmod +x ~/.claude/scripts/plan-annotate.py
```

Note: when installed manually, update `${CLAUDE_PLUGIN_ROOT}` references inside `make.md` to use the appropriate local paths instead.

Add the plan-annotate hook to `~/.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "ExitPlanMode",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/scripts/plan-annotate.py",
        "timeout": 345600
      }]
    }]
  }
}
```

**release-tools** — skills + scripts:
```bash
cp -r plugins/release-tools/skills/new ~/.claude/skills/
cp -r plugins/release-tools/skills/last-tag ~/.claude/skills/
chmod +x ~/.claude/skills/release/scripts/*.sh
```

**thinking-tools** — skills:
```bash
cp -r plugins/thinking-tools/skills/ask-codex ~/.claude/skills/
cp -r plugins/thinking-tools/skills/dialectic ~/.claude/skills/
cp -r plugins/thinking-tools/skills/root-cause-investigator ~/.claude/skills/
```

**skill-eval** — hook:
```bash
cp plugins/skill-eval/hooks/skill-forced-eval-hook.sh ~/.claude/scripts/
chmod +x ~/.claude/scripts/skill-forced-eval-hook.sh
```

Add the skill-eval hook to `~/.claude/settings.json`:
```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "~/.claude/scripts/skill-forced-eval-hook.sh"
      }]
    }]
  }
}
```

**workflow** — skills:
```bash
cp -r plugins/workflow/skills/learn ~/.claude/skills/
cp -r plugins/workflow/skills/clarify ~/.claude/skills/
cp -r plugins/workflow/skills/wrong ~/.claude/skills/
cp -r plugins/workflow/skills/md-copy ~/.claude/skills/
cp -r plugins/workflow/skills/txt-copy ~/.claude/skills/
```

Restart Claude Code for changes to take effect.

</details>

## Updating plugins

The `/plugin` menu has two update paths, and they behave differently:

- `/plugin` → **Marketplaces** → **Update marketplace** — pulls the latest plugin catalog from the repo immediately. This is the reliable way to get updates.
- `/plugin` → **Installed** → **Update now** — uses a local cache that can be stale for a long time and may not reflect recent changes. Use this as a fallback after updating the marketplace.

To keep plugins current automatically, enable `/plugin` → **Marketplaces** → **Enable auto-update**. This updates the marketplace catalog on each session start.

## Plugins

| Plugin | Description |
|--------|-------------|
| [brainstorm](#brainstorm) | Collaborative design dialogue — idea to approaches to design to plan |
| [code](#code) | Code analysis and review — parallel code review, codebase exploration, architecture design |
| [review](#review) | PR review + interactive git diff annotation review + writing style guide |
| [planning](#planning) | Structured implementation planning with plan execution and interactive annotation review |
| [release-tools](#release-tools) | Release workflow — auto-versioning, release notes, changelog |
| [thinking-tools](#thinking-tools) | Analytical thinking — dialectic analysis, root cause investigation, codex consultation |
| [skill-eval](#skill-eval) | Forces skill evaluation before every response |
| [workflow](#workflow) | Session helpers — knowledge capture, confusion handling, clipboard copy |

### brainstorm

Collaborative design skill with codebase-aware exploration. Invoke with `/brainstorm:do` or trigger phrases like "brainstorm", "let's brainstorm", "help me design", "explore options for", "I have an idea", etc.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/brainstorm:do` | Collaborative design dialogue — idea → approaches → design → plan |
| command | `/brainstorm:do <desc>` | Entry point for brainstorm skill |

Guides a 4-phase dialogue to turn ideas into designs:

1. **Understand** — reads project context and explores the relevant code (`code:explorer` subagents for large/unfamiliar areas, direct Read/Grep for small ones), anchoring on the user's words rather than the code's shape. Bug reports get reproduced before designing; investigation requests get measured before scoping questions. Questions are batched via AskUserQuestion (multiple choice preferred), and concrete facts in options are grep-verified first
2. **Explore Approaches** — only when multiple viable approaches genuinely remain: proposes 2-3 options (or launches `code:architect` subagents for complex ideas), sanity-checked against Phase 1 findings, leading with a recommendation; skipped honestly when Phase 2 answers already settled the approach
3. **Present Design** — breaks design into sections of 200-300 words as visible message text, validates each incrementally via AskUserQuestion; load-bearing UX choices (keybindings, removals, narrow-width behavior) are surfaced as their own questions instead of buried in bulk approvals
4. **Next Steps** — offers to save design doc and create plan (`/planning:make <design-path>`), enter plan mode, or start implementing. Brainstorm never commits — the design file is left uncommitted; a commit prompt happens later in `/planning:make` or `/code:sweep`

### code

Code analysis, review, and fixing tools — specialized reviewer agents, fixer agent, codebase exploration, and architecture design. Used by brainstorm and planning plugins for codebase-aware workflows.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/code:review` | Parallel code review — reports issues, does not fix |
| skill | `/code:sweep` | Thorough 2-phase review + fix using specialized agents and fixer |
| skill | `/code:use-modern-go` | Modern Go syntax guidelines scoped to the project's detected Go version |
| skill | `/code:ponytail` | Lazy-senior-dev mode — forces the simplest solution that works (YAGNI → stdlib → native → one line) |
| skill | `/code:ponytail-review` | Review a diff for over-engineering only — delete-list, one line per finding |
| skill | `/code:ponytail-audit` | Whole-repo over-engineering audit — ranked delete-list |
| skill | `/code:ponytail-debt` | Harvest `ponytail:` shortcut comments into a tracked debt ledger |
| skill | `/code:ponytail-help` | Quick-reference card for the ponytail skills and intensity levels |
| command | `/code:review [scope]` | Entry point for code review skill |
| agent | `explorer` | Deep codebase analysis — traces execution paths, maps architecture layers |
| agent | `architect` | Architecture design — analyzes patterns, produces implementation blueprints |
| agent | `reviewer-correctness` | Focused review: bugs, security, logic errors, integration, wiring |
| agent | `reviewer-structure` | Focused review: over-engineering, code smells, conventions, anti-patterns |
| agent | `reviewer-testing` | Focused review: test coverage, quality, fake test detection |
| agent | `reviewer-documentation` | Focused review: README/CLAUDE.md update gaps |
| agent | `fixer` | Verifies review findings, fixes confirmed issues, validates, reports |

**code:review** — reports issues but does NOT fix them. Launches 4 specialized reviewer agents in parallel (correctness, structure, testing, documentation). Consolidates findings, deduplicates, filters by confidence >= 80, groups by severity. Standalone trigger checks `git diff` for unstaged changes. If the [hunk](https://hunk.dev/) CLI is installed and a live Hunk session is open for the reviewed repo, findings are also mirrored into the session as inline comments — the session is reloaded to the reviewed diff first, stale agent comments are cleared, and all findings land in one batch (severity + description as the summary, suggested fix + confidence as the rationale; per-comment fallback if the batch is rejected). The screen report stays canonical; without hunk the review is unchanged.

**code:sweep** — thorough 2-phase review that finds AND fixes issues. Used by `/planning:execute` after all tasks complete, or invoke standalone with `/code:sweep`. Before reviewing, the orchestrator assembles a decision-context block (design doc decisions, user choices from the session, environment constraints) that every reviewer and fixer prompt carries — findings that contradict an approved decision are escalated to the user as design conflicts instead of being "fixed". Each phase prints findings to the user, buckets them by severity (`[CRITICAL]`/`[MAJOR]`/`[MINOR]`; MAJOR requires behavior/correctness impact — missing tests and doc drift are MINOR), creates one task per non-empty bucket, then runs fixers serially in priority order (critical → major → minor). Before each fixer batch the orchestrator records a `git stash create` safety snapshot (dangling object, tree untouched) so agent mistakes can't destroy uncommitted work. No commits during the run — fixes accumulate in the working tree across both phases (phase 2 uses a working-tree-inclusive diff so it sees uncommitted phase 1 fixes). At the very end, sweep prompts once via AskUserQuestion whether to commit, and discloses whether any runtime verification happened beyond static build/test gates. Phases:
1. **Comprehensive** (4 agents) — correctness, structure, testing, documentation reviewers run in parallel. Tagged findings go to one fixer batch per severity. Reviewers analyze only — they don't run full builds/test suites (fixers own validation) and never run interactive/UI-driving tests.
2. **Verification** (4 agents) — all reviewers with critical-and-major filter, prioritizing the files phase 1 fixers modified (fixer-introduced regressions are what this phase catches). Tagged findings go to one fixer batch per severity.

**Specialized reviewer agents** — four focused reviewers used by both `/code:review` and `/code:sweep`. Each is read-only and reports findings in `file:line — description` format; `reviewer-correctness` inherits the session model, the other three run on opus:
- **reviewer-correctness** — bugs, security vulnerabilities, logic errors, edge cases, error handling, resource management, concurrency, requirement coverage, wiring/integration
- **reviewer-structure** — code smells, convention adherence, anti-patterns, dead code, duplication, naming. Over-engineering is judged against ponytail's ladder (should it exist at all → stdlib → native platform → installed dependency → shorter form) plus its rules and constraints — `ponytail:` markers read as intent, smoke tests and trust-boundary code never flagged; every finding names its concrete replacement and net line change
- **reviewer-testing** — missing tests, test quality, fake test detection, edge case coverage
- **reviewer-documentation** — README/CLAUDE.md documentation gaps for new features, APIs, configs

**fixer** — receives review findings, verifies each against actual code (20-30 lines of context), fixes confirmed issues, validates (build + tests), and reports structured results (fixed / false positive / design conflict). Never commits or stages unless explicitly instructed, never runs working-tree-reverting git commands (`checkout -- <paths>`, `restore`, `stash`, `reset`, `clean`), and bounds long validation (no interactive/UI tests, compile-only checks for those targets). Used by `/code:sweep` (one batch per severity, serial).

**code-explorer** — traces feature implementations from entry points through all abstraction layers. Outputs file:line references, execution flow, architecture insights, and essential file lists. Used by brainstorm for codebase context gathering.

**code-architect** — designs feature architectures by analyzing existing codebase patterns. Outputs decisive blueprints with component design, implementation maps, data flows, and build sequences. Used by brainstorm for approach exploration on complex features.

**code:use-modern-go** — detects the project's Go version from `go.mod` and provides modern Go syntax guidelines (built-ins, `slices`/`maps`/`cmp`, iterators, etc.) up to and including that version. The `architect`, `reviewer-correctness`, and `fixer` agents invoke this skill automatically when they detect Go code (a `go.mod` or `.go` files) so designs, reviews, and fixes follow current idioms.

**ponytail skills** — a set of "lazy senior dev" skills adapted from [ponytail](https://github.com/DietrichGebert/ponytail) (MIT, by Dietrich Gebert), ported as skills only (no always-on hooks):
- **code:ponytail** — forces the laziest solution that actually works. Climbs a ladder before writing code (does it need to exist at all → stdlib → native platform feature → one line → minimum), refuses unrequested abstractions, and marks deliberate shortcuts with `ponytail:` comments. Has three intensities — `lite` (name the lazier option), `full` (the ladder enforced, default), `ultra` (deletion-first, challenges the requirement). Without the upstream hooks it activates on invocation and stays active for the rest of the conversation; say "stop ponytail" or "normal mode" to revert.
- **code:ponytail-review** — reviews the current diff for over-engineering only (not correctness). One line per finding, tagged `delete`/`stdlib`/`native`/`yagni`/`shrink`, ending with net lines removable. Lists fixes, applies nothing.
- **code:ponytail-audit** — same as ponytail-review but scans the whole repo instead of a diff, ranked biggest cut first.
- **code:ponytail-debt** — greps the repo for `ponytail:` shortcut comments and collects them into a debt ledger so deferrals get tracked instead of forgotten; flags any marker that names no upgrade trigger.
- **code:ponytail-help** — one-shot reference card for the ponytail skills and intensity levels.

### review

PR review, interactive git diff annotation review, and writing style tools. Install together — review-pr uses writing-style for drafting comments.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/review:pr <number>` | PR review with architecture analysis, scope creep detection, and merge workflow |
| skill | `/review:git-review [ref]` | Interactive git diff annotation review — editor overlay with feedback loop |
| skill | `/review:writing-style` | Direct technical communication — anti-AI-speak, brevity, no filler |

**review-pr** — analyzes code quality, architecture, test coverage, and identifies scope creep:
- **Phase 0** — detects PR vs issue (issues get a simpler comment-only flow)
- **Phase 1** — fetches PR metadata, discussion history, merge status, and inline suggestions
- **Phase 1.5** — asks review mode: Full (worktree + tests + linter + architecture) or Quick (diff-only)
- **Phase 2** — sets up worktree and launches a subagent for deep analysis
- **Phase 3-4** — presents findings, resolves open design questions
- **Phase 5** — drafts review comment using `/review:writing-style`, posts as formal review
- **Post-approve** — recommends merge strategy (rebase vs squash vs merge)

Uses `gh` CLI for all GitHub operations and git worktrees to avoid disrupting the current checkout.

**git-review** — interactive annotation-based code review. Generates a cleaned-up diff, opens it in `$EDITOR` via tmux popup, kitty overlay, or wezterm split-pane. You annotate directly in the diff, and the script returns your changes as a git diff. Claude reads annotations, fixes code, regenerates the diff, and loops until you close the editor without changes. Supports auto-detection of uncommitted changes or branch diffs.

Run tests: `python3 plugins/review/skills/git-review/scripts/git-review.py --test`

**writing-style** — enforces direct, brief writing for tickets, PRs, code reviews, and commit messages. Core principles: brevity, honest feedback, problem-solution structure, technical precision, anti-AI-speak. Does NOT apply to README.md, public docs, or blog posts.

### planning

Structured implementation planning with plan execution via subagents and interactive annotation review.

| Component | Trigger | Description |
|-----------|---------|-------------|
| command | `/planning:make <desc>` | Structured implementation plan with interactive review loop |
| command | `/planning:execute [path]` | Execute plan task-by-task with fresh subagents, then run `/code:sweep` |
| hook | `PreToolUse` / CLI | Plan annotation in `$EDITOR` with diff-based feedback loop |
| agent | `plan-review` | Automated plan quality review — completeness, over-engineering, testing |
| agent | `task-executor` | Executes individual plan tasks following TDD workflow |

**plan command** — creates a plan file in `docs/plans/YYYY-MM-DD-<task-name>.md` through interactive context gathering. No commits during planning — when the user picks "Done" or finishes "Implement", the command prompts via AskUserQuestion whether to commit; "Execute with subagents" defers the commit decision to the end of the execute → sweep chain.
- **Step 0** — parses intent and explores codebase. If `$ARGUMENTS` is a path to a `*-design.md` file under `docs/plans/`, it's recorded in the plan's `Design:` header so the design file moves to `docs/plans/completed/` alongside the plan on completion
- **Step 1** — asks focused questions batched into one AskUserQuestion call (goal, scope, constraints, testing approach), skipping anything a design doc or the session already settled; the plan title is derived, not asked
- **Step 1.5** — proposes 2-3 implementation approaches with trade-offs (skipped if obvious)
- **Step 2** — creates the plan file with `Design:` and `Base:` headers (base branch@sha lets execute verify it runs on the base the plan assumed), tasks, file lists, test requirements, a runtime-verification acceptance item, and progress tracking. Supports both Regular (checkbox) and TDD (test-first with verify fail/pass steps) task formats. Bug-fix plans reproduce the bug first when feasible, or mark the diagnosis UNVERIFIED
- **Step 3** — offers execute with subagents (`/planning:execute`, which auto-runs the plan-review agent first), interactive review, start implementation directly, or done (4 options — the AskUserQuestion cap)

**execute command** — runs an implementation plan task-by-task using fresh `task-executor` subagents (one per task, mandatory; adjacent trivial doc/one-line tasks may share one spawn). Before executing it verifies the plan's `Base:` commit is an ancestor of HEAD (catching stale-worktree bases) and auto-runs the `plan-review` agent — NEEDS REVISION fixes are applied to the plan automatically before any task runs. Every executor prompt carries a session-constraints block (environment quirks, user decisions, mid-session corrections). Tasks needing the user's machine or screen (visual checks, UI-driving tests) are negotiated via AskUserQuestion instead of blindly delegated. After all tasks complete, invokes `/code:sweep` for thorough 2-phase review + fix. The single end-of-workflow commit prompt lives in sweep, covering plan + task changes + sweep fixes together. Handles failures gracefully — stops, reports, and asks user to retry/skip/stop.

**plan-annotate.py** — interactive plan annotation tool. Opens plans in your `$EDITOR` via a terminal overlay (tmux popup, kitty overlay, or wezterm split-pane), lets you annotate directly, and feeds a unified diff back to Claude so it revises the plan. Two modes:

- *Hook mode* (default) — intercepts `ExitPlanMode`, opens plan in editor, denies tool call with diff if changes made, forcing revision loop
- *File mode* (`plan-annotate.py <plan-file>`) — outputs unified diff to stdout for integration with custom workflows

Requirements: tmux, kitty, or wezterm terminal, `$EDITOR` (defaults to `micro`). **Kitty users** must enable remote control in `kitty.conf`:

```
allow_remote_control yes
listen_on unix:/tmp/kitty-$KITTY_PID
```

*Note*: when `revdiff` is installed, the `ExitPlanMode` hook and `/planning:make` interactive review both route through `launch-plan-review.sh` instead, which supports a wider set of overlays: agterm, tmux, zellij, herdr, kitty, wezterm/kaku, cmux, ghostty, iTerm2, and emacs vterm. The 3-terminal list above applies only to the `$EDITOR` fallback when revdiff is not installed.

*Disabling review*: set `PLANNING_DISABLE_REVDIFF=1` to skip interactive plan review entirely on both routes (revdiff and the `$EDITOR` fallback). No overlay opens and the plan proceeds to the normal `ExitPlanMode` confirmation. This exists for remote clients (`claude /remote-control`): the overlay always opens on the host terminal, which a mobile or web client cannot see or interact with, so review would otherwise block the session. The variable is read when review fires, so export it in your shell before starting a session you may later drive remotely.

The overlay popup size is configurable via env vars:

| Env var | Description | Default |
|---------|-------------|---------|
| `REVDIFF_POPUP_WIDTH` | Tmux/Zellij popup width (e.g., `100%`, `80%`) | `90%` |
| `REVDIFF_POPUP_HEIGHT` | Tmux/Zellij popup height / wezterm split percent | `90%` |

Run tests: `python3 plugins/planning/scripts/plan-annotate.py --test`

**plan-review agent** — automated plan quality reviewer. Analyzes plans for problem definition, solution correctness, scope creep, over-engineering, testing requirements, task granularity, and convention adherence. Runs automatically at the start of `/planning:execute`; also available on request from the plan command's next-step menu. Outputs a structured report with severity-rated findings and an APPROVE/NEEDS REVISION verdict.

### release-tools

Release workflow tools for creating versioned releases with auto-generated notes.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/release-tools:new` | Create GitHub/GitLab/Gitea release with auto-versioning and release notes |
| skill | `/release-tools:last-tag` | Show commits since the last tag in a formatted table |

**release** — full release workflow: asks release type (hotfix/minor/major), auto-detects platform (GitHub/GitLab/Gitea), calculates semantic version, generates release notes grouped by type (features/improvements/fixes) from merged PRs and commits, updates CHANGELOG if present, shows preview for confirmation, then publishes. Includes helper scripts for platform detection, version calculation, and notes generation.

**last-tag** — shows commits since the last tag in a formatted table with date, author, hash, and description. Detects single vs multiple authors and adjusts table layout. Offers interactive drill-down into individual commit details.

### thinking-tools

Analytical thinking tools for objective analysis.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/thinking-tools:ask-codex` | Consult OpenAI Codex (GPT-5.5) for investigation, debugging, or code review |
| skill | `/thinking-tools:dialectic <statement>` | Prove and counter-prove a statement using parallel agents |
| skill | `/thinking-tools:root-cause-investigator` | Systematic 5-Why root cause analysis for errors and bugs |

**ask-codex** — consults OpenAI Codex (GPT-5.5) as a second opinion for debugging, investigation, or code review. Builds a focused prompt from conversation context, runs codex in read-only sandbox mode in the background, and presents findings with an independent assessment. Requires `codex` CLI to be installed and authenticated.

**dialectic** — runs two agents in parallel with opposing goals (thesis vs antithesis) to eliminate confirmation bias. One agent finds all positive evidence, the other finds all negative evidence. After both complete, synthesizes findings into an objective conclusion and verifies cited evidence against actual code.

Use cases: architecture decisions, bug analysis, performance claims, refactoring safety, code review.

**root-cause-investigator** — applies 5-Why methodology to drill from symptoms to fundamental root causes. Structures investigation through progressive depth: surface cause → process issues → system problems → design issues → root cause. Includes reference materials for common patterns (race conditions, resource exhaustion, integration failures) and investigation techniques.

### skill-eval

Forces skill evaluation before every response.

| Component | Trigger | Description |
|-----------|---------|-------------|
| hook | `UserPromptSubmit` | Forces skill evaluation before every response |

By default, Claude Code often ignores available skills and jumps straight to generic responses. This hook injects a system reminder on every prompt that enforces an evaluate → activate → implement sequence. When installed, Claude will either list relevant skills and call `Skill()` for each before implementing, or proceed directly when no skills are relevant.

### workflow

Session workflow helpers for knowledge capture, confusion handling, course correction, and clipboard operations.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/workflow:learn` | Capture strategic project knowledge to project CLAUDE.md (routes per-developer / per-checkout discoveries to CLAUDE.local.md when that file is present) |
| skill | `/workflow:clarify` | Investigate and explain user confusion, determine if real issue exists |
| skill | `/workflow:wrong` | Reset and re-evaluate when current approach isn't working |
| skill | `/workflow:md-copy` | Format final answer as markdown and copy to clipboard |
| skill | `/workflow:txt-copy` | Copy generated text content to clipboard |

**learn** — reviews conversation history, extracts strategic project knowledge (architecture patterns, conventions, operational insights), and saves selected items to the project CLAUDE.md. When `CLAUDE.local.md` is present, per-developer / per-checkout discoveries (machine-specific tooling, environment quirks) are routed there instead. Defers to any project-defined memory-placement guidance documented in `CLAUDE.md` or `.claude/rules/`. Uses granular selection via AskUserQuestion so the user picks exactly what to keep.

**clarify** — activates on confusion signals ("I don't understand", "why is this happening", etc.). Investigates the actual codebase to determine whether the confusion stems from a misunderstanding or a real issue. If real, proceeds to plan mode for a fix.

**wrong** — resets the current approach when it's not working. Re-analyzes the core problem, proposes 2-3 fresh alternatives with trade-offs, and recommends the best path forward.

**md-copy** — formats the session's final answer as clean markdown (bold titles instead of headings, proper tables, code blocks) and copies to clipboard. Cross-platform clipboard detection (macOS pbcopy, Linux xclip/xsel).

**txt-copy** — copies generated text (emails, messages, letters) to clipboard via a timestamped temp file. Cross-platform clipboard detection (macOS pbcopy, Linux xclip/xsel).

## Custom Rules

Both the **planning** and **brainstorm** plugins support custom rules injection — free-form markdown files loaded at skill invocation time and applied as additional instructions alongside built-in behavior.

**Two levels**, checked in order (first-found-wins, never merged):

1. **Project-level**: `.claude/<rules-file>.md` in the current working directory
2. **User-level**: `$CLAUDE_PLUGIN_DATA/<rules-file>.md` (per-plugin persistent storage)

When both non-empty files exist, only the project-level file is used. Empty files are treated as absent and fall through to the next level.

| Plugin | Rules file | Affects |
|--------|-----------|---------|
| planning | `planning-rules.md` | make, exec, plan-review |
| brainstorm | `brainstorm-rules.md` | brainstorm skill |

**Example** — create `.claude/planning-rules.md` in your project:

```markdown
## testing conventions
- use table-driven tests with testify
- mock external dependencies with moq
- aim for 80% coverage minimum

## plan structure preferences
- max 5 checkboxes per task
- always include rollback steps for migrations
```

**Managing rules** — ask the make command or brainstorm skill to add, show, or clear rules at either level (exec loads rules but management is done through make or brainstorm):

- "show my planning rules" — displays current rules and which level they came from
- "add Go testing rules to project-level planning rules" — writes to `.claude/planning-rules.md`
- "set up brainstorm rules from my-conventions.md" — reads file and writes to rules location
- "clear user-level brainstorm rules" — deletes `$CLAUDE_PLUGIN_DATA/brainstorm-rules.md`

## Credits

Some skills and scripts were influenced by or adapted from community ideas, blog posts, and open-source examples. Sources were not tracked accurately from the start. If you recognize your work and want proper attribution, please [open an issue](https://github.com/aohoyd/cc-utils/issues) — I'll fix it.

## License

MIT
