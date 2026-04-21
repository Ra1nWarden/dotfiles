---
description: Fetch open PRs for a group of engineers and evenly distribute review assignments based on PR size, relevance, and existing reviews
argument-hint: <github-id-1> <github-id-2> ... [--no-review <github-id>] [--project <keyword>] [--ignore-reviewed]
---

# PR Review Assignment

You are assigning PR reviews across a team of engineers. Follow this workflow strictly.

## Arguments

Parse `$ARGUMENTS` to extract:

- **GitHub IDs**: All positional args (space-separated GitHub usernames). These people are both PR authors AND potential reviewers.
- **`--no-review <id>`**: Can appear multiple times. These people's PRs are included in the pool, but they will NOT be assigned any reviews (e.g., non-engineers, people OOO). Remove them from the reviewer list.
- **`--project <keyword>`**: Optional. If provided, filter PRs to only those matching this keyword in title, labels, or file paths. If omitted, include all PRs.
- **`--ignore-reviewed`**: Optional flag. When set, exclude carried-over PRs from the Slack message if the reviewer has already acted (commented or requested changes) and the author has NOT responded with new commits or replies since. These PRs are in the **author's court**, not the reviewer's.

If no arguments are provided, ask the user for the list of GitHub usernames.

## State Persistence

Assignment state is persisted across sessions in a JSON file:

```
~/.claude/projects/-Users-zwang-figma-figma/pr-assignments-state.json
```

### State file schema

```json
{
  "round": 9,
  "date": "2026-04-15",
  "team": {
    "<github-id>": {
      "name": "Human-readable name",
      "slackHandle": "@slack-handle",
      "role": "reviewer | no-review"
    }
  },
  "assignments": {
    "<pr-number>": {
      "author": "<github-id>",
      "reviewer": "<github-id>",
      "title": "PR title",
      "size": "M",
      "lines": 441,
      "assignedRound": 8,
      "status": "new | carried | approved",
      "reviewCount": 0
    }
  }
}
```

### Reading state (Step 0)

At the start of each run, read the state file if it exists. Use it to:
- **Pre-populate team roster**: reuse known `name` and `slackHandle` mappings so you don't need to ask or look them up again
- **Detect stale assignments**: if a PR was assigned in a previous round but still has `reviewCount: 0`, flag it with a warning and how many rounds it has been unreviewed
- **Increment round number**: set the new round to `previous round + 1`

If the state file does not exist, start fresh at round 1.

### Writing state (Step 6)

After presenting results (Step 5), write the updated state file with:
- Incremented round number and today's date
- Full team roster with any newly learned name/slackHandle mappings
- All active assignments (carried over + new). Remove PRs that were approved or excluded this round.

## Process

### Step 1: Fetch open PRs

For each GitHub ID, run in parallel:

```bash
gh pr list --author <id> --state open --limit 20 \
  --json number,title,additions,deletions,files,createdAt,reviews,labels,body,reviewDecision,isDraft \
  --jq '[.[] | {number, title, additions, deletions, createdAt, reviewDecision, isDraft, reviews: [.reviews[] | select(.author.login != "cursor" and .author.login != "figma-opengrep") | {author: .author.login, state: .state}], labels: [.labels[].name], files: [.files[].path]}]'
```

### Step 2: Filter PRs

Remove PRs that match ANY of these criteria:
- **Older than 2 weeks** from today's date
- **Draft**: `isDraft == true`
- **Already approved**: `reviewDecision == "APPROVED"`
- **WIP**: title contains "WIP" (case-insensitive) OR labels contain "wip"
- **Project filter**: if `--project` was given, exclude PRs where the keyword does NOT appear in the title, any label, or any file path

For each remaining PR, compute size as `additions + deletions` and classify:
- XS: < 50 lines
- S: 50–200 lines
- M: 200–500 lines
- L: 500–1000 lines
- XL: > 1000 lines

### Step 2.5: Detect PR stacks

Group filtered PRs into **stacks** — sets of PRs from the same author that form a logical series. Two PRs belong in the same stack if ANY of these are true:

- **Sequential naming**: titles contain part indicators like `[1/3]`, `[2/2]`, `(part 1)`, `step 1`, or similar sequential markers
- **Shared topic prefix**: titles share the same bracketed prefix AND a common topic keyword (e.g., `[web][assistant] add foo` and `[web][assistant] read from foo`)
- **High file overlap**: more than 50% of one PR's files also appear in the other PR (common in incremental refactors where each PR builds on the prior)
- **Dependency chain**: one PR's branch is based on another PR's branch (visible when file sets are strict supersets with shared diffs)

When a stack is detected:
- Treat the stack as a **single assignment unit** — all PRs in the stack go to the same reviewer
- If any PR in the stack already has a team reviewer (carried over), assign the entire stack to that reviewer
- In the output, visually group stacked PRs together and label the stack (e.g., "Stack: activeViewState atom refactor (3 PRs)")
- For load balancing, count a stack as 1 assignment unit but sum the total lines across all PRs in the stack

### Step 3: Identify existing reviewers

For each filtered PR, check if any team member (from the GitHub IDs list) has already left a review (COMMENTED, APPROVED, or CHANGES_REQUESTED). If so, mark that PR as "carried over" with the existing reviewer. Do NOT reassign it.

### Step 3.5: Determine court (only when `--ignore-reviewed` is set)

For each carried-over PR, determine whether the ball is in the **reviewer's court** or the **author's court** by fetching the timeline:

```bash
gh pr view <number> --json commits,reviews,comments,author \
  --jq '.author.login as $author | {
    latest_commit: ([.commits[].committedDate] | sort | last),
    latest_non_author_action: ([
      (.reviews[]  | select(.author.login != $author and .author.login != "cursor" and .author.login != "figma-opengrep" and .author.login != "aviator-app") | .submittedAt),
      (.comments[] | select(.author.login != $author and .author.login != "cursor" and .author.login != "figma-opengrep" and .author.login != "aviator-app" and .author.login != "figma-ci-4-production") | .createdAt)
    ] | sort | last)
  }'
```

Run these in parallel for all carried-over PRs.

**Court rules:**
- Compare the **latest non-author, non-bot action** (any review or comment from anyone except the PR author and known bots) against the **latest commit** pushed by the author
- If a non-author has acted more recently than the latest commit → **author's court** (someone responded, author hasn't pushed since)
- If the latest commit is more recent than any non-author action, OR if there are no non-author actions at all → **reviewer's court** (needs review or re-review)

Add a "Court" column to the carried-over table showing `Author` or `Reviewer`.

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

Use the reviewer's **Slack handle** from the state file if known. If not known, look up via `mcp__slack__users` and save to state. Fall back to GitHub username if Slack lookup fails. Include full GitHub PR URLs (https://github.com/figma/figma/pull/<number>). Group by reviewer.

If there are PRs with zero reviews that were assigned in a previous round, add a reminder line with the full PR link.

When `--ignore-reviewed` is set, **exclude** carried-over PRs that are in the **author's court** from the Slack message. Only include PRs where the reviewer needs to take action (reviewer's court + new assignments).

### Step 6: Save state

After presenting results, write the updated state file to `~/.claude/projects/-Users-zwang-figma-figma/pr-assignments-state.json` with:
- Incremented round number and today's date
- Full team roster (including any newly learned name/slackHandle mappings — preserve existing mappings for team members not in this round)
- All active assignments (carried over + new). Remove PRs that were approved or excluded this round.
- For each assignment, record `assignedRound` (the round it was first assigned) and `reviewCount` (number of team reviews on the PR)

## Important

- Run all `gh` commands in parallel where possible to minimize latency
- Pipe verbose output through `| head` or `| tail` to keep context clean
- If a PR has no description (default template only), flag it in the output
- Do NOT create memory entries — but DO write the state file after each round
