---
name: mempalace-auto-context
description: Use before substantive work to retrieve project memory from MemPalace, scoped to the same deterministic cwd-derived wing used by the Claude Code and Codex hooks installed by mempalace-init.
user-invocable: true
allowed-tools: mcp__mempalace__*
---

# MemPalace Auto Context

## When To Use

Use before substantive work that may depend on project memory:

- Prior decisions, incidents, bugs, deploy notes, or runbooks
- Project/module-specific conventions
- Earlier Claude Code or Codex conversations
- Durable user preferences or workflow facts

Skip only for clearly self-contained one-line tasks.

## Canonical Wing Rule

Use exactly the same wing rule as `mempalace-init` hooks:

```text
cwd_abs = resolved absolute cwd path
folder = basename(cwd_abs)
slug = lowercase(folder), replace non [a-z0-9] runs with _, trim _, fallback project
hash = sha1(cwd_abs).hexdigest()[:10]
wing = wing_{slug}_{hash}
```

This ensures Claude Code, Codex CLI, and manual MemPalace lookups all hit the same wing for the same folder, while duplicate folder names at different absolute paths stay separate.

If the exact cwd is unavailable, derive it from the active session context. If a canonical wing cannot be computed, ask for the project path instead of guessing.

## Required Scope

For normal project work, every MemPalace call must be scoped to the canonical current-project wing:

- `mempalace_search`: pass `wing=<canonical-wing>`
- `mempalace_diary_write`: pass `wing=<canonical-wing>`
- `mempalace_add_drawer`: pass `wing=<canonical-wing>`
- duplicate checks: scope to the same wing when the API supports it

Only search globally or in another wing when the user explicitly asks, e.g. "查所有项目", "search globally", or "在 wing_xxx 里找".

## Startup Flow

1. Call `mempalace_status` to confirm the service is available.
2. Compute the canonical wing from the current cwd.
3. Search targeted keywords in that wing before relying on memory.
4. Verify drift-prone facts against the live filesystem, commands, services, or browser state.

## Write Policy

At the end of non-trivial work, write a compact diary entry to the canonical wing:

- `agent_name`: `claude-<slug>` or `codex-<slug>` when known
- `topic`: short area slug such as `docs`, `build`, `ops`, `ui`, `api`
- `entry`: concise AAAK-style summary with conclusion, changed files, verification, and next step

For durable facts, check for duplicates first, then add a drawer to the canonical wing.

## Do Not Store

Never write secrets, tokens, cookies, verification codes, passwords, full raw logs, or low-value mechanical narration.

## If Tools Are Missing

Tell the user to verify the MCP server exposes:

```text
mempalace search diary kg query status add drawer duplicate
```

Then fall back to local project docs until MemPalace is available.
