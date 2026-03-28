# Cortex: Agent Team Coordination Plugin for Claude Code

## Overview

Cortex is a Claude Code plugin that lets users coordinate a team of AI agents through a shared folder of markdown files. A chief of staff agent receives instructions via Telegram, dispatches work to project-specific worker agents, and monitors progress — all through plain markdown with YAML frontmatter.

No special tooling required. The shared folder can be an Obsidian vault, a git repo, or any directory of markdown files.

## Architecture

### Communication Model

```
User
  |
  +-- Telegram --> Chief of Staff (Claude Code session)
  |                  |
  |                  +-- Reads/writes team dir (agents/, projects/)
  |                  +-- Dispatches tasks -> project work queues
  |                  +-- Monitors agent session logs
  |                  +-- Daily briefing & review via Telegram
  |                  |
  |                  v
  |               Team Dir (~/cortex-team/)
  |                  ^
  |                  |
  +-- Terminal --> Worker Agents (each in own Claude Code session)
                     +-- /join-cortex on session start
                     +-- Heartbeat polls work queue every 15 min
                     +-- Pick up ready tasks, do work, report done
                     +-- Update session log in agent note
```

No direct agent-to-agent communication. All coordination happens through the shared markdown files — the team dir is the message bus.

Session independence: each agent runs in its own Claude Code session. They don't need to be running simultaneously. A worker can pick up tasks hours after the chief of staff wrote them.

## Plugin Structure

```
cortex/
  skills/
    setup-cortex/SKILL.md
    join-cortex/SKILL.md
    leave-cortex/SKILL.md
    register-agent/SKILL.md
  README.md
  package.json
```

## Configuration

`/setup-cortex` creates `~/.cortex/config.yaml`:

```yaml
team_dir: ~/cortex-team
heartbeat_minutes: 15
daily_briefing: "09:00"
daily_review: "18:00"
telegram_bot_token: "bot123456:ABC-DEF..."
telegram_chat_id: "123456789"
```

Skills read this config at runtime. If config doesn't exist, skills tell the user to run `/setup-cortex`.

## Team Directory

Created by `/setup-cortex`:

```
~/cortex-team/
  agents/
    chief-of-staff.md
    billing-dev.md
    dashboard-dev.md
  projects/
    billing-service.md
    dashboard.md
  templates/
    agent-template.md
```

## Agent Notes

Each agent has a markdown file at `agents/<slug>.md`.

Example (`agents/billing-dev.md`):

```yaml
---
name: Billing Dev
project: ~/Projects/billing-service
status: active
joined: 2026-03-27
---
```

```markdown
## Role
Build and maintain the billing service — payment processing, invoicing, subscription management.

## Projects
- billing-service — primary project

## Capabilities
- Python/FastAPI development
- Stripe integration
- Database migrations

## Session Log
Last session: --
Status: registered
```

### Fields

- `name` — display name
- `project` — absolute path to the agent's working directory
- `status` — `active` or `inactive`
- `joined` — date registered

### Sections

- `## Role` — one-line description of what this agent does
- `## Projects` — links to project notes this agent serves
- `## Capabilities` — skills, tools, domain expertise
- `## Session Log` — last session date, current state, blockers (updated by the agent)

### Cross-References

Agent and project notes reference each other by slug (e.g., `billing-service`, `billing-dev`). These are plain text — not Obsidian wikilinks or functional hyperlinks. Agents resolve references by looking for `<slug>.md` in the appropriate directory (`agents/` or `projects/`). If users open the team dir in Obsidian, they can use `[[slug]]` syntax for clickable links, but the plugin does not depend on this.

## Project Notes

Each project has a markdown file at `projects/<slug>.md`.

Example (`projects/billing-service.md`):

```yaml
---
type: project
created: 2026-03-27
status: active
---
```

```markdown
## Next Action
- Implement webhook handler for failed payments

## Notes
- Agent: billing-dev at ~/Projects/billing-service
- Stack: Python, FastAPI, PostgreSQL

## Work Queue

### 2026-03-27T10:00
**Task:** Add Stripe webhook endpoint for payment failures
**Scope:** Create POST /webhooks/stripe, verify signature, handle payment_intent.payment_failed events, update subscription status
**Status:** ready
```

### Work Queue Format

Each task entry under `## Work Queue`:

```
### <ISO timestamp>
**Task:** Short title
**Scope:** Detailed description of what to do
**Status:** ready | in-progress | done
```

- Chief of staff writes tasks with status `ready`
- Worker picks up, marks `in-progress`, does work, marks `done`
- Worker appends a summary of what was done when marking `done`

## Skills

### /setup-cortex — First-Time Setup

Interactive skill that:

1. Asks for team directory path (default: `~/cortex-team`)
2. Creates the directory structure (`agents/`, `projects/`, `templates/`)
3. Writes `templates/agent-template.md`
4. Asks for Telegram bot token and chat ID (for chief of staff communication). Guides user through creating a bot via @BotFather if they don't have one yet.
5. Asks for chief of staff project directory path
6. Writes `~/.cortex/config.yaml` with all settings
7. Creates `agents/chief-of-staff.md` with the coordinator role

### /register-agent \<name\> — Register a New Agent

Run by the chief of staff to add a new team member:

1. Creates `agents/<slug>.md` from template
2. Asks for: project path, role, capabilities
3. Tells the user to run `/join-cortex <name>` in the agent's project

### /join-cortex \<name\> — Worker Onboarding

Run by the worker in its own project directory:

1. Reads `~/.cortex/config.yaml` to find team dir
2. Reads `agents/<slug>.md` from team dir
3. Generates `TEAM.md` in the agent's project root with the full protocol
4. Updates `CLAUDE.md` with the auto-sync line
5. Suggests setting up heartbeat: tells the agent to run `/loop` or use CronCreate for periodic polling at the configured interval
6. Idempotent — safe to re-run as a sync

### /leave-cortex \<name\> — Worker Offboarding

1. Removes `TEAM.md` from project root
2. Cleans Cortex section from `CLAUDE.md`
3. Sets agent status to `inactive` in team dir

## TEAM.md Protocol

Generated by `/join-cortex`, placed in the agent's project root. Contains:

- Agent identity and team dir path
- Session start: run `/join-cortex` to sync
- Check work queues: read linked project notes, pick up `ready` tasks
- Do work: mark `in-progress`, execute, mark `done` with summary
- Report back: update project note work queue and agent note session log
- Update `SESSION_LOG.md` in project directory for local session continuity
- Heartbeat: poll team dir every N minutes (from config, default 15) when looping

## Chief of Staff

The chief of staff is a Cortex agent with additional coordinator responsibilities.

### Core Duties

- Receive instructions from the user (primarily via Telegram)
- Register new agents via `/register-agent`
- Dispatch work by writing tasks to project work queues with status `ready`
- Monitor agent status by reading `## Session Log` in agent notes
- Flag stale or blocked agents to the user

### Heartbeat

The chief of staff's heartbeat polls for:

- Agent session logs that haven't updated (staleness detection)
- Blocked agents (reads blockers from agent notes)
- User messages via Telegram

### Daily Briefing (configurable, default 09:00)

Scan all agents and projects, summarize status, flag anything needing attention, send via Telegram.

### Daily Review (configurable, default 18:00)

Summarize what got done today, what's blocked, what's coming tomorrow, send via Telegram.

### Project Directory

The chief of staff runs in its own Claude Code session. Its `project` path in the agent note points to wherever the user wants — a dedicated project directory, or the team dir itself. `/setup-cortex` asks for this path when creating the chief of staff agent note.

### What It Does NOT Do

- Execute work on other agents' projects — it delegates
- Approve its own tasks — the user is the authority

## Key Decisions

- **Plain markdown over Obsidian-specific:** The team dir is just a folder of markdown files. Works with Obsidian, git, or any editor. No external CLI dependency.
- **Skills-only plugin:** No MCP server, no CLI wrapper. Skills use Claude Code's built-in file tools (Read, Write, Edit, Grep, Glob) to operate on the team dir.
- **Plain markdown files as coordination layer:** No dependency on Obsidian or any specific tool. If demand exists for other backends (database, API), extract a storage abstraction later.
- **Separate register and join:** Chief of staff decides who's on the team (/register-agent). Workers onboard themselves (/join-cortex). Clean separation of concerns.
- **Heartbeat default 15 min:** Configurable in `~/.cortex/config.yaml`.
- **One agent per project:** Each project has one assigned worker agent. This avoids task-claiming races — when a task is `ready`, the assigned agent picks it up without contention. If a project needs multiple agents (e.g., frontend and backend work on the same repo), split into separate project notes with scoped work queues (e.g., `billing-service-api.md`, `billing-service-frontend.md`). This keeps coordination simple for v1. Task claiming can be added later if multi-agent-per-project becomes a real need.

## Naming

Working name: "Cortex." The name has conflicts in the AI/dev tools space (Palo Alto Networks Cortex, Snowflake Cortex, janhq/cortex). Needs a more distinctive name before public release.

Candidates to consider:
- **Hivemind** — collective intelligence, agent swarm coordination
- **Dispatch** — the core action (chief of staff dispatches work)
- **Relay** — agents relay status through shared state
- **Cadence** — heartbeats, daily rhythms, coordination tempo
- **Syndicate** — a group working together toward a goal
- **Nexus** — connection point between agents

Decision deferred until closer to public release. Using "Cortex" internally for now.
