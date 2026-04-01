# Claude Code Commands System - Core Infrastructure Analysis

## Architecture Overview

The Commands System is a sophisticated command registry and management infrastructure that supports multiple command types, lazy loading, dynamic discovery, and fine-grained availability controls. It serves as the backbone for slash commands, skills, plugins, and workflows in Claude Code.

### Core Files

1. **`/home/claudeuser/ClaudeCode/commands.ts`** - Command registry and management
2. **`/home/claudeuser/ClaudeCode/types/command.ts`** - Command type definitions
3. **`/home/claudeuser/ClaudeCode/utils/commandLifecycle.ts`** - Command lifecycle events

### Supporting Files

4. **`/home/claudeuser/ClaudeCode/skills/loadSkillsDir.ts`** - Skill loading from directories
5. **`/home/claudeuser/ClaudeCode/skills/bundledSkills.ts`** - Bundled skill registration
6. **`/home/claudeuser/ClaudeCode/utils/plugins/loadPluginCommands.ts`** - Plugin command loading
7. **`/home/claudeuser/ClaudeCode/plugins/builtinPlugins.ts`** - Built-in plugin registry

---

## Three Command Types

The system defines three distinct command types via a discriminated union in `Command` type:

### 1. PromptCommand (`type: 'prompt'`)

**Purpose**: Model-invokable commands that expand into conversation content

**Key Fields**:
```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number  // For token estimation
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: { pluginManifest: PluginManifest; repository: string }
  disableNonInteractive?: boolean
  hooks?: HooksSettings
  skillRoot?: string  // Base directory for skill resources
  context?: 'inline' | 'fork'  // Execution context
  agent?: string  // Agent type for forked execution
  effort?: EffortValue
  paths?: string[]  // Glob patterns for conditional activation
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

**Characteristics**:
- Expands into text/content blocks sent to the model
- Can invoke tools via `allowedTools`
- Supports argument substitution
- Can run in "inline" (default) or "fork" (sub-agent) context
- Supports conditional activation via `paths` glob patterns
- Primary mechanism for skills

**Sources**:
- Builtin commands (hardcoded)
- Skills from `.claude/skills/` directories
- Legacy `.claude/commands/` directories
- Plugins (marketplace and built-in)
- Bundled skills (compiled into CLI)
- MCP servers

### 2. LocalCommand (`type: 'local'`)

**Purpose**: Commands that execute locally and return text results

**Key Fields**:
```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

type LocalCommandModule = {
  call: (args: string, context: LocalJSXCommandContext) => Promise<LocalCommandResult>
}

type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

**Characteristics**:
- Returns text results programmatically
- Used for non-interactive operations
- Results can be "compact" (context window management)
- Supports lazy loading via `load()` function

### 3. LocalJSXCommand (`type: 'local-jsx'`)

**Purpose**: Commands that render interactive React/Ink UI components

**Key Fields**:
```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

type LocalJSXCommandModule = {
  call: (
    onDone: LocalJSXCommandOnDone,
    context: ToolUseContext & LocalJSXCommandContext,
    args: string
  ) => Promise<React.ReactNode>
}

type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay  // 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  }
) => void
```

**Characteristics**:
- Renders interactive React/Ink components
- Uses callback-based completion (`onDone`)
- Rich UI interactions (pickers, forms, etc.)
- Context includes `setMessages`, IDE status, theme, etc.
- Lazy loaded to defer heavy dependencies

**Command Result Display**:
- `'skip'` - No visible output
- `'system'` - Show as system message
- `'user'` - Show as user message (default)

---

## Command Type Comparison

| Aspect | PromptCommand | LocalCommand | LocalJSXCommand |
|--------|--------------|--------------|-----------------|
| **Execution** | Expands to model prompt | Executes code locally | Renders React UI |
| **Output** | Content blocks | Text/compact result | Interactive components |
| **Interaction** | Model-driven | Programmatic | User-interactive |
| **Lazy Loading** | No (except bundled file extraction) | Yes | Yes |
| **Use Cases** | Skills, custom commands | Clear, cost, status | Config, IDE, login |
| **Non-interactive** | Via `disableNonInteractive` | Via `supportsNonInteractive` | Not applicable |
| **Context** | ToolUseContext | LocalJSXCommandContext | LocalJSXCommandContext |

---

## Command Registration and Discovery

Commands come from multiple sources, loaded in a specific priority order:

### Loading Hierarchy (from `loadAllCommands()`)

```
1. Bundled skills (getBundledSkills)
2. Built-in plugin skills (getBuiltinPluginSkillCommands)
3. Skill directory commands (skillDirCommands from getSkills)
4. Workflow commands (getWorkflowCommands)
5. Plugin commands (getPluginCommands)
6. Plugin skills (getPluginSkills)
7. Built-in commands (COMMANDS array)
```

### Source Categories

#### 1. Built-in Commands (`COMMANDS` array in commands.ts)
- Hardcoded command objects imported at module initialization
- Feature-flagged commands use conditional requires with `feature()`
- Examples: `/help`, `/clear`, `/cost`, `/git` (lazy loaded)

#### 2. Skills

**File-based Skills** (from `loadSkillsDir.ts`):
- Loaded from `.claude/skills/` directories
- Directory format: `skill-name/SKILL.md`
- Supports frontmatter configuration
- Deduplication via file identity (realpath)

**Loading Sources**:
- Policy settings: `~/.claude-code/.claude/skills/`
- User settings: `~/.claude/skills/`
- Project settings: `./.claude/skills/` (and parent directories)
- Additional directories: `--add-dir` paths

**Legacy Commands**:
- Still supported from `.claude/commands/`
- Supports both directory and single-file format
- Marked with `loadedFrom: 'commands_DEPRECATED'`

#### 3. Bundled Skills (from `bundledSkills.ts`)
- Compiled into CLI binary at build time
- Registered via `registerBundledSkill()`
- Can include embedded files extracted on first invocation
- Extraction uses secure file operations (O_NOFOLLOW, O_EXCL)

#### 4. Plugin Commands (from `loadPluginCommands.ts`)
- Loaded from enabled marketplace plugins
- Supports commands/ and skills/ directories
- Namespaced with plugin name: `plugin-name:command-name`
- Metadata overrides from manifest
- Inline content support

#### 5. Built-in Plugins (from `builtinPlugins.ts`)
- User-toggleable plugins shipped with CLI
- Registered via `registerBuiltinPlugin()`
- Stored separately from marketplace plugins
- Enabled/disabled state in user settings

#### 6. Workflow Commands
- Generated from workflow scripts
- Feature-flagged behind `WORKFLOW_SCRIPTS`
- Badged as `(workflow)` in autocomplete

#### 7. Dynamic Skills
- Discovered during file operations
- Activated by path patterns (conditional skills)
- Walks up directory tree from touched files
- Gitignore-respecting discovery

---

## Command Availability Filtering

### `meetsAvailabilityRequirement()` Function

Commands can declare `availability` to restrict who can use them:

```typescript
type CommandAvailability = 'claude-ai' | 'console'
```

**Semantics**:
- No `availability` = universal (available to everyone)
- With `availability` = must match at least one listed type

**Availability Types**:

| Type | Description | Check Logic |
|------|-------------|-------------|
| `'claude-ai'` | claude.ai OAuth subscribers (Pro/Max/Team/Enterprise) | `isClaudeAISubscriber()` |
| `'console'` | Direct Console API key users (api.anthropic.com) | `!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl()` |

**Key Implementation**:
```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

**Important**: Availability filtering runs BEFORE `isEnabled()` checks, so provider-gated commands are hidden regardless of feature-flag state.

### Feature Flag Integration (`isEnabled`)

Commands can define conditional enablement:

```typescript
isEnabled?: () => boolean  // Defaults to true
```

Examples from codebase:
- `feature('PROACTIVE')` - Proactive features
- `feature('KAIROS')` - Kairos mode features
- `feature('WORKFLOW_SCRIPTS')` - Workflow support
- `!isUsing3PServices()` - Login/logout only for 1P users

---

## Lazy Loading Patterns

### 1. Built-in Command Lazy Loading

**Pattern**: Module-level conditional require

```typescript
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default
  : null

// Later in COMMANDS array:
...(proactive ? [proactive] : []),
```

**Benefits**:
- Code elimination when feature flag is off
- Dead code elimination by bundler
- Module only parsed if needed

### 2. Command Implementation Lazy Loading

**Pattern**: `load()` function returning a Promise

Used by `LocalCommand` and `LocalJSXCommand`:

```typescript
const compact: Command = {
  type: 'local',
  // ... other fields ...
  load: async () => {
    const { default: compactCommand } = await import('./commands/compact/index.js')
    return { call: compactCommand }
  },
}
```

**Benefits**:
- Defer heavy dependencies (Ink, React, etc.)
- Reduce startup time
- Only load when command is actually invoked

### 3. Bundled Skill File Extraction

**Pattern**: Promise memoization with closure

```typescript
let extractionPromise: Promise<string | null> | undefined
const inner = definition.getPromptForCommand
getPromptForCommand = async (args, ctx) => {
  extractionPromise ??= extractBundledSkillFiles(definition.name, files)
  const extractedDir = await extractionPromise
  const blocks = await inner(args, ctx)
  // ... prepend base directory
}
```

**Benefits**:
- Files extracted only on first invocation
- Concurrent callers share same promise
- Safe extraction with security controls

### 4. Dynamic Import for Prompt Commands

**Example**: `/insights` command

```typescript
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  // ...
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    if (real.type !== 'prompt') throw new Error('unreachable')
    return real.getPromptForCommand(args, context)
  },
}
```

**Benefits**:
- 113KB module deferred until needed
- Includes heavy dependencies (diff rendering)

---

## `loadAllCommands()` Memoization Strategy

### Memoization Design

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

**Key Decisions**:

1. **Memoized by cwd**: Loading is expensive (disk I/O, dynamic imports)

2. **Cache clearing**: `clearCommandMemoizationCaches()` clears specific caches

3. **Not memoized - availability checks**: `meetsAvailabilityRequirement()` and `isCommandEnabled()` run fresh every time because auth state can change mid-session (e.g., after `/login`)

4. **Separate memoization layers**:
   - `loadAllCommands` - expensive loading
   - `getSkillToolCommands` - skill filtering
   - `getSlashCommandToolSkills` - slash command filtering
   - `getSkillIndex` (in skillSearch/localSearch.ts) - search index

### Cache Management

```typescript
export function clearCommandMemoizationCaches(): void {
  loadAllCommands.cache?.clear?.()
  getSkillToolCommands.cache?.clear?.()
  getSlashCommandToolSkills.cache?.clear?.()
  clearSkillIndexCache?.()
}

export function clearCommandsCache(): void {
  clearCommandMemoizationCaches()
  clearPluginCommandCache()
  clearPluginSkillsCache()
  clearSkillCaches()
}
```

---

## INTERNAL_ONLY_COMMANDS

Commands that are eliminated from external builds:

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions,
  breakCache,
  bughunter,
  commit,
  commitPushPr,
  ctx_viz,
  goodClaude,
  issue,
  initVerifiers,
  ...(forceSnip ? [forceSnip] : []),
  mockLimits,
  bridgeKick,
  version,
  ...(ultraplan ? [ultraplan] : []),
  ...(subscribePr ? [subscribePr] : []),
  resetLimits,
  resetLimitsNonInteractive,
  onboarding,
  share,
  summary,
  teleport,
  antTrace,
  perfIssue,
  env,
  oauthRefresh,
  debugToolCall,
  agentsPlatform,
  autofixPr,
].filter(Boolean)
```

**Inclusion Logic**:
```typescript
...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
  ? INTERNAL_ONLY_COMMANDS
  : []),
```

These commands are only available to internal Anthropic users (`USER_TYPE === 'ant'`) and not in demo mode.

---

## Command Typeahead and Help System Integration

### `getSkillToolCommands()`

Filters to model-invocable prompt commands:

```typescript
export const getSkillToolCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const allCommands = await getCommands(cwd)
  return allCommands.filter(
    cmd =>
      cmd.type === 'prompt' &&
      !cmd.disableModelInvocation &&
      cmd.source !== 'builtin' &&
      (cmd.loadedFrom === 'bundled' ||
        cmd.loadedFrom === 'skills' ||
        cmd.loadedFrom === 'commands_DEPRECATED' ||
        cmd.hasUserSpecifiedDescription ||
        cmd.whenToUse),
  )
})
```

Used by SkillTool to show model available skills.

### `getSlashCommandToolSkills()`

Filters to user-facing slash commands:

```typescript
export const getSlashCommandToolSkills = memoize(async (cwd: string): Promise<Command[]> => {
  const allCommands = await getCommands(cwd)
  return allCommands.filter(
    cmd =>
      cmd.type === 'prompt' &&
      cmd.source !== 'builtin' &&
      (cmd.hasUserSpecifiedDescription || cmd.whenToUse) &&
      (cmd.loadedFrom === 'skills' ||
        cmd.loadedFrom === 'plugin' ||
        cmd.loadedFrom === 'bundled' ||
        cmd.disableModelInvocation),
  )
})
```

### Description Formatting

```typescript
export function formatDescriptionWithSource(cmd: Command): string {
  if (cmd.type !== 'prompt') return cmd.description
  if (cmd.kind === 'workflow') return `${cmd.description} (workflow)`
  if (cmd.source === 'plugin') {
    const pluginName = cmd.pluginInfo?.pluginManifest.name
    return pluginName ? `(${pluginName}) ${cmd.description}` : `${cmd.description} (plugin)`
  }
  // ... other sources
}
```

---

## Command Lifecycle (commandLifecycle.ts)

Simple pub/sub pattern for tracking command execution:

```typescript
type CommandLifecycleState = 'started' | 'completed'

type CommandLifecycleListener = (uuid: string, state: CommandLifecycleState) => void

let listener: CommandLifecycleListener | null = null

export function setCommandLifecycleListener(cb: CommandLifecycleListener | null): void
export function notifyCommandLifecycle(uuid: string, state: CommandLifecycleState): void
```

Used for analytics and tracking command execution state.

---

## Extension Points Summary

### For Skills
1. **File-based**: Create `.claude/skills/my-skill/SKILL.md`
2. **Bundled**: Call `registerBundledSkill()` at startup
3. **Dynamic**: Add `paths` frontmatter for conditional activation

### For Plugins
1. **Commands directory**: `commands/` in plugin root
2. **Skills directory**: `skills/` in plugin root
3. **Manifest-declared**: `commandsPaths` and `skillsPaths` in manifest
4. **Built-in**: Call `registerBuiltinPlugin()` for CLI-shipped plugins

### For Workflows
- Workflow scripts auto-generate commands when `WORKFLOW_SCRIPTS` feature is enabled
- Badged as `(workflow)` in UI

### Security Considerations
1. **MCP skills**: Shell command execution disabled (remote/untrusted)
2. **Plugin skills**: Shell commands execute with plugin root as cwd
3. **File extraction**: Uses O_NOFOLLOW, O_EXCL to prevent symlink attacks
4. **Path validation**: Resolves paths to prevent directory traversal
5. **Gitignore respect**: Dynamic skill discovery skips gitignored directories

---

## Summary

The Commands System is a mature, multi-layered architecture that balances:

- **Flexibility**: Three command types for different use cases
- **Performance**: Aggressive memoization and lazy loading
- **Extensibility**: Skills, plugins, workflows, bundled commands
- **Security**: Availability filtering, auth checks, safe file operations
- **User Experience**: Dynamic discovery, conditional activation, rich UI support

The memoization strategy is particularly well-designed, separating expensive I/O (cached) from fast auth checks (recomputed) while providing clear cache invalidation paths.
