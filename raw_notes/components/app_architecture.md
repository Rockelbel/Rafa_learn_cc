# Claude Code Core App Component Architecture Analysis

## Overview

This document analyzes the core App components in the Claude Code codebase, focusing on the React-based terminal UI architecture.

## Component Hierarchy

```
App.tsx (Root Provider Wrapper)
├── FpsMetricsProvider
├── StatsProvider
└── AppStateProvider
    └── Children (actual application content)

Messages.tsx (Message List Container)
├── LogoHeader (React.memo - memoized header)
├── VirtualMessageList (when scrollRef present)
│   └── MessageRow (virtualized)
└── Direct render: renderableMessages.flatMap(renderMessageRow)
    ├── MessageRow (wrapped in MessageActionsSelectedContext.Provider)
    │   └── Message.tsx (message type dispatcher)
    │       ├── UserMessage (user content types)
    │       │   ├── UserTextMessage
    │       │   ├── UserImageMessage
    │       │   └── UserToolResultMessage
    │       ├── AssistantMessageBlock (assistant content types)
    │       │   ├── AssistantTextMessage
    │       │   ├── AssistantToolUseMessage
    │       │   ├── AssistantThinkingMessage
    │       │   ├── AssistantRedactedThinkingMessage
    │       │   └── AdvisorMessage
    │       ├── AttachmentMessage
    │       ├── SystemTextMessage
    │       ├── GroupedToolUseContent
    │       └── CollapsedReadSearchContent
    ├── Unseen Divider (conditional)
    ├── StreamingText (live preview)
    └── StreamingThinking (live thinking)
```

## Key Components Analysis

### 1. App.tsx - Root Provider Composition

**Location:** `/home/claudeuser/ClaudeCode/components/App.tsx`

**Purpose:** Top-level wrapper providing context to the component tree.

**Architecture Pattern:**
- Uses React Compiler runtime (`_c` function from "react/compiler-runtime")
- Implements manual memoization with cache slots ($[0], $[1], etc.)
- Three-layer provider nesting: FpsMetricsProvider → StatsProvider → AppStateProvider

**State Management:**
```typescript
interface Props {
  getFpsMetrics: () => FpsMetrics | undefined
  stats?: StatsStore
  initialState: AppState
  children: React.ReactNode
}
```

**React Compiler Pattern:**
The compiled output shows explicit cache management:
```javascript
const $ = _c(9);  // 9 cache slots
if ($[0] !== children || $[1] !== initialState) {
  // Recompute and cache
  $[0] = children;
  $[1] = initialState;
  $[2] = t1;
} else {
  t1 = $[2];  // Use cached value
}
```

### 2. Messages.tsx - Message List Container

**Location:** `/home/claudeuser/ClaudeCode/components/Messages.tsx`

**Purpose:** Main message list component handling message transformation, virtualization, and rendering.

**Key State Management:**

| Hook | Purpose |
|------|---------|
| `useMemo(normalizedMessages)` | Normalize and filter empty messages |
| `useMemo(isStreamingThinkingVisible)` | Check if streaming thinking should show (30s timeout) |
| `useMemo(lastThinkingBlockId)` | Find last thinking block for hiding past thinking |
| `useMemo(latestBashOutputUUID)` | Track latest bash output for auto-expansion |
| `useMemo(renderableMessages)` | Apply grouping, collapsing, and slicing |
| `useState(expandedKeys)` | Track clicked-to-expand message keys |
| `useCallback(onItemClick)` | Toggle expanded state on click |
| `useRef(sliceAnchorRef)` | Track message slice anchor for virtualization |
| `useRef(searchTextCache)` | WeakMap for search text caching |

**Message Transformation Pipeline:**

```
raw messages
  → normalizeMessages() → normalizedMessages
  → filter(isNotEmptyMessage)
  → getMessagesAfterCompactBoundary() [if not verbose/fullscreen]
  → reorderMessagesInUI()
  → filter(shouldShowUserMessage)
  → filterForBriefTool() / dropTextInBriefTurns() [brief mode]
  → slice(-MAX_MESSAGES_TO_SHOW_IN_TRANSCRIPT_MODE) [transcript mode]
  → applyGrouping() → groupedMessages
  → collapseReadSearchGroups()
  → collapseTeammateShutdowns()
  → collapseHookSummaries()
  → collapseBackgroundBashNotifications()
  → collapsed (final renderable array)
```

**Performance Optimizations:**

1. **Message Cap with UUID Anchoring:**
   - `MAX_MESSAGES_WITHOUT_VIRTUALIZATION = 200`
   - Uses UUID-based slice anchor instead of count-based slicing
   - Prevents full terminal reset on every append (CC-941)
   - Quantized to 50-message steps (CC-1154)

2. **Memoization Strategy:**
   - `LogoHeader`: Memoized on agentDefinitions only
   - `MessagesImpl`: Custom comparator ignoring callback refs
   - Streaming tool uses compared by contentBlock reference
   - Set equality for inProgressToolUseIDs

3. **Virtual Scrolling:**
   - Conditional VirtualMessageList when scrollRef present
   - Falls back to flatMap for non-virtualized path
   - `disableRenderCap` option for headless renders

### 3. Message.tsx - Message Type Dispatcher

**Location:** `/home/claudeuser/ClaudeCode/components/Message.tsx`

**Purpose:** Switch component dispatching to appropriate message sub-components based on type.

**Message Type Dispatch:**

```typescript
switch (message.type) {
  case 'attachment': return <AttachmentMessage />
  case 'assistant': return <Box>{message.message.content.map(...)}</Box>
  case 'user': return isCompactSummary ? <CompactSummary /> : <UserMessage />
  case 'system': return handleSystemSubtypes()
  case 'grouped_tool_use': return <GroupedToolUseContent />
  case 'collapsed_read_search': return <OffscreenFreeze><CollapsedReadSearchContent /></OffscreenFreeze>
}
```

**Assistant Message Content Types:**

| Type | Component |
|------|-----------|
| `tool_use` | AssistantToolUseMessage |
| `text` | AssistantTextMessage |
| `redacted_thinking` | AssistantRedactedThinkingMessage (conditional) |
| `thinking` | AssistantThinkingMessage (conditional) |
| `server_tool_use` / `advisor_tool_result` | AdvisorMessage (if isAdvisorBlock) |
| `connector_text` | AssistantTextMessage (feature flag) |

**User Message Content Types:**

| Type | Component |
|------|-----------|
| `text` | UserTextMessage |
| `image` | UserImageMessage |
| `tool_result` | UserToolResultMessage |

**Memoization:**
- Exported `areMessagePropsEqual` for custom comparison
- `React.memo(MessageImpl, areMessagePropsEqual)`
- Checks: uuid, lastThinkingBlockId, verbose, latestBashOutputUUID, isTranscriptMode, containerWidth, isStatic

### 4. MessageRow.tsx - Message Row Layout

**Location:** `/home/claudeuser/ClaudeCode/components/MessageRow.tsx`

**Purpose:** Individual message row with metadata, progress tracking, and static rendering decisions.

**Key Computations:**

1. **isActiveCollapsedGroup:**
   ```typescript
   isCollapsed && (hasAnyToolInProgress(msg, inProgressToolUseIDs) || (isLoading && !hasContentAfter))
   ```

2. **displayMsg:** Normalizes grouped/collapsed messages to display format

3. **progressMessagesForMessage:** Lookup from message lookups (empty for grouped/collapsed)

4. **isStatic:** Computed via `shouldRenderStatically()`

5. **shouldAnimate:** Based on tool progress state

**Static Rendering Decision:**
```typescript
shouldRenderStatically(msg, streamingToolUseIDs, inProgressToolUseIDs, siblingToolUseIDs, screen, lookups)
```

Returns `true` for:
- Transcript mode (always static)
- Messages without tool_use IDs
- Resolved tool uses (in resolvedToolUseIDs)
- Non-streaming, resolved messages

**Memoization:**
- `areMessageRowPropsEqual`: Conservative comparator
- Bails out only when CERTAIN message won't change
- Checks: message reference, screen, verbose, columns, latestBashOutputUUID, lastThinkingBlockId, isStreaming, isResolved

## React Hooks Usage Patterns

### 1. React Compiler Integration

All components use the React Compiler runtime (`_c` function) which provides:
- Automatic memoization via cache slots
- Manual dependency checking with `$[n]` slots
- Batched updates and reduced re-renders

Example pattern:
```javascript
const $ = _c(64);  // Declare 64 cache slots
if ($[0] !== dep1 || $[1] !== dep2) {
  // Recompute
  $[0] = dep1;
  $[1] = dep2;
  $[2] = result;
} else {
  result = $[2];
}
```

### 2. useMemo Patterns

**Message Transformation Chain:**
```typescript
const normalizedMessages = useMemo(() =>
  normalizeMessages(messages).filter(isNotEmptyMessage),
  [messages]
);
```

**Expensive Lookups:**
```typescript
const { collapsed, lookups, hasTruncatedMessages, hiddenMessageCount } = useMemo(() => {
  // Complex O(n) transforms over 27k messages
  // Build Maps, filter, group, collapse
}, [verbose, normalizedMessages, isTranscriptMode, ...]);
```

**Cheap Slicing (separate from transforms):**
```typescript
const renderableMessages = useMemo(() => {
  const sliceStart = computeSliceStart(collapsed_0, sliceAnchorRef);
  return collapsed_0.slice(sliceStart);
}, [collapsed_0, renderRange, virtualScrollRuntimeGate]);
```

### 3. useCallback Patterns

**Stable Callbacks:**
```typescript
const onItemClick = useCallback((msg: RenderableMessage) => {
  const k = expandKey(msg);
  setExpandedKeys(prev => {
    const next = new Set(prev);
    if (next.has(k)) next.delete(k); else next.add(k);
    return next;
  });
}, []);
```

**Ref-based Staleness Avoidance:**
```typescript
const lookupsRef = useRef(lookups_0);
lookupsRef.current = lookups_0;
const isItemClickable = useCallback((msg: RenderableMessage): boolean => {
  // Access lookups via ref to avoid callback churn
  const name = lookupsRef.current.toolUseByToolUseID.get(...);
}, [tools]); // tools is session-stable
```

### 4. useRef Patterns

**Anchor Tracking:**
```typescript
const sliceAnchorRef = useRef<SliceAnchor>(null);
// Mutated during render (idempotent under StrictMode)
```

**Cache Storage:**
```typescript
const searchTextCache = useRef(new WeakMap<RenderableMessage, string>());
// WeakMap allows GC when messages are no longer referenced
```

**Previous State Comparison:**
```typescript
const prevProgressState = useRef<string | null>(null);
useEffect(() => {
  if (prevProgressState.current === state) return;
  prevProgressState.current = state;
  progress(state);
}, [progress, progressEnabled, hasToolsInProgress]);
```

## State Management Architecture

### 1. Context Providers (from App.tsx)

| Provider | Purpose |
|----------|---------|
| FpsMetricsProvider | FPS tracking for performance monitoring |
| StatsProvider | Session statistics aggregation |
| AppStateProvider | Core application state with onChangeAppState callback |

### 2. Component-Level State

**Messages.tsx:**
- `expandedKeys`: Set of user-expanded message keys
- `sliceAnchorRef`: Mutable ref for message slice window
- `searchTextCache`: WeakMap for search text memoization

**MessageRow.tsx (computed, not stored):**
- `isActiveCollapsedGroup`: Derived from tool progress
- `isStatic`: Derived from streaming/resolution state
- `shouldAnimate`: Derived from inProgressToolUseIDs

### 3. Message Lookups

**buildMessageLookups** creates efficient access structures:
```typescript
interface MessageLookups {
  toolUseByToolUseID: Map<string, ToolUseInfo>
  resolvedToolUseIDs: Set<string>
  erroredToolUseIDs: Set<string>
  // ... other indices
}
```

Used for:
- O(1) tool result lookup by ID
- Progress message association
- Sibling tool use ID retrieval

## Performance Optimizations

### 1. Memoization Strategy

| Component | Memoization Type | Key Dependencies |
|-----------|-----------------|------------------|
| LogoHeader | React.memo | agentDefinitions |
| Messages | React.memo + custom comparator | Deep comparison for streamingToolUses, sets |
| Message | React.memo + areMessagePropsEqual | uuid, lastThinkingBlockId, verbose, etc. |
| MessageRow | React.memo + areMessageRowPropsEqual | message ref, screen, streaming state |

### 2. Virtualization

**VirtualMessageList:**
- Only renders visible messages
- Height caching for scroll position stability
- Sticky prompt tracking via ScrollChromeContext

**OffscreenFreeze:**
- Caches offscreen message elements
- Prevents re-render of scrolled-out content
- Returns cached element ref for zero-diff bailout

### 3. Batched Updates

**Message Cap with UUID Anchoring:**
- Prevents per-frame full terminal resets
- Quantized advancement (50-message steps)
- Anchor survives collapse/regrouping operations

### 4. Search Optimization

**Search Text Caching:**
```typescript
const searchTextCache = useRef(new WeakMap<RenderableMessage, string>());
// Cache LOWERED: setSetQuery's hot loop indexOfs per keystroke
// Lowering here (once, at warm) vs there (every keystroke)
```

### 5. Transform Separation

Transforms are split into two useMemo hooks:
1. **Expensive transforms** (grouping, collapsing, lookups) - run when messages change
2. **Cheap slice** - run when scroll range changes

This prevents ~50ms alloc per scroll from rebuilding 6 Maps over 27k messages.

## Tool Use Rendering Integration

### Tool Use Flow

1. **Streaming Phase:**
   - `streamingToolUses` updates on every input_json_delta
   - Synthetic streaming messages created with deterministic UUIDs
   - `isStatic = false` for streaming tool uses

2. **In-Progress Phase:**
   - `inProgressToolUseIDs` Set tracks executing tools
   - Animation continues while tool in progress
   - `shouldAnimate = true`

3. **Completed Phase:**
   - Tool use ID added to `resolvedToolUseIDs`
   - `isStatic = true` enables memoization
   - Animation stops

### Tool Result Rendering

**UserToolResultMessage** receives:
- `param`: The tool_result block
- `message`: Full user message
- `lookups`: For tool use lookup
- `progressMessagesForMessage`: Associated progress messages
- `style`: 'condensed' or undefined
- `tools`: Tool definitions
- `verbose`: Expanded detail view
- `width`: Terminal columns - 5
- `isTranscriptMode`: Screen mode

## Keyboard Input Handling

Keyboard handling is primarily managed at the screen level (REPL.tsx), not in these components. The components receive:

- `cursor`: MessageActionsState for selection
- `setCursor`: Callback to update selection
- `cursorNavRef`: Ref for navigation actions
- `jumpRef`: For transcript search jumping

Selection state flows through:
```
MessageActionsSelectedContext.Provider
  → MessageRow
    → Message
      → Individual message components (can read selection state)
```

## Key Findings

1. **React Compiler Integration:** All components use the React Compiler runtime with manual cache slot management for optimal memoization.

2. **Multi-Layer Memoization:** Components use React.memo with custom equality functions, preventing unnecessary re-renders during streaming.

3. **Transform Pipeline:** Message processing follows a clear pipeline with expensive operations memoized and cheap operations separated.

4. **Virtualization Support:** Dual-path rendering (virtualized vs direct) based on scrollRef presence.

5. **UUID-Based Anchoring:** Clever message capping strategy using UUID anchors to prevent scroll position loss during updates.

6. **WeakMap Caching:** Search text caching uses WeakMap for automatic GC when messages are no longer referenced.

7. **Static Rendering Optimization:** `shouldRenderStatically` enables aggressive memoization for completed, non-streaming messages.
