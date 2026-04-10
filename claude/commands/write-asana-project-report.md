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
- **Links**: GitHub PRs, Figma files, Notion docs, Slack threads, etc.
- **Status changes**: When the task moved between statuses

Use this context to write a more informative summary for each task in the
final report. Task titles alone are often too terse — the stories reveal
what was actually shipped (e.g., "Canvas entrypoints not being created from
left panel chat" becomes "Fixed bug where prompts sent from the left panel
chat weren't generating on-canvas entrypoints, caused by missing thread
association logic").

Parallelize story fetches across tasks using subagents to avoid slow
sequential calls.

---

## Phase 4: Categorize Tasks

Using the detailed task data, categorize into:

1. **Completed within window**: Tasks completed in the time window. Note
   assignee for each.

2. **In progress / Pending**: Open tasks that have assignees and appear active
   (have due dates, are in active sections, have status fields set, etc.).

3. **At risk / Needs attention**: Tasks that are past due, unassigned but
   tagged as high priority, or have a blocked status.

If custom fields were fetched and priority data is available, tag each task
with its priority level and rank Blockers above other items.

---

## Phase 5: Draft the Status Update

Present the update in this format:

```
## <Project Name> — <Month Day>

**Status**: <On Track / At Risk / Off Track> <color emoji>

<2-3 sentence summary: wins, current focus, and risks if any>

### Completed This Week

- <task name> (<assignee>)
- ...

### Pending Next

1. <next priority>
2. ...
```

**Guidelines**:
- Keep the tone positive and concise
- Lead with wins before risks
- Group completed items by priority if priority data is available
- Use numbered list for pending items to signal priority order
- Keep the full update under 30 lines — this is a status update, not a novel

---

## Phase 6: Review and Post

1. Present the draft to the user for review.
2. Iterate on feedback until the user approves.
3. When the user says to post, check if an Asana tool for creating project
   statuses is available. If not, format the update as plain text ready
   to copy-paste into Asana.
