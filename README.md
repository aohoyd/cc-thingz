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

1. **Understand** — reads project context, launches `code:explorer` subagents in parallel to gather codebase context, asks questions one at a time (multiple choice preferred, uses AskUserQuestion tool exclusively)
2. **Explore Approaches** — for simple solutions, proposes 2-3 options; for complex ones, launches `code:architect` subagents in parallel. Leads with recommendation
3. **Present Design** — breaks design into sections of 200-300 words, validates each incrementally via AskUserQuestion
4. **Next Steps** — offers to save design doc and create plan (`/planning:make`), enter plan mode, or start implementing

### code

Code analysis, review, and fixing tools — specialized reviewer agents, fixer agent, codebase exploration, and architecture design. Used by brainstorm and planning plugins for codebase-aware workflows.

| Component | Trigger | Description |
|-----------|---------|-------------|
| skill | `/code:review` | Parallel code review — reports issues, does not fix |
| skill | `/code:sweep` | Thorough 3-phase review + fix using specialized agents and fixer |
| command | `/code:review [scope]` | Entry point for code review skill |
| agent | `explorer` | Deep codebase analysis — traces execution paths, maps architecture layers |
| agent | `architect` | Architecture design — analyzes patterns, produces implementation blueprints |
| agent | `reviewer` | Code review — bugs, logic errors, security, conventions with confidence scoring |
| agent | `reviewer-quality` | Focused review: bugs, security, simplicity |
| agent | `reviewer-implementation` | Focused review: requirement correctness, wiring, completeness |
| agent | `reviewer-testing` | Focused review: test coverage, quality, fake test detection |
| agent | `reviewer-simplification` | Focused review: over-engineering detection |
| agent | `reviewer-documentation` | Focused review: README/CLAUDE.md update gaps |
| agent | `reviewer-smells` | Focused review: conventions, code smells, anti-patterns |
| agent | `fixer` | Verifies review findings, fixes confirmed issues, validates, commits |

**code:review** — reports issues but does NOT fix them. Two modes: quick (single reviewer, used after individual tasks) and comprehensive (3 parallel reviewers with different focuses: simplicity/DRY, bugs/correctness, conventions/patterns). Consolidates findings, deduplicates, filters by confidence >= 80, groups by severity. Standalone trigger checks `git diff` for unstaged changes.

**code:sweep** — thorough 3-phase review that finds AND fixes issues. Used by `/planning:execute` after all tasks complete, or invoke standalone with `/code:sweep`. Phases:
1. **Comprehensive** (5 agents) — quality, implementation, testing, simplification, documentation reviewers run in parallel. Findings go to fixer. Loops up to 3 iterations until clean.
2. **Smells** (1 agent) — convention adherence and code quality. Single pass with fixer.
3. **Critical** (2 agents) — quality + implementation with critical-only constraint. Single pass with fixer.

**Specialized reviewer agents** — six focused reviewers used by `/code:sweep`. Each is read-only (sonnet model) and reports findings in `file:line — description` format:
- **reviewer-quality** — bugs, security vulnerabilities, error handling, resource management, concurrency
- **reviewer-implementation** — requirement coverage, correctness of approach, wiring/integration, completeness
- **reviewer-testing** — missing tests, test quality, fake test detection, edge case coverage
- **reviewer-simplification** — excessive abstraction, premature generalization, unnecessary indirection
- **reviewer-documentation** — README/CLAUDE.md documentation gaps for new features, APIs, configs
- **reviewer-smells** — project convention adherence, dead code, duplicated logic, anti-patterns

**fixer** — receives review findings, verifies each against actual code (20-30 lines of context), fixes confirmed issues, validates (build + tests), commits, and reports structured results. Used by `/code:sweep` after each review phase.

**code-explorer** — traces feature implementations from entry points through all abstraction layers. Outputs file:line references, execution flow, architecture insights, and essential file lists. Used by brainstorm for codebase context gathering.

**code-architect** — designs feature architectures by analyzing existing codebase patterns. Outputs decisive blueprints with component design, implementation maps, data flows, and build sequences. Used by brainstorm for approach exploration on complex features.

**code-reviewer** — reviews code against CLAUDE.md guidelines and language idioms. Confidence-based filtering (0-100 scale, only reports >= 80). Groups findings by severity with concrete fix suggestions.

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

**plan command** — creates a plan file in `docs/plans/yyyymmdd-<task-name>.md` through interactive context gathering:
- **Step 0** — parses intent and explores codebase for relevant context
- **Step 1** — asks focused questions one at a time (goal, scope, constraints, testing approach, title)
- **Step 1.5** — proposes 2-3 implementation approaches with trade-offs (skipped if obvious)
- **Step 2** — creates the plan file with tasks, file lists, test requirements, and progress tracking. Supports both Regular (checkbox) and TDD (test-first with verify fail/pass steps) task formats
- **Step 3** — offers interactive review, auto review, execute with subagents (`/planning:execute`), start implementation directly, or done

**execute command** — runs an implementation plan task-by-task using fresh `task-executor` subagents (one per task, mandatory). After all tasks complete, invokes `/code:sweep` for thorough 3-phase review + fix. Handles failures gracefully — stops, reports, and asks user to retry/skip/stop.

**plan-annotate.py** — interactive plan annotation tool. Opens plans in your `$EDITOR` via a terminal overlay (tmux popup, kitty overlay, or wezterm split-pane), lets you annotate directly, and feeds a unified diff back to Claude so it revises the plan. Two modes:

- *Hook mode* (default) — intercepts `ExitPlanMode`, opens plan in editor, denies tool call with diff if changes made, forcing revision loop
- *File mode* (`plan-annotate.py <plan-file>`) — outputs unified diff to stdout for integration with custom workflows

Requirements: tmux, kitty, or wezterm terminal, `$EDITOR` (defaults to `micro`). **Kitty users** must enable remote control in `kitty.conf`:

```
allow_remote_control yes
listen_on unix:/tmp/kitty-$KITTY_PID
```

Run tests: `python3 plugins/planning/hooks/plan-annotate.py --test`

**plan-review agent** — automated plan quality reviewer. Analyzes plans for problem definition, solution correctness, scope creep, over-engineering, testing requirements, task granularity, and convention adherence. Used by the plan command's "Auto review" option. Outputs a structured report with severity-rated findings and an APPROVE/NEEDS REVISION verdict.

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
| skill | `/thinking-tools:ask-codex` | Consult OpenAI Codex (GPT-5) for investigation, debugging, or code review |
| skill | `/thinking-tools:dialectic <statement>` | Prove and counter-prove a statement using parallel agents |
| skill | `/thinking-tools:root-cause-investigator` | Systematic 5-Why root cause analysis for errors and bugs |

**ask-codex** — consults OpenAI Codex (GPT-5) as a second opinion for debugging, investigation, or code review. Builds a focused prompt from conversation context, runs codex in read-only sandbox mode in the background, and presents findings with an independent assessment. Requires `codex` CLI to be installed and authenticated.

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
| skill | `/workflow:learn` | Capture strategic project knowledge to local CLAUDE.md |
| skill | `/workflow:clarify` | Investigate and explain user confusion, determine if real issue exists |
| skill | `/workflow:wrong` | Reset and re-evaluate when current approach isn't working |
| skill | `/workflow:md-copy` | Format final answer as markdown and copy to clipboard |
| skill | `/workflow:txt-copy` | Copy generated text content to clipboard |

**learn** — reviews conversation history, extracts strategic project knowledge (architecture patterns, conventions, operational insights), and saves selected items to local CLAUDE.md. Uses granular selection via AskUserQuestion so the user picks exactly what to keep.

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
