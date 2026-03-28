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

## 高级用法

### 自动化布局（tmuxinator）

每次手动分屏很烦？用 **tmuxinator** 一次配置，一键启动：

```bash
# 安装
brew install tmuxinator

# 创建项目配置
tmuxinator new myproject
```

编辑生成的 `~/.tmuxinator/myproject.yml`：

```yaml
name: myproject
root: ~/projects/myproject
windows:
  - editor:
      layout: main-vertical
      panes:
        - claude
        - npm run dev
  - test:
      layout: main-vertical
      panes:
        - npm test -- --watch
        - lazygit
```

以后直接：
```bash
tmuxinator start myproject
```

秒开完整布局，不用再手动分屏。

### 从 tmux 外部发送命令到 AI pane

AI 在后台跑着，你可以从其他 pane 甚至其他终端往 AI pane 发消息：

```bash
# 向指定 pane 发送文字（模拟键盘输入）
tmux send-keys -t myproject:0.0 "请重构 auth 模块" Enter
```

**实用场景：**
- 写个脚本，跑完测试后自动把结果发给 AI："测试失败了，请修复"
- 在另一个 pane 写代码时，不用切过去就能给 AI 派新任务

### 配合 watch 自动监控文件变化

```bash
# 右侧 pane 用 watch 监控 AI 改了哪些文件
watch -n 2 "git diff --stat"

# 或者监控特定文件的变化
watch -n 1 "tail -20 logs/app.log"

# 监控 AI 的 token 消耗（如果工具支持输出日志）
watch -n 5 "tail -5 ~/.claude/stats.log"
```

### 远程开发（SSH + tmux）

在服务器上开发时，tmux + AI CLI 是最强组合：

```bash
# SSH 到服务器
ssh dev-server

# 启动 tmux 会话
tmux new -s backend-api

# 跑 AI 工具
claude

# 网络断了也没关系，重连后：
ssh dev-server
tmux attach -t backend-api
# 一切都在，AI 可能还在跑
```

**多窗口管理：**
```bash
# 在同一个会话里开多个窗口（类似浏览器标签页）
Ctrl+a c        # 新建窗口
Ctrl+a 1/2/3    # 切换到第 1/2/3 个窗口
Ctrl+a ,        # 重命名当前窗口
Ctrl+a &        # 关闭当前窗口
```

推荐按功能分窗口：
- 窗口 1：`code` —— AI + 编辑器
- 窗口 2：`test` —— 测试 + 日志
- 窗口 3：`git` —— lazygit + diff

### AI 长任务的后台管理

AI 跑一个耗时任务（比如重构整个模块），你可以：

```bash
# 方案 1：detach 让 AI 在后台跑
Ctrl+a d
# 去干别的，回来时 tmux attach -t 会话名

# 方案 2：开新窗口干别的事
Ctrl+a c        # 新窗口，AI 还在旧窗口跑

# 方案 3：把 AI pane 移到后台窗口
Ctrl+a :        # 输入 break-pane -d
# AI pane 变成独立窗口，你可以继续在当前窗口工作
```

### 自动保存 tmux 环境（tmux-resurrect）

```bash
# 安装插件管理器
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# 在 ~/.tmux.conf 末尾加：
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
run '~/.tmux/plugins/tpm/tpm'

# 安装插件：在 tmux 里按 Ctrl+a I（大写 i）
```

之后：
- `Ctrl+a Ctrl+s` —— 保存所有会话、窗口、pane 布局
- `Ctrl+a Ctrl+r` —— 恢复

电脑重启后也能恢复之前的布局。AI 的运行状态不会恢复（进程没了），但布局不用重新搭。

### 脚本化 AI 工作流

把常见操作写成脚本，一键执行：

```bash
#!/bin/bash
# ~/bin/ai-review.sh —— AI 改完代码后自动 review

PROJECT_SESSION="myproject"

# 1. 跑测试
tmux send-keys -t $PROJECT_SESSION:1.0 "npm test" Enter

# 2. 等 10 秒让测试跑完
sleep 10

# 3. 把测试结果发给 AI
TEST_OUTPUT=$(tmux capture-pane -t $PROJECT_SESSION:1.0 -p | tail -20)
tmux send-keys -t $PROJECT_SESSION:0.0 "测试结果：$TEST_OUTPUT，请修复失败项" Enter
```

### pane 里直接捕获 AI 输出

```bash
# 捕获当前 pane 的可见内容到文件
tmux capture-pane -t myproject:0.0 -S -3000 -p > ai-output.txt

# -S -3000 表示向上捕获 3000 行（AI 输出通常很长）
# -p 输出纯文本，不带 tmux 格式

# 配合管道做后续处理
tmux capture-pane -t myproject:0.0 -p | grep "Error:" | sort | uniq
```

---

## 实用建议

- **先从 2 分屏练起**，熟练了再加 pane
- **每次 AI 改完就 review**（看 diff + 跑测试），不要一次改太多
- **让 AI 经常 commit**，方便回滚
- **一个项目一个 tmux 会话**，名字起清楚方便管理
- **scrollback 设够大**（配置里已设 100000），AI 输出经常很长
- **用 tmuxinator 固化常用布局**，别每次手动搭
- **`send-keys` + 脚本** 是自动化 AI 工作流的杀手锏
- **远程开发时 tmux 是刚需**，网络断了也不丢进度
