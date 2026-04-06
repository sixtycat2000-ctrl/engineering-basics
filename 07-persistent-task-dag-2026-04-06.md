# AI Task Graph：持久化任务图

> 扁平清单不够用。真实目标有依赖：B 要等 A 做完才能开始，C 和 D 可以并行，E 要等 C 和 D 都完成。任务图是项目骨架——对话会消失，任务不会。

---

## 为什么 s03 的 TodoManager 不够

s03 的 TodoManager 解决了"不忘记计划"的问题。但它有两个致命限制：

**1. 没有依赖关系。** 10 个待办项之间谁先谁后？哪些可以并行？模型靠"感觉"判断，经常搞错。

**2. 只活在内存里。** s06 的上下文压缩一跑，TodoManager 的状态就没了。模型忘了哪些做完了、哪些没开始。重启 Agent？清单彻底消失。

真实的目标是有结构的。一个"重构认证模块"的大目标：

```
拆解后的依赖关系：
  设计方案 → 实现 DAO 层 → 实现服务层 → 实现 API 层 → 写集成测试
                    ↘ 实现缓存层 ↗              ↘ 写单元测试 ↗
```

没有显式的依赖图，Agent 分不清什么能做、什么被卡住、什么能同时跑。

---

## 任务图（DAG）

把扁平清单升级为**有向无环图（DAG）**。每个任务是一个 JSON 文件，有状态和前置依赖：

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

任务图：
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

顺序: task 1 必须先完成，才能开始 2 和 3
并行: task 2 和 3 可以同时执行
依赖: task 4 要等 2 和 3 都完成
```

任务图随时回答三个问题：
- **什么可以做？** → pending 且 blockedBy 为空
- **什么被卡住？** → blockedBy 里有未完成的任务
- **什么做完了？** → completed，完成时自动解锁后续任务

---

## 工作原理

### TaskManager：磁盘持久化 + 依赖图

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

每个任务一个 JSON 文件。为什么不是一个大 JSON？因为：
- 并发安全：多个 Agent 同时操作不同任务不会冲突（后面 s09 的团队协作需要）
- 可读性：`cat .tasks/task_3.json` 直接看单个任务
- 原子性：读写一个文件不会影响其他文件

### 依赖解除：完成任务时自动解锁

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

当 task 1 完成时，遍历所有其他任务，把 `blockedBy` 里的 1 移除。这会让 task 2 和 task 3 的 `blockedBy` 变空——它们从"被卡住"变成"可以做"。

**这是 DAG 的核心机制：完成一个节点，自动解锁它的下游节点。**

### 状态变更 + 依赖关联

```python
def update(self, task_id, status=None,
           add_blocked_by=None, remove_blocked_by=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    if add_blocked_by:
        task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
    if remove_blocked_by:
        task["blockedBy"] = [x for x in task["blockedBy"]
                             if x not in remove_blocked_by]
    self._save(task)
```

四个操作合一：更新状态、添加依赖、移除依赖、自动解锁。一个 `update` 调用搞定所有状态转换。

### 四个任务工具

```python
TOOL_HANDLERS = {
    # ...基础工具...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

---

## 隐性知识

### 为什么任务要持久化到磁盘

内存状态有两个敌人：压缩和重启。s06 的上下文压缩会清除之前的对话内容；重启 Agent 会清空所有内存。但任务图的存在超越了单次对话。

磁盘持久化意味着：
- Agent 被压缩后可以重新读取 `.tasks/` 恢复进度
- Agent 重启后可以从磁盘加载任务图继续工作
- 多个 Agent 实例可以共享同一个任务图（后面 s09 的基础）

### 任务图是后续所有机制的骨架

从 s07 开始，任务图成为协调的核心：
- s08（后台任务）→ 后台执行任务图中的可执行项
- s09（Agent 团队）→ 多个 Agent 从同一个任务图认领工作
- s12（Worktree 隔离）→ 每个任务绑定独立的 git worktree

**所有这些机制都读写同一个 `.tasks/` 目录。** 任务图是它们的共享状态。

### 为什么用文件而不是数据库

SQLite 或 Redis 当然更"正规"。但文件系统有几个独特优势：
- **零依赖**：不需要安装数据库
- **人类可读**：`ls .tasks/ && cat .tasks/task_1.json`
- **Git 友好**：任务状态可以 commit，跨机器同步
- **足够快**：任务数量通常不超过几百个，文件系统完全够用

这是 "worse is better" 的经典体现——简单方案覆盖 99% 的场景。

### owner 字段为后面留了门

`"owner": ""` 字段当前为空，但 s09 的团队协作会用到：Agent 认领任务时写入自己的名字。同一个字段，两种用途，不需要改结构。

---

## 状态流转

```
pending → in_progress → completed
                          │
                          └──→ 自动从下游任务的 blockedBy 中移除
```

三个状态，一个方向。没有 `completed → pending` 的回退——任务做完了就是做完了。如果需要重做，创建新任务。

---

## 相对 s06 的变更

| 组件 | s06 | s07 |
|------|-----|-----|
| 工具 | 5 | 8（+task_create/update/list/get） |
| 规划模型 | 扁平清单（内存） | 带依赖的任务图（磁盘） |
| 关系 | 无 | `blockedBy` 有向边 |
| 状态追踪 | 做完没做完 | pending → in_progress → completed |
| 持久化 | 压缩后丢失 | 压缩和重启后存活 |

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
