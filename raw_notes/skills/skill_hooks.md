# Claude Code Skill Hooks System Analysis

## Overview

The Skill Hooks System is an extensible framework that allows users to execute custom logic at various points in Claude Code's lifecycle. Hooks can be defined as shell commands, LLM prompts, HTTP requests, or agent-based verifiers. The system provides fine-grained control over tool execution, session management, and workflow automation.

## Hook Event Types (28 Events)

Defined in `/home/claudeuser/ClaudeCode/entrypoints/sdk/coreTypes.ts`:

| Event | Description |
|-------|-------------|
| `PreToolUse` | Before a tool is executed |
| `PostToolUse` | After a tool completes successfully |
| `PostToolUseFailure` | After a tool fails |
| `Notification` | When a notification is triggered |
| `UserPromptSubmit` | When user submits a prompt |
| `SessionStart` | When a session begins |
| `SessionEnd` | When a session ends |
| `Stop` | When the agent stops |
| `StopFailure` | When stop fails |
| `SubagentStart` | When a subagent starts |
| `SubagentStop` | When a subagent stops |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction |
| `PermissionRequest` | When permission is requested |
| `PermissionDenied` | When permission is denied |
| `Setup` | During initial setup |
| `TeammateIdle` | When a teammate becomes idle |
| `TaskCreated` | When a task is created |
| `TaskCompleted` | When a task completes |
| `Elicitation` | When elicitation starts |
| `ElicitationResult` | When elicitation completes |
| `ConfigChange` | When configuration changes |
| `WorktreeCreate` | When a worktree is created |
| `WorktreeRemove` | When a worktree is removed |
| `InstructionsLoaded` | When instructions are loaded |
| `CwdChanged` | When working directory changes |
| `FileChanged` | When a file changes |

## Hook Types (5 Types)

### 1. Command Hook (`type: 'command'`)
- **Execution**: Shell command via bash or PowerShell
- **Use case**: Running scripts, git operations, file system checks
- **Timeout**: Default 10 minutes (TOOL_HOOK_EXECUTION_TIMEOUT_MS)
- **Key fields**: `command`, `shell` (bash/powershell), `if` condition, `timeout`, `async`, `asyncRewake`
- **Features**:
  - Supports async execution with `async: true`
  - Async rewake mode wakes model on exit code 2
  - Prompt request protocol for interactive hooks
  - Env var substitution (`$CLAUDE_PLUGIN_ROOT`, `$CLAUDE_PLUGIN_DATA`)

### 2. Prompt Hook (`type: 'prompt'`)
- **Execution**: Single-turn LLM query with structured JSON output
- **Use case**: Content validation, semantic checks, simple verification
- **Timeout**: Default 30 seconds
- **Key fields**: `prompt`, `model`, `if` condition, `timeout`
- **Features**:
  - `$ARGUMENTS` placeholder for hook input JSON
  - Returns `{ok: boolean, reason?: string}` schema
  - Non-blocking by default (can return blocking error)

### 3. Agent Hook (`type: 'agent'`)
- **Execution**: Multi-turn LLM agent with tool access
- **Use case**: Complex verification requiring file inspection, search, or multiple steps
- **Timeout**: Default 60 seconds
- **Key fields**: `prompt`, `model`, `if` condition, `timeout`
- **Features**:
  - Uses `SyntheticOutputTool` for structured responses
  - Max 50 turns per agent execution
  - Can access transcript and use available tools
  - Runs in isolated session with separate agent ID

### 4. HTTP Hook (`type: 'http'`)
- **Execution**: POST request to external URL
- **Use case**: Integration with external services, webhooks
- **Timeout**: Default 10 minutes
- **Key fields**: `url`, `headers`, `allowedEnvVars`, `if` condition, `timeout`
- **Features**:
  - JSON body with hook input
  - Header env var interpolation (`$VAR_NAME`)
  - URL allowlist security policy
  - SSRF protection with IP validation
  - Sandbox proxy support

### 5. Function Hook (`type: 'function'`)
- **Execution**: In-memory TypeScript callback
- **Use case**: Session-scoped validation, programmatic checks
- **Timeout**: Default 5 seconds
- **Key fields**: `callback`, `errorMessage`, `timeout`
- **Features**:
  - Session-scoped only (not persistable)
  - Access to message history
  - Used for structured output enforcement

## Execution Pipeline

### 1. Hook Matching (`getMatchingHooks`)
```typescript
// Located in: /home/claudeuser/ClaudeCode/utils/hooks.ts

// Merge hooks from multiple sources:
// - Settings snapshot (user/project/local settings.json)
// - Registered hooks (SDK callbacks, plugin native hooks)
// - Session hooks (ephemeral, in-memory)

const hookMatchers = getHooksConfig(appState, sessionId, hookEvent)

// Match by query:
// - PreToolUse/PostToolUse: match on tool_name
// - SessionStart: match on source
// - Stop: match on reason
// - etc.

const filteredMatchers = matchQuery
  ? hookMatchers.filter(matcher => matchesPattern(matchQuery, matcher.matcher))
  : hookMatchers
```

### 2. Hook Deduplication
- Command hooks: dedup by `shell\0command\0ifCondition`
- Prompt hooks: dedup by `prompt\0ifCondition`
- Agent hooks: dedup by `prompt\0ifCondition`
- HTTP hooks: dedup by `url\0ifCondition`
- Callback/function hooks: no dedup (each unique)

### 3. `if` Condition Filtering
- Uses permission rule syntax (e.g., `"Bash(git *)"`)
- Evaluated against tool_name and tool_input
- Prevents spawning hooks for non-matching commands

### 4. Parallel Execution
```typescript
// All matched hooks run in parallel
const hookPromises = matchingHooks.map(async function* ({ hook }) {
  if (hook.type === 'callback') {
    yield executeHookCallback({...})
  } else if (hook.type === 'function') {
    yield executeFunctionHook({...})
  } else if (hook.type === 'command') {
    yield execCommandHook({...})
  } else if (hook.type === 'prompt') {
    yield execPromptHook({...})
  } else if (hook.type === 'agent') {
    yield execAgentHook({...})
  } else if (hook.type === 'http') {
    yield execHttpHook({...})
  }
})
```

### 5. Result Aggregation
```typescript
// Results aggregated into AggregatedHookResult:
{
  message?: Message
  blockingErrors?: HookBlockingError[]
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  additionalContexts?: string[]
  updatedInput?: Record<string, unknown>
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

## Hook Result Handling

### JSON Output Schema

```typescript
// Sync hook response
{
  continue?: boolean           // Stop execution if false
  suppressOutput?: boolean     // Hide from transcript
  stopReason?: string          // Reason for stopping
  decision?: 'approve' | 'block'
  reason?: string              // Explanation
  systemMessage?: string       // Warning to user
  hookSpecificOutput?: {       // Event-specific data
    hookEventName: string
    // ... event-specific fields
  }
}

// Async hook response
{
  async: true
  asyncTimeout?: number
}
```

### Event-Specific Output Fields

| Event | Special Fields |
|-------|---------------|
| `PreToolUse` | `permissionDecision`, `updatedInput`, `additionalContext` |
| `UserPromptSubmit` | `additionalContext` |
| `SessionStart` | `additionalContext`, `initialUserMessage`, `watchPaths` |
| `PostToolUse` | `additionalContext`, `updatedMCPToolOutput` |
| `PermissionDenied` | `retry` |
| `PermissionRequest` | `decision` (allow/deny with behavior) |
| `Elicitation` | `action`, `content` |

## Permission Integration

### PreToolUse Hook Permission Decisions
- `allow`: Grant permission automatically
- `deny`: Block the tool execution
- `ask`: Show standard permission prompt

### PermissionRequest Hook
Can modify permission decisions:
```typescript
{
  decision: {
    behavior: 'allow',
    updatedInput?: Record<string, unknown>
    updatedPermissions?: PermissionUpdate[]
  } | {
    behavior: 'deny',
    message?: string
    interrupt?: boolean
  }
}
```

### Security Measures
1. **Workspace Trust**: ALL hooks require trust dialog acceptance in interactive mode
2. **Managed Hooks Only**: `allowManagedHooksOnly` setting restricts to admin-defined hooks
3. **HTTP URL Allowlist**: `allowedHttpHookUrls` restricts HTTP hook destinations
4. **Env Var Restrictions**: `allowedEnvVars` controls env var interpolation in headers
5. **SSRF Protection**: Blocks private/link-local IPs (except loopback for dev)

## Hook Type Comparison

| Aspect | Command | Prompt | Agent | HTTP | Function |
|--------|---------|--------|-------|------|----------|
| **Execution** | Shell spawn | LLM query | Multi-turn agent | HTTP POST | TS callback |
| **Complexity** | Low | Low | High | Medium | Low |
| **Tool Access** | No | No | Yes | No | No |
| **Persistence** | Yes (settings) | Yes (settings) | Yes (settings) | Yes (settings) | No (session only) |
| **Timeout Default** | 10 min | 30 sec | 60 sec | 10 min | 5 sec |
| **Async Support** | Yes | No | No | No | No |
| **Use Case** | Scripts, git | Validation | Complex checks | Webhooks | Programmatic |
| **Input Access** | stdin JSON | $ARGUMENTS | $ARGUMENTS | JSON body | Message history |
| **Output Format** | JSON/stdout | JSON | Structured tool | JSON | Boolean |

## Key Files

| File | Purpose |
|------|---------|
| `/home/claudeuser/ClaudeCode/utils/hooks.ts` | Main hook execution, matching, aggregation |
| `/home/claudeuser/ClaudeCode/utils/hooks/execCommandHook.ts` | Shell command execution (inline in hooks.ts) |
| `/home/claudeuser/ClaudeCode/utils/hooks/execPromptHook.ts` | Prompt hook execution |
| `/home/claudeuser/ClaudeCode/utils/hooks/execAgentHook.ts` | Agent hook execution |
| `/home/claudeuser/ClaudeCode/utils/hooks/execHttpHook.ts` | HTTP hook execution |
| `/home/claudeuser/ClaudeCode/utils/hooks/sessionHooks.ts` | Session-scoped function hooks |
| `/home/claudeuser/ClaudeCode/utils/hooks/hookEvents.ts` | Event emission system |
| `/home/claudeuser/ClaudeCode/utils/hooks/hookHelpers.ts` | Shared utilities |
| `/home/claudeuser/ClaudeCode/types/hooks.ts` | TypeScript type definitions |
| `/home/claudeuser/ClaudeCode/schemas/hooks.ts` | Zod validation schemas |
| `/home/claudeuser/ClaudeCode/entrypoints/sdk/coreTypes.ts` | SDK hook event constants |

## Hook Event Flow Example

```
User submits prompt
    |
    v
UserPromptSubmit hook fires
    |
    v
Match hooks by event (no matcher query)
    |
    v
Filter by `if` conditions
    |
    v
Execute all hooks in parallel
    |
    v
Aggregate results
    |
    v
Check for blocking errors
    |
    v
Add additionalContext to system prompt
    |
    v
Continue with tool execution
    |
    v
PreToolUse hook fires (for each tool)
    |
    v
Match by tool_name, filter by `if`
    |
    v
Execute hooks, check permissionDecision
    |
    v
Allow/block/ask based on hook output
    |
    v
PostToolUse hook fires
```

## Async Hook Protocol

Command hooks can return async response on first line:
```json
{"async": true, "asyncTimeout": 60000}
```

This backgrounds the hook process:
- Process continues running without blocking main execution
- Output captured via `AsyncHookRegistry`
- Results injected back when complete
- `asyncRewake` mode wakes model on exit code 2 (blocking error)

## Prompt Request Protocol

Command hooks can request user input mid-execution:
```json
// Hook stdout
{"prompt": "request-id", "message": "Question?", "options": [...]}
```

System displays prompt, user responds, response written back to hook stdin as:
```json
{"prompt_response": "request-id", "selected": "option-key"}
```
