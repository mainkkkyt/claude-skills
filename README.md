# claude-skills

Personal Claude Code skills collection, distributed as a plugin marketplace.

## Install via Marketplace

In a Claude Code session, run:

```
/plugin marketplace add mainkkkyt/claude-skills
```

Then install any plugin:

```
/plugin install review
```

## Available Plugins

| Plugin | Description |
|--------|-------------|
| `review` | Code review across 4 dimensions: vulnerabilities, logic, coupling, reusability. Outputs a prioritized optimization list. |

## Structure

```
.claude-plugin/
└── marketplace.json       # Marketplace catalog
plugins/
└── <plugin-name>/
    ├── .claude-plugin/
    │   └── plugin.json    # Plugin manifest
    └── skills/
        └── <skill-name>/
            └── SKILL.md   # Skill definition
```

## Alternatively: Direct Clone

```bash
git clone https://github.com/mainkkkyt/claude-skills ~/.claude/skills
```
