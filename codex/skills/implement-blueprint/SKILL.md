---
name: implement-blueprint
description: Use when turning an approved blueprint design or plan into code, including dependency graphing, implementation waves, review passes, and final verification.
---

# Implement Blueprint

Use this only after the user approves a design or plan.

## Workflow

1. Resolve the input:
   - Full path to a blueprint file, or
   - Slug under `$BLUEPRINTS_DIR/<project>/spec/` or `plan/`.
2. Read the approved document and identify the recommended approach. Implement that approach only.
3. Build a dependency graph:
   - Task ID
   - Title
   - Files or modules affected
   - Dependencies
   - Scope estimate
4. Present implementation waves when the work is substantial. For small work, proceed directly.
5. Execute each wave:
   - Use subagents only for independent, clearly bounded tasks.
   - Give each subagent concrete ownership and tell it not to commit.
   - For local execution, keep edits scoped and follow repo patterns.
6. After each substantial wave, run review passes with reviewer agents when available:
   - Architecture
   - Readability
   - Correctness
7. Address Critical and Warning findings once unless the user asks for a deeper cycle.
8. Run integration verification: build, tests, typecheck, lint, and browser checks for UI where feasible.
9. Compare the final diff against the approved design and report deviations.
10. If persisting reviews or conformance reports, save them under `$BLUEPRINTS_DIR/<project>/review/` and run commit-on-write.

## Constraints

- Do not deviate from the approved design without flagging it.
- Prefer editing existing files over creating new abstractions.
- If implementation uncovers a major design issue, pause and rescope with the user.
