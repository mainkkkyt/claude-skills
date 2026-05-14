---
name: mempalace-init
description: Install mempalace hooks for Claude Code and Codex CLI. Each session's transcript is auto-archived to wing_<basename(cwd)> in the local mempalace, enabling cross-tool context retrieval via `mempalace search`. Idempotent. Use when the user wants to "set up mempalace hooks", "install mempalace integration", or asks how to start auto-saving conversations.
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion
---

## 任务

为本机的 Claude Code 和 Codex CLI 安装 mempalace hook，使每个 session 的对话自动入库到 `wing_<basename(cwd)>`。脚本必须幂等——多次跑不出错也不重复污染配置。

## 第一步：前置检查

```bash
which mempalace
```

若不存在，告诉用户先 `uv tool install mempalace` 然后再调用本 skill，**终止**。

解析 mempalace 自带 python（wrapper 脚本的 shebang 要用它）：

```bash
MP_PY="$(uv tool dir)/mempalace/bin/python"
test -x "$MP_PY"
```

若 `uv tool dir` 失败或 `$MP_PY` 不存在，从 `which mempalace` 的 shebang 反推（`head -1 $(which mempalace)` 取 `#!` 后的路径）。

## 第二步：Palace 骨架

```bash
test -d ~/.mempalace || {
  mkdir -p ~/.mempalace/bootstrap-empty
  mempalace init ~/.mempalace/bootstrap-empty --yes --no-llm
}
```

注意：`~/.mempalace` 是 mempalace 的 kill-switch——目录不存在则所有 hook 静默 no-op。已存在则跳过 init。

## 第三步：Claude Code hooks

读 `~/.claude/settings.json`（若不存在则用 `{}` 作起始）。用 Python 安全 merge，**不要**覆盖用户已有的其他 hook 或配置：

```bash
python3 <<'PY'
import json, os
from pathlib import Path

path = Path.home() / ".claude" / "settings.json"
data = json.loads(path.read_text()) if path.exists() else {}

# 强制移除 env.MEMPAL_DIR（会让每个 session 全量 mine 那个项目）
env = data.get("env", {})
if "MEMPAL_DIR" in env:
    del env["MEMPAL_DIR"]
    if not env:
        data.pop("env", None)
    else:
        data["env"] = env

hooks = data.setdefault("hooks", {})

def upsert(event, cmd, timeout, extra=None):
    spec = {"type": "command", "command": cmd, "timeout": timeout}
    if extra: spec.update(extra)
    entries = hooks.setdefault(event, [])
    # 删除任何指向 mempalace 的旧条目
    for entry in entries:
        entry["hooks"] = [h for h in entry.get("hooks", []) if "mempalace" not in h.get("command", "")]
    entries[:] = [e for e in entries if e.get("hooks")]
    entries.append({"hooks": [spec]})

upsert("SessionStart", "mempalace hook run --hook session-start --harness claude-code", 30)
upsert("Stop",         "mempalace hook run --hook stop --harness claude-code",          60, {"async": True})
upsert("PreCompact",   "mempalace hook run --hook precompact --harness claude-code",    120)

path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
print("claude settings.json updated")
PY
```

若用户原本设了 `env.MEMPAL_DIR`，明确告知已删除——这是必要的清理。

## 第四步：Codex wrapper

`~/.codex/hooks/mempalace_wrapper.py` 的 shebang 必须是 `$MP_PY`。用 heredoc 写出，**这是机器特定路径**：

```bash
mkdir -p ~/.codex/hooks
cat > ~/.codex/hooks/mempalace_wrapper.py <<EOF
#!${MP_PY}
import io, json, os, shutil, sys
from pathlib import Path
import mempalace.hooks_cli as hc

STAGING_ROOT = Path.home() / ".mempalace" / "codex-staging"

def _normalize(name):
    return (name.lower().replace(" ", "_").replace("-", "_")) or "session"

def _resolve(transcript_path, session_id):
    if not transcript_path: return None, None
    path = Path(transcript_path).expanduser()
    if not path.is_file(): return None, None
    cwd = None
    try:
        with open(path, encoding="utf-8", errors="replace") as f:
            meta = json.loads(f.readline())
        if meta.get("type") == "session_meta":
            cwd = meta.get("payload", {}).get("cwd")
    except (OSError, json.JSONDecodeError, ValueError):
        pass
    if not cwd: return None, None
    wing = "wing_" + _normalize(Path(cwd).name)
    stage_dir = STAGING_ROOT / session_id
    try:
        stage_dir.mkdir(parents=True, exist_ok=True)
        staged = stage_dir / path.name
        if staged.is_symlink() or staged.exists():
            try: staged.unlink()
            except OSError: pass
        real = path.resolve()
        try: os.link(str(real), str(staged))
        except OSError: shutil.copy2(str(real), str(staged))
    except OSError:
        return wing, None
    return wing, stage_dir

def main():
    raw = sys.stdin.read()
    try: data = json.loads(raw) if raw.strip() else {}
    except json.JSONDecodeError: data = {}
    session_id = hc._sanitize_session_id(str(data.get("session_id", "unknown")))
    wing, stage_dir = _resolve(data.get("transcript_path", ""), session_id)
    if wing:
        hc._wing_from_transcript_path = lambda _: wing
        def _staged_ingest(_):
            if stage_dir is None: return
            try:
                from mempalace.config import MempalaceConfig
                MempalaceConfig()
            except Exception: return
            try:
                hc._spawn_mine([hc._mempalace_python(), "-m", "mempalace", "mine",
                                str(stage_dir), "--mode", "convos", "--wing", wing])
                hc._log(f"Codex ingest -> {wing} (stage={stage_dir.name})")
            except OSError: pass
        hc._ingest_transcript = _staged_ingest
    sys.stdin = io.StringIO(raw)
    hc.run_hook(sys.argv[1] if len(sys.argv) > 1 else "stop", "codex")

if __name__ == "__main__":
    main()
EOF
chmod +x ~/.codex/hooks/mempalace_wrapper.py
```

设计要点（**必须遵守**，不能擅自改）：

- Codex transcript 路径不含项目名，wing 派生必须从 `session_meta.payload.cwd` 读
- Staging dir 按 session_id 隔离，避免 `~/.codex/sessions/YYYY/MM/DD/` 整目录被 mine 时混入其他项目
- 必须用 hardlink (`os.link`)，**不能用 symlink**——`mempalace.convo_miner.scan_convos` 跳过 symlink，会导致 `Files processed: 0`

## 第五步：Codex hooks.json

用 absolute path（不能是 `$HOME`，Codex 不解析变量）：

```bash
WRAPPER="$HOME/.codex/hooks/mempalace_wrapper.py"
python3 <<PY
import json, os
from pathlib import Path

path = Path.home() / ".codex" / "hooks.json"
data = json.loads(path.read_text()) if path.exists() else {}
hooks = data.setdefault("hooks", {})

def upsert(event, cmd, timeout, extra=None):
    spec = {"type": "command", "command": cmd, "timeout": timeout}
    if extra: spec.update(extra)
    entries = hooks.setdefault(event, [])
    for entry in entries:
        entry["hooks"] = [h for h in entry.get("hooks", [])
                          if "mempalace" not in h.get("command", "")
                          and "$WRAPPER" not in h.get("command", "")]
    entries[:] = [e for e in entries if e.get("hooks")]
    entries.append({"hooks": [spec]})

upsert("SessionStart", "mempalace hook run --hook session-start --harness codex", 30)
upsert("Stop",         "$WRAPPER stop",       60, {"async": True})
upsert("PreCompact",   "$WRAPPER precompact", 120)

path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
print("codex hooks.json updated")
PY
```

## 第六步：Codex config.toml feature flag

确保 `~/.codex/config.toml` 含：

```toml
hooks = "/Users/<user>/.codex/hooks.json"

[features]
codex_hooks = true
hooks = true
```

用 grep 检查这两行，不存在才追加。**不要用 sed 重写整个文件**——用户可能有 `model_providers`、`projects` 等其他配置。

```bash
CFG="$HOME/.codex/config.toml"
HOOKS_PATH="$HOME/.codex/hooks.json"
touch "$CFG"
grep -q "^hooks = " "$CFG" || echo "hooks = \"$HOOKS_PATH\"" >> "$CFG"
grep -q "codex_hooks = true" "$CFG" || cat >> "$CFG" <<EOF

[features]
codex_hooks = true
hooks = true
EOF
```

若 `[features]` 已存在但缺 `codex_hooks` / `hooks`，告诉用户手动添加（不要在中间硬塞，会破坏 toml 结构）。

## 第七步：烟测

```bash
echo '{"session_id":"init-smoketest","transcript_path":"","stop_hook_active":false}' | mempalace hook run --hook session-start --harness claude-code
echo '{"session_id":"init-smoketest","transcript_path":"","stop_hook_active":false}' | mempalace hook run --hook session-start --harness codex
echo '{"session_id":"init-smoketest","transcript_path":"","stop_hook_active":false}' | ~/.codex/hooks/mempalace_wrapper.py stop
```

三个都应返回 `{}` exit 0。若失败，告诉用户具体哪一步出错并打印日志：`tail -20 ~/.mempalace/hook_state/hook.log`。

## 第八步：报告

精简告知用户：

- 改动了哪几个文件（`~/.claude/settings.json`、`~/.codex/hooks.json`、`~/.codex/hooks/mempalace_wrapper.py`、可能的 `~/.codex/config.toml`）
- 若移除了 `env.MEMPAL_DIR` 要明说
- 提示需要**重启**当前的 Claude Code / Codex session，新配置才生效（已运行的 session env 已固化）

不要写"安装成功 🎉"之类的废话。
