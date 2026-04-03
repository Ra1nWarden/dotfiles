---
description: Turn an approved design doc into a dependency graph of tasks and execute them with sub-agents
argument-hint: <path to design .md file, or slug name to look up in blueprints>
allowed-tools: Read Glob Grep Bash(ls *) Bash(cat *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(git add*) Bash(git commit*) Bash(git push*) Bash(git pull*) Bash(pwd) Bash(tree *) Bash(mv *) Bash(date *) Bash(basename *) Bash(cd *)
---

# Implementation Workflow

You are turning an approved design document into working code. Follow this
workflow strictly.

**Input**: The user provides either:
- A full path to a design `.md` file
- A slug name (e.g., `auth-redesign`) to look up in the blueprints repository

---

## Phase 0: Environment Setup

Before any other work:

1. **Check `$BLUEPRINTS_DIR`**: Run `echo "$BLUEPRINTS_DIR"`. If empty, STOP and
   tell the user: *"BLUEPRINTS_DIR is not set. Run `./install` from your dotfiles
   repo or add `export BLUEPRINTS_DIR=~/path/to/blueprints` to `~/.zshrc.local`,
   then restart your shell."*
2. **Derive project name**:
   ```sh
   PROJECT=$(basename "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null \
     | sed 's|/\.git$||; s|/\.bare$||')" 2>/dev/null || basename "$(pwd)")
   ```

---

## Phase 1: Parse the Design

1. **Resolve the input**:
   - If the user provided a full file path, use it directly.
   - If the user provided a slug, look it up:
     `ls "$BLUEPRINTS_DIR/$PROJECT/spec/"*"$SLUG"*`
     If multiple matches, show them and ask the user to pick one.
2. Read the design document at the resolved path.
2. Identify the **recommended approach** — only implement that approach, not alternatives.
3. Extract the key components, modules, or changes that need to be built.

---

## Phase 2: Build a Dependency Graph

Break the implementation into discrete, parallelizable tasks. For each task:

- **ID**: Short identifier (e.g., `T1`, `T2`)
- **Title**: What this task produces
- **Description**: Specific files to create/modify, functions to implement, configs to change
- **Depends on**: List of task IDs that must complete before this one can start
- **Estimated scope**: Small (single file edit) / Medium (few files) / Large (cross-cutting)

Present the dependency graph to the user as a table or list. Group tasks into
**waves** — a wave is a set of tasks with no dependencies on each other (can
run in parallel). Example:

```
Wave 1 (parallel): T1, T2, T3    — no dependencies
Wave 2 (parallel): T4, T5        — depend on T1
Wave 3 (sequential): T6          — depends on T4 and T5
```

**Wait for user approval before proceeding.** The user may reorder, split,
merge, or remove tasks.

---

## Phase 3: Execute Tasks

For each wave, execute all tasks in parallel using sub-agents:

1. **Spawn one `general-purpose` agent per task** in the wave. Each agent gets:
   - The full task description (ID, title, what to build)
   - Relevant context from the design doc (only the sections it needs)
   - Specific file paths and function signatures it should produce
   - Instruction to commit nothing — only make file changes
2. **Wait for all agents in the wave to complete** before starting the next wave.
3. After each wave completes:
   - Summarize what each agent produced (files changed, key decisions)
   - **Spawn three review agents in parallel**, each receiving the list of
     files changed and the task descriptions:
     - `architect-review` — structural problems, component boundaries,
       dependency hygiene
     - `readability-review` — language best practices, style guide conformance,
       naming, clarity
     - `correctness-review` — edge cases, unhappy paths, logic errors, missing
       unit tests
   - Present all three reviews to the user, grouped by reviewer.
   - **Persist reviews**: If `$BLUEPRINTS_DIR` is set, save the combined review
     output for the wave to
     `$BLUEPRINTS_DIR/$PROJECT/review/<timestamp>-wave-<N>-<slug>.md`
     and run commit-on-write.
4. **Fix round (one round only)**: Collect all Critical and Warning issues
   from all three reviewers. For each task that received issues:
   - **Spawn the same task's `general-purpose` agent again** with:
     - The combined review feedback from all three reviewers (issues, file
       paths, line numbers)
     - Instruction to address only the flagged issues — no other changes
     - The original task description for context
   - Wait for all fix agents to complete.
   - Briefly summarize what was fixed. Do **not** run another review cycle —
     one round of fixes is the limit.
5. Use `TaskCreate` and `TaskUpdate` to track progress through waves.

---

## Phase 4: Integration & Verification

After all waves are complete:

1. Run a build/compile check to catch errors across the combined changes.
2. Run tests if a test suite exists.

---

## Phase 5: Design Conformance Review

After integration passes, perform a final review of **all** code changes against
the original design document.

1. **Spawn an `architect-review` agent** with:
   - The full design document content (the recommended approach section)
   - A complete list of every file created or modified across all waves
   - Instruction to review for **design conformance only** (not code quality —
     that was handled in Phase 3)
2. The agent must evaluate:
   - **API conformance**: Do endpoints, function signatures, data models, and
     interfaces match the design spec exactly? Flag any mismatches.
   - **Architecture adherence**: Were the key architecture decisions (patterns,
     data flow, component boundaries) implemented as designed?
   - **Scope creep**: Were any features, abstractions, utilities, or behaviors
     added that are **not described** in the design doc? Flag anything that was
     not explicitly called for.
   - **Completeness**: Is anything described in the design doc missing from the
     implementation? Are there TODO comments or stub implementations that should
     have been filled in?
   - **Deviations**: If any deviation from the design was necessary, is it
     justified by a technical constraint discovered during implementation?
3. Present the agent's findings to the user as a final judgement:

```
## Design Conformance Report

### API Conformance: PASS / FAIL
<details>

### Architecture Adherence: PASS / FAIL
<details>

### Scope Creep: NONE DETECTED / ISSUES FOUND
<details>

### Completeness: COMPLETE / GAPS FOUND
<details>

### Overall Verdict: CONFORMANT / DEVIATIONS FOUND
<one-paragraph summary of how faithfully the implementation matches the design>
```

4. **Persist conformance report**: If `$BLUEPRINTS_DIR` is set, save the report
   to `$BLUEPRINTS_DIR/$PROJECT/review/<timestamp>-conformance-<slug>.md` and run
   commit-on-write.
5. If the verdict is **DEVIATIONS FOUND**, list each deviation clearly and ask
   the user how to proceed (fix, accept, or update the design doc).

---

## Phase 6: Archive Blueprint

After successful implementation and conformance review:

1. Move the consumed spec to the archive:
   ```sh
   mkdir -p "$BLUEPRINTS_DIR/$PROJECT/archive/"
   mv "$BLUEPRINTS_DIR/$PROJECT/spec/<file>" "$BLUEPRINTS_DIR/$PROJECT/archive/"
   ```
2. **Commit-on-write**: Run the blueprints commit protocol:
   ```sh
   cd "$BLUEPRINTS_DIR" && git add -A "$PROJECT/" && \
     git commit -m "archive($PROJECT): <slug>" && \
     git push || (git pull --rebase && git push)
   ```
3. Inform the user the blueprint has been archived.

---

## Guidelines

- **Do not deviate from the design doc** without flagging it to the user first.
- **Keep sub-agent prompts specific** — include file paths, function names, and
  expected behavior. Vague prompts produce vague code.
- **Prefer editing existing files** over creating new ones.
- If a task turns out to be more complex than estimated, pause and re-scope with
  the user rather than producing half-finished work.
- Each sub-agent should run in a worktree (`isolation: "worktree"`) when tasks
  in the same wave edit overlapping files. For non-overlapping changes, run
  without isolation for speed.
