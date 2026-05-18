---
description: Research a codebase topic — architecture, APIs, components, and how they connect
argument-hint: <topic to understand, e.g. "auth system", "data pipeline", "API layer">
allowed-tools: Read Glob Grep Bash(ls *) Bash(mkdir *) Bash(cat *) Bash(head *) Bash(tail *) Bash(wc *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(git add*) Bash(git commit*) Bash(git push*) Bash(git pull*) Bash(pwd) Bash(which *) Bash(echo *) Bash(tree *) Bash(date *) Bash(basename *) Bash(cd *)
---

# Research Workflow

You are helping a developer understand a codebase topic. The goal is pure
comprehension — not feature design or implementation planning. Produce a
research brief that can later serve as input to `/design`.

**Never research on the main thread** — subagents do all codebase exploration.
The main thread validates, synthesizes, and presents.

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
3. **Prepare output directory**: `mkdir -p "$BLUEPRINTS_DIR/$PROJECT/research/"`

---

## Phase 1: Scope the Topic

Research topic: $ARGUMENTS

**Actions**:
1. If the topic is too broad (e.g., "everything"), ask the user to narrow it to
   a specific system, layer, or concern.
2. Identify 2–3 concrete questions the research should answer. Present them to
   the user for confirmation before proceeding. Examples:
   - "What components make up the auth system and how do they interact?"
   - "What are the public API endpoints and their request/response shapes?"
   - "How does data flow from ingestion to storage?"

---

## Phase 2: Parallel Exploration

**Spawn 2–3 explorer agents in parallel** (subagent_type=Explore, model=opus),
one per question from Phase 1. Each agent gets:

- The specific question to answer
- Instruction to report: file paths, key types/interfaces, function signatures,
  data flow, and component relationships
- Instruction to return findings as text — do NOT write files

**Depth guidance for agents:**
- Read key files to understand architecture
- Trace call chains 1–2 levels deep for important interfaces
- Include file paths with line references for key definitions
- Map dependencies between components
- Note any conventions, patterns, or non-obvious design decisions

---

## Phase 3: Validate Findings

Before synthesizing, spot-check the agents' claims:

1. Pick 3–5 architectural claims from the agents' findings (e.g., "X calls Y",
   "Z implements interface W", "config lives in path P").
2. For each: Grep or Read a few lines to verify. Do NOT read entire files.
3. If a claim is wrong, note the correction. Do not re-dispatch agents for
   minor errors — just fix them in the synthesis.

---

## Phase 4: Synthesize Research Brief

Write the research brief to `$BLUEPRINTS_DIR/$PROJECT/research/<timestamp>-<slug>.md`
using this structure:

```markdown
# Research: <Topic>

**Date**: <YYYY-MM-DD>

**Project**: <project name>

**Scope**: <1-line summary of what was investigated>

## Overview

High-level summary of the topic area — 2-3 paragraphs max. What is this system,
what problem does it solve, and what is the overall architecture?

## Key Components

For each major component:

### <Component Name>
- **Location**: file paths
- **Role**: what it does
- **Key interfaces**: important types, function signatures, API endpoints
- **Dependencies**: what it depends on, what depends on it

## How It Works

Describe the main flows — how components interact to accomplish the system's
purpose. Use numbered steps for important sequences. Reference file paths and
function names.

## API Surface

Public interfaces, endpoints, exported types, or extension points that other
parts of the codebase (or external consumers) use. Include request/response
shapes or type signatures where relevant.

## Design Decisions & Patterns

Notable architectural choices, conventions, or patterns used in this area.
Note anything non-obvious that a developer new to this code would need to know.
```

---

## Phase 5: Commit and Present

1. **Commit-on-write**: Run the blueprints commit protocol:
   ```sh
   cd "$BLUEPRINTS_DIR" && git add -A "$PROJECT/" && \
     git commit -m "research($PROJECT): <slug>" && \
     git push || (git pull --rebase && git push)
   ```
   If push fails due to no remote, warn but continue.

2. **Present to the user**:
   - The full research brief content (not just a summary — the user wants to
     read and understand)
   - The file path where it was saved
   - Remind: *"This research is available as context for `/design` — reference
     it by slug when you're ready to design a solution."*

---

## Follow-Up Questions

When the user asks follow-up questions after the initial research:

1. **Dispatch a new explorer agent** to investigate the follow-up topic, passing
   the existing research brief as context so it can build on prior findings.
2. **Update the existing research brief** — integrate the new findings into the
   relevant sections (add components, expand flows, update API surface, etc.).
   Do not create a separate file.
3. **Commit-on-write**: Run the blueprints commit protocol after updating.
4. **Present the updated sections** to the user — show what changed, not the
   entire brief again.
