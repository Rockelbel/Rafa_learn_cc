# Claude Code Streaming System Analysis

## Overview

The Claude Code streaming system implements a multi-layered architecture for handling real-time data flow from the Anthropic API through to the user interface. The system is designed around async generators, custom Stream implementations, and sophisticated buffering strategies.

## Core Streaming Architecture

### 1. Stream Class (`utils/stream.ts`)

The foundation of the streaming system is a custom `Stream<T>` class that implements `AsyncIterator<T>`:

**Key Components:**
- **Queue-based buffering**: Internal queue (`T[]`) stores values before consumption
- **Promise-based backpressure**: Uses `readResolve`/`readReject` to handle async iteration
- **Single-iteration safety**: `started` flag ensures stream can only be consumed once
- **Completion signaling**: `isDone` flag and `done()` method for clean termination
- **Error propagation**: `hasError` state with `error()` method for exception handling

**Design Pattern:**
```typescript
// Producer side: enqueue values as they arrive
stream.enqueue(value)
stream.done()  // Signal completion

// Consumer side: for-await loop
for await (const value of stream) { ... }
```

### 2. API Response Streaming (`services/api/claude.ts`)

The `queryModel()` function is the primary streaming generator that consumes the Anthropic SDK's raw stream.

**Stream Event Processing:**
- **message_start**: Initializes partial message, captures TTFT (Time To First Token)
- **content_block_start**: Creates placeholder blocks (text, tool_use, thinking)
- **content_block_delta**: Accumulates deltas into content blocks
  - `text_delta`: Appends to text blocks
  - `input_json_delta`: Accumulates JSON for tool inputs
  - `thinking_delta`: Accumulates thinking content
  - `signature_delta`: Captures thinking signatures
- **content_block_stop**: Yields complete `AssistantMessage`
- **message_delta**: Updates usage stats and stop_reason
- **message_stop**: Signals message completion

**Dual Streaming Path:**
1. **Primary path**: Raw SDK stream with full event processing
2. **Fallback path**: Non-streaming mode for error recovery

### 3. Streaming Tool Executor (`services/tools/StreamingToolExecutor.ts`)

Executes tools as they stream in from the API with concurrency control.

**Concurrency Model:**
- **Concurrent-safe tools**: Execute in parallel (e.g., Read, WebFetch)
- **Non-concurrent tools**: Execute exclusively (e.g., Bash, File Edit)
- **Queue management**: Tools are queued and processed based on concurrency rules

**Status Tracking:**
```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

**Progress Streaming:**
- Tools can yield progress messages during execution
- `pendingProgress` queue for immediate progress yield
- `progressAvailableResolve` signal wakes up consumer

## Chunk Processing

### 1. API Stream Chunk Handling

The `for await (const part of stream)` loop in `queryModel()` processes chunks with:

**Stall Detection:**
- Tracks `lastEventTime` between chunks
- Logs warnings for gaps > 30 seconds
- Accumulates total stall time for telemetry

**Idle Timeout Watchdog:**
- `STREAM_IDLE_TIMEOUT_MS` (default 90s) aborts hung streams
- Separate warning at 50% timeout
- Prevents indefinite hangs from silently dropped connections

**Content Accumulation:**
```typescript
// Tool input accumulation (JSON streaming)
contentBlock.input += delta.partial_json

// Text accumulation
contentBlock.text += delta.text

// Thinking accumulation
contentBlock.thinking += delta.thinking
```

### 2. Message Assembly

At `content_block_stop`, a complete `AssistantMessage` is assembled:
- Content blocks are normalized via `normalizeContentFromAPI()`
- UUID and timestamp assigned
- Research metadata attached (internal builds)
- Advisor model tracking (if enabled)

### 3. Stream Event Yielding

Every API event yields a `StreamEvent` for telemetry/UI:
```typescript
yield {
  type: 'stream_event',
  event: part,
  ttftMs,  // Only on message_start
}
```

## Real-Time UI Updates

### 1. Structured IO (`cli/structuredIO.ts`)

The SDK protocol layer manages bidirectional streaming:

**Input Stream:**
- `AsyncGenerator<StdinMessage | SDKMessage>` from `read()`
- Line-buffered NDJSON parsing
- Prepended message support for injected turns

**Output Stream:**
- `Stream<StdoutMessage>` via `outbound` queue
- Control request/response protocol for permissions
- Hook callback support

### 2. Streamlined Transform (`utils/streamlinedTransform.ts`)

Distillation-resistant output format for SDK consumers:

**Transformation Rules:**
- Text messages: Preserved intact
- Tool-only messages: Summarized as counts ("searched 3 patterns, read 2 files")
- Cumulative counting: Resets when text appears
- Thinking content: Omitted

**Tool Categories:**
- `searches`: Grep, Glob, WebSearch, LSP
- `reads`: FileRead, ListMcpResources
- `writes`: FileWrite, FileEdit, NotebookEdit
- `commands`: Bash, PowerShell, Tmux, TaskStop

### 3. Progress Messages

Tools emit progress via `onToolProgress` callback:
```typescript
onToolProgress({
  toolUseID: string,
  data: ToolProgressData
})
```

Progress is wrapped in `ProgressMessage` and yielded immediately through the streaming executor.

## Buffering Strategies

### 1. Queue-Based Buffering

**Stream Class (`utils/stream.ts`):**
- Unbounded queue array for value buffering
- Automatic draining when consumer is ready
- FIFO ordering guarantee

**StreamingToolExecutor:**
- `pendingProgress` per-tool queues
- Completed results buffer until yielded in order

### 2. Line Buffering (`streamJsonStdoutGuard.ts`)

Protects NDJSON stream output from corruption:

**Mechanism:**
- Intercepts `process.stdout.write`
- Buffers until newline
- Validates JSON before forwarding
- Diverts non-JSON to stderr with marker

**Cleanup:**
- Flushes partial lines on exit
- Restores original write function

### 3. Content Block Accumulation

**In `queryModel()`:**
```typescript
const contentBlocks: (BetaContentBlock | ConnectorTextBlock)[] = []
```

Blocks are accumulated by index from `content_block_start`, then mutated by `content_block_delta`, and finalized at `content_block_stop`.

### 4. Tool Result Buffering

**StreamingToolExecutor buffering:**
- Tools may complete out of order due to concurrency
- Results buffered in `TrackedTool.results`
- `getCompletedResults()` yields in tool reception order
- Non-concurrent tools block subsequent tools until yielded

## Error Handling and Recovery

### 1. Streaming Fallback

On stream errors, the system falls back to non-streaming mode:
- Captures partial state before fallback
- Discards incomplete tool executions
- Retries with full request

### 2. Streaming Fallback Prevention

`CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` prevents fallback when:
- Streaming tool execution is active
- Prevents double tool execution

### 3. Abort Handling

Multi-level abort propagation:
- User abort (ESC key): Triggers immediate cleanup
- Timeout abort: SDK internal timeout
- Sibling abort: Bash error cancels parallel tools
- Watchdog abort: Idle timeout kills hung streams

## Performance Optimizations

### 1. Raw Stream Usage

Uses raw `Stream<BetaRawMessageStreamEvent>` instead of `BetaMessageStream`:
- Avoids O(n²) partial JSON parsing
- Handles tool input accumulation internally

### 2. Fire-and-Forget Telemetry

Analytics events use `void logEvent()` to not block streaming.

### 3. Lazy JSON Serialization

Transcript writes use 100ms lazy flush interval to batch serialization.

### 4. Cache Break Detection

`PROMPT_CACHE_BREAK_DETECTION` feature tracks:
- Prompt state hashing
- Cache hit/miss analysis
- Automatic recompaction on break

## File Locations

| Component | Path |
|-----------|------|
| Stream Class | `/home/claudeuser/ClaudeCode/utils/stream.ts` |
| API Streaming | `/home/claudeuser/ClaudeCode/services/api/claude.ts` |
| Tool Executor | `/home/claudeuser/ClaudeCode/services/tools/StreamingToolExecutor.ts` |
| Tool Execution | `/home/claudeuser/ClaudeCode/services/tools/toolExecution.ts` |
| Query Engine | `/home/claudeuser/ClaudeCode/query.ts` |
| Structured IO | `/home/claudeuser/ClaudeCode/cli/structuredIO.ts` |
| Streamlined Transform | `/home/claudeuser/ClaudeCode/utils/streamlinedTransform.ts` |
| Stdout Guard | `/home/claudeuser/ClaudeCode/utils/streamJsonStdoutGuard.ts` |
