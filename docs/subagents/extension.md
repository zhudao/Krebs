# Subagent Extension

> 基于 `.pi/extensions/subagent/index.ts` 源码

## 概述

Subagent Extension 是 Krebs 的**子代理管理模块**，通过 Tools 提供给 LLM 调用，实现启动、调度、监控子代理的能力。

## 核心文件

```
.p.i/extensions/subagent/index.ts          ← 入口，注册 12 个 Tools
server/services/subagent/
  ├── agent-manager.ts                     ← 代理生命周期 + 并发控制
  ├── agent-runner.ts                     ← 创建独立 AgentSession
  ├── scheduler.ts                         ← 定时任务调度
  ├── fleet-view.ts                       ← 代理状态视图
  └── custom-agents.ts                    ← 加载 .pi/agents
```

## 架构分层

```
┌─────────────────────────────────────────────────────────┐
│ Tool API Layer (12 Tools)                               │
│  Agent / TaskCreate / TaskExecute / Schedule ...        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Task State Layer (taskRecords)                          │
│  Map<sessionId, Map<taskId, TaskRecord>>              │
│  管理任务状态 + agentId 绑定关系                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Agent Execution Layer                                   │
│  agent-manager.ts → agent-runner.ts → AgentSession     │
│  并发控制 (max=5)、超时、上下文继承                      │
└─────────────────────────────────────────────────────────┘
```

## 12 个 Tools

| Tool | 底层调用 | 说明 |
|------|----------|------|
| `Agent` | `createAgent()` | 直接启动子代理 |
| `get_subagent_result` | `getAgentResult()` | 获取代理执行结果 |
| `steer_subagent` | `steerAgent()` | 向运行中的代理发消息 |
| `TaskCreate` | `taskMap.set()` | 创建任务记录 |
| `TaskList` | `taskMap.values()` | 列出所有任务 |
| `TaskGet` | `taskMap.get()` | 获取任务详情 |
| `TaskUpdate` | `task.status = ...` | 更新任务状态 |
| `TaskExecute` | `createAgent()` + 绑定 | 用代理执行任务 |
| `FleetView` | `createFleetView()` | 查看运行中的代理 |
| `Schedule` | `scheduler.addJob()` | 调度周期性任务 |
| `CancelSchedule` | `scheduler.removeJob()` | 取消调度 |
| `CleanupAgents` | `cleanupAgents()` | 清理所有代理 |

## Task vs Agent

**Task** 和 **Agent** 是两层抽象：

| | Task | Agent |
|--|------|-------|
| 本质 | 逻辑工作单元 | 实际执行者 |
| 角色 | "what needs to be done" | "who does it" |
| 数据结构 | `TaskRecord` | `AgentRecord` |
| 创建方式 | `TaskCreate` tool | `createAgent()` |

### TaskRecord 结构

```typescript
interface TaskRecord {
  id: string;
  name: string;
  description?: string;
  status: "pending" | "running" | "done" | "failed" | "cancelled";
  agentId?: string;     // ← 与 Agent 层关联
  result?: any;
  createdAt: number;
}
```

### 两种使用模式

| 模式 | Tool | 流程 |
|------|------|------|
| **Task 模式** | `TaskExecute` | TaskCreate → TaskExecute → 自动绑定 `task.agentId` |
| **直接模式** | `Agent` | 直接创建 Agent，不经过 Task |

### 绑定时机

`TaskExecute` 执行时：

```typescript
const task = taskMap.get(taskId);
task.status = "running";

const result = await createAgent(sessionId, task.description, ...);
task.agentId = result.agentId;  // ← 绑定建立
```

## Agent 执行流程

### 直接模式：Agent tool

```
Agent(tool params)
    ↓
createAgent()
    ↓
agent-manager.ts
    ├── createSubagentSession() → agent-runner.ts
    │     ├── 创建独立 AgentSession
    │     ├── 过滤 Krebs 专用扩展 (memory-context, memory 等)
    │     ├── 注入 bash tool
    │     └── buildSubagentPrompt() 处理上下文继承
    ├── record = AgentRecord { id, session, status }
    └── startAgent() 或 queue.enqueue()
```

### Task 模式：TaskExecute

```
TaskExecute(taskId)
    ↓
taskMap.get(taskId) → TaskRecord
task.status = "running"
    ↓
createAgent() + 绑定 task.agentId
    ↓
后续同上...
```

## 并发控制

```typescript
SessionState {
  queue: AgentQueue;      // 等待队列
  records: Map;            // 运行中的 agents
  maxConcurrent: 5;        // 上限
}

// 超过上限时入队，有 agent 完成后 dequeue 启动下一个
```

## 定时任务调度

### Job / Task / Agent 概念区分

| 概念 | 层级 | 本质 | LLM 感知 |
|------|------|------|----------|
| **Job** | Scheduler 层 | "什么时候执行"的计划 | 无 |
| **Task** | Extension 层 | 逻辑工作单元 | `TaskCreate` 等 Tools |
| **Agent** | Execution 层 | 实际执行者 | `Agent` tool |

```
Schedule Tool
    ↓
scheduler.addJob({ task: "xxx", cron: "0 9 * * *" })
    ↓
Job { id, task, cron, nextRunAt, isRunning }
    ↓
到期执行时
    ↓
createAgent() → AgentSession (子代理)
```

Job 是 Scheduler 的内部概念，LLM 感知不到。LLM 只知道 `Schedule` / `CancelSchedule` 等 Tools。

### 两种调度方式

| 方式 | 参数 | 实现 |
|------|------|------|
| **Interval** | `intervalMs` | `setInterval` 轮询 |
| **Cron** | `cron` | `setTimeout` + 简化 cron 解析 |

### Interval 模式

```typescript
scheduler.addJob({
  task: "do something",
  intervalMs: 60000,  // 每分钟
});

// 内部用 setInterval 轮询，最少每分钟检查一次
setInterval(() => {
  if (job.isDue()) this.executeJob(job);
}, Math.min(intervalMs, 60000));
```

### Cron 模式

```typescript
scheduler.addJob({
  task: "do something",
  cron: "0 9 * * 1-5",  // 每周一至周五 9:00
});

// 每次执行完再计算下一次时间，用 setTimeout 等待
executeJob() → scheduleNextCron() → setTimeout(delay) → executeJob() → ...
```

### Cron 解析

支持标准 5 段 cron：`* * * * *` (分 时 日 月 周)

```typescript
matchesCronPart(part, value):
  *      → 始终匹配
  */5    → 每 5 个单位
  1-5    → 范围匹配
  1,3,5  → 列表匹配
```

### Job 并发规则

```
同一个 Job:  isRunning=true 时直接跳过，不并发
不同 Job:   各自独立，可并发执行
```

### 两层并发控制

```
Schedule Job 1 ──┐
Schedule Job 2 ──┼──→ createAgent() ──→ agent-manager ──→ maxConcurrent=5
Schedule Job 3 ──┘                              ↑
                                        实际的 Agent 并发限制
```

- **Scheduler 层**：Job 之间可并发
- **Agent 层**：受 `maxConcurrent=5` 限制

## 上下文继承

`inheritContext: true` 时，`buildSubagentPrompt()` 会：

1. 从父会话提取最近 N 条消息 (`maxContextMessages`)
2. 过滤敏感信息 (`filterSensitive`)
3. 拼接成 prompt 发送给子代理

```typescript
// agent-runner.ts:200
buildSubagentPrompt(task, ctx, options) {
  const cleanContext = buildCleanContext(ctx, {
    maxMessages: options.maxContextMessages ?? 10,
    filterSensitive: options.filterSensitive ?? true,
  });
  return `${cleanContext}\n\n# Your Task\n${task}`;
}
```

## 状态管理

```
schedulers:     Map<sessionId, SubagentScheduler>   ← 调度器实例
taskRecords:    Map<sessionId, Map<taskId, TaskRecord>>  ← 任务记录
sessionStates:  Map<parentSessionId, SessionState>  ← Agent 状态
```

## 子代理扩展过滤

子代理创建时会排除 Krebs 专用扩展：

```typescript
// agent-runner.ts:24
const KREBS_EXCLUDE_EXTENSIONS = [
  'memory-context',
  'memory',
  'session-history-rag',
  'goal-constraint',
  'self-verification',
];
```

因为这些扩展的功能（上下文压缩、记忆等）应由父代理处理，子代理保持独立。

## 关键特性

| 特性 | 说明 |
|------|------|
| Session 隔离 | 每个 session 有独立的 scheduler、task map、session state |
| 并发控制 | 最多 5 个并发子代理，超出进入队列等待 |
| 超时控制 | `timeoutMs` 默认 5 分钟 |
| 上下文继承 | 可选从父会话继承消息 |
| 定时任务 | 支持 cron 表达式或 interval 调度 |
