# Cortex

Give your agents a chief of staff.

Cortex is an open protocol for coordinating AI agents through plain markdown files. It ships with a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin, but the protocol works with any agent runtime — Codex, Cursor, Gemini CLI, or anything that can read and write files.

A **chief of staff** agent manages **worker agents** across projects. The chief of staff receives your instructions, dispatches work to agents, and monitors progress — all through plain markdown with YAML frontmatter.

**Not using Claude Code?** See the [protocol spec](docs/protocol.md) and [adapter examples](docs/adapters/) for your runtime.

## How It Works

```
You --> Chief of Staff (Claude Code) --> Team Directory (markdown files) <-- Worker Agents (Claude Code)
```

- **Team directory** — a shared folder of markdown files. Agent notes, project notes, and work queues. Can be an Obsidian vault, a git repo, or any directory.
- **Chief of staff** — a coordinator agent that receives your instructions, dispatches tasks to workers, and sends daily briefings. Talk to it via Telegram (recommended), terminal, or remote control.
- **Worker agents** — project-specific agents that poll for work, execute tasks, and report status.
- **No direct agent-to-agent communication.** All coordination happens through the shared markdown files.

## Quick Start

**1. Install the plugin**

From a local clone:

```bash
claude plugins marketplace add /path/to/cortex --scope user
claude plugins install cortex@agentweave
```

From GitHub:

```bash
claude plugins marketplace add https://github.com/agentweave/cortex --scope user
claude plugins install cortex@agentweave
```

The marketplace name (`agentweave`) comes from `.claude-plugin/marketplace.json` in the repo. This file is required for the plugin system to discover and install the plugin.

**2. Run setup**

In your terminal, start Claude Code and run:

```
/setup
```

This creates your team directory, config, and chief of staff agent note. It will ask for:
- Team directory path (default: `~/cortex-team`)
- Telegram chat ID (optional — for mobile access via the Telegram plugin)
- Heartbeat interval and daily schedule times

**3. Join as chief of staff**

In the chief of staff's project directory:

```
/join Chief of Staff
```

This generates `.cortex.md` (the protocol file), updates `CLAUDE.local.md`, and automatically starts the heartbeat (polls for work every 15 min). The heartbeat restarts on each session when `/join` runs.

**4. Register a worker agent**

As the chief of staff (or in any session):

```
/register Billing Dev
```

Then in the worker's project directory:

```
/join Billing Dev
```

## What You Get

When an agent runs `/join`, two files are created in the project:

**`.cortex.md`** — the full protocol that tells the agent how to operate:

```markdown
# Cortex Protocol

You are **Billing Dev**, a member of Cortex — a coordinated team of AI agents.

## Your Identity
- Agent note: agents/billing-dev.md in the team directory
- Project: ~/Projects/billing-service
- Team directory: ~/cortex-team

## Protocol
### Checking for Work
- Read your project's ## Work Queue
- Pick up tasks with status "ready"
- Mark "in-progress", do the work, mark "done"

### Heartbeat
- Poll every 15 minutes for new work
...
```

**`CLAUDE.local.md`** — a one-line reference so the agent auto-syncs on session start (untracked, agent-specific):

```markdown
## Cortex
On session start, run /join Billing Dev to sync with Cortex. See .cortex.md for full protocol.
```

## How Agents Communicate

Agents never talk to each other directly. All coordination flows through the team directory — a shared folder of markdown files that acts as a message bus.

- **Chief of staff writes tasks** to project notes' `## Work Queue` sections
- **Workers poll** their project's work queue on a heartbeat, pick up `ready` tasks, and report `done`
- **Heartbeat timestamps** — every agent updates a `last-heartbeat` field in its agent note on each poll. The chief of staff checks these to detect agents that have gone down (> 30 min stale)
- **Status flows back** through agent notes' `## Session Log` — the chief of staff reads these to monitor progress

This means agents don't need to run simultaneously. A worker can pick up tasks hours after the chief of staff wrote them.

### Housekeeping

The team directory should be a git repo. During the daily review, the chief of staff prunes old data:

- **Done tasks** older than 7 days are deleted from work queues
- **Session logs** are trimmed to only the latest entry per agent
- **Git preserves history** — pruned content lives in `git log` if you ever need it

## Work Queue

Tasks live in project notes under `## Work Queue`. The chief of staff writes them, workers execute them.

**Status lifecycle:** `ready` → `in-progress` → `done`

Example:

```markdown
## Work Queue

### 2026-03-28T10:00
**Task:** Add Stripe webhook endpoint for payment failures
**Scope:** Create POST /webhooks/stripe, verify signature, handle payment_intent.payment_failed events
**Status:** ready
```

When a worker picks it up:

```markdown
**Status:** in-progress
```

When complete:

```markdown
**Status:** done
Summary: Implemented POST /webhooks/stripe with signature verification. Handles payment_intent.payment_failed by updating subscription status to past_due. Added tests.
```

## Skills Reference

### /setup

First-time Cortex setup. Creates the team directory structure, writes `~/.cortex/config.yaml`, and registers the chief of staff.

### /register \<name\>

Register a new agent. Creates an agent note in `<team_dir>/agents/` from the template. Asks for the agent's project path, role, and capabilities.

### /join \<name\>

Onboard an agent. Reads the agent note from the team directory, generates `.cortex.md` in the current project with the full protocol, and updates `CLAUDE.local.md`. Idempotent — safe to re-run as a sync.

For the chief of staff, generates an extended protocol with coordinator duties (dispatch, monitoring, daily briefing/review).

### /leave \<name\>

Offboard an agent. Removes `.cortex.md`, cleans `CLAUDE.local.md`, and sets agent status to inactive in the team directory.

## Configuration

`~/.cortex/config.yaml`:

```yaml
team_dir: ~/cortex-team
heartbeat_minutes: 15
daily_briefing: "09:00"
daily_review: "18:00"
telegram_chat_id: "your-chat-id"
chief_of_staff_project: "~/Projects/chief-of-staff"
```

Telegram is handled by the separate [Claude Code Telegram plugin](https://docs.anthropic.com/en/docs/claude-code). Cortex only needs the `telegram_chat_id` to know where to send messages. If you skip Telegram, the chief of staff works fine via terminal or remote control.

## Team Directory Structure

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
    project-template.md
```

## Slug Derivation

Agent names are converted to slugs for file naming: lowercase, spaces replaced with hyphens, special characters stripped.

| Name | Slug | File |
|------|------|------|
| Billing Dev | billing-dev | `agents/billing-dev.md` |
| Chief of Staff | chief-of-staff | `agents/chief-of-staff.md` |
| Dashboard Dev | dashboard-dev | `agents/dashboard-dev.md` |

## Plugin Structure

```
cortex/
  .claude-plugin/
    plugin.json          — plugin metadata
    marketplace.json     — marketplace definition (required for installation)
  skills/
    setup/        — first-time setup
    join/         — agent onboarding
    leave/        — agent offboarding
    register/      — agent registration
  README.md
  LICENSE
```

## Other Runtimes

Cortex is runtime-agnostic. The [protocol spec](docs/protocol.md) defines the team directory format, agent notes, work queues, and coordination rules. Any agent that can read and write files can participate.

Adapter examples:
- [Codex](docs/adapters/codex.md)
- [Cursor](docs/adapters/cursor.md)
- [Gemini CLI](docs/adapters/gemini-cli.md)
- [OpenCode](docs/adapters/opencode.md)
- [Generic (any agent)](docs/adapters/generic.md)

A mixed team — Claude Code chief of staff coordinating Codex and Gemini workers — works out of the box. The team directory is the universal interface.

## License

MIT
