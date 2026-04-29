# Global Instructions

## Planning & Communication

- For non-trivial tasks, write and push the plan to the blueprints repo as soon as it's generated, BEFORE presenting it for approval. Do NOT make any code changes during planning.
  1. Research the codebase using read-only tools (Read, Grep, Glob, Explore subagents). Do NOT use the built-in `EnterPlanMode` tool for the design phase — it blocks the Write tool, which would force the plan commit to happen only after you approve via `ExitPlanMode`.
  2. Once the plan is fully drafted, write it to `$BLUEPRINTS_DIR/<project>/plan/` following the blueprints convention (see `~/.dotfiles/claude/rules/blueprints.md`) and run commit-on-write immediately.
  3. Present the plan summary and the remote URL to me for review and approval (in chat, or via `ExitPlanMode` for a formal approval gate — the file is already committed at this point).
  4. Only begin implementation after I approve the plan.
- For trivial tasks (typos, single-line fixes, obvious changes), skip plan mode and proceed directly.
- Use task/todo lists whenever possible to track progress and show what's been done vs. what remains.
- For each change or step, provide a high-level summary that includes relevant technical details (e.g., which files/modules are affected, what patterns are used, key implementation decisions).
- Proactively present alternative approaches with trade-offs (performance, complexity, maintainability, etc.) and explain why you recommend a particular approach. Don't wait for me to ask.

## Verification

- After completing code changes, run a build/compile check (e.g., `npm run build`, `vite build`) to catch basic syntax and compilation errors before considering the task done.
- After UI changes, use the Playwright MCP tools to verify the changes visually if feasible (e.g., navigate to the running dev server, click buttons, fill forms, take snapshots).

## Sub-Agent Usage

- Use sub-agents to keep the main context clean and to parallelize independent work.
- Spawn `Explore` agents for codebase search, understanding architecture, and finding patterns across files.
- Spawn `Plan` agents for designing implementation approaches before coding.
- Spawn `Bash` agents for running builds, tests, and git operations.
- Spawn `general-purpose` agents for multi-step tasks that combine search and execution.
- Launch multiple agents in parallel when tasks are independent (e.g., searching multiple codebases, running tests while linting).
- Do NOT spawn sub-agents for simple single-file edits, short searches, or tasks that depend on each other sequentially.
- Always summarize sub-agent results back to me since I cannot see their output directly.
