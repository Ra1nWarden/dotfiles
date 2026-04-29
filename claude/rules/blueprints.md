# Blueprints Convention

Blueprints are persistent, git-tracked design artifacts (specs, plans, reviews)
stored outside of project repos in a dedicated blueprints repository.

## Environment Check

Before any blueprints operation, verify `$BLUEPRINTS_DIR` is set:

```sh
echo "$BLUEPRINTS_DIR"
```

If empty, STOP and tell the user:
> `BLUEPRINTS_DIR` is not set. Run `./install` from your dotfiles repo or add
> `export BLUEPRINTS_DIR=~/path/to/blueprints` to `~/.zshrc.local`, then
> restart your shell.

## Project Derivation

```sh
basename "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null \
  | sed 's|/\.git$||; s|/\.bare$||')" 2>/dev/null || basename "$(pwd)"
```

## Directory Layout

```
$BLUEPRINTS_DIR/<project>/research/   # codebase research briefs (from /research)
$BLUEPRINTS_DIR/<project>/spec/       # design specs (from /design)
$BLUEPRINTS_DIR/<project>/plan/       # implementation plans
$BLUEPRINTS_DIR/<project>/review/     # code review output (from /review)
$BLUEPRINTS_DIR/<project>/archive/    # consumed blueprints (all types)
```

Create on first write: `mkdir -p "$BLUEPRINTS_DIR/<project>/<type>/"`

## Naming

All files use `<timestamp>-<slug>.md` where timestamp is `date +%Y%m%d%H%M`.
Example: `202604031530-auth-redesign.md`. No type-specific prefixes.

## Commit-on-Write

After every blueprint file write or move:

```sh
cd "$BLUEPRINTS_DIR" && \
  git add -A "<project>/" && \
  git commit -m "<type>(<project>): <slug>" && \
  git push || (git pull --rebase && git push)
```

After a successful push, build and present a clickable link to the file on the
remote. Derive the base URL from the remote:

```sh
REMOTE_URL=$(cd "$BLUEPRINTS_DIR" && git remote get-url origin 2>/dev/null)
```

Convert to a browser URL and append the file path on the default branch:
- `git@github.com:owner/repo.git` → `https://github.com/owner/repo/blob/main/<project>/<type>/<file>`
- `https://github.com/owner/repo.git` → `https://github.com/owner/repo/blob/main/<project>/<type>/<file>`

Present the link to the user so they can click to open it in a browser.

- If `git push` fails because no remote is configured, warn the user but do
  not treat it as a blocking error. The commit is still saved locally.
- If rebase fails, STOP and alert the user with conflict details. Do not
  continue — blueprint data may be at risk.

## Plan Workflow

For non-trivial tasks, the plan file must be **written, committed, and
pushed as soon as the plan is generated** — *before* it is presented to
the user for approval. Do not gate the commit on user approval.

Because Claude Code's `EnterPlanMode` blocks file writes, do **not** use
`EnterPlanMode` for the design phase — it would force the commit to
happen only after the user approves via `ExitPlanMode`. Instead,
self-enforce read-only research using `Read`, `Grep`, `Glob`, and
`Explore` subagents.

1. Research the codebase with read-only tools.
2. Draft the plan content.
3. Write the plan to `$BLUEPRINTS_DIR/<project>/plan/<timestamp>-<slug>.md`
   (create the directory on first write):
   ```sh
   mkdir -p "$BLUEPRINTS_DIR/<project>/plan/"
   ```
   Use this structure:
   ```markdown
   # Plan: <Title>

   **Date**: <YYYY-MM-DD>
   **Status**: Proposed

   ## Goal
   What we're trying to accomplish.

   ## Approach
   How we'll do it — key decisions, files affected, patterns used.

   ## Tasks
   Ordered list of implementation steps.
   ```
4. Immediately run commit-on-write to push the plan; capture the remote URL.
5. Present the plan summary and the URL to the user for approval (in chat,
   or via `ExitPlanMode` if you want a formal approval gate — by this point
   the file is already committed, so plan mode no longer blocks anything).
6. Begin implementation only after the user approves.
## Archive Protocol

Blueprints are **not** archived automatically. Only archive when the user
explicitly asks to archive a blueprint:

```sh
mkdir -p "$BLUEPRINTS_DIR/<project>/archive/"
mv "$BLUEPRINTS_DIR/<project>/<type>/<file>" \
   "$BLUEPRINTS_DIR/<project>/archive/"
```

Then run commit-on-write.
