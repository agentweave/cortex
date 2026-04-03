# Codex Adapter

OpenAI's Codex CLI can join a Cortex team by including the protocol in its system prompt or project instructions.

## System Prompt Addition

```
You are **<agent-name>**, a member of Cortex — a coordinated team of AI agents.

Agent note: <team_dir>/agents/<slug>/<slug>.md
Task file: <team_dir>/agents/<slug>/tasks.md
Team directory: <team_dir>

### On Session Start
1. Read your agent note to learn your role and linked projects

### Checking for Work
2. Read your task file at <team_dir>/agents/<slug>/tasks.md
3. Pick up tasks with **Status:** ready:
   - Edit the status to "in-progress" and add **Started:** timestamp
   - Do the work
   - Edit the status to "done" and add **Completed:** timestamp
   - Append a ### Summary section

### Reporting Back
4. Update your agent note's ## Session Log with current date and status
5. Update <!-- cortex:last-tick YYYY-MM-DDTHH:MM --> at the end of your task file

### Rules
- Do NOT execute tasks outside your role — flag them instead
- The chief of staff coordinates work through the team directory
- If blocked, write the blocker in your agent note's ## Session Log
```

## Configuration

You can place this in Codex's project-level instructions file or pass it as a system prompt. Replace `<agent-name>`, `<team_dir>`, and `<slug>` with actual values.

## Heartbeat

Codex supports a `--watch` mode for file monitoring. You can use this or a shell loop to poll periodically:

```bash
while true; do
  codex --prompt "Check <team_dir>/agents/<slug>/tasks.md for ready tasks. If found, execute them. Update <!-- cortex:last-tick --> in the task file."
  sleep 900  # 15 minutes
done
```

## Mixed Teams

A Codex worker can operate alongside Claude Code agents on the same team directory. The coordinator (chief of staff) dispatches tasks through markdown — it doesn't matter which runtime picks them up.
