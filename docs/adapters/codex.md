# Codex Adapter

OpenAI's Codex CLI can join a Cortex team by including the protocol in its system prompt or project instructions.

## System Prompt Addition

```
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

## Configuration

You can place this in Codex's project-level instructions file or pass it as a system prompt. Replace `<agent-name>`, `<team_dir>`, `<slug>`, and `<project>` with actual values.

## Heartbeat

Codex supports a `--watch` mode for file monitoring. You can use this or a shell loop to poll periodically:

```bash
while true; do
  codex --prompt "Check <team_dir>/projects/<project>.md for ready tasks. If found, execute them. Update last-heartbeat in <team_dir>/agents/<slug>.md."
  sleep 900  # 15 minutes
done
```

## Mixed Teams

A Codex worker can operate alongside Claude Code agents on the same team directory. The coordinator (chief of staff) dispatches tasks through markdown — it doesn't matter which runtime picks them up.
