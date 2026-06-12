---
name: git-worktree-clean
description: 清理 Git Worktree，支持自动 prune 清理悬空引用，或交互式选择某个 worktree 手动删除。当用户说"清理 worktree"、"删除 worktree"、"移除工作树"或需要整理仓库 worktree 时触发。
category: Workflow
tags: [git, worktree, clean, prune, remove]
---

# Git Worktree 清理

## 适用场景

- 删除某个已完成开发的 worktree 目录及其 git 引用
- 清理目录已被手动删除但 git 引用仍残留的悬空 worktree
- 批量整理仓库中的 worktree

---

## 执行流程

### 【前置检查】确认当前目录是 Git 仓库

执行 `git rev-parse --is-inside-work-tree` 检查：

- ✅ 是 Git 仓库 → 直接进入 Step 1
- ❌ 不是 Git 仓库 → 提示用户"当前目录不是 Git 仓库，无法执行清理"，**终止流程**

---

### Step 1：询问清理方式

用 `ask_user_question` 询问用户选择清理方式：

> "请选择 Worktree 清理方式："

- 选项 A：**自动清理（prune）** — 自动删除目录已不存在的悬空 worktree 引用（安全，不删除实际目录）
- 选项 B：**手动选择清理** — 列出所有 worktree，由用户选择要删除的 worktree（同时删除目录和 git 引用）

---

## 分支 A：自动清理（prune）

### Step A1：执行 dry-run 预览

先执行 dry-run，预览将被清理的内容，**不实际执行**：

```bash
git worktree prune --dry-run
```

- 如果输出为空 → 提示"没有发现需要清理的悬空 worktree 引用，仓库已是干净状态 ✅"，**终止流程**
- 如果有输出 → 将输出展示给用户，进入 Step A2

---

### Step A2：确认执行清理

用 `ask_user_question` 展示 dry-run 结果并询问：

> "以上悬空引用将被清理，确认执行吗？"

- 选项1：**确认执行**
- 选项2：**取消**

---

### Step A3：执行 prune

用户确认后执行：

```bash
git worktree prune
```

执行后：
- ✅ 成功 → 提示"悬空引用已清理完毕 ✅"
- ❌ 失败 → 输出错误信息，提示用户手动处理

---

## 分支 B：手动选择清理

### Step B1：获取所有 Worktree 信息

执行以下命令获取结构化数据：

```bash
git worktree list --porcelain
```

解析输出，收集每个 worktree 的：
- 路径（`worktree <path>`）
- 分支名（`branch <ref>`，去掉 `refs/heads/` 前缀；若为 `detached` 则标注游离状态）
- HEAD commit hash（`HEAD <hash>`）

对每个 worktree，进一步执行：

```bash
git log -1 --format="%s|%ai" <hash>
```

获取**最近提交标题**和**提交时间**。

---

### Step B2：展示 Worktree 列表并询问选择

将所有 worktree 整理为选项，**排除主 worktree（列表第一个）**（主 worktree 不允许通过此方式删除），用 `ask_user_question` 展示：

> "请选择要删除的 Worktree（主仓库不可删除）："

选项格式如下（每个 worktree 一个选项）：

```
[分支名] 路径 | 最近提交：提交时间 | 提交摘要
```

示例：
```
[feature/my-feature] D:/Works/project-feat | 最近提交：2026-03-26 18:22 | feat: 新增用户个人中心
[fix/some-bug] D:/Works/project-fix | 最近提交：2026-03-24 11:05 | fix: 修复文件上传失败
```

另加一个选项：**取消，不删除任何 worktree**

将用户选择记为 **`<target_worktree>`**（包含路径）。

---

### Step B3：检查目标 Worktree 状态

在执行删除前，检查目标 worktree 是否有未提交的修改：

```bash
git -C <target_path> status --porcelain
```

- 输出为空 → 工作区干净，进入 Step B4
- 有输出 → 说明存在未提交的修改，用 `ask_user_question` 警告用户：

  > "⚠️ 该 Worktree 存在未提交的修改，删除后将无法恢复！确认继续吗？"

  - 选项1：**确认，强制删除**（使用 `--force`）
  - 选项2：**取消**（终止流程）

---

### Step B4：执行删除

根据 Step B3 的检查结果执行：

**工作区干净时：**
```bash
git worktree remove <target_path>
```

**有未提交修改且用户确认强制删除时：**
```bash
git worktree remove --force <target_path>
```

执行后：
- ✅ 成功 → 提示"Worktree 已成功删除 ✅：`<target_path>`"
- ❌ 失败（如目录仍被占用）→ 输出错误信息，提示用户关闭可能占用该目录的程序后重试

---

### Step B5：询问是否继续清理

用 `ask_user_question` 询问：

> "是否继续删除其他 Worktree？"

- 选项1：**是，继续选择**（返回 Step B1 重新获取列表）
- 选项2：**否，完成清理**

---

## 完成摘要

所有清理操作完成后，执行一次 `git worktree list` 展示当前剩余的 worktree 列表，并输出：

```
🧹 Worktree 清理完成！

当前仓库剩余 Worktree：
  [1] main        →  D:/Works/project
  [2] feature/xxx →  D:/Works/project-xxx

如需进一步查看详情，可使用"查看 worktree"命令。
```

---

## 注意事项

- `git worktree remove` 会**同时删除** worktree 目录和 git 引用，请确认后再执行
- `git worktree prune` 只清理**悬空引用**（目录已消失的记录），不删除实际存在的目录
- 主 worktree（`git worktree list` 第一条）不能通过 `git worktree remove` 删除
- 若目录正在被其他程序（如 IDE、终端）占用，`remove` 可能失败，关闭相关程序后重试
- 建议清理前先使用"查看 worktree"skill 了解各 worktree 的状态
