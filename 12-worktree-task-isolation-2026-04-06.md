# AI Worktree Isolation：任务级目录隔离

> 所有 Agent 共享一个目录？A 改 config.py，B 也改 config.py，未提交的改动互相污染。给每个任务一个独立的 git worktree，用任务 ID 绑定。任务板管"做什么"，worktree 管"在哪做"，各管各的。

---

## 为什么共享目录不行

到 s11，Agent 已经能自主认领和完成任务。但所有任务共享一个工作目录。问题场景：

```
Agent A 认领了 task 1：重构 config.py
Agent B 认领了 task 2：更新 README（但 README 里引用了配置格式）

两个 Agent 在同一个目录里工作：
1. A 修改了 config.py（未提交）
2. B 也读了 config.py，基于旧格式更新了 README
3. A 提交了新的 config.py
4. B 的 README 改动基于旧 config，内容已经过时
```

更严重的情况：两个 Agent 同时 `git add .`，互相提交了对方的未完成代码。

**任务板管"做什么"但不管"在哪做"。** 解法：给每个任务一个独立的目录。

---

## 架构：控制面 + 执行面

```
Control plane (.tasks/)             Execution plane (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (worktree 注册表)
                          events.jsonl (生命周期日志)

State machines:
  Task:     pending → in_progress → completed
  Worktree: absent  → active      → removed | kept
```

两层各自管理各自的状态，通过 `task_id` + `worktree_name` 双向绑定。

---

## 工作原理

### 1. 创建任务

```python
TASKS.create("Implement auth refactor")
# → .tasks/task_1.json  status=pending  worktree=""
```

任务先进入 pending 状态，还没有关联 worktree。

### 2. 创建 worktree 并绑定任务

```python
WORKTREES.create("auth-refactor", task_id=1)
# → git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# → index.json 新增条目
# → task_1.json 更新 worktree="auth-refactor" + status="in_progress"
```

一个调用同时写入两侧状态：

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

创建 worktree 时自动把任务推进到 `in_progress`。**这是合理的——有了工作空间就意味着开始工作了。**

### 3. 在 worktree 中执行命令

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

`cwd` 指向隔离目录。每个 worktree 有自己的分支（`wt/auth-refactor`），和主分支完全隔离。

### 4. 收尾：keep 或 remove

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

两个选择：
- `worktree_keep(name)` —— 保留目录，供后续使用（比如需要 review）
- `worktree_remove(name, complete_task=True)` —— 删除目录 + 完成任务 + 记录事件。一个调用搞定拆除 + 完成

### 5. 事件流

每个生命周期步骤写入 `.worktrees/events.jsonl`：

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

事件类型：`worktree.create.before/after/failed`、`worktree.remove.before/after/failed`、`worktree.keep`、`task.completed`。

---

## 隐性知识

### 任务 ID 是外键

关系型数据库用外键关联表。这里用 `task_id` 关联 `.tasks/` 和 `.worktrees/`。两边都能通过这个 ID 找到对方：
- 从 task 找 worktree：读 task JSON 的 `worktree` 字段
- 从 worktree 找 task：读 index.json 里 worktree 的 `task_id` 字段

这个双向关联是整个隔离机制的基础。没有它，任务板和 worktree 就是两个不相关的系统。

### 为什么用 git worktree 而不是 cp -r

复制目录也能隔离。但 git worktree 有独特优势：
- **共享 git 历史**：worktree 不是独立的 clone，它和主仓库共享 .git。clone 要几百 MB，worktree 只要几 KB 的元数据
- **分支管理**：每个 worktree 自动关联一个分支（`wt/auth-refactor`）。做完了直接合并，不用手动同步
- **创建/删除快**：`git worktree add` 比 `cp -r` 快一个数量级
- **Git 原生支持**：不需要额外的同步机制

### events.jsonl 是可审计的历史

为什么需要事件流？因为 `.tasks/` 和 `.worktrees/` 只存当前状态，不存历史。"task 1 什么时候开始的？worktree auth-refactor 创建过几次？" 这些问题只能从事件流回答。

事件流和数据库的 WAL（Write-Ahead Log）一样：记录每一次状态变更，用于恢复和审计。

### 崩溃恢复

如果 Agent 在执行过程中崩溃（进程被杀、机器重启），磁盘上的状态还在：
- `.tasks/task_1.json` 显示 `status: "in_progress"`
- `.worktrees/index.json` 显示 `auth-refactor: active`
- `.worktrees/auth-refactor/` 目录还在，文件还在

恢复流程：读取 `.tasks/` 和 `.worktrees/index.json`，找到所有 `in_progress` 的任务和 `active` 的 worktree，重新绑定关系。Agent 可以从断点继续。

**会话记忆是易失的，磁盘状态是持久的。** 这就是为什么从 s07 开始所有状态都持久化到磁盘。

### keep vs remove 的选择

什么时候 keep？任务完成了但代码需要人工 review。保留 worktree 让人类可以 inspect。

什么时候 remove？任务完成且代码已合并到主分支。worktree 不再需要。

`complete_task=True` 参数的设计是便捷性考虑：通常"删除 worktree"和"完成任务"是一起发生的。一个参数省去了两次调用。

---

## 相对 s11 的变更

| 组件 | s11 | s12 |
|------|-----|-----|
| 协调 | 任务板（owner/status） | 任务板 + worktree 显式绑定 |
| 执行范围 | 共享目录 | 每个任务独立目录 |
| 可恢复性 | 仅任务状态 | 任务状态 + worktree 索引 |
| 收尾 | 任务完成 | 任务完成 + keep/remove |
| 生命周期可见性 | 隐式 | events.jsonl 显式事件流 |

---





## 系列导航

1. [Agent Loop 基础](01-agent-loop-fundamentals-2026-04-06.md) →
2. [工具分发设计](02-tool-dispatch-design-2026-04-06.md) →
3. [Todo 驱动规划](03-todo-driven-planning-2026-04-06.md) →
4. [Subagent 上下文隔离](04-subagent-context-isolation-2026-04-06.md) →
5. [按需加载 Skill](05-on-demand-skill-loading-2026-04-06.md) →
6. [三层上下文压缩](06-three-layer-context-compression-2026-04-06.md) |
7. [持久化任务图](07-persistent-task-dag-2026-04-06.md) →
8. [后台执行](08-background-task-execution-2026-04-06.md) →
9. [多 Agent 团队](09-multi-agent-team-coordination-2026-04-06.md) →
10. [团队协议](10-request-response-protocols-2026-04-06.md) →
11. [自组织团队](11-self-organizing-agent-teams-2026-04-06.md) →
12. [Worktree 任务隔离](12-worktree-task-isolation-2026-04-06.md)
