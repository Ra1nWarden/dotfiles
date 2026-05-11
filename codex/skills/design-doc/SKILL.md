---
name: design-doc
description: Use when the user asks to design a feature, compare implementation approaches, or produce a blueprint design spec before coding.
---

# Design Doc

Use this for non-trivial design before implementation.

## Workflow

1. Use `blueprints` if the design should be persisted.
2. Research the codebase with read-only tools before recommending an approach.
3. Check `$BLUEPRINTS_DIR/<project>/research/` for relevant prior research when available.
4. Explore at least two meaningfully different approaches unless the task is already constrained to one.
5. Recommend one approach with concrete trade-offs.
6. Write a design document:
   - `# Design: <Title>`
   - Date and status
   - Problem statement
   - Context
   - Approach A
   - Approach B
   - Recommendation
7. If persisting, save to `$BLUEPRINTS_DIR/<project>/spec/<timestamp>-<slug>.md` and run commit-on-write.
8. If a design-reviewer agent is available, ask it for adversarial review. Append the review feedback and commit the update.

## Plan Mode Constraint

In Codex Plan Mode, do not write files. Present the complete plan or design in chat and persist it only after the user switches out of Plan Mode.
