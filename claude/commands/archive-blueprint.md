---
description: Archive a blueprint by moving it to the archive folder so it is ignored in future sessions
argument-hint: <filename or slug to archive, e.g. "auth-redesign" or "202604031530-auth-redesign.md">
allowed-tools: Read Glob Grep Bash(ls *) Bash(mkdir *) Bash(mv *) Bash(find *) Bash(git add*) Bash(git commit*) Bash(git push*) Bash(git pull*) Bash(git remote*) Bash(echo *) Bash(basename *) Bash(pwd) Bash(date *) Bash(cd *)
---

# Archive Blueprint

Move a blueprint from its active folder (`research/`, `spec/`, `plan/`, or
`review/`) into `archive/` so it is no longer surfaced in future sessions.

---

## Phase 0: Environment Setup

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

## Phase 1: Locate the Blueprint

Target: $ARGUMENTS

**If no argument was given**, list all non-archived blueprints for the project
and ask the user which one to archive:

```sh
find "$BLUEPRINTS_DIR/$PROJECT" \
  -not -path "*/archive/*" \
  -name "*.md" \
  | sort
```

Present the list grouped by type (research / spec / plan / review). Ask the
user to specify which file to archive, then stop and wait.

**If an argument was given**, search for a matching file across all active
subfolders:

```sh
find "$BLUEPRINTS_DIR/$PROJECT" \
  -not -path "*/archive/*" \
  -name "*<slug>*" \
  | sort
```

- If **exactly one match** is found, proceed with that file.
- If **multiple matches** are found, list them and ask the user to confirm
  which one to archive.
- If **no match** is found, tell the user and list available blueprints (same
  as the no-argument case above), then stop.

---

## Phase 2: Archive the Blueprint

Once the target file is confirmed:

1. **Create the archive directory** if it does not exist:
   ```sh
   mkdir -p "$BLUEPRINTS_DIR/$PROJECT/archive/"
   ```

2. **Move the file**:
   ```sh
   mv "<source-path>" "$BLUEPRINTS_DIR/$PROJECT/archive/"
   ```

3. **Commit-on-write** — run the blueprints commit protocol:
   ```sh
   cd "$BLUEPRINTS_DIR" && \
     git add -A "$PROJECT/" && \
     git commit -m "archive($PROJECT): <slug>" && \
     git push || (git pull --rebase && git push)
   ```
   If push fails because no remote is configured, warn the user but do not
   treat it as a blocking error. If rebase fails, STOP and alert the user.

4. **Build the archive link**: derive the remote URL and construct the browser
   link to the archived file:
   ```sh
   REMOTE_URL=$(cd "$BLUEPRINTS_DIR" && git remote get-url origin 2>/dev/null)
   ```
   Convert SSH → HTTPS as needed and append
   `$PROJECT/archive/<filename>` on `main`.

---

## Phase 3: Confirm

Tell the user:
- Which file was archived (original path → archive path)
- The clickable link to the archived file on the remote (if available)
- A one-line reminder: *"This blueprint will no longer appear in active
  blueprint listings."*
