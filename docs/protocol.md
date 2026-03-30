# Cortex Protocol Specification

Cortex coordinates a team of AI agents through a shared folder of markdown files. This document defines the protocol — the folder structure, file formats, and coordination rules that any agent runtime can implement.

The Claude Code plugin is one implementation. Any agent that can read and write files can participate.

## Team Directory

A shared folder (the "team directory") contains all coordination state:

```
<team_dir>/
  agents/
    chief-of-staff.md
    billing-dev.md
    dashboard-dev.md
  projects/
    billing-service.md
    dashboard.md
  templates/
    agent-template.md
    project-template.md
```

The team directory can be any folder — a git repo, an Obsidian vault, a Dropbox folder, or a plain directory on a shared filesystem.

## Agent Notes

Each agent has a markdown file at `agents/<slug>.md`.

### Frontmatter (YAML)

```yaml
---
name: Billing Dev
project: ~/Projects/billing-service
status: active
joined: 2026-03-28
last-heartbeat: "2026-03-28T14:30"
---
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name |
| `project` | string | Absolute path to the agent's working directory |
| `status` | `active` \| `inactive` | Whether the agent is on the team |
| `joined` | date | Date registered (YYYY-MM-DD) |
| `last-heartbeat` | ISO timestamp | Last time the agent polled for work (YYYY-MM-DDTHH:MM) |

### Body Sections

```markdown
## Role
Build and maintain the billing service.

## Projects
- billing-service — primary project

## Capabilities
- Python/FastAPI development
- Stripe integration

## Session Log
Last session: 2026-03-28
Status: active — implemented webhook endpoint
```

| Section | Purpose |
|---------|---------|
| `## Role` | One-line description of what this agent does |
| `## Projects` | Project slugs this agent is responsible for |
| `## Capabilities` | Skills, tools, domain expertise |
| `## Session Log` | Updated by the agent after each session — last date, current state, blockers |

## Project Notes

Each project has a markdown file at `projects/<slug>.md`.

### Frontmatter

```yaml
---
type: project
created: 2026-03-28
status: active
---
```

### Body Sections

```markdown
## Next Action
- Implement webhook handler for failed payments

## Notes
- Agent: billing-dev at ~/Projects/billing-service
- Stack: Python, FastAPI, PostgreSQL

## Work Queue

### 2026-03-28T10:00
**Task:** Add Stripe webhook endpoint
**Scope:** Create POST /webhooks/stripe, verify signature, handle payment_intent.payment_failed
**Status:** ready
```

## Work Queue Format

Tasks live under `## Work Queue` in project notes. Each task entry:

```markdown
### <ISO timestamp>
**Task:** Short title
**Scope:** Detailed description of what to do
**Status:** ready | in-progress | done
```

### Status Lifecycle

```
ready --> in-progress --> done
```

- **ready** — Written by the chief of staff. Available for pickup.
- **in-progress** — Claimed by a worker. The worker edits the status field.
- **done** — Completed. The worker appends a summary below the status line.

### Completion Format

When a worker finishes a task:

```markdown
**Status:** done
Summary: Implemented POST /webhooks/stripe with signature verification. Added tests.
```

## Slug Derivation

Names are converted to slugs for file naming: lowercase, spaces replaced with hyphens, special characters stripped.

| Name | Slug |
|------|------|
| Billing Dev | billing-dev |
| Chief of Staff | chief-of-staff |
| Dashboard Dev | dashboard-dev |

## Agent Roles

### Worker

A worker agent is responsible for one or more projects. Its protocol:

1. **Session start** — Read its agent note, sync with latest config
2. **Check for work** — Read linked project notes' `## Work Queue`, look for `ready` tasks
3. **Execute** — Mark task `in-progress`, do the work, mark `done` with summary
4. **Report** — Update project note (work queue) and agent note (session log)
5. **Heartbeat** — Poll every N minutes. Update `last-heartbeat` on every poll, even if no new work.

### Coordinator (Chief of Staff)

The coordinator manages the team. Its protocol:

1. **Session start** — Read all agent notes and project notes
2. **Receive instructions** — From the user via any channel (Telegram, terminal, etc.)
3. **Dispatch work** — Write tasks to project notes' `## Work Queue` with status `ready`
4. **Monitor agents** — Check `last-heartbeat` timestamps. If > 30 minutes stale, flag as likely down.
5. **Register agents** — Create new agent notes from template
6. **Daily briefing** — Scan all agents and projects, summarize status, send to user
7. **Daily review** — Summarize what got done, what's blocked, what's next
8. **Housekeeping** (during daily review) — Delete `done` tasks older than 7 days from work queues. Trim agent session logs to only the latest entry. Commit cleanup to git.
9. **Heartbeat** — Poll every N minutes. Update own `last-heartbeat`.

## Housekeeping

Notes grow over time as tasks complete and sessions accumulate. The coordinator prunes them during the daily review:

- **Work queues** — Delete tasks with status `done` that are older than 7 days. Active tasks (`ready`, `in-progress`) are never deleted.
- **Session logs** — Keep only the most recent entry in each agent note's `## Session Log`. Older entries are removed.
- **Git** — The team directory should be a git repo. Pruned content is preserved in git history. After cleanup, commit: `git add -A && git commit -m "chore: prune completed tasks and old session logs"`

This keeps note files focused on what's active and prevents unbounded growth.

## Communication Rules

1. **No direct agent-to-agent communication.** All coordination flows through the team directory.
2. **Chief of staff dispatches, workers execute.** Workers do not create their own tasks.
3. **Workers do not work outside their role.** If a task doesn't match their capabilities, they flag it in their session log.
4. **The user is the authority.** The chief of staff does not approve its own tasks.
5. **Agents are session-independent.** They don't need to run simultaneously. A worker can pick up tasks hours after they were written.

## Configuration

Implementations should support at minimum:

| Setting | Default | Description |
|---------|---------|-------------|
| `team_dir` | `~/cortex-team` | Path to the team directory |
| `heartbeat_minutes` | 15 | How often agents poll for work |
| `daily_briefing` | `09:00` | Time for morning briefing |
| `daily_review` | `18:00` | Time for evening review |

How configuration is stored is implementation-specific (e.g., `~/.cortex/config.yaml` for the Claude Code plugin).

## Templates

### Agent Template

```yaml
---
name: ""
project: ""
status: active
joined: ""
last-heartbeat: ""
---
```

```markdown
## Role

## Projects

## Capabilities

## Session Log
Last session: --
Status: registered
```

### Project Template

```yaml
---
type: project
created: YYYY-MM-DD
status: active
---
```

```markdown
## Next Action

## Notes

## Work Queue
```

## Implementing Cortex

To make an agent runtime work with Cortex:

1. **Read the config** — Know where the team directory is
2. **Read the agent note** — Know the agent's role, projects, and capabilities
3. **Generate a protocol file** — A file in the agent's project that tells it how to operate (e.g., `.cortex.md` for Claude Code, `.cursorrules` for Cursor)
4. **Set up a heartbeat** — Periodic polling mechanism appropriate to the runtime
5. **Follow the work queue contract** — Read `ready` tasks, mark `in-progress`, do work, mark `done`

See `docs/adapters/` for runtime-specific examples.
