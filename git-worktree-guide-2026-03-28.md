# Git Worktree 完整工作流程

## 工作流程图

```
main 分支工作目录          新 worktree 目录
┌─────────────┐          ┌─────────────┐
│ ~/project   │          │ ~/project-  │
│ ▼           │          │ feature     │
│ (稳定版)    │ ──────►  │ ▼           │
│             │  git     │ (开发新功能) │
│             │ worktree │             │
└─────────────┘          └─────────────┘
          │                       │
          │    开发完成，合并代码   │
          └───────────┬───────────┘
                      ▼
                git merge feature
                      │
                      ▼
              ┌─────────────────┐
              │ 清理 worktree   │
              │ git worktree    │
              │ remove          │
              └─────────────────┘
```

---

## 1. 创建 Worktree

### 基于当前分支创建新分支

```bash
cd ~/my-project

# 确认当前分支和工作区状态
git status

# 创建 worktree（基于当前分支新建分支）
git worktree add ../my-project-feature -b feature/login
```

### 其他创建方式

```bash
# 基于特定分支创建
git worktree add ../my-project-hotfix -b hotfix/bug-123 main

# 基于标签创建
git worktree add ../my-project-v2 v2.0.0

# 为已有分支创建 worktree
git worktree add ../my-project-develop develop
```

**创建后的目录结构：**
- `~/my-project` —— 原 main 分支工作目录
- `~/my-project-feature` —— 新 feature/login 分支工作目录（独立文件夹，完全隔离）

### ⚠️ 注意：分支互斥

同一个分支不能同时被两个 worktree checkout。如果 `main` 已经在主目录 checkout，就不能再创建一个 worktree 也 checkout `main`，除非先在主目录切换到别的分支。

---

## 2. 并行开发

```bash
# 在新 worktree 中开发（完全独立的工作区）
cd ../my-project-feature

git status  # 显示 feature/login 分支

echo "new feature" > file.txt
git add .
git commit -m "feat: add login feature"

# 同时，原目录保持 main 分支不变，互不影响
cd ~/my-project
git status  # 仍然显示 main 分支，干净状态
```

---

## 3. 保持同步（推荐）

在 worktree 开发期间，定期同步主分支最新代码，避免最终合并时冲突过大：

```bash
cd ../my-project-feature

git fetch origin
git rebase origin/main   # 或 git merge origin/main
```

---

## 4. 合并回主分支

### 方法一：本地合并（直接合并）

```bash
# 回到原项目目录
cd ~/my-project

# 确保 main 分支最新
git pull origin main

# 合并 feature 分支
git merge feature/login

# 解决冲突（如有），然后推送
git push origin main
```

### 方法二：推送后提 PR（适合团队协作）

```bash
cd ../my-project-feature

# 推送到远程
git push origin feature/login

# 在 GitHub/GitLab 上创建 Pull Request，审查通过后合并到 main
```

---

## 5. 清理 Worktree

### 推荐：一步删除（Git 2.17+）

```bash
# 在任意 git 仓库目录下执行，自动删除目录 + 清理记录
cd ~/my-project
git worktree remove ../my-project-feature
```

### 手动清理（旧版 Git 或目录已被误删）

```bash
# 如果目录已经被手动删除了，清理 git 记录
cd ~/my-project
git worktree prune
```

### 删除分支（可选）

```bash
# feature 分支已合并后，清理本地和远程分支
git branch -d feature/login           # 删除本地分支
git push origin --delete feature/login # 删除远程分支
```

---

## 常用命令速查表

| 命令 | 作用 |
|------|------|
| `git worktree add <path> -b <branch>` | 创建新 worktree 并新建分支 |
| `git worktree add <path> <branch>` | 为已有分支创建 worktree |
| `git worktree add <path> <tag>` | 基于标签创建 worktree |
| `git worktree list` | 查看所有 worktree |
| `git worktree remove <path>` | 删除 worktree（Git 2.17+，推荐） |
| `git worktree prune` | 清理已不存在目录的 worktree 记录 |
| `git worktree move <old> <new>` | 移动 worktree 目录（Git 2.17+） |
| `git worktree lock <path>` | 锁定 worktree，防止被 prune 误删 |

---

## 最佳实践

### 1. 命名规范

```bash
# 统一使用：项目名-分支名
git worktree add ../project-feature-xxx -b feature/xxx
git worktree add ../project-hotfix-123 -b hotfix/bug-123
```

### 2. 定期清理已合并的 worktree

```bash
git worktree list    # 查看所有 worktree 状态
git worktree prune   # 清理僵尸记录（目录已不存在的）
```

### 3. 配合 bare 仓库使用（推荐）

bare 仓库本身没有工作区，所有 worktree 都是平等的，不存在"主目录占一个分支"的限制：

```bash
# 克隆为 bare 仓库（无工作区）
git clone --bare <repo-url> project.git
cd project.git

# 创建 worktree
git worktree add ../project-main main
git worktree add ../project-develop develop
git worktree add ../project-feature-xxx -b feature/xxx
```

这种方式下每个 worktree 地位完全对等，是 worktree 最推荐的用法。

### 4. 锁定正在使用的 worktree

CI/CD 或自动化脚本可能会误删 worktree，用 `lock` 保护：

```bash
# 锁定
git worktree lock ../my-project-feature

# 解锁
git worktree unlock ../my-project-feature
```
