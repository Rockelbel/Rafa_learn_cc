# Claude Code Tool System Infrastructure Analysis

## Files Analyzed
- `/home/claudeuser/ClaudeCode/Tool.ts` (793 lines) - Core Tool type definitions
- `/home/claudeuser/ClaudeCode/tools.ts` (390 lines) - Tool registration and management

---

## Architecture Overview

The Claude Code Tool System is a sophisticated, type-safe infrastructure for defining, registering, and executing AI-accessible tools. It combines advanced TypeScript type-level programming with a clean factory pattern to manage 60+ tools across multiple categories.

### Core Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TOOL DEFINITION LAYER                            │
├─────────────────────────────────────────────────────────────────────────┤
│  Tool Interface (50+ properties/methods)                                │
│  ├── Core: name, call(), description(), inputSchema, outputSchema       │
│  ├── Capability: aliases, searchHint, isEnabled, isConcurrencySafe      │
│  ├── Safety: isReadOnly, isDestructive, checkPermissions                │
│  ├── Lifecycle: validateInput(), interruptBehavior                      │
│  ├── Rendering: renderToolResultMessage, renderToolUseMessage, etc.     │
│  └── Deferred: shouldDefer, alwaysLoad (ToolSearch integration)         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        FACTORY LAYER (buildTool)                        │
├─────────────────────────────────────────────────────────────────────────┤
│  TOOL_DEFAULTS (fail-closed defaults)                                   │
│  ├── isEnabled: () => true                                              │
│  ├── isConcurrencySafe: () => false                                     │
│  ├── isReadOnly: () => false                                            │
│  ├── isDestructive: () => false                                         │
│  ├── checkPermissions: () => Promise.resolve({behavior: 'allow'})       │
│  ├── toAutoClassifierInput: () => ''                                    │
│  └── userFacingName: () => ''                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    REGISTRATION LAYER (tools.ts)                        │
├─────────────────────────────────────────────────────────────────────────┤
│  getAllBaseTools(): Tools[]                                             │
│  ├── Conditional imports via feature() and process.env                  │
│  ├── Lazy requires to break circular dependencies                       │
│  └── Environment-specific tool filtering                                │
│                                                                         │
│  getTools(permissionContext): Tools[]                                   │
│  ├── Simple mode filtering (--simple flag)                              │
│  ├── REPL mode handling (hides primitive tools)                         │
│  └── Deny rule filtering via filterToolsByDenyRules()                   │
│                                                                         │
│  assembleToolPool(): Combines built-in + MCP tools                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key Design Patterns

### 1. The `buildTool()` Factory Pattern

**Purpose**: Provide fail-closed defaults while preserving full type safety.

```typescript
// ToolDef vs Tool distinction:
// - ToolDef: Optional properties (callers can omit defaults)
// - Tool: Complete interface (always has all properties)

type ToolDef<Input, Output, P> = Omit<Tool, DefaultableToolKeys> &
  Partial<Pick<Tool, DefaultableToolKeys>>

type DefaultableToolKeys =
  | 'isEnabled'
  | 'isConcurrencySafe'
  | 'isReadOnly'
  | 'isDestructive'
  | 'checkPermissions'
  | 'toAutoClassifierInput'
  | 'userFacingName'

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D>
```

**Key insight**: The factory uses TypeScript's conditional types to merge defaults at the type level, mirroring the runtime spread `{ ...TOOL_DEFAULTS, ...def }`.

### 2. Type-Level Spread Implementation

The `BuiltTool<D>` type is a sophisticated type-level spread that:
- Preserves exact arity and literal types from the definition
- Fills in defaults only for omitted properties
- Maintains strict type safety across all 60+ tools

```typescript
type BuiltTool<D> = Omit<D, DefaultableToolKeys> & {
  [K in DefaultableToolKeys]-?: K extends keyof D
    ? undefined extends D[K]
      ? ToolDefaults[K]
      : D[K]
    : ToolDefaults[K]
}
```

The `-?` (definite assignment) modifier ensures all defaultable keys become required in the final Tool.

### 3. Tool Lifecycle Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    TOOL EXECUTION PIPELINE                      │
├────────────────────────────────────────────────────────────────┤
│  1. VALIDATION PHASE                                            │
│     └── validateInput?(input, context): Promise<ValidationResult>│
│         ├── Returns { result: true } or error with message      │
│         └── Runs BEFORE permission checks                       │
│                                                                 │
│  2. PERMISSION PHASE                                            │
│     └── checkPermissions(input, context): Promise<PermissionResult>│
│         ├── Returns { behavior: 'allow' | 'deny', updatedInput }│
│         └── Integrates with permission system (allow/ask/deny)  │
│                                                                 │
│  3. EXECUTION PHASE                                             │
│     └── call(args, context, canUseTool, parentMessage, onProgress)│
│         ├── Returns ToolResult<T> with data + optional messages │
│         ├── Supports progress callbacks for streaming UI        │
│         └── Can modify context via contextModifier              │
│                                                                 │
│  4. RENDERING PHASE                                             │
│     ├── renderToolUseMessage(input, options): React.ReactNode   │
│     ├── renderToolResultMessage(content, progress, options)     │
│     ├── renderToolUseProgressMessage(progress, options)         │
│     └── mapToolResultToToolResultBlockParam(content, toolUseID) │
└────────────────────────────────────────────────────────────────┘
```

### 4. ToolUseContext: The Dependency Injection Container

With 40+ properties, `ToolUseContext` serves as a comprehensive context object:

**Core Dependencies**:
- `options`: Commands, tools, model config, MCP resources
- `abortController`: Cancellation signal
- `getAppState/setAppState`: Global state access
- `messages`: Current conversation history

**UI Integration**:
- `setToolJSX`: Render tool-specific UI components
- `addNotification`: System notifications
- `sendOSNotification`: Native OS notifications
- `setStreamMode`: Control loading spinner

**Progress & Tracking**:
- `setInProgressToolUseIDs`: Track active tool calls
- `onCompactProgress`: Context compaction callbacks
- `toolDecisions`: Map of tool permission decisions

**Advanced Features**:
- `handleElicitation`: MCP URL elicitation handler
- `requestPrompt`: Interactive user prompt factory
- `contentReplacementState`: Tool result budget tracking
- `renderedSystemPrompt`: Cache-sharing for subagents

---

## Type System Techniques

### 1. Generic Tool Definition with Progress Types

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,  // Zod schema type
  Output = unknown,                      // Return data type
  P extends ToolProgressData = ToolProgressData,  // Progress event type
> = {
  call(args: z.infer<Input>, context: ToolUseContext, ...): Promise<ToolResult<Output>>
  // Each tool can define its own progress data structure
  renderToolUseProgressMessage?(progressMessages: ProgressMessage<P>[], ...): React.ReactNode
}
```

This allows each tool to have strongly-typed progress events while sharing the same interface.

### 2. Discriminated Union Types for Tool Results

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}
```

### 3. Conditional Type for Validation Results

```typescript
export type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

### 4. DeepImmutable Wrapper for Permission Context

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  // ... 10+ more properties
}>
```

Ensures permission state cannot be accidentally mutated.

---

## Deferred Tool Loading Pattern (ToolSearch)

### Motivation
Reduce context window usage by deferring tool definitions until needed.

### Implementation

```typescript
// In Tool interface:
readonly shouldDefer?: boolean  // Tool requires ToolSearch before use
readonly alwaysLoad?: boolean   // Tool is never deferred
readonly searchHint?: string     // Keywords for finding this tool

// ToolSearch tool (fetched dynamically):
// - Query: "notebook jupyter" → finds NotebookEditTool
// - Returns: Full tool schema for the matched tool
```

### Deferred Tool Lifecycle
```
1. Initial prompt contains only ToolSearch (not deferred tools)
2. Model calls ToolSearch with query
3. System returns matched tool's full schema
4. Model can now call the deferred tool
5. Tool executes normally
```

### Key Files
- `ToolSearchTool.ts`: The tool that enables deferred loading
- `utils/toolSearch.ts`: `isToolSearchEnabledOptimistic()` check

---

## Tool Permission System Integration

### Permission Context Structure

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode  // 'default' | 'bypass' | 'auto'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean  // For background agents
  awaitAutomatedChecksBeforeDialog?: boolean  // Coordinator workers
}>
```

### Deny Rule Filtering

```typescript
export function filterToolsByDenyRules<T extends { name: string; mcpInfo?: {...} }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

Rules can target:
- Tool name: `"Bash"` - blocks all Bash tool calls
- MCP server prefix: `"mcp__server"` - blocks all tools from that server
- Specific MCP tool: `"mcp__server__tool"`

---

## Tool vs ToolDef: The Key Distinction

| Aspect | ToolDef | Tool |
|--------|---------|------|
| **Purpose** | Input to factory | Output from factory |
| **Completeness** | Partial (omits defaults) | Complete (has all properties) |
| **Usage** | Tool authors write this | System consumes this |
| **Type Safety** | Validates required methods | Guarantees all methods present |

```typescript
// Tool author writes ToolDef:
const MyToolDef = {
  name: 'MyTool',
  call: async (args, context) => ({ data: 'result' }),
  // isEnabled, isConcurrencySafe, etc. are optional
}

// Factory produces complete Tool:
const MyTool = buildTool(MyToolDef)
// Now has all defaults: isEnabled, isConcurrencySafe, isReadOnly, etc.
```

---

## Interesting Implementation Details

### 1. Circular Dependency Breaking via Lazy Requires

```typescript
// Instead of top-level import (causes cycle):
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool

// Called at runtime when needed:
...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
```

### 2. Feature Flag System Integration

Uses `feature('FLAG_NAME')` from `bun:bundle` for conditional tool inclusion:

```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### 3. Environment-Based Tool Filtering

```typescript
// Simple mode: only Bash, Read, Edit
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  return [BashTool, FileReadTool, FileEditTool]
}

// Ant-only tools
...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : [])

// Test-only tools
...(process.env.NODE_ENV === 'test' ? [TestingPermissionTool] : [])
```

### 4. Tool Aliases for Backwards Compatibility

```typescript
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}
```

### 5. MCP Tool Integration

MCP tools are treated as first-class citizens with special metadata:

```typescript
readonly isMcp?: boolean
mcpInfo?: { serverName: string; toolName: string }
```

The `assembleToolPool()` function merges built-in and MCP tools:
```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // Built-ins take precedence on name conflict
  return uniqBy([...builtInTools, ...allowedMcpTools], 'name')
}
```

### 6. Tool Name Matching Utilities

```typescript
// Primary lookup by name or alias
export function findToolByName(tools: Tools, name: string): Tool | undefined

// Used in REPL mode to hide primitive tools when REPL is enabled
const REPL_ONLY_TOOLS = new Set(['Bash', 'Read', 'Edit', ...])
```

### 7. Interrupt Behavior Control

Tools can declare how they handle user interruption:

```typescript
interruptBehavior?(): 'cancel' | 'block'
// 'cancel' - stop tool and discard result
// 'block' - keep running, new message waits (default)
```

### 8. Search/Read Command Classification

For UI collapsing/condensing:

```typescript
isSearchOrReadCommand?(input): {
  isSearch: boolean   // grep, find, glob
  isRead: boolean     // cat, head, tail, file read
  isList?: boolean    // ls, tree, du
}
```

---

## Tool Rendering Architecture

The system separates model-facing from user-facing representations:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MODEL-FACING OUTPUT                          │
├─────────────────────────────────────────────────────────────────┤
│  mapToolResultToToolResultBlockParam(content, toolUseID)        │
│  └── Returns: ToolResultBlockParam for API                      │
│      ├── tool_use_id                                            │
│      ├── content (string | array of text/image blocks)          │
│      └── _meta (MCP metadata)                                   │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    USER-FACING OUTPUT                           │
├─────────────────────────────────────────────────────────────────┤
│  renderToolResultMessage(content, progress, options)            │
│  └── Returns: React.ReactNode for UI                            │
│      ├── Condensed view (style: 'condensed')                    │
│      ├── Verbose view (verbose: true)                           │
│      └── Transcript mode (isTranscriptMode: true)               │
│                                                                 │
│  extractSearchText?(output): string                             │
│  └── For transcript search indexing                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lessons for Other Agent Systems

### 1. **Fail-Closed Defaults Matter**
The `TOOL_DEFAULTS` pattern ensures safety by default:
- `isConcurrencySafe: false` - assumes unsafe until proven
- `isReadOnly: false` - assumes writes until declared
- `isDestructive: false` - requires explicit opt-in for dangerous ops

### 2. **Type-Level Programming Enables Safety**
The `BuiltTool<D>` type demonstrates how TypeScript's type system can mirror runtime behavior, ensuring:
- All tools have complete interfaces
- Authors only define what differs from defaults
- Refactoring is type-safe across 60+ tools

### 3. **Context as Dependency Injection**
The 40+ property `ToolUseContext` centralizes dependencies without globals:
- State access via functions (not direct references)
- UI callbacks for tool-specific rendering
- Lifecycle hooks for progress and cancellation

### 4. **Progressive Tool Loading**
The deferred tool pattern (`shouldDefer`, `ToolSearch`) solves context window constraints:
- Only load tools the model explicitly requests
- Maintain discoverability via `searchHint` keywords
- Never load unused tools in the prompt

### 5. **Separation of Concerns in Rendering**
Three distinct output paths:
- **API/Model**: `mapToolResultToToolResultBlockParam()`
- **User UI**: `renderToolResultMessage()`
- **Search/Indexing**: `extractSearchText()`

### 6. **Permission System Integration**
Tools participate in permission decisions via:
- `validateInput()` - tool-specific validation
- `checkPermissions()` - tool-specific permission logic
- `preparePermissionMatcher()` - hook pattern matching

### 7. **Circular Dependency Management**
Lazy requires and getter functions prevent import cycles:
```typescript
const getTeamCreateTool = () => require('./TeamCreateTool').TeamCreateTool
```

### 8. **Environment-Aware Registration**
The `getAllBaseTools()` function serves as the single source of truth:
- Respects feature flags
- Handles environment-specific tools
- Supports conditional tool inclusion

---

## Summary

The Claude Code Tool System demonstrates sophisticated TypeScript patterns for building large-scale, type-safe AI tool infrastructures. Key innovations include:

1. **The `buildTool` factory** with type-level spread for fail-closed defaults
2. **Generic Tool interface** with typed progress events
3. **Deferred loading** via ToolSearch for context efficiency
4. **Comprehensive ToolUseContext** as a dependency injection container
5. **Multi-phase lifecycle**: Validation → Permission → Execution → Rendering
6. **Clean separation** between ToolDef (author input) and Tool (system consumption)

These patterns could be adapted for other AI agent systems requiring:
- Type-safe tool definitions at scale
- Dynamic tool loading and permission control
- Rich UI integration with React rendering
- Context window optimization via deferred loading
