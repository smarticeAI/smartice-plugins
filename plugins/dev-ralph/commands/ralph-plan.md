---
description: "Start Ralph planning phase with in-depth interview to create specs and implementation plan"
argument-hint: "[task-description]"
allowed-tools: ["Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-plan.sh *)", "Read", "Write", "Glob", "Grep", "AskUserQuestion", "EnterPlanMode"]
---

# Ralph Planning Phase

Task description: $ARGUMENTS

## Step 1: Initialize

Run the setup script:
```bash
${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-plan.sh
```

## Step 2: In-Depth Interview

Interview me in detail using the AskUserQuestion tool about literally anything:
- Technical implementation details
- UI & UX preferences
- Concerns and constraints
- Tradeoffs and priorities
- Edge cases
- Error handling approaches
- Performance requirements
- Security considerations

**Interview Guidelines:**
- **Ask 4 questions at a time** using the AskUserQuestion tool's multi-question capability for efficient interviews
- **Make sure the questions are not obvious.** Dig deep. Ask follow-up questions based on my answers.
- Continue interviewing until you have a complete understanding of what I want to build.
- Do not stop after a fixed number of questions. Keep asking until the spec is fully fleshed out.

## Step 2.5: stdlib Requirements

Ask about reusable utilities/patterns the project needs:

```
Question: What reusable utilities/patterns does this project need?

Examples:
- Error handling utilities (Result types, error classes)
- API client wrappers
- Validation helpers
- Common data transformations
- Logging/telemetry
- Testing utilities

Options:
- Yes, define stdlib modules now
- No, add them during build as needed
- Skip stdlib (use framework defaults)
```

If "Yes", interview about each stdlib module:
- Module name (e.g., "errors", "validation", "api-client")
- Purpose (what problem does it solve?)
- Key functions/classes needed
- Patterns to follow (e.g., "always return Result, never throw")

For each module, create `specs/stdlib/[module-name].md` using the template from `${CLAUDE_PLUGIN_ROOT}/templates/stdlib-spec.md.template`

**Why stdlib matters:**
- Single source of truth for patterns
- Features import from stdlib, don't duplicate utilities
- When Ralph generates wrong patterns, update stdlib spec → Ralph regenerates

## Step 3: Write Specs

After the interview is complete, create:

1. **`.ralph/specs/stdlib/*.md`** - stdlib module specs (if defined in Step 2.5)
2. **`.ralph/specs/*.md`** - One spec file per major feature/concern
3. **`.ralph/IMPLEMENTATION_PLAN.md`** - Prioritized task list with:
   - Phase 0: Project setup
   - Phase 1: stdlib modules (build first!)
   - Phase 2+: Features (use stdlib)
4. **`.ralph/PROMPT.md`** - Using template from `${CLAUDE_PLUGIN_ROOT}/templates/PROMPT.md.template`

Each feature spec should include:
- Requirements
- Acceptance criteria
- Edge cases
- Dependencies
- Verification checklist

Each stdlib spec should follow `${CLAUDE_PLUGIN_ROOT}/templates/stdlib-spec.md.template`

## Step 4: Confirm

Display the checklist gate:
```
Planning Complete!
═══════════════════

[x] .ralph/specs/stdlib/*.md   - N stdlib modules (or skipped)
[x] .ralph/specs/*.md          - N feature specs
[x] IMPLEMENTATION_PLAN.md     - stdlib as Phase 1, features after
[x] PROMPT.md                  - Configured
```

## Step 5: Implementation Path

Ask the user how they want to proceed using AskUserQuestion:

```
Question: How would you like to proceed with implementation?

Options:
1. "Enter Plan Mode" - Refine the plan with codebase-specific details (file paths, existing patterns, architectural considerations). Claude will analyze the codebase and improve the implementation plan before coding begins.

2. "Run /ralph-build" - Proceed directly to autonomous implementation. Ralph will follow the specs and IMPLEMENTATION_PLAN.md as-is.

3. "Exit and review" - End planning here. Review the generated specs manually and run /ralph-build later.
```

**If user chooses "Enter Plan Mode":**
1. Use the EnterPlanMode tool to transition into plan mode
2. In plan mode, Claude will:
   - Read `.ralph/IMPLEMENTATION_PLAN.md` as the starting context
   - Analyze the actual codebase for existing patterns and file structure
   - Refine the plan with specific file paths, imports, and integration points
   - Present the improved plan for user approval
3. After approval, implementation can begin (either via /ralph-build or direct coding)

**If user chooses "Run /ralph-build":**
Display: `Ready: /ralph-build`

**If user chooses "Exit and review":**
Display:
```
Planning artifacts saved to .ralph/
- Review specs in .ralph/specs/
- Review implementation plan in .ralph/IMPLEMENTATION_PLAN.md
- When ready, run: /ralph-build
```
