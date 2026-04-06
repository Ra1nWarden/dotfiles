---
description: Fetch open PRs for a group of engineers and evenly distribute review assignments based on PR size, relevance, and existing reviews
argument-hint: <github-id-1> <github-id-2> ... [--no-review <github-id>] [--project <keyword>]
---

# PR Review Assignment

You are assigning PR reviews across a team of engineers. Follow this workflow strictly.

## Arguments

Parse `$ARGUMENTS` to extract:

- **GitHub IDs**: All positional args (space-separated GitHub usernames). These people are both PR authors AND potential reviewers.
- **`--no-review <id>`**: Can appear multiple times. These people's PRs are included in the pool, but they will NOT be assigned any reviews (e.g., non-engineers, people OOO). Remove them from the reviewer list.
- **`--project <keyword>`**: Optional. If provided, filter PRs to only those matching this keyword in title, labels, or file paths. If omitted, include all PRs.

If no arguments are provided, ask the user for the list of GitHub usernames.

## Process

### Step 1: Fetch open PRs

For each GitHub ID, run in parallel:

```bash
gh pr list --author <id> --state open --limit 20 \
  --json number,title,additions,deletions,files,createdAt,reviews,labels,body,reviewDecision \
  --jq '[.[] | {number, title, additions, deletions, createdAt, reviewDecision, reviews: [.reviews[] | select(.author.login != "cursor" and .author.login != "figma-opengrep") | {author: .author.login, state: .state}], labels: [.labels[].name], files: [.files[].path]}]'
```

### Step 2: Filter PRs

Remove PRs that match ANY of these criteria:
- **Older than 2 weeks** from today's date
- **Already approved**: `reviewDecision == "APPROVED"`
- **WIP**: title contains "WIP" (case-insensitive) OR labels contain "wip"
- **Project filter**: if `--project` was given, exclude PRs where the keyword does NOT appear in the title, any label, or any file path

For each remaining PR, compute size as `additions + deletions` and classify:
- XS: < 50 lines
- S: 50–200 lines
- M: 200–500 lines
- L: 500–1000 lines
- XL: > 1000 lines

### Step 3: Identify existing reviewers

For each filtered PR, check if any team member (from the GitHub IDs list) has already left a review (COMMENTED, APPROVED, or CHANGES_REQUESTED). If so, mark that PR as "carried over" with the existing reviewer. Do NOT reassign it.

### Step 4: Assign remaining PRs

For PRs with no existing team reviewer, distribute evenly among eligible reviewers:

**Constraints:**
- A person cannot review their own PR
- People listed in `--no-review` cannot be assigned reviews

**Balancing criteria (in priority order):**
1. **Even PR count** per reviewer — no reviewer should have more than 1 PR above the minimum
2. **Review chain continuity** — prefer assigning a reviewer who is already reviewing other PRs from the same author
3. **Related code** — prefer assigning a reviewer whose own PRs touch similar file paths or labels
4. **Line count balance** — among otherwise equal options, prefer the reviewer with fewer total lines to review

### Step 5: Present results

Show the following sections:

**1. Already approved (no action needed)**
Table with: PR number (as link), author, reviewer, status

**2. Carried over (existing reviewer kept)**
Table with: PR number (as link), author, reviewer, status

**3. New assignments**
One section per reviewer with a table containing: PR number (as link), author, size, rationale for assignment

**4. Active load summary**
Table with: reviewer, number of active PRs, total lines

**5. Slack message**
A copy-pasteable Slack message block formatted as:

```
**@reviewer-name**
• <pr-url> (Author, Size) — short description
• <pr-url> (Author, Size) — short description
```

Use the reviewer's GitHub username for the @-mention (the user can adjust to Slack handles). Include full GitHub PR URLs (not shorthand). Group by reviewer.

If there are PRs with zero reviews that were assigned in a previous round, add a reminder line with the full PR link.

## Important

- Run all `gh` commands in parallel where possible to minimize latency
- Pipe verbose output through `| head` or `| tail` to keep context clean
- If a PR has no description (default template only), flag it in the output
- Do NOT create memory entries or modify any files — this command only produces output
