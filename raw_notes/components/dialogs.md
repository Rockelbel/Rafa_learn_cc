# Dialog Components Analysis

## Overview

Claude Code's dialog system is built on a layered architecture using React with Ink (terminal UI library). The dialog components demonstrate sophisticated patterns for terminal-based interactive UI components.

## Core Dialog Architecture

### 1. Base Dialog Component (`design-system/Dialog.tsx`)

The foundation of all dialogs with the following key features:

**Props Interface:**
```typescript
type DialogProps = {
  title: React.ReactNode;
  subtitle?: React.ReactNode;
  children: React.ReactNode;
  onCancel: () => void;
  color?: keyof Theme;
  hideInputGuide?: boolean;
  hideBorder?: boolean;
  inputGuide?: (exitState: ExitState) => React.ReactNode;
  isCancelActive?: boolean;  // Controls if Esc/n keybindings are active
};
```

**Key Patterns:**
- **Exit State Integration**: Uses `useExitOnCtrlCDWithKeybindings` for consistent Ctrl+C/D handling
- **Configurable Input Guide**: Supports custom input guides or defaults to "Enter to confirm, Esc to cancel"
- **Cancel Activation Control**: `isCancelActive` prop allows child components (like TextInput) to disable dialog-level cancel handling
- **Bordered Container**: Wraps content in a `Pane` component with configurable color

**Exit Flow Handling:**
- Integrates double-press exit mechanism (Ctrl+C or Ctrl+D twice to exit)
- Shows "Press X again to exit" message on first press
- Uses `useKeybinding` for "confirm:no" action (Esc by default)

### 2. FuzzyPicker Component (`design-system/FuzzyPicker.tsx`)

A reusable search/select component used by multiple dialogs.

**Props Interface:**
```typescript
type Props<T> = {
  title: string;
  placeholder?: string;
  initialQuery?: string;
  items: readonly T[];
  getKey: (item: T) => string;
  renderItem: (item: T, isFocused: boolean) => React.ReactNode;
  renderPreview?: (item: T) => React.ReactNode;
  previewPosition?: 'bottom' | 'right';
  visibleCount?: number;
  direction?: 'down' | 'up';
  onQueryChange: (query: string) => void;
  onSelect: (item: T) => void;
  onTab?: PickerAction<T>;
  onShiftTab?: PickerAction<T>;
  onFocus?: (item: T | undefined) => void;
  onCancel: () => void;
  emptyMessage?: string | ((query: string) => string);
  matchLabel?: string;
  selectAction?: string;
};
```

**Key Patterns:**
- **Generic Type Support**: Fully typed with `<T>` for item flexibility
- **Windowed Rendering**: Only renders visible items for performance
- **Direction Support**: 'up' puts items at bottom (atuin-style), 'down' is standard
- **Keyboard Navigation**: Arrow keys, Ctrl+P/N, Enter, Tab handling
- **Async Preview Loading**: `onFocus` callback for loading previews outside render
- **Responsive Layout**: Switches preview position based on terminal width

**Input Handling:**
- Uses `useSearchInput` hook for query management
- Prevents backspace-on-empty from exiting (critical for search UX)
- Handles Tab/Shift+Tab for secondary actions

## Dialog Implementations

### 1. ExportDialog (`ExportDialog.tsx`)

**Purpose**: Export conversation to clipboard or file.

**State Management:**
```typescript
const [selectedOption, setSelectedOption] = useState<ExportOption | null>(null);
const [filename, setFilename] = useState<string>(defaultFilename);
const [cursorOffset, setCursorOffset] = useState<number>(defaultFilename.length);
const [showFilenameInput, setShowFilenameInput] = useState(false);
```

**Multi-Step Flow Pattern:**
1. Initial state shows option selection (clipboard vs file)
2. Selecting 'file' transitions to filename input
3. Cancel goes back to option selection when in filename mode
4. Complete cancel closes dialog

**Key Pattern - Contextual Cancel Handling:**
```typescript
const handleCancel = useCallback(() => {
  if (showFilenameInput) {
    handleGoBack();  // Return to option selection
  } else {
    onDone({ success: false, message: 'Export cancelled' });
  }
}, [showFilenameInput, handleGoBack, onDone]);
```

**Dynamic Input Guide:**
```typescript
function renderInputGuide(exitState: ExitState): React.ReactNode {
  if (showFilenameInput) {
    return <Byline>
      <KeyboardShortcutHint shortcut="Enter" action="save" />
      <ConfigurableShortcutHint action="confirm:no" context="Settings" fallback="Esc" description="go back" />
    </Byline>;
  }
  // ... default guide
}
```

**TextInput Integration:**
- Uses `isCancelActive={!showFilenameInput}` on Dialog
- Registers separate keybinding with 'Settings' context when filename input is shown
- Allows typing 'n' in filename without triggering cancel

### 2. GlobalSearchDialog (`GlobalSearchDialog.tsx`)

**Purpose**: Workspace-wide ripgrep search (Ctrl+Shift+F).

**Key Features:**
- **Debounced Search**: 100ms debounce on query changes
- **Streaming Results**: Uses `ripGrepStream` for real-time results
- **AbortController Pattern**: Cancels in-flight searches on new queries
- **Client-Side Filtering**: Filters existing results while waiting for rg
- **Preview Loading**: Loads file context around matches on focus

**State Management:**
```typescript
const [matches, setMatches] = useState<Match[]>([]);
const [truncated, setTruncated] = useState(false);
const [isSearching, setIsSearching] = useState(false);
const [query, setQuery] = useState("");
const [focused, setFocused] = useState<Match | undefined>(undefined);
const [preview, setPreview] = useState<PreviewState | null>(null);
const abortRef = useRef<AbortController | null>(null);
const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);
```

**Search Flow:**
1. Query change triggers `handleQueryChange`
2. Clear previous timeout and abort controller
3. If query empty: clear results
4. Set searching state, truncate false
5. Client-filter existing matches
6. Set timeout for debounced ripgrep call
7. Stream results with deduplication
8. Cap at MAX_TOTAL_MATCHES (500)

**Result Deduplication:**
```typescript
setMatches(prev => {
  const seen = new Set(prev.map(matchKey));
  const fresh = parsed.filter(p => !seen.has(matchKey(p)));
  if (!fresh.length) return prev;
  const next = prev.concat(fresh);
  return next.length > MAX_TOTAL_MATCHES ? next.slice(0, MAX_TOTAL_MATCHES) : next;
});
```

### 3. HistorySearchDialog (`HistorySearchDialog.tsx`)

**Purpose**: Search command history (similar to atuin).

**Data Loading Pattern:**
```typescript
useEffect(() => {
  let cancelled = false;
  void (async () => {
    const reader = getTimestampedHistory();
    const loaded: Item[] = [];
    for await (const entry of reader) {
      if (cancelled) {
        void reader.return(undefined);
        return;
      }
      // Process entry...
    }
    if (!cancelled) setItems(loaded);
  })();
  return () => { cancelled = true; };
}, []);
```

**Filtering Strategy:**
- Exact matches first (substring includes)
- Fuzzy matches second (subsequence match)
- No scoring, simple concatenation

**Item Preprocessing:**
- Pre-computes display, lower, firstLine, age for each item
- Uses stringWidth for proper terminal width calculation

### 4. QuickOpenDialog (`QuickOpenDialog.tsx`)

**Purpose**: Fuzzy file finder (Ctrl+Shift+P).

**Query Generation Pattern:**
```typescript
const queryGenRef = useRef(0);

const handleQueryChange = (q: string) => {
  setQuery(q);
  const gen = queryGenRef.current = queryGenRef.current + 1;
  if (!q.trim()) {
    setResults([]);
    return;
  }
  generateFileSuggestions(q, true).then(items => {
    if (gen !== queryGenRef.current) return;  // Stale result check
    // Process results...
  });
};
```

**Preview Loading:**
- Loads first N lines of focused file
- Uses AbortController to cancel stale reads
- Shows "loading..." when preview path doesn't match focused

**Responsive Design:**
```typescript
const previewOnRight = columns >= 120;
const effectivePreviewLines = previewOnRight ? VISIBLE_RESULTS - 1 : PREVIEW_LINES;
```

### 5. Onboarding (`Onboarding.tsx`)

**Purpose**: Multi-step user setup flow.

**Step Definition Pattern:**
```typescript
type StepId = 'preflight' | 'theme' | 'oauth' | 'api-key' | 'security' | 'terminal-setup';

interface OnboardingStep {
  id: StepId;
  component: React.ReactNode;
}
```

**Dynamic Step Assembly:**
```typescript
const steps: OnboardingStep[] = [];
if (oauthEnabled) steps.push({ id: 'preflight', component: preflightStep });
steps.push({ id: 'theme', component: themeStep });
if (apiKeyNeedingApproval) steps.push({ id: 'api-key', component: ... });
// ... etc
```

**Step Navigation:**
```typescript
function goToNextStep() {
  if (currentStepIndex < steps.length - 1) {
    const nextIndex = currentStepIndex + 1;
    setCurrentStepIndex(nextIndex);
    logEvent('tengu_onboarding_step', { oauthEnabled, stepId: steps[nextIndex]?.id });
  } else {
    onDone();
  }
}
```

**SkippableStep Component:**
```typescript
export function SkippableStep({ skip, onSkip, children }) {
  useEffect(() => {
    if (skip) onSkip();
  }, [skip, onSkip]);
  if (skip) return null;
  return children;
}
```

## Common Patterns

### 1. Overlay Registration
All dialogs register with the overlay context:
```typescript
useRegisterOverlay('dialog-name');
```
This manages modal stacking and focus.

### 2. Terminal Size Awareness
All dialogs use `useTerminalSize` for responsive layouts:
```typescript
const { columns, rows } = useTerminalSize();
```

### 3. Consistent Callback Signatures
- `onDone()`: Dialog completion
- `onCancel()`: User cancellation
- `onSelect(item)`: Item selection

### 4. Result Capping
All search dialogs cap results for performance:
- GlobalSearch: 500 matches
- QuickOpen: Uses generateFileSuggestions internal limits
- HistorySearch: No hard cap but streams from async iterator

### 5. Analytics Integration
All dialogs log events:
```typescript
logEvent('tengu_dialog_action', {
  result_count: results.length,
  query_length: query.length,
  // ... context-specific data
});
```

## Exit Flow Architecture

### Double-Press Exit Pattern
From `useExitOnCtrlCD.ts`:

```typescript
export type ExitState = {
  pending: boolean;
  keyName: 'Ctrl-C' | 'Ctrl-D' | null;
};

// First press: Show "Press X again to exit"
// Second press within timeout: Exit application
```

### Keybinding Context System
- **Global context**: Exit handlers (Ctrl+C/D)
- **Confirmation context**: Yes/no prompts (y/n, Enter/Esc)
- **Settings context**: Text input (allows typing 'n' without canceling)

### Dialog-Level Control
```typescript
// Disable dialog cancel when child input is focused
<Dialog isCancelActive={!showFilenameInput} ... />

// Register alternative keybinding with different context
useKeybinding('confirm:no', handleCancel, {
  context: 'Settings',
  isActive: showFilenameInput
});
```

## Input Handling Patterns

### Search Input Hook (`useSearchInput`)
Used by FuzzyPicker-based dialogs:
- Manages query state and cursor position
- Handles exit/cancel callbacks
- Prevents backspace-on-empty from exiting

### TextInput Component
Used by ExportDialog:
- Full cursor control (offset management)
- Focus and showCursor props
- onChange and onSubmit callbacks

### Select Component (`CustomSelect/select.tsx`)
Used by Onboarding:
- Complex option types (text, input, with descriptions)
- Index-based navigation
- Support for inline descriptions

## Best Practices Observed

1. **Stale Result Cancellation**: All async operations use AbortController or generation counters
2. **Memory Management**: Streaming data with proper cleanup on unmount
3. **Terminal Constraints**: Height/width calculations to prevent overflow
4. **Keyboard Navigation**: Consistent arrow keys, Enter, Tab, Esc handling
5. **Loading States**: Explicit loading indicators during async operations
6. **Error Handling**: Graceful fallbacks (e.g., "(preview unavailable)")
7. **Responsive Design**: Layout changes based on terminal width
8. **Event Logging**: Comprehensive analytics for user actions
