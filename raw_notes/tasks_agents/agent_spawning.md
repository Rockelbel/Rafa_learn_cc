# Claude Code Agent Spawning System Analysis

## Overview

The Agent Spawning System in Claude Code provides a comprehensive framework for creating and managing subagents (delegated tasks) and swarm teammates (coordinated multi-agent systems). This analysis covers the core components and their interactions.

---

## 1. Core Files and Responsibilities

| File | Purpose |
|------|---------|
| `/home/claudeuser/ClaudeCode/tools/AgentTool/AgentTool.tsx` | Main entry point for agent spawning, handles sync/async routing |
| `/home/claudeuser/ClaudeCode/tools/AgentTool/runAgent.ts` | Core agent execution engine with lifecycle management |
| `/home/claudeuser/ClaudeCode/utils/agentContext.ts` | AsyncLocalStorage-based context for analytics attribution |
| `/home/claudeuser/ClaudeCode/utils/agentId.ts` | Deterministic ID generation for swarm teammates |
| `/home/claudeuser/ClaudeCode/utils/uuid.ts` | UUID generation for subagent IDs |
| `/home/claudeuser/ClaudeCode/utils/forkedAgent.ts` | Subagent context creation and forked agent execution |

---

## 2. Agent Lifecycle (Spawn, Run, Complete)

### 2.1 Spawn Phase (AgentTool.tsx)

```typescript
// Agent spawning entry point
async call({ prompt, subagent_type, description, ... }, toolUseContext) {
  // 1. Resolve agent definition (built-in, custom, or fork path)
  const selectedAgent = resolveAgent(effectiveType);

  // 2. Determine execution mode (sync vs async)
  const shouldRunAsync = (
    run_in_background === true ||
    selectedAgent.background === true ||
    isCoordinator ||
    forceAsync ||  // Fork subagent experiment
    assistantForceAsync  // KAIROS mode
  ) && !isBackgroundTasksDisabled;

  // 3. Create agent context with AsyncLocalStorage
  const agentContext = {
    agentId: asyncAgentId,
    parentSessionId: getParentSessionId(),
    agentType: 'subagent',
    subagentName: selectedAgent.agentType,
    isBuiltIn: isBuiltInAgent(selectedAgent),
    invokingRequestId: assistantMessage?.requestId,
    invocationKind: 'spawn',
    invocationEmitted: false
  };

  // 4. Route to sync or async execution path
  if (shouldRunAsync) {
    return runAsyncAgent(...);
  } else {
    return runWithAgentContext(agentContext, () => runSyncAgent(...));
  }
}
```

### 2.2 Run Phase (runAgent.ts)

The `runAgent` function is an async generator that:

1. **Initializes MCP servers** - Connects to agent-specific MCP servers
2. **Builds tool pool** - Resolves and filters tools for the agent
3. **Creates subagent context** - Isolates mutable state via `createSubagentContext`
4. **Registers hooks** - Sets up frontmatter-defined lifecycle hooks
5. **Preloads skills** - Loads any skills specified in agent definition
6. **Runs query loop** - Yields messages from the main query loop
7. **Records transcripts** - Persists messages to sidechain storage

```typescript
export async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  canUseTool,
  isAsync,
  availableTools,
  ...
}): AsyncGenerator<Message, void> {
  // Tool resolution with filtering
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools;

  // Create isolated subagent context
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    options: agentOptions,
    agentId,
    agentType: agentDefinition.agentType,
    messages: initialMessages,
    readFileState: agentReadFileState,
    abortController: agentAbortController,
    getAppState: agentGetAppState,
    shareSetAppState: !isAsync,  // Sync agents share state updates
    shareSetResponseLength: true, // Both sync/async contribute to metrics
  });

  // Run query loop
  for await (const message of query({ ... })) {
    // Record and yield each message
    await recordSidechainTranscript([message], agentId, lastRecordedUuid);
    yield message;
  }
}
```

### 2.3 Complete Phase

The completion phase handles cleanup in the `finally` block of `runAgent`:

```typescript
finally {
  // Clean up MCP servers
  await mcpCleanup();
  // Clear session hooks
  clearSessionHooks(rootSetAppState, agentId);
  // Clear prompt cache tracking
  cleanupAgentTracking(agentId);
  // Release file state cache
  agentToolUseContext.readFileState.clear();
  // Unregister from Perfetto tracing
  unregisterPerfettoAgent(agentId);
  // Kill background bash tasks
  killShellTasksForAgent(agentId, ...);
  // Remove from todo tracking
  rootSetAppState(prev => {
    const { [agentId]: _removed, ...todos } = prev.todos;
    return { ...prev, todos };
  });
}
```

---

## 3. Sync vs Async Agent Execution

### 3.1 Sync Agents (Foreground)

Sync agents run within the parent's async iteration and block until completion:

- Share `setAppState` with parent (state updates propagate)
- Share `abortController` with parent (parent abort propagates)
- Can show permission prompts (UI is shared)
- Support mid-execution backgrounding via `registerAgentForeground`
- Progress yielded in real-time to parent

```typescript
// Sync agent registration
const registration = registerAgentForeground({
  agentId: syncAgentId,
  description,
  prompt,
  selectedAgent,
  setAppState: rootSetAppState,
  autoBackgroundMs: getAutoBackgroundMs() || undefined
});

// Race between next message and background signal
const raceResult = await Promise.race([
  nextMessagePromise,
  backgroundPromise
]);

if (raceResult.type === 'background') {
  // Transition to async continuation
  void runWithAgentContext(syncAgentContext, async () => {
    // Continue in background...
  });
}
```

### 3.2 Async Agents (Background)

Async agents run independently with their own AbortController:

- Do NOT share `setAppState` (no-op callback)
- Unlinked `abortController` (surives parent ESC)
- Cannot show permission prompts (auto-deny)
- Registered via `registerAsyncAgent`
- Results delivered via notifications

```typescript
const agentBackgroundTask = registerAsyncAgent({
  agentId: asyncAgentId,
  description,
  prompt,
  selectedAgent,
  setAppState: rootSetAppState,
  toolUseId: toolUseContext.toolUseId
});

// Run detached
void runWithAgentContext(asyncAgentContext, () =>
  runAsyncAgentLifecycle({
    taskId: agentBackgroundTask.agentId,
    abortController: agentBackgroundTask.abortController!,
    makeStream: onCacheSafeParams => runAgent({ ... }),
    // ...
  })
);
```

### 3.3 Execution Mode Decision Matrix

| Factor | Sync | Async |
|--------|------|-------|
| `run_in_background=true` | No | Yes |
| `selectedAgent.background=true` | No | Yes |
| Coordinator mode | No | Yes |
| Fork subagent enabled | No | Yes (forced) |
| KAIROS enabled | No | Yes (forced) |
| User pressed ESC | Aborts | Continues |
| Can show prompts | Yes | No |
| Shares setAppState | Yes | No |

---

## 4. Agent Context Creation

### 4.1 SubagentContext via createSubagentContext (forkedAgent.ts)

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides,
): ToolUseContext {
  // Abort controller: explicit > share > new child
  const abortController =
    overrides?.abortController ??
    (overrides?.shareAbortController
      ? parentContext.abortController
      : createChildAbortController(parentContext.abortController));

  return {
    // Mutable state - cloned by default
    readFileState: cloneFileStateCache(overrides?.readFileState ?? parentContext.readFileState),
    nestedMemoryAttachmentTriggers: new Set<string>(),
    loadedNestedMemoryPaths: new Set<string>(),
    dynamicSkillDirTriggers: new Set<string>(),
    discoveredSkillNames: new Set<string>(),
    toolDecisions: undefined,
    contentReplacementState: cloneContentReplacementState(parentContext.contentReplacementState),

    // State callbacks - no-op by default (isolated)
    setAppState: overrides?.shareSetAppState ? parentContext.setAppState : () => {},
    setAppStateForTasks: parentContext.setAppStateForTasks ?? parentContext.setAppState,

    // New agent ID and query tracking
    agentId: overrides?.agentId ?? createAgentId(),
    queryTracking: {
      chainId: randomUUID(),
      depth: (parentContext.queryTracking?.depth ?? -1) + 1,
    },
    // ...
  };
}
```

### 4.2 Analytics Context via runWithAgentContext (agentContext.ts)

Uses Node.js `AsyncLocalStorage` to track agent identity across async operations:

```typescript
const agentContextStorage = new AsyncLocalStorage<AgentContext>();

export function runWithAgentContext<T>(context: AgentContext, fn: () => T): T {
  return agentContextStorage.run(context, fn);
}

export function getAgentContext(): AgentContext | undefined {
  return agentContextStorage.getStore();
}
```

**Why AsyncLocalStorage (not AppState):**
> When agents are backgrounded (ctrl+b), multiple agents can run concurrently in the same process. AppState is a single shared state that would be overwritten, causing Agent A's events to incorrectly use Agent B's context. AsyncLocalStorage isolates each async execution chain.

---

## 5. Tool Filtering for Agents

### 5.1 Filter Layers (agentToolUtils.ts)

```typescript
export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode,
}: {
  tools: Tools;
  isBuiltIn: boolean;
  isAsync?: boolean;
  permissionMode?: PermissionMode;
}): Tools {
  return tools.filter(tool => {
    // Layer 1: Allow MCP tools for all agents
    if (tool.name.startsWith('mcp__')) return true;

    // Layer 2: Allow ExitPlanMode for plan mode agents
    if (toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME) &&
        permissionMode === 'plan') return true;

    // Layer 3: Block tools disallowed for ALL agents
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;

    // Layer 4: Block additional tools for custom agents
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;

    // Layer 5: Restrict async agents to allowed tool set
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) {
      // Exception: In-process teammates get additional tools
      if (isAgentSwarmsEnabled() && isInProcessTeammate()) {
        if (toolMatchesName(tool, AGENT_TOOL_NAME)) return true;
        if (IN_PROCESS_TEAMMATE_ALLOWED_TOOLS.has(tool.name)) return true;
      }
      return false;
    }
    return true;
  });
}
```

### 5.2 Tool Disallow Sets (constants/tools.ts)

```typescript
// Blocked for ALL agents
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,        // Prevent recursion
  EXIT_PLAN_MODE_V2_TOOL_NAME,  // Plan mode is main thread only
  ENTER_PLAN_MODE_TOOL_NAME,
  AGENT_TOOL_NAME,              // Nested agents (external only)
  ASK_USER_QUESTION_TOOL_NAME,  // UI not available to subagents
  TASK_STOP_TOOL_NAME,          // Main thread task state only
  WORKFLOW_TOOL_NAME,           // Prevent recursive workflows
]);

// Allowed for ASYNC agents only
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
]);

// Additional tools for in-process teammates
export const IN_PROCESS_TEAMMATE_ALLOWED_TOOLS = new Set([
  TASK_CREATE_TOOL_NAME,
  TASK_GET_TOOL_NAME,
  TASK_LIST_TOOL_NAME,
  TASK_UPDATE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  CRON_CREATE_TOOL_NAME,
  CRON_DELETE_TOOL_NAME,
  CRON_LIST_TOOL_NAME,
]);
```

### 5.3 Agent-Specific Tool Resolution

```typescript
export function resolveAgentTools(
  agentDefinition: Pick<AgentDefinition, 'tools' | 'disallowedTools' | ...>,
  availableTools: Tools,
  isAsync = false,
  isMainThread = false,
): ResolvedAgentTools {
  // Step 1: Filter through agent tool allow/disallow lists
  const filteredAvailableTools = isMainThread
    ? availableTools
    : filterToolsForAgent({ tools: availableTools, isBuiltIn, isAsync });

  // Step 2: Apply agent-specific disallowed list
  const disallowedToolSet = new Set(agentDefinition.disallowedTools?.map(...));
  const allowedAvailableTools = filteredAvailableTools.filter(
    tool => !disallowedToolSet.has(tool.name)
  );

  // Step 3: Handle wildcard or explicit tool list
  const hasWildcard = agentTools === undefined || agentTools[0] === '*';
  if (hasWildcard) {
    return { hasWildcard: true, resolvedTools: allowedAvailableTools };
  }

  // Step 4: Resolve specific tool names
  for (const toolSpec of agentTools) {
    const { toolName, ruleContent } = permissionRuleValueFromString(toolSpec);
    // Handle special case: Agent(tool1,tool2) carries allowedAgentTypes
    if (toolName === AGENT_TOOL_NAME && ruleContent) {
      allowedAgentTypes = ruleContent.split(',').map(s => s.trim());
    }
    // Resolve tool from available pool...
  }
}
```

---

## 6. Agent ID Format and Generation

### 6.1 Subagent IDs (uuid.ts)

```typescript
/**
 * Generate a new agent ID with prefix for consistency with task IDs.
 * Format: a{label-}{16 hex chars}
 * Example: aa3f2c1b4d5e6f7a8, acompact-a3f2c1b4d5e6f7a8
 */
export function createAgentId(label?: string): AgentId {
  const suffix = randomBytes(8).toString('hex');
  return (label ? `a${label}-${suffix}` : `a${suffix}`) as AgentId;
}
```

Pattern: `a` prefix + optional label + 16 hex characters (8 bytes)
- Examples: `a3f2c1b4d5e6f7a8`, `acommit-a3f2c1b4d5e6f7a8`

### 6.2 Teammate IDs (agentId.ts)

```typescript
/**
 * Agent IDs: agentName@teamName
 * Example: team-lead@my-project, researcher@my-project
 * The @ symbol acts as a separator between agent name and team name
 */
export function formatAgentId(agentName: string, teamName: string): string {
  return `${agentName}@${teamName}`;
}

/**
 * Request IDs: {requestType}-{timestamp}@{agentId}
 * Example: shutdown-1702500000000@researcher@my-project
 */
export function generateRequestId(requestType: string, agentId: string): string {
  const timestamp = Date.now();
  return `${requestType}-${timestamp}@${agentId}`;
}
```

### 6.3 ID Types (types/ids.ts)

```typescript
/**
 * A session ID uniquely identifies a Claude Code session.
 */
export type SessionId = string & { readonly __brand: 'SessionId' };

/**
 * An agent ID uniquely identifies a subagent within a session.
 * When present, indicates the context is a subagent (not the main session).
 */
export type AgentId = string & { readonly __brand: 'AgentId' };

// Validation
const AGENT_ID_PATTERN = /^a(?:.+-)?[0-9a-f]{16}$/;
export function toAgentId(s: string): AgentId | null {
  return AGENT_ID_PATTERN.test(s) ? (s as AgentId) : null;
}
```

---

## 7. Agent Memory Management

### 7.1 Sidechain Transcript Storage

Agents record their messages to separate transcript storage:

```typescript
// In runAgent.ts
void recordSidechainTranscript(initialMessages, agentId).catch(...);

// During execution
for await (const message of query(...)) {
  if (isRecordableMessage(message)) {
    await recordSidechainTranscript([message], agentId, lastRecordedUuid);
  }
}
```

### 7.2 Agent Metadata Persistence

```typescript
void writeAgentMetadata(agentId, {
  agentType: agentDefinition.agentType,
  worktreePath,      // For worktree isolation
  description,       // Original task description
}).catch(...);
```

### 7.3 Content Replacement State

Cloned from parent for cache stability:

```typescript
// In createSubagentContext
contentReplacementState:
  overrides?.contentReplacementState ??
  (parentContext.contentReplacementState
    ? cloneContentReplacementState(parentContext.contentReplacementState)
    : undefined),
```

---

## 8. Subagent vs Main Thread Distinction

### 8.1 Key Differences

| Aspect | Main Thread | Subagent |
|--------|-------------|----------|
| **Agent ID** | None (undefined) | `createAgentId()` generates `a{16hex}` |
| **Context** | No wrapping | Wrapped in `runWithAgentContext()` |
| **setAppState** | Functional, updates UI | No-op (async) or shared (sync) |
| **Permission Prompts** | Can show | Cannot show (`shouldAvoidPermissionPrompts: true`) |
| **Tool Access** | Full set | Filtered via `filterToolsForAgent` |
| **Transcript** | Main transcript | Sidechain storage |
| **Abort Handling** | User ESC | Detached controller (async) |
| **MCP Servers** | Parent's + agent-specific | Parent's + agent-specific |

### 8.2 Detecting Subagent Context

```typescript
// Via agent context
const context = getAgentContext();
if (isSubagentContext(context)) {
  // Running inside a subagent
  console.log(context.agentId);        // Subagent's ID
  console.log(context.subagentName);   // e.g., "Explore"
  console.log(context.isBuiltIn);      // true/false
}

// Via tool use context
if (toolUseContext.agentId) {
  // Has an agent ID → is a subagent
}

// Via branded type
const maybeAgentId = toAgentId(someString);
if (maybeAgentId) {
  // Valid agent ID format
}
```

### 8.3 Teammate vs Subagent

Teammates (swarm agents) have additional context:

```typescript
export type TeammateAgentContext = {
  agentId: string;           // e.g., "researcher@my-team"
  agentName: string;         // e.g., "researcher"
  teamName: string;          // e.g., "my-team"
  agentColor?: string;       // UI color
  planModeRequired: boolean;
  parentSessionId: string;   // Team lead's session
  isTeamLead: boolean;
  agentType: 'teammate';
  // ... same invocation tracking as subagent
};
```

---

## 9. Spawn Patterns Summary

### 9.1 Built-in Agent Spawn

```typescript
// AgentTool call with subagent_type
Agent({
  subagent_type: 'Explore',
  prompt: 'Find all usages of function X',
  description: 'Explore codebase'
});
```

### 9.2 Fork Subagent Spawn

```typescript
// When fork experiment is enabled and no subagent_type specified
// Child inherits parent's system prompt for cache hits
Agent({
  prompt: 'Continue this task',
  description: 'Fork continuation'
});
// → Uses FORK_AGENT definition, inherits parent context messages
```

### 9.3 Custom Agent Spawn

```typescript
// User-defined agent from .claude/agents/
Agent({
  subagent_type: 'my-custom-agent',
  prompt: 'Do something custom',
  description: 'Custom task'
});
```

### 9.4 Teammate Spawn

```typescript
// Multi-agent swarm (requires name + team context)
Agent({
  name: 'researcher',
  team_name: 'my-team',
  prompt: 'Research this topic',
  description: 'Research task'
});
// → Spawns in-process or tmux teammate, not a subagent
```

### 9.5 Remote Agent Spawn

```typescript
// Ant-only: Remote isolation via CCR
Agent({
  subagent_type: 'Explore',
  prompt: 'Analyze this code',
  isolation: 'remote'
});
// → Launches in remote Claude Code environment
```

---

## 10. Critical Implementation Notes

1. **Cache Safety**: Fork agents share parent's system prompt and exact tool array for prompt cache hits
2. **Background Safety**: Async agents get unlinked abort controllers to survive parent cancellation
3. **Permission Isolation**: Subagents get `shouldAvoidPermissionPrompts: true` to prevent UI access
4. **State Isolation**: `createSubagentContext` clones all mutable state by default
5. **Cleanup Guarantees**: `finally` blocks ensure cleanup even on abort/error
6. **Transcript Separation**: Subagent messages go to sidechain storage, not main transcript
7. **Tool Filtering**: Multiple layers (built-in vs custom, sync vs async, agent-specific disallowed)
