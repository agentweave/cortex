# Cortex Plugin Implementation Plan

> **Note:** This is the original implementation plan. The plugin has evolved since — see the skills and README for the current state.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that coordinates a team of AI agents through a shared folder of markdown files.

**Architecture:** Skills-only plugin — four Claude Code skills (/setup-cortex, /register-agent, /join-cortex, /leave-cortex) that read/write a shared team directory of markdown files. Configuration stored in ~/.cortex/config.yaml. No external dependencies beyond Claude Code's built-in file tools.

**Tech Stack:** Claude Code skills (markdown), YAML config, markdown templates

---

## File Structure

```
cortex/
  package.json                    — plugin metadata, declares skills
  LICENSE                         — MIT license
  README.md                       — what it does, quick start, architecture, skill reference
  skills/
    setup-cortex/SKILL.md         — first-time setup: creates team dir, config, chief of staff
    register-agent/SKILL.md       — registers a new agent (creates agent note from template)
    join-cortex/SKILL.md          — worker onboarding (reads agent note, generates TEAM.md)
    leave-cortex/SKILL.md         — worker offboarding (removes TEAM.md, cleans CLAUDE.md)
```

Templates (agent-template.md, TEAM.md protocol) are embedded inline in the skills that write them — not separate files.

---

### Task 1: Package Structure

**Files:**
- Create: `package.json`
- Create: `LICENSE`
- Create: `.gitignore`

- [ ] **Step 1: Initialize git repo**

```bash
cd ~/Projects/cortex && git init
```

- [ ] **Step 2: Create .gitignore**

Create `.gitignore`:

```
.DS_Store
*.swp
*~
```

- [ ] **Step 3: Create LICENSE (MIT)**

Create `LICENSE` with MIT license text. Copyright holder: the repo owner. Year: 2026.

- [ ] **Step 4: Create package.json**

Create `package.json`:

```json
{
  "name": "cortex",
  "version": "0.1.0",
  "description": "Coordinate a team of AI agents through a shared folder of markdown files",
  "skills": [
    "skills/setup-cortex",
    "skills/register-agent",
    "skills/join-cortex",
    "skills/leave-cortex"
  ],
  "license": "MIT"
}
```

- [ ] **Step 5: Create skills directories**

```bash
mkdir -p skills/setup-cortex skills/register-agent skills/join-cortex skills/leave-cortex
```

- [ ] **Step 6: Commit**

```bash
git add .gitignore LICENSE package.json
git commit -m "chore: initialize cortex plugin package"
```

---

### Task 2: /setup-cortex Skill

**Files:**
- Create: `skills/setup-cortex/SKILL.md`

This is the entry point for new users. It creates the team directory, writes config, and sets up the chief of staff agent note.

- [ ] **Step 1: Write the skill**

Create `skills/setup-cortex/SKILL.md`:

```markdown
---
name: setup-cortex
description: First-time Cortex setup — creates team directory, config, and chief of staff agent.
user_invocable: true
---

# /setup-cortex — First-Time Cortex Setup

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

### 5. Ask for Telegram configuration

Ask: "Cortex uses Telegram for the chief of staff to communicate with you. Do you have a Telegram bot set up?"

If yes: ask for the bot token and chat ID.

If no: guide them:
> "To create a Telegram bot:
> 1. Open Telegram and message @BotFather
> 2. Send /newbot and follow the prompts
> 3. Copy the bot token (looks like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)
> 4. Start a chat with your new bot and send any message
> 5. To get your chat ID, visit: `https://api.telegram.org/bot<TOKEN>/getUpdates`
>
> Paste your bot token and chat ID when ready."

If they want to skip Telegram for now, allow it — set both values to empty strings in config.

### 6. Ask for chief of staff project directory

Ask: "Where does the chief of staff agent run? This is the project directory where the chief of staff's Claude Code session lives."

Default: the team directory itself.

### 7. Ask for heartbeat and schedule preferences

Ask: "How often should agents poll for new work (in minutes)?"
Default: 15

Ask: "What time should the daily briefing be sent? (HH:MM, 24h format)"
Default: 09:00

Ask: "What time should the daily review be sent? (HH:MM, 24h format)"
Default: 18:00

### 8. Write config

Write `~/.cortex/config.yaml`:

```yaml
team_dir: <team_dir>
heartbeat_minutes: <heartbeat>
daily_briefing: "<briefing_time>"
daily_review: "<review_time>"
telegram_chat_id: "<chat_id>"
chief_of_staff_project: "<cos_project_dir>"
```

### 9. Create chief of staff agent note

Write `<team_dir>/agents/chief-of-staff.md`:

```
---
name: Chief of Staff
project: <cos_project_dir>
status: active
joined: <today's date YYYY-MM-DD>
---

## Role
Coordinate the Cortex agent team — receive instructions via Telegram, dispatch work to agents, monitor progress, send daily briefings and reviews.

## Projects
(All projects — the chief of staff monitors every project in the team directory)

## Capabilities
- Agent registration and onboarding via /register-agent
- Work dispatch — write tasks to project work queues
- Agent monitoring — read session logs, detect staleness and blockers
- Daily briefing (morning) and review (evening) via Telegram
- Telegram communication with the user

## Session Log
Last session: --
Status: registered
```

### 10. Confirm

Respond:

> "Cortex is set up!
>
> - Team directory: `<team_dir>`
> - Config: `~/.cortex/config.yaml`
> - Chief of staff registered at `agents/chief-of-staff.md`
>
> **Next steps:**
> 1. In the chief of staff's project (`<cos_project_dir>`), run `/join-cortex Chief of Staff`
> 2. To add worker agents, have the chief of staff run `/register-agent <name>`
> 3. In each worker's project, run `/join-cortex <name>`"
```

- [ ] **Step 2: Verify skill by reading it back**

Read `skills/setup-cortex/SKILL.md` and confirm it covers all steps from the spec: team dir creation, template, Telegram config, chief of staff project dir, heartbeat/schedule config, config file, chief of staff agent note.

- [ ] **Step 3: Commit**

```bash
git add skills/setup-cortex/SKILL.md
git commit -m "feat: add /setup-cortex skill for first-time setup"
```

---

### Task 3: /register-agent Skill

**Files:**
- Create: `skills/register-agent/SKILL.md`

Run by the chief of staff to add a new team member.

- [ ] **Step 1: Write the skill**

Create `skills/register-agent/SKILL.md`:

```markdown
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

If it doesn't exist, use the default template structure (see /setup-cortex step 4 for the template content).

### 6. Create agent note

Write `<team_dir>/agents/<slug>.md` with the template filled in:

- `name`: the display name
- `project`: the project directory path
- `status`: active
- `joined`: today's date (YYYY-MM-DD)
- `## Role`: the role description
- `## Projects`: the project slug derived from the project path
- `## Capabilities`: the capabilities list (as bullet points)
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
```

- [ ] **Step 2: Verify skill by reading it back**

Read `skills/register-agent/SKILL.md` and confirm it covers: config check, slug derivation, duplicate check, gathering details, template usage, agent note creation.

- [ ] **Step 3: Commit**

```bash
git add skills/register-agent/SKILL.md
git commit -m "feat: add /register-agent skill for agent registration"
```

---

### Task 4: /join-cortex Skill

**Files:**
- Create: `skills/join-cortex/SKILL.md`

The core onboarding skill. Reads the agent note from the team dir, generates TEAM.md with the full protocol, updates CLAUDE.md. Detects chief of staff and generates extended protocol.

- [ ] **Step 1: Write the skill**

Create `skills/join-cortex/SKILL.md`:

```markdown
---
name: join-cortex
description: Join the Cortex agent team — reads agent config from team directory, generates TEAM.md protocol, updates CLAUDE.md. Idempotent — safe to re-run as a sync.
user_invocable: true
argument-hint: <agent-name>
---

# /join-cortex — Join the Cortex Agent Team

Onboard this agent to Cortex by reading its config from the team directory and generating the local protocol files.

## Arguments

`<agent-name>` — the agent's name (e.g., "Billing Dev", "Chief of Staff"). If not provided, ask: "What is this agent's name in Cortex?"

## Steps

### 1. Read config

Read `~/.cortex/config.yaml` to get `team_dir` and `heartbeat_minutes`.

If config doesn't exist, stop and respond:
> "Cortex is not set up yet. Run `/setup-cortex` first."

### 2. Resolve name and slug

Use the provided `<agent-name>`.

Derive the slug: lowercase, replace spaces with hyphens.

### 3. Read agent note

Read `<team_dir>/agents/<slug>.md`.

If the file doesn't exist, stop and respond:
> "Agent `<name>` is not registered in Cortex. Run `/register-agent <name>` to create the agent note first."

### 4. Extract config from agent note

From the frontmatter and body, extract:
- `name` — from frontmatter
- `slug` — derived from name
- `project` — from frontmatter (the project directory path)
- `projects` — from ## Projects section (list of project slugs)
- `role` — from ## Role section (first line after heading)

### 5. Determine if this is a re-run

Check if `TEAM.md` exists in the current working directory. If it does, this is a sync (re-run).

### 6. Generate TEAM.md

Write `TEAM.md` in the current working directory.

If the agent slug is `chief-of-staff`, generate the **coordinator TEAM.md** (see Section A below).

Otherwise, generate the **worker TEAM.md** (see Section B below).

### 7. Update CLAUDE.md

Read the CLAUDE.md in the current working directory.

If it exists and does NOT already contain the text "join-cortex":
- Append this section at the very end:

```
## Cortex
On session start, run /join-cortex <name> to sync with Cortex. See TEAM.md for full protocol.
```

If CLAUDE.md doesn't exist, create it with just that content.

If CLAUDE.md already contains "join-cortex", do nothing (already configured).

### 8. Suggest heartbeat setup

Respond with heartbeat guidance:

> "To enable the heartbeat (polls for new work every <heartbeat_minutes> minutes), start Claude Code with `/loop` or set up a cron:
> ```
> CronCreate with cron \"*/<heartbeat_minutes> * * * *\"
> ```"

### 9. Confirm

If this was a first join (TEAM.md didn't exist before):

> "Joined Cortex as **<name>**. TEAM.md generated, CLAUDE.md updated. This agent will auto-sync on every session start."

If this was a re-run (TEAM.md already existed):

> "Synced **<name>** with latest Cortex config. TEAM.md regenerated."

---

## Section A: Coordinator TEAM.md (Chief of Staff)

```
# Cortex Protocol

You are **{name}**, the coordinator of Cortex — a team of AI agents.

## Your Identity
- Agent note: agents/{slug}.md in the team directory
- Project: {project}
- Team directory: {team_dir}

## Team Directory Access
- Path: {team_dir}
- Use Claude Code's built-in Read, Write, Edit, Grep, and Glob tools to access files

## Protocol

### Session Start
1. Run /join-cortex {name} to sync with latest config

### Coordinator Duties
2. Check for user messages via Telegram
3. Read all agent notes in agents/ — check ## Session Log for staleness or blockers
4. Read all project notes in projects/ — check ## Work Queue for status updates
5. Flag any issues to the user via Telegram

### Dispatching Work
6. To assign work, write a task entry to the relevant project note's ## Work Queue:
   ### <ISO timestamp>
   **Task:** Short title
   **Scope:** Detailed description of what to do
   **Status:** ready
7. The assigned agent will pick up the task on its next heartbeat

### Registering New Agents
8. Run /register-agent <name> to create a new agent note
9. Tell the user to run /join-cortex <name> in the agent's project directory

### Daily Briefing ({daily_briefing})
10. Scan all agents and projects
11. Summarize: who is active, what's in progress, what's blocked, what's coming up
12. Send via Telegram

### Daily Review ({daily_review})
13. Summarize what got done today, what's blocked, what's planned for tomorrow
14. Send via Telegram

### Heartbeat
15. Poll the team directory every {heartbeat_minutes} minutes for:
    - New user messages via Telegram
    - Agent session logs that haven't updated (staleness)
    - Blocked agents (check for blockers in agent notes)

### Reporting Back
16. After each session, update your agent note (agents/{slug}.md ## Session Log) with:
    - Last session date
    - Current state and any blockers
17. Update SESSION_LOG.md in your project directory

## Communication
- The user communicates with you primarily via Telegram
- You coordinate agents by writing to the team directory — never by communicating directly with agents
- Do NOT execute work on other agents' projects — delegate by writing tasks
- Do NOT approve your own tasks — the user is the authority
```

## Section B: Worker TEAM.md

```
# Cortex Protocol

You are **{name}**, a member of Cortex — a coordinated team of AI agents.

## Your Identity
- Agent note: agents/{slug}.md in the team directory
- Project: {project}
- Team directory: {team_dir}

## Team Directory Access
- Path: {team_dir}
- Use Claude Code's built-in Read, Write, Edit, Grep, and Glob tools to access files

## Protocol

### Session Start
1. Run /join-cortex {name} to sync with latest config

### Checking for Work
2. Read your agent note to find your linked projects
3. For each project, read the project note's ## Work Queue in the team directory
4. Pick up tasks with status "ready":
   - Edit the task status to "in-progress" in the project note
   - Do the work
   - Edit the task status to "done"
   - Append a summary of what was done below the task

### Reporting Back
5. After completing work, update two places:
   a. **Project note** (## Work Queue) — task status and completion summary
   b. **Agent note** (agents/{slug}.md ## Session Log) — last session date, current state, blockers
6. Update SESSION_LOG.md in your project directory (local session continuity)

### Heartbeat
7. Poll the team directory every {heartbeat_minutes} minutes for new work in your project's work queue

## Communication
- The chief of staff monitors your work via the team directory
- If blocked, update your agent note with the blocker — the chief of staff will see it on heartbeat
- Do NOT execute tasks unrelated to your role — flag them instead
```
```

- [ ] **Step 2: Verify skill by reading it back**

Read `skills/join-cortex/SKILL.md` and confirm:
- Config reading works
- Agent note lookup works
- Chief of staff detection (slug check) works
- Both TEAM.md templates are complete and match the spec
- CLAUDE.md update logic is correct
- Heartbeat suggestion is included
- Idempotent (re-run safe)

- [ ] **Step 3: Commit**

```bash
git add skills/join-cortex/SKILL.md
git commit -m "feat: add /join-cortex skill with worker and coordinator protocols"
```

---

### Task 5: /leave-cortex Skill

**Files:**
- Create: `skills/leave-cortex/SKILL.md`

- [ ] **Step 1: Write the skill**

Create `skills/leave-cortex/SKILL.md`:

```markdown
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

If config exists and `<team_dir>/agents/<slug>.md` exists, update the frontmatter `status` field to `inactive` using the Edit tool:

Replace `status: active` with `status: inactive`.

If the config or agent note doesn't exist, skip this step (already cleaned up or never registered).

### 6. Confirm

Respond:

> "**<name>** has left Cortex. TEAM.md removed, CLAUDE.md cleaned, team directory status set to inactive."
```

- [ ] **Step 2: Verify skill by reading it back**

Read `skills/leave-cortex/SKILL.md` and confirm: handles both present/absent TEAM.md, cleans CLAUDE.md section, updates team dir status, handles missing config gracefully.

- [ ] **Step 3: Commit**

```bash
git add skills/leave-cortex/SKILL.md
git commit -m "feat: add /leave-cortex skill for agent offboarding"
```

---

### Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Create `README.md`:

```markdown
# Cortex

Coordinate a team of AI agents through a shared folder of markdown files.

Cortex is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that sets up a **chief of staff** agent to manage **worker agents** across projects. The chief of staff receives instructions via Telegram, dispatches work to agents, and monitors progress — all through plain markdown files with YAML frontmatter.

## How It Works

```
You (Telegram) --> Chief of Staff (Claude Code) --> Team Directory (markdown files) <-- Worker Agents (Claude Code)
```

- **Team directory** — a shared folder of markdown files. Agent notes, project notes, and work queues. Can be an Obsidian vault, a git repo, or any directory.
- **Chief of staff** — a coordinator agent that receives your instructions via Telegram, dispatches tasks to workers, and sends daily briefings.
- **Worker agents** — project-specific agents that poll for work, execute tasks, and report status.
- **No direct agent-to-agent communication.** All coordination happens through the shared markdown files.

## Quick Start

**1. Install the plugin**

```bash
claude plugins add cortex
```

**2. Run setup**

In your terminal, start Claude Code and run:

```
/setup-cortex
```

This creates your team directory, config, and chief of staff agent note. It will ask for:
- Team directory path (default: `~/cortex-team`)
- Telegram bot token and chat ID
- Heartbeat interval and daily schedule times

**3. Join as chief of staff**

In the chief of staff's project directory:

```
/join-cortex Chief of Staff
```

**4. Register a worker agent**

As the chief of staff (or in any session):

```
/register-agent Billing Dev
```

Then in the worker's project directory:

```
/join-cortex Billing Dev
```

## Skills Reference

### /setup-cortex

First-time Cortex setup. Creates the team directory structure, writes `~/.cortex/config.yaml`, and registers the chief of staff.

### /register-agent \<name\>

Register a new agent. Creates an agent note in `<team_dir>/agents/` from the template. Asks for the agent's project path, role, and capabilities.

### /join-cortex \<name\>

Onboard an agent. Reads the agent note from the team directory, generates `TEAM.md` in the current project with the full protocol, and updates `CLAUDE.md`. Idempotent — safe to re-run as a sync.

For the chief of staff, generates an extended protocol with coordinator duties (dispatch, monitoring, daily briefing/review).

### /leave-cortex \<name\>

Offboard an agent. Removes `TEAM.md`, cleans `CLAUDE.md`, and sets agent status to inactive in the team directory.

## Configuration

`~/.cortex/config.yaml`:

```yaml
team_dir: ~/cortex-team
heartbeat_minutes: 15
daily_briefing: "09:00"
daily_review: "18:00"
telegram_chat_id: "your-chat-id"
chief_of_staff_project: "~/Projects/chief-of-staff"
```

## Team Directory Structure

```
~/cortex-team/
  agents/
    chief-of-staff.md
    billing-dev.md
    dashboard-dev.md
  projects/
    billing-service.md
    dashboard.md
  templates/
    agent-template.md
```

## License

MIT
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with quick start and skill reference"
```

---

### Task 7: Integration Test — Run /setup-cortex

**Files:**
- None (manual test)

- [ ] **Step 1: Run /setup-cortex in this project**

Run `/setup-cortex` and provide test values:
- Team dir: `/tmp/cortex-test-team`
- Telegram: skip for now
- Chief of staff project: `/tmp/cortex-test-cos`
- Heartbeat: 15 min
- Briefing: 09:00
- Review: 18:00

- [ ] **Step 2: Verify outputs**

Check that all expected files were created:
- `~/.cortex/config.yaml` with correct values
- `/tmp/cortex-test-team/agents/chief-of-staff.md`
- `/tmp/cortex-test-team/templates/agent-template.md`
- `/tmp/cortex-test-team/agents/`, `projects/`, `templates/` directories exist

- [ ] **Step 3: Run /register-agent**

Run `/register-agent Test Worker` and provide:
- Project: `/tmp/cortex-test-worker`
- Role: "Test worker for integration testing"
- Capabilities: "Testing"

Verify `agents/test-worker.md` was created with correct content.

- [ ] **Step 4: Run /join-cortex**

Run `/join-cortex Test Worker` and verify:
- `TEAM.md` generated in current directory with worker protocol
- `CLAUDE.md` updated with Cortex section
- Heartbeat suggestion shown

- [ ] **Step 5: Run /leave-cortex**

Run `/leave-cortex Test Worker` and verify:
- `TEAM.md` removed
- `CLAUDE.md` cleaned
- `agents/test-worker.md` status set to inactive

- [ ] **Step 6: Clean up test artifacts**

```bash
rm -rf /tmp/cortex-test-team /tmp/cortex-test-cos
```

Remove test `~/.cortex/config.yaml` if it overwrote a real one (back up first).

- [ ] **Step 7: Final commit with any fixes**

If any fixes were needed during testing:

```bash
git add -A
git commit -m "fix: address issues found during integration testing"
```

---

### Task 8: Update Agent Note and Session Log

**Files:**
- Modify: `<team_dir>/projects/<project>.md` — mark task done in work queue
- Modify: `<team_dir>/agents/<agent>.md` — update session log
- Create: `SESSION_LOG.md` — local session continuity

- [ ] **Step 1: Update project note work queue**

Mark the implementation task as done with a summary.

- [ ] **Step 2: Update agent session log**

Update the agent note's Session Log with current date, status, and what was accomplished.

- [ ] **Step 3: Write SESSION_LOG.md**

Create `SESSION_LOG.md` in the cortex project root with a summary of the session.

- [ ] **Step 4: Commit session log**

```bash
git add SESSION_LOG.md
git commit -m "docs: add session log"
```
