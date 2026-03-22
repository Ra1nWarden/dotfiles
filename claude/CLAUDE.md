# Global Instructions

## Planning & Communication

- For non-trivial tasks, always use the built-in EnterPlanMode tool first. Do NOT make any code changes during planning.
  1. Enter plan mode to explore the codebase (read, search, research only — no edits).
  2. Design the plan and present it via ExitPlanMode for review and approval.
  3. Only begin implementation after I approve the plan.
- For trivial tasks (typos, single-line fixes, obvious changes), skip plan mode and proceed directly.
- Use task/todo lists whenever possible to track progress and show what's been done vs. what remains.
- For each change or step, provide a high-level summary that includes relevant technical details (e.g., which files/modules are affected, what patterns are used, key implementation decisions).
- Proactively present alternative approaches with trade-offs (performance, complexity, maintainability, etc.) and explain why you recommend a particular approach. Don't wait for me to ask.

## Sub-Agent Usage

- Use sub-agents to keep the main context clean and to parallelize independent work.
- Spawn `Explore` agents for codebase search, understanding architecture, and finding patterns across files.
- Spawn `Plan` agents for designing implementation approaches before coding.
- Spawn `Bash` agents for running builds, tests, and git operations.
- Spawn `general-purpose` agents for multi-step tasks that combine search and execution.
- Launch multiple agents in parallel when tasks are independent (e.g., searching multiple codebases, running tests while linting).
- Do NOT spawn sub-agents for simple single-file edits, short searches, or tasks that depend on each other sequentially.
- Always summarize sub-agent results back to me since I cannot see their output directly.
