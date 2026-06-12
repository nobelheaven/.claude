---
name: git-worktree-list
description: 列出当前 Git 仓库所有 Worktree 的详细信息，包括路径、分支、最近提交时间和提交摘要。当用户说"查看 worktree"、"列出 worktree"、"显示工作树信息"或需要了解当前仓库有哪些 worktree 时触发。
category: Workflow
tags: [git, worktree, list, info]
---

# Git Worktree 信息查看

## 适用场景

- 想了解当前仓库有哪些 worktree 及其状态
- 需要查看各 worktree 对应的分支和最近提交信息
- 在清理或管理 worktree 之前，先做全局概览

---

## 执行流程

### 【前置检查】确认当前目录是 Git 仓库

执行 `git rev-parse --is-inside-work-tree` 检查：

- ✅ 是 Git 仓库 → 直接进入 Step 1
- ❌ 不是 Git 仓库 → 提示用户"当前目录不是 Git 仓库，无法查看 worktree 信息"，**终止流程**

---

### Step 1：获取所有 Worktree 基础信息

执行以下命令获取结构化数据：

```bash
git worktree list --porcelain
```

解析 `--porcelain` 输出，每个 worktree 块包含：
- `worktree <path>` — 路径
- `HEAD <hash>` — 最新 commit hash（完整）
- `branch <ref>` — 分支引用（如 `refs/heads/main`），裸 checkout 时为 `detached`
- `bare` — 若存在此字段，表示这是裸仓库（通常跳过）

将所有 worktree 收集为列表，记为 **`<worktrees>`**。

---

### Step 2：获取每个 Worktree 的详细 Commit 信息

对 **`<worktrees>`** 中的每一个，使用其 `HEAD` hash 执行：

```bash
git log -1 --format="%H|%s|%ai|%an" <hash>
```

字段说明：
- `%H` — 完整 commit hash
- `%s` — commit 标题（subject）
- `%ai` — 提交时间（ISO 8601 格式）
- `%an` — 提交作者

记录每个 worktree 的 **最近提交时间** 和 **提交标题**。

---

### Step 3：整理信息并输出摘要

将收集到的信息整理为如下格式输出，每个 worktree 一个卡片，按**最近提交时间**从新到旧排序：

```
📁 Git Worktree 信息总览
═══════════════════════════════════════════════════════

[1] 主仓库（Main）
  路径：      D:/Works/project
  分支：      main
  最近提交：  2026-03-27 14:30:00 +0800
  提交作者：  张三
  提交摘要：  fix: 修复登录页面样式问题
  Commit：    abc1234

───────────────────────────────────────────────────────

[2] feature/my-feature
  路径：      D:/Works/project-feat
  分支：      feature/my-feature
  最近提交：  2026-03-26 18:22:11 +0800
  提交作者：  李四
  提交摘要：  feat: 新增用户个人中心页面
  Commit：    def5678

───────────────────────────────────────────────────────

[3] fix/some-bug
  路径：      D:/Works/project-fix
  分支：      fix/some-bug
  最近提交：  2026-03-24 11:05:44 +0800
  提交作者：  王五
  提交摘要：  fix: 修复文件上传失败的问题
  Commit：    ghi9012

═══════════════════════════════════════════════════════
共 3 个 Worktree
```

---

### Step 4：AI 辅助摘要（可选）

在输出卡片之后，根据每个 worktree 的最近提交信息，用一句话总结各 worktree 当前的工作状态，帮助用户快速判断哪些 worktree 仍在活跃使用、哪些可能已经闲置：

```
💡 AI 小结：
- [1] 主仓库最近在修复样式问题，活跃中
- [2] feature/my-feature 正在开发用户个人中心，最近有提交（昨天）
- [3] fix/some-bug 最近提交已有 3 天，可能已完成或闲置
```

---

## 注意事项

- `--porcelain` 输出中 `branch` 字段若为 `detached`，表示 HEAD 处于游离状态（detached HEAD）
- 如果某 worktree 路径不存在（目录已被手动删除），`git log` 命令可能报错，此时该字段显示"路径已失效，建议执行清理"
