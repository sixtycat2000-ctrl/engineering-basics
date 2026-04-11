# 上下文即命运：三角色Agent分工架构

> Agent的行为由上下文决定。把规划、实现、验证分给三种角色的Agent，每个Agent用干净的上下文、单一的目标，通过验证循环不断收敛到正确结果。像一个建筑工地——总工不搬砖，工人不管验收，监理不改墙。

---

## 为什么需要这个架构

### 问题：上下文污染

Agent越工作越笨。不是模型差，是**上下文越来越脏**。

```
Agent上下文随时间的变化：

Turn 1: [用户需求]                              ← 干净
Turn 2: [用户需求] + [读5个文件]                 ← 开始膨胀
Turn 3: [...] + [写3个函数] + [调试输出]         ← 无关信息累积
Turn 4: [...] + [改了又改] + [搜索结果]          ← 思维发散
Turn 5: ...全部混在一起...                       ← 什么都干不好
```

两种失败模式：

| 失败模式 | 原因 | 后果 |
|---------|------|------|
| **无关上下文累积** | 任务太宽，读了太多不相关信息 | 推理方向发散 |
| **对抗性上下文累积** | 自己实现的东西自己评估 | 偏袒自己的实现 |

### 核心洞察

**Agent的上下文决定了它的行为。**

LLM的推理是append-only的——每一步思考都是之前所有步骤的函数。模型追求一致性（coherence），会把上下文里的信息整合成一个统一的世界观。

当上下文里堆积了和当前目标无关的信息，性能就下降。当上下文里包含了之前的实现过程，Agent就很难客观地审查自己的代码。

一句话：**上下文即命运。**

---

## 整体架构

```
┌────────────────────────────────────┐
│           Orchestrator             │
│  规划 → 写合约 → 拆Feature → 看结果 │
│  不实现、不验证，只规划              │
└───┬────────────┬────────────┬─────┘
    │            │            │
  派发Feature  触发Validate  处理Fix
    │            │            │
    ▼            ▼            │
┌────────┐  ┌──────────┐    │
│ Worker │  │Validator │    │
│ 先写测试│  │独立审查  │    │
│ 再写代码│  │不改代码  │    │
│ 跑完就交│  │只报问题  │    │
└────────┘  └────┬─────┘    │
                  │          │
              问题清单        │
                  │          │
                  ▼          │
           ┌──────────┐     │
           │Fix Feature│◄───┘
           │(新Worker) │
           └──────────┘

     共享状态（合约/Feature列表/知识库）
```

三种角色，三种上下文，三种目标：

| 角色 | 目标 | 看什么 | 不看什么 |
|------|------|--------|---------|
| **Orchestrator** | 规划正确、验收通过 | 需求、Feature列表、验证结果 | 实现细节 |
| **Worker** | Feature实现正确 | Feature规格、相关代码 | 其他Feature、整体架构 |
| **Validator** | 找出所有问题 | 实现代码、验证合约 | 之前的实现过程 |

---

## 核心原则

### 原则 1：实现者和验收者必须是不同的Agent

```python
# ❌ 错误：让实现者自己验收
worker_prompt = """
实现登录功能。
实现完后检查一下有没有bug。
"""
# Agent会偏袒自己的实现——
# "我写的代码，应该没问题"

# ✅ 正确：独立的验收者
worker_prompt = """
实现登录功能。测试用例必须覆盖：
- 正常登录
- 密码错误
- 账号不存在
"""
validator_prompt = """
以下代码是否满足验收标准？
只报告问题，不修改代码。
"""
```

**为什么：** 模型追求一致性。如果一个Agent先实现了代码再审查代码，它会倾向于证明自己是对的——"我的代码有bug"和之前的推理过程矛盾。

### 原则 2：先定义正确性，再定义实现

```
传统流程：
  需求 → 实现Feature → 写测试（按实现写）

正确流程：
  需求 → 写验证合约 → 拆Feature → 实现 → 验证
         ↑ 先定义"什么是正确"
```

如果先写Feature再写验收标准，标准会被已规划的实现污染——你会写出"正好能通过"的标准，而不是"真正正确"的标准。

**两层TDD：**

| 层级 | 谁写 | 写什么 | 何时写 |
|------|------|--------|--------|
| Feature层 | Worker | 单元测试 | 写代码**之前** |
| Mission层 | Orchestrator | 验证合约 | 定义Feature**之前** |

### 原则 3：外部化状态

没有一个Agent需要掌握全部信息。状态分布在共享文件里：

```
mission/
├── contract.md          # 验收标准
├── features/
│   ├── 01-auth.md      # Feature规格
│   ├── 02-channel.md   # 每个Feature自包含
│   └── ...
├── knowledge.md         # 积累的经验教训
└── guidelines.md        # Worker行为规范
```

每个Agent只读自己需要的文件。

```python
# ❌ Orchestrator 什么都读
ctx = [requirement, all_code, all_tests, all_logs]
# 上下文爆炸，推理质量下降

# ✅ Orchestrator 只看摘要
ctx = [requirement, feature_summary, validation_report]
# 调查委托子Agent，自己不碰细节
```

### 原则 4：模型专精化

角色分离后，可以为每个角色选最合适的模型：

| 角色 | 需要什么 | 选什么模型 |
|------|---------|-----------|
| Orchestrator | 宽泛规划、判断力 | 最强推理模型 |
| Worker | 可靠执行、代码质量 | 性价比高的编码模型 |
| Validator | 吹毛求疵、全面审查 | 最仔细的审查模型 |

---

## 实现步骤

### Step 1：Orchestrator 规划

接到需求后的工作流：

```
1. 调查需求
   ├── 读代码库（委托子Agent）
   ├── 问用户澄清问题
   └── 直到需求无歧义

2. 写验证合约
   └── 可测试的行为断言
   └── 定义"完成"的标准

3. 拆Feature
   ├── 每个Feature声明满足哪些断言
   └── 分组为Milestone

4. 创建共享状态
   ├── Worker行为规范
   └── 知识库（初始为空）
```

**验证合约长什么样：**

```markdown
# 验证合约：Slack Clone

- [ ] 用户可以注册和登录
- [ ] 可以创建Channel
- [ ] Channel内可以发消息
- [ ] 消息支持@提及，被提及者收到通知
- [ ] 可以上传文件到Channel
- [ ] 搜索能找到包含关键词的消息
```

每个断言都是**可黑盒验证**的——不看代码，只用系统就能验证。

### Step 2：Worker 实现

每个Feature由一个**全新的Worker**执行。Worker的上下文只包含：

```python
worker_context = {
    "system": WORKER_PROMPT,       # 行为规范
    "feature": feature_spec,        # 这个Feature的规格
    "guidelines": guidelines,       # 共享规范
    "related_code": relevant_files, # 只读相关文件
}
# 没有：其他Feature的上下文
# 没有：之前的实现历史
# 没有：全局架构讨论
```

Worker的工作顺序：

```
1. 读Feature规格
2. 先写测试（反映预期行为）
3. 写实现代码
4. 跑测试，确保通过
5. 交出去——正确性由Validator判断
```

⚠️ Worker不判断自己是否最终正确。它认为自己做完了就交，最终判断权在Validator。

### Step 3：Validator 审查

Milestone内所有Feature完成后，触发验证。**用全新的Agent。**

两种Validator：

| 类型 | 做什么 | 怎么做 |
|------|--------|--------|
| **代码审查** | 检查质量和正确性 | 读代码 + 审查实现轨迹 |
| **用户模拟** | 像用户一样用系统 | 黑盒测试，不看代码 |

```python
# 用户模拟Validator的上下文
validator_context = {
    "system": VALIDATOR_PROMPT,  # 严格质检员
    "contract": contract,         # 验收标准
    "system_url": "http://localhost:3000",
}
# 没有：实现代码
# 没有：Worker的推理过程
# 纯黑盒：像真实用户一样操作
```

### Step 4：验证循环

```
    ┌──────────────┐
    │ 实现所有      │
    │ Feature       │
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  Validation   │ ← 全新Agent
    │  审查 + 测试  │
    └──────┬───────┘
      ┌────┴────┐
   通过✅   不通过❌
      │         │
      ▼         ▼
    完成    创建Fix Feature
                 │
                 ▼
          新Worker实现修复 ◄┐
                 │          │
                 ▼          │
          重新Validation ───┘
          （2-4轮收敛）
```

真实数据（Factory.ai的Slack Clone项目）：

| 指标 | 数值 |
|------|------|
| 总代码行数 | 38,800行 |
| 测试占代码比例 | 52.5% |
| 语句覆盖率 | 89.25% |
| Validation发现的问题 | 81个 |
| 生成的修复Feature | 21个 |
| 修复占总实现工作 | 34.4% |
| 每Milestone收敛轮数 | 2-4轮 |
| Validation占总运行时间 | 37.2% |

---

## 三档递进

### 入门：两个Agent + 验证循环

5分钟能跑的最小版本：

```python
def run_task(requirement: str):
    # Agent 1: 实现
    worker = Agent(role="worker")
    worker.run(f"实现：{requirement}\n先写测试再写代码。")

    # Agent 2: 验证（全新上下文）
    validator = Agent(role="validator")
    issues = validator.run(
        f"验收标准：{requirement}\n"
        f"检查代码是否满足。只报问题：\n"
        f"{worker.output}"
    )

    if not issues:
        return "PASS"

    # 修复（新Agent）
    fixer = Agent(role="worker")
    fixer.run(f"修复以下问题：\n{issues}")
    return "FIXED"
```

核心就一条：**实现和验证用不同的Agent。**

### 进阶：三角色 + 外部化状态 + Milestone

加入Orchestrator，用文件共享状态：

```python
def run_milestone(milestone, contract):
    # Phase 1: 每个Feature一个Worker
    for feature in milestone["features"]:
        worker = Agent(role="worker", context=[
            f"规格：{feature['spec']}",
            read("guidelines.md"),
        ])
        worker.run()  # 先写测试再写代码
        write(f"results/{feature['id']}.md", worker.output)

    # Phase 2: 验证（全新Agent）
    validator = Agent(role="validator", context=[
        f"验收标准：{contract}",
        "入口：http://localhost:3000",
    ])
    report = validator.run()  # 黑盒测试

    # Phase 3: 有问题就修
    if report.has_issues:
        fixes = orchestrator.run(
            f"根据报告创建修复：{report}"
        )
        return run_milestone(fixes, contract)  # 递归

    return "MILESTONE PASSED"
```

日常80%的场景够用。

### 高级：模型专精 + 知识积累 + 收敛保障

```python
class MissionRunner:
    def __init__(self):
        self.models = {
            "orchestrator": "claude-opus",  # 最强规划
            "worker": "claude-sonnet",      # 性价比编码
            "validator": "claude-opus",     # 最严格审查
        }
        self.knowledge = KnowledgeBase()
        self.max_rounds = 4  # 防止无限循环

    def run(self, requirement):
        contract = self.orchestrate(requirement)

        for milestone in contract.milestones:
            for _ in range(self.max_rounds):
                self.execute_features(milestone)
                report = self.validate(contract)
                self.knowledge.update(report.learnings)

                if report.passed:
                    break
                milestone = self.create_fixes(report)
            else:
                # 超过最大轮数，交给人类
                self.escalate_to_user(milestone)
```

---

## 常见错误

```
❌ 错误 1：Worker自己验收

Worker: "我实现了登录，测试通过了，应该没问题。"
→ 模型会自然地确认自己的工作

✅ 正确：独立Validator验收

Validator（全新上下文）:
  "密码少于8位时没有错误提示。"
```

```
❌ 错误 2：Orchestrator什么都读

读了全部源码、全部测试输出、全部日志
→ 上下文膨胀，推理质量下降

✅ 正确：委托调查

派子Agent去调查，自己只看摘要
```

```
❌ 错误 3：验收标准写在Feature之后

先写Feature → 再写验收标准
→ 标准会被实现思路污染

✅ 正确：先写合约再拆Feature

写验收标准 → 拆Feature → 实现
→ 标准反映真实需求，不受实现影响
```

---

## 速查表

### 三角色速查

| 角色 | 唯一目标 | 做 | 不做 |
|------|---------|-----|------|
| Orchestrator | 规划正确 | 规划、调度、创建修复 | 不实现、不验证 |
| Worker | Feature正确 | 写测试、写代码 | 不判断最终正确性 |
| Validator | 找出问题 | 审查代码、黑盒测试 | 不修改代码 |

### 设计决策速查

| 决策 | 选择 | 原因 |
|------|------|------|
| Worker验收自己代码？ | 否 | 自我偏袒 |
| Validator修复bug？ | 否 | 角色混淆 |
| 合约何时写？ | Feature之前 | 防止标准被实现污染 |
| Worker复用上下文？ | 否 | 上下文污染 |
| Orchestrator读源码？ | 否 | 委托子Agent |
| 验证循环有上限？ | 是 | 防止无限循环 |

### 健康指标参考

| 指标 | 健康范围 | 说明 |
|------|---------|------|
| 测试占代码比例 | >50% | Worker先写测试的结果 |
| Validation占时间 | 30-40% | 这是投资，不是浪费 |
| 每Milestone收敛轮数 | 2-4轮 | 超过4轮要人工介入 |
| 修复工作占比 | 30-35% | Validation的价值体现 |
| Worker中位步数 | 40-60步 | 太多说明任务太宽 |

---

## 参考资料

- [How Missions Work — Factory.ai](https://factory.ai/news/missions-architecture) — 本文的模式来源
- [Agent Loop 基础](01-agent-loop-fundamentals-2026-04-06.md) — 单Agent循环机制
- [Subagent 上下文隔离](04-subagent-context-isolation-2026-04-06.md) — 上下文隔离的基础概念
- [多 Agent 团队](09-multi-agent-team-coordination-2026-04-06.md) — 多Agent协调模式
- [持久化任务图](07-persistent-task-dag-2026-04-06.md) — 任务依赖管理

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
12. [Worktree 任务隔离](12-worktree-task-isolation-2026-04-06.md) →
13. **三角色Agent分工架构**（本文）
