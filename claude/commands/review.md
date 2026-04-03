---
description: Run specialized code reviews (architecture, readability, correctness) on a commit, PR, or set of changes
argument-hint: <commit hash, PR number/URL, or branch name>
allowed-tools: Read Glob Grep Bash(ls *) Bash(cat *) Bash(git log*) Bash(git diff*) Bash(git status*) Bash(git show*) Bash(git branch*) Bash(git rev-parse*) Bash(pwd) Bash(tree *) Bash(gh pr view*) Bash(gh pr diff*)
---

# Code Review Workflow

You are reviewing a code change using three specialized review agents. Follow
this workflow strictly.

**Input**: The user provides one of:
- A **commit hash** (e.g., `abc1234`)
- A **PR number or URL** (e.g., `#42` or `https://github.com/org/repo/pull/42`)
- A **branch name** to compare against the main branch

---

## Phase 1: Gather the Changes

Based on the input type:

- **Commit hash**: Run `git show --stat <hash>` to get files changed, then
  `git show <hash>` for the full diff.
- **PR**: Run `gh pr view <number> --json title,body,baseRefName,headRefName` for
  context, then `gh pr diff <number>` for the full diff.
- **Branch name**: Run `git diff main...<branch> --stat` for files changed,
  then `git diff main...<branch>` for the full diff.

Collect:
1. The list of all files changed
2. A summary of what the change is trying to accomplish (from commit message,
   PR description, or diff context)
3. The full diff content

---

## Phase 2: Parallel Review

**Spawn three review agents in parallel**, passing each one the list of files
changed, the change summary, and the full diff:

1. **`architect-review`** — structural problems, component boundaries,
   dependency hygiene
2. **`readability-review`** — language best practices, style guide conformance,
   naming, clarity
3. **`correctness-review`** — edge cases, unhappy paths, logic errors, missing
   unit tests

Wait for all three to complete.

---

## Phase 3: Present Results

Present the reviews to the user in a structured format:

```
## Review: <change description>

### Architecture
<architect-review verdict and issues>

### Readability
<readability-review verdict and issues>

### Correctness
<correctness-review verdict and issues>

---

### Overall
- Critical issues: <count>
- Warnings: <count>
- Nits: <count>
- Overall verdict: PASS / PASS WITH WARNINGS / FAIL
```

The overall verdict is:
- **FAIL** if any reviewer returned FAIL
- **PASS WITH WARNINGS** if any reviewer returned PASS WITH WARNINGS
- **PASS** only if all three reviewers returned PASS
