# Claude Code Message System Analysis

## Overview

The Message System in Claude Code handles conversation state, message types, normalization for API communication, and message utilities. This document analyzes the core message architecture based on the codebase.

## Message Type Hierarchy

### Core Message Types

The system defines several distinct message types that form the conversation state:

```
Message (union type)
├── UserMessage
├── AssistantMessage
├── SystemMessage
├── ProgressMessage
└── AttachmentMessage
```

### 1. UserMessage

**Purpose**: Represents user input to the system

**Key Fields**:
- `type: 'user'` - Discriminator field
- `message: { role: 'user', content: string | ContentBlockParam[] }` - The actual message content
- `uuid: UUID` - Unique identifier
- `timestamp: string` - ISO timestamp
- `isMeta?: true` - Whether this is a metadata message (not visible to model)
- `isVisibleInTranscriptOnly?: true` - Transcript-only visibility
- `isVirtual?: true` - Virtual message flag
- `isCompactSummary?: true` - Whether this is a compaction summary
- `toolUseResult?: unknown` - Result from tool execution (for tool_result messages)
- `imagePasteIds?: number[]` - IDs of pasted images
- `sourceToolAssistantUUID?: UUID` - Reference to source assistant message
- `permissionMode?: PermissionMode` - Permission mode at message creation
- `origin?: MessageOrigin` - Message provenance
- `mcpMeta?: {...}` - MCP protocol metadata (never sent to model)

**Special Variants**:
- **Tool Result Messages**: User messages with `toolUseResult` field set, containing tool execution results
- **Meta Messages**: Hidden messages (isMeta=true) for system coordination
- **Compact Summary Messages**: Generated during conversation compaction

### 2. AssistantMessage

**Purpose**: Represents assistant/model responses

**Key Fields**:
- `type: 'assistant'` - Discriminator field
- `message: BetaMessage` - API response containing content blocks
- `uuid: UUID` - Unique identifier
- `timestamp: string` - ISO timestamp
- `isMeta?: true` - Metadata flag
- `isVirtual?: true` - Virtual message flag
- `isApiErrorMessage?: true` - Marks synthetic error messages
- `apiError?: {...}` - API error details
- `error?: SDKAssistantMessageError` - Error classification
- `errorDetails?: string` - Error details
- `advisorModel?: string` - Model used for advisory content
- `requestId?: string` - Associated request ID

**Content Structure**:
```typescript
message: {
  id: string          // API message ID
  type: 'message'
  role: 'assistant'
  model: string       // Model identifier
  content: BetaContentBlock[]  // Array of content blocks
  stop_reason: BetaStopReason
  stop_sequence: string
  usage: Usage        // Token usage statistics
  context_management: null
}
```

### 3. SystemMessage

**Purpose**: System-level events and notifications

**Subtypes**:
- `init` - Initial system configuration
- `compact_boundary` - Conversation compaction marker
- `status` - Status updates (e.g., compacting)
- `api_error` - API error notifications
- `api_retry` - API retry information
- `local_command` - Local command output
- `task_notification` - Task completion notifications
- `task_started` - Task start notifications
- `task_progress` - Task progress updates
- `hook_started/progress/response` - Hook lifecycle events
- `compact` - Compaction events
- `microcompact_boundary` - Micro-compaction markers
- `session_state_changed` - Session state changes
- `files_persisted` - File persistence events
- `elicitation_complete` - MCP elicitation completion
- `post_turn_summary` - Post-turn summary
- `bridge_status` - Bridge connection status
- `agents_killed` - Agent termination events
- `memory_saved` - Memory persistence events
- `permission_retry` - Permission retry events
- `turn_duration` - Turn timing information
- `away_summary` - Away-mode summary
- `stop_hook_summary` - Stop hook summary
- `informational` - General informational messages

**Common Fields**:
- `type: 'system'`
- `subtype: string` - Specific subtype discriminator
- `uuid: UUID`
- `timestamp: string`
- `level?: SystemMessageLevel` - 'info' | 'warning' | 'error' | 'success'

### 4. ProgressMessage

**Purpose**: Real-time progress updates during tool execution

**Key Fields**:
- `type: 'progress'`
- `toolUseID: string` - Associated tool use ID
- `parentToolUseID: string` - Parent tool use ID
- `data: P` - Progress data (generic)
- `uuid: UUID`
- `timestamp: string`

**Progress Types**:
- Hook progress events
- Tool execution progress
- Task progress updates

### 5. AttachmentMessage

**Purpose**: Attachments to messages (hooks, permissions, tool results)

**Subtypes**:
- Hook attachments (success, error, cancelled, etc.)
- Permission decision attachments
- Tool use summary attachments

## Normalized Message Types

The system uses normalized message variants for UI rendering:

```
NormalizedMessage
├── NormalizedUserMessage
├── NormalizedAssistantMessage<T = BetaContentBlock>
└── (System, Progress, Attachment pass through unchanged)
```

### Normalization Process

**Purpose**: Split multi-block messages into individual messages, one per content block

**Key Behavior**:
1. Single-block messages pass through with UUID preserved
2. Multi-block messages are split, with derived UUIDs for subsequent blocks
3. String content is wrapped in text blocks
4. Image paste IDs are distributed to corresponding image blocks

**UUID Derivation**:
```typescript
function deriveUUID(parentUUID: UUID, index: number): UUID {
  const hex = index.toString(16).padStart(12, '0')
  return `${parentUUID.slice(0, 24)}${hex}` as UUID
}
```

### NormalizedUserMessage

Same as UserMessage but with guaranteed array content (single text block for string input).

### NormalizedAssistantMessage

Assistant message with single content block and derived UUID if part of a split.

## Message Flow

### 1. Creation Flow

```
User Input
    ↓
createUserMessage() → UserMessage
    ↓
(normalization) → NormalizedUserMessage[]
    ↓
API Request (ContentBlockParam[])

API Response (BetaMessage)
    ↓
baseCreateAssistantMessage() → AssistantMessage
    ↓
(normalization) → NormalizedAssistantMessage[]
    ↓
UI Rendering
```

### 2. Tool Use Flow

```
AssistantMessage with tool_use blocks
    ↓
Tool execution
    ↓
Tool result creation
    ↓
UserMessage with toolUseResult
    ↓
Normalization → NormalizedUserMessage with tool_result content
```

### 3. Message Reordering

The `reorderMessagesInUI()` function groups related messages:
- Tool use request
- Pre-hooks (attachments)
- Tool result
- Post-hooks (attachments)

## Message Utilities

### Predicate Functions (messagePredicates.ts)

**isHumanTurn(m: Message)**: Determines if a message is from a human user
- Checks `type === 'user'`
- Excludes meta messages (`!m.isMeta`)
- Excludes tool results (`m.toolUseResult === undefined`)

**Key Insight**: Tool result messages share `type: 'user'` with human turns, so the discriminant is the optional `toolUseResult` field.

### Creation Helpers (messages.ts)

**User Messages**:
- `createUserMessage(options)` - General user message creation
- `createUserInterruptionMessage(options)` - Interruption messages
- `createSyntheticUserCaveatMessage()` - Synthetic caveat for local commands
- `prepareUserContent()` - Prepares content blocks

**Assistant Messages**:
- `createAssistantMessage(options)` - Standard assistant message
- `createAssistantAPIErrorMessage(options)` - Error message creation
- `baseCreateAssistantMessage(options)` - Internal base function

**Progress Messages**:
- `createProgressMessage(options)` - Progress update creation

### Message Validation

**isNotEmptyMessage(message)**: Filters empty messages
- Skips progress, attachment, system messages (always considered non-empty)
- Checks string content for non-empty text
- Validates content blocks for non-text or non-empty text

**isSyntheticMessage(message)**: Detects synthetic messages
- Checks for predefined synthetic content strings
- Used for interrupt/cancel/reject messages

### Message Lookups (buildMessageLookups)

Builds O(1) lookup structures:
- `siblingToolUseIDs` - Maps tool use ID to sibling IDs in same request
- `progressMessagesByToolUseID` - Progress messages per tool
- `toolResultByToolUseID` - Tool result lookup
- `toolUseByToolUseID` - Tool use block lookup
- `inProgressHookCounts` - Active hook counts
- `resolvedHookCounts` - Resolved hook counts
- `resolvedToolUseIDs` / `erroredToolUseIDs` - Resolution tracking

### Message Normalization Functions

**normalizeMessages(messages)**:
```typescript
// Overloads for type preservation
normalizeMessages(messages: AssistantMessage[]): NormalizedAssistantMessage[]
normalizeMessages(messages: UserMessage[]): NormalizedUserMessage[]
normalizeMessages(messages: (AssistantMessage | UserMessage)[]): NormalizedMessage[]
```

**Behavior**:
- Flattens multi-block messages into individual messages
- Generates derived UUIDs for split messages
- Preserves all metadata

**Type Guards**:
- `isToolUseRequestMessage(message)` - Checks for tool_use content
- `isToolUseResultMessage(message)` - Checks for tool_result or toolUseResult

### Message Extraction

**extractTextContent(blocks, separator)**:
- Extracts text from ContentBlockParam[]
- Handles string or array input
- Joins with specified separator

**extractTag(html, tagName)**:
- Extracts content from XML-style tags
- Handles nested tags
- Case-insensitive matching

## SDK Integration

### SDK Message Types (entrypoints/sdk/coreSchemas.ts)

The SDK defines parallel message schemas:
- `SDKUserMessage` - User messages for SDK consumers
- `SDKAssistantMessage` - Assistant messages
- `SDKSystemMessage` - System messages
- `SDKResultMessage` - Query result messages
- `SDKProgressMessage` - Progress updates

### Message Mappers (utils/messages/mappers.ts)

**toInternalMessages(sdkMessages)**:
- Converts SDK messages to internal format
- Handles assistant, user, and compact_boundary system messages

**toSDKMessages(internalMessages)**:
- Converts internal messages to SDK format
- Filters non-convertible message types
- Normalizes assistant messages for SDK consumption

## Command Queue Integration

The message system integrates with the command queue (messageQueueManager.ts):

**QueuedCommand** can contain:
- String input
- ContentBlockParam[] (with images)
- Associated metadata (pastedContents, origin, etc.)

**Editable Queue Commands**:
- `isQueuedCommandEditable(cmd)` - Checks if command can be edited
- `isQueuedCommandVisible(cmd)` - Checks if command should be visible
- `popAllEditable()` - Extracts editable commands for user input

## Key Design Patterns

### 1. Discriminated Unions

All message types use discriminated unions with a `type` field:
```typescript
type Message = UserMessage | AssistantMessage | SystemMessage | ProgressMessage | AttachmentMessage
```

### 2. Type Predicates

Type guards for safe narrowing:
```typescript
function isHumanTurn(m: Message): m is UserMessage
function isToolUseRequestMessage(m: Message): m is ToolUseRequestMessage
```

### 3. Normalization for Rendering

Multi-block messages are split for individual rendering:
- Each content block becomes its own message
- UUIDs are derived deterministically
- Parent relationships are preserved

### 4. Virtual/Synthetic Messages

Special message types for system coordination:
- Not sent to the model
- Used for UI state, hooks, and internal tracking
- Marked with `isMeta`, `isVirtual`, or `isSynthetic`

### 5. Content Block Architecture

Messages use Anthropic's content block pattern:
- Text blocks
- Image blocks
- Tool use blocks
- Tool result blocks
- Thinking blocks
- Redacted thinking blocks

## Constants and Messages

**NO_CONTENT_MESSAGE**: `'(no content)'` - Placeholder for empty content

**Synthetic Messages Set**:
- INTERRUPT_MESSAGE
- INTERRUPT_MESSAGE_FOR_TOOL_USE
- CANCEL_MESSAGE
- REJECT_MESSAGE
- NO_RESPONSE_REQUESTED

**Rejection Messages**:
- Built-in rejection templates for various scenarios
- Support for custom rejection reasons
- Subagent-specific rejection messages

## Summary

The Message System provides:
1. **Type Safety**: Discriminated unions and type predicates
2. **Flexibility**: Multiple message variants for different use cases
3. **Normalization**: Unified representation for UI rendering
4. **Extensibility**: Hook and attachment system for custom behaviors
5. **SDK Compatibility**: Bidirectional mapping to SDK types
6. **Tool Integration**: First-class support for tool use/result pairing
