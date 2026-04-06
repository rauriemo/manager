# Manager -- Claude Code Context

## What Is Manager

Manager is a **full Anthem orchestrator agent** with portfolio-level visibility. It runs the complete Anthem pipeline -- GitHub issue tracker, orchestrator, workspace management -- and can edit its own codebase. It also serves as the portfolio oversight hub via the `/fast` slash command, which uses Anthem's lean `claude -p` path for quick, read-only queries across all sibling projects.

Users interact with Manager through Prism's built-in Manager tab. Regular messages go through the full orchestrator for thoughtful, multi-turn responses. `/fast <message>` bypasses the orchestrator for instant lightweight replies. Manager can produce rich visual output (HTML dashboards, charts, data grids, markdown) that Prism renders in the visual pane.

## Dual-Mode Architecture

Manager operates in two modes depending on how the user sends a message:

### Full Orchestrator (default)
Regular chat messages go through Anthem's complete pipeline: issue tracking, orchestrator agent consultation, workspace management, code editing. This is how Manager maintains its own codebase.

### Fast/Lean Mode (`/fast` command)
Messages prefixed with `/fast` (which Prism translates to `[system:fast]`) bypass the orchestrator entirely. Anthem's `handleLeanMessage` builds a prompt and invokes `claude -p` for a single-shot response. This is ideal for portfolio status checks, code review queries, and quick questions.

The same lean path powers `/status` queries across all agents.

## Plans and Architecture Docs

- `docs/plan.md` -- Full build plan with architecture decisions, Anthem upstream changes, and implementation checklist

## Ecosystem Overview

Manager operates within a system of interconnected projects, all living as sibling directories under `C:\Users\I9 Ultra\`:

| Project | Local Path | GitHub Repo | Port | Role |
|---------|-----------|-------------|------|------|
| **Prism** | `C:\Users\I9 Ultra\prism` | `rauriemo/prism` | 3100 (HTTP), 3101 (WS) | Visual workstation -- A2UI canvas, chat, TTS, STT |
| **Manager** | `C:\Users\I9 Ultra\Manager` | `rauriemo/manager` | 3106 (WS) | Portfolio oversight (this project) |
| **Forge** | `C:\Users\I9 Ultra\Forge` | `rauriemo/forge` | 3102 (WS) | Project scaffolding agent |
| **Anthem** | `C:\Users\I9 Ultra\anthem` | `rauriemo/anthem` | 3105 (WS) | Go orchestrator daemon (the runtime for all agents) |
| **Dispatch** | `C:\Users\I9 Ultra\Dispatch` | `rauriemo/dispatch` | 3104 (WS) | Voice-first command channel |

### How Projects Relate

- **Anthem** is the runtime. Every agent project (including Manager) IS an Anthem instance.
- **Prism** is the visual frontend. It connects to all agents via WebSocket and provides chat, visual display, voice I/O.
- **Forge** creates new agent projects. It scaffolds directories, writes WORKFLOW.md, registers agents.
- **Dispatch** is the voice channel. Ambient, always-on voice commands.
- **Manager** (this project) has cross-project visibility. It reads code and metadata from all siblings.

## Context Gathering Strategy

When answering questions, Manager gathers context from two sources:

### 1. Local Directories (Primary -- Fast, Offline)

Manager's `WORKFLOW.md` includes `additional_dirs` entries for all sibling projects (`../prism`, `../anthem`, `../forge`, `../Dispatch`). This means your Claude Code sandbox can read files directly from those directories -- no need for absolute paths or workarounds.

Read source code, configs, and git history directly from sibling project directories:

- `CLAUDE.md` in each project root -- architecture context
- `WORKFLOW.md` -- Anthem configuration
- `.git/` -- commit history, branch status, diff analysis
- Source files -- for code review, coverage analysis, dependency checks

Use tools: `Read`, `Grep`, `Glob`, `Bash(git log ...)`, `Bash(git diff ...)`, `Bash(git status ...)`

### 2. GitHub API (Fallback -- Network Required)

For data not available locally (issue counts, CI status, PR data, contributor activity):

- `gh repo view <owner/repo>` -- repo metadata
- `gh issue list -R <owner/repo>` -- open issues
- `gh pr list -R <owner/repo>` -- open PRs
- `gh run list -R <owner/repo>` -- CI status
- `gh api repos/<owner/repo>/commits/<sha>/status` -- commit CI status

Use tool: `Bash(gh ...)`

### Priority

Always try local directories first. Fall back to GitHub API when:
- The repo isn't cloned locally
- You need issue/PR/CI data (not in local git)
- You need cross-repo comparison data from GitHub

## Prism Visual Output

Manager can send rich visual content to Prism via A2UI display frames. When responding to questions from Prism, prefer visual output over plain text.

### Available Display Component Types

| Kind | Use For | Key Fields |
|------|---------|------------|
| `html` | Dashboards, styled reports, interactive content | `content` (HTML string) |
| `markdown` | Formatted text, tables, documentation | `content` (markdown string) |
| `code` | Source code snippets with syntax highlighting | `content`, `language`, `title` |
| `data` | Spreadsheet-style data grids (sortable, filterable) | `columns`, `rows`, `title` |
| `chart` | Bar, line, area, pie, scatter, radar, treemap | `chartType`, `data`, `title` |
| `image` | Images | `src`, `alt`, `title` |
| `kanban` | Task boards | `columns` with cards |

### HTML Display Guidelines

HTML displays are rendered in a sandboxed iframe. They must be self-contained:
- Inline all CSS (no external stylesheets)
- Inline all JS (no external scripts)
- No external resources (images, fonts, etc.)
- Use dark theme to match Prism: `#1e1e1e` backgrounds, `#252526` panels, `#cccccc` text
- Links to GitHub (or other external sites) open in new tabs

### Example: Status Dashboard HTML

When asked for a portfolio status, produce an HTML dashboard with:
- Project cards showing commit activity, branch status, CI status
- Color-coded health indicators (green/yellow/red)
- Recent commit summaries per project
- Open issue/PR counts

### Example: Data Grid

When asked about test coverage or metrics across projects, produce a data grid:
```json
{
  "kind": "data",
  "title": "Test Coverage by Project",
  "columns": [
    {"key": "project", "name": "Project"},
    {"key": "coverage", "name": "Coverage %"},
    {"key": "tests", "name": "Test Count"}
  ],
  "rows": [
    {"project": "anthem", "coverage": "85%", "tests": 142},
    {"project": "prism", "coverage": "82%", "tests": 98}
  ]
}
```

## Anthem WebSocket Protocol (Reference)

Manager communicates with Prism via the standard Anthem WebSocket protocol:

### Inbound (Prism -> Manager)
```json
{"type": "auth", "token": "<PRISM_ANTHEM_TOKEN>", "client": "prism"}
{"type": "req", "id": "<uuid>", "text": "Give me a status report across all projects"}
```

### Outbound (Manager -> Prism)
```json
{"type": "res", "id": "<uuid>", "text": "Here's the portfolio summary..."}
{"type": "stream", "text": "<delta>", "thread": "<uuid>", "done": false}
{"type": "stream", "text": "", "thread": "<uuid>", "done": true}
{"type": "display", "component": {"kind": "html", "content": "<dashboard>"}, "id": "<display-id>"}
{"type": "event", "event": "task.completed", "text": "..."}
```

## Port and Voice Assignments (Locked In)

- **Prism channel port**: `3106` (WebSocket server that Prism connects to)
- **Voice**: `google/en-US-Chirp3-HD-Fenrir` (primary), `en-US-ChristopherNeural` (fallback)
- **Token**: Uses shared `PRISM_ANTHEM_TOKEN` (same as all agents)

## Key Files in Sibling Projects

Read these for full context when answering cross-project questions:

| File | What It Tells You |
|------|-------------------|
| `C:\Users\I9 Ultra\prism\CLAUDE.md` | Prism architecture, design decisions, current status |
| `C:\Users\I9 Ultra\anthem\CLAUDE.md` | Anthem architecture, all phases, design decisions |
| `C:\Users\I9 Ultra\Forge\CLAUDE.md` | Forge architecture, scaffolding API, voice/port allocation |
| `C:\Users\I9 Ultra\Dispatch\CLAUDE.md` | Dispatch architecture, voice commands, wake words |
| `C:\Users\I9 Ultra\prism\backend\agents.yaml` | All registered agents with endpoints, voices, repos |
| `C:\Users\I9 Ultra\anthem\internal\config\config.go` | All WORKFLOW.md config fields |
| `C:\Users\I9 Ultra\anthem\internal\channel\prism\adapter.go` | Prism WebSocket protocol implementation |

## Coding Standards

Manager has no application code of its own -- it's purely configuration and documentation. But when analyzing other projects:

- Respect each project's coding standards (documented in their `CLAUDE.md`)
- Never suggest changes that violate locked-in design decisions
- Reference specific files and line numbers when discussing code

## Current Status

**Phase**: Full orchestrator agent with lean `/fast` path.

Manager boots as a full Anthem instance with GitHub issue tracking (`rauriemo/manager`), connects to Prism on port 3106, and handles both orchestrator-driven tasks (via regular chat) and lightweight queries (via `/fast`). The built-in Manager tab in Prism doubles as both the portfolio dashboard and the agent chat interface.

**Planned enhancements (future)**:
- Skills and MCP tools for expanded capabilities
- Agent lifecycle management (start/stop other agents via Prism REST API)
- Automated scheduled reports
- Knowledge promotion across projects
