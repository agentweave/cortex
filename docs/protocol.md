# Cortex Protocol Specification

Cortex coordinates a team of AI agents through a shared folder of markdown files. This document defines the protocol — the folder structure, file formats, and coordination rules that any agent runtime can implement.

The Claude Code plugin is one implementation. Any agent that can read and write files can participate.

## Team Directory

A shared folder (the "team directory") contains all coordination state:

```
<team_dir>/
  agents/
    chief-of-staff/
      chief-of-staff.md
      tasks.md
    billing-dev/
      billing-dev.md
      tasks.md
    dashboard-dev/
      dashboard-dev.md
      tasks.md
  projects/
    billing-service.md
    dashboard.md
  templates/
    agent-template.md
    project-template.md
```

The team directory can be any folder — a git repo, an Obsidian vault, a Dropbox folder, or a plain directory on a shared filesystem.

## Agent Notes

Each agent has a markdown file at `agents/<slug>/<slug>.md`.

### Frontmatter (YAML)

```yaml
---
name: Billing Dev
project: ~/Projects/billing-service
status: active
joined: 2026-03-28
---
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name |
| `project` | string | Absolute path to the agent's working directory |
| `status` | `active` \| `inactive` | Whether the agent is on the team |
| `joined` | date | Date registered (YYYY-MM-DD) |

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
```

## Task File

Each agent has a task file at `agents/<slug>/tasks.md`. Tasks follow the [Shuttle Protocol](https://github.com/agentweave/shuttle/blob/main/docs/PROTOCOL.md) format:

```markdown
## <title>
**Status:** <status>
**Started:** <timestamp>
**Completed:** <timestamp>

<body>
```

### Status Lifecycle

```
ready --> in-progress --> done
```

- **ready** — Written by the chief of staff. Available for pickup. No Started/Completed timestamps yet.
- **in-progress** — Claimed by the worker. The worker adds `**Started:** YYYY-MM-DDTHH:MM`.
- **done** — Completed. The worker sets `**Completed:** YYYY-MM-DDTHH:MM` and appends a `### Summary`.

Tasks are picked up in document order (first `ready` task in the file wins).

### Completion Format

When a worker finishes a task:

```markdown
## Add Stripe webhook endpoint
**Status:** done
**Started:** 2026-03-28T10:15
**Completed:** 2026-03-28T11:42

Create POST /webhooks/stripe, verify signature, handle payment_intent.payment_failed

### Summary
Implemented POST /webhooks/stripe with signature verification. Added tests.
```

### Liveness

Each task file ends with a liveness timestamp:

```markdown
<!-- cortex:last-tick YYYY-MM-DDTHH:MM -->
```

Agents update this comment on every poll cycle.

### Idle Suppression

Workers skip tick updates when they have no active tasks (no `ready` or `in-progress` entries in their task file). The coordinator uses context-aware staleness detection:

- **Stale tick + active tasks** = agent is likely **down** (flag for attention)
- **Stale tick + no active tasks** = agent is **idle** (normal, no action needed)

### External Task Runners

When an agent uses an external task runner (e.g., [Shuttle](https://github.com/agentweave/shuttle)) for granular task execution, the Cortex task file acts as a monitoring wrapper rather than the execution layer.

**Pattern:**

1. The user starts a task list in the external runner (e.g., `/shuttle:kickoff` creates `.shuttle.md` in the agent's project)
2. The coordinator writes a single Cortex task to the agent's task file: "Execute Shuttle tasks — report status"
3. The **external runner's heartbeat** does the actual work (picks up items, executes, marks done)
4. The **Cortex heartbeat** reads the external task file and updates progress in the Cortex task
5. When all external tasks are complete, the Cortex heartbeat marks the Cortex task as done

**Key principles:**

- The external runner has no knowledge of Cortex — it runs independently
- Cortex does not duplicate execution — the Cortex tick reports, the external tick works
- The coordinator sees one task per agent, not every granular item
- Both heartbeats run concurrently with different concerns: execution vs. coordination

This pattern applies to any task runner that manages its own file — Cortex wraps it for team-level visibility.

## Slug Derivation

Names are converted to slugs for file and folder naming: lowercase, spaces replaced with hyphens, special characters stripped.

| Name | Slug | Agent Note | Task File |
|------|------|------------|-----------|
| Billing Dev | billing-dev | `agents/billing-dev/billing-dev.md` | `agents/billing-dev/tasks.md` |
| Chief of Staff | chief-of-staff | `agents/chief-of-staff/chief-of-staff.md` | `agents/chief-of-staff/tasks.md` |
| Dashboard Dev | dashboard-dev | `agents/dashboard-dev/dashboard-dev.md` | `agents/dashboard-dev/tasks.md` |

## Agent Roles

### Worker

A worker agent is responsible for one or more projects. Its protocol:

1. **Session start** — Read its agent note, sync with latest config
2. **Check for work** — Read `agents/<slug>/tasks.md`, look for the first `ready` task
3. **Execute** — Mark task `in-progress` with `**Started:**` timestamp, do the work, mark `done` with `**Completed:**` timestamp and `### Summary`
4. **Report** — Update agent note (session log)
5. **Heartbeat** — Update `<!-- cortex:last-tick -->` in task file on every poll. Idle ticks suppressed when no active tasks.

### Coordinator (Chief of Staff)

The coordinator manages the team. Its protocol:

1. **Session start** — Read all agent notes, task files, and project notes
2. **Receive instructions** — From the user via any channel (Telegram, terminal, etc.)
3. **Dispatch work** — Write tasks to `agents/<slug>/tasks.md` with status `ready`
4. **Monitor agents** — Read `<!-- cortex:last-tick -->` from each agent's task file. Context-aware staleness: stale + active tasks = likely down, stale + no active tasks = idle (normal).
5. **Register agents** — Create agent subfolders (`agents/<slug>/`) with agent note and task file
6. **Daily briefing** — Scan all agents and projects, summarize status, send to user
7. **Daily review** — Summarize what got done, what's blocked, what's next
8. **Housekeeping** (during daily review) — Delete `done` tasks older than 7 days from task files (using `**Completed:**` timestamp). Trim agent session logs to only the latest entry. Commit cleanup to git.
9. **Heartbeat** — Update `<!-- cortex:last-tick -->` in own task file.

## Housekeeping

Notes grow over time as tasks complete and sessions accumulate. The coordinator prunes them during the daily review:

- **Task files** — Delete tasks with status `done` whose `**Completed:**` timestamp is older than 7 days. Active tasks (`ready`, `in-progress`) are never deleted.
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
```

## Implementing Cortex

To make an agent runtime work with Cortex:

1. **Read the config** — Know where the team directory is
2. **Read the agent note** — Know the agent's role, projects, and capabilities
3. **Generate a protocol file** — A file in the agent's project that tells it how to operate (e.g., `.cortex.md` for Claude Code, `.cursorrules` for Cursor)
4. **Set up a heartbeat** — Periodic polling mechanism that updates `<!-- cortex:last-tick -->` in the agent's task file
5. **Follow the task file contract** — Read `agents/<slug>/tasks.md`, pick up `ready` tasks in document order, mark `in-progress` with `**Started:**`, do work, mark `done` with `**Completed:**` and `### Summary`

See `docs/adapters/` for runtime-specific examples.
