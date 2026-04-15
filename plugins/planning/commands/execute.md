---
description: Execute an implementation plan task-by-task using fresh subagents
argument-hint: path to plan file (or leave empty to find in docs/plans/)
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion, Task, TaskCreate, TaskUpdate, TaskList, Skill
---

# Plan Execution

Execute an implementation plan by spawning a fresh `planning:task-executor` subagent for each task. After all tasks complete, run `/code:sweep` for thorough 3-phase code review with fixes.

## MANDATORY: Fresh Subagent Per Task

> **CRITICAL — NON-NEGOTIABLE.** You MUST use the Task tool to spawn a fresh `planning:task-executor` subagent for EVERY task. Do NOT execute task steps yourself in the main conversation. Each task MUST be delegated to a fresh subagent via the Task tool. No exceptions. No shortcuts.

## Execution Process

### Step 1: Locate and Parse the Plan

1. If path provided via `$ARGUMENTS`, use it
2. Otherwise, check `docs/plans/` for plan files (exclude `completed/`). If multiple plans exist, use AskUserQuestion to let user pick
3. Read the plan file and extract:
   - **Header metadata**: Goal, Architecture, Tech Stack
   - **Task list**: All `### Task N:` sections in order
   - **Task count**: Total number of tasks

Report to user:
> "Found N tasks in the implementation plan.
>
> **Goal:** [goal from header]
>
> Starting execution..."

### Step 2: Create Tracking Tasks

Immediately after parsing, create a TaskCreate for each plan task plus a sweep task:

```
TaskCreate({ subject: "Task N: [Component Name]", activeForm: "Executing Task N: [Component Name]" })
```

Also create:
```
TaskCreate({ subject: "Code sweep", activeForm: "Running 3-phase code review + fix..." })
```

### Step 3: Execute Tasks Sequentially

For each task in order:

1. **Mark task `in_progress`** via TaskUpdate
2. **Report start**: "Starting Task N: [Component Name]"
3. **Spawn subagent** (MANDATORY): Use the Task tool with `subagent_type: "planning:task-executor"`. Pass:
   - Full task markdown (files, steps, code snippets)
   - Context about the overall goal
   - Instructions to follow the TDD steps exactly (if plan uses TDD format)
4. **Wait for completion**
5. **Mark task `completed`** via TaskUpdate (or keep `in_progress` on failure)
6. **Report result**

### Step 4: Handle Failures

On task failure:
1. **Stop immediately** — do not proceed to next task
2. **Report**: "Task N failed: [error details]"
3. **Ask user** via AskUserQuestion:

```json
{
  "questions": [{
    "question": "Task N encountered an error. How to proceed?",
    "header": "Failure",
    "options": [
      {"label": "Retry", "description": "Re-run this task with a fresh subagent"},
      {"label": "Skip", "description": "Mark as skipped, continue to next task"},
      {"label": "Stop", "description": "End execution and investigate manually"}
    ],
    "multiSelect": false
  }]
}
```

### Step 5: Code Sweep (MANDATORY)

> **DO NOT SKIP.** After all tasks complete, run the code sweep.

Mark code sweep as `in_progress`. Invoke the sweep skill:

```
Skill("code:sweep", "review all changes on this branch")
```

The sweep skill handles all 3 review phases (comprehensive, smells, critical) and fixes internally. Wait for it to complete, then mark as `completed`.

### Step 6: Final Summary

```
Execution complete!

Summary:
- Tasks completed: N/N
- Code sweep: [sweep results summary]

All tasks from the implementation plan have been executed and reviewed.
```

## Key Rules

- Each subagent gets a fresh context — no accumulated state
- You are the ORCHESTRATOR — never read code, debug, or fix issues yourself
- If a subagent fails, retry with fresh subagent — do NOT investigate yourself

## Progress Reporting

Report progress conversationally only — do NOT modify the plan file during execution.

## Partial Completion

If execution stops mid-way:

```
Execution paused at Task N/M.

Completed: 1-[N-1]
Failed: N
Remaining: [N+1]-M

To resume, fix the issue and run /planning:execute again.
```

Initial request: $ARGUMENTS
