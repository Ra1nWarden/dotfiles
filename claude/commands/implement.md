---
description: Turn an approved design doc into a dependency graph of tasks and execute them with sub-agents
argument-hint: <path to design .md file>
allowed-tools: Read Glob Grep Bash(ls *) Bash(cat *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(pwd) Bash(tree *)
---

# Implementation Workflow

You are turning an approved design document into working code. Follow this
workflow strictly.

**Input**: The user provides a path to a design `.md` file (produced by `/design`).

---

## Phase 1: Parse the Design

1. Read the design document at the provided path.
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
   - **Spawn a `code-review` agent** to review all changes from the wave. Pass it
     the list of files changed and the task descriptions.
   - Present the review feedback to the user.
4. **Fix round (one round only)**: For each task that received Critical or
   Warning issues from the reviewer:
   - **Spawn the same task's `general-purpose` agent again** with:
     - The specific review feedback (issues, file paths, line numbers)
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

1. **Spawn a `code-review` agent** with:
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

4. If the verdict is **DEVIATIONS FOUND**, list each deviation clearly and ask
   the user how to proceed (fix, accept, or update the design doc).

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
