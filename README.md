# agent-team-up

A skill for spawning and coordinating multi-agent teams for parallel work. Built for [Claude Code](https://claude.ai/code), designed to be agent-platform agnostic in the future.

<p align="center">
  <a href="#agent-team-up中文">📖 中文说明 / Chinese Docs</a>
</p>

<p align="center">
  <img src="assets/agent-team-up.gif" alt="agent-team-up animation" width="700"/>
</p>

## How It Works

```mermaid
graph TB
    User([👤 User])
    Lead[🎯 Team Lead]

    User -- "/team-up project" --> Lead

    subgraph Parallel Execution
        A1[🤖 Agent A<br/>Module A<br/><i>worktree: feat/a</i>]
        A2[🤖 Agent B<br/>Module B<br/><i>worktree: feat/b</i>]
        A3[🤖 Agent C<br/>Research<br/><i>no isolation</i>]
    end

    Lead -- "spawn + assign tasks" --> A1
    Lead -- "spawn + assign tasks" --> A2
    Lead -- "spawn + assign tasks" --> A3

    A1 -. "SendMessage:<br/>API spec" .-> A2
    A2 -. "SendMessage:<br/>interface ready" .-> A1

    A1 -- "results + branch" --> Lead
    A2 -- "results + branch" --> Lead
    A3 -- "results" --> Lead

    Lead -- "merge + summary" --> User

    Health[⏰ Health Monitor<br/><i>every 3 min</i>]
    Health -. "check status" .-> A1
    Health -. "check status" .-> A2
    Health -. "check status" .-> A3

    Progress[(📁 Progress<br/>progress.md<br/>history.log)]
    Lead -. "checkpoint" .-> Progress

    style User fill:#f0f4ff,stroke:#4a6fa5
    style Lead fill:#fff3e0,stroke:#e65100
    style A1 fill:#e8f5e9,stroke:#2e7d32
    style A2 fill:#e8f5e9,stroke:#2e7d32
    style A3 fill:#e8f5e9,stroke:#2e7d32
    style Health fill:#fce4ec,stroke:#c62828
    style Progress fill:#f3e5f5,stroke:#6a1b9a
```

## Subagent Mode vs Team Mode

<table>
<tr><td>

**Subagent Mode** — lightweight, spawn & collect

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant L as 🎯 Lead
    participant A as 🤖 Agent A
    participant B as 🤖 Agent B

    U->>L: "do X and Y in parallel"
    L->>U: Show plan, ask permission
    U->>L: ✅ Approved

    par spawn all at once
        L->>A: Agent(prompt, worktree)
        L->>B: Agent(prompt, worktree)
    end

    Note over A: working independently...
    Note over B: working independently...

    A-->>L: ✅ done + branch
    B-->>L: ✅ done + branch

    L->>L: merge branches
    L->>U: summary + merged result
```

</td><td>

**Team Mode** — full orchestration

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant L as 🎯 Lead
    participant A as 🤖 Agent A
    participant B as 🤖 Agent B

    U->>L: "build feature X"
    L->>L: TeamCreate + TaskCreate
    L->>U: Show task table
    U->>L: ✅ Approved

    L->>A: spawn (task 1)
    L->>B: spawn (task 2)

    A->>B: SendMessage: API endpoint
    B->>A: SendMessage: interface ready

    Note over L: ⏰ health check every 3min

    A-->>L: task 1 done
    L->>L: checkpoint progress
    B-->>L: task 2 done

    L->>U: summary + merge checklist
```

</td></tr>
</table>

## Error Recovery Flow

```mermaid
flowchart LR
    Stuck{{"🔍 Agent stuck?"}}
    Check["TaskOutput<br/>(block=false)"]
    Alive{Alive?}
    Msg["SendMessage<br/>2min deadline"]
    Response{Responded?}
    Resume["Agent(resume=id)"]
    New["New agent<br/>+ failure context"]
    Ask["Ask user<br/>for help"]
    OK(["✅ Recovered"])

    Stuck --> Check --> Alive
    Alive -- yes --> Msg --> Response
    Alive -- no --> Resume
    Response -- yes --> OK
    Response -- no --> Resume
    Resume -- success --> OK
    Resume -- fail --> New
    New -- success --> OK
    New -- fail --> Ask

    style Stuck fill:#fff3e0,stroke:#e65100
    style OK fill:#e8f5e9,stroke:#2e7d32
    style Ask fill:#fce4ec,stroke:#c62828
```

## Features

- **Two execution modes** — choose the right level of orchestration for your task
- **Worktree isolation** — each agent gets its own copy of the repo, including multi-repo support
- **Health monitoring** — periodic cron-based checks + dead agent detection
- **Progress checkpointing** — persistent state that survives crashes and restarts
- **Auto error recovery** — resume → retry with context → escalate to user
- **Peer messaging** — agents exchange data directly, decisions go through the lead
- **Ghost agent prevention** — mandatory cleanup before spawning replacements
- **Display modes** — split-pane (default with tmux) or in-process, with direct teammate interaction via keyboard shortcuts
- **Quality gate hooks** — enforce linting, tests, or coverage checks when teammates finish tasks
- **Self-claiming tasks** — agents autonomously pick up the next matching task after completing their assignment

## Relationship to Official Documentation

This skill is an implementation of the [official Claude Code Agent Teams documentation](https://code.claude.com/docs/en/agent-teams). We follow the official guidance as the foundation, and supplement it with practical patterns learned from real-world usage.

### What We Implemented from the Official Docs

| Feature | Description |
|---------|-------------|
| Team & Task API | `TeamCreate`, `TaskCreate`, `TaskUpdate`, `TaskList` for structured orchestration |
| Agent spawning | `Agent()` with `team_name`, `isolation: "worktree"`, `mode`, `run_in_background` |
| Inter-agent messaging | `SendMessage` for direct teammate communication |
| Display modes | `teammateMode`: split-pane (default with tmux) vs in-process |
| Direct teammate interaction | Keyboard shortcuts (Shift+Down, Enter, Escape, Ctrl+T) in split-pane mode |
| Self-claiming tasks | Agents pick up next unassigned, unblocked task after completing their assignment |
| Quality gate hooks | `TeammateIdle` / `TaskCompleted` hooks to enforce tests and linting |
| Known limitations | 8 official limitations documented (agent count, context windows, ephemeral state, etc.) |
| Feature flag | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable |

### What We Added in Practice

The official docs describe the available tools and APIs. In practice, we found some additional patterns helpful for running stable, predictable agent teams:

#### Workflow & Architecture

| Practice | What it supplements |
|----------|---------------------|
| **Subagent Mode** | A lightweight alternative for simple parallel tasks — no TeamCreate/TaskCreate overhead. Complements the official Team mode. |
| **T0–T8 step-by-step workflow** | A sequenced playbook built on the official APIs: preflight → create team → spawn → environment → monitor → recover → checkpoint → summarize → shutdown. |
| **User permission rule** | Adds a consent step: ask the user before spawning or shutting down any agent. |
| **Permission mode guide** | Compares `auto`/`plan`/`dontAsk`/`default`/`bypassPermissions` with practical recommendations. |

#### Reliability & Recovery

| Practice | What it supplements |
|----------|---------------------|
| **Auto error recovery** | 3-tier escalation: `Agent(resume=id)` → new agent with failure context → ask user. |
| **Ghost agent prevention** | 5-step cleanup before spawning replacements — prevents duplicate agents on the same task. |
| **"Check alive FIRST" protocol** | Always `TaskOutput(block=false)` before `SendMessage` — avoids sending messages to dead agents. |
| **Health monitoring** | `CronCreate` periodic checks (every 3 min) + stuck agent detection criteria, built on `TaskOutput`. |
| **Progress checkpointing** | Persistent `.claude/teams/{name}/` directory (`progress.md`, `history.log`, `results/`) for crash recovery. |
| **Structured shutdown** | Normal and emergency shutdown protocols with checkpoint-before-close guarantees. |

#### Agent Coordination

| Practice | What it supplements |
|----------|---------------------|
| **Role-constrained self-claim** | Adds role filtering and file locking on top of the official self-claim mechanism. |
| **Peer messaging rules** | Practical guidelines for `SendMessage`: factual data → direct to peer; scope changes → through lead. |
| **Multi-repo worktree isolation** | Extends `isolation: "worktree"` with explicit `git worktree` setup for additional repos. |
| **Environment setup checklist** | Concrete checklist for child agent env setup: venv, env vars, PATH, working dir, build commands, system tools. |

#### Developer Experience

| Practice | What it supplements |
|----------|---------------------|
| **Bilingual prompt template** | Ready-to-use EN+CN template with environment setup, teammate info, escalation guidance, and "If You Get Stuck" section. |
| **Troubleshooting guide** | 13 solved problems with root cause analysis, bilingual (EN + CN). |

## Modes

| | Subagent Mode | Team Mode |
|---|---|---|
| **Best for** | Independent parallel tasks — research, parallel bug fixes, "do A and B simultaneously" | Complex coordination — task dependencies, agents sharing interfaces, long-running projects |
| **Overhead** | Low — spawn and collect | High — task tracking, cron monitoring, progress files |
| **Communication** | Agents return results to lead only | Agents message each other + lead |
| **Monitoring** | Automatic notifications when done | CronCreate health checks + TaskOutput + SendMessage |

**Default to Subagent mode** unless the task clearly needs inter-agent coordination.

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/dirkxie/agent-team-up.git ~/.claude/skills/team-up
```

Or copy the files manually into `~/.claude/skills/team-up/`.

## Usage

Invoke manually:

```
/team-up my-project
```

Or Claude will automatically invoke it when you mention teams, agents, swarms, parallel work, or multi-agent coordination.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main instructions (loaded into context when skill is invoked) |
| `prompt-template.md` | Ready-to-use template for agent prompts with escalation guidance |
| `TROUBLESHOOTING.md` | Common problems and solutions (English + 中文) |

## Quick Start

### Subagent Mode (lightweight)

```python
# 1. Spawn all agents in one message
Agent(name="agent-a", prompt="...", isolation="worktree",
      mode="auto", run_in_background=True)
Agent(name="agent-b", prompt="...",
      mode="auto", run_in_background=True)

# 2. Wait — you'll be notified as each finishes

# 3. Synthesize results + merge branches
# git merge <branch> && git worktree remove <path>
```

### Team Mode (full orchestration)

```python
# 1. Create team + progress directory
TeamCreate(team_name="proj-x", description="...")
# mkdir -p .claude/teams/proj-x/results

# 2. Create tasks with dependencies
TaskCreate(subject="Module A", description="...")  # → id: 1
TaskCreate(subject="Module B", description="...")  # → id: 2
TaskUpdate(taskId="3", addBlockedBy=["1", "2"])

# 3. Assign BEFORE spawning
TaskUpdate(taskId="1", owner="agent-a")

# 4. Spawn agents (after user approval)
Agent(name="agent-a", team_name="proj-x", isolation="worktree",
      mode="auto", run_in_background=True, prompt="...task ID: 1...")

# 5. Monitor with CronCreate health checks

# 6. When done: checkpoint → summary → shutdown → cleanup
```

## Key Principles

- **User permission required** — never spawn or shut down agents without explicit approval
- **Pre-assign initial tasks before spawning** — prevents race conditions on startup
- **Environment setup is explicit** — child agents don't inherit the parent's environment
- **Agents self-claim within role** — finished agents pick up the next unassigned, unblocked task matching their role; if none remain, they report to lead and stop
- **Quality gates via hooks** — enforce automated checks (tests, linting) when teammates complete tasks
- **Check alive before messaging** — don't send messages to dead agents
- **Known limitations** — be aware of agent count limits, per-agent context windows, and ephemeral team state

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for 13 solved problems with detailed root cause analysis and fixes (English + 中文).

---

# agent-team-up（中文）

<p align="center">
  <a href="#agent-team-up">📖 English Docs / 英文说明</a>
</p>

一个用于启动和协调多 agent 团队并行工作的 skill。当前适配 [Claude Code](https://claude.ai/code)，未来计划支持更多 agent 平台。

## 功能特性

- **双模式执行** — 根据任务复杂度选择轻量或完整编排模式
- **Worktree 隔离** — 每个 agent 获得独立的代码副本，支持多 repo 隔离
- **健康监控** — 基于 cron 的定期检查 + 死亡 agent 检测
- **进度持久化** — 崩溃重启后不丢失进度
- **自动错误恢复** — resume → 带上下文重试 → 升级到用户
- **Peer 通讯** — agent 之间直接交换数据，决策通过 lead
- **幽灵 agent 防护** — 替换前强制清理旧 agent
- **显示模式** — 分屏（tmux 下默认）或进程内，支持键盘快捷键直接与队友交互
- **质量门禁** — 通过 hooks 在队友完成任务时自动执行 lint、测试或覆盖率检查
- **任务自行领取** — agent 完成当前任务后自动领取下一个匹配其角色的可用任务

## 与官方文档的关系

本 skill 是 [Claude Code Agent Teams 官方文档](https://code.claude.com/docs/en/agent-teams) 的落地实现。我们以官方指导意见为基础，补充了实际使用中积累的实践经验。

### 已实现的官方功能

| 功能 | 说明 |
|------|------|
| Team & Task API | `TeamCreate`、`TaskCreate`、`TaskUpdate`、`TaskList` 结构化编排 |
| Agent 启动 | `Agent()` 支持 `team_name`、`isolation: "worktree"`、`mode`、`run_in_background` |
| Agent 间通讯 | `SendMessage` 直接队友通讯 |
| 显示模式 | `teammateMode`：分屏（tmux 下默认）vs 进程内 |
| 直接与队友交互 | 分屏模式下键盘快捷键（Shift+Down、Enter、Escape、Ctrl+T） |
| 任务自行领取 | Agent 完成任务后自动领取下一个匹配的未分配任务 |
| 质量门禁 | `TeammateIdle` / `TaskCompleted` hooks 自动执行测试和 lint |
| 已知限制 | 8 条官方限制（agent 数量、上下文窗口、临时状态等） |
| 功能开关 | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 环境变量 |

### 我们在实践中补充的内容

官方文档描述了可用的工具和 API。在实际使用中，我们发现以下模式有助于运行稳定、可预期的 agent 团队：

#### 工作流与架构

| 实践 | 补充了什么 |
|------|-----------|
| **Subagent 模式** | 简单并行任务的轻量替代方案——无需 TeamCreate/TaskCreate 开销。与官方 Team 模式互补。 |
| **T0–T8 分步工作流** | 基于官方 API 构建的完整执行剧本：预检 → 建团队 → 启动 → 环境 → 监控 → 恢复 → 存档 → 汇总 → 关闭。 |
| **用户许可规则** | 增加了确认环节：启动或关闭任何 agent 前征得用户同意。 |
| **权限模式指南** | 对比 `auto`/`plan`/`dontAsk`/`default`/`bypassPermissions` 并给出实用建议。 |

#### 可靠性与故障恢复

| 实践 | 补充了什么 |
|------|-----------|
| **自动错误恢复** | 3 级升级：`Agent(resume=id)` → 新 agent + 失败上下文 → 问用户。 |
| **幽灵 agent 防护** | 替换前强制 5 步清理——防止同一任务出现两个 agent。 |
| **"先检查存活"协议** | 始终先 `TaskOutput(block=false)` 再 `SendMessage`——避免给死掉的 agent 发消息。 |
| **健康监控** | 基于 `TaskOutput` 构建 `CronCreate` 每 3 分钟检查 + 卡住判定标准。 |
| **进度持久化** | `.claude/teams/{name}/` 目录（`progress.md`、`history.log`、`results/`）用于崩溃恢复。 |
| **结构化关闭协议** | 正常和紧急关闭协议，保证关闭前先存档。 |

#### Agent 协调

| 实践 | 补充了什么 |
|------|-----------|
| **角色约束的自行领取** | 在官方自行领取机制基础上增加角色过滤和文件锁。 |
| **Peer 通讯规则** | `SendMessage` 使用指南：事实性数据 → 直接发队友；范围变更 → 通过 lead。 |
| **多 repo worktree 隔离** | 扩展 `isolation: "worktree"`，为额外 repo 添加显式 `git worktree` 配置。 |
| **环境配置清单** | 子 agent 环境配置的具体清单：venv、环境变量、PATH、工作目录、构建命令、系统工具。 |

#### 开发者体验

| 实践 | 补充了什么 |
|------|-----------|
| **双语 prompt 模板** | 现成的中英文模板，包含环境配置、队友信息、升级指引和"卡住时怎么办"段落。 |
| **问题排查指南** | 13 个已解决问题，含根因分析，中英双语。 |

## 使用场景

- 需要多个 agent 同时处理一个任务的不同部分
- 任务过于庞大，单个 agent 难以完成，需要并行执行
- 需要结构化协调：任务依赖、worktree 隔离、权限管理
- Agent 需要跨多个 repo 修改文件
- 需要韧性保障：自动检测卡住的 agent、故障恢复、进度持久化

## 安装

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/dirkxie/agent-team-up.git ~/.claude/skills/team-up
```

或手动复制文件到 `~/.claude/skills/team-up/`。

## 调用方式

手动调用：
```
/team-up my-project
```

或者当你要求 Claude 启动 agent 团队、创建集群、协调并行任务时，Claude 会自动调用此 skill。

## 文件说明

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 主指令文件（skill 被调用时加载到上下文） |
| `prompt-template.md` | Agent prompt 模板，包含 "卡住时上报" 指引 |
| `TROUBLESHOOTING.md` | 常见问题与解决方案（英文 + 中文） |

## 问题排查

参见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)，包含 13 个已解决问题的详细根因分析和修复方案。

## License

[MIT](LICENSE)
