# elhico Agent Skills

Reusable agent skills for the elhico team. Works with OpenCode, Claude Code, Cursor, GitHub Copilot, and [many more](https://github.com/vercel-labs/skills#supported-agents).

## Install

```bash
npx skills add vw-elhico/agent-skills
```

Install specific skills only:

```bash
npx skills add vw-elhico/agent-skills --skill git-commit --skill github-issue
```

Install globally (available across all projects):

```bash
npx skills add vw-elhico/agent-skills -g
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `agent-md` | Manage agent instruction files (AGENTS.md, CLAUDE.md, COPILOT.md) — add rules, refactor, split into linked files |
| `architecture` | Create ARCHITECTURE.md documentation for codebases based on matklad's pattern |
| `execplan` | Create, implement, discuss, and maintain Execution Plans (ExecPlans) |
| `git-commit` | Analyze changes, ensure code quality, generate conventional commit messages with issue number |
| `github-issue` | Write and refine GitHub issues (features and bugs) |
| `slack-notify` | Send Slack notifications via Bot Token to channels, group DMs, or individual users |
| `testing-principles` | Test type selection, mocking vs faking decisions, managed vs unmanaged dependencies |

## Adding New Skills

1. Create a new directory with the skill name
2. Add a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: What this skill does and when the agent should use it
---

# My Skill

Instructions for the agent...
```

3. Commit and push — the skill is immediately available via `npx skills add vw-elhico/agent-skills`

## Updating Installed Skills

```bash
npx skills update
```
