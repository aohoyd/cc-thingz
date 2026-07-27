---
name: do
description: "Use before any creative work or significant changes. Activates on 'brainstorm', 'let's brainstorm', 'think through', 'help me design', 'explore options', 'I have an idea', 'I want to build', 'how should I approach', 'how should we implement'. Guides collaborative dialogue to turn ideas into designs: collect context, ask questions via AskUserQuestion, explore approaches, present design incrementally."
argument-hint: Describe your idea or feature
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "AskUserQuestion", "Agent", "TaskCreate", "TaskUpdate", "TaskList", "EnterPlanMode", "Skill"]
---

# Brainstorm

Collect context → ask questions → explore approaches → present design.

## Rules

1. **Every response that needs user input ends with an AskUserQuestion tool call** — never a plain-text question or a trailing "let me know". The tool call comes after your visible text, never instead of it: context, summaries, and design sections must appear as visible message text before the call. Thinking blocks are not visible to the user.
2. **Batch related questions into a single AskUserQuestion call** (up to 4). Don't drip-feed one at a time, and don't mix intent questions with edge-case details in the same round.
3. **Never write code, scaffold, or implement.** This skill's only output is a design. Design approval does not grant permission to implement — implementation happens after Phase 5, via whatever the user picks there.
4. **Intent before implementation.** First questions are about scope and goals; edge cases come last.
5. **Anything you ask the user to approve must be visible.** When asking whether a section looks good, the full section must be in your reply as visible message text, and mirrored into the "Looks good" option's `preview` field so the approval UI shows what is being approved. The preview is a backup, never the primary. A bare "does section N look good?" with nothing to judge is a failure — users have blind-approved empty sections four rounds in a row and then discovered the design was wrong.
6. **Free-text corrections are final.** If the user corrects your framing or mapping in an answer, acknowledge it and re-frame — never re-ask a question built on the rejected framing.

## Process

### Phase 0: Initialize Tracking

Before doing anything else, create tasks for all phases so progress is visible and no phase gets skipped:

```
TaskCreate({ subject: "Phase 1: Collect context",       activeForm: "Collecting context" })
TaskCreate({ subject: "Phase 2: Clarify requirements",  activeForm: "Clarifying requirements" })
TaskCreate({ subject: "Phase 3: Explore approaches",    activeForm: "Exploring approaches" })
TaskCreate({ subject: "Phase 4: Present design",        activeForm: "Presenting design" })
TaskCreate({ subject: "Phase 5: Decide next steps",     activeForm: "Deciding next steps" })
```

Mark each phase `in_progress` when you start it, `completed` when done. Phase 5 must be completed before the brainstorm ends.

### Phase 1: Collect Context

Explore the codebase to understand relevant architecture before asking questions, so your questions are informed and specific.

1. **Read project context** — CLAUDE.md, README.md, last 10 git commits
2. **Explore the relevant code** — for a large or unfamiliar area, launch 1-2 `code:explorer` subagents in parallel via Agent tool (e.g. "Find features similar to [idea] and trace their implementation" / "Map the architecture and patterns relevant to [area]"); for a small or well-understood area, explore directly with Read/Grep (faster, no subagent overhead)
3. **For bug reports: reproduce before designing.** If the idea is "X is broken / hangs / doesn't work", attempt to reproduce the failure (run the command, test, or app) before any design work — or ask the user for repro steps. A design built on an unreproduced diagnosis has shipped a confident non-fix before. If reproduction isn't feasible, say so explicitly and prefer approaches that verify the diagnosis at runtime before committing to a fix.
4. **For investigation requests** ("investigate why...", "find ways to make X faster"): gather data first — measurements, traces, comparisons. Don't ask the user scoping/tooling questions before you have exploration results to ground them; that ordering has caused users to abandon the flow.
5. **Anchor on the user's words, not the code.** Summarize findings in 3-5 sentences — no questions in this text — and restate the user's request in their own terms, mapping each term they used to the concrete code element you believe it means.
6. **Never state unverified concrete facts** (key chords, command names, API names, "X is the default") in summaries or question options — grep/read first. A fabricated fact in an option propagates through plan and docs as ground truth.
7. **Proceed directly to Phase 2** — call AskUserQuestion with your first question

Your first question is about scope/intent (what the user wants), not implementation details. When the value of the feature itself is uncertain, ask that first — "is this needed?" is the user's question to answer, not one they should have to raise.

### Phase 2: Ask Clarifying Questions (3-6 rounds)

Each response: 1-3 sentences of context informed by Phase 1 findings, then an AskUserQuestion call. Put several tightly-related questions in the same call when it saves round-trips — just keep each round focused on one progression stage:

- Round 1-2: Intent and scope — WHAT does the user want? What problem does it solve?
- Round 3-4: Behavior and UX — HOW should it feel? Existing patterns to follow?
- Round 5-6: Edge cases and constraints — only now ask implementation details

### Phase 3: Explore Approaches (optional — skip honestly)

This phase exists only when multiple genuinely viable approaches remain after Phase 2. In practice Phase 2's answer options often settle the approach — in that case skip this phase and mark its task `completed` with the note "settled in Phase 2"; don't stage a fake comparison or invent elaborate alternatives to fill the phase (that has produced over-engineered machinery the user replaced with a one-liner).

When it does apply:

**Simple idea?** Present 2-3 approaches yourself with trade-offs. Lead with a recommendation. End with AskUserQuestion to choose.

**Complex idea?** Launch 2-3 `code:architect` subagents in parallel via Agent tool, each exploring a different design angle. Pass each the idea, requirements from Phase 2, and codebase context from Phase 1. Summarize results, lead with a recommendation, end with AskUserQuestion.

Sanity-check every approach against your own Phase 1 findings before presenting it — a design that contradicts evidence you already collected (topology, layout, call graph) fails at first contact with execution.

### Phase 4: Present Design

Break the design into sections of 200-300 words, covering: architecture, components, data flow, error handling, testing. Present each section per Rule 5 — full section as visible message text, then AskUserQuestion ("Looks good" / "Needs changes") with the section mirrored in the preview.

**Surface load-bearing choices as questions, not prose.** Any decision the user will experience directly (a keybinding, what happens at narrow widths, what gets hidden/removed, replacing existing UX) gets its own AskUserQuestion — never buried inside a section's "Looks good?" approval. Bulk approvals rubber-stamp; buried choices have shipped visible regressions. When the design changes or removes something that exists today (a label, a filter, a shortcut), name the removal explicitly and ask.

**When the design depends on the user's runtime environment** (active theme, install/deployment mode, monitor setup), ask — don't validate against your own defaults.

### Phase 5: Next Steps (required)

Reach this phase before the brainstorm ends — design approval in Phase 4 does not mean "start implementing"; always ask the user how to proceed.

**No commits during brainstorm.** Don't run `git commit` or `git add` at any point — not when saving the design, not at hand-off. Any commit prompt happens later inside `/planning:make` or `/code:sweep`.

AskUserQuestion with options: "Write plan" / "Plan mode" / "Start now"

- **Write plan**: save the design to `docs/plans/YYYY-MM-DD-<topic>-design.md` (do not commit it). Then invoke `/planning:make <design-path>`, passing the design file path as the argument so the resulting plan can reference it in its `Design:` header and move both files together on completion.
- **Plan mode**: use EnterPlanMode tool
- **Start now**: implement directly with TaskCreate tracking. Don't commit during implementation; if the user wants a commit at the end, they will ask.

## Key Principles

- **Multiple choice preferred** - easier to answer than open-ended when possible
- **Lead with recommendation** - have an opinion, explain why, but let user decide
- **YAGNI ruthlessly** - remove unnecessary features from all designs, keep scope minimal
- **Be flexible** - go back and clarify when something doesn't make sense
- **Duplication vs abstraction** - when code repeats, ask user: prefer duplication (simpler, no coupling) or abstraction (DRY but adds complexity)? explain trade-offs before deciding
