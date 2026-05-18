---
name: review-change
description: Use when the user asks for a code review of a commit, branch, pull request, or current diff, especially with architecture, readability, and correctness review passes.
---

# Review Change

Use a code-review stance: findings first, ordered by severity, with file and line references.

## Workflow

1. Identify the target:
   - Commit: `git show --stat <hash>` then `git show <hash>`
   - PR: use GitHub tooling when available, otherwise `gh pr view` and `gh pr diff`
   - Branch: compare against the main branch
   - No target: review the current diff
2. Collect changed files, intent, and the relevant diff.
3. Run focused review passes:
   - Architecture: boundaries, dependencies, API shape
   - Correctness: edge cases, unhappy paths, tests, security boundaries
   - Readability: style, naming, idioms, maintainability
4. Use custom reviewer agents when available. If not, perform the passes locally.
5. Present:
   - Findings first, with severity and file references
   - Open questions or assumptions
   - Brief change summary only after findings
6. If persisting, save to `$BLUEPRINTS_DIR/<project>/review/<timestamp>-<slug>.md` and run commit-and-push-on-write.
