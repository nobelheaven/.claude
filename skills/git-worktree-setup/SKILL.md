---
name: git-worktree-setup
description: 创建 Git Worktree 并初始化 Python 虚拟环境。当用户说"帮我新建一个 worktree"、"创建 worktree"、"新建工作树"或需要为某个分支创建独立工作目录时触发。
category: Workflow
tags: [git, worktree, venv, python, setup]
---

# Git Worktree 创建 & Python 虚拟环境初始化

## 适用场景

- 需要为某个分支创建独立的工作目录（worktree）
- 希望在新 worktree 中配置好独立的 Python 虚拟环境并安装依赖

---

## 执行流程

### 【前置检查】确认当前目录是 Git 仓库

执行 `git rev-parse --is-inside-work-tree` 检查：

- ✅ 是 Git 仓库 → 直接进入 Step 1
- ❌ 不是 Git 仓库 → **询问用户是否初始化为 Git 仓库**：
  > "当前目录不是 Git 仓库，是否执行 `git init` 初始化？"
  - 用户确认 → 执行 `git init`，然后进入 Step 1
  - 用户拒绝 → 终止流程，提示需要在 Git 仓库中使用本技能

---

### Step 1：询问 Worktree 本地目录名

用 `ask_user_question` 询问：
> "请输入新 worktree 的目录名（仅目录名，不含路径，例如 `dev-feature`）："

将用户回答记为 **`<worktree_dir>`**。

---

### Step 2：询问 Worktree 存放路径

- 自动计算当前工程根目录的**上一级目录**作为默认路径（即 `../`，绝对路径形式展示更友好）
- 用 `ask_user_question` 询问，提供默认选项 + 自定义输入：
  > "请选择 worktree 存放的父目录（新 worktree 将创建在该目录下）："
  - 选项1：`<上一级目录的绝对路径>（当前工程的同级目录）(Recommended)`
  - 选项2：手动输入

将最终路径记为 **`<base_path>`**，完整 worktree 路径为 **`<base_path>/<worktree_dir>`**。

---

### Step 3：询问分支操作方式

用 `ask_user_question` 询问：
> "请选择分支操作方式："

- 选项 A：**基于现有分支新建分支**（会创建新分支并 checkout）(Recommended)
- 选项 B：**直接 checkout 到已有分支**（不创建新分支）

---

### Step 4：选择基础分支

执行 `git branch -a` 列出所有本地和远程分支，整理后用 `ask_user_question` 展示给用户选择：
> "请选择基础分支（worktree 将基于此分支）："

将用户选择记为 **`<base_branch>`**。

> 💡 如果选择的是远程分支（如 `remotes/origin/xxx`），自动去掉 `remotes/origin/` 前缀，并在命令中加 `--track` 参数。

---

### Step 5：询问新分支名（仅限选项 A）

如果 Step 3 选择了**基于现有分支新建分支**，用 `ask_user_question` 询问：
> "请输入新分支名："

将用户回答记为 **`<new_branch>`**。

---

### Step 6：执行 `git worktree add`

根据 Step 3 的选择，执行对应命令：

**选项 A（新建分支）：**
```bash
git worktree add -b <new_branch> <base_path>/<worktree_dir> <base_branch>
```

**选项 B（checkout 已有分支）：**
```bash
git worktree add <base_path>/<worktree_dir> <base_branch>
```

执行后验证：
- ✅ 成功 → 提示 "Worktree 已创建：`<base_path>/<worktree_dir>`"，进入 Step 7
- ❌ 失败 → 输出错误信息，提示用户检查路径或分支是否已被其他 worktree 占用，**终止流程**

---

### Step 7：询问虚拟环境名称

用 `ask_user_question` 询问：
> "请选择虚拟环境目录名："

- 选项1：`.venv` (Recommended)
- 选项2：`.env`
- 选项3：手动输入

将用户回答记为 **`<venv_name>`**。

---

### Step 8：选择 Python 版本并创建虚拟环境

#### Step 8.1：检测系统中已安装的 Python 版本

根据操作系统执行以下命令检测可用的 Python：

**Windows：**
```bash
py --list
```
> 💡 `py --list` 会列出所有通过 Python Launcher 注册的版本（如 `-3.13-64`、`-3.11-64` 等）。若该命令失败（未安装 Launcher），则尝试在 `PATH` 中搜索 `python3*.exe`。

**Linux / macOS：**
```bash
ls /usr/bin/python3* /usr/local/bin/python3* 2>/dev/null && python3 --version
```
> 💡 也可以用 `which python3.x` 逐一探测，或解析 `ls /usr/bin/python3*` 的输出。

将检测到的所有可用版本整理为列表，记为 **`<available_pythons>`**。

#### Step 8.2：询问用户选择 Python 版本

用 `ask_user_question` 展示检测到的版本列表：
> "请选择用于创建虚拟环境的 Python 版本："

- 将 **`<available_pythons>`** 中的每个版本作为一个选项，**始终按版本号从高到低排序**（新版本在前，旧版本在后）
- 将排序后版本号最新的版本放在第一位并标注 `(Recommended)`
- 最后追加一个选项：手动输入

将用户选择记为 **`<python_version>`**（如 `3.11`、`3.13`）。

#### Step 8.3：执行虚拟环境创建命令

根据操作系统和所选版本，在新 worktree 目录下执行：

**Windows：**
```bash
cd <base_path>/<worktree_dir> && py -<python_version> -m venv <venv_name>
```

**Linux / macOS：**
```bash
cd <base_path>/<worktree_dir> && python<python_version> -m venv <venv_name>
```

执行后验证：
- ✅ 成功 → 提示 "虚拟环境已创建：`<base_path>/<worktree_dir>/<venv_name>`（Python <python_version>）"，进入 Step 9
- ❌ 失败 → 提示可能原因（如所选 Python 版本不支持 `-m venv`），终止流程

---

### Step 9：安装 Python 依赖

使用新 worktree 内虚拟环境的 pip，基于**当前工程根目录**的 `requirements.txt` 安装依赖：

**Windows：**
```bash
<base_path>/<worktree_dir>/<venv_name>/Scripts/pip.exe install -r <当前工程根目录>/requirements.txt
```

**Linux / macOS：**
```bash
<base_path>/<worktree_dir>/<venv_name>/bin/pip install -r <当前工程根目录>/requirements.txt
```

> 💡 虚拟环境 pip 路径：Windows 为 `<venv_name>/Scripts/pip.exe`，Linux/macOS 为 `<venv_name>/bin/pip`。

执行后：
- ✅ 成功 → 输出完成摘要（见下方）
- ❌ 失败 → 提示错误，建议用户手动执行安装命令

---

## 完成摘要

全部步骤完成后，输出如下摘要：

```
✅ Git Worktree 创建完成！

  路径：    <base_path>/<worktree_dir>
  分支：    <new_branch 或 base_branch>
  虚拟环境：<base_path>/<worktree_dir>/<venv_name>
  依赖：    已从 requirements.txt 安装完毕

下一步（Windows）：
  cd <base_path>/<worktree_dir>
  <venv_name>\Scripts\activate

下一步（Linux / macOS）：
  cd <base_path>/<worktree_dir>
  source <venv_name>/bin/activate
```

---

## 注意事项

- 本 skill 支持 **Windows、Linux、macOS** 跨平台使用，虚拟环境路径因 OS 不同有所差异：
  - Windows：`<venv_name>/Scripts/`
  - Linux / macOS：`<venv_name>/bin/`
- 同一个分支不能同时被两个 worktree checkout，如报错请先检查 `git worktree list`
- `requirements.txt` 必须存在于当前工程根目录，否则 pip install 步骤会失败
