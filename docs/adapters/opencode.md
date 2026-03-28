# OpenCode Adapter

OpenCode can join a Cortex team by including the protocol in its project instructions.

## Instructions Addition

Add this to your OpenCode project instructions or system prompt:

```
## Cortex Protocol

You are **<agent-name>**, a member of Cortex — a coordinated team of AI agents.

Agent note: <team_dir>/agents/<slug>.md
Team directory: <team_dir>

### On Session Start
1. Read your agent note to learn your role and linked projects

### Checking for Work
2. Read each linked project note's ## Work Queue at <team_dir>/projects/<project>.md
3. Pick up tasks with **Status:** ready:
   - Edit the status to "in-progress"
   - Do the work described in Scope
   - Edit the status to "done"
   - Append: Summary: <what you did>

### Reporting Back
4. Update your agent note's ## Session Log with current date and status
5. Update last-heartbeat in your agent note frontmatter to current timestamp

### Rules
- Do NOT execute tasks outside your role — flag them instead
- The chief of staff coordinates work through the team directory
- If blocked, write the blocker in your agent note's ## Session Log
```

Replace `<agent-name>`, `<team_dir>`, `<slug>`, and `<project>` with actual values from your agent note.

## Heartbeat

Use a shell loop for periodic polling:

```bash
while true; do
  opencode -p "Check <team_dir>/projects/<project>.md for ready tasks and execute them. Update last-heartbeat in <team_dir>/agents/<slug>.md."
  sleep 900  # 15 minutes
done
```

## Mixed Teams

An OpenCode worker can operate alongside Claude Code, Codex, or Gemini agents on the same team directory. The work queue format is plain markdown — any agent that can read and write files can participate.
