---
name: leave
description: Leave the Cortex agent team — removes .cortex.md, cleans CLAUDE.local.md, sets agent status to inactive in team directory.
user_invocable: true
argument-hint: <agent-name>
---

# /leave — Leave the Cortex Agent Team

Remove this agent from Cortex by cleaning up local protocol files and updating the team directory.

## Arguments

`<agent-name>` — the agent's name (e.g., "Billing Dev"). If not provided, ask: "What is this agent's name in Cortex?"

## Steps

### 1. Resolve name and slug

Use the provided `<agent-name>`.

Derive the slug: lowercase, replace spaces with hyphens, strip any characters that are not alphanumeric or hyphens.

### 2. Check current state

Check if `.cortex.md` exists in the current working directory.
Check if `CLAUDE.local.md` contains "## Cortex".

If neither exists, respond:
> "This agent doesn't appear to be on the Cortex team. Nothing to clean up."

Stop.

### 3. Remove .cortex.md

If .cortex.md exists in the current working directory, delete it:

```bash
rm .cortex.md
```

### 4. Clean CLAUDE.local.md

If CLAUDE.local.md contains a `## Cortex` section (the line `## Cortex` through the next `##` heading or end of file), remove that entire section using the Edit tool.

### 5. Cancel heartbeat and clean up hooks

**5a.** Use CronList to find any active cron whose prompt starts with `Cortex ` (with trailing space). If found, cancel each with CronDelete. If no cron is found, skip — the agent may not have had a heartbeat running.

**5b.** If `.claude/hooks/cortex-precheck.sh` exists, delete it:

```bash
rm -f .claude/hooks/cortex-precheck.sh
```

**5c.** Read `.claude/settings.json`. If it contains a hook entry with `"matcher": "Cortex tick:"` in the `hooks.UserPromptSubmit` array, remove that entry. Preserve all other hook entries. Write the updated JSON back.

### 6. Update agent note in team directory

Read `~/.cortex/config.yaml` to get `team_dir`.

If config exists and `<team_dir>/agents/<slug>/<slug>.md` exists, read the agent note and check the current `status` field:

- If `status: active`, replace with `status: inactive` using the Edit tool.
- If `status: inactive` (already inactive), skip — no change needed.

If the config or agent note doesn't exist, skip this step (already cleaned up or never registered).

### 7. Confirm

Respond:

> "**{name}** has left Cortex. .cortex.md removed, CLAUDE.local.md cleaned, team directory status set to inactive."
