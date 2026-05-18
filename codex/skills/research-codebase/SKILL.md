---
name: research-codebase
description: Use when the user asks to research, map, or understand a codebase topic before design or implementation, especially when the result should be saved as a blueprint research brief.
---

# Research Codebase

Use this for comprehension only. Do not design or implement.

## Workflow

1. Use the `blueprints` skill if the research should be persisted.
2. Narrow broad topics into 2-3 concrete research questions. Ask only if scope cannot be inferred from the repo.
3. Explore with read-only tools. Prefer `rg`, `rg --files`, focused file reads, and git history when useful.
4. If subagents are available and the questions are independent, run them in parallel. Otherwise research locally.
5. Spot-check important claims before synthesis.
6. Write a concise research brief with:
   - `# Research: <Topic>`
   - Date, project, and scope
   - Overview
   - Key components with file paths
   - Main flows
   - Public API surface
   - Design decisions and patterns
7. If persisting, save to `$BLUEPRINTS_DIR/<project>/research/<timestamp>-<slug>.md` and run commit-and-push-on-write.

## Follow-ups

For follow-up questions, update the existing research brief when one is clearly in scope. Otherwise create a new brief.
