# Claude Code Design System Analysis

## Overview

The Claude Code design system is a terminal-based UI library built on top of Ink (a React renderer for terminals). It follows a **terminal-native design philosophy** with sophisticated theming support, accessibility considerations, and optimized rendering patterns using React Compiler memoization.

---

## Core Design Principles

### 1. Terminal-Native Design
- Built for CLI/terminal environments using Ink (React for terminals)
- Uses box-drawing characters, ANSI colors, and terminal-specific layouts
- Responsive to terminal dimensions via `useTerminalSize()` hook

### 2. Accessibility First
- **Daltonized themes** for color-blind users (light-daltonized, dark-daltonized)
- **ANSI-only themes** for terminals without true color support
- Semantic color naming (success, error, warning) rather than purely aesthetic names

### 3. Performance Optimization
- All components use React Compiler memoization (`$` cache pattern)
- Careful dependency tracking to minimize re-renders
- Layout calculations respect terminal constraints

---

## Theme System Architecture

### Theme Structure (`/utils/theme.ts`)

The theme system defines **89 distinct color tokens** organized into semantic categories:

```typescript
type Theme = {
  // Brand colors
  claude: string                    // Primary brand orange
  claudeShimmer: string             // Lighter variant for shimmer effects
  permission: string                // Permission blue
  suggestion: string                // Interactive suggestion blue

  // Semantic colors
  success: string
  error: string
  warning: string
  merged: string

  // Diff colors (for code diffing)
  diffAdded: string
  diffRemoved: string
  diffAddedWord: string
  diffRemovedWord: string

  // Backgrounds
  userMessageBackground: string
  messageActionsBackground: string
  selectionBg: string

  // Agent colors (for subagent output)
  red_FOR_SUBAGENTS_ONLY: string
  blue_FOR_SUBAGENTS_ONLY: string
  // ... etc

  // Rainbow (ultrathink keyword highlighting)
  rainbow_red: string
  rainbow_red_shimmer: string
  // ... etc
}
```

### Theme Variants

Six theme variants are defined:
1. **dark** - Full RGB dark theme (default)
2. **light** - Full RGB light theme
3. **light-daltonized** - Color-blind friendly light theme
4. **dark-daltonized** - Color-blind friendly dark theme
5. **light-ansi** - 16-color ANSI light theme
6. **dark-ansi** - 16-color ANSI dark theme

### Theme Provider Pattern

The `ThemeProvider` (`ThemeProvider.tsx`) manages theme state:
- Supports `'auto'` mode (follows system preference via `$COLORFGBG`)
- Live theme preview capability
- OSC 11 system theme detection
- Context-based theme distribution via `useTheme()` hook

### Color Resolution

Colors can be specified as:
- **Theme keys**: `'success'`, `'error'`, `'suggestion'` (resolved at render)
- **Raw RGB**: `'rgb(215,119,87)'`
- **Hex**: `'#ff0000'`
- **ANSI**: `'ansi:blue'`, `'ansi256(123)'`

The `resolveColor()` function handles theme key resolution:
```typescript
function resolveColor(color: keyof Theme | Color, theme: Theme): Color {
  if (isRawColor(color)) return color
  return theme[color]
}
```

---

## Component Architecture

### Layout Components

#### Pane (`Pane.tsx`)
**Purpose**: Primary container for slash-command screens
- Renders below REPL prompt
- Colored top border with horizontal padding
- Modal-aware (adjusts rendering inside FullscreenLayout)
- Used by: `/config`, `/help`, `/plugins`, `/sandbox`, `/stats`, `/permissions`

```typescript
type PaneProps = {
  children: React.ReactNode
  color?: keyof Theme  // Theme color for top border
}
```

#### Dialog (`Dialog.tsx`)
**Purpose**: Confirmation/cancel dialogs with keyboard bindings
- Built-in keybindings (Esc to cancel, Enter to confirm)
- Input guide footer with configurable hints
- Cancel state management for Ctrl+C/D
- Composes with Pane for border styling

#### Tabs (`Tabs.tsx`)
**Purpose**: Tab navigation for multi-pane interfaces
- Header row with tab labels
- Content area with scroll management
- Keyboard navigation (↑/↓, Tab/Shift+Tab)
- Focus management between header and content
- Modal scroll integration via `useTabHeaderFocus()` hook

### Presentation Components

#### Divider (`Divider.tsx`)
**Purpose**: Horizontal separator lines
- Full-width or fixed width
- Centered title support
- Custom character support (default: '─')
- Padding subtraction for indented content

```typescript
type DividerProps = {
  width?: number
  color?: keyof Theme
  char?: string
  padding?: number
  title?: string
}
```

#### ProgressBar (`ProgressBar.tsx`)
**Purpose**: Visual progress indication
- Uses Unicode block characters for sub-character precision
- 9-level block gradients: `[' ', '▏', '▎', '▍', '▌', '▋', '▊', '▉', '█']`
- Theme-aware fill/empty colors
- Configurable width in characters

#### StatusIcon (`StatusIcon.tsx`)
**Purpose**: Semantic status indicators
- Predefined states: success, error, warning, info, pending, loading
- Theme-appropriate colors for each state
- Optional trailing space for text pairing
- Uses `figures` library for consistent icons

```typescript
const STATUS_CONFIG: Record<Status, {
  icon: string
  color: 'success' | 'error' | 'warning' | 'suggestion' | undefined
}> = {
  success: { icon: figures.tick, color: 'success' },
  error: { icon: figures.cross, color: 'error' },
  // ...
}
```

#### ListItem (`ListItem.tsx`)
**Purpose**: Selection list item (dropdowns, multi-selects)
- Pointer indicator (❯) for focused items
- Checkmark indicator (✓) for selected items
- Scroll indicators (↑↓) for truncated lists
- Description text support
- Disabled state handling
- Cursor position declaration for screen readers

### Input Components

#### FuzzyPicker (`FuzzyPicker.tsx`)
**Purpose**: Searchable selection interface
- Search box with live filtering
- Preview pane support (right or bottom)
- Direction control ('up' for atuin-style)
- Tab/Shift+Tab custom actions
- Match count display
- Compact mode for narrow terminals

```typescript
type Props<T> = {
  title: string
  items: readonly T[]
  getKey: (item: T) => string
  renderItem: (item: T, isFocused: boolean) => React.ReactNode
  renderPreview?: (item: T) => React.ReactNode
  direction?: 'down' | 'up'
  onQueryChange: (query: string) => void
  onSelect: (item: T) => void
  onTab?: PickerAction<T>
  onShiftTab?: PickerAction<T>
}
```

#### CustomSelect (`/components/CustomSelect/`)
**Purpose**: Single and multi-select inputs
- Single select: `select.tsx`
- Multi select: `SelectMulti.tsx`
- Input options with inline editing
- Image paste support
- Keyboard navigation (arrows, space to toggle, enter to submit)
- Scroll arrows for long lists

### Utility Components

#### ThemedBox / ThemedText
**Purpose**: Theme-aware wrappers around Ink primitives
- Accept theme keys as color props
- Resolve to actual colors at render time
- Cross Box boundary style cascade via context

```typescript
// ThemedBox.tsx
function resolveColor(color: keyof Theme | Color, theme: Theme): Color {
  if (color.startsWith('rgb(') || color.startsWith('#')) {
    return color as Color  // Raw color
  }
  return theme[color as keyof Theme] as Color  // Theme key
}
```

#### Ratchet (`Ratchet.tsx`)
**Purpose**: Height-locked scrolling container
- Prevents layout shifts during content changes
- Viewport tracking
- Configurable lock mode ('always' | 'offscreen')
- Measures children and sets minHeight

#### Byline (`Byline.tsx`)
**Purpose**: Inline metadata separator
- Joins children with middot separator (" · ")
- Filters null/undefined/false children
- Common pattern: keyboard shortcut hints

#### KeyboardShortcutHint (`KeyboardShortcutHint.tsx`)
**Purpose**: Standardized shortcut display
- Pattern: "Enter to confirm"
- Bold shortcut option
- Parentheses wrapping option
- Designed for use within Byline

---

## Color System

### Semantic Color Mapping

| Token | Purpose | Light Theme | Dark Theme |
|-------|---------|-------------|------------|
| `claude` | Brand primary | `rgb(215,119,87)` | `rgb(215,119,87)` |
| `permission` | Permissions UI | `rgb(87,105,247)` | `rgb(177,185,249)` |
| `suggestion` | Interactive hints | `rgb(87,105,247)` | `rgb(177,185,249)` |
| `success` | Success states | `rgb(44,122,57)` | `rgb(78,186,101)` |
| `error` | Error states | `rgb(171,43,63)` | `rgb(255,107,128)` |
| `warning` | Warning states | `rgb(150,108,30)` | `rgb(255,193,7)` |
| `inactive` | Disabled text | `rgb(102,102,102)` | `rgb(153,153,153)` |
| `text` | Primary text | `rgb(0,0,0)` | `rgb(255,255,255)` |

### Shimmer Pattern
Most colors have a `*Shimmer` variant used for:
- Loading states
- Animated spinners
- Highlight effects

Example: `claude` → `claudeShimmer` (lighter variant)

### Daltonization Strategy
- Green → Blue (for deuteranopia)
- Red → Maintained but intensified
- Orange → Adjusted for better distinction

---

## Layout Patterns

### Modal Context
Components check `useIsInsideModal()` to adjust behavior:
- Pane skips its own divider when inside modal
- Tabs adjust flexShrink for modal containers
- Scroll behavior adapts to modal constraints

### Responsive Patterns

**Terminal width constraints:**
- `< 120 cols`: Compact mode (shortened labels)
- Dynamic visible item counts based on terminal height
- Content height capping to prevent overflow

**Spacing system:**
- Uses `flexDirection`, `gap`, `paddingX/Y`, `marginTop/Bottom` props
- Consistent `gap={1}` for internal spacing
- `paddingX={2}` standard for Pane content

### Component Composition

**Standard slash-command layout:**
```tsx
<Pane color="permission">
  <Tabs title="Settings:">
    <Tab title="General">
      <Select options={...} />
    </Tab>
    <Tab title="Advanced">
      <Dialog title="Confirm" onCancel={...}>
        {/* content */}
      </Dialog>
    </Tab>
  </Tabs>
</Pane>
```

---

## Typography

### Text Styling via Ink
- `bold`: Emphasis
- `dimColor`: Secondary/hint text (uses theme's inactive color)
- `color`: Theme key or raw color
- `backgroundColor`: Theme key only
- `inverse`: Swap fg/bg colors

### Font Treatment
- Monospace assumed (terminal environment)
- String width calculations via `stringWidth()` for Unicode
- No font family control (terminal determines)

---

## Key Design Patterns

### 1. Memoization Pattern
All components use React Compiler memoization:
```typescript
const $ = _c(N)  // N = cache slot count
// Cache reads/writes via $[index]
```

### 2. Theme Key Resolution
Colors accepted as theme keys, resolved at render:
```typescript
// Component props accept theme keys
type Props = { color?: keyof Theme }

// Resolved during render
const resolvedColor = theme[color] || color
```

### 3. Conditional Rendering with Stability
Components preserve structure to avoid layout shifts:
```typescript
// Structure stable - no fragment switching
<Box>
  {showPreview ? <Preview /> : <Box flexGrow={1} />}
</Box>
```

### 4. Keyboard Navigation
- Arrow keys for navigation
- Enter/Space for selection
- Esc for cancellation
- Tab for secondary actions

### 5. Focus Management
- Declared cursor positions for screen readers
- Focus state passed through props
- `tabIndex` and `autoFocus` for keyboard control

---

## File Organization

```
/components/design-system/
├── ThemeProvider.tsx       # Theme context and resolution
├── ThemedBox.tsx           # Theme-aware Box wrapper
├── ThemedText.tsx          # Theme-aware Text wrapper
├── color.ts                # Curried color function
├── Pane.tsx                # Slash-command container
├── Dialog.tsx              # Confirmation dialogs
├── Tabs.tsx                # Tab navigation
├── ListItem.tsx            # Selection list items
├── Divider.tsx             # Horizontal dividers
├── ProgressBar.tsx         # Progress indication
├── StatusIcon.tsx          # Status indicators
├── LoadingState.tsx        # Loading spinner + text
├── FuzzyPicker.tsx         # Searchable picker
├── KeyboardShortcutHint.tsx# Shortcut hints
├── Byline.tsx              # Metadata separator
├── Ratchet.tsx             # Height-locking container

/components/CustomSelect/
├── select.tsx              # Single select component
├── SelectMulti.tsx         # Multi-select component
├── select-option.tsx       # Option item component
├── select-input-option.tsx # Editable input option
├── use-select-state.ts     # Single select state hook
├── use-multi-select-state.ts # Multi select state hook
├── use-select-navigation.ts # Navigation hook
├── use-select-input.ts     # Input handling hook
└── option-map.ts           # Option data structures

/utils/theme.ts             # Theme definitions and types
```

---

## Integration Points

### With Ink
- Extends Ink's Box/Text with theme resolution
- Uses Ink's layout system (flexDirection, gap, padding, etc.)
- Leverages Ink's event system for keyboard input

### With Keybindings
- Components integrate with `useKeybinding()` for shortcuts
- Contextual keybinding activation (isActive flag)
- Global vs scoped keybinding support

### With Modal System
- `useIsInsideModal()` for contextual rendering
- `useModalScrollRef()` for scroll coordination
- Modal slot rendering for overlays

---

## Summary

The Claude Code design system demonstrates sophisticated terminal UI engineering:

1. **Comprehensive theming** with accessibility variants
2. **Performance-optimized** rendering with React Compiler
3. **Terminal-native** patterns respecting terminal constraints
4. **Modular composition** enabling consistent UI patterns
5. **Semantic color system** supporting multiple themes
6. **Keyboard-first** interaction model suitable for CLI workflows

The architecture prioritizes consistency, accessibility, and performance while maintaining flexibility for complex terminal-based interactions.
