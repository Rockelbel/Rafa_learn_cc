# Checkpoint 4 - 2026-04-01

## Module 2: Tools System - 已完成

### 分析文件 (13)
1. ✅ Tool.ts (793 行) - 核心 Tool 接口定义
2. ✅ tools.ts (390 行) - 工具注册和管理
3. ✅ tools/FileReadTool/FileReadTool.ts - 文件读取工具
4. ✅ tools/FileEditTool/FileEditTool.ts - 文件编辑工具
5. ✅ tools/FileWriteTool/FileWriteTool.ts - 文件写入工具
6. ✅ tools/GlobTool/GlobTool.ts - 文件搜索工具
7. ✅ tools/GrepTool/GrepTool.ts - 内容搜索工具
8. ✅ tools/BashTool/BashTool.tsx - Bash 执行工具
9. ✅ tools/BashTool/bashSecurity.ts - Bash 安全检查
10. ✅ tools/BashTool/bashPermissions.ts - Bash 权限系统
11. ✅ tools/BashTool/destructiveCommandWarning.ts - 破坏性命令警告
12. ✅ tools/BashTool/shouldUseSandbox.ts - 沙盒决策逻辑
13. ✅ tools/AgentTool/AgentTool.tsx - Agent 工具
14. ✅ tools/AgentTool/runAgent.ts - Agent 运行核心
15. ✅ tools/AgentTool/forkSubagent.ts - Fork 子代理
16. ✅ tools/AgentTool/resumeAgent.ts - 恢复 Agent
17. ✅ tools/AgentTool/builtInAgents.ts - 内置 Agent 类型
18. ✅ services/tools/toolExecution.ts - 工具执行管道
19. ✅ services/tools/toolOrchestration.ts - 工具并发编排
20. ✅ services/tools/StreamingToolExecutor.ts - 流式执行器
21. ✅ services/tools/toolHooks.ts - 工具 Hook 系统

### 生成的笔记文件
- raw_notes/tools/tool_infrastructure.md - 工具基础设施分析
- raw_notes/tools/file_tools.md - 文件工具分析
- raw_notes/tools/bash_security.md - Bash 安全系统分析
- raw_notes/tools/agent_system.md - Agent 系统分析
- raw_notes/tools/tool_execution_services.md - 工具执行服务分析

### 关键发现摘要

#### 1. Tool 接口设计 (Tool.ts)
- 50+ 属性/方法的完整接口
- 泛型设计: `Tool<Input, Output, P>` 支持类型安全
- `buildTool()` 工厂模式提供 fail-closed 默认实现
- 类型级展开: `BuiltTool<D>` 实现编译时类型合并
- 三层渲染: model-facing, user UI, search indexing

#### 2. 工具注册系统 (tools.ts)
- 条件导入: `feature()` 和 `process.env` 控制
- 懒加载: `require()` 打破循环依赖
- 权限过滤: `filterToolsByDenyRules()` 运行时过滤
- 60+ 工具通过 `getAllBaseTools()` 注册

#### 3. 文件工具模式
- Zod `lazySchema()` 懒初始化
- `z.strictObject()` 严格验证
- 两阶段验证: I/O 前检查 vs I/O 后检查
- UNC 路径保护 (防 NTLM 凭证泄露)
- 文件大小限制 (防 OOM)

#### 4. Bash 安全系统
- 20+ 独立验证器
- 解析器差异防御 (CR, 反斜杠转义操作符, brace expansion)
- Zsh 危险命令拦截 (zmodload, sysopen, ztcp)
- 标志混淆检测 (ANSI-C quoting, 引号拼接)
- 三层权限模式: auto/plan/bypassPermissions

#### 5. Agent 系统
- 两种 spawn 模式: normal vs fork (实验性)
- Fork 模式支持 prompt cache 共享
- 内置 Agent 类型: explore, plan, verification, claude-code-guide
- 上下文隔离: `createSubagentContext()`
- 后台 Agent 支持: `runAsyncAgentLifecycle()`

#### 6. 工具执行管道
- 7 阶段流程: Tool Resolution → Input Validation → Pre-Tool Hooks → Permission Check → Tool Execution → Post-Tool Hooks → Result Processing
- 生成器流式: 增量结果返回
- 并发控制: `isConcurrencySafe()` 分区，最大 10 并发
- Hook 系统: PreToolUse, PostToolUse, PostToolUseFailure
- 中止控制器层级: parent → sibling → per-tool

### 架构模式提取

#### 模式 1: 类型安全工厂 (buildTool)
```typescript
const TOOL_DEFAULTS = { isEnabled: () => true, ... }
type BuiltTool<D> = Omit<D, DefaultableKeys> & { [K in DefaultableKeys]-?: ... }
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D>
```

#### 模式 2: 延迟工具加载 (ToolSearch)
- 大工具标记 `shouldDefer: true`
- 初始请求不包含完整 schema
- ToolSearch 触发后加载完整定义
- 节省 context window

#### 模式 3: 多层权限检查
```
1. validateInput() - 工具级验证
2. PreToolUse Hooks - 钩子可拒绝
3. checkPermissions() - 规则匹配
4. 交互式确认 - 用户最终决定
```

#### 模式 4: 并发安全分区
```typescript
const batches = partition(tools, t => t.isConcurrencySafe(input))
// 安全工具: 并行执行 (max 10)
// 不安全工具: 串行执行
```

### 复用经验

1. **工具接口标准化** - 60+ 工具共享统一接口，新增工具成本极低
2. **Fail-Closed 默认** - 默认不安全，必须显式声明安全属性
3. **渐进式权限** - 从自动到询问到拒绝的渐进式安全模型
4. **Hook 扩展点** - 不修改核心代码即可拦截和修改工具行为
5. **类型级编程** - TypeScript 条件类型实现编译时验证

### 有趣彩蛋
- `REPL_ONLY_TOOLS` - REPL 模式专用工具隐藏
- `OVERFLOW_TEST_TOOL` - 溢出测试工具 (feature flag)
- `CtxInspectTool` - 上下文检查工具 (调试专用)
- 内置 Agent 的幽默描述: "Fast agent specialized for exploring codebases"

## 下一步
开始 Module 3: Commands System

