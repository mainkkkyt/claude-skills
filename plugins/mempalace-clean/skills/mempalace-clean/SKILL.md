---
name: mempalace-clean
description: Delete all MemPalace wing data while keeping Claude Code and Codex hooks installed. Use this to reset memory contents without uninstalling the hook integration.
user-invocable: true
allowed-tools: Bash, AskUserQuestion
---

# MemPalace Clean

## Goal

Clear every wing and all transient MemPalace state, but keep hooks installed and reusable.

Delete:

- `~/.mempalace/palace`
- `~/.mempalace/session-staging`
- `~/.mempalace/codex-staging`
- `~/.mempalace/hook_state`
- `~/.mempalace/wal`

Keep:

- `~/.mempalace/`
- `~/.mempalace/config.json`
- `~/.mempalace/hooks/project_wing_wrapper.py`
- Claude Code and Codex hook configuration

Keeping `~/.mempalace/` matters because the hooks treat its absence as a kill switch.

## Steps

1. Show current data:

```bash
mempalace status 2>&1 | head -60 || true
pgrep -fl "mempalace.*mine" 2>/dev/null || true
```

2. Ask for explicit confirmation with `AskUserQuestion`. This is destructive. If cancelled, stop immediately.

3. Clean data only:

```bash
set -euo pipefail
pgrep -f "mempalace.*mine" | xargs -r kill 2>/dev/null || true
sleep 1
mkdir -p ~/.mempalace
rm -rf ~/.mempalace/palace
rm -rf ~/.mempalace/session-staging
rm -rf ~/.mempalace/codex-staging
rm -rf ~/.mempalace/hook_state
rm -rf ~/.mempalace/wal
```

4. Verify:

```bash
mempalace status 2>&1 | head -30 || true
test -d ~/.mempalace && echo "OK: ~/.mempalace kept"
test -x ~/.mempalace/hooks/project_wing_wrapper.py && echo "OK: hook wrapper kept"
```

Report only what was deleted and that hooks remain installed.
