# Claude Code Agent System Analysis

## Overview

The Agent Tool system in Claude Code provides a multi-agent architecture that enables:
- Subagent spawning and lifecycle management
- Context isolation and sharing between agents
- Background/async task execution
- Resume patterns for long-running agents
- Built-in specialized agent types (explore, plan, verify, etc.)

## Core Architecture

### Key Files

| File | Purpose |
|------|---------|
| `AgentTool.tsx` | Main tool implementation, handles spawning logic |
| `runAgent.ts` | Core agent execution engine |
| `forkSubagent.ts` | Fork-based subagent spawning (experimental) |
| `resumeAgent.ts` | Agent resume functionality for background agents |
| `builtInAgents.ts` | Built-in agent type registry |
| `loadAgentsDir.ts` | Agent definition loading and parsing |
| `agentToolUtils.ts` | Shared utilities for agent lifecycle |
| `forkedAgent.ts` | Context isolation helpers |

---

## Subagent Spawning and Management

### Spawning Methods

#### 1. **Normal Subagent Spawn**
- Uses `runAgent()` with agent definition and prompt messages
- Creates isolated context via `createSubagentContext()`
- Supports both sync and async execution modes
- Tool pool is filtered based on agent definition

#### 2. **Fork Subagent (Experimental)**
- Enabled via `FORK_SUBAGENT` feature flag
- Inherits parent's full conversation context
- Uses `buildForkedMessages()` to construct child messages
- Optimized for prompt cache sharing (byte-identical prefixes)
- Recursive fork guard prevents nested forking

```typescript
// Fork path key characteristics:
- selectedAgent = FORK_AGENT (synthetic definition)
- promptMessages = buildForkedMessages(prompt, assistantMessage)
- forkContextMessages = toolUseContext.messages (parent's full context)
- useExactTools = true (cache-identical tool pool)
```

### Agent Lifecycle States

```
Spawned
  ├── Sync Execution (foreground)
  │     ├── Running (with optional background hint after 2s)
  │     ├── Can be backgrounded via race condition
  │     └── Completes → finalizeAgentTool()
  │
  └── Async Execution (background)
        ├── Running → registered in AppState.tasks
        ├── Can be killed via abortController
        ├── Completes → enqueueAgentNotification()
        └── Can be resumed later
```

---

## Fork vs Resume Patterns

### Fork Pattern (`forkSubagent.ts`)

**Purpose:** Create child agents with inherited context for parallel/async work

**Key Features:**
- Inherits parent's system prompt for cache sharing
- Clones parent's conversation messages
- Uses placeholder tool results for cache-identical prefixes
- Injects boilerplate message with strict rules for fork children

**Message Construction:**
```typescript
// 1. Clone full assistant message (all tool_use blocks)
// 2. Build tool_result blocks with identical placeholder text
// 3. Append per-child directive as final text block
// Result: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
```

**Anti-Recursion Guard:**
```typescript
// Detects fork boilerplate tag in conversation history
export function isInForkChild(messages: MessageType[]): boolean
```

### Resume Pattern (`resumeAgent.ts`)

**Purpose:** Resume previously spawned background agents

**Key Features:**
- Loads transcript from session storage via `getAgentTranscript()`
- Reconstructs content replacement state for cache stability
- Handles worktree path restoration (with fallback if deleted)
- Supports both fork and non-fork agent types

**Resume Process:**
```typescript
1. Load transcript and metadata
2. Filter/resume messages (remove orphaned thinking, unresolved tools)
3. Reconstruct content replacement state
4. Resolve agent definition (FORK_AGENT or from activeAgents)
5. Run with reconstructed context
```

---

## Built-in Agent Types

### Agent Definition Structure

```typescript
type BuiltInAgentDefinition = {
  agentType: string           // Unique identifier
  whenToUse: string          // Description for model routing
  tools?: string[]           // Allowed tools (['*'] = all)
  disallowedTools?: string[] // Explicitly blocked tools
  model?: string             // Model override ('inherit' = parent's model)
  maxTurns?: number          // Turn limit
  permissionMode?: PermissionMode
  getSystemPrompt: (params) => string  // Dynamic prompt generator
  source: 'built-in'
  // ... additional fields
}
```

### Available Agents

| Agent | Type | Key Characteristics |
|-------|------|---------------------|
| `general-purpose` | Built-in | Default agent, full tool access, research-focused |
| `Explore` | Built-in | Read-only search, fast (haiku for non-ant), disallows write/edit |
| `Plan` | Built-in | Read-only architect agent, creates implementation plans |
| `verification` | Built-in | Background verification agent, adversarial testing |
| `claude-code-guide` | Built-in | Documentation/help agent, uses web fetch/search |
| Custom | User-defined | Loaded from `.claude/agents/*.md` |
| Plugin | Plugin-defined | From installed plugins |

### Tool Filtering by Agent Type

```typescript
// In agentToolUtils.ts
export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode,
}): Tools {
  // MCP tools always allowed
  // ExitPlanMode allowed for plan mode agents
  // ALL_AGENT_DISALLOWED_TOOLS blocked for all
  // CUSTOM_AGENT_DISALLOWED_TOOLS blocked for non-built-in
  // ASYNC_AGENT_ALLOWED_TOOLS restricts async agents
}
```

---

## Context Passing and Isolation

### `createSubagentContext()` (`forkedAgent.ts`)

Creates isolated ToolUseContext for subagents with configurable sharing:

**Isolated by Default:**
- `readFileState`: Cloned from parent
- `abortController`: New child controller (linked to parent)
- `getAppState`: Wrapped with `shouldAvoidPermissionPrompts`
- All mutation callbacks: No-op

**Explicit Sharing Options:**
- `shareSetAppState`: Share parent's state updates
- `shareSetResponseLength`: Share response metrics
- `shareAbortController`: Share parent's controller

**Code Pattern:**
```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,  // Sync agents share state
  shareSetResponseLength: true, // Both sync/async contribute metrics
})
```

### Context Inheritance Chain

```
Parent Context
  ├── System Prompt (inherited or rebuilt)
  ├── Tools (filtered or exact copy)
  ├── User/System Context (inherited)
  ├── File State Cache (cloned)
  ├── Content Replacement State (cloned)
  ├── Abort Controller (linked or shared)
  └── App State (wrapped or shared)
```

---

## Agent Memory and Persistence

### Transcript Storage

```typescript
// Sidechain transcript recording for subagents
recordSidechainTranscript(messages, agentId, parentUuid?)
```

- Each agent has isolated transcript subdirectory
- UUID-chain linked for parent-child continuity
- Used by resume to reconstruct conversation state

### Metadata Persistence

```typescript
writeAgentMetadata(agentId, {
  agentType: selectedAgent.agentType,
  worktreePath,    // For worktree isolation
  description,     // Original task description
})
```

### Memory Scopes (for custom agents)

- `user`: Shared across all user's projects
- `project`: Shared within project
- `local`: Isolated to agent instance

Managed via `agentMemory.ts` and `agentMemorySnapshot.ts`

---

## AgentTool Wrapping Other Tools

### Tool Resolution Flow

```typescript
// 1. Agent tool receives spawn request
// 2. Resolves agent definition (built-in, custom, or plugin)
// 3. Filters tools based on agent's allow/disallow lists
resolveAgentTools(agentDefinition, availableTools, isAsync)

// 4. Assembles worker tool pool with agent's permission mode
const workerTools = assembleToolPool(workerPermissionContext, mcpTools)

// 5. Runs agent with filtered tools via runAgent()
```

### Permission Mode Inheritance

```typescript
// Agent can override parent's permission mode (with restrictions)
if (agentPermissionMode &&
    parentMode !== 'bypassPermissions' &&
    parentMode !== 'acceptEdits') {
  toolPermissionContext = {
    ...toolPermissionContext,
    mode: agentPermissionMode,
  }
}
```

### MCP Server Integration

Agents can define their own MCP servers in frontmatter:
```typescript
// AgentDefinition.mcpServers
export type AgentMcpServerSpec =
  | string  // Reference by name
  | { [name: string]: McpServerConfig }  // Inline definition
```

- Agent-specific servers are additive to parent's servers
- Newly created clients cleaned up on agent finish
- Shared clients (referenced by name) remain for parent

---

## Multi-Agent Architecture Patterns

### Pattern 1: Hierarchical Delegation
```
Main Agent
  └── Subagent (general-purpose)
        └── Subagent (Explore for search)
```

### Pattern 2: Parallel Forking
```
Main Agent
  ├── Fork Worker A (cache-identical prefix)
  ├── Fork Worker B (cache-identical prefix)
  └── Fork Worker C (cache-identical prefix)
```

### Pattern 3: Background-Resume
```
Main Agent
  └── Spawn background agent
        (later) Resume agent with new prompt
```

### Pattern 4: Agent Teams (In-Process Teammates)
```
Team Lead (tmux window)
  ├── Teammate A (split pane)
  ├── Teammate B (split pane)
  └── Teammate C (split pane)
```

### Pattern 5: Worktree Isolation
```
Main Agent (main worktree)
  └── Agent with isolation: 'worktree'
        (isolated git worktree, same repo)
```

---

## Background Agent Lifecycle (`runAsyncAgentLifecycle`)

```typescript
export async function runAsyncAgentLifecycle({
  taskId,
  abortController,
  makeStream,          // Generator factory
  metadata,
  description,
  toolUseContext,
  rootSetAppState,
  agentIdForCleanup,
  enableSummarization,
  getWorktreeResult,
}): Promise<void>
```

**Stages:**
1. **Init**: Create progress tracker, activity resolver
2. **Execution**: Iterate messages from `makeStream()`
3. **Progress Tracking**: Update AppState.tasks with progress
4. **Summarization**: Optional background summarization via `startAgentSummarization()`
5. **Completion**:
   - Success: `completeAsyncAgent()` + notification
   - Abort: `killAsyncAgent()` + partial result notification
   - Error: `failAsyncAgent()` + error notification
6. **Cleanup**: Clear skills, dump state

---

## Key Design Decisions

### 1. **Cache-Identical Forking**
Fork children share parent's system prompt and tool definitions for prompt cache hits. The key insight: "byte-identical prefixes" enable Anthropic API's prompt caching.

### 2. **No Recursive Forking**
Fork children cannot spawn further subagents (detected via boilerplate tag). This prevents exponential complexity and resource exhaustion.

### 3. **Permission Mode Bubble**
`permissionMode: 'bubble'` surfaces permission prompts to parent terminal, enabling async agents to request user input.

### 4. **Content Replacement State Cloning**
Subagents clone parent's content replacement state to make identical replacement decisions, maintaining cache stability.

### 5. **setAppStateForTasks**
Always-shared channel to root AppState for infrastructure (task registration, hooks, bash tasks). Unlike `setAppState` which is no-op for async agents.

---

## Safety and Guardrails

| Mechanism | Purpose |
|-----------|---------|
| Recursive fork guard | Prevents nested fork spawning |
| `isInForkChild()` | Detects fork context from message history |
| Tool filtering | Enforces agent type capabilities |
| Permission mode inheritance | Controls what agents can do |
| Worktree isolation | Filesystem isolation for dangerous operations |
| Handoff classifier | Safety review before returning to parent |
| `allowedTools` | Explicit tool whitelisting |

---

## File Locations

```
/home/claudeuser/ClaudeCode/tools/AgentTool/
  ├── AgentTool.tsx          # Main tool, 1300+ lines
  ├── runAgent.ts            # Execution engine, 900+ lines
  ├── forkSubagent.ts        # Fork pattern implementation
  ├── resumeAgent.ts         # Resume pattern implementation
  ├── builtInAgents.ts       # Built-in agent registry
  ├── loadAgentsDir.ts       # Agent loading/parsing
  ├── agentToolUtils.ts      # Shared utilities
  ├── built-in/
  │   ├── generalPurposeAgent.ts
  │   ├── exploreAgent.ts
  │   ├── planAgent.ts
  │   ├── verificationAgent.ts
  │   ├── claudeCodeGuideAgent.ts
  │   └── statuslineSetup.ts

/home/claudeuser/ClaudeCode/utils/
  ├── forkedAgent.ts         # Context isolation helpers
  ├── sessionStorage.ts      # Transcript persistence
  └── worktree.ts            # Git worktree management
```
