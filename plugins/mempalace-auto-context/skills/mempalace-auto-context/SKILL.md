---
name: mempalace-auto-context
description: Use before substantive work that may depend on project memory, prior decisions, user preferences, historical bugs, previous sessions, or durable context stored in MemPalace.
user-invocable: true
allowed-tools: mcp__mempalace__*
---

# MemPalace Auto Context

## When To Use

Use before substantive work when the task touches any of the following:

- Project name, subproject, or module references
- Historical bugs, incidents, or known issues
- Documentation titles, runbooks, or commands
- Prior agent sessions or recent conversations
- Build / deploy workflows (Android, iOS, Flutter, web, etc.)
- Service startup, configuration, or environment setup
- Agents, skills, or local automations
- Any request that implies "what did we decide / do before?"

Skip only for clearly self-contained one-line tasks that cannot benefit from memory.

## Wing Scope (Hard Rule)

**Default scope is the current project wing only.** Compute it from the session's working directory: `wing_<basename(cwd) lowercased, spaces and dashes → underscores>`. Example: cwd `/Users/kangkai/termix` → `wing_termix`; cwd `/private/tmp` → `wing_tmp`.

Apply to **every** mempalace tool call:

- `mempalace_search` — always pass `wing=<current-project-wing>`
- `mempalace_diary_write` — always pass `wing=<current-project-wing>`
- `mempalace_add_drawer` — always pass `wing=<current-project-wing>`
- `mempalace_check_duplicate` — also scope to the same wing if the API supports it

**Only** drop the wing filter (or pass a different wing) when the user explicitly asks for cross-project lookup. Triggers include but are not limited to: "查所有项目"、"across all wings"、"看 X 项目里的…"、"search globally"、"在 wing_xxx 里找"。If the request is ambiguous, default to current wing and tell the user one line: "已限定在 `wing_<name>` 内搜索，若需跨项目请明说"。

Do not "open the scope just in case" — broader hits cost noise and leak unrelated project decisions into the current task.

## Startup Flow

1. Call `mempalace_status` to confirm the memory service is available and identify the active wing/room layout.
2. Search with `mempalace_search` before relying on remembered facts. Use concise keywords: project name, subproject, module, bug symptom, doc title, command, or workflow. **Always pass `wing=<current-project-wing>`** (see Wing Scope rule).
3. Combine multiple targeted searches over one broad query — narrow keywords return higher-signal results.
4. Verify drift-prone facts against live files, commands, simulators, services, or browser state. Memory reflects a past snapshot; current code is the source of truth.

## Write Policy

At the end of every non-trivial task, call `mempalace_diary_write`:

- `agent_name`: the agent identifier for this project (e.g. `codex-<project>`). If unknown, ask the user or infer from prior diary entries.
- `topic`: short slug for the area (`docs`, `build`, `ui`, `api`, or a project-specific module name).
- `entry`: compact AAAK-style summary with **Conclusion**, **Files Changed**, **Verification**, and **Next Step**.

For durable facts that should outlive a single task, first call `mempalace_check_duplicate`, then `mempalace_add_drawer` if not a duplicate:

- `wing`: `wing_<basename(cwd)>` per the Wing Scope rule. Do not ask the user; do not infer from prior drawers — the cwd is authoritative.
- `room`: choose based on content
  - `docs` — rules, runbooks, conventions
  - `general` — workflows, processes, lifecycle facts
  - `components` — UI/component conclusions, design decisions
  - `src` — source structure, commands, build/run details
- `added_by`: the agent identifier (matches `agent_name` above).
- `source_file`: strongest local evidence path that supports the fact.

Do not dump routine session summaries into legacy markdown logs. Use MemPalace diary/drawer by default unless the user explicitly asks for a local markdown file, or the MemPalace service is unavailable.

## What NOT To Write

Never write any of the following to memory:

- Secrets, tokens, cookies, verification codes, passwords
- Full raw logs or transient build noise
- Step-by-step narration of low-value mechanical actions
- Duplicates of facts already present (always check first)
- Information already documented in CLAUDE.md or project README

## If Tools Are Not Visible

If `mempalace_*` tools are not exposed in this session, surface tool discovery keywords to the user so they can verify the MCP server is registered:

> `mempalace search diary kg query status add drawer duplicate`

If the service is genuinely unavailable, fall back to reading the project's local docs (CLAUDE.md, README, `obs/` or `docs/` directories) and note the degraded mode to the user before proceeding.
