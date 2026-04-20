# Manager -- Claude Code Context

## What Is Manager

Manager is an **Anthem project agent runtime** with portfolio-level visibility. It runs with **Loop mode enabled** (tracker-backed self-maintenance on `rauriemo/manager`) and responds to portfolio-wide questions in Chat mode across all sibling projects. Plan and Execute modes are available for coordinating multi-repo changes.

Users interact with Manager through Prism's built-in Manager tab. Regular messages use Chat mode. `/fast <message>` takes the lean `claude -p` branch for instant lightweight replies (legacy `[system:fast]` tag that remaps to Chat's lean path). Manager can produce rich visual output (HTML dashboards, charts, data grids, markdown) that Prism renders in the visual pane; during multi-repo Execute runs it also drives the execution panel via `execution.*` events.

## Modes

Manager supports all four Anthem modes. The mode router picks one based on the `[system:<mode>]` tag in the incoming frame (default = Chat).

### Chat (default)
Portfolio analyst. Reads source code, git history, and GitHub data across sibling repos (`additional_dirs`) and answers conversationally. Produces A2UI display frames (HTML dashboards, data grids, charts) as part of the response.

### Plan
Used when a portfolio-wide change requires coordinated work across repos. The orchestrator synthesizes an `ExecutionPlan` with steps scoped to specific repos and, if needed, guest agents.

### Execute
`PlanRunner` dispatches the approved plan's steps. For portfolio work this usually means one repo per step with approval gates between phases. Emits `execution.*` events consumed by Prism's execution panel.

### Loop
`GitHubLoopBackend` polls `rauriemo/manager` and dispatches Claude Code workers for issue-driven self-maintenance. Uses the standard Anthem tick -> reconcile -> dispatch pipeline.

### `/fast` (legacy lean path)
`/fast` (Prism translates to `[system:fast]`) and `/status` take Chat mode's lean branch, which invokes `claude -p` directly for a single-shot response. Ideal for instant portfolio status checks and code review queries. Works from every agent tab in Prism, not just Manager.

## Plans and Architecture Docs

- `docs/plan.md` -- Full build plan with architecture decisions, Anthem upstream changes, and implementation checklist

## Ecosystem Overview

Manager operates within a system of interconnected projects, all living as sibling directories under `C:\Users\rafa\Projects\`:

| Project | Local Path | GitHub Repo | Port | Role |
|---------|-----------|-------------|------|------|
| **manager** | `C:\Users\rafa\Projects\manager` | `rauriemo/manager` | 3106 (WS) | Portfolio oversight (this project) |
| **forge** | `C:\Users\rafa\Projects\forge` | `rauriemo/forge` | 3102 (WS) | Project scaffolding agent |
| **prism** | `C:\Users\rafa\Projects\prism` | `rauriemo/prism` | 3100 (HTTP), 3101 (WS) | Visual workstation -- A2UI canvas, chat, TTS, STT |
| **anthem** | `C:\Users\rafa\Projects\anthem` | `rauriemo/anthem` | 3105 (WS) | Go orchestrator daemon (the runtime for all agents) |
| **dispatch** | `C:\Users\rafa\Projects\dispatch` | `rauriemo/dispatch` | 3104 (WS) | Voice-first command channel |

### How Projects Relate

- **anthem** is the runtime. Every agent project (including manager) IS an anthem instance.
- **prism** is the visual frontend. It connects to all agents via WebSocket and provides chat, visual display, voice I/O.
- **forge** creates new agent projects. It scaffolds directories, writes WORKFLOW.md, registers agents.
- **dispatch** is the voice channel. Ambient, always-on voice commands.
- **manager** (this project) has cross-project visibility. It reads code and metadata from all siblings.

## Context Gathering Strategy

When answering questions, Manager gathers context from two sources:

### 1. Local Directories (Primary -- Fast, Offline)

Manager's `WORKFLOW.md` includes `additional_dirs` entries for all sibling projects (forge, prism, anthem, dispatch). This means your Claude Code sandbox can read files directly from those directories -- no need for absolute paths or workarounds.

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
| `C:\Users\rafa\Projects\forge\CLAUDE.md` | forge architecture, scaffolding API, voice/port allocation |
| `C:\Users\rafa\Projects\prism\CLAUDE.md` | prism architecture, design decisions, current status |
| `C:\Users\rafa\Projects\prism\backend\agents.yaml` | All registered agents with endpoints, voices, repos |
| `C:\Users\rafa\Projects\anthem\CLAUDE.md` | anthem architecture, all phases, design decisions |
| `C:\Users\rafa\Projects\anthem\internal\config\config.go` | All WORKFLOW.md config fields |
| `C:\Users\rafa\Projects\anthem\internal\channel\prism\adapter.go` | prism WebSocket protocol implementation |
| `C:\Users\rafa\Projects\dispatch\CLAUDE.md` | dispatch architecture, voice commands, wake words |

## Coding Standards

Manager has no application code of its own -- it's purely configuration and documentation. But when analyzing other projects:

- Respect each project's coding standards (documented in their `CLAUDE.md`)
- Never suggest changes that violate locked-in design decisions
- Reference specific files and line numbers when discussing code

## Current Status

**Phase**: Project agent runtime with Chat + Plan + Execute + Loop, plus the lean `/fast` branch.

Manager boots as an Anthem project agent runtime with GitHub issue tracking (`rauriemo/manager`) enabled for Loop mode, connects to Prism on port 3106, and handles portfolio queries (Chat / `/fast`), multi-repo change synthesis (Plan), coordinated cross-repo execution (Execute with approval gates), and self-maintenance (Loop). The built-in Manager tab in Prism doubles as both the portfolio dashboard and the agent chat interface.

**Planned enhancements (future)**:
- Skills and MCP tools for expanded capabilities
- Agent lifecycle management (start/stop other agents via Prism REST API)
- Automated scheduled reports
- Knowledge promotion across projects
