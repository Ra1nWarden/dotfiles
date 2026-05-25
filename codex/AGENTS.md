# Global Codex Instructions

## Working Style

- Read the codebase before changing it. Let existing structure, naming, and tooling drive the implementation.
- Protect user work. Never revert or overwrite changes you did not make unless explicitly asked.
- Use `rg` or `rg --files` for search. Prefer focused reads over dumping large files or logs into context.
- Prefer `apply_patch` for manual edits. Use formatters and code generators only when they are part of the requested change.
- Keep changes scoped to the task. Avoid opportunistic refactors and dependency churn.
- At the beginning of each Codex session, and whenever first entering a folder where you plan to make changes, if the current directory is inside a git repository with a configured remote, run a fast-forward-only pull before editing. Prefer `git pull --ff-only origin master` when `origin/master` exists; otherwise pull the configured upstream or remote default branch, such as `origin/main`. If the pull cannot complete cleanly, stop and report the blocker before making changes.
- For non-trivial work, produce a decision-complete plan before implementation and persist the plan as a blueprint before making code changes. After pushing the committed blueprint to remote, stop and present the GitHub URL so the user can review it before any implementation starts. If Codex Plan Mode is active, stay read-only and present the plan in chat, then persist it as soon as the user switches out of Plan Mode or explicitly asks to save it.
- Track meaningful multi-step work with a short checklist and keep the user informed while working.

## Verification

- After code changes, run the narrowest useful test, build, typecheck, or lint command available.
- If a frontend UI changed, verify it in a browser when feasible. Check the rendered page, interactions, console errors, and responsive layout.
- If verification cannot run, report the exact blocker and the residual risk.

## Blueprints

Use `$BLUEPRINTS_DIR` as the mandatory home for git-tracked research, design specs, implementation plans, and reviews.

- Before any blueprint operation, verify `echo "$BLUEPRINTS_DIR"`.
- For every non-trivial task, create or update the relevant blueprint artifact before implementation:
  - research and codebase mapping go in `research/`
  - design decisions and trade-off analysis go in `spec/`
  - implementation sequencing and task breakdowns go in `plan/`
  - code reviews, conformance checks, and post-implementation assessments go in `review/`
- Do not skip blueprint persistence just because the user did not explicitly ask for it. Skip only for trivial tasks such as typos, one-line obvious fixes, or direct answers that do not create a durable plan, design, research brief, or review.
- If Codex Plan Mode is active, do not write files during that planning turn. Present the blueprint content in chat and persist it after the user switches out of Plan Mode or asks to save it.
- Derive the project from the current git root when possible, otherwise from `basename "$PWD"`.
- Use this layout:
  - `$BLUEPRINTS_DIR/<project>/research/`
  - `$BLUEPRINTS_DIR/<project>/spec/`
  - `$BLUEPRINTS_DIR/<project>/plan/`
  - `$BLUEPRINTS_DIR/<project>/review/`
  - `$BLUEPRINTS_DIR/<project>/archive/`
- Name files `<YYYYMMDDHHMM>-<slug>.md`.
- After every blueprint write or move, commit and push from `$BLUEPRINTS_DIR` so the remote is immediately current.
- After every successful blueprint push, present a clickable GitHub URL for each new or moved blueprint file. Derive the base from `git -C "$BLUEPRINTS_DIR" remote get-url origin`, convert `git@github.com:owner/repo.git` or `https://github.com/owner/repo.git` to `https://github.com/owner/repo`, and append `/blob/<branch>/<project>/<type>/<file>` using the pushed branch, normally `main`. If the blueprint is a plan or design that precedes implementation, stop after presenting the link and wait for explicit user approval before making code changes.
- If the remote branch has moved, pull with rebase and retry the push. If no remote is configured, push fails for any other reason, or rebase conflicts, stop and ask the user.

## Subagents

- Use subagents only when they materially improve the work: parallel research, independent implementation slices, or focused review.
- Keep subagent prompts concrete and bounded. Include only the context needed for the assigned task.
- Summarize subagent results back to the user because they cannot see raw subagent output.
- Do not use subagents for simple single-file edits, short searches, or tightly sequential work.
