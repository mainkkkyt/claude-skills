---
name: commit
description: Smart git commit. Checks if all modified files are staged, handles partial staging with interactive confirmation, then writes and executes a conventional commit. Use when the user wants to commit changes.
user-invocable: true
allowed-tools: Bash, AskUserQuestion
---

## 任务

检查当前 git 暂存状态，根据情况决定是否需要交互确认，然后生成规范的 commit message 并执行提交。

## 第一步：分析暂存状态

运行以下命令获取当前状态：

```
git status --porcelain
```

解析输出，区分以下情况：

- **行首两个字符**：第一个字符表示暂存区状态，第二个字符表示工作区状态
- 已暂存的变更：第一个字符为 `M`、`A`、`D`、`R`、`C` 且第二个字符为空格
- 未暂存的变更：第二个字符为 `M`、`D` 等非空格字符
- 未跟踪文件：`??` 开头

判断逻辑：
- 若**存在未暂存变更或未跟踪文件**，且同时**存在已暂存变更** → 部分暂存，进入第二步
- 若**只有已暂存变更，无任何未暂存内容** → 完全暂存，跳到第三步
- 若**没有任何已暂存变更** → 告知用户没有内容可提交，终止

## 第二步：部分暂存时的交互确认（仅在部分暂存时执行）

使用 AskUserQuestion 工具向用户展示当前状态并询问：

- 展示已暂存的文件列表和未暂存的文件列表
- 问题：当前存在未暂存的变更，如何处理？
- 选项：
  - **全部 add 后提交**：执行 `git add` 将所有未暂存变更加入暂存区，再提交
  - **仅提交已暂存内容**：忽略未暂存部分，只提交当前暂存区内容
  - **中断**：停止操作，让用户自行决定如何处理

若用户选择**全部 add 后提交**，执行 `git add -u`（只 add 已跟踪文件的变更，不包含未跟踪文件），再进入第三步。
若用户选择**中断**，停止所有操作，提示用户可以手动 `git add` 后重新执行。

## 第三步：分析变更内容，生成 commit message

运行以下命令了解本次提交的内容：

```bash
git diff --cached --stat
git diff --cached
git log --oneline -5
```

根据变更内容，生成符合 **Conventional Commits** 规范的 commit message：

### Commit Message 格式

```
<type>(<scope>): <subject>

[optional body]
```

**type 选择规则：**

| type | 使用场景 |
|------|----------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `refactor` | 重构（不影响功能） |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `docs` | 文档变更 |
| `style` | 代码格式（不影响逻辑） |
| `chore` | 构建、依赖、配置等杂项 |
| `ci` | CI/CD 相关 |

**规则：**
- subject 使用动词原形开头，首字母小写，不加句号
- subject 控制在 72 字符以内
- scope 根据实际改动范围填写（可省略）
- 若变更跨多个模块或较复杂，在 body 中补充说明
- 参考 `git log` 中已有的 commit 风格保持一致

## 第四步：执行提交

展示生成的 commit message 让用户确认，然后执行：

```bash
git commit -m "<生成的 commit message>"
```

提交成功后，输出提交结果（commit hash 和 subject）。

## 注意事项

- 不要使用 `git add .` 或 `git add -A`，只提交用户已经暂存的内容
- 不要在未经用户确认的情况下修改暂存区
- commit message 语言与变更内容或已有 commit 历史保持一致（中文项目用中文，英文项目用英文）
- 禁止在 commit message 中追加 `Co-Authored-By` 或任何 trailer 行
