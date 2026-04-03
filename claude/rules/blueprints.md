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

All files use `<epoch>-<slug>.md` where epoch is Unix seconds from `date +%s`.
Example: `1711324800-auth-redesign.md`. No type-specific prefixes.

## Commit-on-Write

After every blueprint file write or move:

```sh
cd "$BLUEPRINTS_DIR" && \
  git add -A "<project>/" && \
  git commit -m "<type>(<project>): <slug>" && \
  git push || (git pull --rebase && git push)
```

- If `git push` fails because no remote is configured, warn the user but do
  not treat it as a blocking error. The commit is still saved locally.
- If rebase fails, STOP and alert the user with conflict details. Do not
  continue — blueprint data may be at risk.

## Plan Mode Integration

When using `EnterPlanMode` / `ExitPlanMode` for non-trivial tasks:

1. Explore the codebase and design the plan as normal during plan mode.
2. Before exiting plan mode, write the plan to the blueprints repo:
   ```sh
   mkdir -p "$BLUEPRINTS_DIR/<project>/plan/"
   ```
   Write the plan as `<epoch>-<slug>.md` using this structure:
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
3. Run commit-on-write after saving the plan.
4. Present the plan to the user via `ExitPlanMode` for approval.
5. After implementation is complete, archive the plan:
   ```sh
   mv "$BLUEPRINTS_DIR/<project>/plan/<file>" \
      "$BLUEPRINTS_DIR/<project>/archive/"
   ```
   Then run commit-on-write.

## Archive Protocol

When a blueprint is consumed by a downstream command (e.g., `/implement`
consumes a spec):

```sh
mkdir -p "$BLUEPRINTS_DIR/<project>/archive/"
mv "$BLUEPRINTS_DIR/<project>/<type>/<file>" \
   "$BLUEPRINTS_DIR/<project>/archive/"
```

Then run commit-on-write.
