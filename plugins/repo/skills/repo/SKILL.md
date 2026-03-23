---
name: repo
description: Open the current git repository's CI/Actions page in the browser. Reads the remote URL from git config, detects GitHub or GitLab, and opens the corresponding Actions or Pipelines page.
user-invocable: true
allowed-tools: Bash
---

## 任务

获取当前 git 仓库的 remote URL，解析出平台和仓库路径，然后在浏览器中打开对应的 CI 页面。

## 第一步：获取 remote URL

按以下优先级查找 remote：

```bash
git remote -v
```

优先使用名为 `origin` 的 remote，其次是名为 `github` 的 remote。取 fetch URL（第一列）。

如果没有任何 remote，告知用户当前仓库没有配置 remote，终止。

## 第二步：解析 URL

支持以下格式：

**HTTPS 格式：**
- `https://github.com/owner/repo.git` → host = `github.com`
- `https://gitlab.com/owner/repo.git` → host = `gitlab.com`

**SSH 格式：**
- `git@github.com:owner/repo.git` → host = `github.com`
- `git@gitlab.com:owner/repo.git` → host = `gitlab.com`

解析出 `host`、`owner`、`repo`（去掉末尾的 `.git`）。

## 第三步：构造并打开链接

根据 host 判断平台：

- 若 host 包含 `github` → 打开 GitHub Actions：
  ```
  https://github.com/<owner>/<repo>/actions
  ```

- 否则（默认按 GitLab 处理）→ 打开 GitLab Pipelines：
  ```
  https://<host>/<owner>/<repo>/-/pipelines
  ```

执行打开命令：

```bash
open "<URL>"
```

输出已打开的链接，告知用户。
