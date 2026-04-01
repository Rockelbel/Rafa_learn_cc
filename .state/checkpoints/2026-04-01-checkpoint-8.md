# Checkpoint 8 - 2026-04-01

## Module 6: Tasks & Agents - 已完成

### 分析文件 (15+)
1. ✅ utils/tasks.ts - Task 系统核心
2. ✅ commands/tasks/tasks.tsx - Task 命令 UI
3. ✅ tools/TodoWriteTool/TodoWriteTool.ts
4. ✅ tools/TaskCreateTool/TaskCreateTool.ts
5. ✅ components/TaskListV2.tsx
6. ✅ tools/AgentTool/AgentTool.tsx
7. ✅ tools/AgentTool/runAgent.ts
8. ✅ utils/agentContext.ts
9. ✅ utils/agentId.ts
10. ✅ utils/agentSwarmsEnabled.ts
11. ✅ tools/AgentTool/forkSubagent.ts
12. ✅ tools/AgentTool/resumeAgent.ts
13. ✅ tools/TeamCreateTool/TeamCreateTool.ts
14. ✅ tools/AgentTool/agentMemory.ts
15. ✅ tools/AgentTool/agentMemorySnapshot.ts

### 生成的笔记文件
- raw_notes/tasks_agents/task_system.md - Task 系统
- raw_notes/tasks_agents/agent_spawning.md - Agent 生成
- raw_notes/tasks_agents/agent_coordination.md - Agent 协调
- raw_notes/tasks_agents/agent_memory.md - Agent 内存

### 关键发现摘要

#### 1. Task 数据模型 (utils/tasks.ts)

**Task Schema**:
```typescript
{
  id: string           // 顺序 ID (1, 2, 3...)
  subject: string      // 简短主题
  description: string  // 详细描述
  activeForm: string   // 进行时形式 (spinner 显示)
  owner: string        // Agent ID
  status: 'pending' | 'in_progress' | 'completed'
  blocks: string[]     // 此任务阻塞的任务 ID
  blockedBy: string[]  // 阻塞此任务的任务 ID
  metadata: Record<string, unknown>
}
```

**存储架构**:
- 文件存储: `~/.claude/tasks/{taskListId}/{id}.json`
- High water mark: `.highwatermark` 文件跟踪最大 ID
- 路径清理防止目录遍历

**并发控制**:
- 两级锁定策略
- Task-level 锁用于更新
- List-level 锁用于 ID 生成
- 配置支持 ~10 并发 swarm agents

**依赖管理**:
- 双向阻塞关系 (blocks/blockedBy)
- 声明时自动同步
- 阻塞任务阻止 claim 操作

#### 2. Agent 生命周期

**生成阶段** (AgentTool.tsx):
- 解析 agent 定义
- 确定 sync vs async 执行
- 选择工具池

**执行阶段** (runAgent.ts):
- `createSubagentContext()` 创建隔离上下文
- Query loop 管理
- 父/子状态同步或隔离

**完成阶段**:
- MCP 连接清理
- Hooks 清除
- 后台任务终止

**Sync vs Async**:
| 特性 | Sync | Async |
|------|------|-------|
| setAppState | 共享父级 | 隔离 |
| abortController | 共享 | 独立 |
| 可显示 Prompt | 是 | 否 |
| 阻塞父级 | 是 | 否 |

#### 3. Agent 协调

**Swarm 模式**:
- Fork 模式: 继承完整父上下文用于 cache sharing
- Resume 模式: 从 transcripts 重建 agent 状态
- Coordinator 模式: 并行 worker 编排

**团队架构**:
- 团队文件: `~/.claude/teams/{team-name}/`
- 成员跟踪: pane IDs, colors, permission modes
- 支持 tmux, iTerm2, in-process 后端

**协调机制**:
- LocalAgentTask 管理后台执行
- SendMessageTool 实现 agent 间通信
- Token 计数和 activity 分类的进度跟踪
- 通过 byte-identical API prefixes 实现 cache sharing

**并行执行**:
- 只读任务可自由并行
- 写任务按文件集串行化
- Continue vs Spawn 决策矩阵基于上下文重叠

#### 4. Agent 内存

**三层作用域**:
```
User (最广泛) → Project → Local (最窄)
```

**Snapshot 机制**:
- 通过版本控制分发 agent 内存
- 支持 initialize/prompt-update 动作
- 允许个人修改

**特性**:
- 启用内存的 agent 自动注入文件工具
- 路径验证防止目录遍历
- 颜色分配用于 UI 识别

#### 5. Agent ID 格式

```typescript
// Subagents
`a${16hex}`  // 例如: a3f2c1b4d5e6f7a8

// Teammates
`${agentName}@${teamName}`  // 例如: researcher@my-project

// Request IDs
`${type}-${timestamp}@${agentId}`
```

### 架构模式提取

#### 模式 1: 文件级锁
```typescript
const LOCK_OPTIONS = {
  retries: { retries: 30, minTimeout: 5, maxTimeout: 100 }
}
// 支持 ~10 并发 agents
```

#### 模式 2: Signal 通知
```typescript
const tasksUpdated = createSignal()
export const onTasksUpdated = tasksUpdated.subscribe
export function notifyTasksUpdated() {
  tasksUpdated.emit()
}
```

#### 模式 3: 上下文隔离
```typescript
function createSubagentContext(parent: Context): Context {
  return {
    ...parent,
    setAppState: isAsync ? noop : parent.setAppState,
    abortController: isAsync ? new AbortController() : parent.abortController
  }
}
```

#### 模式 4: 双层工具过滤
```
MCP tools → ExitPlanMode → ALL_AGENT_DISALLOWED_TOOLS →
CUSTOM_AGENT_DISALLOWED_TOOLS → ASYNC_AGENT_ALLOWED_TOOLS
```

### 复用经验

1. **文件存储 + 锁** - 简单可靠，支持多进程
2. **High water mark** - 避免 ID 重用
3. **Signal 模式** - 轻量级 pub/sub
4. **上下文克隆** - 显式控制共享/隔离
5. **团队文件** - 文件系统作为协调层

### 有趣彩蛋
- Task 锁配置预算为 ~10 并发 agents
- AsyncLocalStorage 用于 analytics 归因
- Agent 颜色分配是简单的 Map-based 系统
- Coordinator UI 用于引导后台 agents
- `isTodoV2Enabled()` 在非交互模式强制启用

## 进度总结

已完成模块:
- ✅ Module 1: Entry & Core (15 文件)
- ✅ Module 2: Tools System (21 文件)
- ✅ Module 3: Commands System (10 文件)
- ✅ Module 4: Components System (10 文件)
- ✅ Module 5: Skills System (10 文件)
- ✅ Module 6: Tasks & Agents (15 文件)

总计: ~81 文件已分析

剩余模块:
- Module 7: Query & State
- Module 8: Services Layer
- Module 9: Utils & Helpers
- Module 10: Bridge & Remote
- Module 11: Hooks & Keybindings
- Module 12: Integration & Synthesis

