# Cortex

An open protocol and Claude Code plugin for coordinating AI agent teams through plain markdown files.

## Project Structure

```
skills/                  — Claude Code skills (the plugin)
  setup/          — first-time setup
  join/           — agent onboarding (worker + coordinator .cortex.md)
  leave/          — agent offboarding
  register/        — agent registration
docs/
  protocol.md            — standalone protocol spec (runtime-agnostic)
  adapters/              — onboarding examples for Codex, Cursor, Gemini CLI, OpenCode
  internal/              — specs and plans (gitignored)
.claude-plugin/
  plugin.json            — plugin metadata
  marketplace.json       — marketplace definition for installation
```

## Key Concepts

- **Team directory** — a shared folder of markdown files where agents coordinate. Not part of this repo — created by /setup.
- **Skills are prompts, not code** — each SKILL.md is instructions that Claude Code follows. No runtime, no build step.
- **Protocol vs plugin** — `docs/protocol.md` is the universal spec. The skills are one implementation of it.
- **Coordinator vs Worker .cortex.md** — `/join` generates different protocols based on whether the agent slug is `chief-of-staff` or not. Both templates live in `skills/join/SKILL.md`.

## Conventions

- Commit messages: `feat:`, `fix:`, `docs:`, `chore:` prefixes
- Skills are self-contained — each SKILL.md includes all templates inline rather than referencing external files
- Config lives at `~/.cortex/config.yaml`, not in the repo
- .cortex.md and CLAUDE.local.md are gitignored — they're generated per-agent, not part of the plugin

## When Editing Skills

- Skill files are markdown with YAML frontmatter (`name`, `description`, `user_invocable`, `argument-hint`)
- Steps are numbered and sequential — renumber if you insert or remove steps
- Templates (agent notes, .cortex.md) are embedded inline with `{placeholder}` syntax
- After editing, sync the plugin cache: `rsync -av --exclude='.git' --exclude='.cortex.md' ~/Projects/cortex/ ~/.claude/plugins/cache/cortex-local/cortex/0.4.0/`