# Compact Service Analysis

## Overview

The Compact Service is responsible for managing Claude Code's context window through message summarization and compaction. It handles both automatic and manual compaction to prevent conversations from exceeding token limits.

## Core Files

| File | Purpose |
|------|---------|
| `/home/claudeuser/ClaudeCode/services/compact/compact.ts` | Main compaction logic and API |
| `/home/claudeuser/ClaudeCode/services/compact/autoCompact.ts` | Auto-compact triggers and decision-making |
| `/home/claudeuser/ClaudeCode/services/compact/grouping.ts` | Message grouping by API round |
| `/home/claudeuser/ClaudeCode/services/compact/prompt.ts` | Summary prompts and formatting |
| `/home/claudeuser/ClaudeCode/services/compact/sessionMemoryCompact.ts` | Session memory-based compaction |
| `/home/claudeuser/ClaudeCode/services/compact/microCompact.ts` | Micro-compaction for tool results |
| `/home/claudeuser/ClaudeCode/services/compact/postCompactCleanup.ts` | Cleanup after compaction |

---

## Compaction Strategies

### 1. Full Compaction (`compactConversation`)

Summarizes the entire conversation history into a compact summary message.

**Process:**
1. Execute pre-compact hooks for custom instructions
2. Strip images and reinjected attachments from messages
3. Stream summary generation via forked agent or direct API
4. Create compact boundary marker and summary messages
5. Generate post-compact attachments (files, skills, plans)
6. Execute session start and post-compact hooks
7. Return compacted message structure

**Key Features:**
- Uses `stripImagesFromMessages()` to remove image/document blocks (replaced with `[image]`/`[document]` markers)
- Uses `stripReinjectedAttachments()` to filter out skill_discovery/skill_listing attachments
- Prompt cache sharing via forked agent for efficiency
- Handles prompt-too-long (PTL) errors with `truncateHeadForPTLRetry()`

### 2. Partial Compaction (`partialCompactConversation`)

Summarizes a portion of the conversation based on a pivot message.

**Directions:**
- `'from'`: Summarize messages after pivot, keep earlier ones (prefix-preserving)
- `'up_to'`: Summarize messages before pivot, keep later ones (suffix-preserving)

**Use Case:** User selects a specific message to compact around.

### 3. Session Memory Compaction (`trySessionMemoryCompaction`)

Uses pre-extracted session memory as the summary instead of generating one.

**Process:**
1. Check if session memory compaction is enabled (feature flags)
2. Wait for any in-progress session memory extraction
3. Calculate messages to keep based on `lastSummarizedMessageId`
4. Expand to meet minimum token/message requirements
5. Adjust for tool_use/tool_result pairs
6. Create compaction result from session memory content

**Configuration (via GrowthBook):**
```typescript
minTokens: 10_000          // Minimum tokens to preserve
minTextBlockMessages: 5    // Minimum messages with text blocks
maxTokens: 40_000          // Maximum tokens to preserve (hard cap)
```

### 4. Micro-Compaction (`microcompactMessages`)

Lightweight compaction that clears old tool result content without full summarization.

**Two Paths:**

#### a) Time-Based Micro-Compact
Triggered when gap since last assistant message exceeds threshold (default: 30 minutes).
- Content-clears old tool results
- Keeps N most recent tool results
- Mutates message content directly

#### b) Cached Micro-Compact (Ant-only)
Uses Anthropic's cache editing API:
- Tracks registered tools in `cachedMCState`
- Queues `cache_edits` blocks for API layer
- Does NOT modify local message content
- Uses count-based triggers from GrowthBook config

**Compactable Tools:**
- FileRead, FileEdit, FileWrite
- Shell tools (Bash, etc.)
- Grep, Glob
- WebSearch, WebFetch

---

## Token Management

### Token Estimation

```typescript
// From microCompact.ts
export function estimateMessageTokens(messages: Message[]): number
```

**Method:**
- Sums tokens from text blocks, tool results, images (~2000 tokens each)
- Pads estimate by 4/3 for conservatism
- Images/documents estimated at fixed 2000 tokens

### Context Window Calculation

```typescript
// From autoCompact.ts
export function getEffectiveContextWindowSize(model: string): number
```

**Formula:**
```
effectiveWindow = contextWindow - reservedTokensForSummary
reservedTokensForSummary = min(maxOutputTokensForModel, 20_000)
```

The 20,000 token reserve is based on p99.99 of compact summary output (17,387 tokens).

### Buffer Thresholds

```typescript
AUTOCOMPACT_BUFFER_TOKENS = 13_000      // Trigger autocompact at (window - 13k)
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // Warning level
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000   // Error level
MANUAL_COMPACT_BUFFER_TOKENS = 3_000     // Blocking limit for manual
```

---

## Auto-Compact Triggers

### Trigger Conditions

**`shouldAutoCompact()` returns true when:**
1. Auto-compact is enabled (not disabled via env or config)
2. Query source is not `session_memory` or `compact` (recursion guards)
3. Not in reactive-only mode (when feature flag is on)
4. Context collapse is not enabled
5. Token count exceeds auto-compact threshold

**Token Check:**
```typescript
tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
threshold = getAutoCompactThreshold(model)
```

### Auto-Compact Threshold

```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

Default: ~13,000 tokens before context window limit.

### Circuit Breaker

After 3 consecutive auto-compact failures, auto-compact stops retrying to prevent API hammering:

```typescript
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### Auto-Compact Flow (`autoCompactIfNeeded`)

1. Check if disabled via `DISABLE_COMPACT` env
2. Check circuit breaker (consecutive failures >= 3)
3. Check if should auto-compact (token threshold)
4. **Try session memory compaction first** (experimental)
5. If SM fails, fall back to legacy `compactConversation`
6. Run post-compact cleanup
7. Return result with failure tracking

---

## Message Summarization

### Summary Prompt Structure

From `prompt.ts`, the compaction prompt includes:

**NO_TOOLS_PREAMBLE:**
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Tool calls will be REJECTED and will waste your only turn
- Your entire response must be plain text: an <analysis> block followed by a <summary> block
```

**Summary Sections Required:**
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections (with snippets)
4. Errors and fixes
5. Problem Solving
6. All user messages (non-tool results)
7. Pending Tasks
8. Current Work
9. Optional Next Step

### Analysis Block

The model is instructed to wrap analysis in `<analysis>` tags as a drafting scratchpad. This is stripped by `formatCompactSummary()` before storing.

### Summary Formatting

```typescript
export function formatCompactSummary(summary: string): string
```

- Strips `<analysis>...</analysis>` blocks
- Extracts and formats `<summary>...</summary>` content
- Replaces XML tags with readable headers

---

## Context Window Management

### Message Grouping

```typescript
// From grouping.ts
export function groupMessagesByApiRound(messages: Message[][])
```

Groups messages at API-round boundaries:
- Boundary fires when a NEW assistant response begins (different `message.id`)
- Used for reactive compact retry logic
- Breaks cycles between compact.ts and compactMessages.ts

### PTL (Prompt Too Long) Retry

```typescript
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage
): Message[] | null
```

When compaction API call hits prompt-too-long:
1. Groups messages by API round
2. Calculates token gap from error response
3. Drops oldest groups until gap is covered
4. Falls back to dropping 20% if gap unparseable
5. Prepends synthetic marker if resulting in assistant-first sequence

### Post-Compact Attachments

After compaction, these attachments are re-injected:

1. **File Attachments** (`createPostCompactFileAttachments`)
   - Up to 5 most recently accessed files
   - 50,000 token budget, 5,000 tokens per file max
   - Skips files already in preserved messages

2. **Plan Attachment** (`createPlanAttachmentIfNeeded`)
   - Preserves active plan file content

3. **Plan Mode Attachment** (`createPlanModeAttachmentIfNeeded`)
   - Reminds model of plan mode if active

4. **Skill Attachment** (`createSkillAttachmentIfNeeded`)
   - Preserves invoked skills content
   - 25,000 token budget, 5,000 tokens per skill
   - Sorted most-recent-first

5. **Async Agent Attachments** (`createAsyncAgentAttachmentsIfNeeded`)
   - Status of running/finished background agents

6. **Delta Attachments**
   - Deferred tools delta
   - Agent listing delta
   - MCP instructions delta

---

## Configuration & Feature Flags

### Environment Variables

```bash
DISABLE_COMPACT=true           # Disable all compaction
DISABLE_AUTO_COMPACT=true      # Disable auto-compact only
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80  # Override threshold percentage
CLAUDE_CODE_AUTO_COMPACT_WINDOW=100000  # Override context window
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE=50000  # Override blocking limit
ENABLE_CLAUDE_CODE_SM_COMPACT=true  # Enable session memory compaction
```

### Feature Flags

```typescript
// From autoCompact.ts and other files
feature('CONTEXT_COLLAPSE')     // Disable autocompact when collapse enabled
feature('REACTIVE_COMPACT')     // Reactive-only mode
feature('PROMPT_CACHE_BREAK_DETECTION')  // Cache break detection
feature('KAIROS')               // Session transcript writing
feature('CACHED_MICROCOMPACT')  // Cache editing micro-compact
```

### GrowthBook Configs

```typescript
// Session memory compact config
tengu_sm_compact_config: {
  minTokens: number
  minTextBlockMessages: number
  maxTokens: number
}

// Cached micro-compact config
tengu_cache_plum_violet: boolean  // Enable cached MC
tengu_compact_cache_prefix: boolean  // Prompt cache sharing
tengu_compact_streaming_retry: boolean  // Streaming retry
```

---

## Key Data Structures

### CompactionResult

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage        // Compact boundary metadata
  summaryMessages: UserMessage[]       // Summary content
  attachments: AttachmentMessage[]     // Post-compact attachments
  hookResults: HookResultMessage[]     // Session start hooks
  messagesToKeep?: Message[]           // For partial compact
  userDisplayMessage?: string          // Display info from hooks
  preCompactTokenCount?: number
  postCompactTokenCount?: number       // API call total usage
  truePostCompactTokenCount?: number   // Resulting context size
  compactionUsage?: TokenUsage
}
```

### RecompactionInfo

```typescript
export type RecompactionInfo = {
  isRecompactionInChain: boolean       // Same-chain loop detection
  turnsSincePreviousCompact: number
  previousCompactTurnId?: string
  autoCompactThreshold: number
  querySource?: QuerySource
}
```

### AutoCompactTrackingState

```typescript
export type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  turnId: string
  consecutiveFailures?: number  // Circuit breaker counter
}
```

---

## Analytics Events

| Event | Description |
|-------|-------------|
| `tengu_compact` | Full compaction completed |
| `tengu_partial_compact` | Partial compaction completed |
| `tengu_compact_failed` | Compaction failure (various reasons) |
| `tengu_compact_ptl_retry` | PTL retry attempt |
| `tengu_compact_cache_sharing_success` | Forked agent cache hit |
| `tengu_compact_cache_sharing_fallback` | Fallback to streaming |
| `tengu_sm_compact_*` | Session memory compaction events |
| `tengu_cached_microcompact` | Cached micro-compact events |
| `tengu_time_based_microcompact` | Time-based micro-compact |

---

## Integration Points

### Pre/Post Compact Hooks

```typescript
executePreCompactHooks({
  trigger: 'auto' | 'manual',
  customInstructions: string | null
})

executePostCompactHooks({
  trigger: 'auto' | 'manual',
  compactSummary: string
})
```

### Session Start Hooks

Called after compaction to restore context:
```typescript
processSessionStartHooks('compact', { model })
```

### Post-Compact Cleanup

```typescript
runPostCompactCleanup(querySource)
```

Clears:
- Micro-compact state
- Context collapse state (main thread only)
- User context cache
- System prompt sections
- Classifier approvals
- Speculative checks
- Beta tracing state
- Session messages cache

---

## Summary

The Compact Service provides a multi-layered approach to context management:

1. **Prevention**: Micro-compact clears old tool results incrementally
2. **Automatic**: Auto-compact triggers at 13k buffer before limit
3. **Efficient**: Session memory compaction avoids API calls when possible
4. **Flexible**: Partial compact lets users compact around specific messages
5. **Robust**: PTL retry, circuit breakers, and fallback paths handle edge cases
