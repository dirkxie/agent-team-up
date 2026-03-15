# Troubleshooting / 问题排查

## Problem 1: Failed to start member agents / 启动 member agent 失败

**Status: Solved / 已解决**

The skill addresses multiple common causes of agent startup failure:

1. **Use Agent tool, not tmux** (SKILL.md Step 2): tmux has known issues — command truncation, silent pane exits, no permission mode support. The skill mandates using the `Agent()` tool with proper parameters (`isolation`, `mode`, `run_in_background`, `team_name`, `name`).

2. **Explicit environment setup** (SKILL.md Step 3): Child agents do NOT inherit the parent's environment. The skill requires including all setup (venv, env vars, PATH, working directory, system tools) in each agent's prompt. A ready-to-use [prompt-template.md](prompt-template.md) is provided.

3. **Error recovery if agent dies after start** (SKILL.md Step 5): Escalation path — `Agent(resume=<id>)` → new agent with failure context → ask user.

---

Skill 从多个层面解决了 agent 启动失败的问题：

1. **用 Agent tool 而非 tmux**（SKILL.md Step 2）：tmux 有命令截断、pane 静默退出、无法传递权限模式等已知问题。Skill 要求使用 `Agent()` 工具并配置正确参数。

2. **显式环境配置**（SKILL.md Step 3）：子 agent 不继承父进程环境。Skill 要求在每个 agent 的 prompt 中包含所有环境配置（venv、环境变量、PATH、工作目录、系统工具）。提供了 [prompt-template.md](prompt-template.md) 模板。

3. **启动后死亡的恢复机制**（SKILL.md Step 5）：升级路径 — `Agent(resume=<id>)` → 新 agent + 失败上下文 → 问用户。

---

## Problem 2: Member agents fail to display in separate tmux panes / Agent 无法在 tmux pane 中显示

**Status: Solved (bypassed) / 已解决（绕过）**

The skill no longer uses tmux at all. SKILL.md Step 2 explicitly states: *"Do NOT use tmux to launch agents."* Agents are spawned via the `Agent()` tool with `run_in_background: true`, and coordinated through `SendMessage`, `TaskList`, and `TaskOutput` — no tmux panes needed.

---

Skill 不再使用 tmux。SKILL.md Step 2 明确规定：*"不要用 tmux 启动 agent。"* Agent 通过 `Agent()` 工具以后台模式启动，通过 `SendMessage`、`TaskList`、`TaskOutput` 协调——完全不需要 tmux pane。

---

## Problem 3: Lead did not display task list with IDs to user / Lead 没有向用户展示带 ID 的任务列表

**Status: Fixed / 已修复**

Previously, the skill didn't explicitly require the lead to show the task list after creation. Tasks had internal IDs, but users couldn't see them — making progress tracking impossible.

**Fix**: SKILL.md Step 1 now requires the lead to display a full task table (ID, subject, assignee, status, dependencies) to the user after creating all tasks, and get user confirmation before spawning agents.

---

之前 skill 没有明确要求 lead 在创建任务后展示任务列表。任务有内部 ID，但用户看不到——无法跟踪进度。

**修复**：SKILL.md Step 1 现在要求 lead 在创建完所有任务后，向用户展示完整的任务表格（ID、标题、负责人、状态、依赖关系），并在启动 agent 前获得用户确认。

---

## Problem 4: Some member agents failed to get their assigned tasks / 部分 agent 启动后找不到自己的任务

**Status: Fixed / 已修复**

**Root cause**: Race condition — agents were spawned before tasks were assigned. The agent calls `TaskList` to find work, but `TaskUpdate(owner=...)` hasn't run yet, so the agent sees no tasks assigned to it.

**Fix**: SKILL.md now enforces a strict order: **assign tasks first** (`TaskUpdate(owner=...)`), **then spawn agents**. Additionally, each agent's prompt must include its specific task ID so it knows exactly what to work on without relying solely on `TaskList` discovery.

```
Before (broken):  TaskCreate → Agent spawn → TaskUpdate(owner=...)  ← race!
After  (fixed):   TaskCreate → TaskUpdate(owner=...) → Agent spawn  ← safe
```

---

**根因**：竞态条件——agent 在任务分配之前就启动了。Agent 调用 `TaskList` 查找工作，但 `TaskUpdate(owner=...)` 还没执行，所以 agent 看不到分配给自己的任务。

**修复**：SKILL.md 现在强制要求严格顺序：**先分配任务**（`TaskUpdate(owner=...)`），**再启动 agent**。同时，每个 agent 的 prompt 中必须包含具体的 task ID，不完全依赖 `TaskList` 发现。

---

## Problem 5: Member agents cannot get correct environment to code and run tests / Agent 无法获得正确的运行环境

**Status: Solved / 已解决**

**Root cause**: Child agents do NOT inherit the parent's environment — no venv, no env vars, no PATH, no aliases. Without explicit setup, agents can't find interpreters, packages, or test tools.

**Solution** (SKILL.md Step 3 + [prompt-template.md](prompt-template.md)):

The skill now dedicates an entire step to environment setup and requires the lead to:
1. Read the project's `CLAUDE.md` for all required setup commands
2. Include **every** environment command in the agent's prompt — venv activation, env vars, PATH additions, working directory, build/run commands, system tool verification
3. Use the provided prompt template which has placeholder sections for all of these

Key checklist from SKILL.md Step 3:
- Language runtimes: `source .venv/bin/activate`, `nvm use`, `conda activate`, etc.
- Environment variables: all `export` commands
- PATH additions: tool-specific paths
- Working directory: `cd` to the correct path (worktree path if isolated)
- Build/run commands: project-specific test examples
- System dependencies: tools to verify (`node`, `verilator`, etc.)

---

**根因**：子 agent 不继承父进程环境——没有 venv、环境变量、PATH、别名。没有显式配置，agent 找不到解释器、包和测试工具。

**解决方案**（SKILL.md Step 3 + [prompt-template.md](prompt-template.md)）：

Skill 专门用一个步骤处理环境配置，要求 lead：
1. 读取项目 `CLAUDE.md` 获取所有必需的配置命令
2. 在 agent prompt 中包含**所有**环境命令——venv 激活、环境变量、PATH、工作目录、构建/测试命令、系统工具验证
3. 使用提供的 prompt 模板（包含所有配置项的占位符）

---

## Problem 6: Lead does not check if member is stuck, does not inform user / Lead 不检查 agent 是否卡住，也不通知用户

**Status: Solved / 已解决**

Previously there was no monitoring mechanism — the lead just spawned agents and hoped for the best.

**Solution** (SKILL.md Step 4 + Step 5):

1. **Periodic health check** (Step 4): The lead sets up a `CronCreate` job (every 3 minutes) that runs `TaskList` to identify tasks stuck in `in_progress` for too long, and pings silent agents via `SendMessage`.

2. **Manual health check** (Step 4): The lead can check agent status at any time using:
   - `TaskList` — see all task statuses at a glance
   - `TaskOutput(task_id=<agent_id>, block=false)` — detect dead agents without blocking
   - `SendMessage` — ping a specific agent to see if it responds

3. **Stuck agent detection criteria** (Step 4): An agent is likely stuck if its task has been `in_progress` too long with no messages, or `TaskOutput` shows it completed but the task isn't marked done.

4. **Error recovery escalation** (Step 5): When a stuck agent is detected: `resume` → new agent with failure context → ask user for help.

5. **Agent-side escalation** ([prompt-template.md](prompt-template.md)): The "If You Get Stuck" section in the agent prompt template tells agents to immediately `SendMessage` the lead when blocked, rather than silently spinning.

---

之前没有监控机制——lead 启动 agent 后就放任不管。

**解决方案**（SKILL.md Step 4 + Step 5）：

1. **定期健康检查**（Step 4）：Lead 通过 `CronCreate` 设置每 3 分钟检查一次 `TaskList`，识别卡住的任务，并通过 `SendMessage` ping 沉默的 agent。

2. **手动健康检查**（Step 4）：Lead 随时可用 `TaskList`、`TaskOutput(block=false)`、`SendMessage` 检查 agent 状态。

3. **卡住判定标准**（Step 4）：任务长时间处于 `in_progress` 且无消息，或 `TaskOutput` 显示 agent 已退出但任务未完成。

4. **错误恢复升级路径**（Step 5）：检测到卡住后：`resume` → 新 agent + 失败上下文 → 问用户。

5. **Agent 端主动上报**（[prompt-template.md](prompt-template.md)）："如果卡住了" 段落要求 agent 被阻塞时立即通过 `SendMessage` 通知 lead，而非默默转圈。

---

## Problem 7: After crash/restart, new agents redo completed work / 崩溃重启后新 agent 重复已完成的工作

**Status: Fixed / 已修复**

**Root cause**: Checkpoints were saved to arbitrary user-chosen locations, and there was no standard place for new agents/teams to discover previous progress. New teams started from scratch.

**Solution** (SKILL.md Step 6 — Team Progress Directory):

Every team now gets a **persistent progress directory** at a fixed, discoverable location:

```
.claude/teams/{team_name}/
├── progress.md          # Latest state (overwritten each update)
├── history.log          # Append-only timeline of events
└── results/             # Performance data, test results, artifacts
```

Key improvements:
1. **Fixed location**: `.claude/teams/{team_name}/` — any new team or agent can find it
2. **`progress.md`**: Current state — completed tasks, pending work, agent status, key results (commit hashes, branches)
3. **`history.log`**: Append-only timeline — what was tried, what failed, why. Prevents new agents from repeating mistakes
4. **`results/`**: Persists expensive-to-produce data (perf measurements, test results)
5. **Resume instructions**: SKILL.md Step 1 and Step 6 tell the lead to check for existing progress before creating tasks, skip completed work, and include context in new agent prompts
6. **Auto-update on milestones**: Progress files are updated whenever a task completes, an agent fails, or the team pauses

---

**根因**：Checkpoint 保存到用户随意指定的位置，新 agent/团队无法发现之前的进度。新团队只能从零开始。

**解决方案**（SKILL.md Step 6 — 团队进度目录）：

每个团队在固定位置 `.claude/teams/{team_name}/` 拥有持久化进度目录：
- **`progress.md`**：最新状态——已完成任务、待处理工作、agent 状态、关键结果（commit hash、分支名）
- **`history.log`**：只追加的时间线——尝试了什么、失败了什么、为什么。防止新 agent 重复犯错
- **`results/`**：保存昂贵的数据（性能测量、测试结果）
- **恢复指引**：Step 1 和 Step 6 要求 lead 在创建任务前检查现有进度，跳过已完成工作，在新 agent prompt 中包含上下文
- **里程碑自动更新**：每次任务完成、agent 失败或团队暂停时都更新进度文件

---

## Problem 7b: Leader forwards user's command to the stuck agent instead of handling it / Leader 把用户指令转发给卡住的 agent

**Status: Fixed / 已修复**

**Root cause**: The previous error recovery flow started with `SendMessage` to the stuck agent (step 2), then checked if it was alive (step 3). The leader would send a message to a dead agent and wait forever.

**Fix** (SKILL.md Step 5):

1. **Check alive FIRST**: `TaskOutput(task_id=<agent_id>, block=false)` — determine if the agent process is alive before anything else
2. **Only message if alive**: If alive, send ONE message with a ~2 minute deadline. If no response, treat as dead
3. **Lead acts directly**: Dead/unresponsive agent → lead immediately updates progress files and performs recovery (resume → new agent → ask user)
4. **Explicit rule added**: *"Do NOT just forward the user's request to a stuck agent — a stuck or dead agent will never respond, and you'll be stuck forever waiting."*

```
Before (broken):  TaskList → SendMessage(stuck agent) → wait forever...
After  (fixed):   TaskOutput(block=false) → alive? message with deadline : restart directly
```

---

**根因**：之前的恢复流程先给卡住的 agent 发消息（步骤 2），再检查是否活着（步骤 3）。Leader 给一个死掉的 agent 发消息然后无限等待。

**修复**（SKILL.md Step 5）：
1. **先检查是否活着**：`TaskOutput(block=false)` 判断 agent 进程状态
2. **只在活着时才发消息**：设 ~2 分钟期限，无响应则视为死亡
3. **Lead 直接处理**：死亡/无响应 → lead 立即更新进度文件并执行恢复
4. **新增明确规则**：*"不要把用户的请求转发给卡住的 agent——卡住的 agent 永远不会回复"*

---

## Problem 8: Leader starts replacement without closing the stuck one — ghost agents / Leader 启动替代 agent 前没关闭旧的

**Status: Fixed / 已修复**

**Root cause**: The recovery flow jumped straight to "start new agent" without first saving the old agent's progress, closing it, or cleaning its worktree. Result: two agents for the same task.

**Fix** (SKILL.md Step 5 — "Cleanup Before Replacement"):

A mandatory 5-step cleanup sequence before any replacement:

```
1. Save old agent's work    →  check worktree for commits, capture branch/hash
2. Close old agent           →  shutdown_request if alive, log as dead if not
3. Update progress files     →  progress.md + history.log with failure details
4. Clean up old worktree     →  git worktree remove (or keep if useful commits)
5. THEN start replacement    →  resume or new agent
```

---

**根因**：恢复流程直接跳到 "启动新 agent"，没有先保存旧 agent 的进度、关闭它或清理 worktree。结果：同一任务有两个 agent。

**修复**（SKILL.md Step 5 — "替换前清理"）：启动替代 agent 前必须执行 5 步清理。

---

## Problem 9: Member agents did not use worktree isolation / Agent 没有用 worktree 隔离

**Status: Solved / 已解决**

**Root cause**: (a) agents spawned without `isolation: "worktree"` edit the live branch directly; (b) `isolation: "worktree"` only isolates the startup repo, so agents modifying multiple repos have no protection on the other repos.

**Solution** (SKILL.md Step 2):

1. **Single-repo isolation**: `isolation: "worktree"` is **required when modifying code**
2. **Multi-repo isolation**: Rule: *"Every repo an agent modifies MUST have its own worktree"* — with bash template and 4-point checklist
3. **Path redirection**: Agent must work in worktree paths, never `cd` to the original repo

---

**根因**：(a) 没有设 `isolation: "worktree"`，直接修改 live 分支；(b) 只隔离启动 repo，多 repo 时其他 repo 没有保护。

**解决方案**：单 repo 用 `isolation: "worktree"`；多 repo 每个都必须有自己的 worktree。

---

## Problem 10: Member agent goes beyond its designated task scope / Agent 超出任务范围

**Status: Fixed / 已修复**

**Root cause**: The prompt template told agents to check `TaskList` for the next task — agents would self-assign work outside their role.

**Fix**:
1. New scope rules: *"Stay within your assigned task scope. After completing, notify the lead and STOP."*
2. Workflow step 7 changed to: "STOP and wait for lead to assign your next task"
3. SKILL.md gotcha: *"Agents must stay in scope"*

---

**根因**：模板让 agent 去 `TaskList` 找下一个任务——agent 自行领取超出范围的工作。

**修复**：完成任务后通知 lead 并停下，不要自己领取新任务。

---

## Problem 12: Member agents keep asking user for permission on every command / Agent 每个命令都问用户许可

**Status: Fixed / 已修复**

**Root cause**: The skill defaulted to `plan` mode, which requires approval before coding. With multiple agents, the user is constantly interrupted.

**Fix** (SKILL.md Step 2 — Permission Mode):

1. **Ask user upfront**: Lead asks what permission level to use before spawning
2. **`auto` is the recommended default** — low interruptions, agents work independently
3. **Safety via isolation**: `dontAsk` + `isolation: "worktree"` for maximum speed with safety

---

**根因**：默认使用 `plan` 模式，多 agent 时用户被频繁打断。

**修复**：提前询问用户权限级别，推荐 `auto` 为默认。`dontAsk` + worktree 组合提供安全的最高速度。

---

## Problem 13: Member agents never talk to each other / Agent 之间不直接沟通

**Status: Fixed / 已修复**

**Root cause**: Agents only knew about the lead. All info had to go: agent-a → lead → agent-b (3 hops instead of 1).

**Fix**:
1. **Teammate section in prompt template**: Each agent gets a list of teammates
2. **Clear peer messaging rules**: Factual handoffs → direct to peer; scope changes → through lead
3. **"Ask teammates before searching"**: A 5-second message beats a 5-minute codebase search

---

**根因**：Agent 只知道 lead，不知道队友。所有信息走 3 跳而非 1 跳。

**修复**：模板新增队友段落 + peer 通讯规则。数据交接直接发给队友，任务/范围变更通过 lead。
