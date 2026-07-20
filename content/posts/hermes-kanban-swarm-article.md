---
title: "Hermes Agent v0.16 Kanban Swarm 深度解析：从任务看板到多智能体协作拓扑"
date: 2026-07-20
draft: false
description: "深入解析 Hermes Agent v0.16 的 Kanban Swarm 功能，探索其架构设计、协作模式和实战应用。面向 AI 开发者。"
tags: ["Hermes Agent", "Kanban", "Multi-Agent", "Swarm", "AI", "Nous Research"]
---

# Hermes Agent v0.16 Kanban Swarm 深度解析：从任务看板到多智能体协作拓扑

## 引言：多智能体系统的"可见性危机"

想象这样一个场景：五个 AI Agent 同时在五个终端窗口中运行。一个在调研话题，一个在撰写内容，一个在分析数据，一个在审核输出，第五个可能在三分钟前已经静默崩溃——但没人注意到。

这就是目前大多数多智能体部署的现实。Agent 很强大，底层模型越来越智能，但操作层——即人类管理并行工作 AI 的实际体验——仍然惊人地原始。工程师们盯着日志文件、刷新终端，试图拼凑出数字员工团队实际在做什么的连贯图景。

业界给这个问题起了个名字，尽管很少大声说出来：**多智能体可见性危机（Multi-Agent Visibility Problem）**。它正在悄悄使 Agent 系统变得更难信任、更难扩展、更难交到非技术人员手中。

Nous Research 决定，这是下一个值得解决的问题。

## 从 delegate_task 到 Kanban：RPC 调用 vs 持久化工作队列

在 Hermes Agent 中，`delegate_task` 一直是最直接的子任务工具。它的工作方式像一个同步 RPC 调用：父 Agent 分叉出一个子 Agent，给它一个目标，等待它返回，然后拿到摘要。快速、有用，而且**只适用于一次性推理子任务**。

但当你试图用它做更多事情时，问题就暴露了：

| 维度 | delegate_task | Kanban |
|------|--------------|--------|
| **形态** | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| **父进程行为** | 阻塞直到子进程返回 | 创建后即遗忘 |
| **子进程身份** | 匿名子 Agent，无持久记忆 | 命名 profile，长期记忆 |
| **可恢复性** | 失败 = 彻底丢失 | 阻塞 → 解除 → 重试；崩溃 → 回收 |
| **人在回路** | 不支持 | 随时评论 / 解除阻塞 |
| **每任务 Agent** | 一个调用 = 一个子 Agent | 任务生命周期内 N 个 Agent |
| **审计轨迹** | 上下文压缩后丢失 | SQLite 中永久持久化 |
| **协调模式** | 层级式（调用者 → 被调用者） | 对等式——任何 profile 可读写任何任务 |

**一句话区分：** `delegate_task` 是一个函数调用；Kanban 是一个工作队列，每次交接都是任何 profile（或人类）可见且可编辑的数据行。

### 什么时候该用什么？

- **用 `delegate_task`**：父 Agent 需要快速推理答案，没有人类参与，结果可以安全地回到父上下文。例如："分析这个错误日志"、"搜索代码库中的模式"。
- **用 Kanban**：工作跨越 Agent 边界、需要持久化、可能需人类介入、可能由不同角色接手、或者需要在事后可发现。例如：跨越多阶段的博客流水线、研究团队协调、工程管道（分解 → 实现 → 审核 → 迭代 → PR）。

两者可以共存——Kanban worker 可以在运行期间内部调用 `delegate_task`。

## v0.16 Kanban Swarm 架构核心

Kanban Swarm 的底层架构出奇地简洁。去掉仪表板、CLI 和工具层，得到三个核心原语：

### 1. SQLite 持久化任务板

所有数据存储在 `~/.hermes/kanban.db`（默认板）或 `~/.hermes/kanban/boards/<slug>/kanban.db`（命名板）。使用 SQLite WAL 模式，确保并发安全。存什么？

- **tasks** — 每一行是一个任务：标题、描述、状态、指派人、租户命名空间、幂等键
- **task_links** — 父→子依赖关系，形成有向无环图（DAG）
- **comments** — 跨 Agent 通信协议，Agent 和人类都可以追加评论
- **runs** — 每次尝试都有一行记录，完整历史永不丢失

### 2. 状态机与依赖 DAG

每个任务经过七种状态：

```
Triage → Todo → Ready → Running → Blocked → Done → Archived
```

- **Triage**：粗略想法，尚未明确或分解。使用 `--triage` 标志创建。
- **Todo**：已定义但等待依赖或分配。
- **Ready**：已指派、无未完成父任务、等待调度器。引擎自动从 Todo 升入。
- **Running**：Worker 已认领任务并正在工作。
- **Blocked**：Worker 需要人类输入或熔断器触发。
- **Done**：完成，附带结构化摘要和元数据。
- **Archived**：清理完成，从默认视图中隐藏。

关键洞察：**最常见转换是自动的**。
- **Promotion**（Todo → Ready）在父任务全部 Done 后自动触发
- **Claim**（Ready → Running）由调度器原子执行，SQLite WAL 模式确保一次只有一个调度器 tick 能认领
- **Cleanup**（Running → Ready via 回收，或 Running → Blocked via 熔断器）在下一个调度器 tick 自动发生

依赖 DAG 的设计非常优雅：不需要手动协调，只需要用 `parents=[...]` 创建任务。规则简单到一行：一个任务保持在 `todo` 直到所有父任务进入 `done`，然后自动提升到 `ready`。

### 3. Dispatcher 循环

调度器是一个常驻循环，默认运行在 **gateway 内部**（`kanban.dispatch_in_gateway: true`），每 60 秒 tick 一次。在每个 tick 中，调度器执行六阶段清查：

1. **Reclaim stale claims** — 检查认领是否超时（时间戳 + TTL），超时的认领被释放
2. **Reclaim crashed workers** — 使用 `kill(pid, 0)` 探测 worker 进程是否存活。进程已消失但 TTL 未过期 → 释放认领。如果熔断器阈值已到 → 自动阻塞（Running → Blocked）
3. **Promote ready tasks** — 检查所有父任务 Done 的 Todo 任务，自动升为 Ready
4. **Spawn ready workers** — 为每个 Ready 且可调度的任务原子地认领并 spawn worker
5. **Decompose triage cards**（v0.16 新增） — 如果有 orchestrator 配置，自动分解 Triage 卡片
6. **Report fleet health** — 可选向服务端报告状态

**Spawn 失败保护**：连续 N 次 spawn 失败（默认 2 次）后，调度器自动阻塞该任务，附上最后一次错误原因——防止在 profile 不存在、workspace 无法挂载等场景下无限震荡。

### 4. 三种工作空间（Workspace）

每个 worker 在一个工作空间中运行，有三种选择：

| 类型 | 生命周期 | 用途 |
|------|---------|------|
| **scratch**（默认） | 任务完成后删除 | 一次性任务，ephemeral 工作 |
| **dir:<path>** | 持久保留 | 共享目录（Obsidian vault、邮件目录） |
| **worktree** | 持久保留 | Git worktree，用于编码任务 |

scratch 空间中的文件如果通过 `kanban_complete(artifacts=[...])` 明确声明，会复制到持久化的附件存储后再清理。未声明的路径会保持任务 in-flight 以便 worker 修正。

## 八种协作模式

Hermes Kanban 的设计文档定义了八种规范协作模式，覆盖大多数多智能体工作负载：

| 模式 | 描述 | 典型场景 |
|------|------|---------|
| **序列管道** | A → B → C 线性传递 | 调研 → 写作 → 发布 |
| **扇出/扇入** | 并行调研 → 综合 | 多源研究 → 报告 |
| **审核循环** | 提交 → 审核 → 重试 → 通过 | PR 代码审核 |
| **人在回路** | Agent work → 人类审查 → 放行 | 关键决策 |
| **竞标** | 多个 Agent 给出方案，择优执行 | 架构设计 |
| **守护进程** | 常驻任务，重启后继续 | 日更简报 |
| **数字孪生** | 持久命名助手，随时间积累记忆 | 收件箱分类 |
| **舰队** | 一个 Specialist 管理 N 个目标 | 50 个社交媒体账号 |

## Kanban Swarm vs 竞品深度对比

| 维度 | Hermes Kanban | AutoGen | CrewAI | LangGraph |
|------|--------------|---------|--------|-----------|
| **持久化** | SQLite WAL，跨重启持久 | 内存级 | 内存级 | 可选持久化 |
| **人在回路** | 原生支持评论/阻塞 | 需手动实现 | 有限支持 | 通过中断支持 |
| **可见性** | 完整仪表板 + Event Stream | 无内置仪表板 | CLI 日志 | LangSmith |
| **Agent 身份** | 命名 profile + 长期记忆 | 匿名函数 | 角色对象 | 节点函数 |
| **容错** | Crash 回收 + 熔断器 | 无内置 | 有限重试 | 通过检查点 |
| **审计** | 每行 SQLite 永久记录 | 无 | 无 | 有轨迹 |
| **Workspace** | scratch/dir/worktree | 无 | 无 | 无 |
| **学习曲线** | 中（配置文件） | 低 | 低 | 中高 |
| **部署** | 6 种终端后端 | 本地进程 | 本地进程 | 本地进程 |

Hermes Kanban 最大的差异化优势不在于 Agent 能力——底层 LLM 大家都一样——而在于 **操作层**：持久化、可见性、容错、人在回路。

## 实战：博客写作流水线全流程

以下是我们实际运行的 Kanban Swarm 博客流水线，展示完整的五阶段协作拓扑：

```
用户微信 📱 → "写一篇关于 X 的文章"
         ↓
  🔷 Orchestrator（拆解为：调研→写作→审核→发布）
         ↓
  🔷 Researcher（web_search + web_extract 收集素材）
         ↓
  🔷 Writer（撰写 ≥2000 字技术文章）
         ↓
  🔷 Reviewer（语法检查 + 质量审核）
         ↓
  🔷 Publisher（git commit → push → GitHub Pages）
         ↓
  🚀 完成 → 微信通知
```

每个角色都是一个独立的 **Hermes profile**（`orchestrator`、`researcher`、`writer`、`reviewer`、`publisher`），各有不同的工具集、技能和提示词。它们的记忆互不干扰，但共享同一个 Kanban 任务看板。

典型任务流中 observer 可以看到：

```bash
$ hermes kanban --board blog list
  t_5efa6b8d  ready  orchestrator  Hermes Agent v0.16 Kanban Swarm 深度解析
```

调度器自动将 ready 任务分配给对应 profile 的 worker。Worker 启动后使用 kanban_* 工具集与看板交互——不是 shell 调用 CLI，而是直接在 Agent 循环中调用 Python 函数。

## v0.16 在这个语境下意味着什么

Hermes Agent v0.16 "The Surface Release"（2026年6月5日发布）包含了 **874 次提交、542 个合并 PR、205,216 行新增、399 个 Issue 关闭**。在这个版本中，Kanban Swarm 虽然核心架构继承自 v0.12/v0.15，但获得了关键改进：

1. **Orchestrator 自动分解** — Triage 卡片可以自动被分解为子任务
2. **Dashboard 集成** — Kanban 看板现在可以从桌面 App 和 Web UI 原生访问
3. **Kanban Swarm 拓扑助手** — `hermes kanban swarm` CLI 子命令，一键启动一个协作拓扑
4. **技能环境门控** — Kanban 技能只对相关环境可见，减少 prompt 噪声
5. **同时运行多个看板** — 每个项目/仓库一个独立看板
6. **附件系统** — 任务可以携带 PDF、图片等源文件

v0.16 真正把 Kanban 从"一个实验性功能"变成了"产品级的多智能体平台"。

## 配置参考

以下是与 Kanban Swarm 相关的关键配置项：

```yaml
# ~/.hermes/config.yaml
kanban:
  orchestrator_profile: "orchestrator"   # 分解 Triage 卡片的 profile
  default_assignee: ""                    # 备用指派
  auto_decompose: true                    # 调度器是否在每个 tick 前自动分解
  auto_decompose_per_tick: 3              # 单 tick 最大分解数（burst 保护）
  dispatch_in_gateway: true               # 调度器常驻 gateway
  failure_limit: 2                        # 连续 spawn 失败上限

auxiliary:
  kanban_decomposer:                      # 分解器使用的人工智能模型（可选）
    provider: deepseek
    model: deepseek-v4-flash
```

## 总结与展望

Hermes Agent 的 Kanban Swarm 代表了多智能体系统从"玩具级"到"生产级"的重要跨越。它解决的不是 Agent 智能的问题——那是底层模型的事——而是 **Agent 协作的操作层问题**：

- 当 Agent 崩溃时，工作不会丢失
- 当需要人类决策时，可以在流程中随时介入
- 当需要了解系统状态时，有完整的可见性和审计轨迹
- 当需要跨 Agent 边界协调时，有统一的持久化通信协议

从技术架构角度看，Kanban Swarm 的成功在于三个设计选择：
1. **SQLite 持久化** — 简单可靠，无需外部依赖
2. **状态机自动转换** — 减少手动协调成本
3. **Profile 隔离** — 每个角色独立配置、独立记忆

展望未来（v0.17+ 已开始实现）：背景子 Agent、watch-windows 实时流、更丰富的仪表板集成，以及通过 Raft 网络和 Photon iMessage 的跨平台协调。

对于 AI 开发者而言，Kanban Swarm 提供了一个可立即部署的多智能体协作框架。**不再需要在几个终端窗口中手忙脚乱地管理 Agent 了**——一个 Kanban 看板，就能掌控一切。

---

*本文由 Hermes Agent Kanban Swarm 自动写作流水线生成：Orchestrator 拆解 → Researcher 调研 → Writer 撰写 → Reviewer 审核 → Publisher 发布。*

*参考资料：*
- [Hermes Agent 官方文档 - Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Hermes Agent 官方文档 - 架构](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)
- [GitHub: Multi-Agent Architecture Issue #344](https://github.com/NousResearch/hermes-agent/issues/344)
- [Hermes Agent v0.16 Release Notes](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5)
- [The Hermes Kanban: A Complete Guide](https://magnus919.com/2026/05/the-hermes-kanban-a-complete-guide-to-multi-agent-task-orchestration/)
- [From Black Box to Kanban Board (Medium)](https://medium.com/open-intelligence/from-black-box-to-kanban-board-how-hermes-agent-is-redefining-multi-agent-visibility-b5cffd2659d6)
