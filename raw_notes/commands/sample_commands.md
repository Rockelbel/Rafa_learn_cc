# Claude Code Command Implementation Analysis

This document analyzes sample command implementations from the Claude Code codebase, focusing on the different command types and their patterns.

## Command Type Overview

Claude Code supports three main command types defined in `/home/claudeuser/ClaudeCode/types/command.ts`:

1. **PromptCommand** (`type: 'prompt'`) - Sends text prompts to the AI model
2. **LocalCommand** (`type: 'local'`) - Executes local JavaScript/TypeScript functions
3. **LocalJSXCommand** (`type: 'local-jsx'`) - Renders React/Ink-based UI components

---

## 1. PromptCommand with `getPromptForCommand`

**File**: `/home/claudeuser/ClaudeCode/commands/commit.ts`

Prompt commands expand into text that is sent to the AI model. They define a `getPromptForCommand` async function that returns content blocks.

### Key Pattern

```typescript
import type { Command } from '../commands.js'

const ALLOWED_TOOLS = [
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git commit:*)',
]

const command = {
  type: 'prompt',
  name: 'commit',
  description: 'Create a git commit',
  allowedTools: ALLOWED_TOOLS,
  contentLength: 0, // Dynamic content
  progressMessage: 'creating commit',
  source: 'builtin',
  async getPromptForCommand(_args, context) {
    const promptContent = getPromptContent()
    const finalContent = await executeShellCommandsInPrompt(
      promptContent,
      context,
      '/commit',
    )
    return [{ type: 'text', text: finalContent }]
  },
} satisfies Command

export default command
```

### Characteristics

- **Type**: `'prompt'`
- **Core Method**: `getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>`
- **Return**: Array of content blocks (typically text) sent to the model
- **Allowed Tools**: Optional whitelist of tools the model can use
- **Progress Message**: Shown while the command is executing
- **Source**: `'builtin'`, `'mcp'`, `'plugin'`, etc.

### Arguments

- `_args`: String containing any arguments passed after the command name
- `context`: `ToolUseContext` with app state, message history, and utilities

---

## 2. LocalCommand with Lazy `load()`

**Index File**: `/home/claudeuser/ClaudeCode/commands/clear/index.ts`
**Implementation File**: `/home/claudeuser/ClaudeCode/commands/clear/clear.ts`

Local commands execute JavaScript/TypeScript code directly. They use a lazy loading pattern to defer importing heavy dependencies until the command is invoked.

### Index File Pattern (Metadata Only)

```typescript
import type { Command } from '../../commands.js'

const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history and free up context',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false,
  load: () => import('./clear.js'),
} satisfies Command

export default clear
```

### Implementation File Pattern

```typescript
import type { LocalCommandCall } from '../../types/command.js'
import { clearConversation } from './conversation.js'

export const call: LocalCommandCall = async (_, context) => {
  await clearConversation(context)
  return { type: 'text', value: '' }
}
```

### Characteristics

- **Type**: `'local'`
- **Core Method**: `call(args: string, context: LocalJSXCommandContext): Promise<LocalCommandResult>`
- **Lazy Loading**: `load: () => import('./clear.js')` defers module loading
- **Return**: `LocalCommandResult` with type `'text'`, `'compact'`, or `'skip'`

### LocalCommandResult Types

```typescript
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

---

## 3. LocalJSXCommand with React UI

**Index File**: `/home/claudeuser/ClaudeCode/commands/config/index.ts`
**Implementation File**: `/home/claudeuser/ClaudeCode/commands/config/config.tsx`

JSX commands render interactive terminal UI using React and Ink. They receive an `onDone` callback to signal completion.

### Index File Pattern (Metadata Only)

```typescript
import type { Command } from '../../commands.js'

const config = {
  aliases: ['settings'],
  type: 'local-jsx',
  name: 'config',
  description: 'Open config panel',
  load: () => import('./config.js'),
} satisfies Command

export default config
```

### Simple Implementation Pattern

```typescript
import * as React from 'react'
import { Settings } from '../../components/Settings/Settings.js'
import type { LocalJSXCommandCall } from '../../types/command.js'

export const call: LocalJSXCommandCall = async (onDone, context) => {
  return <Settings onClose={onDone} context={context} defaultTab="Config" />
}
```

### Complex Implementation Pattern (from plan.tsx)

```typescript
import * as React from 'react'
import { Box, Text } from '../../ink.js'
import type { LocalJSXCommandCall, LocalJSXCommandOnDone } from '../../types/command.js'

export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
  args: string,
): Promise<React.ReactNode> {
  const { getAppState, setAppState } = context
  const appState = getAppState()

  // Parse arguments
  const argList = args.trim().split(/\s+/)

  // Handle different modes
  if (argList[0] === 'open') {
    // Do something
    onDone('Opened plan in editor')
    return null
  }

  // Return React component for rendering
  return (
    <Box flexDirection="column">
      <Text bold>Current Plan</Text>
      <Text>{planContent}</Text>
    </Box>
  )
}
```

### Characteristics

- **Type**: `'local-jsx'`
- **Core Method**: `call(onDone: LocalJSXCommandOnDone, context: LocalJSXCommandContext, args: string): Promise<React.ReactNode>`
- **UI Framework**: React with Ink components (`Box`, `Text`, etc.)
- **Completion**: Call `onDone()` when finished

### Command Completion Callback (`onDone`)

```typescript
type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay  // 'skip' | 'system' | 'user'
    shouldQuery?: boolean           // Send messages to model after completion
    metaMessages?: string[]         // Additional meta messages
    nextInput?: string              // Pre-fill next input
    submitNextInput?: boolean       // Auto-submit next input
  },
) => void
```

---

## 4. Command Argument Parsing

Arguments are passed as a single string and parsed manually:

```typescript
// From plan.tsx - parsing "open" or description arguments
const argList = args.trim().split(/\s+/)
if (argList[0] === 'open') {
  // Handle "/plan open"
}

const description = args.trim()
if (description && description !== 'open') {
  onDone('Enabled plan mode', { shouldQuery: true })
}
```

### Argument Hints

Commands can define `argumentHint` for UI typeahead:

```typescript
const plan = {
  type: 'local-jsx',
  name: 'plan',
  description: 'Enable plan mode or view the current session plan',
  argumentHint: '[open|<description>]',  // Shown in gray after command
  load: () => import('./plan.js'),
} satisfies Command
```

---

## 5. Command Base Properties

All commands share common metadata properties:

```typescript
type CommandBase = {
  name: string                    // Command name (e.g., 'commit')
  description: string             // Short description for help/UI
  aliases?: string[]              // Alternative names (e.g., ['reset', 'new'])
  argumentHint?: string           // Argument hint for typeahead
  whenToUse?: string              // Detailed usage scenarios
  isEnabled?: () => boolean      // Conditional enablement
  isHidden?: boolean              // Hide from typeahead/help
  availability?: CommandAvailability[] // Auth/provider requirements
  // ... more options
}
```

---

## 6. Command Registration

Commands are imported and registered in `/home/claudeuser/ClaudeCode/commands.ts`:

```typescript
import commit from './commands/commit.js'
import clear from './commands/clear/index.js'
import config from './commands/config/index.js'
import plan from './commands/plan/index.js'
import help from './commands/help/index.js'

const COMMANDS = memoize((): Command[] => [
  commit,
  clear,
  config,
  plan,
  help,
  // ... more commands
])
```

---

## Summary Table

| Aspect | PromptCommand | LocalCommand | LocalJSXCommand |
|--------|---------------|--------------|-----------------|
| **Type** | `'prompt'` | `'local'` | `'local-jsx'` |
| **Entry Point** | `getPromptForCommand` | `call` (via `load()`) | `call` (via `load()`) |
| **Return Type** | `Promise<ContentBlockParam[]>` | `Promise<LocalCommandResult>` | `Promise<React.ReactNode>` |
| **Primary Use** | AI model prompts | Local execution | Interactive UI |
| **Lazy Loading** | No (direct import) | Yes (`load: () => import(...)`) | Yes (`load: () => import(...)`) |
| **Completion** | N/A (model handles) | Return result object | Call `onDone()` callback |
| **Arguments** | `args: string`, `context: ToolUseContext` | `args: string`, `context: LocalJSXCommandContext` | `onDone`, `context`, `args: string` |
| **Key Features** | `allowedTools`, `progressMessage` | `supportsNonInteractive` | React/Ink components |

---

## Key Files Reference

- **Type Definitions**: `/home/claudeuser/ClaudeCode/types/command.ts`
- **Command Registry**: `/home/claudeuser/ClaudeCode/commands.ts`
- **Prompt Command Example**: `/home/claudeuser/ClaudeCode/commands/commit.ts`
- **Local Command Example**: `/home/claudeuser/ClaudeCode/commands/clear/index.ts`, `/home/claudeuser/ClaudeCode/commands/clear/clear.ts`
- **JSX Command Examples**:
  - `/home/claudeuser/ClaudeCode/commands/config/index.ts`, `/home/claudeuser/ClaudeCode/commands/config/config.tsx`
  - `/home/claudeuser/ClaudeCode/commands/plan/index.ts`, `/home/claudeuser/ClaudeCode/commands/plan/plan.tsx`
  - `/home/claudeuser/ClaudeCode/commands/help/index.ts`, `/home/claudeuser/ClaudeCode/commands/help/help.tsx`
