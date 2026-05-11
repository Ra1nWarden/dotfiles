---
name: archive-blueprint
description: Use when the user asks to archive a blueprint by moving a research, spec, plan, or review artifact into the project archive folder.
---

# Archive Blueprint

Move active blueprint files into `$BLUEPRINTS_DIR/<project>/archive/`.

## Workflow

1. Use the `blueprints` skill checks.
2. If no slug or filename is provided, list active markdown files under the project, excluding `archive/`, and ask the user which to archive.
3. If a slug is provided, search active folders for matching files.
4. If exactly one file matches, create the archive directory and move it there.
5. If multiple files match, ask the user to choose.
6. Run commit-on-write with message `archive(<project>): <slug>`.
7. Report original path, archive path, and remote link when available.

Do not archive automatically after implementation. Archive only when explicitly requested.
