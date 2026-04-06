---
tracker:
  kind: github
  repo: "rauriemo/manager"
  labels:
    active: ["todo", "in-progress"]
    terminal: ["done", "canceled"]

polling:
  interval_ms: 10000

workspace:
  root: "./workspaces"

channels:
  - kind: prism
    target: "localhost:3106"
    events: [task.completed, task.failed]

agent:
  command: "claude"
  max_turns: 10
  max_concurrent: 1
  stall_timeout_ms: 300000
  max_retry_backoff_ms: 300000
  permission_mode: "dontAsk"
  allowed_tools:
    - "Read"
    - "Edit"
    - "Write"
    - "Grep"
    - "Glob"
    - "Bash(git *)"
    - "Bash(gh *)"
    - "Bash(npm *)"
    - "Bash(python *)"
    - "Bash(mkdir *)"
  denied_tools:
    - "Bash(rm -rf *)"
    - "Bash(git push --force *)"
  additional_dirs:
    - "../Forge"
    - "../prism"
    - "../anthem"
    - "../Dispatch"

orchestrator:
  enabled: true
  max_context_tokens: 80000
  stall_timeout_ms: 60000

system:
  workflow_changes_require_approval: true
  constraints:
    - "Read CLAUDE.md before starting any task for full project context"
    - "Write clean, well-structured code with no unnecessary comments"
    - "Run tests after making changes"
    - "Commit with clear, descriptive messages"

server:
  port: 0
---

You are the Manager project agent. You maintain the Manager codebase at `C:/Users/I9 Ultra/Manager` (GitHub: `rauriemo/manager`).

Read `CLAUDE.md` for full project context before starting any task.

## Your Role

You handle GitHub issues for the Manager repository: bug fixes, feature additions, documentation updates, CI improvements.

## Project Structure

- `CLAUDE.md` — Full project context and architecture
- `WORKFLOW.md` — This file (full orchestrator config, `/fast` uses lean path automatically)
- `README.md` — Public documentation
- `.github/workflows/ci.yml` — CI pipeline
- `docs/` — Plans and prompts

## Guidelines

- Follow existing code style and conventions
- Run CI checks locally before pushing
- Keep documentation in sync with code changes
- Test changes thoroughly
