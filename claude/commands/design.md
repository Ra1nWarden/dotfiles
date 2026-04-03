---
description: Explore approaches, write a design doc, and get adversarial review
argument-hint: <problem or feature to design>
allowed-tools: Read Glob Grep Bash(ls *) Bash(mkdir *) Bash(cat *) Bash(head *) Bash(tail *) Bash(wc *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(pwd) Bash(which *) Bash(echo *) Bash(tree *)
---

# Design Document Workflow

You are helping a developer think through a design problem. Follow a systematic
approach: understand the codebase, explore multiple approaches, write a design
document, then get it reviewed by a fresh-context adversarial reviewer.

## Core Principles

- **Explore before deciding**: Always investigate at least 2 genuinely different approaches before recommending one
- **Write for a cold reader**: The design doc must stand on its own without verbal context
- **Adversarial review matters**: The reviewer agent must challenge assumptions, not rubber-stamp

---

## Phase 1: Discovery & Codebase Exploration

**Goal**: Understand the problem and existing codebase context

Design request: $ARGUMENTS

**Actions**:
1. If the request is ambiguous, ask the user to clarify scope, constraints, and goals before proceeding
2. Explore the codebase to understand relevant architecture, patterns, and constraints. Launch 2 explorer agents in parallel:
   - Agent 1: Find code, patterns, and architecture relevant to the problem
   - Agent 2: Find similar prior art, related features, or adjacent systems
3. Summarize findings briefly to the user

---

## Phase 2: Approach Exploration

**Goal**: Develop at least 2 genuinely different approaches

**Actions**:
1. Based on discovery, identify at least 2 distinct approaches. They should differ meaningfully — not just minor variations. Consider axes like:
   - Build vs. buy vs. extend existing
   - Simple-now vs. flexible-later
   - Different architectural patterns (event-driven vs. request-response, etc.)
   - Different data models or storage strategies
2. For each approach, think through:
   - How it works (brief technical description)
   - Key trade-offs (complexity, performance, maintainability, risk)
   - What it handles well and what it handles poorly
   - Rough implementation effort
3. Form a recommendation with clear reasoning

---

## Phase 3: Write the Design Document

**Goal**: Produce a structured design doc in `.claude/designs/`

**Actions**:
1. Create the `.claude/designs/` directory in the project root if it does not exist
2. Write the design document to `.claude/designs/<date>-<slug>.md` where `<date>` is today's date (YYYY-MM-DD) and `<slug>` is a short kebab-case topic name
3. Use this structure:

```markdown
# Design: <Title>

**Date**: <YYYY-MM-DD>
**Status**: Draft

## Problem Statement
What problem are we solving and why does it matter?

## Context
Relevant codebase architecture, constraints, and prior art discovered.

## Approach A: <Name>
### Description
How this approach works technically.
### Trade-offs
Pros, cons, risks, effort estimate.

## Approach B: <Name>
### Description
How this approach works technically.
### Trade-offs
Pros, cons, risks, effort estimate.

## Recommendation
Which approach and why. What assumptions does this depend on?
```

4. Tell the user the design doc has been written and briefly summarize it

---

## Phase 4: Adversarial Review

**Goal**: Get a fresh-context agent to challenge the design

**Actions**:
1. Read the design document you just wrote
2. Launch a `design-reviewer` agent. Pass the **full content** of the design doc to the agent. The agent should ONLY receive the doc content — no exploration context from earlier phases.
3. Collect the reviewer's response

---

## Phase 5: Finalize & Present

**Goal**: Integrate review feedback and present to the user

**Actions**:
1. Append the review feedback to the design document under a new section:

```markdown
---

## Review Feedback

**Verdict**: <APPROVE / APPROVE WITH CONCERNS / REQUEST CHANGES>

### Unstated Assumptions
...
### Failure Modes
...
### Simpler Alternatives
...
### Scaling & Evolution
...
### Missing Details
...
### Strongest Concern
...
```

2. Update the document **Status** from "Draft" to "Reviewed"
3. Present to the user:
   - The file path of the design doc
   - Your recommendation (brief)
   - The reviewer's verdict and strongest concern
   - Ask if the user wants to revise anything before moving to implementation
