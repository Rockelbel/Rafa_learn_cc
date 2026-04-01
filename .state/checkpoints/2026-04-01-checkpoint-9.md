# Checkpoint 9 - 2026-04-01

## Module 7: Query & State - 已完成

### 分析文件 (8)
1. ✅ query.ts - Query 执行核心 (3000+ 行)
2. ✅ state/store.ts - Store 实现
3. ✅ state/AppStateStore.ts - App 状态
4. ✅ state/selectors.ts - Selectors
5. ✅ state/onChangeAppState.ts - 变更处理
6. ✅ types/message.ts - 消息类型
7. ✅ utils/messages.ts - 消息工具
8. ✅ utils/stream.ts - 流实现

### 生成的笔记文件
- raw_notes/query_state/query_system.md - Query 系统
- raw_notes/query_state/state_management.md - 状态管理
- raw_notes/query_state/message_system.md - 消息系统
- raw_notes/query_state/streaming.md - 流系统

### 关键发现摘要

#### 1. Query 执行流程 (query.ts)

```
1. 消息准备 → 2. API 请求 → 3. 响应流式处理 → 4. 工具执行 → 5. 结果集成 → 6. 状态更新
```

**关键组件**:
- `StreamingMessageAccumulator` - 流式内容累积
- `runTools()` - 工具执行编排
- `normalizeMessagesForAPI()` - API 消息规范化
- `queryModel()` - API 调用

#### 2. Store 模式 (state/store.ts)

**极简实现**:
```typescript
export function createStore<T>(initialState: T): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const next = updater(state)
      if (Object.is(next, state)) return
      state = next
      listeners.forEach(l => l())
    },
    subscribe: (listener) => { ... }
  }
}
```

**特性**:
- 函数式更新 `(prev) => next`
- `Object.is()` 变更检测
- Set-based 订阅

#### 3. AppState 结构 (state/AppStateStore.ts)

**庞大单体状态** (~450 行):
- Settings (配置)
- UI (界面状态)
- Bridge (桥接模式)
- MCP (服务器连接)
- Plugins (插件状态)
- Tasks (任务状态)
- Notifications (通知)
- Permissions (权限)

**任务存储**:
```typescript
tasks: { [taskId: string]: TaskState }
```

#### 4. Message 类型系统 (types/message.ts)

**消息类型层次**:
```
Message
├── UserMessage (用户输入、工具结果)
├── AssistantMessage (模型响应)
│   ├── text
│   ├── tool_use
│   └── thinking
├── SystemMessage (20+ 子类型)
├── ProgressMessage (进度)
└── AttachmentMessage (附件)
```

**消息规范化**:
- 多 block 消息拆分为单独消息
- UUID 确定性派生 `deriveUUID(parentUUID, index)`
- 扁平消息列表用于 UI 渲染

#### 5. 流系统 (utils/stream.ts)

**Stream<T> 类**:
- Async iterator 接口
- 基于队列的缓冲
- 支持 abort

**API 流式**:
- SSE 事件处理
- Event types: `message_start`, `content_block_delta`, `message_stop`
- Content accumulation
- 90s idle timeout watchdog

**工具流式**:
- `StreamingToolExecutor` 并发执行
- 进度消息流式
- 顺序保证

#### 6. Selectors (state/selectors.ts)

- 纯函数接受最小状态形状 `Pick<>`
- 返回 discriminated unions 用于类型安全路由
- 例如: `getActiveAgentForInput()`

#### 7. 变更流 (onChangeAppState.ts)

```
setState → reference check → state update → onChangeAppState
→ subscriber notification → component re-render
```

**副作用集中化**:
- CCR 同步
- 持久化
- 缓存清理
- 权限模式传播

### 架构模式提取

#### 模式 1: 极简 Store
```typescript
const store = createStore(initialState)
store.subscribe(() => rerender())
store.setState(prev => ({ ...prev, changed }))
```

#### 模式 2: 流式 Accumulator
```typescript
for await (const chunk of stream) {
  accumulator.add(chunk)
  yield partialResult
}
```

#### 模式 3: 消息规范化
```typescript
const normalized = normalizeMessagesForAPI(messages)
const forUI = reorderMessagesInUI(normalized)
```

#### 模式 4: 函数式状态更新
```typescript
setState(prev => ({
  ...prev,
  nested: { ...prev.nested, changed: true }
}))
```

### 复用经验

1. **极简 Store** - 无需 Redux，自定义 35 行足够
2. **Object.is()** - 精确变更检测
3. **Deterministic UUID** - 从 parent + index 派生
4. **流式累积** - 渐进式处理大响应
5. **Selective Pick** - Selector 只依赖所需状态

### 有趣彩蛋
- `DeepImmutable` 包装器用于类型安全
- Tasks 以可变 keyed object 存储 (性能)
- 90 秒 idle timeout watchdog
- SSE 事件类型完整枚举
- `normalizeMessages` 与 `normalizeMessagesForAPI` 分离

## 进度总结

已完成模块:
- ✅ Module 1: Entry & Core (15 文件)
- ✅ Module 2: Tools System (21 文件)
- ✅ Module 3: Commands System (10 文件)
- ✅ Module 4: Components System (10 文件)
- ✅ Module 5: Skills System (10 文件)
- ✅ Module 6: Tasks & Agents (15 文件)
- ✅ Module 7: Query & State (8 文件)

总计: ~89 文件已分析

剩余模块:
- Module 8: Services Layer
- Module 9: Utils & Helpers
- Module 10: Bridge & Remote
- Module 11: Hooks & Keybindings
- Module 12: Integration & Synthesis

