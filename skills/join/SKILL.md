---
name: join
description: Join the Cortex agent team — reads agent config from team directory, generates .cortex.md protocol, updates CLAUDE.local.md. Idempotent — safe to re-run as a sync.
user_invocable: true
argument-hint: <agent-name>
---

# /join — Join the Cortex Agent Team

Onboard this agent to Cortex by reading its config from the team directory and generating the local protocol file.

## Arguments

`<agent-name>` — the agent's name (e.g., "Billing Dev", "Chief of Staff"). If not provided, ask: "What is this agent's name in Cortex?"

## Steps

### 1. Read config

Read `~/.cortex/config.yaml` to get `team_dir`, `heartbeat_minutes`, `daily_briefing`, and `daily_review`.

If config doesn't exist, stop and respond:
> "Cortex is not set up yet. Run `/setup` first."

### 2. Resolve name and slug

Use the provided `<agent-name>`.

Derive the slug: lowercase, replace spaces with hyphens, strip any characters that are not alphanumeric or hyphens.

### 3. Read agent note

Read `<team_dir>/agents/<slug>/<slug>.md`.

If the file doesn't exist, stop and respond:
> "Agent `<name>` is not registered in Cortex. Run `/register <name>` to create the agent note first."

### 4. Extract config from agent note

From the frontmatter and body, extract:
- `name` — from frontmatter
- `slug` — derived from name
- `project` — from frontmatter (the project directory path)
- `projects` — from ## Projects section (list of project slugs)
- `role` — from ## Role section (first line after heading)

### 5. Determine if this is a re-run

Check if `.cortex.md` exists in the current working directory. If it does, this is a sync (re-run).

### 6. Generate .cortex.md

Write `.cortex.md` in the current working directory.

If the agent slug is `chief-of-staff`, generate the **coordinator .cortex.md** (see Section A below).

Otherwise, generate the **worker .cortex.md** (see Section B below).

Replace all `{placeholder}` values with the actual extracted values.

### 7. Update CLAUDE.local.md

Read the CLAUDE.local.md in the current working directory.

If it exists and does NOT already contain the text "## Cortex":
- Append this section at the very end:

```
## Cortex
On session start, run /join <name> to sync with Cortex. See .cortex.md for full protocol.
```

If CLAUDE.local.md doesn't exist, create it with just that content.

If CLAUDE.local.md already contains "## Cortex", do nothing (already configured).

### 8. Start heartbeat

**8a. Clean up existing cron (idempotency)**

Use CronList to check for any cron whose prompt starts with `Cortex `. If found, use CronDelete to remove it.

**8b. Collision detection**

Read `<team_dir>/agents/<slug>/tasks.md`. Look for a comment matching:
```
<!-- cortex:last-tick YYYY-MM-DDTHH:MM -->
```

If found, parse the timestamp. If the timestamp is less than `{heartbeat_minutes}` minutes ago, ask the user:

> "A Cortex heartbeat was active X minutes ago (possibly in another session). Start anyway?"

If the user says no, skip to Step 9 (confirm) without starting the heartbeat. If yes (or no timestamp found), continue.

**8c. Create the heartbeat cron**

Convert the interval to a cron expression: `{heartbeat_minutes}` becomes `*/{heartbeat_minutes} * * * *`.

**For worker agents** (slug is NOT `chief-of-staff`), use CronCreate with:
- cron: `*/{heartbeat_minutes} * * * *`
- prompt:

```
Cortex tick: read {team_dir}/agents/{slug}/tasks.md and continue working.

1. Read {team_dir}/agents/{slug}/tasks.md
2. If a task is in-progress, continue it (you may be resuming from a previous session — read progress notes carefully)
3. If no task is in-progress, pick the first ready task, mark it in-progress with **Started:** timestamp
4. Do the work
5. Update progress in tasks.md as you go
6. When complete, mark done with **Completed:** timestamp and ### Summary
7. If more ready tasks remain, pick up the next one
8. Update the last-tick timestamp: <!-- cortex:last-tick YYYY-MM-DDTHH:MM -->
```

**For chief of staff** (slug IS `chief-of-staff`), use CronCreate with:
- cron: `*/{heartbeat_minutes} * * * *`
- prompt:

```
Cortex custom: Coordinator poll per .cortex.md protocol.

1. Check for user messages (via Telegram if configured)
2. Read all agent task files in {team_dir}/agents/*/tasks.md:
   - Parse cortex:last-tick for staleness (> 30 min AND has active tasks = likely down)
   - Check task statuses for progress updates
3. Read all project notes in {team_dir}/projects/ for context
4. Flag any issues to the user
5. Update <!-- cortex:last-tick YYYY-MM-DDTHH:MM --> in {team_dir}/agents/chief-of-staff/tasks.md
```

**8d. Install idle suppression hook (workers only)**

Skip this step if the agent slug is `chief-of-staff`.

Create the `.claude/hooks/` directory if it doesn't exist. Write the following script to `.claude/hooks/cortex-precheck.sh`:

```bash
#!/bin/bash
# Cortex idle suppression pre-check
# Blocks Cortex tick prompts when no active tasks exist.
# Installed by /cortex:join, removed by /cortex:leave.

TASK_FILE="{team_dir}/agents/{slug}/tasks.md"

if grep -q '^\*\*Status:\*\* \(ready\|in-progress\)' "$TASK_FILE" 2>/dev/null; then
  exit 0
else
  echo '{"decision":"block","reason":"No active tasks"}' >&2
  exit 2
fi
```

Replace `{team_dir}` and `{slug}` with the actual resolved values.

Make the script executable using Bash: `chmod +x .claude/hooks/cortex-precheck.sh`.

Read `.claude/settings.json` in the current project directory. If it doesn't exist, create it. If it exists, parse the existing JSON.

If the `hooks.UserPromptSubmit` array does NOT already contain an entry with `"matcher": "Cortex tick:"`, add the following entry to the array:

```json
{
  "matcher": "Cortex tick:",
  "hooks": [
    {
      "type": "command",
      "command": ".claude/hooks/cortex-precheck.sh"
    }
  ]
}
```

Preserve all existing hook entries. If `hooks` or `hooks.UserPromptSubmit` keys don't exist, create them.

### 9. Confirm

If this was a first join (.cortex.md didn't exist before):

> "Joined Cortex as **{name}**. .cortex.md generated, CLAUDE.local.md updated, heartbeat started (every {heartbeat_minutes} min)."

If this was a re-run (.cortex.md already existed):

> "Synced **{name}** with latest Cortex config. .cortex.md regenerated, heartbeat restarted."

---

## Section A: Coordinator .cortex.md (Chief of Staff)

Generate this .cortex.md when the agent slug is `chief-of-staff`:

```
# Cortex Protocol

You are **{name}**, the coordinator of Cortex — a team of AI agents.

## Your Identity
- Agent note: agents/{slug}/{slug}.md in the team directory
- Task file: agents/{slug}/tasks.md
- Project: {project}
- Team directory: {team_dir}

## Team Directory Access
- Path: {team_dir}
- Use Claude Code's built-in Read, Write, Edit, Grep, and Glob tools to access files

## Protocol

### Session Start
1. Run /join {name} to sync with latest config

### Coordinator Duties
2. Check for user messages (via Telegram if configured, or wait for terminal/remote control input)
3. Read all agent task files in agents/*/tasks.md:
   - Parse `<!-- cortex:last-tick -->` for staleness. If > 30 min AND agent has active tasks (`**Status:** ready` or `in-progress`), flag as likely down. If stale with no active tasks, agent is idle (normal).
   - Check task statuses for progress updates
4. Read all project notes in projects/ for context
5. Flag any issues to the user

### Dispatching Work
6. To assign work, write a task entry to the relevant agent's task file (`agents/<slug>/tasks.md`):

   ## <descriptive title>
   **Status:** ready

   <description of what to do>

7. The assigned agent will pick up the task on its next heartbeat

### Registering New Agents
8. Run /register <name> to create a new agent note and task file
9. Tell the user to run /join <name> in the agent's project directory

### Daily Briefing ({daily_briefing})
10. Scan all agents and projects
11. Summarize: who is active, what's in progress, what's blocked, what's coming up
12. Send to the user (via Telegram if configured, otherwise output in session)

### Daily Review ({daily_review})
13. Summarize what got done today, what's blocked, what's planned for tomorrow
14. Send to the user (via Telegram if configured, otherwise output in session)

### Housekeeping (during daily review)
15. Prune completed tasks: delete tasks with status "done" that are older than 7 days (by **Completed:** timestamp) from all agent task files
16. Trim agent session logs: keep only the most recent entry in each agent note's ## Session Log — delete older entries
17. Commit the cleanup: `git add -A && git commit -m "chore: prune completed tasks and old session logs"` in the team directory

### Heartbeat
18. Poll the team directory every {heartbeat_minutes} minutes for:
    - New user messages (via Telegram if configured)
    - Agent liveness — read `<!-- cortex:last-tick -->` from each agent's tasks.md
    - Blocked agents (check for blockers in agent notes)
19. On every heartbeat poll, update `<!-- cortex:last-tick YYYY-MM-DDTHH:MM -->` in agents/chief-of-staff/tasks.md

### Reporting Back
20. After each session, update your agent note (agents/{slug}/{slug}.md ## Session Log) with:
    - Last session date
    - Current state and any blockers

## Communication
- The user communicates with you via Telegram (if configured), terminal, or remote control
- You coordinate agents by writing to the team directory — never by communicating directly with agents
- Do NOT execute work on other agents' projects — delegate by writing tasks
- Do NOT approve your own tasks — the user is the authority
```

## Section B: Worker .cortex.md

Generate this .cortex.md for all non-chief-of-staff agents:

```
# Cortex Protocol

You are **{name}**, a member of Cortex — a coordinated team of AI agents.

## Your Identity
- Agent note: agents/{slug}/{slug}.md in the team directory
- Task file: agents/{slug}/tasks.md
- Project: {project}
- Team directory: {team_dir}

## Team Directory Access
- Path: {team_dir}
- Use Claude Code's built-in Read, Write, Edit, Grep, and Glob tools to access files

## Protocol

### Session Start
1. Run /join {name} to sync with latest config

### Checking for Work
2. Read your task file at {team_dir}/agents/{slug}/tasks.md
3. Pick up tasks with **Status:** ready:
   - Edit the status to `in-progress` and add `**Started:** YYYY-MM-DDTHH:MM`
   - Do the work
   - Edit the status to `done` and add `**Completed:** YYYY-MM-DDTHH:MM`
   - Append a `### Summary` section with what was done

### Reporting Back
4. After completing work, update your agent note (agents/{slug}/{slug}.md ## Session Log) with:
   - Last session date
   - Current state and any blockers

### Heartbeat
5. Poll your task file every {heartbeat_minutes} minutes for new work
6. On every active heartbeat tick, update `<!-- cortex:last-tick YYYY-MM-DDTHH:MM -->` at the end of your task file
7. When no tasks are active, your heartbeat tick is suppressed (no model invocation, no token cost)

## Communication
- The chief of staff monitors your work via the team directory
- If blocked, update your agent note with the blocker — the chief of staff will see it on heartbeat
- Do NOT execute tasks unrelated to your role — flag them instead
```
