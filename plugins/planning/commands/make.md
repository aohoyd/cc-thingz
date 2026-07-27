---
description: Create structured implementation plan in docs/plans/
argument-hint: describe the feature or task to plan
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion, Task, EnterPlanMode, TaskCreate, TaskUpdate, TaskList, Skill
---

# Implementation Plan Creation

create an implementation plan in `docs/plans/YYYY-MM-DD-<task-name>.md` with interactive context gathering.

**No commits during planning.** This command never runs `git commit`. When the user is fully done (picks "Done" or finishes "Implement"), the command prompts via AskUserQuestion whether to commit the plan / changes. "Execute with subagents" defers the commit decision to the end of `/planning:execute` → `/code:sweep`.

## step 0: parse intent and gather context

before asking questions, understand what the user is working on:

1. **parse user's command arguments** to identify intent:
   - if `$ARGUMENTS` looks like a path under `docs/plans/` ending in `-design.md`:
     - verify the file exists (`test -f <path>`)
     - if it exists, store the path for the plan's `Design:` header (step 2) and read the file content as additional context for the rest of step 0
     - if it does NOT exist, ask the user via AskUserQuestion whether to proceed without a design link (Yes → store `none`) or supply a correct path. Do NOT silently record a dead reference.
   - "add feature Z" / "implement W" → feature development
   - "fix bug" / "debug issue" → bug fix plan
   - "refactor X" / "improve Y" → refactoring plan
   - "migrate to Z" / "upgrade W" → migration plan
   - generic request → explore current work

2. **gather relevant context quickly** — use direct tool calls (Read, Glob, Grep), NOT an Explore agent. keep discovery under 30 seconds:

   **for feature development:**
   - glob for files matching the feature area (e.g., `**/*auth*`, `**/*cache*`)
   - read 1-3 most relevant files to understand existing patterns
   - check project structure with a quick `ls` of key directories

   **for bug fixing:**
   - grep for error messages or function names mentioned in the request
   - read the specific file(s) involved
   - check `git log --oneline -5` for recent changes
   - **reproduce the bug first when feasible** (run the failing command, test, or app). if reproduction is not feasible, mark the diagnosis as UNVERIFIED in the plan's Overview — a plan built on an assumed root cause must say so, and its verify task must reproduce the original symptom before declaring success

   **for refactoring/migration:**
   - glob for files matching the area being refactored
   - read 2-3 key files to understand current structure
   - grep for imports/references to identify dependencies

   **for generic/unclear requests:**
   - check `git status` and `git log --oneline -5`
   - read README.md or CLAUDE.md for project overview
   - `ls` the top-level directory structure

   **CRITICAL: do NOT launch an Explore agent or read more than 5 files in this step. the goal is a quick scan, not exhaustive analysis. if more context is needed, ask the user in step 1.**

3. **synthesize findings** into a brief context summary (3-5 bullet points):
   - what the project is and primary language/framework
   - which files/areas are relevant to the request
   - key patterns or conventions observed

## step 1: present context and ask focused questions

show the discovered context, then ask the questions below using the AskUserQuestion tool:

"based on your request, i found: [context summary]"

**skip questions already answered.** if step 0 loaded a design doc, or the session already settled a topic (brainstorm answers, explicit user statements earlier in the conversation), do NOT re-ask it — a decision the user already made is settled. in the design-doc path this often leaves nothing to ask — skip the AskUserQuestion call entirely and proceed to step 1.5.

**batch the remaining questions into a single AskUserQuestion call** (the tool takes up to 4 per call, max 4 options each). do not drip-feed one question per round.

topics to cover (only where not already settled):

1. **plan purpose**: "what is the main goal?" — multiple choice with suggested answer based on discovered intent
2. **scope**: "which components/files are involved?" — multiple choice with suggested discovered files/areas
3. **constraints**: "any specific requirements or limitations?" — can be open-ended if constraints vary widely

**testing approach — decide it yourself, don't ask.** pick TDD or regular based on the work and record the choice with a one-line rationale in the plan's Development Approach. TDD fits behavior that can be specified before writing code (bug fixes with a reproducible failing case, pure logic, parsers, contracts); regular fits work whose shape emerges while building (UI, wiring/glue, exploratory refactors). mixing is fine when tasks differ — write TDD tasks in the TDD step format. a preference the user states at any point overrides your choice.

**plan title**: derive it yourself from the design filename or the request (e.g. `docs/plans/2026-07-23-glow-dot-design.md` → `glow-dot`). only ask if the topic is genuinely ambiguous — a title question with near-identical options is a round-trip tax.

after answers arrive, synthesize responses into plan context.

**do not assert unverified codebase facts in questions or the plan** (e.g. "this project has no test framework") — verify with a quick Glob/Grep first; downstream skills treat plan statements as ground truth.

## step 1.5: explore approaches

once the problem is understood, propose implementation approaches:

1. **propose 2-3 different approaches** with trade-offs for each
2. **lead with recommended option** and explain reasoning
3. **present conversationally** - not a formal document yet

example format:
```
i see three approaches:

**Option A: [name]** (recommended)
- how it works: ...
- pros: ...
- cons: ...

**Option B: [name]**
- how it works: ...
- pros: ...
- cons: ...

which direction appeals to you?
```

use AskUserQuestion tool to let user select preferred approach before creating the plan.

**skip this step** if:
- the implementation approach is obvious (single clear path)
- user explicitly specified how they want it done
- it's a bug fix with clear solution

## step 2: create plan file

check `docs/plans/` for existing files, then create `docs/plans/YYYY-MM-DD-<task-name>.md` (use current date):

### plan structure

```markdown
# [Plan Title]

> **For Claude:** use `/planning:execute` to implement this plan task-by-task with fresh subagents.

**Goal:** [one sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [key technologies/libraries]

**Design:** [relative path to source design file under `docs/plans/`, or `none`]

**Base:** [output of `git rev-parse --abbrev-ref HEAD`@`git rev-parse --short HEAD` at plan time — lets /planning:execute verify it runs on the same base the plan assumed]

## Overview
- clear description of the feature/change being implemented
- problem it solves and key benefits
- how it integrates with existing system

## Context (from discovery)
- files/components involved: [list from step 0]
- related patterns found: [patterns discovered]
- dependencies identified: [dependencies]

## Development Approach
- **testing approach**: [TDD / Regular — chosen during planning; one-line rationale. user-stated preference overrides]
- complete each task fully before moving to the next
- make small, focused changes
- **CRITICAL: every task MUST include new/updated tests** for code changes in that task
  - tests are not optional - they are a required part of the checklist
  - write unit tests for new functions/methods
  - write unit tests for modified functions/methods
  - add new test cases for new code paths
  - update existing test cases if behavior changes
  - tests cover both success and error scenarios
- **CRITICAL: all tests must pass before starting next task** - no exceptions
- **CRITICAL: update this plan file when scope changes during implementation**
- run tests after each change
- maintain backward compatibility

## Testing Strategy
- **unit tests**: required for every task — see the testing mandate in Development Approach above (not restated here)
- **e2e tests**: if project has UI-based e2e tests (Playwright, Cypress, etc.):
  - UI changes → add/update e2e tests in same task as UI code
  - backend changes supporting UI → add/update e2e tests in same task
  - treat e2e tests with same rigor as unit tests (must pass before next task)
  - store e2e tests alongside unit tests (or in designated e2e directory)
  - example: if task implements new form field, add e2e test checking form submission

## Progress Tracking
- mark completed items with `[x]` immediately when done
- add newly discovered tasks with ➕ prefix
- document issues/blockers with ⚠️ prefix
- update plan if implementation deviates from original scope
- keep plan in sync with actual work done

## Solution Overview
- high-level approach and architecture chosen
- key design decisions and rationale
- how it fits into the existing system

## Technical Details
- data structures and changes
- parameters and formats
- processing flow

## What Goes Where
- **Implementation Steps** (`[ ]` checkboxes): tasks achievable within this codebase - code changes, tests, documentation updates
- **Post-Completion** (no checkboxes): items requiring external action - manual testing, changes in consuming projects, deployment configs, third-party verifications

## Implementation Steps

<!--
Task structure guidelines:
- Each task = ONE logical unit (one function, one endpoint, one component)
- Use specific descriptive names, not generic "[Core Logic]" or "[Implementation]"
- Each task MUST have a **Files:** block listing files to Create/Modify (before checkboxes)
- Aim for ~5 checkboxes per task (more is OK if logically atomic)
- **Tests per task**: apply the testing mandate from Development Approach above. The only format-specific rule here: list tests as SEPARATE checklist items, not bundled with implementation.
- **CRITICAL: number ALL tasks with concrete sequential integers** - the two trailing tasks below are shown as "Task N-1" and "Task N" where N is a PLACEHOLDER for the total task count, NOT literal text. Substitute real numbers continuing the sequence from your last implementation task (e.g. with 14 implementation tasks they become "Task 15: Verify acceptance criteria" and "Task 16: ... Update documentation"). NEVER write the literal strings "Task N-1" or "Task N" into the plan.
- **CRITICAL: Each task MUST end with writing/updating tests before moving to next**
  - tests are not optional - they are a required deliverable of every task
  - write tests for all NEW code added in this task
  - write tests for all MODIFIED code in this task
  - include both success and error scenarios in tests
  - list tests as SEPARATE checklist items, not bundled with implementation

Example for Regular approach (NOTICE: Files block + tests as separate checklist items):

### Task 1: Add password hashing utility

**Files:**
- Create: `src/auth/hash`
- Create: `src/auth/hash_test`

- [ ] create `src/auth/hash` with HashPassword and VerifyPassword functions
- [ ] implement bcrypt-based hashing with configurable cost
- [ ] write tests for HashPassword (success + error cases)
- [ ] write tests for VerifyPassword (success + error cases)
- [ ] run tests - must pass before task 2

Example for TDD approach (NOTICE: explicit test-first → verify fail → implement → verify pass):

### Task 1: Add password hashing utility

**Files:**
- Create: `src/auth/hash`
- Test: `src/auth/hash_test`

**Step 1: Write the failing test**
```
// test code that defines expected behavior
```

**Step 2: Run test to verify it fails**
Run: `go test ./src/auth/...`
Expected: FAIL — function not defined

**Step 3: Write minimal implementation**
```
// implementation code
```

**Step 4: Run test to verify it passes**
Run: `go test ./src/auth/...`
Expected: PASS

Example for Regular approach (continued):

### Task 2: Add user registration endpoint

**Files:**
- Create: `src/api/users`
- Modify: `src/api/router`
- Create: `src/api/users_test`

- [ ] create `POST /api/users` handler in `src/api/users`
- [ ] add input validation (email format, password strength)
- [ ] integrate with password hashing utility
- [ ] write tests for handler success case with table-driven cases
- [ ] write tests for handler error cases (invalid input, missing fields)
- [ ] run tests - must pass before task 3
-->

### Task 1: [specific name - what this task accomplishes]

**Files:**
- Create: `exact/path/to/new_file`
- Modify: `exact/path/to/existing`

- [ ] [specific action with file reference - code implementation]
- [ ] [specific action with file reference - code implementation]
- [ ] write tests for new/changed functionality (success cases)
- [ ] write tests for error/edge cases
- [ ] run tests - must pass before next task

<!-- replace "N-1" and "N" below with the actual next sequential numbers continuing from your last implementation task - do NOT emit the literal letter N -->
### Task N-1: Verify acceptance criteria
- [ ] verify all requirements from Overview are implemented
- [ ] verify edge cases are handled
- [ ] run full test suite: `<project test command>`
- [ ] run e2e tests if project has them: `<project e2e test command>`
- [ ] verify test coverage meets project standard
- [ ] **runtime verification** (where the project is runnable): launch/run the actual application and exercise the changed behavior end-to-end — for a bug-fix plan, reproduce the original symptom and confirm it is gone; for UI work, verify the feature is reachable and visible. If runtime verification is not feasible from this environment, record exactly what was NOT verified so the final report can disclose it — green tests alone are not proof the user-reported problem is solved

### Task N: [Final] Update documentation
- [ ] update README.md if needed
- [ ] update CLAUDE.md if new patterns discovered
- [ ] move this plan to `docs/plans/completed/` (create dir if needed)
- [ ] move the linked design file too, if any. Extract the value with: `grep -E '^\*\*Design:\*\*' <plan-file> | sed 's/^\*\*Design:\*\* *//'`. If the extracted value is empty, the literal placeholder text, or `none`, skip. Otherwise `test -f <design-path>` and `mv <design-path> docs/plans/completed/` — if the file is missing, print a warning and continue.

## Post-Completion
*Items requiring manual intervention or external systems - no checkboxes, informational only*

**Manual verification** (if applicable):
- manual UI/UX testing scenarios
- performance testing under load
- security review considerations

**External system updates** (if applicable):
- consuming projects that need updates after this library change
- configuration changes in deployment systems
- third-party service integrations to verify
```

## step 3: next steps

after creating the file, tell user: "created plan: `docs/plans/YYYY-MM-DD-<task-name>.md`"

then use AskUserQuestion. **CRITICAL: the tool caps options at 4 — never offer more than 4 options in one question.**

```json
{
  "questions": [{
    "question": "Plan created. What's next?",
    "header": "Next step",
    "options": [
      {"label": "Execute with subagents", "description": "Auto-review the plan with the plan-review agent, then run /planning:execute for task-by-task execution with fresh subagents"},
      {"label": "Interactive review", "description": "Open plan in editor for manual annotation and feedback loop"},
      {"label": "Implement", "description": "Implement task by task in this session"},
      {"label": "Done", "description": "No further action"}
    ],
    "multiSelect": false
  }]
}
```

- **Execute with subagents**: do NOT commit the plan. Invoke `/planning:execute <plan-file-path>` directly (it runs the plan-review agent first — see execute's Step 1.5). The execute → sweep chain prompts about committing everything (plan + task changes + sweep fixes) once at the very end.
- **Interactive review**: check if `revdiff` is installed (`which revdiff`).
  - **if revdiff is available**: run `${CLAUDE_PLUGIN_ROOT}/scripts/launch-plan-review.sh <plan-file-path>` via Bash.
    the script opens revdiff TUI showing the plan with syntax highlighting. user adds line-level annotations.
    on quit, annotations are output to stdout in structured format:
    ```
    ## filename:line ( )
    annotation comment text
    ```
    when annotation output is present:
    1. read each annotation — the line number and comment describe what the user wants changed
    2. revise the plan file to address each annotation
    3. run `${CLAUDE_PLUGIN_ROOT}/scripts/launch-plan-review.sh <plan-file-path>` via Bash
    4. repeat until no output (user quit without annotations)
  - **if revdiff is not available**: fall back to `${CLAUDE_PLUGIN_ROOT}/scripts/plan-annotate.py <plan-file-path>` via Bash.
    the script opens a copy of the plan in $EDITOR via terminal overlay. if the user makes annotations,
    it outputs a unified diff to stdout. when diff output is present:
    1. read the diff carefully — added lines (+) are user annotations, removed lines (-) are deletions, modified lines show requested changes
    2. revise the plan file to address each annotation
    3. run `${CLAUDE_PLUGIN_ROOT}/scripts/plan-annotate.py <plan-file-path>` via Bash
    4. repeat until no diff output (user closed editor without changes)
  when the annotation loop completes, ask again with the remaining options (minus "Interactive review")
- **Implement**: begin implementing task 1 interactively in this session. Use TodoWrite tool to track progress and mark todos completed immediately (do not batch). Do NOT commit during implementation. When the user finishes implementing (or pauses), call AskUserQuestion with "Commit all changes now?" (Yes/No). On Yes: `git add -A && git commit -m "<topic>: implement <plan-title>"`. On No: leave uncommitted.
- **Done**: do NOT auto-commit. Call AskUserQuestion: "Plan file is ready. Commit it now?" with options Yes / No. On Yes: `git add <plan-file-path> && git commit -m "docs: add <topic> implementation plan"`. On No: leave the plan uncommitted in `docs/plans/`. Either way, stop after this prompt.

a standalone plan review is still available any time: the user can ask for it in free text (via "Other") and you launch the plan-review agent (Agent tool with subagent_type=planning:plan-review), then re-ask the menu.

## execution enforcement

**CRITICAL testing rules during implementation:**

1. **after completing code changes in a task**:
   - STOP before moving to next task
   - add tests for all new functionality
   - update tests for modified functionality
   - run project test command
   - mark completed items with `[x]` in plan file

2. **if tests fail**:
   - fix the failures before proceeding
   - do NOT move to next task with failing tests
   - do NOT skip test writing

3. **only proceed to next task when**:
   - all task items completed and marked `[x]`
   - tests written/updated
   - all tests passing

4. **plan tracking during implementation**:
   - update checkboxes immediately when tasks complete
   - add ➕ prefix for newly discovered tasks
   - add ⚠️ prefix for blockers
   - modify plan if scope changes significantly

5. **on completion**:
   - verify all checkboxes marked
   - run final test suite
   - create directory if needed: `mkdir -p docs/plans/completed`
   - move plan to `docs/plans/completed/`
   - read the plan header's `Design:` field (parse via `grep -E '^\*\*Design:\*\*' <plan-file> | sed 's/^\*\*Design:\*\* *//'`). If the extracted value is empty, the literal template placeholder, or `none`, skip. Otherwise verify the file exists and move it to `docs/plans/completed/`
   - do NOT commit here. Which commit prompt applies depends on how this plan is being executed:
     - if the user picked "Execute with subagents" from step 3, sweep at the end of `/planning:execute` issues the only commit prompt
     - if the user picked "Implement" from step 3, the in-session prompt described under "Implement" is the only commit prompt

6. **partial implementation exception**:
   - if a task provides partial implementation where tests cannot pass until a later task:
     - still write the tests as part of this task (required)
     - add TODO comment in test code explaining the dependency
     - mark the test checkbox as completed with note: `[x] write tests ... (fails until Task X)`
     - do NOT skip test writing or defer until later
   - when the dependent task completes, remove the TODO comment and verify tests pass

this ensures each task is solid before building on top of it.

## key principles

- **batch related questions, skip settled ones** - group remaining questions into one AskUserQuestion call (up to 4, max 4 options each); never re-ask what a design doc or earlier session answer already settled
- **multiple choice preferred** - easier to answer than open-ended when possible
- **DRY, YAGNI ruthlessly** - avoid unnecessary duplication and features, keep scope minimal (but prefer duplication over premature abstraction when it reduces coupling)
- **lead with recommendation** - have an opinion, explain why, but let user decide
- **explore alternatives** - always propose 2-3 approaches before settling (unless obvious)
- **duplication vs abstraction** - when code repeats, ask user: prefer duplication (simpler, no coupling) or abstraction (DRY but adds complexity)? explain trade-offs before deciding
