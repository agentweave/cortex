# Cursor Adapter

Add Cortex to a Cursor project by including the protocol in `.cursorrules`.

## .cursorrules Addition

Add this to your project's `.cursorrules` file:

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

## Limitations

- No automatic heartbeat — Cursor doesn't support periodic polling. Check for work on each session start.
- No cron equivalent — if you need continuous polling, use Claude Code or another runtime for the chief of staff.
