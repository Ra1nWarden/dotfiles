---
description: File a synthesized Asana ticket from a FigJam sticky/section or Slack thread. Supports per-project custom fields, dedup search, and title confirmation.
argument-hint: <source-url> [more-context-url ...] --project=<gid> [--release=<name>] [--priority=<name>] [--type=<name>] [--status=<name>] [--assignee=<email|me|none>] [--custom-field=<name>=<value> ...] [--skip-dedup] [--skip-confirm]
---

# File Asana Ticket

You are filing one Asana ticket synthesized from one or more sources. Sources
can be FigJam stickies, FigJam sections (threaded discussions), or Slack
message permalinks. Output is always a clean bug-report-style ticket: clear
description, summarized details, repro steps when applicable, and back-links
to the original source(s) for traceability.

**Input**: $ARGUMENTS

---

## Arguments

Parse `$ARGUMENTS` into:

- **Positional URLs** (one or more):
  - The first URL is the **primary source**.
  - Remaining URLs are **additional context** (linked in the Source section
    but not driving the title or repro).
- **`--project=<gid>`** — Asana project GID. Accept either a raw GID or a
  project URL like `https://app.asana.com/.../project/<GID>/...` and
  extract the GID. If omitted, see "Resolving a missing project" below.
- **`--workspace=<gid>`** — optional. Defaults to the project's workspace
  (resolved via `asana_get_project`).
- **`--release=<name>`** — sets the "Expected release" custom field if it
  exists in the project (e.g. `Beta`, `GA`, `Post-GA`). Resolved by name.
- **`--priority=<name>`** — sets the "Priority SLA" custom field if it
  exists (e.g. `High`, `Normal`, `🛑 Blocker`). Resolved by name; emoji
  prefixes in option names are matched without requiring the user to type
  them (e.g. `--priority=High` matches `🔺 High`).
- **`--type=<name>`** — sets the "Type" custom field if it exists (e.g.
  `Bug`, `Polish`).
- **`--status=<name>`** — sets the "Status" custom field if it exists.
- **`--assignee=<email|me|none>`** — assigns the task. `none` (default)
  leaves it unassigned.
- **`--custom-field=<name>=<value>`** — generic escape hatch, repeatable.
  Resolves the field by name in the project's custom_field_settings, then
  matches `value` against the enum option name (or sets directly for text /
  number / date fields).
- **`--skip-dedup`** — skip the duplicate search (Phase 6).
- **`--skip-confirm`** — skip the title + dedup confirmation gate (Phase 7).
  Headless / batch mode. Without this flag, always confirm before creating.

### Resolving a missing project

If `--project` is missing, **don't fail** — the user typically just pastes a
URL. Instead:

1. Check user memory for known Asana project entries. The user's memory
   index (`~/.claude/projects/.../memory/MEMORY.md`) often lists projects
   under "Reference" entries (e.g., `reference_asana_assistant_planning.md`
   stores GIDs for Assistant Planning, UX & Launch Readiness, etc.).
2. Read those memory files to extract project GIDs and names.
3. Present the candidates plus an "other" option:
   ```
   No --project specified. Known projects:
     1. Assistant Planning (GID: ...)
     2. UX & Launch Readiness (GID: ...)
     3. Entrypoints (GID: ...)
     4. Other — paste a project URL or GID

   Reply with a number, or paste a URL/GID.
   ```
4. Wait for user reply, then resolve to a project GID and proceed.

If memory has no project entries, just ask the user for a GID or URL.

---

## Phase 1: Route the primary source URL

Inspect the primary URL:

- `figma.com/board/<fileKey>/...?node-id=<N-N>` or
  `staging.figma.com/board/...` → **FigJam flow** (Phase 2A).
  Pick the staging vs prod Figma MCP server based on the hostname.
  - `staging.figma.com` → use `mcp__plugin_figma-plugin_figma-mcp-staging__*`
  - `figma.com` / `www.figma.com` → use
    `mcp__plugin_figma-plugin_figma-mcp-prod__*`
- `*.slack.com/archives/<CHANNEL>/p<TS>[?thread_ts=...]` → **Slack flow**
  (Phase 2B).

Convert FigJam node IDs from URL form (`20-141`) to colon form (`20:141`)
before passing to MCP tools.

If the URL doesn't match either pattern, stop and tell the user the source
isn't supported.

---

## Phase 2A: FigJam flow

### 2A.1: Read the primary node

Call `get_figjam(fileKey, nodeId)` on the primary URL. The result tells you
the node type. Branch:

- **Single sticky** (`<sticky ...>`): you have the bug report. Capture
  `author` and verbatim text. Sticky color is **ignored** (per project
  convention).
- **Section / group / frame** containing children (`<section ...>`,
  `<group ...>`, `<frame ...>`): walk the immediate children. Filter to
  `<sticky>` and any image/`media` companion nodes. Order stickies by
  their `(x, y)` position (top→bottom, left→right) — that's the thread
  order in FigJam.

### 2A.2: Read companion media

If additional URLs were passed, or if the section contains image / `media`
children, treat each as a companion to be linked from the Source section.

For each companion node, read its FigJam representation with `get_figjam`
to know its type. Label it in the ticket Source section as:

- `media` → "Video"
- image / `rounded-rectangle` / generic visual → "Screenshot"

Do **not** download companion media to disk by default — `get_screenshot`
URLs are short-lived, but the FigJam back-link is durable and is what goes
into the ticket. (Only download with `mcp__plugin_figma-plugin_*_get_screenshot`
+ `curl` if the user explicitly asks for an attachment workflow.)

### 2A.3: Surface author

If the (lead) sticky's `author` ≠ the current user (current user is the
git user from `git config user.email`, or otherwise inferred from
`mcp__plugin_asana_asana__asana_get_user(user_id="me")`), surface the
author name in the Source section as "reported by …".

For sections with multiple authors, name the lead reporter and (if
distinct) the person who confirmed repro.

---

## Phase 2B: Slack flow

### 2B.1: Fetch the thread

Use `mcp__slack__fetch` on the primary URL. If the URL has a `thread_ts`
parameter, fetch the full thread; otherwise fetch the parent message and
its reply chain.

No channel allowlist — the command runs on whichever thread the user has
access to via Slack OAuth. Inherits Slack permissions naturally.

### 2B.2: Synthesize aggressively

Slack threads are **never** pasted verbatim. The original messages are
context only — the ticket extracts:

- **The bug**: what's broken, scope, who's affected
- **Repro**: walk the thread looking for steps; reconstruct if scattered
- **Expected vs actual**: pull from the thread or infer
- **Repro confirmations**: roll up as "confirmed by N others" without
  listing names unless distinct content (e.g. one person saw it on a
  different platform — keep that)
- **Files / images**: link Slack file permalinks directly in Source
  (these don't expire)
- **Cross-references**: PR links, dashboards, runbooks, commit hashes —
  these are gold, surface them in Source

Drop:

- Reactions, "+1", "lol", "👀"
- Pure clarifying questions ("which env?") unless the answer adds
  load-bearing info — fold the answer into Description in that case
- Side-conversations that resolved to "nm, user error"

For long threads (>50 replies), sample: read the first ~15, the last
~15, and any with file attachments or links. Note in Source that the
thread was sampled.

### 2B.3: Surface author

The lead reporter is the author of the parent message. Surface as
"reported by @user in #channel".

---

## Phase 3: Synthesize the ticket body

Use a uniform output template across all source types:

```html
<body>
<h2>Description</h2>
<blockquote>One paragraph: what's broken, scope, who's affected.</blockquote>
<h2>Repro</h2>
<ol><li>step</li>...</ol>
<h2>Expected vs actual</h2>
<ul>
  <li><strong>Expected:</strong> ...</li>
  <li><strong>Actual:</strong> ...</li>
</ul>
<h2>Source</h2>
<ul>
  <li>Primary: <a href="...">label</a> (reported by @author)</li>
  <li>Additional context: <a href="...">label</a></li>
</ul>
</body>
```

Drop sections that don't apply (e.g., omit Repro and Expected vs actual
for a polish-nit ticket where the sticky is just an observation). Always
keep Description and Source.

For **single FigJam stickies**, the Description can quote the verbatim
text in `<blockquote>` since the sticky is itself short — synthesis
isn't needed. Use the bug-report shape only when there's enough material
to synthesize (a section thread or a Slack conversation).

### Title

Generate a concise, imperative or noun-phrase title. The title is the
single most-skimmed field — it should:

- Name the affected UI element by its canonical product term
- State the issue plainly
- Avoid restating the question form from the source ("Should we…" → "X
  is missing", "Y is off-center", "Z is broken")
- Be 5–12 words

Title generation is the weakest part of this flow — see the Title
confirmation phase.

---

## Phase 4: Resolve custom fields by name

Custom field GIDs are project-specific. Resolve at runtime, every time:

```
asana_get_project(
  project_id=<--project>,
  opt_fields="name,workspace.gid,custom_field_settings.custom_field.name,custom_field_settings.custom_field.gid,custom_field_settings.custom_field.resource_subtype,custom_field_settings.custom_field.enum_options.gid,custom_field_settings.custom_field.enum_options.name"
)
```

For each `--release`, `--priority`, `--type`, `--status`, and
`--custom-field=<name>=<value>` pair:

1. Find the field setting where `custom_field.name` equals the requested
   field name (case-insensitive match).
2. For enum fields, find the option whose `name` matches the requested
   value. Match flexibly:
   - Exact case-insensitive match first
   - If that fails, match ignoring leading emoji + whitespace (so
     `--priority=High` matches `🔺 High`)
3. Build the `custom_fields` object as
   `{<field_gid>: <option_gid>, ...}` for enum fields, or
   `{<field_gid>: <value>}` for text/number fields.

If a requested field doesn't exist in the project, warn the user and
skip it. If a requested option doesn't match any enum option, list the
available options and stop — don't guess.

---

## Phase 5: Dedup search (same project only)

Unless `--skip-dedup` is set:

1. Extract 2–4 distinctive terms from the proposed title. Skip stop
   words ("the", "a", "is", "on"). Examples:
   - "Show loading indicator on canvas after page refresh" →
     `loading indicator canvas refresh`
   - "AI Chat circle off-center" → `AI Chat circle off-center`
2. Run:
   ```
   asana_search_tasks(
     workspace=<workspace_gid>,
     projects_any=<--project>,
     completed=false,
     text=<distinctive terms>,
     opt_fields="name,permalink_url,completed"
   )
   ```
   **Always** scope with `projects_any` — never search the whole
   workspace. Cross-project hits are noise.
3. Take the top 3 hits by relevance. If zero hits, proceed without
   surfacing any.

---

## Phase 6: Confirm title and duplicates

Unless `--skip-confirm` is set, present a single combined gate before
creating:

```
Proposed ticket:

  Title: <generated title>
  Project: <project name>
  Custom fields: <release / priority / type / status / etc., as requested>
  Assignee: <assignee or "none">

Possible duplicates in this project (asana_search_tasks, top 3):
  1. <title> — <permalink>
  2. <title> — <permalink>
  (or "no candidates")

Reply with one of:
  - `create` to proceed
  - `dup <n>` if it's a duplicate (I'll surface that ticket's link
    instead of creating a new one)
  - a rewritten title to use instead, then `create`
```

Wait for the user's response. Title generation is unreliable (~50%
miss rate based on past usage), so this gate is the single most
valuable interaction in the flow.

If the user replies with a rewritten title, accept it as-is and
proceed.

---

## Phase 7: Create the task

Call `mcp__plugin_asana_asana__asana_create_task`:

```
asana_create_task(
  name=<final title>,
  project_id=<--project>,
  html_notes=<body>...</body>,
  custom_fields=<JSON string from Phase 4>,
  assignee=<resolved assignee or omitted>
)
```

On success, present the `permalink_url` to the user.

If the API rejects the HTML notes, see "HTML notes rules" below —
common cause is an invalid tag (`<p>` is not allowed) or mixed
inline/block content under `<body>`. Fix the structure and retry, do
not strip the entire body.

---

## HTML notes rules (Asana)

Strict rules learned the hard way — encode these in the body builder:

- Wrap in `<body>...</body>`. Must be well-formed XML.
- **Allowed tags only**: `<body>`, `<strong>`, `<em>`, `<u>`, `<s>`,
  `<code>`, `<ol>`, `<ul>`, `<li>`, `<a>`, `<blockquote>`, `<pre>`,
  `<h1>`, `<h2>`, `<hr/>`, `<img>`.
- **`<p>` is NOT allowed.** Do not use it.
- Avoid mixed inline+block siblings under `<body>` — wrap headings in
  `<h2>` and put short body text inside `<blockquote>` or list items
  to keep the XML well-formed.
- Only `<a>` may carry attributes. Other elements with attributes get
  a 400.
- HTML-escape `&`, `<`, `>` in user-supplied text before embedding.

---

## Hard rules (encoded learnings)

- **Title gate** — never skip the title-confirmation step unless
  `--skip-confirm` is set. The model's draft is a starting point, not
  ground truth. The user may rewrite, accept the draft, or revise
  again.
- **Sticky color is ignored.** Don't infer severity from color.
- **"nit" / "minor" / "polish" prefix** in source text does not
  auto-downgrade priority. Use the user's `--priority` value as-is.
- **Sticky/message author** is surfaced in Source only when ≠ the
  current user.
- **No attachments via MCP.** The Asana plugin doesn't expose an
  attachment endpoint. Always link sources via the FigJam back-link or
  Slack file permalink. Don't try to upload.
- **`get_screenshot` URLs are short-lived.** Do not embed them in
  ticket notes.
- **`media` (video) FigJam nodes** return a poster frame from
  `get_screenshot`, not the video. The FigJam back-link is the only
  way to view the video — label it "Video" in Source.
- **Synthesize aggressively for threads.** Original message text is
  context for *you*, not the ticket. The ticket is a clean bug report;
  the back-link preserves originals.
- **Drop clarifying questions** from threads. Keep their answers if
  the answers add load-bearing info.
- **Custom field GIDs are project-specific.** Resolve every time —
  don't cache across projects.
- **Dedup is project-scoped.** Always pass `projects_any` to
  `asana_search_tasks`. Never search the whole workspace.

---

## Examples

### Single FigJam sticky, default fields

```
/file-asana-ticket https://staging.figma.com/board/qO1eh2jB8bedWeNAcKP6eg/...?node-id=20-141 \
  --project=1213644398525039 \
  --release=Beta \
  --priority=High
```

→ Reads sticky `20:141`, generates title, checks dedup in project
`1213644398525039`, prompts to confirm, creates with Expected
release=Beta, Priority SLA=🔺 High.

### FigJam sticky + screenshot companion

```
/file-asana-ticket \
  https://staging.figma.com/.../?node-id=20-141 \
  https://staging.figma.com/.../?node-id=20-137 \
  --project=1213644398525039 --release=Beta --priority=High
```

→ Same flow; both URLs land in Source, screenshot labeled by node type.

### FigJam threaded section

```
/file-asana-ticket \
  https://staging.figma.com/.../?node-id=54-787 \
  --project=1213644398525039 --release=Beta --priority=High
```

→ Reads the section, walks child stickies in spatial order, drops
clarifying questions, synthesizes Description + Repro + Expected vs
actual.

### Slack thread

```
/file-asana-ticket \
  https://figma.slack.com/archives/C123/p1234567890?thread_ts=1234567890.000 \
  --project=1213644398525039 --release=Beta --priority=High
```

→ Fetches the thread via `mcp__slack__fetch`, synthesizes the ticket,
links the message permalink + any file attachments in Source.

### Slack thread + FigJam screenshot for context

```
/file-asana-ticket \
  https://figma.slack.com/.../p... \
  https://figma.com/.../?node-id=42-7 \
  --project=1213644398525039 --priority=Blocker --type=Bug
```

→ Slack thread is the primary source; FigJam screenshot is additional
context in Source.

### Headless batch mode

```
/file-asana-ticket <url> --project=<gid> --release=Beta --priority=High \
  --skip-dedup --skip-confirm
```

→ No prompts; uses generated title; creates immediately. Use only when
batching pre-vetted sources.
