# State Management Analysis

## Store Implementation Pattern (/home/claudeuser/ClaudeCode/state/store.ts)

The store implements a **minimal, framework-agnostic pub/sub pattern** with functional updates:

### Core Types
```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

### Implementation Details
- **State container**: Simple closure variable (`let state = initialState`)
- **Listeners**: Set-based subscription management for O(1) add/remove
- **Update pattern**: Functional updater `(prev: T) => T` (immutable pattern)
- **Reference equality check**: Uses `Object.is()` to skip updates when state unchanged
- **Change notification**: Optional `onChange` callback receives old/new state for side effects
- **Subscription cleanup**: Returns unsubscribe function

This is a lightweight Redux/Zustand-inspired pattern without the middleware complexity.

---

## App State Structure (/home/claudeuser/ClaudeCode/state/AppStateStore.ts)

The `AppState` is a large, feature-rich state object (~450 lines) with:

### Key Design Patterns

1. **DeepImmutable wrapper**: `DeepImmutable<{}>` for type-level immutability
2. **Explicit mutable exceptions**: `tasks`, `agentNameRegistry` excluded from immutability (contain functions/Map)

### State Categories

| Category | Fields |
|----------|--------|
| **Settings** | `settings`, `mainLoopModel`, `mainLoopModelForSession`, `verbose` |
| **UI State** | `statusLineText`, `expandedView`, `isBriefOnly`, `footerSelection`, `activeOverlays` |
| **Agent/Teammate** | `selectedIPAgentIndex`, `coordinatorTaskIndex`, `viewSelectionMode`, `viewingAgentTaskId`, `foregroundedTaskId` |
| **Bridge/Remote** | `replBridgeEnabled`, `replBridgeConnected`, `remoteSessionUrl`, `remoteConnectionStatus` |
| **Permissions** | `toolPermissionContext` (centralized permission mode) |
| **MCP** | `mcp.clients`, `mcp.tools`, `mcp.commands`, `mcp.resources` |
| **Plugins** | `plugins.enabled`, `plugins.disabled`, `plugins.errors`, `plugins.needsRefresh` |
| **Tasks** | `tasks: { [taskId: string]: TaskState }` - keyed by ID |
| **Notifications** | `notifications.current`, `notifications.queue` |
| **Auth** | `authVersion` - counter for triggering auth-dependent refetches |
| **Feature Flags** | `kairosEnabled`, `thinkingEnabled`, `fastMode` |
| **Experimental** | `speculation`, `ultraplanSessionUrl`, `bagelActive` |

### Default State Factory
`getDefaultAppState()` creates a comprehensive initial state including:
- Settings initialization
- Empty collections (Maps, Sets, Arrays)
- Permission mode detection (teammate vs leader)
- Feature flag evaluation

---

## Selectors for Derived State (/home/claudeuser/ClaudeCode/state/selectors.ts)

Minimal, pure selector pattern for computed values:

### Pattern
```typescript
// Selectors accept minimal state shape via Pick<>
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>
): InProcessTeammateTaskState | undefined
```

### Current Selectors
1. **getViewedTeammateTask**: Returns the task being viewed (with type validation)
2. **getActiveAgentForInput**: Returns discriminated union for input routing:
   - `{ type: 'leader' }` - input goes to main agent
   - `{ type: 'viewed', task }` - input goes to viewed teammate
   - `{ type: 'named_agent', task }` - input goes to named local agent

### Selector Philosophy
- Pure data extraction (no side effects)
- Type-safe with discriminated unions
- Minimal state dependency (Pick for reusability)
- Used for input routing and view logic

---

## Change Notification & Side Effects (/home/claudeuser/ClaudeCode/state/onChangeAppState.ts)

The `onChangeAppState` function is the **centralized side effect handler** invoked on every state change.

### Architecture
```
setState(updater)
    ↓
Object.is check
    ↓
state = next
    ↓
onChange?.({ newState, oldState })  ← side effects here
    ↓
notify listeners
```

### Key Responsibilities

| State Change | Side Effect |
|--------------|-------------|
| `toolPermissionContext.mode` | Notify CCR/SDK of permission mode changes |
| `mainLoopModel` → `null` | Remove from settings |
| `mainLoopModel` → value | Save to settings |
| `expandedView` | Persist to global config (backward compat) |
| `verbose` | Persist to global config |
| `tungstenPanelVisible` | Persist to global config (ant-only) |
| `settings` | Clear auth caches (API key, AWS, GCP) |
| `settings.env` | Re-apply environment variables |

### Permission Mode Sync Pattern
The most critical section - ensures **single source of truth** for permission mode:
- Any state mutation that changes mode notifies CCR via `notifySessionMetadataChanged()`
- SDK notified via `notifyPermissionModeChanged()`
- External mode conversion filters internal-only modes (bubble, ungated auto)
- Ultraplan mode detection for initial plan cycle

### State Restoration
`externalMetadataToAppState()` provides inverse mapping for restoring state from external metadata (used on worker restart).

---

## State Update Flow

```
┌─────────────────────────────────────────────────────────────┐
│  Component/Service calls setState(updater)                  │
│  Example: setState(prev => ({ ...prev, verbose: true }))   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Store compares prev/next with Object.is()                  │
│  If equal → skip update                                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Update state reference                                     │
│  state = next                                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  onChange callback (side effects)                           │
│  - Sync to external systems (CCR, SDK)                      │
│  - Persist to disk (config, settings)                       │
│  - Clear caches                                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Notify all subscribers                                     │
│  listeners.forEach(l => l())                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Components re-render (React via useSyncExternalStore)      │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect | Pattern |
|--------|---------|
| Store | Minimal pub/sub with functional updates |
| State Shape | Monolithic, feature-grouped, with immutable wrapper |
| Updates | `(prev) => next` updater functions |
| Change Detection | `Object.is()` reference equality |
| Side Effects | Centralized in `onChangeAppState` |
| Derived State | Pure selectors with Pick<> for minimal deps |
| Persistence | Handled in onChange (config, settings) |
| External Sync | Permission mode, CCR metadata in onChange |
