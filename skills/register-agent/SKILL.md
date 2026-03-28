---
name: register-agent
description: Register a new agent in Cortex — creates an agent note in the team directory from the template.
user_invocable: true
argument-hint: <agent-name>
---

# /register-agent — Register a New Cortex Agent

Create an agent note for a new team member. Run this as the chief of staff or from any session with access to the team directory.

## Arguments

`<agent-name>` — the agent's display name (e.g., "Billing Dev", "Dashboard Dev"). If not provided, ask: "What is the new agent's name?"

## Steps

### 1. Read config

Read `~/.cortex/config.yaml` to get `team_dir`.

If config doesn't exist, stop and respond:
> "Cortex is not set up yet. Run `/setup-cortex` first."

### 2. Resolve name and slug

Use the provided `<agent-name>`.

Derive the slug: lowercase, replace spaces with hyphens (e.g., "Billing Dev" -> "billing-dev").

### 3. Check for existing agent

Check if `<team_dir>/agents/<slug>.md` already exists.

If it does, stop and respond:
> "Agent `<name>` is already registered at `agents/<slug>.md`."

### 4. Gather agent details

Ask: "What is the project directory for this agent? (absolute path, e.g., ~/Projects/billing-service)"

Ask: "What is this agent's role? (one line, e.g., 'Build and maintain the billing service')"

Ask: "What are this agent's capabilities? (comma-separated, e.g., 'Python, FastAPI, database migrations')"

### 5. Read template

Read `<team_dir>/templates/agent-template.md`.

If it doesn't exist, use this default template structure:

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

### 6. Create agent note

Write `<team_dir>/agents/<slug>.md` with the template filled in:

- `name`: the display name
- `project`: the project directory path
- `status`: active
- `joined`: today's date (YYYY-MM-DD)
- `## Role`: the role description
- `## Projects`: the project slug derived from the project path basename
- `## Capabilities`: the capabilities list (as bullet points, one per line)
- `## Session Log`: initialized with "Last session: --" and "Status: registered"

### 7. Confirm

Respond:

> "Registered **<name>** in Cortex.
>
> Agent note: `<team_dir>/agents/<slug>.md`
>
> **Next step:** In the agent's project directory (`<project_path>`), run:
> ```
> /join-cortex <name>
> ```"
