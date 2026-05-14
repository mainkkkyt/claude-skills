---
name: mempalace-clean
description: Wipe all mempalace wings (drawers, diary, codex staging, hook state) while keeping hooks functional — palace lazy-rebuilds on next session. Preserves ~/.mempalace/ directory and config.json so hooks don't hit the kill-switch. Use when the user wants to "clear all wings", "reset mempalace history", or "start fresh without disabling hooks".
user-invocable: true
allowed-tools: Bash, AskUserQuestion
---

## 任务

清空 mempalace 所有 wing 内容（drawer、diary、staging、hook state），但**保留** `~/.mempalace/` 目录和 `config.json`，让 hook 继续工作、palace 下次写入时 lazy 重建。

**关键约束**：不要 `rm -rf ~/.mempalace`——那个会触发 mempalace 的 kill-switch，hook 永久 no-op。

## 第一步：检查 palace 存在

```bash
test -d ~/.mempalace || { echo "no palace to clean"; }
```

若不存在，告诉用户 mempalace 还没初始化（需要先跑 `/mempalace-init`），**终止**。

## 第二步：展示当前状态

```bash
mempalace status 2>&1 | head -30
```

把输出原样展示给用户——包含 wing 名、room、drawer 数，让用户看清楚要清掉什么。

同时检查是否有正在跑的 mine 进程：

```bash
pgrep -fl "mempalace.*mine" 2>&1
```

若有，告知用户后续清理会让这些进程的写入丢失，但 hook 会重建。

## 第三步：确认

用 AskUserQuestion 双重确认。**必须**让用户主动确认，不能默认执行——这是不可逆操作。

提问示例：
- question: "确认清空 mempalace 全部 wing？上面列出的 N 个 drawer 都会删除。"
- options:
  - 标签 "确认清空" / 描述 "删除所有 wing 内容，保留 hook 配置"
  - 标签 "取消" / 描述 "不动任何文件"

若选取消，**立即终止**，不输出任何"已取消但…"之类的废话。

## 第四步：清理

先杀掉所有 mempalace mine / MCP 子进程，避免文件锁冲突：

```bash
pgrep -f "mempalace.*mine" | xargs -r kill 2>/dev/null
sleep 1
```

不要碰 `mempalace-mcp` 进程（那是各 AI 客户端连着的 MCP server，杀了会影响别的 session 检索能力——但**它们会自动重连**到新的空 palace，没数据风险）。如果要稳一点，留着不动。

清理内容（**只**删这些）：

```bash
rm -rf ~/.mempalace/palace
rm -rf ~/.mempalace/codex-staging
rm -rf ~/.mempalace/hook_state
rm -rf ~/.mempalace/wal
```

不要碰：
- `~/.mempalace/` 本体（kill-switch）
- `~/.mempalace/config.json`（palace 配置）
- `~/.mempalace/bootstrap-empty/`（init 痕迹，没东西但留着无害）

## 第五步：验证

```bash
mempalace status 2>&1 | head -10
```

应该报告 0 drawer（或 "No palace found"——palace 子目录被删，下次写入会 lazy 重建）。把输出原样展示给用户。

## 第六步：报告

一句话：清掉了什么，hooks 还在工作，下个 session 会自动重建 palace 写入新内容。
