---
description: Fetch Asana project tasks and draft a status update in Asana project report style
argument-hint: "<Asana project URL or GID> [natural language search criteria, e.g. 'all blocker tickets completed this past week assigned to Bernie']"
---

# Write Asana Project Report

You are drafting a weekly status update for an Asana project. The goal is to
fetch tasks, categorize them, and produce a concise project status update
ready to post.

**Input**: $ARGUMENTS

The first argument is the Asana project URL or GID. Everything after it is a
natural language description of what to include in the report. Interpret this
to determine:

- **Time window**: How far back to look for completed tasks (default: 7 days)
- **Assignee filter**: Specific person(s) to focus on, or all
- **Priority filter**: Specific priority levels (e.g., "blockers only")
- **Status filter**: Specific statuses (e.g., "blocked", "in review")
- **Custom fields**: Only fetch custom fields if the query references them
  (e.g., priority, blocker status, expected release). Custom fields are
  extremely verbose — skip them unless needed.
- **Scope**: Any other constraints (e.g., "only bug fixes", "only frontend")

If no search criteria are provided, default to: all tasks completed in the
last 7 days + all open tasks, no custom fields.

---

## Phase 1: Parse Input and Fetch Project

1. **Extract project GID** from the input. If a URL like
   `https://app.asana.com/.../project/<GID>/...`, extract the GID. Otherwise
   treat the first positional argument as a GID directly.

2. **Interpret the natural language criteria** into concrete filters:
   - Time expressions ("this week", "past 2 weeks", "since Monday") → date range
   - People ("assigned to Caroline", "Bernie's tasks") → assignee name
   - Priority ("blockers", "high priority and above") → priority levels
   - Status ("in progress", "blocked") → status values
   - Whether custom fields are needed for the requested filters

3. **Fetch project info**:
   ```
   asana_get_project(project_id=GID, opt_fields="name,current_status")
   ```

4. **Fetch project sections**:
   ```
   asana_get_project_sections(project_id=GID)
   ```

---

## Phase 2: Fetch Task IDs (Lightweight Discovery)

The first pass fetches only task GIDs and minimal metadata for filtering.
This avoids exceeding context limits on large projects.

```
asana_get_tasks_for_project(
  project_id=GID,
  limit=100,
  opt_fields="name,completed,completed_at,assignee.name,due_on"
)
```

Paginate with the `offset` token until all tasks are fetched.

### Apply interpreted filters to narrow the list

From the full task list, select tasks matching the interpreted criteria:

1. **Completed within window**: Tasks where `completed_at` falls within the
   interpreted time window from today.
2. **Open tasks**: Tasks where `completed` is false (unless the user only
   asked about completed tasks).
3. **Assignee**: Match by assignee name if specified.
4. **Due date**: Apply any due date filters from the query.

Build a **short list of task GIDs** from these filtered results.

---

## Phase 3: Fetch Full Details (Targeted)

For each task GID in the short list, fetch complete details using
`asana_get_multiple_tasks_by_gid`:

**Default opt_fields**: `name,completed,completed_at,assignee.name,due_on,notes,tags.name,memberships.section.name`

**If custom fields are needed** (based on interpreted criteria): add
`custom_fields` to opt_fields.

```
asana_get_multiple_tasks_by_gid(
  task_gids=[list of GIDs],
  opt_fields=<built opt_fields>
)
```

If the batch tool is unavailable, fall back to individual
`asana_get_task(task_gid=GID)` calls. Parallelize where possible using
subagents.

### Enrich with context

For each task in the short list, fetch its stories (comments, status changes,
links, PR references) to build a richer understanding of what was actually
done:

```
asana_get_task_stories(task_gid=TASK_GID)
```

From the stories, extract:
- **Comments**: Summarize key discussion points, decisions, or progress notes
- **Status changes**: When the task moved between statuses

Use this context to write a more informative summary in the final report.
Task titles alone are often too terse — the stories reveal what was actually
shipped. Do **not** embed PR numbers or PR URLs in the final report body.

Parallelize story fetches across tasks using subagents to avoid slow
sequential calls.

### Optional: merge supplementary sources

If the user provides additional context sources (e.g., a FigJam board with a
weekly standup column, a Google Doc, a Slack digest), fetch those too and
merge them into the Completed This Week section. For FigJam boards, use
`mcp__plugin_figma-plugin_figma-mcp-prod__get_figjam` (or the staging
equivalent) with the fileKey and nodeId parsed from the URL.

---

## Phase 4: Categorize Tasks

Using the detailed task data, categorize into:

1. **Completed within window**: Tasks completed in the time window.

2. **In progress / Pending**: Open tasks that have assignees and appear active
   (have due dates, are in active sections, have status fields set, etc.).

3. **At risk / Needs attention**: Tasks that are past due, unassigned but
   tagged as high priority, or have a blocked status.

If custom fields were fetched and priority data is available, rank Blockers
above other items.

### Group by workstream, not by ticket

The final report is organized by **workstream / theme**, not by individual
ticket. Before drafting, cluster the completed tickets and any supplementary
updates into 6–10 thematic buckets (e.g., "NUX", "Mini-chat retrieval",
"Work receipts polishes", "Beta rollout infrastructure", "Stop button",
"Resume inflight chat states"). Each bucket becomes one bullet in the
Completed This Week section, rolling up multiple related tickets into a
single outcome-focused statement.

---

## Phase 5: Draft the Status Update

Use a strict three-section format. Do **not** add a Risks section, a Progress
section, or any other top-level sections beyond these three.

```
## <Project Name> — <Month Day>

**Status**: <On Track / At Risk / Off Track> <color emoji>

<2-4 sentence summary that explicitly explains the status color. If At Risk
or Off Track, name the specific blockers or slips driving that call. Reflect
the current state of the launch/milestone accurately — verify against user
context or recent conversation before asserting something has shipped or is
live.>

### Completed This Week

- **<Workstream / theme name>** — <1–2 sentence outcome summary rolling up
  the tickets and updates in this bucket. No IC names. No PR numbers or PR
  links.>
- ...

### Pending Next

1. **<Project / theme, not a single ticket>** — <brief description of what
   that entails and why it matters>
2. ...
```

**Hard rules** (learned from user feedback):

- **Three sections only**: Status / Completed This Week / Pending Next.
- **No IC names** in bullet points. The report is about workstream progress,
  not individual attribution. (Exception: names may appear in the status
  summary when referring to a specific person's circumstance, e.g.,
  someone's PTO affecting coverage, and only if the user's input explicitly
  flags it.)
- **No PR links, PR numbers, or GitHub URLs** in the report body. Other
  links (tech-spec docs, runbooks) are acceptable when meaningfully
  referenced.
- **Completed This Week = workstream rollups, not ticket lists**. Merge
  multiple related tickets into a single themed bullet.
- **Pending Next = thematic projects, not tickets**. Each item should be a
  named workstream/project with a short "what that entails" clause, not a
  one-line ticket title.
- **Status summary must justify its color**. If At Risk, state why in the
  summary itself (e.g., "Marking at risk as we continue to have open Beta
  blockers and we are still investigating solutions to Stop action.").
- **Verify launch state**. Do not assert that a product/milestone is "live",
  "launched", "shipped", or "feature complete" unless the user's input or
  recent conversation explicitly confirms it. When in doubt, frame work as
  in-progress.
- Keep the full update concise — aim for under 30 lines.

---

## Phase 6: Review and Post

1. Present the draft to the user for review.
2. Iterate on feedback until the user approves. **Do not post until the user
   explicitly says to post** — even in auto mode, posting to Asana affects
   shared state and requires an explicit go-ahead. If the user says "hold"
   or "do not post", hold the draft indefinitely.
3. When the user says to post, use
   `mcp__plugin_asana_asana__asana_create_project_status` with:
   - `project_gid`: the project GID
   - `color`: `green` | `yellow` | `red`
   - `title`: `<Project Name> - <Month Day>`
   - `text`: plain-text version of the draft (convert markdown bullets to
     dashes; drop bold markers; use `----------------------------------------`
     as section separators to match prior-status convention)
4. Confirm the posted status ID back to the user.
