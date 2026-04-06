# AI Tool Dispatch：工具分发

> 加一个工具 = 加一个 handler 函数 + 加一个 dict entry。循环永远不动。dispatch map 是 Open-Closed Principle 的最纯粹实现——对扩展开放，对修改关闭。

---

## 为什么需要专用工具

只有 bash 时，所有操作都走 shell。表面上看什么都能做，实际上处处是坑：

- `cat` 读大文件时截断不可预测，不知道 LLM 拿到的是完整内容还是前半截
- `sed` 替换遇到特殊字符（反斜杠、引号、换行）就崩
- 每次 bash 调用都是不受约束的安全面——`rm -rf /` 在 bash 里完全合法
- LLM 生成的 shell 命令不总是正确的，shell 语法太灵活，容易出错

专用工具的核心价值不是"更好用"，而是**更可控**。`read_file` 可以精确控制读多少行、截断到多少字符。`write_file` 不经过 shell 解析，不存在注入问题。`edit_file` 做 old/new 替换，比 `sed` 安全得多。

**关键洞察：加工具不需要改循环。** 循环只做一件事——查名字，调函数。函数做什么，循环不关心。

---

## 分发机制

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

dispatch map = {tool_name: handler_function}
一个字典查找替代任何 if/elif 链。
```

### 为什么是字典而不是 if/elif

if/elif 在 3 个工具时还能看，到 10 个工具就是面条代码。字典查找：
- O(1) 时间复杂度
- 加工具 = 加一行，不动已有代码
- 可以在运行时动态注册/注销
- 工具名和处理函数的映射关系一目了然

这不是什么高深的设计模式，这就是 Strategy Pattern 去掉了所有仪式感。

---

## 路径沙箱

在讲工具实现之前，先解决安全问题。

**LLM 可以被 prompt injection 攻击。** 一个恶意文件里的注释可能让模型读取 `/etc/passwd` 或覆盖重要配置。如果工具没有路径限制，模型拿到的是完整的文件系统访问权。

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

`resolve()` 把相对路径变成绝对路径（处理 `../../etc/passwd` 这类攻击）。`is_relative_to()` 确保最终路径仍在工作区内。**这个函数是所有文件操作工具的安全边界。**

这个沙箱的隐含假设：工作目录是信任边界。工作目录外的文件，Agent 不应该碰。这个假设在大多数场景下成立——你让 Agent 重构一个项目，它不需要也不应该访问其他项目。

---

## 工具实现

### read_file

```python
def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

两个截断点：
- `limit` 控制行数——LLM 不需要读 10000 行的日志文件
- `50000` 字符上限——防止单次工具调用吃光上下文

对比 bash 的 `cat`：cat 没有截断控制，输出多长就是多长，直接塞进上下文可能撑爆窗口。

### write_file

```python
def run_write(path: str, content: str) -> str:
    safe_path(path).write_text(content)
    return f"Wrote {len(content)} chars to {path}"
```

不经过 shell 解析，不存在 shell 注入问题。内容直接写入文件。

### edit_file

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    fpath = safe_path(path)
    content = fpath.read_text()
    if old_text not in content:
        return f"Error: old_text not found in {path}"
    if content.count(old_text) > 1:
        return f"Error: old_text appears {content.count(old_text)} times"
    new_content = content.replace(old_text, new_text, 1)
    fpath.write_text(new_content)
    return f"Edited {path}"
```

三个保护措施：
1. `old_text` 必须存在——编辑一个不存在的内容没有意义
2. `old_text` 必须唯一——如果出现多次，说明匹配不精确，不编辑比乱编辑好
3. 只替换第一次出现——精确控制修改范围

这些检查在 bash 的 `sed` 里都不存在。`sed -i 's/old/new/' file` 会替换所有匹配，如果你的 old 不够精确，可能改错地方。

---

## 分发表注册

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

循环中按名称查找：

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

`TOOL_HANDLERS.get(block.name)` 找不到就返回 None，走到 else 分支输出 "Unknown tool"。模型看到这个错误信息，会尝试用别的工具或调整参数——**错误信息本身也是引导模型行为的方式。**

---

## 隐性知识

### 工具是 Agent 的"手"

没有工具，LLM 只能想，不能做。它可以写出完美的代码，但代码只存在于回复文本里，不会自动出现在文件系统上。工具把 LLM 的"想法"变成"动作"。

选择暴露哪些工具，就是选择给 Agent 多大的行动范围。给 bash 就是给了一切权限；给 read_file + write_file 就是限制了行动方式但提高了安全性。**工具集的设计 = Agent 的能力边界设计。**

### LLM 怎么知道用什么工具

模型通过 `tools` 参数拿到工具的 JSON Schema——名字、描述、参数定义。它根据用户 prompt 和当前上下文，决定调用哪个工具、传什么参数。

这意味着**工具的描述文本直接决定了模型会不会用它、用得好不好。** 一个描述为"读文件"的工具和一个描述为"读取指定文件的内容，支持行号范围限制，适用于查看代码、配置文件、日志"的工具，模型的使用准确度完全不同。

### 输出截断是必要的设计

每个工具的返回值都有长度限制（`[:50000]`）。这不是偷懒，而是保护上下文窗口。一个 `ls -la` 在大项目里可能输出几千行，一个 `cat` 读日志文件可能输出几万行。如果不过滤，几次工具调用就能把上下文塞满。

截断策略的选择反映了优先级：**丢失尾部内容比撑爆上下文好。** 因为尾部通常是不重要的（日志的后面部分、列表的后半截），而上下文满了意味着 Agent 无法继续工作。

---

## 相对 s01 的变更

| 组件 | s01 | s02 |
|------|-----|-----|
| 工具数量 | 1（仅 bash） | 4（bash, read, write, edit） |
| 分发方式 | 硬编码 bash 调用 | `TOOL_HANDLERS` 字典 |
| 路径安全 | 无 | `safe_path()` 沙箱 |
| Agent loop | 不变 | 不变 |

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
