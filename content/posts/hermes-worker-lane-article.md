---
title: "Hermes Agent Kanban Worker Lanes：如何编排 Claude Code、Codex 与 OpenCode"
date: 2026-07-20
draft: false
description: "深入解析 Hermes Agent v0.17 的 Worker Lane 架构——如何将 Claude Code、Codex CLI、OpenCode 等编码 Agent 变成 Kanban 持久化任务系统的工作单元。Kanban Swarm 系列第三篇。"
tags: ["Hermes Agent", "Kanban", "Worker Lane", "Multi-Agent", "Claude Code", "Codex", "OpenCode", "Nous Research"]
author: "lihaoran"
---

# Hermes Agent Kanban Worker Lanes：如何编排 Claude Code、Codex 与 OpenCode

> 你是选择 Claude Code 还是 Kanban？——这可能是个错误的问题。

你的项目中同时跑着 Claude Code 重构后端、Codex CLI 写单元测试、OpenCode 做代码审查。三个 Agent 各司其职，但谁在协调它们？谁在跟踪它们有没有卡住？谁在某个 Agent 失败时自动安排重试？

如果答案是「没有」，那你就遇到了多智能体系统的**运维层真空（Operations Vacuum）**：Agent 的代码生成能力越来越强，但编排它们的工具还停留在手动开 tmux 窗口和写 shell 脚本的时代。

Hermes Agent v0.17 引入的 **Worker Lane** 架构，正是填补这个真空的方案。它不是又一个 Agent 框架——它是一个持久化的任务操作系统，让 Claude Code、Codex CLI、OpenCode 这些编码 Agent 变成它「插件」一样的工作单元。

---

## 2. Worker Lane 架构详解

Worker Lane 的定义出乎意料地简洁：**调度器可将任务路由至的一类进程**。每个 Lane 只有三个要素：

| 要素 | 作用 |
|------|------|
| `assignee` 字符串 | 任务的归属标识（如 `writer`、`codex-worker`） |
| spawn 机制 | 调度器如何启动这个 Lane 的 worker |
| 生命周期终结器 | worker 完成或失败后如何清理 |

### 为什么重要

这种抽象刻意保持极简——不绑定 Agent 实现、不要求特定通信协议。把「如何启动」和「如何清理」交给 Lane 自己定义，调度器只关心「谁该做」和「做完没」。这意味着你可以为 Python 脚本、Shell 命令、Docker 容器甚至远程 SSH 任务各自定义一个 Lane，而调度逻辑完全复用。

### 三种 Lane 形态

**Hermes Profile Lane（默认）**：`assignee = writer` 时，调度器执行 `hermes -p writer chat -q <prompt>`。Worker 启动时自动注入 `KANBAN_GUIDANCE` 系统提示，包含完整的生命周期协议——Worker 不需要额外 skill 或配置。

**Orchestrator Profile Lane**：工具集只包含 `kanban_*` 工具，没有 `terminal`、`file`、`code`。专职将大任务拆解为子任务 DAG，不负责执行。这种职责分离保证了「思考者不执行，执行者不思考」——每个环节只用自己需要的工具集，降低了幻觉风险。

**外部 CLI Worker Lane（设计阶段）**：Issue #19931 讨论的形态——调度器直接 spawn `codex exec` 或 `claude -p`，通过环境变量注入上下文信息，worker 通过 stdout JSON 协议回报结果。尚未成为 paved path，但方向已定。

每次 spawn 注入的环境变量暴露了 Lane 的核心契约：

```
HERMES_KANBAN_TASK      # 当前任务 ID（Worker 通过这个回写结果）
HERMES_KANBAN_DB        # 看板 SQLite 路径
HERMES_KANBAN_BOARD     # 看板标识
HERMES_KANBAN_WORKSPACE # 专属工作空间路径
HERMES_KANBAN_RUN_ID    # 当前运行 ID（区分同一任务的不同尝试）
```

任何能读取环境变量和操作文件系统的进程，都可以成为一个 Lane Worker。这就是「拓展而非锁定」的体现。

---

## 3. 调度器容错设计深度解析

调度器运行在 Gateway 内部，每 60 秒执行一次六阶段清查循环。其中几个设计决策值得展开。

### 3.1 六阶段 Dispatch Tick

1. **Reclaim stale claims** —— 认领超时（默认 15 分钟）→ 释放资源，让其他调度器实例可以接手
2. **Reclaim crashed workers** —— `kill(pid, 0)` 探测存活；进程死亡 → 释放认领并触发重试；达到熔断阈值 → 自动阻塞任务
3. **Promote ready tasks** —— 父任务全部 Done → 子任务从 Todo 自动升为 Ready
4. **Spawn ready workers** —— 原子认领 + spawn，避免多调度器重复派发
5. **Decompose triage cards** —— 自动拆解 Triage 卡片为子任务 DAG
6. **Report fleet health** —— 集群健康状态汇总

### 3.2 存活 PID 扩展

大多数任务调度系统（如 Celery）超时即杀死进程。Kanban 调度器不同——它先用 `kill(pid, 0)` 探测 worker 是否还在运行。如果存活但超时，**自动扩展 TTL 而非杀死**。

### 为什么重要

运行 DeepSeek-R1 或 Claude Opus 时，一次推理可能长达数分钟。简单的超时杀死策略意味着昂贵的 token 浪费和数分钟的计算损失。存活 PID 扩展保证了慢模型不会因「推理太长」而丢失任务——这是面向 LLM workload 的调度器与传统任务队列的关键区别。

### 3.3 协议违规检测：双重防护

Worker 以 exit code 0 退出但未调用 `kanban_complete` 或 `kanban_block`，构成「协议违规（protocol violation）」。误报率必须低，因为正常场景下的 Agent 可能只是忘了最后一步。系统设计了两层防护：

1. **Agent 侧 nudge**：每次 tool 调用后检查上下文，如果 worker 似乎完成任务但未声明，自动提示「看起来你完成了工作，别忘了调用 kanban_complete」
2. **调度器侧 bounded retry**：连续 3 次协议违规后自动阻塞任务，等待人工审查——不浪费无限次重试

### 3.4 Stranded 任务告警

任务在 Ready 状态停留超过 30 分钟（可配置）→ 系统发出分级告警（Error / Critical 两级）。看似简单，但在实战中挽救过不止一次——当调度器进程因未捕获异常而停止 tick 循环、但主进程仍然存活时，你会在半天后发现积压了几十个「应该被处理」的任务。

---

## 4. 编码 Agent CLI 竞品生态

### 4.1 市场概览

| 产品 | 开发商 | 定位 | 开源协议 | 核心模型 |
|------|--------|------|----------|----------|
| **Claude Code** | Anthropic | 自主编码 Agent CLI | 闭源 | Claude Sonnet/Opus |
| **Codex CLI** | OpenAI | 自主编码 Agent CLI | Apache 2.0 | o4-mini/o3 |
| **OpenCode** | Anomaly Co. | 多模型编码 Agent CLI | AGPL-3.0 | 任意（OpenRouter） |
| **Aider** | Paul Gauthier | 终端配对编程 | Apache 2.0 | 任意（20+ Provider） |

### 4.2 功能矩阵

| 维度 | Claude Code | Codex CLI | OpenCode | Aider |
|------|-------------|-----------|----------|-------|
| 子 Agent 系统 | ✅ `/agents`、`--agents` | ❌ 无 | ✅ Build/Plan 双 Agent | ✅ Architect/Editor |
| 多会话并行 | tmux 手动 | 后台进程 | `background=true` | ❌ 不支持 |
| MCP 支持 | ✅ 原生 | ❌ | ❌ | ❌ |
| Git 集成 | ✅ worktree + PR 检查 | ✅ 必须 Git repo | ❌ 有限 | ✅ 自动 commit |
| 持久上下文 | ✅ CLAUDE.md + 自动记忆 | ❌ 无 | ✅ 会话 ID 恢复 | ✅ `.aider.conf.yml` |
| 仪表盘 / Dashboard | ❌ | ❌ | ❌ | ❌ |
| 成本追踪 | ✅ `/cost` | ❌ 无显式 | ✅ `opencode stats` | ✅ `--analytics` |
| Provider 灵活度 | 仅 Anthropic | 仅 OpenAI | 任意（OpenRouter 等） | 任意（20+） |
| 沙箱安全 | ❌ | ✅ bubblewrap | ❌ | ❌ |
| JSON 输出 | ✅ `--output-format json` | ❌ | ✅ `--format json` | ❌ |

### 4.3 关键差异分析

**多 Agent 编排能力**：Claude Code 拥有最成熟的子 Agent 系统（自定义 agents、CLAUDE.md hook 系统），但它无法跨进程协调任务——每个 `claude -p` 调用独立运行，互相不知道彼此的存在。Codex CLI 完全单进程单任务。OpenCode 的双 Agent 设计接近 Claude Code，但同样没有持久化任务队列。

**共同盲区**：这四款产品都没有持久化任务队列、没有跨进程协调、没有审计轨迹、没有 Dashboard。它们解决的是同一个问题——「如何让 LLM 生成好代码」。而 Hermes Kanban 解决的是另一层问题——「如何管理好生成代码的 Agent 集群」。

---

## 5. Kanban Swarm vs 编码 Agent：分层互补

### 5.1 核心论点

这是**分层互补**关系，不是竞争关系。从维度对比可以看得很清楚：

| 层面 | Kanban Swarm | 编码 Agent CLI（Claude Code 等） |
|------|-------------|--------------------------------|
| 任务持久化 | ✅ SQLite 状态机，任务生命周期跨进程存活 | ❌ 进程即任务，进程退出任务消失 |
| 可视化和审计 | ✅ Dashboard + 事件流 + 完整审计轨迹 | ❌ 只有终端日志 |
| 容错机制 | ✅ Reclaim → Crash Detection → Circuit Breaker 三层 | ❌ 失败即重启，无自动重试 |
| 人在回路 | ✅ Block / Waiting for Review | ❌ 仅有终端交互 |
| 代码生成能力 | ❌ 不生成代码 | ✅ 核心能力 |
| Provider 灵活性 | ✅ 任意模型 | ⚠️ 取决于产品 |

### 5.2 为什么重要

这种分层结构意味着你不必在「用 Kanban 编排」和「用 Claude Code 写代码」之间二选一。Kanban 处理任务队列、重试、并行度和审计；Claude Code 处理代码生成、重构和测试。两者各司其职，通过 Worker Lane 衔接。

竞品在协调能力上的共同局限是：它们假设「一个进程 = 一个任务」。在真实项目中，任务可能跨越数小时、需要人工审批、需要在失败后自动重试——这些正是 Kanban 状态机擅长的场景。

---

## 6. 实战：用 Kanban 编排外部编码 Agent

### 6.1 模式一：串行 Claude Code 子任务

常见的场景：Kanban Worker 接收一个重构任务，调用 Claude Code 执行具体代码修改，然后汇报结果。

```python
# Kanban Profile Worker 中启动 Claude Code 处理子任务
import json

def refactor_module(task_id, repo_path, module_path):
    prompt = f"Refactor {module_path} to use dependency injection pattern"
    
    result = terminal(
        f"claude -p '{prompt}' --output-format json",
        timeout=300,  # 5 分钟超时
        workdir=repo_path
    )
    
    if result["exit_code"] != 0:
        # 失败时自动重试（Kanban 调度器处理）
        kanban_block(
            task_id=task_id,
            reason=f"Claude Code failed: {result['error']}",
            kind="transient"
        )
        return
    
    # 提取变更文件列表
    changed = json.loads(result["output"])
    
    kanban_complete(
        task_id=task_id,
        summary=f"Refactored {module_path}: {len(changed['files'])} files changed",
        metadata={"changed_files": changed["files"], "agent": "claude-code"}
    )
```

### 6.2 模式二：并行编码工作流

将三个不同的子任务分配给三个不同的编码 Agent，同时运行，Kanban 统一管理生命周期。

```python
# 分配 3 个文件给不同编码 Agent
tasks = [
    ("claude", "Refactor src/auth/service.py"),
    ("codex",  "Add unit tests for src/auth/service.py -t pytest"),
    ("opencode", "Update API docs for auth module"),
]

# 并行启动（每个 terminal background 调用）
processes = []
for agent, instruction in tasks:
    p = terminal(
        f"{agent} -p '{instruction}'",
        background=True,
        notify_on_complete=True
    )
    processes.append(p)

# 全部完成后收集结果
final_results = []
for p in processes:
    result = process(action="wait", session_id=p["session_id"], timeout=600)
    final_results.append(result)
```

### 6.3 为什么重要

这个模式的技术复杂度并不高——它本质上只是 `terminal()` + `process()`。核心价值在于 `terminal()` 之外的整个 Kanban 层：**失败检测（exit code ≠ 0）、超时控制（timeout 参数）、重试策略（调度器自动处理）、进度可视化（Dashboard 实时刷新）**。这些是裸调 `claude -p` 所没有的。

### 6.4 Worker Lane 展望（Issue #19931）

远期目标是消除中间 Worker 层：调度器直接识别 `assignee = "codex-worker"`，在 Lane 配置中定义 spawn 命令和环境变量注入规则。

```yaml
# 未来 Lane 配置示例
lanes:
  - assignee: codex-worker
    spawn: "codex exec --task-id $KANBAN_TASK --workspace $KANBAN_WORKSPACE"
    terminator: "kill $PID"
    max_concurrency: 3
  
  - assignee: claude-worker
    spawn: "claude -p '$KANBAN_PROMPT' --output-format json"
    terminator: "kill $PID"
    max_concurrency: 2
```

届时，Kanban Worker 只需创建子任务并指定 `assignee="codex-worker"`，调度器直接 spawn Codex CLI 进程，无需中间 Python 层。Worker Lane 的定义保持不变的三个要素——assignee、spawn、terminator——但 spawn 的形态从 `hermes -p` 变成了 `codex exec`。

---

## 7. 总结与展望

从 v0.15 首次引入 Kanban Swarm（2026 年 5 月），到 v0.16 的 Dashboard + Auto-Decompose（6 月初），再到 v0.17 的 Worker Lane + Goal Mode（6 月下旬），Hermes Agent 的演进节奏近乎每月一个里程碑。脉络清晰：**从单 Agent 工具，到多 Agent 协调器，再到 Agent 编排操作系统**。

Worker Lane 的哲学是「拓展而非锁定」。它不要求你放弃 Claude Code、Codex 或 OpenCode——相反，它让你用这些工具最擅长的代码生成能力，同时赋予它们欠缺的运维层：持久化任务队列、跨进程可见性、三层容错和人在回路。

在同一台机器的终端里同时跑着多个编码 Agent 的场景，正在从「混乱的手动管理」走向「结构化的 Kanban 编排」。这篇文章讨论的 Worker Lane 是这个转变的关键基础设施——它定义了 Agent 如何被调度，如何沟通，如何在失败后恢复。这是多智能体系统从玩具验证走向生产环境的必经之路。

---

*参考：调研素材来自 [Hermes Agent 官方 Kanban 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban/)、v0.17 Release Notes 及社区讨论。本文为 Kanban Swarm 系列第三篇，前两篇覆盖了基本架构与 AutoGen/CrewAI/LangGraph 的对比。*
