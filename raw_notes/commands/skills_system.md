# Claude Code Skills System Analysis

## Overview

The Skills System in Claude Code is a modular architecture for extending Claude's capabilities through custom commands. Skills can be bundled with the CLI, loaded dynamically from disk, or provided by plugins/MCP servers.

## Skill Definition Format

### Skill Manifest (SKILL.md)

Skills are defined in markdown files with YAML frontmatter:

```yaml
---
name: skill-name                    # Optional display name
description: What this skill does   # Required: shown in UI
when_to_use: Detailed scenarios     # Optional: guides model usage
allowed-tools: ["Read", "Edit"]     # Optional: tool restrictions
argument-hint: "[query]"            # Optional: args hint for UI
user-invocable: true                # Default: true (user can type /skill)
model: sonnet                       # Optional: specific model to use
disable-model-invocation: false     # Default: false (model can invoke)
context: inline                     # "inline" or "fork" (sub-agent)
agent: general-purpose              # Agent type for fork mode
effort: high                        # Optional: low/medium/high/max
paths: ["src/**/*.ts"]              # Conditional: activate on file patterns
shell: bash                         # Shell for !`cmd` blocks
hooks:                              # Hook definitions
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: "prettier --write $FILE"
---

# Skill content (markdown)

## Goal
What this skill accomplishes...

## Steps
1. Step one
2. Step two

## Rules
- Rule one
- Rule two
```

### Frontmatter Fields Reference

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name (defaults to directory name) |
| `description` | string | Required. Shown in UI and help |
| `when_to_use` | string | Usage guidance for the model |
| `allowed-tools` | string[] | Tool allowlist for this skill |
| `argument-hint` | string | Argument placeholder in UI |
| `user-invocable` | boolean | Can user type `/skill-name`? |
| `model` | string | Model override |
| `disable-model-invocation` | boolean | Block model from invoking via Skill tool |
| `context` | "inline" \| "fork" | Execution context |
| `agent` | string | Agent type for forked execution |
| `effort` | string | Effort level for model |
| `paths` | string[] | File patterns for conditional activation |
| `shell` | "bash" \| "powershell" | Shell for !`cmd` execution |
| `hooks` | HooksSettings | Lifecycle hooks |

## Skill Directory Structure

### Disk-Based Skills Location

```
# User-level skills
~/.claude/skills/
  skill-name/
    SKILL.md

# Project-level skills
.claude/skills/
  skill-name/
    SKILL.md
  another-skill/
    SKILL.md

# Legacy commands (still supported)
~/.claude/commands/
  command-name.md          # Single-file format
  cmd-dir/
    SKILL.md               # Directory format
```

### Skill Directory Format

Only directory format is supported in `/skills/`:
- Each skill is a directory
- Directory name becomes the command name
- Must contain `SKILL.md` (case-insensitive: `skill.md`, `Skill.md`)
- Additional files can be referenced via `${CLAUDE_SKILL_DIR}` variable

### Legacy Commands Directory Format

Supports both:
1. **Directory format**: `command-name/SKILL.md` (same as skills)
2. **Single-file format**: `command-name.md` (file name becomes command name)

For single files, the file stem becomes the command name: `my-command.md` -> `/my-command`

## Loading Architecture

### Skill Loading Pipeline

```
Startup:
  1. Register bundled skills (synchronous, programmatic)
  2. Load skills from disk (async, memoized)
     - Managed: ~/.claude/.managed/.claude/skills/
     - User: ~/.claude/skills/
     - Project: .claude/skills/ (walk up to home)
     - Additional: --add-dir paths
  3. Load legacy commands (async)
     - ~/.claude/commands/
     - .claude/commands/ (walk up to home)
  4. Load plugin skills (async)
  5. Deduplicate by resolved path

Runtime (file operations):
  6. Discover nested skill directories
  7. Activate conditional skills by path patterns
  8. Notify listeners -> clear caches
```

### Source Priority (Loading Order)

1. **Bundled skills** - Compiled into CLI binary
2. **Built-in plugin skills** - From enabled built-in plugins
3. **Skill directory commands** - From ~/.claude/skills/, .claude/skills/
4. **Workflow commands** - Dynamic workflow scripts
5. **Plugin commands** - From installed plugins
6. **Built-in commands** - Core CLI commands

Within skill directories, deeper paths take precedence (closer to files).

### Memoization Strategy

```typescript
// Memoized functions (cached per CWD)
getSkillDirCommands(cwd)      // Disk skills loading
loadAllCommands(cwd)          // Full command aggregation
getSkillToolCommands(cwd)     // Skill tool eligible commands
getSlashCommandToolSkills(cwd) // Slash command eligible skills

// Dynamic skills (not memoized, runtime-discovered)
getDynamicSkills()            // Session-discovered skills
```

Cache invalidation:
- `clearSkillCaches()` - Clears skill directory caches
- `clearCommandsCache()` - Clears all command caches
- File watcher triggers automatic reload

## Skill-to-Command Transformation

### Type Transformation

```typescript
// Skill files become PromptCommand objects
interface PromptCommand {
  type: 'prompt'
  name: string                    // Command name
  description: string             // From frontmatter or extracted
  allowedTools: string[]          // Tool restrictions
  userInvocable: boolean          // Can user invoke?
  disableModelInvocation: boolean // Can model invoke?
  source: SettingSource | 'bundled' | 'mcp' | 'plugin'
  loadedFrom: 'skills' | 'commands_DEPRECATED' | 'bundled' | 'mcp' | 'plugin'
  skillRoot?: string              // Base directory for resources
  hooks?: HooksSettings          // Lifecycle hooks
  context?: 'inline' | 'fork'    // Execution context
  agent?: string                  // Agent type for fork
  paths?: string[]               // Conditional patterns
  effort?: EffortValue
  contentLength: number          // For token estimation
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

### Prompt Construction

The `getPromptForCommand` function transforms SKILL.md content:

1. **Base directory injection**: Prepends `Base directory for this skill: <path>`
2. **Argument substitution**: Replaces `${argName}` with user-provided arguments
3. **Variable replacement**:
   - `${CLAUDE_SKILL_DIR}` -> Skill's directory path
   - `${CLAUDE_SESSION_ID}` -> Current session ID
4. **Shell execution**: Runs `!`command`` and ```! ... ``` blocks (unless MCP skill)
5. **Returns**: Array of ContentBlockParam for the model

### Command Name Resolution

```typescript
// For skills/ format: directory name
skill-name/SKILL.md -> /skill-name

// For legacy commands/ single-file: file stem
command-name.md -> /command-name

// For legacy commands/ directory: parent directory name
cmd-dir/SKILL.md -> /cmd-dir

// Namespaced: parent directories become namespace
sub/dir/skill/SKILL.md -> /sub:dir:skill
```

## Bundled vs Dynamic Skills

### Bundled Skills

**Registration** (compile-time):
```typescript
// skills/bundled/updateConfig.ts
registerBundledSkill({
  name: 'update-config',
  description: 'Configure Claude Code via settings.json',
  allowedTools: ['Read'],
  userInvocable: true,
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: SKILL_PROMPT }]
  }
})
```

**Characteristics**:
- Registered synchronously at startup via `initBundledSkills()`
- Stored in internal array `bundledSkills`
- Can include inline `files` record (extracted to disk on first use)
- Support `isEnabled()` for conditional visibility
- No disk I/O until invoked

**Bundled skill files**:
When `files` is provided, content is extracted to a temp directory on first invocation:
```typescript
registerBundledSkill({
  name: 'my-skill',
  files: {
    'utils/helper.ts': 'export function helper() {...}',
    'config.json': '{"key": "value"}'
  },
  getPromptForCommand: async (args, ctx) => {...}
})
```

### Dynamic Skills

**Sources**:
1. **Disk-based**: From `.claude/skills/` directories
2. **Plugin-provided**: From installed plugins
3. **MCP-provided**: From MCP servers (remote skills)
4. **Dynamically discovered**: Found during file operations

**Characteristics**:
- Loaded asynchronously
- Memoized per CWD for performance
- File-watched for hot-reloading
- Can be conditional (path-filtered)

## Dynamic Discovery Patterns

### Discovery During File Operations

When files are read/written, the system discovers nested skill directories:

```typescript
// In loadSkillsDir.ts
discoverSkillDirsForPaths(filePaths: string[], cwd: string): string[]
```

**Algorithm**:
1. For each file path, walk up to CWD (but not including CWD)
2. Check for `.claude/skills/` at each level
3. Skip if already discovered (tracked in `dynamicSkillDirs` Set)
4. Skip if directory is gitignored
5. Return new directories sorted by depth (deepest first)

### Conditional Skills (Path-Filtered)

Skills with `paths` frontmatter are conditionally activated:

```yaml
---
name: typescript-helpers
paths:
  - "src/**/*.ts"
  - "*.tsx"
---
```

**Activation flow**:
1. On startup, skills with `paths` are stored in `conditionalSkills` Map
2. When files are touched, `activateConditionalSkillsForPaths()` checks matches
3. Uses `ignore` library (gitignore-style matching) against relative paths
4. Activated skills move to `dynamicSkills` Map
5. Listeners notified via `skillsLoaded.emit()`

**Pattern matching**:
- Uses `ignore` npm package (same as gitignore)
- Patterns relative to CWD
- `**` matches any number of directories
- `*.ts` matches TypeScript files
- Can use brace expansion: `*.{ts,tsx}`

### Dynamic Skill State

```typescript
// State containers in loadSkillsDir.ts
const dynamicSkillDirs = new Set<string>()     // Discovered directories
const dynamicSkills = new Map<string, Command>() // Loaded skills
const conditionalSkills = new Map<string, Command>() // Pending activation
const activatedConditionalSkillNames = new Set<string>() // Activated names
```

## Skill Hooks System

### Hook Types

Skills can define hooks in frontmatter that run at specific lifecycle events:

```yaml
hooks:
  PreToolUse:          # Before tool execution
    - matcher: "Write|Edit"
      hooks: [...]
  PostToolUse:         # After successful tool execution
    - matcher: "Bash"
      hooks: [...]
  PostToolUseFailure:  # After failed tool execution
  Stop:                # When session stops
  SessionStart:        # When session starts
  PreCompact:          # Before compaction
  PostCompact:         # After compaction
  UserPromptSubmit:    # When user submits prompt
  Notification:        # On notifications
  PermissionRequest:   # Before permission prompt
```

### Hook Command Types

1. **Command hook** - Shell command:
   ```yaml
   - type: command
     command: "prettier --write $FILE"
     shell: bash
     timeout: 30
     once: false
     async: false
   ```

2. **Prompt hook** - LLM evaluation:
   ```yaml
   - type: prompt
     prompt: "Is this safe? $ARGUMENTS"
     model: sonnet
     timeout: 60
   ```

3. **Agent hook** - Agentic verification:
   ```yaml
   - type: agent
     prompt: "Verify tests pass"
     timeout: 60
   ```

4. **HTTP hook** - HTTP POST:
   ```yaml
   - type: http
     url: "https://hooks.example.com/trigger"
     headers:
       Authorization: "Bearer $TOKEN"
     allowedEnvVars: ["TOKEN"]
   ```

### Hook Execution Context

Hooks receive JSON on stdin:
```json
{
  "session_id": "abc123",
  "tool_name": "Write",
  "tool_input": {"file_path": "/path/to/file", "content": "..."},
  "tool_response": {"success": true}  // PostToolUse only
}
```

Hooks can return JSON to control behavior:
```json
{
  "systemMessage": "Warning to user",
  "continue": false,           // Block operation
  "stopReason": "Why blocked",
  "suppressOutput": false,
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "additionalContext": "Context for model"
  }
}
```

## Lazy Loading of Skill Commands

### Command Loading Strategies

1. **Built-in commands**: Loaded at startup (static imports)
2. **Bundled skills**: Registered at startup, prompt generated on invoke
3. **Disk skills**: Loaded on first `getCommands()` call, memoized
4. **Plugin commands**: Lazy-loaded via dynamic imports

### Lazy Loading Pattern for Local Commands

```typescript
// commands/skills/index.ts
const skills = {
  type: 'local-jsx',
  name: 'skills',
  description: 'List available skills',
  load: () => import('./skills.js'),  // Lazy load
} satisfies Command
```

The `load()` function returns a module with a `call()` function, deferring heavy dependencies until invocation.

### Memoization for Performance

```typescript
// Expensive disk operations are memoized
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    // Load from multiple sources in parallel
    const [managed, user, project, additional, legacy] = await Promise.all([
      loadSkillsFromSkillsDir(managedDir, 'policySettings'),
      loadSkillsFromSkillsDir(userDir, 'userSettings'),
      // ...
    ])
    // Deduplicate and return
  }
)
```

### Hot Reload Implementation

File watching via `skillChangeDetector.ts`:

```typescript
// Uses chokidar for cross-platform file watching
watcher = chokidar.watch(paths, {
  persistent: true,
  depth: 2,  // Skills use skill-name/SKILL.md format
  awaitWriteFinish: { stabilityThreshold: 1000 },
  usePolling: typeof Bun !== 'undefined',  // Bun workaround
})

// Debounced reload (300ms) to handle batch changes
function scheduleReload(changedPath: string) {
  pendingChangedPaths.add(changedPath)
  if (reloadTimer) clearTimeout(reloadTimer)
  reloadTimer = setTimeout(() => {
    clearSkillCaches()
    clearCommandsCache()
    skillsChanged.emit()
  }, RELOAD_DEBOUNCE_MS)
}
```

## MCP Skill Builders

For MCP-provided skills, a registry pattern breaks import cycles:

```typescript
// skills/mcpSkillBuilders.ts
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) throw new Error('MCP skill builders not registered')
  return builders
}
```

Registered at `loadSkillsDir.ts` module init:
```typescript
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

This allows MCP skills to use the same parsing/creation logic without creating circular dependencies.

## Key Files Reference

| File | Purpose |
|------|---------|
| `skills/loadSkillsDir.ts` | Core skill loading logic, dynamic discovery |
| `skills/bundledSkills.ts` | Bundled skill registration, file extraction |
| `skills/bundled/*.ts` | Individual bundled skill implementations |
| `skills/bundled/index.ts` | Init function for all bundled skills |
| `skills/mcpSkillBuilders.ts` | Registry for MCP skill creation |
| `commands/skills/index.ts` | `/skills` command definition |
| `commands.ts` | Command aggregation and filtering |
| `types/command.ts` | Command type definitions |
| `utils/skills/skillChangeDetector.ts` | File watching and hot reload |
| `schemas/hooks.ts` | Hook schema definitions |
| `utils/frontmatterParser.ts` | Frontmatter parsing utilities |
| `utils/settings/types.ts` | Settings and hook type definitions |

## Summary

The Skills System provides a flexible, multi-source architecture for extending Claude Code:

1. **Unified interface**: All skills become Command objects with `getPromptForCommand()`
2. **Multiple sources**: Bundled, disk-based, plugin, MCP - all treated uniformly
3. **Conditional activation**: Path-based patterns enable context-aware skill availability
4. **Hot reloading**: File watching enables rapid iteration during skill development
5. **Security**: Gitignore checking, path traversal prevention, shell command sandboxing
6. **Performance**: Memoization, lazy loading, and efficient file watching
7. **Extensibility**: Hook system allows skills to respond to lifecycle events
