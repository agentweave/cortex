# Cortex

Coordinate a team of AI agents through a shared folder of markdown files.

Cortex is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that sets up a **chief of staff** agent to manage **worker agents** across projects. The chief of staff receives your instructions, dispatches work to agents, and monitors progress — all through plain markdown files with YAML frontmatter.

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
claude plugins install cortex
```

From GitHub:

```bash
claude plugins marketplace add https://github.com/kaichen/cortex --scope user
claude plugins install cortex
```

**2. Run setup**

In your terminal, start Claude Code and run:

```
/setup-cortex
```

This creates your team directory, config, and chief of staff agent note. It will ask for:
- Team directory path (default: `~/cortex-team`)
- Telegram bot token and chat ID (optional, recommended)
- Heartbeat interval and daily schedule times

**3. Join as chief of staff**

In the chief of staff's project directory:

```
/join-cortex Chief of Staff
```

**4. Register a worker agent**

As the chief of staff (or in any session):

```
/register-agent Billing Dev
```

Then in the worker's project directory:

```
/join-cortex Billing Dev
```

## Skills Reference

### /setup-cortex

First-time Cortex setup. Creates the team directory structure, writes `~/.cortex/config.yaml`, and registers the chief of staff.

### /register-agent \<name\>

Register a new agent. Creates an agent note in `<team_dir>/agents/` from the template. Asks for the agent's project path, role, and capabilities.

### /join-cortex \<name\>

Onboard an agent. Reads the agent note from the team directory, generates `TEAM.md` in the current project with the full protocol, and updates `CLAUDE.md`. Idempotent — safe to re-run as a sync.

For the chief of staff, generates an extended protocol with coordinator duties (dispatch, monitoring, daily briefing/review).

### /leave-cortex \<name\>

Offboard an agent. Removes `TEAM.md`, cleans `CLAUDE.md`, and sets agent status to inactive in the team directory.

## Configuration

`~/.cortex/config.yaml`:

```yaml
team_dir: ~/cortex-team
heartbeat_minutes: 15
daily_briefing: "09:00"
daily_review: "18:00"
telegram_bot_token: "your-bot-token"
telegram_chat_id: "your-chat-id"
chief_of_staff_project: "~/Projects/chief-of-staff"
```

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

## License

MIT
