---
name: blueprints
description: Use when working with persistent planning artifacts in $BLUEPRINTS_DIR, including research briefs, design specs, implementation plans, code reviews, and archived blueprints.
---

# Blueprints

Blueprints are git-tracked artifacts stored outside project repos.

## Required Checks

Before any blueprint operation:

1. Run `echo "$BLUEPRINTS_DIR"`.
2. If empty, stop and tell the user to set `BLUEPRINTS_DIR` in `~/.zshrc.local`.
3. Derive the project name from the current git root when possible:
   `basename "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null | sed 's|/\\.git$||; s|/\\.bare$||')" 2>/dev/null || basename "$(pwd)"`.

## Layout

Use `$BLUEPRINTS_DIR/<project>/` with these folders:

- `research/` for codebase research briefs
- `spec/` for design documents
- `plan/` for implementation plans
- `review/` for review output and conformance reports
- `archive/` for consumed blueprints

Create folders on first write. File names are `<YYYYMMDDHHMM>-<slug>.md`.

## Commit-and-Push-on-Write

After every blueprint write or move:

```sh
cd "$BLUEPRINTS_DIR" &&
git add -A "<project>/" &&
git commit -m "<type>(<project>): <slug>" &&
git push || (git pull --rebase && git push)
```

If the first push fails because the remote branch moved, pull with rebase and retry the push. If no remote is configured, push fails for any other reason, or rebase conflicts, stop and ask the user.

When push succeeds, derive a GitHub-style browser URL from `git remote get-url origin` and present it to the user.

## Plan Mode Constraint

If Codex Plan Mode is active, do not write blueprint files during the planning turn. Produce the plan in chat. Once the user switches out of Plan Mode and asks for implementation or persistence, write, commit, and push the blueprint.
