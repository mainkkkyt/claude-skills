# Claude Code Skills

Personal Claude Code skills collection.

## Structure

Each skill lives in its own directory:

```
<skill-name>/
└── SKILL.md        # Skill definition (frontmatter + instructions)
```

## Usage

Skills in this repo are loaded via Claude Code's skill discovery.
Each skill can be invoked with `/<skill-name>` in the Claude Code CLI.

## Skill Frontmatter Reference

```yaml
---
name: skill-name
description: What this skill does and when to use it
argument-hint: "[optional-arg]"
user-invocable: true
allowed-tools: Read, Grep, Bash
---
```
