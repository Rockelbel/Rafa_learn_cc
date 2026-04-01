# Tool Execution Services Analysis

## Overview

This document analyzes the Claude Code tool execution architecture, covering four core files:
- `toolExecution.ts` - Core tool execution pipeline
- `toolOrchestration.ts` - Tool batching and concurrency management
- `StreamingToolExecutor.ts` - Streaming tool execution with concurrency control
- `toolHooks.ts` - Pre/Post tool use hook implementations

---

## 1. Tool Execution Pipeline (`toolExecution.ts`)

### 1.1 Core Function: `runToolUse`

The main entry point for executing a single tool use. Pipeline flow:

```
1. Tool Resolution → 2. Input Validation → 3. Pre-Tool Hooks → 4. Permission Check →
5. Tool Execution → 6. Post-Tool Hooks → 7. Result Processing
```

**Key Steps:**

1. **Tool Lookup**: Finds tool by name, with fallback for deprecated tool aliases
2. **Abort Check**: Early exit if query was cancelled
3. **Permission & Execution**: Delegates to `streamedCheckPermissionsAndCallTool()`

### 1.2 Permission and Execution Flow (`checkPermissionsAndCallTool`)

```
Input Validation (Zod) → Pre-Tool Hooks → Permission Resolution →
Tool.call() → Post-Tool Hooks → Result Mapping
```

**Input Validation:**
- Schema validation via `tool.inputSchema.safeParse()`
- Custom tool validation via `tool.validateInput()`
- Special handling for deferred tools (schema not sent hint)

**Permission Resolution:**
- Hooks can provide permission decisions (`allow`/`deny`/`ask`)
- Falls back to `canUseTool()` for interactive permission dialogs
- Rule-based permissions (`settings.json`) are checked even if hook approves

### 1.3 Progress Reporting

Tools report progress via callbacks:
```typescript
progress => onToolProgress({
  toolUseID: progress.toolUseID,
  data: progress.data,
})
```

Progress is wrapped in `ProgressMessage<HookProgress>` and streamed to the UI.

### 1.4 Error Handling

**Error Classification:** `classifyToolError()` categorizes errors for telemetry:
- `TelemetrySafeError` - Uses vetted `telemetryMessage`
- Node.js fs errors - Logs errno code (ENOENT, EACCES, etc.)
- Known error types - Uses stable `.name` property
- Fallback - "Error" (avoids minified names)

**Error Types Handled:**
- `InputValidationError` - Zod schema failures
- `McpAuthError` - MCP authentication failures (updates client status)
- `AbortError` - User cancellation
- `ShellError` - Bash/PowerShell execution errors

### 1.5 Context Modifiers

Tools and hooks can modify the `ToolUseContext` via `contextModifier`:
```typescript
{
  toolUseID: string,
  modifyContext: (context: ToolUseContext) => ToolUseContext
}
```

Applied after tool completion to update context for subsequent operations.

---

## 2. Tool Orchestration (`toolOrchestration.ts`)

### 2.1 Core Function: `runTools`

Manages execution of multiple tool calls with intelligent batching.

### 2.2 Concurrency Model

**Partitioning Strategy:**

Tool calls are partitioned into batches using `partitionToolCalls()`:
- **Concurrent-safe batch**: Multiple read-only tools run in parallel
- **Non-concurrent batch**: Single tool runs exclusively

**Concurrency Rules:**
```typescript
// Can execute if:
executingTools.length === 0 ||
(isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
```

**Max Concurrency:** Controlled by `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default: 10)

### 2.3 Serial vs Concurrent Execution

**Serial Execution (`runToolsSerially`):**
- Tools execute one at a time
- Context modifiers applied immediately after each tool
- Maintains strict ordering

**Concurrent Execution (`runToolsConcurrently`):**
- Uses `all()` utility to run async generators in parallel
- Context modifiers queued and applied after batch completes
- Maintains order within each tool's results

### 2.4 Execution Flow Diagram

```
toolUseMessages
    ↓
partitionToolCalls() → [Batch1 (concurrent), Batch2 (serial), Batch3 (concurrent)...]
    ↓
For each batch:
    - If concurrent: runToolsConcurrently() → apply context modifiers
    - If serial: runToolsSerially() → apply context modifiers per tool
    ↓
Yield MessageUpdate with updated context
```

---

## 3. Streaming Tool Executor (`StreamingToolExecutor.ts`)

### 3.1 Purpose

Manages tool execution during streaming responses from the LLM. Tools start executing as soon as they stream in, not waiting for the complete response.

### 3.2 Key Data Structure: `TrackedTool`

```typescript
type TrackedTool = {
  id: string                    // Tool use ID
  block: ToolUseBlock           // Original tool use block
  assistantMessage: AssistantMessage
  status: ToolStatus            // 'queued' | 'executing' | 'completed' | 'yielded'
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]    // Immediate-yield progress messages
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 3.3 Concurrency Control

**Abort Controller Hierarchy:**
```
toolUseContext.abortController (parent)
    ↓
siblingAbortController (child)
    ↓
toolAbortController (per-tool child)
```

**Cascade Behavior:**
- Bash errors cancel sibling tools (via `siblingAbortController`)
- Permission dialog rejection bubbles up to parent
- User interruption (`ESC`) handled per-tool based on `interruptBehavior`

### 3.4 Progress Streaming

Progress messages are yielded immediately via `pendingProgress` queue:

```typescript
// In executeTool()
if (update.message.type === 'progress') {
  tool.pendingProgress.push(update.message)
  if (this.progressAvailableResolve) {
    this.progressAvailableResolve()
    this.progressAvailableResolve = undefined
  }
}

// In getCompletedResults()
while (tool.pendingProgress.length > 0) {
  const progressMessage = tool.pendingProgress.shift()!
  yield { message: progressMessage, newContext: this.toolUseContext }
}
```

### 3.5 Error Handling and Sibling Cancellation

**Bash Tool Special Case:**
```typescript
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

Only Bash errors trigger sibling cancellation because commands often have implicit dependencies.

**Synthetic Error Messages:**
Generated for tools cancelled due to:
- `sibling_error` - Parallel tool call errored
- `user_interrupted` - User rejected edit
- `streaming_fallback` - Streaming fallback, execution discarded

### 3.6 Result Ordering

Results yielded in order received (model order) for non-concurrent tools:
```typescript
if (tool.status === 'executing' && !tool.isConcurrencySafe) {
  break  // Stop yielding until this tool completes
}
```

### 3.7 Streaming Fallback (`discard()`)

When streaming fails and falls back to non-streaming:
```typescript
discard(): void {
  this.discarded = true
}
```
- Queued tools won't start
- In-progress tools receive synthetic errors
- Prevents partial results from failed streaming attempt

---

## 4. Tool Hooks (`toolHooks.ts`)

### 4.1 Hook Types

**Pre-Tool Use Hooks (`runPreToolUseHooks`):**
- Execute before tool permission check
- Can modify input via `updatedInput`
- Can provide permission decisions (`allow`/`deny`/`ask`)
- Can prevent continuation
- Can add additional context

**Post-Tool Use Hooks (`runPostToolUseHooks`):**
- Execute after successful tool execution
- Can modify MCP tool output
- Can prevent continuation
- Can add additional context

**Post-Tool Use Failure Hooks (`runPostToolUseFailureHooks`):**
- Execute after failed tool execution
- Same capabilities as Post-Tool Use hooks

### 4.2 Hook Execution Flow

```typescript
for await (const result of executePreToolHooks(...)) {
  switch (result.type) {
    case 'message':           // Progress or attachment message
    case 'hookPermissionResult':  // Permission decision
    case 'hookUpdatedInput':  // Input modification
    case 'preventContinuation':   // Stop after this tool
    case 'stopReason':        // Reason for stopping
    case 'additionalContext': // Extra context
    case 'stop':              // Abort hook execution
  }
}
```

### 4.3 Permission Decision Resolution (`resolveHookPermissionDecision`)

**Decision Flow:**

1. **Hook returns `allow`:**
   - If tool requires interaction and hook didn't provide `updatedInput` → prompt user
   - Check rule-based permissions (deny/ask rules still apply despite hook approval)
   - Return hook decision if no rule conflict

2. **Hook returns `deny`:**
   - Immediate denial

3. **Hook returns `ask` or no decision:**
   - Normal permission flow with optional `forceDecision`

### 4.4 Hook Output Types

**Pre-Tool Use:**
```typescript
| { type: 'message', message: MessageUpdateLazy }
| { type: 'hookPermissionResult', hookPermissionResult: PermissionResult }
| { type: 'hookUpdatedInput', updatedInput: Record<string, unknown> }
| { type: 'preventContinuation', shouldPreventContinuation: boolean }
| { type: 'stopReason', stopReason: string }
| { type: 'additionalContext', message: MessageUpdateLazy }
| { type: 'stop' }
```

**Post-Tool Use:**
```typescript
| MessageUpdateLazy<AttachmentMessage | ProgressMessage<HookProgress>>
| { updatedMCPToolOutput: Output }
```

### 4.5 Hook Error Handling

Errors in hooks are caught and logged:
- Analytics event: `tengu_pre_tool_hook_error`, `tengu_post_tool_hook_error`
- Attachment message: `hook_error_during_execution`
- Hook execution continues for remaining hooks

### 4.6 Hook Cancellation

When query is aborted during hook execution:
- Analytics event: `tengu_pre_tool_hooks_cancelled`, `tengu_post_tool_hooks_cancelled`
- Attachment message: `hook_cancelled`
- Hook execution stops for remaining hooks

---

## 5. Integration with Query Loop

### 5.1 Tool Execution Context

`ToolUseContext` provides execution context:
```typescript
type ToolUseContext = {
  messages: Message[]
  options: ToolUseContextOptions
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppStateFn
  setInProgressToolUseIDs: (update: SetUpdate<string>) => void
  setHasInterruptibleToolInProgress?: (value: boolean) => void
  toolDecisions?: Map<string, DecisionInfo>
  queryTracking?: QueryTrackingInfo
  agentId?: string
  requireCanUseTool?: boolean
  preserveToolUseResults?: boolean
  requestPrompt?: string
}
```

### 5.2 In-Progress Tool Tracking

- `setInProgressToolUseIDs()` tracks currently executing tools
- Used by UI to show loading states
- Cleared via `markToolUseAsComplete()` after each tool

### 5.3 Interruptibility

`setHasInterruptibleToolInProgress()` indicates if current tools can be interrupted:
- `true` if all executing tools have `interruptBehavior: 'cancel'`
- `false` if any tool has `interruptBehavior: 'block'`

### 5.4 Abort Signal Handling

**Abort Reasons:**
- User typed new message (`interrupt`)
- Tool error (Bash errors cascade to siblings)
- Permission dialog rejection
- Streaming fallback

**Behavior:**
- Read/Write tools: typically `interruptBehavior: 'cancel'`
- Bash tools: sibling cancellation on error
- MCP tools: handled per-tool

---

## 6. Key Architecture Patterns

### 6.1 Generator-Based Streaming

All execution functions return `AsyncGenerator<MessageUpdateLazy>`:
- Enables incremental result delivery
- Supports progress messages interleaved with results
- Allows cancellation at any yield point

### 6.2 Context Modifiers

Immutable context updates via functions:
```typescript
(context: ToolUseContext) => ToolUseContext
```

Applied after tool/hook completion to maintain referential transparency.

### 6.3 Concurrent-Safe Tool Classification

Tools declare concurrency safety via `isConcurrencySafe(input)`:
- **Safe**: Read operations (Read, Glob, Grep)
- **Unsafe**: Write operations (Edit, Write, Bash)

### 6.4 Error Telemetry

Comprehensive error logging:
- Tool name, duration, input validation errors
- MCP server type and base URL
- Query chain ID and depth
- Error classification (not raw error messages)

### 6.5 Permission Layering

Multiple permission sources (in order of precedence):
1. Pre-tool hooks (can allow/deny/ask)
2. Rule-based permissions (settings.json)
3. Interactive permission dialog
4. Auto-mode classifiers

---

## 7. Summary

The tool execution architecture follows these principles:

1. **Streaming-First**: Tools execute as they stream in, not batched
2. **Concurrency Control**: Smart partitioning based on tool safety
3. **Hook Extensibility**: Pre/post hooks for customization
4. **Progress Visibility**: Real-time progress reporting
5. **Graceful Degradation**: Fallback handling for streaming failures
6. **Security**: Layered permissions with rule enforcement
7. **Observability**: Comprehensive telemetry and error classification
