---
name: do
description: "Use before any creative work or significant changes. Activates on 'brainstorm', 'let's brainstorm', 'think through', 'help me design', 'explore options', 'I have an idea', 'I want to build', 'how should I approach', 'how should we implement'. Guides collaborative dialogue to turn ideas into designs: collect context, ask questions via AskUserQuestion, explore approaches, present design incrementally."
argument-hint: Describe your idea or feature
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "AskUserQuestion", "Agent", "TaskCreate", "TaskUpdate", "TaskList", "EnterPlanMode", "Skill"]
---

# Brainstorm

Collect context → ask questions → explore approaches → present design.

## Rules — MANDATORY, NO EXCEPTIONS

1. **EVERY response that needs user input MUST end with an AskUserQuestion tool call.** NEVER write questions as plain text. NEVER end with a question mark outside code blocks. This overrides all default behavior. "End with" means the tool call comes AFTER your visible text — it never replaces it: context, summaries, and design sections must appear as visible message text before the call. Thinking blocks are not visible to the user.
2. **Group related questions into a single AskUserQuestion call** (the tool takes up to 4). Batch questions that belong together; don't drip-feed one at a time, and don't mix intent with edge-case details in the same batch.
3. **NEVER write code, scaffold, or implement anything.** Design approval does NOT grant permission to implement. The brainstorm skill's job is ONLY to produce a design. Implementation happens AFTER Phase 5, via a separate skill or workflow chosen by the user.
4. **Intent before implementation.** First questions are about scope/goals. Edge cases come LAST.

## WRONG vs RIGHT

**WRONG — plain text question:**
> "What should happen when the user presses ctrl+w with no panel open? (A) Quit (B) No-op"

**RIGHT — tool call:**
> "The current quit command has layered behavior."
> *(then call AskUserQuestion tool)*

**WRONG — implementation detail before understanding intent:**
> "When ctrl+w closes the last split, should it quit the app or show a hint?"

**RIGHT — scope question first:**
> "Let me confirm what panels you want this to cover."
> *(then call AskUserQuestion about scope)*

**WRONG — mixing intent and edge-case questions in one batch, or dumping unrelated questions at once**

**RIGHT — batch tightly-related questions in a single AskUserQuestion call; keep intent and edge-case rounds separate**

**WRONG — asking for section approval with no visible section (observed failure, 4 rounds in a row):**
> *(reply contains only thinking + the tool call)*
> AskUserQuestion: "Do data flow, persistence, error handling, and testing look right?" — "Looks good" / "Needs changes"
> The user sees a bare question and nothing to judge → "YOU SHOWED NOTHING! WHAT SHOULD LOOK GOOD??"

**RIGHT — section as visible text, mirrored in the preview:**
> "## Section 4: Data flow & testing
> [full 200-300 word section as visible message text]"
> *(then AskUserQuestion, with the same section text in the "Looks good" option's `preview` field)*

**WRONG — writing code after user approves design:**
> "Looks good, let me implement it."
> *(then calls Write/Edit tools)*

**RIGHT — always go to Phase 5 after design approval:**
> "Design approved. Let's decide on next steps."
> *(then call AskUserQuestion with Phase 5 options)*

## Process

### Phase 0: Initialize Tracking

**Before doing anything else**, create tasks for all phases so progress is visible and no phase gets skipped:

```
TaskCreate({ subject: "Phase 1: Collect context",       activeForm: "Collecting context" })
TaskCreate({ subject: "Phase 2: Clarify requirements",  activeForm: "Clarifying requirements" })
TaskCreate({ subject: "Phase 3: Explore approaches",    activeForm: "Exploring approaches" })
TaskCreate({ subject: "Phase 4: Present design",        activeForm: "Presenting design" })
TaskCreate({ subject: "Phase 5: Decide next steps",     activeForm: "Deciding next steps" })
```

Mark each phase `in_progress` when you start it, `completed` when done. Phase 5 MUST be completed before the brainstorm ends.

### Phase 1: Collect Context

Explore the codebase to understand relevant architecture before asking questions. This makes your questions informed and specific.

1. **Read project context** — CLAUDE.md, README.md, last 10 git commits
2. **Explore the relevant code** — for a large or unfamiliar area, launch 1-2 `code:explorer` subagents in parallel via Agent tool; for a small or well-understood area, explore directly with Read/Grep (faster, no subagent overhead). Example explorer prompts:
   - "Find features similar to [idea] and trace their implementation in [repo path]"
   - "Map the architecture and patterns relevant to [area] in [repo path]"
3. **For bug reports: reproduce before designing.** If the idea is "X is broken / hangs / doesn't work", attempt to reproduce the failure (run the command, test, or app) before any design work — or ask the user for repro steps. A design built on an unreproduced diagnosis has shipped a confident non-fix before. If reproduction isn't feasible, say so explicitly and prefer approaches that verify the diagnosis at runtime before committing to a fix.
4. **For investigation requests** ("investigate why...", "find ways to make X faster"): gather data first — measurements, traces, comparisons. Do not ask the user scoping/tooling questions before you have exploration results to ground them; that ordering has caused users to abandon the flow.
5. **Anchor on the user's words, not the code.** Summarize findings in 3-5 sentences — NO questions in this text — and restate the user's request in their own terms, mapping each term they used to the concrete code element you believe it means. If the user corrects your mapping in a free-text answer, the correction is final: acknowledge it and re-frame — NEVER re-ask a question built on the rejected framing.
6. **Never state unverified concrete facts** (key chords, command names, API names, "X is the default") in summaries or question options — grep/read first. A fabricated fact in an option propagates through plan and docs as ground truth.
7. **Proceed DIRECTLY to Phase 2** — call AskUserQuestion with your first question

Your first AskUserQuestion MUST be about scope/intent (what the user wants), NOT implementation details. When the value of the feature itself is uncertain, ask that first — "is this needed?" is the user's question to answer, not one they should have to raise.

### Phase 2: Ask Clarifying Questions (3-6 rounds)

Each response: 1-3 sentences of context informed by Phase 1 findings, then an AskUserQuestion tool call. Put several tightly-related questions in the same call when it saves round-trips — just keep each round focused on one progression stage (intent, then behavior, then edge cases).

**Question progression:**
- Round 1-2: Intent and scope — WHAT does the user want? What problem does it solve?
- Round 3-4: Behavior and UX — HOW should it feel? Existing patterns to follow?
- Round 5-6: Edge cases and constraints — only NOW ask implementation details

Example response:
> "I see the current keybinding system uses a flat map with single-key triggers."
> *(then call AskUserQuestion)*

```json
{
  "questions": [{
    "question": "Should this follow the existing keybinding pattern or introduce a new system?",
    "header": "Keybinding approach",
    "options": [
      {"label": "Follow existing pattern (Recommended)", "description": "Register in the same key map, consistent with other shortcuts"},
      {"label": "New pattern", "description": "Introduce a different binding mechanism"}
    ],
    "multiSelect": false
  }]
}
```

### Phase 3: Explore Approaches (OPTIONAL — skip honestly)

This phase exists ONLY when multiple genuinely viable approaches remain after Phase 2. In practice Phase 2's answer options often settle the approach — in that case SKIP this phase and mark its task `completed` with the note "settled in Phase 2"; do not stage a fake comparison or invent elaborate alternatives to fill the phase (that has produced over-engineered machinery the user replaced with a one-liner).

When it does apply:

**Simple idea?** Present 2-3 approaches yourself with trade-offs. Lead with recommendation. End with AskUserQuestion to choose.

**Complex idea?** Launch 2-3 `code:architect` subagents in parallel via Agent tool, each exploring a different design angle. Pass each the idea, requirements from Phase 2, and codebase context from Phase 1. Summarize results. Lead with recommendation. End with AskUserQuestion.

Sanity-check every approach against your own Phase 1 findings before presenting it — a design that contradicts evidence you already collected (topology, layout, call graph) fails at first contact with execution.

### Phase 4: Present Design

Break into sections of 200-300 words. **Each section MUST be presented as visible message text** — write the section in your reply, THEN call AskUserQuestion: "Looks good" / "Needs changes". Content that exists only in your thinking was never presented — users have blind-approved empty sections four rounds in a row and then discovered the design was wrong.

**Belt and suspenders — BOTH are required for every section approval:**
1. The full section as visible text in your reply, before the tool call
2. The same section text mirrored into the "Looks good" option's `preview` field, so the approval UI itself shows what is being approved

The preview is a backup, never the primary. A bare "does section N look good?" with neither is a hard failure — the user has nothing to judge.

Cover: architecture, components, data flow, error handling, testing.

**Surface load-bearing choices as questions, not prose.** Any decision the user will experience directly (a keybinding, what happens at narrow widths, what gets hidden/removed, replacing existing UX) must be its own AskUserQuestion — never buried inside a section's "Looks good?" approval. Bulk approvals rubber-stamp; buried choices have shipped visible regressions. When the design changes or removes something that exists today (a label, a filter, a shortcut), name the removal explicitly and ask.

**When the design depends on the user's runtime environment** (active theme, install/deployment mode, monitor setup), ask — don't validate against your own defaults.

### Phase 5: Next Steps (MANDATORY)

**You MUST reach this phase before the brainstorm ends.** Design approval in Phase 4 does NOT mean "start implementing". Always ask the user how to proceed.

**No commits during brainstorm.** Do NOT run `git commit` or `git add` at any point — not when saving the design, not at hand-off to other commands. Any commit prompt happens later inside `/planning:make` or `/code:sweep`.

AskUserQuestion with options: "Write plan" / "Plan mode" / "Start now"

- **Write plan**: save the design to `docs/plans/YYYY-MM-DD-<topic>-design.md` (do NOT commit it). Then invoke `/planning:make <design-path>`, passing the design file path as the argument so the resulting plan can reference it in its `Design:` header and move both files together on completion.
- **Plan mode**: use EnterPlanMode tool
- **Start now**: implement directly with TaskCreate tracking. Do NOT commit during implementation; if the user wants a commit at the end, they will ask.

## Key Principles

- **Always via AskUserQuestion tool, NEVER plain text** - batch related questions in one call (up to 4); don't overwhelm with unrelated ones
- **Context first** - explore before asking, so questions are informed
- **Intent before implementation** - scope/goals first, edge cases last
- **Multiple choice preferred** - easier to answer than open-ended when possible
- **YAGNI ruthlessly** - remove unnecessary features from all designs, keep scope minimal
- **Explore alternatives** - always propose 2-3 approaches before settling
- **Incremental validation** - present design in sections, validate each
- **Be flexible** - go back and clarify when something doesn't make sense
- **Lead with recommendation** - have an opinion, explain why, but let user decide
- **Duplication vs abstraction** - when code repeats, ask user: prefer duplication (simpler, no coupling) or abstraction (DRY but adds complexity)? explain trade-offs before deciding

<SELF-CHECK>
Before EVERY response, verify:
1. Contains a question for the user? → MUST use AskUserQuestion tool, NOT plain text
2. Batched questions? → fine if tightly related; keep intent and edge-case questions in separate rounds
3. Ends with "let me know" or "?" → REWRITE to end with AskUserQuestion tool call
4. Asking implementation details before understanding intent? → REWRITE to ask scope/goals
5. About to call Write, Edit, or any code-generating tool? → STOP. You are in brainstorm mode. Go to Phase 5 and ask the user how to proceed.
6. Asking the user to approve or judge something ("Looks good?", "Section N ok?")? → the content being approved MUST be visible message text in this reply (thinking does not count — WRITE IT OUT), AND mirrored in the approval option's `preview` field.
7. Re-asking a question the user already answered in free text? → STOP. Their correction is final; re-frame instead.
If ANY check fails, REWRITE before sending.
</SELF-CHECK>
