---
title: "Hermes Agent Kanban Swarm：从 RPC 到消息队列的多智能体协作工业化实践"
date: 2026-07-20
draft: false
tags: ["Hermes Agent", "Kanban", "Multi-Agent", "AI"]
categories: ["技术深度"]
---

# Hermes Agent Kanban Swarm：从 RPC 到消息队列的多智能体协作工业化实践

> 当你的 Agent 系统从一个智能体变成一群智能体时，最可怕的场景是什么？不是它们答错问题 —— 而是某个 worker 在凌晨三点静默崩溃，你的流水线卡住一整夜，却没有一个人类知道。

---

## 一、引言：RPC 式协作的局限性

当前 AI Agent 框架中，多智能体协作的主流范式是 **RPC 式调用**：父 Agent 通过类似 `delegate_task` 的 primitive 创建一个子进程，阻塞等待它返回结果，然后将结果塞回到父进程的上下文窗口中。这个模型在单个会话内运转良好 —— 就像一个函数调用一样清晰。

但随着生产环境的 Agent 系统扩展，RPC 模式的局限开始暴露：

1. **进程即记忆** —— 子 Agent 失败后，它携带的所有上下文和中间产物立即丢失，父 Agent 甚至不知道发生过什么。
2. **无持久化审计** —— 每次任务完成后，结果被压缩进上下文，一旦上下文被新内容覆盖，历史记录就蒸发了。
3. **人机协作的缺失** —— RPC 调用是封闭的，人类无法在子 Agent 执行过程中介入指导方向。
4. **崩溃不可恢复** —— 子进程死亡 = 任务失败，没有重试、没有接力、没有抢救。

这些问题的根因是什么？因为 **RPC 本质上是函数调用** —— 它是为同进程内、确定性、短生存期的计算设计的。而多智能体协作需要的是 **消息队列** —— 面向持久化、跨进程、可恢复、可审计的异步通信。

这就是 Hermes Agent v0.16 中 **Kanban Swarm** 要解决的问题。

---

## 二、什么是 Hermes Kanban？

Hermes Kanban 是一个**持久化任务看板**，跨所有 Hermes profile 共享。每一行任务对应 SQLite 数据库中的一行记录，每次交接（handoff）都是一条持久化的写入。它与 `delegate_task` 共享同一个 Agent 系统，但在设计哲学和行为模式上完全不同。

### 六列状态机

Kanban 的核心是一台有限状态机，任务在以下六个状态间流转：

```
                    ┌─────────┐
                    │ Triage  │  ← 原始想法、粗略点子
                    └────┬────┘
                         │ specify 或 Decompose
                    ┌────▼────┐
                    │  Todo   │  ← 已创建但等待依赖或未分配
                    └────┬────┘
                         │ 所有 parent 完成 → 自动提升
                    ┌────▼────┐
                    │  Ready  │  ← 已分配，等待 dispatcher 认领
                    └────┬────┘
                         │ dispatcher 原子 claim
                    ┌────▼──────┐
                    │  Running  │  ← worker 正在执行
                    └────┬──────┘
                ┌──────────┼──────────┐
          kanban_complete │    kanban_block
                ┌─────────┘          │
           ┌────▼────┐          ┌────▼──────┐
           │  Done   │          │  Blocked   │
           └────┬────┘          └────┬──────┘
                │ archive            │ unblock
           ┌────▼───────┐       ┌────▼────┐
           │  Archived   │       │  Ready  │
           └────────────┘       └─────────┘
```

这一设计借鉴了经典 Kanban 方法的理念，但将其扩展到了 Agent 协作的场景中。每一列的变化都有明确的含义，不是什么视觉装饰。

### 核心概念一览

| 概念 | 说明 |
|------|------|
| **Board** | 独立任务队列，拥有自己的 SQLite DB、workspace 目录和 dispatcher 循环。默认 `default`，支持多 Board 隔离不同项目。 |
| **Task** | 一行记录：标题、body、assignee（profile 名）、状态、tenant、idempotency key |
| **Link** | `task_links` 表中的 parent→child 依赖。所有 parent 完成时 dispatcher 自动将 child 从 `todo` 提升为 `ready`。 |
| **Comment** | 跨 Agent 通信协议。Agent 和人类都可追加 comment；worker 被 spawn 时会读取完整 comment 线程。 |
| **Workspace** | 三种类型：`scratch`（临时，完成后清理）、`dir:<path>`（共享目录）、`worktree`（git worktree） |
| **Dispatcher** | 后台守护循环，回收过期 claim、检测崩溃、提升 ready 任务、原子 claim 并 spawn profile |
| **Run** | 一次 attempt 的记录。任务重复尝试 3 次则有 3 行 `task_runs`。 |
| **Tenant** | 可选字符串命名空间，同一 board 内隔离不同业务的数据 |

### 存储布局

```bash
~/.hermes/kanban.db                          # SQLite 数据库
~/.hermes/kanban/workspaces/<id>/            # scratch workspace
~/.hermes/kanban/attachments/<task_id>/      # 文件附件
~/.hermes/kanban/logs/                       # worker 日志
```

多 Board 模式下，每个 board 有独立的目录树，支持项目级隔离。

---

## 三、Kanban vs delegate_task 深度对比

这是整个 Kanban 系统的设计原点。理解为什么 Kanban 和 `delegate_task` 不是替代关系，而是适用于不同场景的两个 primitive，能帮你做出正确的架构选择。

| 维度 | `delegate_task` | Kanban |
|------|----------------|--------|
| **形状** | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| **父进程行为** | 阻塞直到子进程返回 | fire-and-forget，创建后立即返回 |
| **子进程身份** | 匿名 subagent | 命名 profile，自带持久化记忆 |
| **可恢复性** | 无 —— 失败即失败 | Block → unblock → 重试；崩溃 → 回收 |
| **人机交互** | 不支持 | 随时 Comment / Unblock |
| **每任务 Agent 数** | 一次调用 = 一个 subagent | 多个 Agent 可接力（重试、审查、跟进） |
| **审计跟踪** | 上下文压缩后丢失 | SQLite 中持久化行，永久保留 |
| **协调模式** | 层级式（调用者 → 被调用者） | 对等式 —— 任何 profile 可读写任何任务 |

一句话总结：**`delegate_task` 是函数调用，Kanban 是工作队列。每次交接都是一行数据库写操作，任何 profile（或人）都能看到和编辑。**

但这不意味着 Kanban 是全能的替代品。两者有各自的适用场景：

- **用 `delegate_task` 的场景**：父 Agent 需要短期的推理答案才能继续，不涉及人类介入，结果需要返回父进程上下文。比如：在写文章之前，先让一个子 Agent 分析一下某篇参考文献的结构。
- **用 Kanban 的场景**：工作跨越 Agent 边界、需要重启后存活、可能需要人类介入、可能被不同角色接起、需要事后可发现。比如：一个博客写作流水线，从调研 → 写作 → 审校 → 发布，每个角色是不同的 profile，可能耗时数小时。

**两者可以共存**：Kanban worker 可以在自己的运行中内部调用 `delegate_task`。这并非"二选一"的决策，而是架构层面的组合。

---

## 四、Swarm 拓扑实战

### `hermes kanban swarm` 命令

v0.16 的最大亮点之一是 `hermes kanban swarm` 命令，它可以一键创建一个多 worker 协作拓扑：

```bash
hermes kanban swarm "设计一个多区域故障转移方案" \
  --workers researcher,architect,sre \
  --verifier reviewer \
  --synthesizer writer
```

### Swarm v1 图结构

```
                          ┌─────────────────┐
                          │    Root Card     │ （已完成的根/黑板书）
                          │   (blackboard)   │
                          └────────┬─────────┘
                                   │
                   ┌───────────────┼───────────────┐
                   │               │               │
            ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
            │  Worker 1   │ │  Worker 2   │ │  Worker 3   │（并行执行）
            │ researcher  │ │  architect  │ │    sre      │
            └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                   │               │               │
                   └───────────────┼───────────────┘
                                   │ （所有 worker 都完成后）
                            ┌──────▼──────┐
                            │  Verifier   │（gated — 审查所有 worker 的输出）
                            │  reviewer   │
                            └──────┬──────┘
                                   │ （verifier 通过后）
                            ┌──────▼──────┐
                            │ Synthesizer │（gated — 整合最终输出）
                            │   writer    │
                            └─────────────┘
```

### 结构特性

- **Root/Blackboard Card**：已完成的根卡片。结构化 JSON comment 存储在根卡片上，任何 worker 都可读取。这是"共享上下文"的持久化实现方式。
- **Parallel Workers**：N 个 worker 并行执行，互不依赖。适合需要多角色同时研究的场景。
- **Gated Verifier**：在所有 worker 完成后被唤醒。检查工作质量，决定是否需要重试。
- **Gated Synthesizer**：在 verifier 标记工作通过后被唤醒。整合所有 worker 的输出为最终成果。

关键的设计决策是：**Swarm 使用标准的 Kanban 依赖引擎（Link + parent→child）实现门控。** 不需要特殊的调度器，所有任务经过正常的 dispatcher 循环分发。这意味着你可以手动调整依赖关系、插入额外步骤、甚至重新排列整个拓扑。

### 实战：博客写作流水线

Kanban Swarm 的一个典型应用场景是博客自动化流水线。以下是使用 Kanban 任务依赖图实现的协作流程：

```python
# Orchestrator 创建 Kanban 任务图
kanban_create(
    title="调研博客主题：[主题]",
    assignee="researcher",
    body="搜索主题相关的中英文资料，收集素材和参考文献。",
)  # → t_research

kanban_create(
    title="撰写文章：[主题]",
    assignee="writer",
    parents=["t_research"],  # 调研完成后才启动
    body="根据调研报告撰写 ≥2000 字文章，保持技术深度。",
)  # → t_writer

kanban_create(
    title="审校文章：[主题]",
    assignee="reviewer",
    parents=["t_writer"],  # 文章写完后才启动
    body="检查语法、术语一致性、逻辑连贯性。",
)  # → t_reviewer

kanban_create(
    title="发布文章：[主题]",
    assignee="publisher",
    parents=["t_reviewer"],  # 审校完成后才启动
    body="保存文件、Git commit、触发构建。",
)  # → t_publisher
```

这个流水线的优雅之处在于：Dispatcher 自动管理依赖关系。当 `t_research` 被标记为 `done` 后，`t_writer` 自动从 `todo` 提升为 `ready`。如果 `t_reviewer` 发现文章有问题怎么办？Reviewer 可以调用 `kanban_block` 阻塞任务，然后在 comment 中说明问题，Writer 看到阻塞后重新打开并修复，再重新完成。

---

## 五、Run 历史与结构化 Handoff

Kanban 与普通 TODO 系统的关键区别之一是 **Run 历史和结构化 Handoff**。

每次 `kanban_complete(summary="...", metadata={...})` 调用都会创建一个新的 `task_runs` 记录。这意味着一个 Task 可以有多次 Run：

```
Task "实现认证 API"
├── Run 1: outcome="blocked",   profile=backend-dev, summary="需要确认鉴权方案"
├── Run 2: outcome="completed", profile=backend-dev, summary="POST /register 已实现"
└── (下一次 dispatch tick 不会重新调度已完成的 task)
```

为什么这个设计重要？因为 **`summary` + `metadata` 是多智能体协作中最被低估的 primitive**。

### `summary` 的作用

`summary` 是给**下游 Agent 和人类**阅读的结构化摘要。当下一个 worker 被 spawn 时，它会通过 `kanban_show()` 读取前一次 run 的 summary。这解决了传统 RPC 调用中最大的痛点：**信息传递依赖上下文压缩**。

在 RPC 模式下，Task A 的结果被压缩进父上下文中，Task B 看到的是一个模糊的、被上下文窗口截断的版本。而在 Kanban 中，每个 run 的 summary 是**独立存储、持久化可查**的。

### `metadata` 的作用

`metadata` 是一个 JSON 字典，用于传递**键值对风格的结构化信息**：

```python
kanban_complete(
    summary="实现了 POST /register 和 POST /login 两个端点",
    metadata={
        "api_routes": ["POST /register", "POST /login"],
        "test_coverage": 87.5,
        "files_changed": ["src/auth/handler.py", "src/auth/validator.py"],
        "breaking_changes": False
    }
)
```

下游 Agent 可以 `kanban_show()` 读取这个 metadata，并根据其中的信息决定下一步策略。例如，如果 `breaking_changes = True`，reviewer 可以自动跳入更严格的审查流程。

### 为什么需要两张表？

Task + Run 的双表设计是为了分离"任务定义"和"执行历史"：

- **`tasks` 表**：任务的定义 —— 标题、body、状态、依赖。生命周期跨越多次 attempt。
- **`task_runs` 表**：执行的记录 —— 谁运行的、结果如何、summary 和 metadata 是什么。每次 attempt 一行。

这样设计的好处是：你可以看到同一个任务在不同 attempt 中的演进 —— Reviewer 第一次拒绝时的 summary，与第二次批准时的 summary，都在同一个任务的 timeline 里。这对审计和复盘至关重要。

---

## 六、生产级可靠性

多 Agent 系统从原型走向生产的最大鸿沟是什么？**可靠性**。Demo 中一切正常，但一旦长期运行，worker 会崩溃、网络会中断、心跳会超时 —— 这些都是常态。

Hermes Kanban 内置了多层可靠性保障：

### 熔断器（Circuit Breaker）

当某个 profile 连续 N 次 spawn 失败（默认 2 次），dispatcher 自动将任务**阻塞**，而不是无限重试。这是为了防止资源浪费在已经崩溃的 worker 上。

```bash
# 配置熔断阈值
hermes config set kanban.circuit_breaker_threshold 3
```

熔断后，任务进入 `blocked` 状态，等待人类介入排查问题。修复后调用 `kanban_unblock()` 即可恢复。

### 崩溃恢复（Crash Recovery）

worker 进程可能因为各种原因死亡 —— OOM、代码 bug、信号中断。Dispatcher 通过以下机制处理：

1. **PID 检测**：如果 worker 的 PID 消失，dispatcher 释放 claim，任务回到 `ready`
2. **崩溃信息记录**：前一次尝试的 crash 信息写入 `task_runs`，新 worker 读取后可以切换策略
3. **有限重试**：重新 spawn，但受熔断器保护

### 协议违规保护（Protocol Violation Guard）

Worker 有明确的协议：必须在完成时调用 `kanban_complete()` 或 `kanban_block()`。如果 worker 进程在任务仍为 `running` 状态时退出（exit code 0），这就是 **协议违规**。

Dispatcher 的防御体系：

- 触发 `protocol_violation` 事件记录到 `task_events`
- Agent 侧在模型即将停止而未调用工具时，注入最多 2 次合成 nudge 提示
- Dispatcher 侧允许最多 3 次连续违规后**自动阻塞**任务

### 心跳检测（Heartbeat）

长时间运行的任务需要定期汇报存活状态。Worker 通过 `kanban_heartbeat(note="...")` 发送心跳信号：

- 如果任务运行超过 4 小时且最近 1 小时没有心跳，dispatcher 会回收任务
- 心跳可以携带进度 note，人类可通过 Dashboard 实时查看

### 完整的 Run 生命周期

每次 attempt 都有明确的 lifecycle：

1. **Spawn** → `kanban_show()` 读取标题 + body + parent handoff + prior attempts + 完整 comment 线程
2. **`cd $HERMES_KANBAN_WORKSPACE`** → 在 workspace 中执行实际工作
3. **`kanban_heartbeat(note="...")`** → 定期汇报
4. **`kanban_complete(summary="...", metadata={...})`** → 完成
5. 或 **`kanban_block(reason="...")`** → 阻塞等待输入

崩溃和超时也会关闭当前 run，outcome 分别为 `crashed` / `timed_out`，确保每个 attempt 都有记录。

---

## 七、Dashboard 和开发者体验

### 轻量插件架构

Kanban Dashboard 是 Hermes Dashboard 的**内置插件**（`plugins/kanban/`），遵循标准的 dashboard 插件契约。它只有约 **700 行 Python** —— 没有新增业务逻辑，所有操作都通过 `kanban_db` 层。

```
┌────────────────────────┐      WebSocket (tails task_events)
│   React SPA (plugin)   │ ◀──────────────────────────────────┐
└──────────┬─────────────┘                                    │
           │ REST over fetchJSON                              │
           ▼                                                  │
┌────────────────────────┐     writes / calls kanban_db.*     │
│  FastAPI router        │     directly — same code path      │
│  plugins/kanban/       │     the CLI verbs use              │
└──────────┬─────────────┘                                    │
           │                                                  │
           ▼                                                  │
┌────────────────────────┐                                    │
│  ~/.hermes/kanban.db   │ ───── append task_events ──────────┘
│  (WAL, shared)         │
└────────────────────────┘
```

### 可视化布局

Dashboard 展示了六列 Kanban **triage | todo | ready | running | blocked | done**（+ 可切换的 archived 列）：

- **Running 列按 profile 分组**（默认开启）—— 每个 worker 的活动一目了然
- **卡片信息**：task id、标题、优先级 badge、tenant 标签、assignee、comment/link 计数、进度 pill
- **扁平化视图**：关闭 "Lanes by profile" 后 Running 列合并为单列表

### 交互功能

| 功能 | 说明 |
|------|------|
| **拖拽** | 卡片在列之间拖拽改变状态 |
| **创建任务** | 点击列头 `+` 打开对话框：标题、assignee、workspace、goal mode、parent |
| **侧边抽屉** | 依赖编辑器、状态操作、Comment 线程、事件日志 |
| **过滤器** | 搜索框、tenant、assignee、"show archived" 切换 |
| **Nudge Dispatcher** | 立即执行 dispatch tick，无需等待 60 秒 |
| **Decompose / Specify** | Triage 列的 LLM 驱动操作 |
| **批量操作** | Shift/Ctrl+Click 或复选框选择，批量状态转换/归档/重新分配 |

### REST API

所有路由挂载在 `/api/plugins/kanban/` 下，支持编程化控制：

```bash
# 获取全 board 状态
curl $DASHBOARD_URL/api/plugins/kanban/board

# 创建任务
curl -X POST $DASHBOARD_URL/api/plugins/kanban/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "修复登录页 bug", "assignee": "frontend-dev"}'

# 追加评论
curl -X POST $DASHBOARD_URL/api/plugins/kanban/tasks/42/comments \
  -H "Content-Type: application/json" \
  -d '{"text": "这是一个已知的 race condition"}'
```

### 自动编排

Dashboard 支持**自动编排模式**（默认启用）：每个 dispatch tick 上，dispatcher 自动对 `triage` 列的任务运行 decomposer（每 tick 最多处理 3 个）。Decomposer 读取已安装的 profile roster + 描述，调用 LLM 生成任务依赖图。

配置项：

| 键 | 默认值 | 说明 |
|----|--------|------|
| `auto_decompose` | `true` | 自动运行 decomposer |
| `auto_decompose_per_tick` | `3` | 每 tick 分解上限 |
| `orchestrator_profile` | `""` | 分解后根/编排任务的 assignee |
| `default_assignee` | `""` | LLM 选到未知 profile 时的退路 |

### Gateway 通知

Kanban 与 Gateway 通知系统联动：`/kanban create` 后自动订阅，任务完成或阻塞时 Gateway 推送消息。这提供了"hands-free"体验 —— 创建任务后不需要轮询 Dashboard，系统会在关键事件发生时通知你。

---

## 八、v0.16 新特性一览

### Goal Mode 卡片（`--goal`）

v0.16 引入了 Goal Mode，让 worker 在一个 **goal loop** 中运行（与 `/goal` 斜杠命令相同的引擎）。每次 turn 后，辅助 judge 检查输出是否满足卡片的 acceptance criteria：

```bash
hermes kanban create "将文档翻译为法语" \
    --body "Acceptance: 每个页面都翻译，无英文遗留，链接完整。" \
    --assignee linguist \
    --goal \
    --goal-max-turns 15
```

- 适用于开放式的多步骤任务（如翻译、代码重构、调研报告）
- 预算耗尽时卡会进入 `blocked`（而不是静默退出）
- 简单任务绕开 Goal Mode（避免额外的 judge 开销）

### File Attachments

任务可以携带文件附件（PDF、图片、源文档等）：

- **上传**：Dashboard 卡片抽屉的 Attachments 部分，多文件同时上传，每个上限 25MB
- **存储**：`~/.hermes/kanban/attachments/<task_id>/`
- **Worker 视角**：context 中包含每个文件的名称和绝对路径，worker 可直接 `read_file`
- **配置**：`HERMES_KANBAN_ATTACHMENTS_ROOT` 自定义存储位置

### Default Assignee

`kanban.default_assignee` 配置项提供了安全网：当 decomposer 的 LLM 选择了一个不存在的 profile 名时，child task 自动分配给 `default_assignee`。这防止了"幽灵 profile"导致的调度死锁。

### Per-Profile Concurrency Cap

`kanban.max_in_progress_per_profile` 限制了单个 profile 同时运行的任务数，与 board 级别的 `max_in_progress` 同时生效（两者都需允许才能 spawn）。防止了某个 profile 被过度调度而崩溃。

### `/todo` 同步

v0.16 新增了 Kanban 与 `/todo` 系统的同步机制，允许任务自动同步到 Hermes 的 todo 系统（更多细节可参考 Release Notes 的 PR 描述）。

---

## 九、总结：Kanban 适合解决什么问题？

### 适合 Kanban 的场景

1. **跨角色流水线** —— 调研 → 写作 → 审校 → 发布，每个角色是不同的 profile
2. **需要人类介入的工作流** —— 模糊决策、质量审核、资源分配
3. **长期运行的任务** —— 可能跨越数小时甚至数天，需要崩溃恢复
4. **需要审计跟踪的场景** —— 合规要求、事后复盘、故障排查
5. **多 Agent 并行协作** —— Fan-out 扇出 → gated verifier → gated synthesizer
6. **定时/重复性工作** —— 日报、周报、监控日志，通过 cron + Kanban 实现

### 继续使用 `delegate_task` 的场景

1. **短期的内联推理** —— 父 Agent 需要快速分析结果才能继续
2. **不需要持久化的子任务** —— once-and-done，完成后不需要追溯
3. **紧耦合的子任务** —— 结果必须立即回到父上下文

### 两者组合的模式

```python
# Orchestrator 使用 delegate_task 做分析
analysis = delegate_task("analyze_topic", topic=topic)

# 然后创建持久化 Kanban 工作流
tasks = []
for worker in ["researcher", "writer", "reviewer", "publisher"]:
    tasks.append(kanban_create(
        title=f"{worker}: {topic}",
        assignee=worker,
        parents=[tasks[-1]] if tasks else None,
        body=analysis.summary  # 从 delegate_task 传过来的上下文
    ))
```

Kanban 的设计并非要替代 `delegate_task`。它们是两个不同层次的协作 primitive —— 一个用于**进程内**的快速调用，一个用于**跨进程、跨时间、跨角色**的持久化协作。理解它们各自的设计哲学，你就能在不同的场景中做出正确的选择。

---

*本文基于 Hermes Agent v0.16 (v2026.6.5) 编写。更多信息请参考 [官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) 和 [v0.16 Release Notes](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5)。*
