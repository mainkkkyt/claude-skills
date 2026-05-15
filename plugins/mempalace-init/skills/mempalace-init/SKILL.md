---
name: mempalace-init
description: Install MemPalace hooks for Claude Code and Codex CLI. After a new session has more than 3 human/assistant exchanges, both tools archive the session transcript into the same deterministic project wing derived from the session cwd absolute path.
user-invocable: true
allowed-tools: Bash
---

# MemPalace Init

## Goal

Install one clean hook integration for Claude Code and Codex CLI:

- Both tools write conversations from the same folder into the same wing.
- The wing is deterministic from the session working directory absolute path.
- Sessions are archived after more than 3 exchanges by setting MemPalace `SAVE_INTERVAL = 3`.
- Submitted prompts and assistant responses are mined as conversation drawers.

## Canonical Wing Rule

Every hook must use the same rule:

```text
cwd_abs = resolved absolute cwd path
folder = basename(cwd_abs)
slug = lowercase(folder), replace non [a-z0-9] runs with _, trim _, fallback project
hash = sha1(cwd_abs).hexdigest()[:10]
wing = wing_{slug}_{hash}
```

Examples:

- `/Users/kangkai/termix` -> `wing_termix_<hash>`
- `/tmp/termix` -> `wing_termix_<different_hash>`
- `/Users/kangkai/我的 项目` -> `wing_project_<hash>`

The hash prevents collisions when different absolute paths share the same folder name. The slug keeps wings readable. This same rule must also be used by `mempalace-auto-context` when searching or writing manually.

## Install

Run this whole block. It is idempotent.

```bash
set -euo pipefail

command -v mempalace >/dev/null || { echo "mempalace not found; install it first, e.g. uv tool install mempalace"; exit 1; }

MP_BIN="$(command -v mempalace)"
MP_PY=""
if command -v uv >/dev/null 2>&1; then
  CANDIDATE="$(uv tool dir 2>/dev/null)/mempalace/bin/python"
  test -x "$CANDIDATE" && MP_PY="$CANDIDATE"
fi
if [ -z "$MP_PY" ]; then
  SHEBANG="$(head -1 "$MP_BIN" | sed 's/^#!//')"
  test -x "$SHEBANG" && MP_PY="$SHEBANG"
fi
test -x "$MP_PY" || { echo "cannot find mempalace python"; exit 1; }

mkdir -p ~/.mempalace/bootstrap-empty ~/.mempalace/hooks
mempalace init ~/.mempalace/bootstrap-empty --yes --no-llm >/dev/null 2>&1 || true

"$MP_PY" - <<'PY'
from pathlib import Path
import inspect, re
import mempalace.hooks_cli as hc
from mempalace.config import MempalaceConfig
from mempalace.backends.chroma import ChromaBackend

# Archive after 3 new human messages instead of the package default.
hooks_cli = Path(inspect.getfile(hc))
text = hooks_cli.read_text()
text2 = re.sub(r"^SAVE_INTERVAL\s*=\s*\d+", "SAVE_INTERVAL = 3", text, count=1, flags=re.M)
if text2 != text:
    hooks_cli.write_text(text2)
print(f"SAVE_INTERVAL={3} in {hooks_cli}")

# Ensure the palace DB exists now; hooks also self-heal if it is deleted later.
cfg = MempalaceConfig()
palace = Path(cfg.palace_path)
collection = cfg._file_config.get("collection_name", "mempalace_drawers")
if not (palace / "chroma.sqlite3").exists():
    ChromaBackend().get_collection(str(palace), collection, create=True)
print(f"palace={palace}")
PY

WRAPPER="$HOME/.mempalace/hooks/project_wing_wrapper.py"
cat > "$WRAPPER" <<PYWRAP
#!$MP_PY
import hashlib, io, json, os, re, shutil, sys, unicodedata
from pathlib import Path
import mempalace.hooks_cli as hc

STAGE_ROOT = Path.home() / ".mempalace" / "session-staging"


def _ensure_palace():
    try:
        from mempalace.config import MempalaceConfig
        from mempalace.backends.chroma import ChromaBackend
        cfg = MempalaceConfig()
        palace = Path(cfg.palace_path)
        collection = cfg._file_config.get("collection_name", "mempalace_drawers")
        if not (palace / "chroma.sqlite3").exists():
            ChromaBackend().get_collection(str(palace), collection, create=True)
            hc._log(f"auto-init palace at {palace}")
    except Exception as e:
        hc._log(f"auto-init skipped: {e}")


def _slug(name):
    base = unicodedata.normalize("NFKD", name).encode("ascii", "ignore").decode()
    slug = re.sub(r"[^A-Za-z0-9]+", "_", base).strip("_").lower()
    if not slug:
        slug = "project"
    if slug[0].isdigit():
        slug = "p_" + slug
    return slug[:48]


def _wing_for_cwd(cwd):
    abs_path = str(Path(cwd).expanduser().resolve(strict=False))
    digest = hashlib.sha1(abs_path.encode("utf-8")).hexdigest()[:10]
    return f"wing_{_slug(Path(abs_path).name)}_{digest}"


def _cwd_from_json(obj):
    if isinstance(obj, dict):
        if isinstance(obj.get("cwd"), str) and obj["cwd"]:
            return obj["cwd"]
        payload = obj.get("payload")
        if isinstance(payload, dict) and isinstance(payload.get("cwd"), str) and payload["cwd"]:
            return payload["cwd"]
    return None


def _cwd_from_transcript(transcript_path):
    path = Path(transcript_path).expanduser()
    if path.is_file():
        try:
            with path.open(encoding="utf-8", errors="replace") as f:
                for _, line in zip(range(300), f):
                    try:
                        cwd = _cwd_from_json(json.loads(line))
                    except Exception:
                        continue
                    if cwd:
                        return cwd
        except OSError:
            pass

    # Fallback for Claude Code folders such as ~/.claude/projects/-Users-name-project.
    parts = str(path).replace("\\", "/").split("/.claude/projects/-", 1)
    if len(parts) == 2:
        encoded = parts[1].split("/", 1)[0]
        if encoded:
            return "/" + encoded.replace("-", "/")
    return None


def _stage_transcript(transcript_path, harness, session_id):
    src = Path(transcript_path).expanduser()
    if not src.is_file() or src.stat().st_size < 100:
        return None
    name_hash = hashlib.sha1(str(src.resolve(strict=False)).encode("utf-8")).hexdigest()[:8]
    stage_dir = STAGE_ROOT / harness / session_id
    stage_dir.mkdir(parents=True, exist_ok=True)
    dst = stage_dir / f"{name_hash}-{src.name}"
    if dst.exists() or dst.is_symlink():
        try:
            dst.unlink()
        except OSError:
            pass
    try:
        os.link(str(src.resolve(strict=False)), str(dst))
    except OSError:
        shutil.copy2(str(src), str(dst))
    return stage_dir


def main():
    if len(sys.argv) < 3:
        print("usage: project_wing_wrapper.py <claude-code|codex> <session-start|stop|precompact>", file=sys.stderr)
        sys.exit(2)
    harness, hook_name = sys.argv[1], sys.argv[2]
    raw = sys.stdin.read()
    try:
        data = json.loads(raw) if raw.strip() else {}
    except json.JSONDecodeError:
        data = {}

    _ensure_palace()

    session_id = hc._sanitize_session_id(str(data.get("session_id") or data.get("payload", {}).get("id") or "unknown"))
    transcript_path = str(data.get("transcript_path") or "")
    cwd = _cwd_from_json(data) or _cwd_from_transcript(transcript_path)

    if cwd:
        wing = _wing_for_cwd(cwd)
        hc._wing_from_transcript_path = lambda _tp: wing

        def _ingest(_tp):
            stage_dir = _stage_transcript(transcript_path, harness, session_id)
            if not stage_dir:
                return
            try:
                from mempalace.config import MempalaceConfig
                MempalaceConfig()
                hc._spawn_mine([hc._mempalace_python(), "-m", "mempalace", "mine", str(stage_dir), "--mode", "convos", "--wing", wing])
                hc._log(f"{harness} ingest -> {wing} cwd={cwd}")
            except Exception as e:
                hc._log(f"ingest skipped: {e}")
        hc._ingest_transcript = _ingest

    sys.stdin = io.StringIO(raw)
    hc.run_hook(hook_name, harness)


if __name__ == "__main__":
    main()
PYWRAP
chmod +x "$WRAPPER"

python3 - <<'PY'
import json
from pathlib import Path
wrapper = str(Path.home() / ".mempalace" / "hooks" / "project_wing_wrapper.py")
path = Path.home() / ".claude" / "settings.json"
data = json.loads(path.read_text()) if path.exists() else {}
hooks = data.setdefault("hooks", {})

def upsert(event, command, timeout, extra=None):
    spec = {"type": "command", "command": command, "timeout": timeout}
    if extra:
        spec.update(extra)
    entries = hooks.setdefault(event, [])
    for entry in entries:
        entry["hooks"] = [h for h in entry.get("hooks", []) if "mempalace" not in h.get("command", "")]
    entries[:] = [e for e in entries if e.get("hooks")]
    entries.append({"hooks": [spec]})

upsert("SessionStart", f"{wrapper} claude-code session-start", 30)
upsert("Stop", f"{wrapper} claude-code stop", 60, {"async": True})
upsert("PreCompact", f"{wrapper} claude-code precompact", 120)
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
print(f"updated {path}")
PY

python3 - <<'PY'
import json, re
from pathlib import Path
wrapper = str(Path.home() / ".mempalace" / "hooks" / "project_wing_wrapper.py")
hooks_path = Path.home() / ".codex" / "hooks.json"
hooks_path.parent.mkdir(parents=True, exist_ok=True)
hooks_path.write_text(json.dumps({"hooks": {
    "SessionStart": [{"hooks": [{"type": "command", "command": f"{wrapper} codex session-start", "timeout": 30}]}],
    "Stop": [{"hooks": [{"type": "command", "command": f"{wrapper} codex stop", "timeout": 60}]}],
    "PreCompact": [{"hooks": [{"type": "command", "command": f"{wrapper} codex precompact", "timeout": 120}]}]
}}, indent=2) + "\n")

cfg = Path.home() / ".codex" / "config.toml"
text = cfg.read_text() if cfg.exists() else ""
lines = text.splitlines()
features_i = next((i for i, line in enumerate(lines) if line.strip() == "[features]"), None)
if features_i is None:
    if lines and lines[-1].strip():
        lines.append("")
    lines.extend(["[features]", "hooks = true"])
else:
    next_i = next((i for i in range(features_i + 1, len(lines)) if lines[i].lstrip().startswith("[") and lines[i].rstrip().endswith("]")), len(lines))
    hook_i = next((i for i in range(features_i + 1, next_i) if re.match(r"\s*hooks\s*=", lines[i])), None)
    if hook_i is None:
        lines.insert(features_i + 1, "hooks = true")
    else:
        lines[hook_i] = "hooks = true"
text = "\n".join(lines).rstrip() + "\n"
cfg.parent.mkdir(parents=True, exist_ok=True)
cfg.write_text(text)
print(f"updated {hooks_path}")
print(f"enabled hooks in {cfg}")
PY

"$WRAPPER" codex session-start <<<'{"session_id":"mempalace-smoke","transcript_path":""}'
"$WRAPPER" claude-code session-start <<<'{"session_id":"mempalace-smoke","transcript_path":""}'

echo "mempalace hooks installed"
```

## Verify

```bash
mempalace status | head -40
rg -n "project_wing_wrapper|hooks = true|mempalace" ~/.claude/settings.json ~/.codex/hooks.json ~/.codex/config.toml ~/.mempalace/hooks/project_wing_wrapper.py
```

New Claude Code and Codex sessions must be restarted after installation so they load the new hook configuration.
