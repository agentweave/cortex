---
name: setup
description: First-time Cortex setup — creates team directory, config, and chief of staff agent.
user_invocable: true
---

# /setup — First-Time Cortex Setup

Set up Cortex for coordinating a team of AI agents through a shared markdown folder.

## Steps

### 1. Check for existing config

Read `~/.cortex/config.yaml`. If it exists, show the current config and ask: "Cortex is already configured. Do you want to reconfigure?" If no, stop.

### 2. Ask for team directory

Ask: "Where should the team directory live? This is the shared folder where agent notes, project notes, and templates are stored."

Default: `~/cortex-team`

Expand `~` to the full home directory path for all file operations, but store the path with `~` in config for readability.

### 3. Create team directory structure

```bash
mkdir -p <team_dir>/agents
mkdir -p <team_dir>/projects
mkdir -p <team_dir>/templates
```

### 4. Write agent template

Write `<team_dir>/templates/agent-template.md`:

```
---
name: ""
project: ""
status: active
joined: ""
---

## Role


## Projects


## Capabilities


## Session Log
Last session: --
Status: registered
```

### 5. Write project template

Write `<team_dir>/templates/project-template.md`:

```
---
type: project
created: YYYY-MM-DD
status: active
---

## Next Action

## Notes
```

### 6. Ask for Telegram chat ID (optional)

Ask: "Would you like to connect Telegram so you can message the chief of staff from your phone? (Recommended, but you can also use terminal or remote control instead.)"

If yes: ask for their Telegram chat ID. The bot token is managed separately by the Claude Code Telegram plugin (`/telegram:configure`) — Cortex only needs the chat ID to know where to send messages.

> "To find your chat ID:
> 1. If you already have a Telegram bot configured with the Claude Code Telegram plugin, your chat ID is in the plugin's config.
> 2. Otherwise, set up the Telegram plugin first with `/telegram:configure`, then come back to provide your chat ID.
>
> Paste your chat ID when ready."

If they want to skip Telegram: set the value to an empty string in config. The chief of staff will work fine via terminal or remote control — Telegram just adds mobile access.

### 7. Ask for chief of staff project directory

Ask: "Where does the chief of staff agent run? This is the project directory where the chief of staff's Claude Code session lives."

Default: the team directory itself.

### 8. Ask for heartbeat and schedule preferences

Ask: "How often should agents poll for new work (in minutes)?"
Default: 15

Ask: "What time should the daily briefing be sent? (HH:MM, 24h format)"
Default: 09:00

Ask: "What time should the daily review be sent? (HH:MM, 24h format)"
Default: 18:00

### 9. Write config

Create the `~/.cortex/` directory if it doesn't exist, then write `~/.cortex/config.yaml`:

```yaml
team_dir: <team_dir>
heartbeat_minutes: <heartbeat>
daily_briefing: "<briefing_time>"
daily_review: "<review_time>"
telegram_chat_id: "<chat_id>"
chief_of_staff_project: "<cos_project_dir>"
```

### 10. Create chief of staff agent

```bash
mkdir -p <team_dir>/agents/chief-of-staff
```

Write `<team_dir>/agents/chief-of-staff/chief-of-staff.md`:

```
---
name: Chief of Staff
project: <cos_project_dir>
status: active
joined: <today's date YYYY-MM-DD>
---

## Role
Coordinate the Cortex agent team — receive instructions from the user, dispatch work to agents, monitor progress, send daily briefings and reviews.

## Projects
(All projects — the chief of staff monitors every project in the team directory)

## Capabilities
- Agent registration and onboarding via /register
- Work dispatch — write tasks to agent task files
- Agent monitoring — read task files for liveness and progress
- Daily briefing (morning) and review (evening)
- User communication via Telegram, terminal, or remote control

## Session Log
Last session: --
Status: registered
```

Write `<team_dir>/agents/chief-of-staff/tasks.md` as an empty file.

### 11. Confirm

Respond:

> "Cortex is set up!
>
> - Team directory: `<team_dir>`
> - Config: `~/.cortex/config.yaml`
> - Chief of staff registered at `agents/chief-of-staff/chief-of-staff.md`
>
> **Next steps:**
> 1. In the chief of staff's project (`<cos_project_dir>`), run `/join Chief of Staff`
> 2. To add worker agents, have the chief of staff run `/register <name>`
> 3. In each worker's project, run `/join <name>`"
