# Claude Code Hooks Execution System Analysis

## Overview

The Hooks Execution System is a flexible event-driven mechanism that allows users and plugins to inject custom logic at specific points in Claude Code's lifecycle. Hooks can be defined via shell commands, LLM prompts, HTTP requests, or in-memory TypeScript callbacks.

---

## Directory Structure

```
/home/claudeuser/ClaudeCode/utils/hooks/
├── AsyncHookRegistry.ts          # Manages async hook lifecycle and registry
├── hookEvents.ts                 # Event emission system for hook execution
├── execAgentHook.ts              # Agent-based hook execution (multi-turn LLM)
├── execPromptHook.ts             # Prompt-based hook execution (single-turn LLM)
├── execHttpHook.ts               # HTTP-based hook execution
├── hookHelpers.ts                # Shared utilities (response schema, structured output)
├── hooksConfigManager.ts         # Configuration grouping and metadata
├── hooksConfigSnapshot.ts        # Captures hook config at startup
├── hooksSettings.ts              # Hook loading from all sources
├── registerSkillHooks.ts         # Registers skill frontmatter hooks
├── registerFrontmatterHooks.ts   # Registers agent frontmatter hooks
├── sessionHooks.ts               # Session-scoped and function hooks
├── fileChangedWatcher.ts         # File watching for FileChanged hooks
├── apiQueryHookHelper.ts         # API query helpers
├── postSamplingHooks.ts          # Post-sampling hook handling
├── skillImprovement.ts           # Skill improvement tracking
├── ssrfGuard.ts                  # SSRF protection for HTTP hooks
└── sessionHooks.ts               # Session hook management
```

---

## Hook Types

### 1. Command Hooks (`type: 'command'`)
- Execute shell commands (bash or PowerShell)
- Receive JSON input via stdin
- Return JSON output via stdout
- Support async execution with `{"async": true}`

### 2. Prompt Hooks (`type: 'prompt'`)
- Single-turn LLM query using small fast model
- Uses `$ARGUMENTS` substitution for JSON input
- Returns `{"ok": true}` or `{"ok": false, "reason": "..."}`
- Timeout: 30 seconds default

### 3. Agent Hooks (`type: 'agent'`)
- Multi-turn LLM agent for complex validation
- Can use tools to inspect codebase
- Maximum 50 turns
- Uses `StructuredOutputTool` for responses
- Timeout: 60 seconds default

### 4. HTTP Hooks (`type: 'http'`)
- POST JSON to external URL
- Supports header interpolation from env vars
- SSRF protection with allowlist
- Proxy support (sandbox and env-var)
- Timeout: 10 minutes default

### 5. Callback Hooks (`type: 'callback'`)
- In-memory TypeScript functions
- Registered by plugins or internal code
- Synchronous or async execution

### 6. Function Hooks (`type: 'function'`)
- Session-scoped callbacks
- Cannot be persisted to settings
- Used for internal validations (e.g., structured output enforcement)

---

## Hook Events

Defined in `/home/claudeuser/ClaudeCode/entrypoints/sdk/coreTypes.ts`:

| Event | Trigger | Input |
|-------|---------|-------|
| `PreToolUse` | Before tool execution | Tool name, input, use ID |
| `PostToolUse` | After tool success | Tool name, input, response |
| `PostToolUseFailure` | After tool failure | Tool name, input, error |
| `PermissionDenied` | Auto-mode denial | Tool name, input, reason |
| `PermissionRequest` | Permission dialog shown | Tool name, input, use ID |
| `UserPromptSubmit` | User submits prompt | Original prompt text |
| `SessionStart` | New session starts | Source (startup/resume/clear/compact) |
| `SessionEnd` | Session ends | Reason (clear/logout/etc) |
| `Stop` | Before response concludes | - |
| `StopFailure` | Turn ends due to API error | Error type |
| `SubagentStart` | Subagent starts | Agent ID, type |
| `SubagentStop` | Before subagent concludes | Agent ID, type, transcript path |
| `PreCompact` | Before conversation compaction | Trigger (manual/auto) |
| `PostCompact` | After conversation compaction | Trigger, summary |
| `Setup` | Repo init/maintenance | Trigger (init/maintenance) |
| `Notification` | Notification sent | Message, type |
| `TeammateIdle` | Teammate about to go idle | Teammate name, team name |
| `TaskCreated` | Task being created | Task ID, subject, description |
| `TaskCompleted` | Task being completed | Task ID, subject, description |
| `Elicitation` | MCP requests user input | Server name, message, schema |
| `ElicitationResult` | User responds to elicitation | Server name, action, content |
| `ConfigChange` | Settings file changes | Source, file path |
| `InstructionsLoaded` | CLAUDE.md loaded | File path, memory type, load reason |
| `CwdChanged` | Working directory changes | Old cwd, new cwd |
| `FileChanged` | Watched file changes | File path, event type |
| `WorktreeCreate` | Creating worktree | Suggested name |
| `WorktreeRemove` | Removing worktree | Path |

---

## Hook Registration Sources

### Priority Order (highest to lowest):

1. **User Settings** (`~/.claude/settings.json`)
2. **Project Settings** (`.claude/settings.json`)
3. **Local Settings** (`.claude/settings.local.json`)
4. **Policy Settings** (managed/admin settings)
5. **Session Hooks** (in-memory, temporary)
6. **Plugin Hooks** (`~/.claude/plugins/*/hooks/hooks.json`)
7. **Built-in Hooks** (internal callbacks)

### Source Types:

```typescript
export type HookSource =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'policySettings'
  | 'pluginHook'
  | 'sessionHook'
  | 'builtinHook'
```

---

## Execution Flow

### 1. Hook Configuration Loading

```
hooksSettings.ts:getAllHooks()
  -> Loads from user/project/local settings
  -> Loads from session hooks (getSessionHooks)
  -> Returns IndividualHookConfig[]
```

### 2. Hook Grouping and Matching

```
hooksConfigManager.ts:groupHooksByEventAndMatcher()
  -> Groups hooks by event type
  -> Groups by matcher (tool name, notification type, etc.)
  -> Includes registered plugin hooks
  -> Includes session hooks
```

### 3. Hook Matching Algorithm

```typescript
// Matches pattern against query
function matchesPattern(matchQuery: string, matcher: string): boolean

// Supports:
// - Simple exact match: "Write"
// - Pipe-separated: "Write|Edit|Read"
// - Regex patterns: "^Write.*", ".*", "^(Write|Edit)$"
// - Wildcard: "*" (matches all)
// - Legacy tool name normalization
```

### 4. Hook Execution (executeHooks generator)

```typescript
async function* executeHooks({
  hookInput,        // Structured input data
  toolUseID,        // Tracking ID
  matchQuery,       // Tool name or matcher query
  signal,           // AbortSignal
  timeoutMs,        // Execution timeout
  toolUseContext,   // For prompt/agent hooks
  messages,         // Conversation history
  forceSyncExecution, // Override async
  requestPrompt,    // Interactive prompt callback
})
```

### Execution Steps:

1. **Disable Check**: Skip if `disableAllHooks` or `CLAUDE_CODE_SIMPLE` mode
2. **Trust Check**: Skip if workspace trust not accepted (interactive mode only)
3. **Hook Matching**: Find matching hooks via `getHooksConfig()`
4. **Deduplication**: Remove duplicate hooks by command/prompt/URL
5. **If-Condition Filtering**: Evaluate `if` conditions for tool hooks
6. **Sequential Execution**: Execute hooks in priority order
7. **Result Aggregation**: Yield aggregated results as generator

---

## Exit Code Semantics

| Exit Code | Meaning |
|-----------|---------|
| `0` | Success - continue normally |
| `2` | Blocking error - show stderr to model, may block operation |
| Other | Non-blocking error - show stderr to user only |

### Event-Specific Behaviors:

- **PreToolUse**: Exit 2 blocks tool call, shows stderr to model
- **PostToolUse**: Exit 2 shows stderr to model immediately
- **UserPromptSubmit**: Exit 2 blocks processing, erases original prompt
- **PreCompact**: Exit 2 blocks compaction
- **Stop/StopFailure**: Exit 2 shows stderr to model and continues

---

## JSON Output Schema

### Sync Response:
```json
{
  "continue": false,
  "stopReason": "Optional reason",
  "systemMessage": "Warning to user",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "updatedInput": {...}
  }
}
```

### Async Response:
```json
{
  "async": true,
  "asyncTimeout": 30000
}
```

---

## Async Hook Execution

### Two Modes:

1. **Standard Async**: Hook backgrounds, response attached later
   - Registered in `AsyncHookRegistry`
   - Checked via `checkForAsyncHookResponses()`
   - Response delivered as attachment message

2. **Async Rewake** (`asyncRewake: true`):
   - Survives user input (not killed on new prompt)
   - Exit code 2 enqueues task notification to wake model
   - Used for stop hooks that need to block after completion

### Async Hook Registry:

```typescript
// In AsyncHookRegistry.ts
const pendingHooks = new Map<string, PendingAsyncHook>()

// Registration
registerPendingAsyncHook({ processId, hookId, ... })

// Polling for responses
checkForAsyncHookResponses() -> Returns settled hook responses
```

---

## Session Hooks

Session hooks are in-memory, temporary hooks scoped to a session.

### API:

```typescript
// Add session hook
addSessionHook(setAppState, sessionId, event, matcher, hook, onHookSuccess?, skillRoot?)

// Add function hook (returns ID)
addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage, options?)

// Remove function hook
removeFunctionHook(setAppState, sessionId, event, hookId)

// Get session hooks
getSessionHooks(appState, sessionId, event?)

// Clear all session hooks
clearSessionHooks(setAppState, sessionId)
```

### Storage:

```typescript
// SessionHooksState uses Map for O(1) mutations without triggering listeners
export type SessionHooksState = Map<string, SessionStore>

// Pattern prevents N² copies during parallel agent hook registration
```

---

## Context Passing

### Hook Input Structure:

```typescript
interface BaseHookInput {
  hook_event_name: HookEvent
  session_id: string
  transcript_path: string
  cwd: string
  permission_mode?: string
  agent_id?: string
  agent_type?: string
}

// Event-specific inputs extend base:
interface PreToolUseHookInput extends BaseHookInput {
  tool_name: string
  tool_input: Record<string, unknown>
  tool_use_id: string
}

interface PostToolUseHookInput extends BaseHookInput {
  tool_name: string
  tool_input: Record<string, unknown>
  tool_use_id: string
  response: unknown
}

interface UserPromptSubmitHookInput extends BaseHookInput {
  prompt: string
}

interface SessionStartHookInput extends BaseHookInput {
  source: 'startup' | 'resume' | 'clear' | 'compact'
}
```

### Environment Variables:

- `CLAUDE_PROJECT_DIR` - Stable project root
- `CLAUDE_PLUGIN_ROOT` - Plugin/skill directory (for plugin/skill hooks)
- `CLAUDE_PLUGIN_DATA` - Plugin data directory
- `CLAUDE_ENV_FILE` - Path for hooks to write env exports (SessionStart/Setup/CwdChanged/FileChanged)
- `CLAUDE_PLUGIN_OPTION_*` - Plugin user config options

---

## State Management

### Hook Configuration Snapshot

```typescript
// Captured once at startup
captureHooksConfigSnapshot() -> initialHooksConfig

// Respects policy settings:
// - disableAllHooks: true -> empty config
// - allowManagedHooksOnly: true -> only policy hooks
```

### Security Gates:

1. **Managed Hooks Only**: `policySettings.allowManagedHooksOnly`
2. **Disable All Hooks**: `policySettings.disableAllHooks`
3. **Plugin-Only Mode**: `strictPluginOnlyCustomization` blocks user/project/local
4. **Workspace Trust**: All hooks require trust in interactive mode

---

## Event Emission

```typescript
// Hook lifecycle events (for SDK consumers)
emitHookStarted(hookId, hookName, hookEvent)
emitHookProgress({ hookId, hookName, hookEvent, stdout, stderr, output })
emitHookResponse({ hookId, hookName, hookEvent, output, stdout, stderr, exitCode, outcome })

// Always emitted: 'SessionStart', 'Setup'
// Conditionally emitted: others (requires includeHookEvents option)
```

---

## Key Implementation Details

### Deduplication Strategy

```typescript
// Command hooks dedup by: pluginRoot/skillRoot + command
function hookDedupKey(m: MatchedHook, payload: string): string {
  return `${m.pluginRoot ?? m.skillRoot ?? ''}\0${payload}`
}

// Callback/function hooks skip dedup (each is unique)
```

### Shell Selection

```typescript
// Resolution order: hook.shell -> 'bash' (DEFAULT_HOOK_SHELL)
// Supports: 'bash', 'powershell'
// Windows bash uses Git Bash (findGitBashPath())
// PowerShell uses: pwsh -> powershell
```

### Windows Path Handling

```typescript
// Bash hooks: Windows paths -> POSIX (/c/Users/foo)
// PowerShell hooks: Native paths preserved
const toHookPath = isWindows && !isPowerShell
  ? windowsPathToPosixPath
  : (p: string) => p
```

### If-Condition Evaluation

```typescript
// Only for tool events (PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest)
// Uses permission rule matching (same as permission system)
async function prepareIfConditionMatcher(hookInput, tools)
```

---

## Execution Order Summary

```
1. Configuration loaded from all sources (priority order)
2. Hooks grouped by event and matcher
3. For each hook execution:
   a. Filter by matcher pattern
   b. Filter by if-condition (tool hooks only)
   c. Deduplicate hooks
   d. Execute in priority order:
      - User/Project/Local settings hooks
      - Policy settings hooks
      - Session hooks
      - Plugin hooks
      - Built-in hooks
   e. Process results sequentially
   f. Aggregate outputs
4. Handle blocking errors (exit code 2)
5. Handle async responses
6. Emit lifecycle events
```

---

## Files Summary

| File | Responsibility |
|------|----------------|
| `hooks.ts` | Main execution logic, result processing, all `execute*Hooks()` functions |
| `AsyncHookRegistry.ts` | Async hook lifecycle, polling, cleanup |
| `hookEvents.ts` | Event emission for SDK consumers |
| `sessionHooks.ts` | Session-scoped hook storage and retrieval |
| `hooksConfigManager.ts` | Hook grouping, metadata, matcher sorting |
| `hooksConfigSnapshot.ts` | Startup snapshot, policy enforcement |
| `hooksSettings.ts` | Loading hooks from all settings sources |
| `execAgentHook.ts` | Multi-turn agent hook execution |
| `execPromptHook.ts` | Single-turn prompt hook execution |
| `execHttpHook.ts` | HTTP POST hook execution |
| `hookHelpers.ts` | Shared utilities, response schema, structured output |
| `registerSkillHooks.ts` | Skill frontmatter hook registration |
| `registerFrontmatterHooks.ts` | Agent frontmatter hook registration |
