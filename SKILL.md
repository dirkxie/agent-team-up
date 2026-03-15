---
name: team-up
description: Spawn and coordinate multi-agent teams for parallel work. Make sure to use this skill whenever the user mentions teams, agents, swarms, parallel work, multi-agent, coordinating tasks, spawning workers, or wants multiple agents working together — even if they don't explicitly say "team". Also use when assigning tasks across agents, troubleshooting stuck or idle agents, shutting down teams, or managing any form of agent collaboration. If the task is complex enough to benefit from parallel execution by multiple agents, proactively suggest using this skill.
argument-hint: [team-name]
allowed-tools: Agent, SendMessage, TeamCreate, TeamDelete, TaskCreate, TaskUpdate, TaskList, TaskGet, Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, CronCreate, CronDelete
---

# Agent Team Management

> **Rule: User Permission Required.** Before starting or closing any member agent, the team lead MUST ask the user for permission first. Never spawn or shut down agents without explicit user approval.

## Step 0: Choose Mode

Two execution modes are available. Ask the user which one to use with `AskUserQuestion`:

```
"How should we organize the parallel work?"
Options:
- "Subagent mode (Recommended for most tasks)" — lightweight, spawn agents and collect results
- "Team mode" — full orchestration with task tracking, inter-agent messaging, and health monitoring
```

| | Subagent Mode | Team Mode |
|---|---|---|
| **Best for** | Independent parallel tasks — research, parallel bug fixes, "do A and B simultaneously" | Complex coordination — task dependencies, agents sharing interfaces, long-running projects |
| **Overhead** | Low — spawn and collect | High — TeamCreate, TaskCreate, cron monitoring, progress files |
| **Communication** | Agents return results to lead only | Agents message each other + lead via SendMessage |
| **Task mgmt** | None — task is in the prompt | Full — TaskCreate with dependencies, TaskUpdate for status |
| **Monitoring** | Automatic notifications when done | CronCreate health checks + TaskOutput + SendMessage |
| **Progress** | Not needed (short-lived) | `.claude/teams/{name}/` with checkpoints |
| **Requires** | Nothing extra | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` flag |

**Default to Subagent mode** unless the task clearly needs inter-agent coordination or has complex dependencies.

---

# Subagent Mode

Lightweight parallel execution. No TeamCreate, no TaskCreate — just spawn agents, wait for results, merge.

## S1: Plan the Work Split

Break the task into independent chunks. Present the plan to the user:

```
| # | Agent | Task | Isolation |
|---|-------|------|-----------|
| 1 | researcher-a | Research approach X | none |
| 2 | implementer-a | Implement module A | worktree |
| 3 | implementer-b | Implement module B | worktree |
```

Key principles:
- Each chunk must be **independently completable** — no agent should block another
- Use `isolation: "worktree"` for any agent that modifies code
- Include clear acceptance criteria in each task description

Get user confirmation before spawning.

## S2: Choose Permission Mode

Ask the user what autonomy level agents should have:

```
"How much autonomy should the agents have?"
Options:
- "Auto mode (Recommended)" — agents work independently, minimal interruptions
- "Plan mode" — agents must get approval before writing code
- "Don't Ask mode" — fastest, agents run everything without asking
```

See [Permission Mode Reference](#permission-mode-reference) for details.

## S3: Spawn Agents

Spawn all agents in a **single message** with `run_in_background: true`. Each agent gets a self-contained prompt — task, environment setup, working directory, acceptance criteria. Agents cannot message each other in this mode, so the prompt must contain everything.

```python
# Spawn all agents in parallel (single message, multiple Agent calls)
Agent(
    name="researcher-a",
    description="Research approach X",
    prompt="...complete task description with env setup...",
    mode="auto",
    run_in_background=True
)
Agent(
    name="implementer-a",
    description="Implement module A",
    prompt="...complete task description with env setup...",
    isolation="worktree",
    mode="auto",
    run_in_background=True
)
```

Key differences from Team mode:
- **No `team_name`** parameter — agents are standalone subprocesses
- **No TaskCreate/TaskUpdate** — the task is fully described in the prompt
- **No SendMessage** — agents cannot message each other or the lead
- Agents return their result automatically; you get notified when each finishes

**Environment setup still matters** — child agents do NOT inherit the parent's environment. Include all setup (venv, env vars, PATH, working directory) in each agent's prompt. See [prompt-template.md](prompt-template.md) for reference.

## S4: Collect Results

You're notified as each agent completes. Each notification includes the agent's output. If an agent used `isolation: "worktree"`, the notification includes the worktree path and branch.

**If an agent fails**: Retry with `Agent(resume="<agent_id>")` or spawn a fresh one with adjusted instructions. No complex recovery needed.

## S5: Merge & Summarize

After all agents finish:

1. **Synthesize results** — combine findings, resolve conflicts between agents' outputs
2. **Merge worktree branches** (if agents modified code):
   ```bash
   git diff main...<agent-branch>   # review changes first
   git merge <agent-branch>          # merge after user approval
   git worktree remove <path>        # clean up
   ```
3. **Report to user** — summarize what each agent accomplished, present merged results

### Subagent Quick Reference

```python
# 1. Spawn all agents in one message (no team_name needed)
Agent(name="agent-a", prompt="...", isolation="worktree",
      mode="auto", run_in_background=True)
Agent(name="agent-b", prompt="...",
      mode="auto", run_in_background=True)

# 2. Wait — you'll be notified as each finishes

# 3. Synthesize results + merge branches if needed
# git merge <branch> && git worktree remove <path>
```

---

# Team Mode

Full orchestration for complex, coordinated work. Use when agents need to communicate, tasks have dependencies, or the work is long-running.

## T0: Preflight — tmux & Feature Flag

### tmux Check

Team mode sessions are long-running. tmux serves two purposes: **session persistence** (protects against SSH timeout, laptop sleep) and **split-pane display** (see [Display Mode](#display-mode-teammatemode) below). Check if inside tmux first:

```bash
echo $TMUX
```

- **If `$TMUX` is set** → proceed. Split-pane display is available.
- **If empty** → warn the user: tmux protects against terminal disconnects and enables split-pane teammate display. Suggest starting tmux first, but let the user choose to continue without it (in-process display still works).

### Feature Flag

Check if the agent teams flag is enabled:

```bash
cat ~/.claude/settings.json | grep -o "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS"
```

If not found, guide the user to add it to `~/.claude/settings.json` under `"env"`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Then restart Claude Code.

### Display Mode (teammateMode)

How teammates appear on screen is controlled by the **display mode** (`teammateMode`), separate from how they are launched:

| Display Mode | How teammates appear | Requires tmux? |
|---|---|---|
| **Split-pane** (default when tmux detected) | Each teammate gets its own tmux pane. You can see all agents working simultaneously. | Yes |
| **In-process** (default when no tmux) | Teammates run inside the lead's process. Output interleaved in the same terminal. | No |

The skill auto-selects: if `$TMUX` is set, split-pane is used; otherwise, in-process. To override, configure in `~/.claude/settings.json`:

```json
{
  "teammateMode": "in-process"
}
```

Or pass via CLI flag: `claude --teammate-mode in-process`

**Important distinction**: The `Agent()` tool **launches** teammates. `teammateMode` only controls **display**. You always launch via `Agent()` — never via tmux commands.

#### Direct Teammate Interaction (split-pane mode)

In split-pane mode, you can interact directly with teammates in their panes:

| Shortcut | Action |
|----------|--------|
| `Shift+Down` | Focus the next teammate pane |
| `Enter` | Start typing in the focused teammate's pane |
| `Escape` | Return focus to the lead pane |
| `Ctrl+T` | Toggle teammate pane visibility |

You can also click directly on a teammate's tmux pane to interact with it.

## T1: Create Team & Tasks

**Order matters: team first, then tasks, then agents.**

```
1. TeamCreate  →  Initialize team and task list
2. mkdir -p .claude/teams/{team_name}/results  →  Create progress directory
3. TaskCreate  →  Break down work, set dependencies (addBlockedBy)
4. TaskUpdate  →  (Optional) Pre-assign task owners
5. Agent spawn →  Start agents (each prompt includes its task ID)
```

**If resuming a previous team**: Check `.claude/teams/{team_name}/progress.md` first — skip completed work. See [T6](#t6-progress-checkpointing) for details.

Task breakdown principles:
- Each task should be independently completable, moderate granularity
- Use `addBlockedBy` to set dependency order and avoid conflicts
- Write clear acceptance criteria in each task description
- Include relevant file paths and constraints — agents lose parent context

**After creating all tasks, display the full task list to the user:**

```
| ID | Task | Assignee | Status | Blocked By |
|----|------|----------|--------|------------|
| 1  | Implement REST API | agent-a | pending | — |
| 2  | Build frontend | agent-b | pending | — |
| 3  | Integration test | — | pending | 1, 2 |
```

Get user confirmation before spawning any agents.

### Task Claiming

The lead can pre-assign tasks before spawning, but agents also **self-claim** tasks after completing their initial assignment:

- **Lead pre-assigns**: Use `TaskUpdate(owner=...)` for initial tasks before spawning — prevents the startup race condition where an agent can't find its task.
- **Agent self-claims**: After completing a task, agents pick up the next unassigned, unblocked task that matches their role. Agents use file locking (`/tmp/.task-claim-{taskId}.lock`) to prevent two agents from claiming the same task.
- **Role constraint**: Agents only self-claim tasks within their expertise (e.g., a frontend agent won't claim a backend task). If no matching tasks remain, the agent notifies the lead and waits.

## T2: Spawn Agents

**Pre-assign initial tasks before spawning agents.** Call `TaskUpdate(taskId=..., owner="agent-name")` for the first task per agent, then spawn. This prevents a race condition where the agent starts but can't find its task. Always include the assigned task ID in the agent's prompt. After completing their initial task, agents self-claim additional matching tasks.

**Ask the user before spawning.** Confirm which agents to start, their permission mode ([see reference](#permission-mode-reference)), and configuration.

### Use the Agent tool, NOT tmux

**Do NOT use tmux to launch agents.** Known issues: long command truncation, silent pane exits, inability to pass permission modes.

```json
{
  "description": "Implement XX feature",
  "prompt": "...(detailed task description)...",
  "subagent_type": "general-purpose",
  "isolation": "worktree",
  "mode": "auto",
  "run_in_background": true,
  "team_name": "my-team",
  "name": "feature-agent"
}
```

| Parameter | When to use |
|-----------|-------------|
| `isolation: "worktree"` | **Required when modifying code** — gives each agent an isolated repo copy |
| `run_in_background: true` | When running multiple agents in parallel |
| `team_name` | **Required in Team mode** — connects agent to the team for SendMessage/TaskList |

### Multi-Repo Worktree Isolation

`isolation: "worktree"` only creates a worktree for the repo where Claude Code was started. If an agent touches files in **other repos**, those edits happen on the live branch with no isolation.

**Rule: Every repo an agent modifies MUST have its own worktree.** Include `git worktree` setup commands in the agent's prompt for all additional repos:

```bash
BRANCH="feat/${AGENT_NAME}-$(date +%s)"
git -C /path/to/other-repo worktree add /tmp/${AGENT_NAME}-other-repo -b "$BRANCH"
cd /tmp/${AGENT_NAME}-other-repo   # NOT the original repo path
```

Prompt checklist:
1. List all repos the agent will modify
2. Create a worktree per repo with a unique branch name
3. Redirect all paths to worktree paths
4. Include teammate info — name and what they're working on, so agents can send relevant info directly

## T3: Environment Setup

**Child processes do NOT inherit the parent's environment (venv, env vars, PATH, aliases, etc.).**

Include **all** environment setup in each agent's prompt. Read the project's `CLAUDE.md` for required commands. See [prompt-template.md](prompt-template.md) for a ready-to-use template.

Checklist:
- Language runtimes: `source .venv/bin/activate`, `nvm use`, `conda activate`, etc.
- Environment variables: all `export` commands
- PATH additions: tool-specific paths
- Working directory: `cd` to the correct path (worktree path if isolated)
- Build/run commands: project-specific test examples
- System dependencies: tools to verify (`node`, `go`, `rustc`, etc.)

## T4: Monitoring & Health Checks

Agents can silently die or get stuck. Actively monitor.

### Periodic Health Check

Set up a recurring check after spawning agents:

```
CronCreate(
  cron="*/3 * * * *",
  prompt="Check team status: run TaskList, identify tasks stuck in_progress too long, ping silent agents via SendMessage.",
  recurring=true
)
```

Delete the cron job when done (`CronDelete`).

### Manual Health Check

1. **TaskList** — see all task statuses at a glance
2. **TaskOutput(task_id=<agent_id>, block=false)** — check if alive without blocking
   - Returns output → agent finished or is idle
   - Timeout → agent is still working
3. **SendMessage** to a specific agent — alive agents will respond
4. **Direct interaction** (split-pane mode only) — use `Shift+Down` to focus a teammate's pane, or click on it directly. Useful for quick checks or manual intervention.

### Detecting Stuck Agents

An agent is likely stuck if:
- Task has been `in_progress` for a long time with no messages
- `TaskOutput(block=false)` shows agent completed but task isn't marked done
- No response to SendMessage after a reasonable wait

When detected → proceed to Error Recovery (T5).

### Quality Gates with Hooks

Use Claude Code hooks to enforce quality checks when teammates finish tasks or go idle:

| Hook Event | When it fires | Use case |
|---|---|---|
| `TeammateIdle` | A teammate finishes its current turn and goes idle | Run linting, type checks, or test suites against the teammate's changes |
| `TaskCompleted` | A task is marked as completed via `TaskUpdate(status="completed")` | Run integration tests, coverage checks, or notify reviewers |

Example hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "TeammateIdle": {
      "command": "cd $WORKTREE_PATH && npm test -- --bail",
      "onFailure": "block"
    }
  }
}
```

When `onFailure` is `"block"`, the teammate cannot proceed until the quality gate passes. This prevents broken code from propagating to dependent tasks.

## T5: Error Recovery

**The lead handles recovery directly. Do NOT forward the user's request to a stuck agent** — a stuck or dead agent will never respond, and you'll be stuck forever waiting.

### Check if Alive FIRST

1. `TaskOutput(task_id=<agent_id>, block=false)` — check if alive
   - Returns final output → agent is **dead**. Go to step 3.
   - Timeout / no output → agent is **alive but stuck**. Go to step 2.

2. **Agent is alive** — send ONE message with ~2 minute deadline. No response → treat as dead.

3. **Agent is dead/unresponsive** — clean up first, then replace. Never start a replacement while the old one lingers — ghost agents waste the user's attention.

### Cleanup Before Replacement (mandatory)

```
1. Save old agent's work    →  check worktree for commits, capture branch/hash
2. Close old agent           →  shutdown_request if alive, log if dead
3. Update progress files     →  progress.md + history.log with failure details
4. Handle worktree           →  keep if useful commits, remove otherwise
5. THEN start replacement    →  resume or new agent
```

### Auto-Recovery Escalation

```
Resume (cheapest) → New agent + context → Ask user
```

1. **Resume**: `Agent(resume="<agent_id>", run_in_background=true, team_name="...")`
2. **New agent**: Include progress context, failure details, branch to continue from
3. **Ask user**: After two failed attempts — root cause may be external

## T6: Progress Checkpointing

Agent contexts are ephemeral. A persistent progress directory lets replacement agents pick up where things stopped.

### Team Progress Directory

```
.claude/teams/{team_name}/
├── progress.md          # Latest checkpoint (overwritten each update)
├── history.log          # Append-only timeline
└── results/             # Key output artifacts
```

Create this when the team is created (T1).

### When to Update

- A task is completed
- A major milestone is reached
- An agent dies and cannot be recovered
- User asks to pause/stop
- Before shutting down

### progress.md Format

```markdown
# Team Progress: {team_name}
**Date**: {date}  **Version**: {version}  **Status**: {running|paused|failed}

## Completed Tasks
- [x] Task 1: {subject} — {key result or commit hash}

## In-Progress Tasks
- [ ] Task 3: {subject} — assigned to {agent}, status: {what they were doing}

## Key Results
- {Performance numbers, test results, commit hashes}

## Agent State
| Agent | Status | Branch | Last Activity |
|-------|--------|--------|---------------|
| agent-a | completed | feat/agent-a-xxx | committed changes |
```

### history.log

Append-only. Each entry timestamped. New teams read this to avoid repeating mistakes.

### Resuming a Team

1. Check `.claude/teams/{team_name}/` for existing progress
2. Read `progress.md` + `history.log` + `results/`
3. Skip completed tasks — only create tasks for unfinished work
4. Include context in new agent prompts: what's done, what branch to continue from

## T7: Completion & Summary

When all tasks are complete:

```markdown
# Team Summary: {team_name}
**Date**: {date}  **Duration**: {time}

## What Was Done
- {Accomplishments}

## Changes Made
| Agent | Branch | Key Changes |
|-------|--------|-------------|
| agent-a | feat/... | Added feature X |

## Merge Checklist
- [ ] Review agent-a branch: `git diff main...feat/agent-a-xxx`
- [ ] Run full regression after merge
```

## T8: Shutdown

**Ask the user before closing agents.**

### Normal Shutdown

```
1. Summary report (T7) → 2. Save checkpoint (T6) → 3. AskUserQuestion confirm
4. SendMessage(type: "shutdown_request") to each agent, ONE BY ONE
5. Wait 60s per agent → 6. CronDelete → 7. TeamDelete
```

Dead agents during shutdown: check `TaskOutput(block=false)`, log and move on. Clean up worktrees: `git worktree remove <path>`.

### Emergency Shutdown

1. Checkpoint first (T6)
2. Inform user
3. Graceful shutdown of surviving agents
4. `CronDelete`, `TeamDelete`

---

## Permission Mode Reference

| Mode | Interruptions | Best for |
|------|--------------|----------|
| `"auto"` | **Low** — autonomous, asks only for unusual ops | Well-defined tasks. **Recommended default.** |
| `"plan"` | **High** — must get approval before coding | Risky or ambiguous tasks |
| `"dontAsk"` | **None** — runs everything freely | Trusted tasks in isolated worktrees |
| `"default"` | **Medium** — asks for writes and bash | Case-by-case control |
| `"bypassPermissions"` | **None** — bypasses all checks | Read-only research tasks |

`plan` mode creates a bottleneck with multiple agents. `auto` is the sweet spot. Use `dontAsk` + `isolation: "worktree"` for maximum speed with safety (worktree changes can be discarded).

## Non-Obvious Gotchas

- **Idle is normal.** Agents go idle after every turn. Idle ≠ stuck. Send a message to wake them. *(Team mode only)*
- **Plain text is invisible to teammates.** You MUST use SendMessage. *(Team mode only)*
- **Do not broadcast casually.** `type: "broadcast"` sends N messages. Default to specific recipients. *(Team mode only)*
- **Include context in replies.** Teammates may have lost context due to compression. Repeat key details.
- **Agents must stay in scope.** Architect shouldn't code; frontend agent shouldn't fix backend. Finished → self-claim next task within role, or report to lead and stop if none match.
- **Agents don't share memory.** Each has its own context window. In Team mode, agents can message each other for factual handoffs. In Subagent mode, only the lead sees all results.
- **Peer messaging is for data, not decisions.** "The API endpoint is /api/v1/users" → direct. Scope changes → through the lead. *(Team mode only)*
- **Ask teammates before searching.** A quick SendMessage beats a 5-minute codebase search. *(Team mode only)*
- **Worktree cleanup matters.** Dead agents leave worktrees. `git worktree list` → `git worktree remove <path>`.

## Known Limitations

These are official limitations of the agent teams feature:

1. **No shared filesystem awareness** — teammates don't see each other's uncommitted changes, even in the same worktree. Coordinate via commits and messages.
2. **Context window is per-agent** — each teammate has its own context window. Long-running agents will hit compression. Critical info should be in files, not just in conversation.
3. **No automatic conflict resolution** — if two agents modify the same file in different worktrees, merge conflicts must be resolved manually by the lead.
4. **Agent count limits** — performance degrades beyond ~5-7 concurrent agents. The lead's context fills with status messages and the host machine's resources are shared.
5. **No persistent agent identity** — a resumed agent gets its old context, but a replacement agent is a fresh process. Include all necessary context in replacement prompts.
6. **Hooks are per-session** — hooks configured for the lead's session don't automatically apply to teammates unless they're in the shared settings file.
7. **Split-pane display requires tmux** — in-process mode works without tmux, but you lose the ability to visually monitor individual agents.
8. **Team state is ephemeral** — `TeamCreate`/`TaskCreate` state lives in memory. If Claude Code restarts, the team is gone. Use progress checkpointing (T6) to survive restarts.

## Quick Reference

### Subagent Mode

```python
# 1. Spawn all agents in one message
Agent(name="agent-a", prompt="...", isolation="worktree",
      mode="auto", run_in_background=True)
Agent(name="agent-b", prompt="...",
      mode="auto", run_in_background=True)

# 2. Wait — notified as each finishes

# 3. Merge branches + summarize
# git merge <branch> && git worktree remove <path>
```

### Team Mode

```python
# 1. Create team + progress directory
TeamCreate(team_name="proj-x", description="Implement XX feature")
# mkdir -p .claude/teams/proj-x/results

# 2. Create tasks with dependencies
TaskCreate(subject="Module A", description="...")  # → id: 1
TaskCreate(subject="Module B", description="...")  # → id: 2
TaskCreate(subject="Integration test", description="...")  # → id: 3
TaskUpdate(taskId="3", addBlockedBy=["1", "2"])

# 3. Pre-assign initial tasks (agents self-claim remaining tasks after)
TaskUpdate(taskId="1", owner="agent-a")
TaskUpdate(taskId="2", owner="agent-b")

# 4. Spawn (after user approval)
Agent(name="agent-a", team_name="proj-x", isolation="worktree",
      mode="auto", run_in_background=True, prompt="...task ID: 1...")
Agent(name="agent-b", team_name="proj-x", isolation="worktree",
      mode="auto", run_in_background=True, prompt="...task ID: 2...")

# 5. Monitor
CronCreate(cron="*/3 * * * *", prompt="Check TaskList, ping silent agents")

# 6. When done: checkpoint → summary → shutdown → cleanup
```

See [prompt-template.md](prompt-template.md) for the agent prompt template.
