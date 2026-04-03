# Gemini CLI Adapter

Google's Gemini CLI can join a Cortex team by including the protocol in its project configuration.

## GEMINI.md Addition

Add this to your project's `GEMINI.md` file:

```
## Cortex Protocol

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

Replace `<agent-name>`, `<team_dir>`, and `<slug>` with actual values from your agent note.

## Heartbeat

Use a shell loop for periodic polling:

```bash
while true; do
  gemini -p "Check <team_dir>/agents/<slug>/tasks.md for ready tasks. If found, execute them. Update <!-- cortex:last-tick --> in the task file."
  sleep 900  # 15 minutes
done
```

## Mixed Teams

A Gemini CLI worker can operate alongside Claude Code or Codex agents on the same team directory. The task file format is plain markdown — any agent that can read and write files can participate.
