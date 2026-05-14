---
name: mempalace-kill
description: Full uninstall of mempalace hook integration. Surgically removes only mempalace-related entries from Claude `~/.claude/settings.json` hooks block (preserves user's other hooks), deletes `~/.codex/hooks.json` + wrapper, strips mempalace lines from `~/.codex/config.toml`, then `rm -rf ~/.mempalace/`. mempalace CLI binary stays installed. Use when the user wants to "uninstall mempalace", "remove mempalace hooks completely", "kill mempalace", or roll back what `/mempalace-init` did.
user-invocable: true
allowed-tools: Bash, AskUserQuestion
---

## 任务

把 `/mempalace-init` 装的所有东西全部撤掉：移除 Claude 和 Codex 的 hook 配置、删除 Codex wrapper、清空 `~/.mempalace/` 整个目录（触发 kill-switch，hook 即便残留也彻底静默）。

**作用域**：仅卸载 hook 集成和 palace 数据。不动 `mempalace` CLI 本体（`uv tool uninstall mempalace` 是用户单独决定的事）。不动 Codex `config.toml` 里的 `[mcp_servers.mempalace]`——那是 MCP server 配置，用户可能想再启用。

## 第一步：盘点要删什么

```bash
echo "=== Claude hooks ==="
test -f ~/.claude/settings.json && python3 -c "
import json
d = json.load(open('$HOME/.claude/settings.json'))
for evt, entries in (d.get('hooks') or {}).items():
    for e in entries:
        for h in e.get('hooks', []):
            if 'mempalace' in h.get('command', ''):
                print(f'  {evt}: {h[\"command\"]}')
"
echo
echo "=== Codex wrapper ==="
ls -la ~/.codex/hooks/mempalace_wrapper.py 2>/dev/null
echo
echo "=== Codex hooks.json ==="
ls -la ~/.codex/hooks.json 2>/dev/null
echo
echo "=== Palace data ==="
test -d ~/.mempalace && mempalace status 2>&1 | head -15
```

把输出原样给用户看，让他清楚要损失什么。

## 第二步：双重确认

用 AskUserQuestion 强制确认。这是不可逆操作：

- question: "确认完全卸载 mempalace hook 集成？所有 wing 数据（X drawer）和上面列出的配置都会删除。"
- options:
  - 标签 "确认卸载" / 描述 "执行完整卸载"
  - 标签 "取消" / 描述 "不动任何文件"

选取消就立即终止，**不**输出任何"已取消"的废话。

## 第三步：移除 Claude hooks（外科手术式，保留用户其他 hook）

```bash
python3 <<'PY'
import json
from pathlib import Path

path = Path.home() / ".claude" / "settings.json"
if not path.exists():
    print("no claude settings.json"); raise SystemExit
data = json.loads(path.read_text())

hooks = data.get("hooks") or {}
removed = []
for event, entries in list(hooks.items()):
    for entry in entries:
        before = len(entry.get("hooks", []))
        entry["hooks"] = [h for h in entry.get("hooks", [])
                          if "mempalace" not in h.get("command", "")]
        if len(entry["hooks"]) < before:
            removed.append(event)
    hooks[event] = [e for e in entries if e.get("hooks")]
    if not hooks[event]:
        del hooks[event]
if not hooks:
    data.pop("hooks", None)
else:
    data["hooks"] = hooks

# 顺手清理可能残留的 env.MEMPAL_DIR
if "MEMPAL_DIR" in (data.get("env") or {}):
    data["env"].pop("MEMPAL_DIR")
    if not data["env"]:
        data.pop("env", None)
    removed.append("env.MEMPAL_DIR")

path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
print(f"claude settings.json: removed {removed or 'nothing'}")
PY
```

**绝对不要**直接 `rm settings.json` 或用 sed 重写——用户的其他配置（permissions、enabledPlugins 等）必须保留。

## 第四步：移除 Codex hooks

Wrapper 和 hooks.json 是我们独占的，直接删：

```bash
rm -f ~/.codex/hooks.json ~/.codex/hooks/mempalace_wrapper.py
rmdir ~/.codex/hooks 2>/dev/null   # 若空则删
```

清理 `~/.codex/config.toml`——**只**删 mempalace-init 加的两个东西，其他配置（model、projects、mcp_servers 等）不要碰：

```bash
python3 <<'PY'
from pathlib import Path
import re

path = Path.home() / ".codex" / "config.toml"
if not path.exists():
    print("no codex config.toml"); raise SystemExit
content = path.read_text()

# 删 hooks = "..." 行
content2 = re.sub(r'^hooks\s*=\s*"[^"]*hooks\.json"\s*\n', '', content, flags=re.M)

# 删 [features] 整段（如果含 codex_hooks 或 hooks）
# 用保守做法：只删 codex_hooks = true 和 hooks = true 两行
# 如果删完后 [features] 块为空（紧跟下一个 [section] 或 EOF），再删 [features] 头
content2 = re.sub(r'^codex_hooks\s*=\s*true\s*\n', '', content2, flags=re.M)
content2 = re.sub(r'^hooks\s*=\s*true\s*\n', '', content2, flags=re.M)
content2 = re.sub(r'\[features\]\s*\n(?=\s*(\[|$))', '', content2)

path.write_text(content2)
print(f"codex config.toml: {len(content) - len(content2)} bytes removed")
PY
```

若用户的 `[features]` 块还有其他 entry，正则只会删 `codex_hooks` / `hooks` 两行，[features] 头保留，不影响其他配置。

## 第五步：删 palace 数据 + 触发 kill-switch

```bash
# 先杀残留 mine 进程，避免文件锁冲突
pgrep -f "mempalace.*mine" | xargs -r kill 2>/dev/null
sleep 1

# 关键：删整个目录，触发 mempalace 的 _palace_root_exists() kill-switch
# 这样即使有 hook 残留在某个还没重启的 session 里，也会静默 no-op
rm -rf ~/.mempalace
```

不要保留 `~/.mempalace/`——这一步**就是**要触发 kill-switch。

## 第六步：验证

```bash
echo "=== verification ==="
test -d ~/.mempalace && echo "FAIL: ~/.mempalace still exists" || echo "OK: palace removed"
test -f ~/.codex/hooks.json && echo "FAIL: codex hooks.json still exists" || echo "OK: codex hooks removed"
test -f ~/.codex/hooks/mempalace_wrapper.py && echo "FAIL: wrapper still exists" || echo "OK: wrapper removed"
grep -c "mempalace" ~/.claude/settings.json 2>/dev/null || echo "OK: no mempalace in claude settings"
```

期望所有结果都是 OK。

## 第七步：报告

精简告诉用户：

- 删了哪几样（用 verification 输出）
- mempalace CLI 本体仍在；若想彻底删：`uv tool uninstall mempalace`
- 提示：当前已运行的 Claude / Codex session 中，hook 配置已固化在 env 里——重启前可能还会触发，但因为 `~/.mempalace/` 没了，全部静默 no-op，不会出错也不会写任何东西。新 session 启动后彻底干净。

不要写"卸载完成 ✅"这类废话。
