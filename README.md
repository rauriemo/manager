# Manager

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Manager** is the portfolio oversight agent for the [Anthem](https://github.com/rauriemo/anthem) ecosystem. It runs as a full Anthem orchestrator agent that can maintain its own codebase via GitHub issues, while also serving as a fast portfolio analyst via the `/fast` slash command.

Regular messages go through the full orchestrator for thoughtful, multi-turn responses. `/fast <message>` bypasses the orchestrator for instant `claude -p` replies across all sibling projects. Think of it as a dual-mode agent: project maintainer + portfolio analyst in one.

[Build plan](docs/plan.md) | [Anthem](https://github.com/rauriemo/anthem) (orchestrator) | [Prism](https://github.com/rauriemo/prism) (workstation) | [Forge](https://github.com/rauriemo/forge) (scaffolder) | [Dispatch](https://github.com/rauriemo/dispatch) (voice channel)

## Quick Start

```bash
# Manager is auto-cloned by Prism on first boot.
# To set up manually:

# 1. Clone
git clone https://github.com/rauriemo/Manager
cd Manager

# 2. Ensure Anthem is installed
go install github.com/rauriemo/anthem/cmd/anthem@latest

# 3. Ensure auth token exists
# ~/.anthem/channels.yaml should have a prism.token entry
# (Prism's setup CLI creates this automatically)

# 4. Run
anthem run
```

**You're done when:**
1. You see `prism adapter listening addr=:3106` in the log output
2. Prism connects and the Manager tab shows "online"
3. You can type a question in the Manager chat and get a response

## How It Works

```mermaid
flowchart LR
  User["User\n(Prism Manager tab)"] -->|"chat or /fast"| Prism["Prism\n(React + FastAPI)"]
  Prism -->|"WebSocket req frame"| Adapter["Prism Adapter\n(port 3106)"]
  Adapter -->|"IncomingMessage"| Router["HandleUserMessage"]
  Router -->|"regular message"| Orch["Full Orchestrator\n(issues, workspace, multi-turn)"]
  Router -->|"/fast or /status"| Lean["Lean path\n(claude -p, single-shot)"]
  Orch -->|"streamed response"| Adapter
  Lean -->|"streamed response"| Adapter
  Adapter -->|"WebSocket"| Prism
  Prism -->|"render"| Display["Visual Canvas\n(dashboards, charts, data grids)"]
```

**Regular messages** go through the full Anthem orchestrator -- issue tracking, multi-turn Claude sessions, workspace management. This is how Manager maintains its own codebase.

**`/fast` messages** bypass the orchestrator entirely. Anthem detects the `[system:fast]` prefix and routes to `handleLeanMessage`, which invokes `claude -p` for a single-shot response. Ideal for portfolio queries, status checks, and quick questions.

## Features

- **Dual-mode operation**: Full orchestrator for project maintenance, `/fast` for instant portfolio queries
- **Self-maintaining**: Tracks its own GitHub issues and can edit its own codebase via the orchestrator
- **Cross-project analysis**: `/fast` queries read source code, git history, and GitHub data across all sibling repos
- **Rich visual output**: HTML dashboards, data grids, charts, and markdown rendered in Prism's A2UI visual pane
- **`/fast` for any agent**: The `/fast` slash command works on every agent tab in Prism, not just Manager
- **Context-aware**: Reads `CLAUDE.md` from each project for architecture context before answering

## Configuration

### WORKFLOW.md

Manager's `WORKFLOW.md` follows Anthem's format (YAML front matter + Go template body) with full orchestrator config:

```yaml
# Key settings (see WORKFLOW.md for full config)
tracker:
  kind: github
  repo: "rauriemo/manager"

orchestrator:
  enabled: true

agent:
  command: "claude"
  max_turns: 10
  permission_mode: "dontAsk"
  allowed_tools:
    - "Read"
    - "Edit"
    - "Write"
    - "Grep"
    - "Glob"
    - "Bash(git *)"
    - "Bash(gh *)"
  additional_dirs:
    - "../prism"
    - "../anthem"
    - "../forge"
    - "../Dispatch"

channels:
  - kind: prism
    target: "localhost:3106"
```

### Port and Voice Assignments

| Setting | Value |
|---------|-------|
| Prism channel port | `3106` (WebSocket) |
| Voice (primary) | `google/en-US-Chirp3-HD-Fenrir` |
| Voice (fallback) | `en-US-ChristopherNeural` |
| Auth token | Shared `PRISM_ANTHEM_TOKEN` |

## Context Gathering

Manager gathers information from two sources:

| Source | Method | Use For |
|--------|--------|---------|
| **Local directories** (primary) | `Read`, `Grep`, `Glob`, `git log/diff/status` | Source code, configs, commit history, branch status |
| **GitHub API** (fallback) | `gh issue list`, `gh pr list`, `gh run list` | Issue counts, PR data, CI status, contributor activity |

Local reads are preferred -- they're instant and work offline. GitHub API fills in data not available in the local git tree.

## Ecosystem

Manager is part of the Anthem ecosystem:

- **[Anthem](https://github.com/rauriemo/anthem)** -- Go orchestrator daemon. Polls GitHub issues, dispatches Claude Code workers, manages workspaces. Manager runs as a lean Anthem instance.
- **[Prism](https://github.com/rauriemo/prism)** -- Interactive visual workstation. Chat, A2UI canvas, TTS/STT. Manager's primary interface.
- **[Forge](https://github.com/rauriemo/forge)** -- Project scaffolding agent. Creates new Anthem projects with workflow configs.
- **[Dispatch](https://github.com/rauriemo/dispatch)** -- Voice-first command channel. Wake-word activation, ambient voice.

### Portfolio

| Project | Repo | Port | Role |
|---------|------|------|------|
| Prism | [rauriemo/prism](https://github.com/rauriemo/prism) | 3100/3101 | Visual workstation |
| Manager | [rauriemo/manager](https://github.com/rauriemo/Manager) | 3106 | Portfolio oversight (this project) |
| Forge | [rauriemo/forge](https://github.com/rauriemo/forge) | 3102 | Project scaffolding |
| Anthem | [rauriemo/anthem](https://github.com/rauriemo/anthem) | 3105 | Go orchestrator daemon |
| Dispatch | [rauriemo/dispatch](https://github.com/rauriemo/dispatch) | 3104 | Voice command channel |

## Project Structure

```
Manager/
  CLAUDE.md         # Portfolio context for Claude Code (architecture, ecosystem, display guidelines)
  WORKFLOW.md       # Anthem config (channel-only, read-only tools, Prism on port 3106)
  README.md         # This file
  .gitignore        # Excludes workspaces/, .env, *.db
  docs/
    plan.md         # Build plan and architecture decisions
```

Manager is intentionally minimal -- it's an Anthem instance whose power comes from its prompt, tool access, and the Claude CLI. There is no custom application code. `WORKFLOW.md` and `CLAUDE.md` are where the intelligence lives.

## Anthem Integration

Manager runs as a full Anthem orchestrator agent with an additional lean fast path:

- **Full orchestrator**: GitHub issue tracking, multi-turn Claude sessions, workspace management for self-maintenance
- **Lean `/fast` path**: `handleLeanMessage` in Anthem's orchestrator invokes `claude -p` directly, bypassing the orchestrator pipeline for instant responses
- **`/status` fast path**: The lean handler also powers the `/status` slash command across all agents for fast, lightweight status checks
- **Unified Prism tab**: Manager's built-in tab in Prism serves as both the portfolio dashboard and the agent chat interface -- no duplicate tabs
- **Streaming**: Real-time token streaming via Anthem's `stream` frame type
- **Display frames**: Rich visual content via Anthem's `display` frame type with A2UI components

## License

[MIT](LICENSE)
