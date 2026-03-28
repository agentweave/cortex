---
name: leave-cortex
description: Leave the Cortex agent team — removes TEAM.md, cleans CLAUDE.md, sets agent status to inactive in team directory.
user_invocable: true
argument-hint: <agent-name>
---

# /leave-cortex — Leave the Cortex Agent Team

Remove this agent from Cortex by cleaning up local protocol files and updating the team directory.

## Arguments

`<agent-name>` — the agent's name (e.g., "Billing Dev"). If not provided, ask: "What is this agent's name in Cortex?"

## Steps

### 1. Resolve name and slug

Use the provided `<agent-name>`.

Derive the slug: lowercase, replace spaces with hyphens.

### 2. Check current state

Check if `TEAM.md` exists in the current working directory.
Check if `CLAUDE.md` contains "join-cortex".

If neither exists, respond:
> "This agent doesn't appear to be on the Cortex team. Nothing to clean up."

Stop.

### 3. Remove TEAM.md

If TEAM.md exists in the current working directory, delete it:

```bash
rm TEAM.md
```

### 4. Clean CLAUDE.md

If CLAUDE.md contains a `## Cortex` section (the line `## Cortex` through the next `##` heading or end of file), remove that entire section using the Edit tool.

### 5. Update agent note in team directory

Read `~/.cortex/config.yaml` to get `team_dir`.

If config exists and `<team_dir>/agents/<slug>.md` exists, read the agent note and check the current `status` field:

- If `status: active`, replace with `status: inactive` using the Edit tool.
- If `status: inactive` (already inactive), skip — no change needed.

If the config or agent note doesn't exist, skip this step (already cleaned up or never registered).

### 6. Confirm

Respond:

> "**{name}** has left Cortex. TEAM.md removed, CLAUDE.md cleaned, team directory status set to inactive."
