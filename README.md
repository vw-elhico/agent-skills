# elhico Agent Skills

Reusable agent skills for the elhico team. Works with OpenCode, Claude Code, Cursor, GitHub Copilot, and [many more](https://github.com/vercel-labs/skills#supported-agents).

## Install

### Core Skills

Core skills are project-relevant and should be installed per project. They are committed to the
repository and shared with the whole team.

```bash
npx skills add https://github.com/vw-elhico/agent-skills/tree/main/skills/core
```

### Optional Skills

Optional skills are personal tools — install them globally so they are available across all
projects but not committed to any repository.

```bash
# All optional skills (global)
npx skills add https://github.com/vw-elhico/agent-skills/tree/main/skills/optional -g

# Single optional skill (global)
npx skills add https://github.com/vw-elhico/agent-skills/tree/main/skills/optional --skill slack-notify -g
```

## Available Skills

### Core

| Skill | Description |
|-------|-------------|
| `agent-md` | Manage agent instruction files (AGENTS.md, CLAUDE.md, COPILOT.md) — add rules, refactor, split into linked files |
| `architecture` | Create ARCHITECTURE.md documentation for codebases based on matklad's pattern |
| `execplan` | Create, implement, discuss, and maintain Execution Plans (ExecPlans) |
| `git-commit` | Analyze changes, ensure code quality, generate conventional commit messages with issue number |
| `github-issue` | Write and refine GitHub issues (features and bugs) |
| `testing-principles` | Test type selection, mocking vs faking decisions, managed vs unmanaged dependencies |

### Optional

| Skill | Description |
|-------|-------------|
| `slack-notify` | Send Slack notifications via Bot Token to channels, group DMs, or individual users |

## Adding New Skills

1. Create a new directory under `skills/core/` or `skills/optional/`
2. Add a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: What this skill does and when the agent should use it
---

# My Skill

Instructions for the agent...
```

3. Commit and push — the skill is immediately available

## Updating Installed Skills

```bash
npx skills update
```
