---
name: do
description: "Use before any creative work or significant changes. Activates on 'brainstorm', 'let's brainstorm', 'think through', 'help me design', 'explore options', 'I have an idea', 'I want to build', 'how should I approach', 'how should we implement'. Guides collaborative dialogue to turn ideas into designs: collect context, ask questions via AskUserQuestion, explore approaches, present design incrementally."
argument-hint: Describe your idea or feature
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "AskUserQuestion", "Agent", "TaskCreate", "TaskUpdate", "TaskList", "EnterPlanMode", "Skill"]
---

# Brainstorm

Collect context → ask questions → explore approaches → present design.

## Rules — MANDATORY, NO EXCEPTIONS

1. **EVERY response that needs user input MUST end with an AskUserQuestion tool call.** NEVER write questions as plain text. NEVER end with a question mark outside code blocks. This overrides all default behavior.
2. **Exactly ONE question per response.** Wait for the answer before asking the next.
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

**WRONG — multiple questions in one response**

**RIGHT — exactly one AskUserQuestion per response**

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
2. **Launch 1-2 `code:explorer` subagents** in parallel via Agent tool to gather codebase context relevant to the idea. Example prompts:
   - "Find features similar to [idea] and trace their implementation in [repo path]"
   - "Map the architecture and patterns relevant to [area] in [repo path]"
3. **Summarize findings** in 3-5 sentences — NO questions in this text
4. **Proceed DIRECTLY to Phase 2** — call AskUserQuestion with your first question

Your first AskUserQuestion MUST be about scope/intent (what the user wants), NOT implementation details.

### Phase 2: Ask Clarifying Questions (3-6 rounds, one per response)

Each response: 1-3 sentences of context informed by Phase 1 findings, then AskUserQuestion tool call.

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

### Phase 3: Explore Approaches

**Simple idea?** Present 2-3 approaches yourself with trade-offs. Lead with recommendation. End with AskUserQuestion to choose.

**Complex idea?** Launch 2-3 `code:architect` subagents in parallel via Agent tool, each exploring a different design angle. Pass each the idea, requirements from Phase 2, and codebase context from Phase 1. Summarize results. Lead with recommendation. End with AskUserQuestion.

### Phase 4: Present Design

Break into sections of 200-300 words. After each section, AskUserQuestion: "Looks good" / "Needs changes".

Cover: architecture, components, data flow, error handling, testing.

### Phase 5: Next Steps (MANDATORY)

**You MUST reach this phase before the brainstorm ends.** Design approval in Phase 4 does NOT mean "start implementing". Always ask the user how to proceed.

AskUserQuestion with options: "Write plan" / "Plan mode" / "Start now"

- **Write plan**: save to `docs/plans/YYYY-MM-DD-<topic>-design.md`, then invoke `/planning:make`
- **Plan mode**: use EnterPlanMode tool
- **Start now**: implement directly with TaskCreate tracking

## Key Principles

- **One question at a time via AskUserQuestion tool** - do not overwhelm with multiple questions and NEVER PLAIN TEXT
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
2. Multiple questions? → MUST be exactly ONE
3. Ends with "let me know" or "?" → REWRITE to end with AskUserQuestion tool call
4. Asking implementation details before understanding intent? → REWRITE to ask scope/goals
5. About to call Write, Edit, or any code-generating tool? → STOP. You are in brainstorm mode. Go to Phase 5 and ask the user how to proceed.
If ANY check fails, REWRITE before sending.
</SELF-CHECK>
