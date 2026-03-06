# elhico Agent Skills

Reusable agent skills for the elhico team. Works with OpenCode, Claude Code, Cursor, GitHub Copilot, and [many more](https://github.com/vercel-labs/skills#supported-agents).

## Install

```bash
npx skills add vw-elhico/agent-skills --skill slack-notify -g
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `slack-notify` | Send Slack notifications via Bot Token to channels, group DMs, or individual users |

## Adding New Skills

1. Create a new directory at the repo root
2. Add a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: What this skill does and when the agent should use it
metadata:
  internal: true
---

# My Skill

Instructions for the agent...
```

3. Commit and push — the skill is immediately available

## Updating Installed Skills

```bash
npx skills update
```
