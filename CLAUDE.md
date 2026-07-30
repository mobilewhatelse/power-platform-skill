# Power Platform Skill Repo

This repo contains skills compatible with both **Claude Code** and **GitHub Copilot**.
Skills live in `.github/skills/<skill-name>/SKILL.md`.

## When adding or modifying skills

Every `SKILL.md` must include these frontmatter fields so the skill works in both tools:

```yaml
---
name: power-platform-<skill-name>
description: <one sentence — what it covers and when to invoke it>
allowed-tools: [shell]
argument-hint: "<short hint shown in Copilot Chat, e.g. 'which table are you working on?'>"
user-invocable: true
---
```

- `name` and `description` — used by both Claude Code and GitHub Copilot
- `allowed-tools`, `argument-hint`, `user-invocable` — Copilot-specific; Claude Code ignores them

## Structure

Each skill lives in its own subdirectory with a `SKILL.md` entry point and a `references/` folder of focused markdown files. The `SKILL.md` links to the reference files — load only the one relevant to the current task, not all of them upfront.
