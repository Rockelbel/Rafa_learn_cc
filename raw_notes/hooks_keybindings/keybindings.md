# Claude Code Keybindings System Analysis

## Overview

The Keybindings system in Claude Code is a comprehensive keyboard shortcut framework built on React/Ink. It provides customizable keyboard shortcuts with context-aware resolution, chord sequence support, and hot-reload capabilities.

## Architecture

### Core Files and Responsibilities

| File | Purpose |
|------|---------|
| `/home/claudeuser/ClaudeCode/keybindings/defaultBindings.ts` | Default keybinding definitions |
| `/home/claudeuser/ClaudeCode/keybindings/schema.ts` | Zod schema and action/context definitions |
| `/home/claudeuser/ClaudeCode/keybindings/parser.ts` | Keystroke parsing (e.g., "ctrl+k" -> ParsedKeystroke) |
| `/home/claudeuser/ClaudeCode/keybindings/resolver.ts` | Key resolution logic with chord state support |
| `/home/claudeuser/ClaudeCode/keybindings/match.ts` | Matching logic between Ink keys and bindings |
| `/home/claudeuser/ClaudeCode/keybindings/KeybindingContext.tsx` | React context for keybinding state |
| `/home/claudeuser/ClaudeCode/keybindings/KeybindingProviderSetup.tsx` | Provider setup with hot-reload |
| `/home/claudeuser/ClaudeCode/keybindings/useKeybinding.ts` | Hook for registering keybinding handlers |
| `/home/claudeuser/ClaudeCode/keybindings/loadUserBindings.ts` | User config loading with file watching |
| `/home/claudeuser/ClaudeCode/keybindings/validate.ts` | Validation for user keybinding configs |
| `/home/claudeuser/ClaudeCode/keybindings/reservedShortcuts.ts` | Reserved/unrebindable shortcuts |
| `/home/claudeuser/ClaudeCode/commands/keybindings/keybindings.ts` | `/keybindings` command implementation |

---

## Keybinding Configuration

### Contexts (19 defined)

Keybindings are organized by context for priority resolution:

```typescript
// From schema.ts - KEYBINDING_CONTEXTS
[
  'Global',        // Active everywhere
  'Chat',          // Chat input focused
  'Autocomplete',  // Autocomplete menu visible
  'Confirmation',  // Permission/dialog confirmation
  'Help',          // Help overlay open
  'Transcript',    // Transcript view active
  'HistorySearch', // History search (ctrl+r)
  'Task',          // Task/agent running
  'ThemePicker',   // Theme picker open
  'Settings',      // Settings menu open
  'Tabs',          // Tab navigation active
  'Attachments',   // Image attachment navigation
  'Footer',        // Footer indicators focused
  'MessageSelector', // Rewind dialog open
  'DiffDialog',    // Diff dialog open
  'ModelPicker',   // Model picker open
  'Select',        // Select/list component focused
  'Plugin',        // Plugin dialog open
]
```

### Actions (70+ defined)

```typescript
// Examples from schema.ts - KEYBINDING_ACTIONS
// App-level: 'app:interrupt', 'app:exit', 'app:toggleTodos', 'app:toggleTranscript'
// History: 'history:search', 'history:previous', 'history:next'
// Chat: 'chat:submit', 'chat:cancel', 'chat:cycleMode', 'chat:modelPicker'
// Autocomplete: 'autocomplete:accept', 'autocomplete:dismiss'
// Confirmation: 'confirm:yes', 'confirm:no'
// Navigation: 'select:next', 'select:previous', 'select:accept'
```

### Default Bindings Structure

From `defaultBindings.ts`:

```typescript
{
  context: 'Global',
  bindings: {
    'ctrl+c': 'app:interrupt',
    'ctrl+d': 'app:exit',
    'ctrl+l': 'app:redraw',
    'ctrl+t': 'app:toggleTodos',
    'ctrl+o': 'app:toggleTranscript',
    'ctrl+r': 'history:search',
  }
}
```

---

## Keybinding Patterns

### 1. Context Hierarchy Pattern

```typescript
// From useKeybinding.ts
const contextsToCheck: KeybindingContextName[] = [
  ...keybindingContext.activeContexts,  // Registered contexts first
  context,                               // Component's context
  'Global',                              // Global fallback
]
// Deduplicate while preserving order (first wins)
const uniqueContexts = [...new Set(contextsToCheck)]
```

More specific contexts take precedence over Global. Context registration via `useRegisterKeybindingContext()`.

### 2. Chord Sequence Pattern

Chords are multi-keystroke sequences like "ctrl+k ctrl+s":

```typescript
// From resolver.ts
export function resolveKeyWithChordState(
  input: string,
  key: Key,
  activeContexts: KeybindingContextName[],
  bindings: ParsedBinding[],
  pending: ParsedKeystroke[] | null,  // Current chord state
): ChordResolveResult
```

Chord states:
- `'chord_started'` - First key of chord pressed, waiting for completion
- `'match'` - Chord completed, action triggered
- `'chord_cancelled'` - Invalid key or timeout, chord aborted
- `'unbound'` - Explicitly unbound key
- `'none'` - No match found

### 3. Handler Registration Pattern

```typescript
// From useKeybinding.ts
useEffect(() => {
  if (!keybindingContext || !isActive) return
  return keybindingContext.registerHandler({ action, context, handler })
}, [action, context, handler, keybindingContext, isActive])
```

Handlers are registered in a Map by action, enabling multiple handlers for the same action.

### 4. Platform-Specific Bindings

```typescript
// From defaultBindings.ts
const IMAGE_PASTE_KEY = getPlatform() === 'windows' ? 'alt+v' : 'ctrl+v'
const MODE_CYCLE_KEY = SUPPORTS_TERMINAL_VT_MODE ? 'shift+tab' : 'meta+m'
```

### 5. Command Binding Pattern

Users can bind keys to execute slash commands:

```typescript
// From useCommandKeybindings.tsx
if (binding.action?.startsWith('command:')) {
  const commandName = action.slice(8) // Remove 'command:' prefix
  onSubmit(`/${commandName}`, NOOP_HELPERS, undefined, { fromKeybinding: true })
}
```

Format in config: `"ctrl+shift+c": "command:compact"`

---

## Input Handling Flow

```
1. Ink Input Event
        |
        v
2. ChordInterceptor (FIRST useInput)
   - Checks if part of chord sequence
   - Stops propagation if chord started/completed/cancelled
        |
        v
3. Component useKeybinding hooks
   - Call resolve() with their context
   - Handle matched actions
   - Stop propagation on match
        |
        v
4. Default terminal handling
```

### ChordInterceptor Critical Role

From `KeybindingProviderSetup.tsx`:

```typescript
// Registers useInput FIRST (before children in render order)
// Intercepts chord keystrokes before PromptInput can capture them
function ChordInterceptor({ bindings, pendingChordRef, setPendingChord, ... }) {
  useInput((input, key, event) => {
    const result = resolveKeyWithChordState(input, key, contexts, bindings, pendingChordRef.current)
    switch (result.type) {
      case 'chord_started':
        setPendingChord(result.pending)
        event.stopImmediatePropagation()  // Prevent other handlers
        break
      case 'match':
        if (wasInChord) {
          // Invoke handler directly for chord completion
          event.stopImmediatePropagation()
        }
        break
      // ...
    }
  })
  return null
}
```

---

## User Configuration

### File Location

```typescript
// From loadUserBindings.ts
export function getKeybindingsPath(): string {
  return join(getClaudeConfigHomeDir(), 'keybindings.json')
}
// ~/.claude/keybindings.json
```

### Configuration Format

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+enter": "chat:submit",
        "ctrl+x": null,
        "ctrl+shift+c": "command:compact"
      }
    }
  ]
}
```

### Validation

From `validate.ts`:

```typescript
export type KeybindingWarningType =
  | 'parse_error'
  | 'duplicate'
  | 'reserved'
  | 'invalid_context'
  | 'invalid_action'
```

Validation includes:
- Duplicate key detection (within same context)
- Reserved shortcut warnings (ctrl+z, ctrl+c on macOS, etc.)
- Context name validation
- Action format validation
- Keystroke parse validation

### Reserved Shortcuts (Non-Rebindable)

From `reservedShortcuts.ts`:

```typescript
export const NON_REBINDABLE: ReservedShortcut[] = [
  { key: 'ctrl+c', reason: 'Interrupt/exit (hardcoded)', severity: 'error' },
  { key: 'ctrl+d', reason: 'Exit (hardcoded)', severity: 'error' },
  { key: 'ctrl+m', reason: 'Identical to Enter (CR)', severity: 'error' },
]
```

---

## Hook Usage Examples

### Single Keybinding

```typescript
import { useKeybinding } from '../keybindings/useKeybinding.js'

function MyComponent() {
  useKeybinding('app:toggleTodos', () => {
    setShowTodos(prev => !prev)
  }, { context: 'Global' })
}
```

### Multiple Keybindings

```typescript
import { useKeybindings } from '../keybindings/useKeybinding.js'

function MyComponent() {
  useKeybindings({
    'chat:submit': () => handleSubmit(),
    'chat:cancel': () => handleCancel(),
  }, { context: 'Chat' })
}
```

### Register Context

```typescript
import { useRegisterKeybindingContext } from '../keybindings/KeybindingContext.js'

function ThemePicker() {
  useRegisterKeybindingContext('ThemePicker')
  // Now ThemePicker's ctrl+t takes precedence over Global
}
```

### Global Handlers Component

```typescript
// From useGlobalKeybindings.tsx
export function GlobalKeybindingHandlers({ screen, setScreen, ... }): null {
  useKeybinding('app:toggleTodos', handleToggleTodos, { context: 'Global' })
  useKeybinding('app:toggleTranscript', handleToggleTranscript, { context: 'Global' })
  // ... more bindings
  return null
}
```

---

## Keystroke Parsing

From `parser.ts`:

```typescript
export function parseKeystroke(input: string): ParsedKeystroke {
  const parts = input.split('+')
  const keystroke: ParsedKeystroke = {
    key: '',
    ctrl: false,
    alt: false,
    shift: false,
    meta: false,
    super: false,
  }
  for (const part of parts) {
    const lower = part.toLowerCase()
    switch (lower) {
      case 'ctrl':
      case 'control':
        keystroke.ctrl = true
        break
      case 'alt':
      case 'opt':
      case 'option':
        keystroke.alt = true
        break
      case 'cmd':
      case 'command':
      case 'super':
      case 'win':
        keystroke.super = true
        break
      // ... special keys (esc -> escape, return -> enter, etc.)
      default:
        keystroke.key = lower
    }
  }
  return keystroke
}
```

### Special Key Mappings

| Input | Maps To |
|-------|---------|
| esc | escape |
| return | enter |
| space | ' ' |
| ↑ | up |
| ↓ | down |
| ← | left |
| → | right |

---

## Key Matching Logic

From `match.ts`:

```typescript
export function matchesKeystroke(
  input: string,
  key: Key,           // From Ink
  target: ParsedKeystroke
): boolean {
  const keyName = getKeyName(input, key)  // Normalize Ink key names
  if (keyName !== target.key) return false

  const inkMods = getInkModifiers(key)

  // Special handling: Ink sets key.meta=true for escape
  if (key.escape) {
    return modifiersMatch({ ...inkMods, meta: false }, target)
  }

  return modifiersMatch(inkMods, target)
}
```

### Modifier Equivalence

Alt and Meta are treated as equivalent (legacy terminal behavior):

```typescript
function modifiersMatch(inkMods: InkModifiers, target: ParsedKeystroke): boolean {
  // ... ctrl and shift checks ...
  const targetNeedsMeta = target.alt || target.meta
  if (inkMods.meta !== targetNeedsMeta) return false
  // Super (cmd/win) is distinct - only via kitty protocol
  if (inkMods.super !== target.super) return false
  return true
}
```

---

## Hot-Reload Implementation

From `loadUserBindings.ts`:

```typescript
export async function initializeKeybindingWatcher(): Promise<void> {
  const userPath = getKeybindingsPath()

  watcher = chokidar.watch(userPath, {
    persistent: true,
    ignoreInitial: true,
    awaitWriteFinish: {
      stabilityThreshold: 500,  // Wait for file write to stabilize
      pollInterval: 200,
    },
  })

  watcher.on('add', handleChange)
  watcher.on('change', handleChange)
  watcher.on('unlink', handleDelete)
}

// Signal-based subscription for updates
export const subscribeToKeybindingChanges = keybindingsChanged.subscribe
```

---

## Display Formatting

From `shortcutFormat.ts` and `parser.ts`:

```typescript
// Platform-appropriate display
export function keystrokeToDisplayString(
  ks: ParsedKeystroke,
  platform: DisplayPlatform = 'linux',
): string {
  const parts: string[] = []
  if (ks.ctrl) parts.push('ctrl')
  if (ks.alt || ks.meta) {
    parts.push(platform === 'macos' ? 'opt' : 'alt')
  }
  if (ks.shift) parts.push('shift')
  if (ks.super) {
    parts.push(platform === 'macos' ? 'cmd' : 'super')
  }
  parts.push(keyToDisplayName(ks.key))
  return parts.join('+')
}
```

---

## Summary

The keybindings system provides:

1. **Context-aware resolution** - More specific contexts override global bindings
2. **Chord sequence support** - Multi-keystroke shortcuts with timeout handling
3. **User customization** - JSON config with validation and hot-reload
4. **Platform adaptation** - Different shortcuts for Windows/macOS/Linux
5. **Command integration** - Bind keys to execute slash commands
6. **Reserved key protection** - Prevent rebinding critical system shortcuts
7. **Type-safe schema** - Zod validation for configuration
