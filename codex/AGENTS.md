# Global Codex Instructions

## Working Style

- Read the codebase before changing it. Let existing structure, naming, and tooling drive the implementation.
- Protect user work. Never revert or overwrite changes you did not make unless explicitly asked.
- Use `rg` or `rg --files` for search. Prefer focused reads over dumping large files or logs into context.
- Prefer `apply_patch` for manual edits. Use formatters and code generators only when they are part of the requested change.
- Keep changes scoped to the task. Avoid opportunistic refactors and dependency churn.
- For non-trivial work, produce a decision-complete plan before implementation. If Codex Plan Mode is active, stay read-only until the user switches out of Plan Mode.
- Track meaningful multi-step work with a short checklist and keep the user informed while working.

## Verification

- After code changes, run the narrowest useful test, build, typecheck, or lint command available.
- If a frontend UI changed, verify it in a browser when feasible. Check the rendered page, interactions, console errors, and responsive layout.
- If verification cannot run, report the exact blocker and the residual risk.

## Blueprints

Use `$BLUEPRINTS_DIR` for persistent, git-tracked research, design specs, plans, and reviews when a workflow asks for blueprints.

- Before any blueprint operation, verify `echo "$BLUEPRINTS_DIR"`.
- Derive the project from the current git root when possible, otherwise from `basename "$PWD"`.
- Use this layout:
  - `$BLUEPRINTS_DIR/<project>/research/`
  - `$BLUEPRINTS_DIR/<project>/spec/`
  - `$BLUEPRINTS_DIR/<project>/plan/`
  - `$BLUEPRINTS_DIR/<project>/review/`
  - `$BLUEPRINTS_DIR/<project>/archive/`
- Name files `<YYYYMMDDHHMM>-<slug>.md`.
- After every blueprint write or move, commit-on-write from `$BLUEPRINTS_DIR`.
- If push fails because there is no remote, warn and continue. If rebase conflicts, stop and ask the user.

## Subagents

- Use subagents only when they materially improve the work: parallel research, independent implementation slices, or focused review.
- Keep subagent prompts concrete and bounded. Include only the context needed for the assigned task.
- Summarize subagent results back to the user because they cannot see raw subagent output.
- Do not use subagents for simple single-file edits, short searches, or tightly sequential work.
