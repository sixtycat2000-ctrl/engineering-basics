# tmux + AI Coding CLI：实用指南

> tmux 让终端变成"指挥中心"，持久、并行、不乱。配合 Claude Code / Codex CLI / Copilot CLI 使用效果最佳。

---

## 为什么用 tmux

- **会话不丢** —— AI 跑几个小时，终端关了、SSH 断了也没事，随时接回来
- **多任务并行** —— 左边跑 AI 生成代码，右边跑测试，下面看 git
- **多 AI 同时用** —— Claude Code 做复杂任务、Copilot CLI 快速问答、Codex 做结构化任务，互不打扰
- **方便 review** —— AI 改完立刻在旁边 pane 看 diff 或跑测试
- **布局固定** —— 每次打开项目同样的干净布局，养成习惯效率高

---

## 推荐配置

在 `~/.tmux.conf` 里放这些（没有就新建）：

```conf
# 前缀键改成 Ctrl+a（比默认 Ctrl+b 好按）
set -g prefix C-a
unbind C-b
bind C-a send-prefix

set -g mouse on                # 鼠标点击切换窗口、滚动
set -g history-limit 100000    # AI 输出很长，必须够大

# vim 风格切换窗格
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# 分屏：保留当前目录
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# 大写 Z 放大/缩小当前 pane（避免和小写 z 冲突）
bind Z resize-pane -Z
```

改完后重启 tmux，或在 tmux 里按 `Ctrl+a :` 输入 `source-file ~/.tmux.conf`。

---

## 核心快捷键

> 所有快捷键先按 `Ctrl+a`，再按对应键。

| 快捷键 | 作用 |
|--------|------|
| `d` | 分离会话（AI 继续后台跑） |
| `\|` | 左右分屏 |
| `-` | 上下分屏 |
| `Z` | 放大/缩小当前 pane（再按一次恢复） |
| `x` | 关闭当前 pane（会确认） |
| `z` | 同上，tmux 默认也是关闭 pane |
| `h/j/k/l` | 切换 pane（左/下/上/右） |
| `[` | 进入复制模式（vim 键滚动，`q` 退出） |

**外部命令：**
- `tmux ls` —— 列出所有会话
- `tmux attach -t 会话名` —— 重新接回
- `tmux kill-session -t 会话名` —— 关闭整个会话

---

## 推荐布局

### 入门：2 分屏

```
┌──────────────────────┬──────────────┐
│                      │              │
│  AI 工具             │  测试 / git  │
│  (claude / codex)    │  (npm test)  │
│                      │              │
└──────────────────────┴──────────────┘
```

启动步骤：
1. `tmux new -s myproject`
2. 输入 `claude`（或 `codex`、`gh copilot`）
3. `Ctrl+a |` 分右屏
4. 在右屏输入测试命令或 `lazygit`

### 进阶：3 分屏

```
┌──────────────────────┬──────────────┐
│                      │   测试       │
│  AI 工具             │  (npm test)  │
│  (claude / codex)    ├──────────────┤
│                      │   Git        │
│                      │  (lazygit)   │
└──────────────────────┴──────────────┘
```

启动步骤：
1. `tmux new -s myproject`
2. 输入 `claude`
3. `Ctrl+a |` 左右分屏
4. 在右屏 `Ctrl+a -` 上下再分一次
5. 右上跑测试，右下跑 `lazygit`

### 高级：多 AI 并行

- 一个 pane 跑 Claude Code（复杂重构）
- 一个 pane 跑 Copilot CLI（快速生成小片段）
- 一个 pane 跑 Codex（结构化任务）

⚠️ 不要让多个 AI 同时改同一个文件，容易冲突。

---

## 完整工作流程

### 开始
```bash
cd myproject
tmux new -s myproject-feature-login   # 名字起清楚
```

### 设置布局
按上面的 2 分屏或 3 分屏方案搭建。

### 工作循环（反复做这个）
1. 在 AI pane 里描述任务，让 AI 生成/修改代码
2. AI 改完后，看旁边的测试是否通过、git diff 是否合理
3. 有问题继续在 AI 里说"修复这个错误"
4. 确认 OK 后 commit（或让 AI 帮你 commit）
5. 重复，直到功能完成

### 临时全屏查看
`Ctrl+a Z` 放大当前 pane 看长输出，看完再按一次恢复。

### 离开
`Ctrl+a d` 分离，AI 继续后台跑，可以关终端。回来时：
```bash
tmux attach -t myproject-feature-login
```

### 收尾
- 每个 pane 里输入 `exit` 退出
- 或直接：`tmux kill-session -t myproject-feature-login`

---

## 分屏管理补充

### 关闭分屏
- **关闭当前 pane**：`Ctrl+a x` → 确认输入 `y`
- **只留当前 pane**：`Ctrl+a :` → 输入 `kill-pane -a`
- **直接输入 `exit`**：退出当前 shell，等同于关闭 pane

### 调整 pane 大小
- `Ctrl+a` 然后按住 `Ctrl` 用方向键调整
- 或 `Ctrl+a :` 输入 `resize-pane -L 10`（左移 10 格）、`-R`、`-U`、`-D`

### 交换 pane 位置
`Ctrl+a :` 输入 `swap-pane -s 源pane -t 目标pane`

---

## 实用建议

- **先从 2 分屏练起**，熟练了再加 pane
- **每次 AI 改完就 review**（看 diff + 跑测试），不要一次改太多
- **让 AI 经常 commit**，方便回滚
- **一个项目一个 tmux 会话**，名字起清楚方便管理
- **scrollback 设够大**（配置里已设 100000），AI 输出经常很长
