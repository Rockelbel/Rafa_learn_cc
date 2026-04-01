# Agent Coordination Analysis

## Overview

Claude Code implements a sophisticated agent coordination system with multiple patterns for parallel agent execution, team management, and inter-agent communication. This analysis covers the swarm patterns, team architecture, and coordination mechanisms.

## 1. Agent Swarm Feature Flag

**File**: `/home/claudeuser/ClaudeCode/utils/agentSwarmsEnabled.ts`

The swarm feature is gated through `isAgentSwarmsEnabled()`:

- **Ant builds**: Always enabled (`USER_TYPE === 'ant'`)
- **External builds**: Require both:
  - Opt-in via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var OR `--agent-teams` flag
  - GrowthBook gate `tengu_amber_flint` enabled (killswitch)

```typescript
export function isAgentSwarmsEnabled(): boolean {
  // Ant: always on
  if (process.env.USER_TYPE === 'ant') {
    return true
  }
  // External: require opt-in + killswitch
  // ...
}
```

## 2. Fork vs Resume Patterns

### Fork Pattern

**File**: `/home/claudeuser/ClaudeCode/tools/AgentTool/forkSubagent.ts`

The fork pattern creates a child agent that inherits the parent's full conversation context:

**Key characteristics**:
- `subagent_type` becomes optional when fork is enabled
- Omitting `subagent_type` triggers implicit fork
- Child inherits parent's system prompt and conversation history
- All agent spawns run in background (async)
- `/fork <directive>` slash command available

**Fork Agent Definition**:
```typescript
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,  // 'fork'
  tools: ['*'],                   // Inherits parent's exact tool pool
  maxTurns: 200,
  model: 'inherit',               // Keeps parent's model
  permissionMode: 'bubble',       // Surfaces permission prompts to parent
  useExactTools: true,            // For cache-identical API prefixes
}
```

**Cache Sharing Strategy**:
All fork children must produce byte-identical API request prefixes for prompt cache sharing:
- Keeps full parent assistant message (all tool_use blocks, thinking, text)
- Builds single user message with tool_results using identical placeholder
- Only final text block differs per child

```typescript
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
```

**Anti-recursion Guard**:
```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    // Detect fork boilerplate tag in conversation history
    return content.some(block =>
      block.type === 'text' &&
      block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    )
  })
}
```

**Fork Child Rules** (enforced via system prompt):
1. Do NOT spawn sub-agents - execute directly
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE tools directly (Bash, Read, Write, etc.)
5. Commit changes before reporting (include hash)
6. Do NOT emit text between tool calls
7. Stay strictly within directive's scope
8. Keep reports under 500 words
9. Response MUST begin with "Scope:"
10. REPORT structured facts, then stop

### Resume Pattern

**File**: `/home/claudeuser/ClaudeCode/tools/AgentTool/resumeAgent.ts`

The resume pattern continues a previously spawned agent:

**Key characteristics**:
- Retrieves agent transcript from session storage
- Reconstructs content replacement state
- Handles worktree path restoration (with fallback)
- Supports both regular agents and fork agents
- Preserves original agent type and configuration

**Resume Process**:
```typescript
const [transcript, meta] = await Promise.all([
  getAgentTranscript(asAgentId(agentId)),
  readAgentMetadata(asAgentId(agentId)),
])

const resumedMessages = filterWhitespaceOnlyAssistantMessages(
  filterOrphanedThinkingOnlyMessages(
    filterUnresolvedToolUses(transcript.messages),
  ),
)
```

**Fork Resume Special Handling**:
- Passes parent's system prompt for cache-identical prefix
- Uses `useExactTools: true` to preserve tool pool
- Reconstructs system prompt if not available in context

## 3. Coordinator Mode

**File**: `/home/claudeuser/ClaudeCode/coordinator/coordinatorMode.ts`

Coordinator mode enables orchestration of multiple background workers:

**Activation**:
```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

**Mutual Exclusivity**:
Fork subagent is disabled when coordinator mode is active (coordinator owns orchestration).

**Coordinator System Prompt**:
- Defines coordinator as orchestrator of software engineering tasks
- Workers spawned via Agent tool with `subagent_type: 'worker'`
- Results arrive as `<task-notification>` XML messages
- Parallelism is emphasized as "superpower"

**Worker Tools**:
- Bash, Read, Edit (simple mode)
- All standard tools + MCP tools + Skill tool (full mode)

**Task Phases**:
| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase |
| Synthesis | Coordinator | Read findings, craft specs |
| Implementation | Workers | Make targeted changes |
| Verification | Workers | Test changes work |

## 4. Team Architecture

**File**: `/home/claudeuser/ClaudeCode/tools/TeamCreateTool/TeamCreateTool.ts`

**Team File Structure** (`/home/claudeuser/ClaudeCode/utils/swarm/teamHelpers.ts`):
```typescript
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    prompt?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

**Team Creation**:
- Leader can only manage one team at a time
- Generates deterministic agent ID: `team-lead@teamName`
- Creates team directory: `~/.claude/teams/{team-name}/`
- Creates task list directory for team coordination
- Registers for session cleanup

**Team Constants** (`/home/claudeuser/ClaudeCode/utils/swarm/constants.ts`):
```typescript
export const TEAM_LEAD_NAME = 'team-lead'
export const SWARM_SESSION_NAME = 'claude-swarm'
export const SWARM_VIEW_WINDOW_NAME = 'swarm-view'
export const TMUX_COMMAND = 'tmux'
export const HIDDEN_SESSION_NAME = 'claude-hidden'
```

## 5. Coordination Mechanisms

### Local Agent Task System

**File**: `/home/claudeuser/ClaudeCode/tasks/LocalAgentTask/LocalAgentTask.tsx`

Task state for background agents:
```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean
  pendingMessages: string[]
  retain: boolean        // UI is holding this task
  diskLoaded: boolean    // Bootstrap has read sidechain JSONL
  evictAfter?: number    // Panel visibility deadline
}
```

**Task Registration**:
```typescript
export function registerAsyncAgent({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  parentAbortController,  // For child abort propagation
  toolUseId
}): LocalAgentTaskState
```

**Agent Notification Format**:
```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent "{description}" completed</summary>
  <result>{agent's final text response}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

### Coordinator Task Panel

**File**: `/home/claudeuser/ClaudeCode/components/CoordinatorAgentStatus.tsx`

- Steerable list of background agents
- Renders below prompt input footer
- Shows main thread + visible agent tasks
- 1-second tick for elapsed time and eviction
- Enter to view/steer, x to dismiss
- Shows: name, description, elapsed time, token count, queued messages

### Backend Execution Modes

**File**: `/home/claudeuser/ClaudeCode/utils/swarm/backends/registry.ts`

Three backend types:
1. **tmux**: Uses tmux for pane management
2. **iterm2**: Uses iTerm2 native split panes via it2 CLI
3. **in-process**: Runs teammate in same Node.js process

**Backend Detection Priority**:
1. If inside tmux, always use tmux
2. If in iTerm2 with it2 available, use iTerm2
3. If in iTerm2 without it2, use tmux fallback
4. If tmux available, use tmux (external session)
5. Otherwise, use in-process

**Teammate Executor Interface**:
```typescript
type TeammateExecutor = {
  readonly type: BackendType
  isAvailable(): Promise<boolean>
  spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult>
  sendMessage(agentId: string, message: TeammateMessage): Promise<void>
  terminate(agentId: string, reason?: string): Promise<boolean>
  kill(agentId: string): Promise<boolean>
  isActive(agentId: string): Promise<boolean>
}
```

## 6. Cache Sharing Between Agents

### Prompt Cache Strategy

**Fork Pattern**:
- All fork children share byte-identical API request prefixes
- Parent's system prompt is threaded via `renderedSystemPrompt`
- Tool pool uses `useExactTools: true` for cache-identical definitions
- Placeholder results are identical across all children

**Coordinator Mode**:
- Workers don't share cache with coordinator (separate API calls)
- Resume preserves cache by using parent's rendered system prompt
- Content replacement state reconstructed for subagent resume

## 7. Parallel Agent Execution

### Concurrency Guidelines (from coordinatorMode.ts)

**Parallelism Rules**:
- **Read-only tasks** (research) — run in parallel freely
- **Write-heavy tasks** (implementation) — one at a time per file set
- **Verification** can sometimes run alongside implementation on different areas

**Tool Call Pattern**:
Multiple tool calls in a single message for parallel execution:
```typescript
${AGENT_TOOL_NAME}({ description: "Investigate auth bug", ... })
${AGENT_TOOL_NAME}({ description: "Research secure token storage", ... })
```

### Continue vs Spawn Decision Matrix

| Situation | Mechanism | Why |
|-----------|-----------|-----|
| Research explored exactly the files that need editing | **Continue** (SendMessage) | Worker already has files in context |
| Research was broad but implementation is narrow | **Spawn fresh** | Avoid dragging exploration noise |
| Correcting a failure or extending recent work | **Continue** | Worker has error context |
| Verifying code a different worker just wrote | **Spawn fresh** | Fresh eyes, no implementation assumptions |
| First implementation attempt used wrong approach | **Spawn fresh** | Clean slate avoids anchoring |
| Completely unrelated task | **Spawn fresh** | No useful context to reuse |

## 8. Agent Communication

### SendMessageTool

**File**: `/home/claudeuser/ClaudeCode/tools/SendMessageTool/SendMessageTool.ts`

- Sends follow-up messages to running agents
- Uses `to` parameter with agent ID
- Queued messages drained at tool-round boundaries
- Messages appear in viewed transcript immediately

### Message Queue Management

**File**: `/home/claudeuser/ClaudeCode/tasks/LocalAgentTask/LocalAgentTask.tsx`

```typescript
export function queuePendingMessage(
  taskId: string,
  msg: string,
  setAppState: SetAppState
): void

export function drainPendingMessages(
  taskId: string,
  getAppState: () => AppState,
  setAppState: SetAppState
): string[]
```

### Agent Progress Tracking

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]
  summary?: string  // AI-generated progress summary
}
```

Progress tracked from assistant messages:
- Input tokens (cumulative from API)
- Output tokens (summed per-turn)
- Tool use count
- Recent activity classification (search vs read)

## Summary

The agent coordination system in Claude Code provides:

1. **Flexible Spawning Patterns**: Fork (inherit context) vs Resume (continue existing) vs Fresh (clean slate)

2. **Multi-Modal Execution**: Pane-based (tmux/iTerm2) for visibility vs In-process for lightweight parallelism

3. **Coordinated Team Management**: Team files, member tracking, permission synchronization

4. **Efficient Cache Utilization**: Byte-identical prefixes for fork children, reconstructed state for resumes

5. **Rich Progress Visibility**: Token counts, tool usage, activity classification, AI summaries

6. **Robust Lifecycle Management**: Abort propagation, cleanup registration, eviction policies
