# Claude Code Query System Analysis

## Overview

The query system in Claude Code is the core orchestration layer that manages the conversation loop with the LLM, handling message processing, tool execution, and response streaming.

## Key File

- `/home/claudeuser/ClaudeCode/query.ts` (~3000+ lines) - Main query execution

## Query Execution Flow

### 1. Entry Point: `query()`

The main query function receives:
- Messages to send
- Tool use context
- Abort controller
- Options (debug, verbose, etc.)

### 2. Message Preparation

```typescript
// Messages are normalized for the API
const normalizedMessages = normalizeMessagesForAPI(messages, {
  includeToolResults: true,
  includeSystemMessages: true,
})
```

Key preparation steps:
- Message normalization
- Context prepending (user context)
- System context appending
- Attachment filtering
- Memory prefetching

### 3. API Request

```typescript
const response = await queryModel({
  messages: normalizedMessages,
  tools,
  abortController,
  ...
})
```

### 4. Response Streaming

The query system handles streaming responses with:
- **Event types**: `message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`
- **Content types**: text, tool_use, thinking, redacted_thinking
- **Accumulation**: Content is accumulated block by block

### 5. Tool Execution

When tool_use blocks are received:

```typescript
// Tool calls are collected and executed
const toolResults = await runTools({
  toolUses,
  context: toolUseContext,
  ...
})
```

Tool execution uses `StreamingToolExecutor` for concurrent execution with safety controls.

### 6. Response Processing

After receiving the complete response:
- Messages are normalized for storage
- Tool results are integrated
- Context is updated
- Hooks are executed (PostSampling, Stop, etc.)

## Key Components

### StreamingMessageAccumulator

Accumulates streaming content into complete messages:
- Text content accumulation
- Tool input accumulation (JSON parsing)
- Thinking block handling
- Multi-block ordering

### Tool Result Integration

Tool results are:
- Mapped to ToolResultBlockParam format
- Integrated into the message history
- Potentially summarized (if `toolSummary` feature enabled)
- Used for context updates

### Error Handling

The query system handles various error types:
- `PROMPT_TOO_LONG_ERROR_MESSAGE` - Context window exceeded
- `FallbackTriggeredError` - Model fallback
- `ImageSizeError` / `ImageResizeError` - Image validation
- `AbortError` - User cancellation

### Token Management

- Token estimation for context
- Auto-compact when approaching limits
- Warning states at thresholds
- Content replacement for long tool results

### Feature Flag Integration

Conditional features via `feature()`:
- `REACTIVE_COMPACT` - Dynamic compaction
- `CONTEXT_COLLAPSE` - Context management
- `EXPERIMENTAL_SKILL_SEARCH` - Skill prefetch
- `TEMPLATES` - Job classification

## State Updates

During query execution, the AppState is updated:
- Messages appended/updated
- Tool progress tracked
- Token counts recorded
- Notifications enqueued

## Loop Control

The query loop continues until:
- User interruption (Ctrl+C)
- Natural completion (no more tool uses)
- Error (unrecoverable)
- Abort signal

## Performance Optimizations

- Parallel tool execution (concurrency-safe only)
- Streaming response processing
- Incremental message updates
- Lazy loading of heavy features
- Profile checkpoints (`queryCheckpoint`)

## Integration Points

- **Tools**: Tool execution and result handling
- **Hooks**: Pre/Post sampling hooks
- **Analytics**: Event logging
- **State**: AppState updates
- **Streaming**: Real-time UI updates

## Architecture Patterns

1. **Event-driven streaming** - SSE events processed incrementally
2. **Generator-based** - Async generators for tool execution
3. **Functional updates** - Immutable state updates
4. **Abort controller** - Cancellation support throughout
5. **Layered error handling** - Classification and recovery

