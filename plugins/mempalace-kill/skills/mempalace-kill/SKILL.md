---
name: mempalace-kill
description: Remove all MemPalace hooks from Claude Code and Codex CLI, then delete all local MemPalace contents. The mempalace CLI binary itself is not uninstalled.
user-invocable: true
allowed-tools: Bash, AskUserQuestion
---

# MemPalace Kill

## Goal

Fully remove the local MemPalace integration from Claude Code and Codex CLI, then delete all MemPalace data.

Remove:

- MemPalace commands from `~/.claude/settings.json`
- `~/.codex/hooks.json`
- MemPalace hook state blocks and `hooks = true` from `~/.codex/config.toml` when they were only used for this integration
- `~/.mempalace/` entirely

Keep:

- The `mempalace` CLI installation itself
- Non-MemPalace Claude/Codex settings
- Codex MCP server configuration unless the user explicitly asks to remove it

## Steps

1. Show what will be removed:

```bash
echo "=== Claude MemPalace hooks ==="
python3 - <<'PY'
import json
from pathlib import Path
p = Path.home() / ".claude" / "settings.json"
if p.exists():
    data = json.loads(p.read_text())
    for event, entries in (data.get("hooks") or {}).items():
        for entry in entries:
            for h in entry.get("hooks", []):
                if "mempalace" in h.get("command", ""):
                    print(f"{event}: {h['command']}")
PY

echo "=== Codex files ==="
ls -la ~/.codex/hooks.json ~/.mempalace/hooks/project_wing_wrapper.py 2>/dev/null || true

echo "=== MemPalace data ==="
mempalace status 2>&1 | head -60 || true
```

2. Ask for explicit confirmation with `AskUserQuestion`. This is destructive. If cancelled, stop immediately.

3. Remove Claude Code hooks surgically:

```bash
python3 - <<'PY'
import json
from pathlib import Path
p = Path.home() / ".claude" / "settings.json"
if not p.exists():
    print("no claude settings.json")
    raise SystemExit

data = json.loads(p.read_text())
hooks = data.get("hooks") or {}
removed = []
for event, entries in list(hooks.items()):
    for entry in entries:
        before = len(entry.get("hooks", []))
        entry["hooks"] = [h for h in entry.get("hooks", []) if "mempalace" not in h.get("command", "")]
        if len(entry["hooks"]) != before:
            removed.append(event)
    hooks[event] = [e for e in entries if e.get("hooks")]
    if not hooks[event]:
        del hooks[event]
if hooks:
    data["hooks"] = hooks
else:
    data.pop("hooks", None)
p.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
print("removed claude mempalace hooks:", sorted(set(removed)) or "none")
PY
```

4. Remove Codex hooks and MemPalace hook config:

```bash
rm -f ~/.codex/hooks.json

python3 - <<'PY'
from pathlib import Path
import re
p = Path.home() / ".codex" / "config.toml"
if not p.exists():
    print("no codex config.toml")
    raise SystemExit
text = p.read_text()
# Remove trust-state blocks for ~/.codex/hooks.json.
text = re.sub(r'\n?\[hooks\.state\."[^"]*\.codex/hooks\.json:[^"\n]+"\]\n(?:[^\[]|\n(?!\[))*', '\n', text)
# Remove the simple feature flag used to enable hooks. Leave other settings intact.
text = re.sub(r'(?m)^hooks\s*=\s*true\s*\n?', '', text)
text = re.sub(r'\n{3,}', '\n\n', text).strip() + '\n'
p.write_text(text)
print("cleaned codex hook config")
PY
```

5. Delete MemPalace data and wrapper:

```bash
pgrep -f "mempalace.*mine" | xargs -r kill 2>/dev/null || true
sleep 1
rm -rf ~/.mempalace
```

6. Verify:

```bash
echo "=== verification ==="
test -d ~/.mempalace && echo "FAIL: ~/.mempalace exists" || echo "OK: ~/.mempalace removed"
test -f ~/.codex/hooks.json && echo "FAIL: codex hooks.json exists" || echo "OK: codex hooks removed"
python3 - <<'PY'
import json
from pathlib import Path
p = Path.home() / ".claude" / "settings.json"
found = False
if p.exists():
    data = json.loads(p.read_text())
    for entries in (data.get("hooks") or {}).values():
        for entry in entries:
            for h in entry.get("hooks", []):
                found = found or "mempalace" in h.get("command", "")
print("FAIL: claude mempalace hook remains" if found else "OK: claude hooks clean")
PY
```

Report the verification results and mention that already-running Claude/Codex sessions should be restarted.
