# Claude Code Skill Loading System - Deep Dive Analysis

## Overview

The Skill Loading System is responsible for discovering, parsing, validating, and loading skills from various sources. Skills are essentially prompt templates that can be invoked via slash commands (`/skill-name`) or triggered automatically based on file patterns.

## File Structure

| File | Purpose |
|------|---------|
| `/home/claudeuser/ClaudeCode/skills/loadSkillsDir.ts` | Main skill loading orchestration |
| `/home/claudeuser/ClaudeCode/skills/bundledSkills.ts` | Built-in skills registry |
| `/home/claudeuser/ClaudeCode/skills/mcpSkillBuilders.ts` | MCP skill builder registration |
| `/home/claudeuser/ClaudeCode/utils/frontmatterParser.ts` | YAML frontmatter parsing |
| `/home/claudeuser/ClaudeCode/utils/argumentSubstitution.ts` | Variable substitution logic |
| `/home/claudeuser/ClaudeCode/utils/markdownConfigLoader.ts` | Markdown file discovery |
| `/home/claudeuser/ClaudeCode/utils/skills/skillChangeDetector.ts` | File watching and hot reload |
| `/home/claudeuser/ClaudeCode/schemas/hooks.ts` | Zod schemas for hooks |
| `/home/claudeuser/ClaudeCode/utils/settings/types.ts` | Settings schema with Zod |

---

## 1. Discovery Algorithm

### 1.1 Directory Structure

Skills are discovered from multiple sources in priority order:

```
Priority (highest to lowest):
1. Managed/Policy: ~/.claude-code/managed/.claude/skills/
2. User: ~/.claude/skills/
3. Project: ./.claude/skills/ (walks up to git root)
4. Additional: --add-dir paths
5. Legacy: ~/.claude/commands/ (DEPRECATED)
```

### 1.2 Skill Directory Format

Skills use a **directory-based** structure (not single files):

```
.skills/
├── my-skill/
│   └── SKILL.md          # Required: skill definition
├── another-skill/
│   ├── SKILL.md
│   └── helper-script.sh  # Optional: bundled files
```

### 1.3 Discovery Process (`getSkillDirCommands`)

```typescript
// From loadSkillsDir.ts lines 638-804
export const getSkillDirCommands = memoize(async (cwd: string): Promise<Command[]> => {
  // 1. Define all source directories
  const userSkillsDir = join(getClaudeConfigHomeDir(), 'skills')
  const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
  const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)
  const additionalDirs = getAdditionalDirectoriesForClaudeMd()

  // 2. Load from all sources in parallel
  const [managedSkills, userSkills, projectSkillsNested, additionalSkillsNested, legacyCommands] =
    await Promise.all([...])

  // 3. Combine and deduplicate
  const allSkillsWithPaths = [...managedSkills, ...userSkills, ...projectSkillsNested.flat(), ...]

  // 4. Deduplicate by resolved realpath (handles symlinks)
  const fileIds = await Promise.all(allSkillsWithPaths.map(({ filePath }) =>
    skill.type === 'prompt' ? getFileIdentity(filePath) : Promise.resolve(null)
  ))

  // 5. Separate conditional vs unconditional skills
  // Conditional skills have 'paths' frontmatter and activate on file matches
})
```

### 1.4 Dynamic Skill Discovery

Skills can be discovered dynamically during file operations:

```typescript
// From loadSkillsDir.ts lines 861-915
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]> {
  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)

    // Walk up from file to cwd (not including cwd)
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')

      // Check if already discovered (avoid repeated stat calls)
      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)
        try {
          await fs.stat(skillDir)
          // Skip gitignored directories for security
          if (await isPathGitignored(currentDir, resolvedCwd)) continue
          newDirs.push(skillDir)
        } catch { /* doesn't exist */ }
      }
      currentDir = dirname(currentDir)
    }
  }

  // Sort by depth (deepest first) - closer skills take precedence
  return newDirs.sort((a, b) => b.split(pathSep).length - a.split(pathSep).length)
}
```

### 1.5 Conditional Skills Activation

Skills with `paths:` frontmatter are conditionally activated:

```typescript
// From loadSkillsDir.ts lines 997-1058
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)  // Uses 'ignore' library

    for (const filePath of filePaths) {
      const relativePath = isAbsolute(filePath) ? relative(cwd, filePath) : filePath

      if (skillIgnore.ignores(relativePath)) {
        // Activate: move from conditional to dynamic skills
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        break
      }
    }
  }
}
```

---

## 2. Parsing Pipeline

### 2.1 SKILL.md Format

```markdown
---
name: Display Name
description: What this skill does
when_to_use: When to invoke this skill
arguments: arg1 arg2          # Named arguments
argument-hint: "[arg1] [arg2]"
allowed-tools: Read, Write   # Tool whitelist
user-invocable: true         # Can users type /command?
model: sonnet                # Override model
effort: high                 # Thinking effort
paths: "src/**/*.ts"         # Conditional activation paths
shell: bash                  # For !`cmd` execution
hooks:                       # Lifecycle hooks
  PreToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "echo Reading file"
---

# Skill Content

This is the prompt content that gets sent to the model.
Supports variable substitution: ${arg1}, ${CLAUDE_SKILL_DIR}

## Dynamic content

!`echo "Current branch: $(git branch --show-current)"`
```

### 2.2 Frontmatter Parsing

**Step 1: Extract frontmatter and content**

```typescript
// From frontmatterParser.ts lines 123-175
export const FRONTMATTER_REGEX = /^---\s*\n([\s\S]*?)---\s*\n?/

export function parseFrontmatter(markdown: string, sourcePath?: string): ParsedMarkdown {
  const match = markdown.match(FRONTMATTER_REGEX)

  if (!match) {
    return { frontmatter: {}, content: markdown }
  }

  const frontmatterText = match[1] || ''
  const content = markdown.slice(match[0].length)

  // Parse YAML with special character handling
  let frontmatter: FrontmatterData = {}
  try {
    const parsed = parseYaml(frontmatterText)
    if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
      frontmatter = parsed
    }
  } catch {
    // Retry with quoted values for problematic characters
    const quotedText = quoteProblematicValues(frontmatterText)
    const parsed = parseYaml(quotedText)
    // ...
  }

  return { frontmatter, content }
}
```

**Step 2: Handle special YAML characters**

```typescript
// From frontmatterParser.ts lines 66-121
const YAML_SPECIAL_CHARS = /[{}[\]*&#!|>%@`]|:\s/

function quoteProblematicValues(frontmatterText: string): string {
  const lines = frontmatterText.split('\n')

  for (const line of lines) {
    const match = line.match(/^([a-zA-Z_-]+):\s+(.+)$/)
    if (match) {
      const [, key, value] = match
      // Skip already quoted values
      if ((value.startsWith('"') && value.endsWith('"')) ||
          (value.startsWith("'") && value.endsWith("'"))) {
        continue
      }
      // Quote if contains special YAML characters
      if (YAML_SPECIAL_CHARS.test(value)) {
        const escaped = value.replace(/\\/g, '\\\\').replace(/"/g, '\\"')
        result.push(`${key}: "${escaped}"`)
      }
    }
  }
}
```

### 2.3 Frontmatter Field Parsing

**Paths expansion with brace expansion:**

```typescript
// From frontmatterParser.ts lines 189-266
export function splitPathInFrontmatter(input: string | string[]): string[] {
  if (Array.isArray(input)) {
    return input.flatMap(splitPathInFrontmatter)
  }

  // Split by comma while respecting braces
  const parts: string[] = []
  let current = ''
  let braceDepth = 0

  for (const char of input) {
    if (char === '{') braceDepth++
    else if (char === '}') braceDepth--
    else if (char === ',' && braceDepth === 0) {
      // Split here
      parts.push(current.trim())
      current = ''
      continue
    }
    current += char
  }

  // Expand brace patterns like src/*.{ts,tsx}
  return parts.flatMap(pattern => expandBraces(pattern))
}

function expandBraces(pattern: string): string[] {
  const braceMatch = pattern.match(/^([^{]*)\{([^}]+)\}(.*)$/)
  if (!braceMatch) return [pattern]

  const [, prefix, alternatives, suffix] = braceMatch
  const parts = alternatives.split(',').map(alt => alt.trim())

  const expanded: string[] = []
  for (const part of parts) {
    const combined = prefix + part + suffix
    expanded.push(...expandBraces(combined))  // Recursive for nested braces
  }
  return expanded
}
```

### 2.4 Skill Command Creation

```typescript
// From loadSkillsDir.ts lines 270-401
export function createSkillCommand({
  skillName,
  displayName,
  description,
  markdownContent,
  allowedTools,
  argumentNames,
  baseDir,
  // ... other fields
}): Command {
  return {
    type: 'prompt',
    name: skillName,
    description,
    allowedTools,
    argNames: argumentNames.length > 0 ? argumentNames : undefined,
    // ...

    async getPromptForCommand(args, toolUseContext) {
      let finalContent = baseDir
        ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
        : markdownContent

      // Step 1: Substitute named/indexed arguments
      finalContent = substituteArguments(finalContent, args, true, argumentNames)

      // Step 2: Replace ${CLAUDE_SKILL_DIR}
      if (baseDir) {
        const skillDir = process.platform === 'win32'
          ? baseDir.replace(/\\/g, '/')  // Normalize backslashes
          : baseDir
        finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
      }

      // Step 3: Replace ${CLAUDE_SESSION_ID}
      finalContent = finalContent.replace(/\$\{CLAUDE_SESSION_ID\}/g, getSessionId())

      // Step 4: Execute shell commands in prompt (for !`cmd` and ```! ... ```)
      if (loadedFrom !== 'mcp') {
        finalContent = await executeShellCommandsInPrompt(finalContent, toolUseContext, `/${skillName}`, shell)
      }

      return [{ type: 'text', text: finalContent }]
    },
  }
}
```

---

## 3. Schema Validation with Zod

### 3.1 Hooks Schema Definition

```typescript
// From schemas/hooks.ts lines 1-223
import { z } from 'zod/v4'
import { lazySchema } from '../utils/lazySchema.js'

// Lazy schema factory to avoid circular imports
const IfConditionSchema = lazySchema(() =>
  z.string().optional().describe('Permission rule syntax to filter when this hook runs')
)

function buildHookSchemas() {
  const BashCommandHookSchema = z.object({
    type: z.literal('command'),
    command: z.string(),
    if: IfConditionSchema(),
    shell: z.enum(['bash', 'powershell']).optional(),
    timeout: z.number().positive().optional(),
    statusMessage: z.string().optional(),
    once: z.boolean().optional(),
    async: z.boolean().optional(),
    asyncRewake: z.boolean().optional(),
  })

  const PromptHookSchema = z.object({
    type: z.literal('prompt'),
    prompt: z.string(),
    if: IfConditionSchema(),
    timeout: z.number().positive().optional(),
    model: z.string().optional(),
    // ...
  })

  const HttpHookSchema = z.object({
    type: z.literal('http'),
    url: z.string().url(),
    headers: z.record(z.string(), z.string()).optional(),
    allowedEnvVars: z.array(z.string()).optional(),
    // ...
  })

  const AgentHookSchema = z.object({
    type: z.literal('agent'),
    prompt: z.string(),
    timeout: z.number().positive().optional(),
    model: z.string().optional(),
    // ...
  })

  return { BashCommandHookSchema, PromptHookSchema, HttpHookSchema, AgentHookSchema }
}

// Discriminated union by 'type' field
export const HookCommandSchema = lazySchema(() => {
  const { BashCommandHookSchema, PromptHookSchema, AgentHookSchema, HttpHookSchema } = buildHookSchemas()
  return z.discriminatedUnion('type', [
    BashCommandHookSchema,
    PromptHookSchema,
    AgentHookSchema,
    HttpHookSchema,
  ])
})

// Hooks settings: partial record of hook events to matchers
export const HooksSchema = lazySchema(() =>
  z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema()))
)
```

### 3.2 Skill Hooks Parsing

```typescript
// From loadSkillsDir.ts lines 136-153
function parseHooksFromFrontmatter(
  frontmatter: FrontmatterData,
  skillName: string,
): HooksSettings | undefined {
  if (!frontmatter.hooks) return undefined

  const result = HooksSchema().safeParse(frontmatter.hooks)
  if (!result.success) {
    logForDebugging(`Invalid hooks in skill '${skillName}': ${result.error.message}`)
    return undefined
  }
  return result.data
}
```

### 3.3 Lazy Schema Pattern

```typescript
// From utils/lazySchema.ts (referenced)
export function lazySchema<T>(factory: () => z.ZodType<T>): () => z.ZodType<T> {
  let cached: z.ZodType<T> | undefined
  return () => {
    if (!cached) {
      cached = factory()
    }
    return cached
  }
}
```

---

## 4. Variable Substitution

### 4.1 Argument Parsing

```typescript
// From argumentSubstitution.ts lines 24-40
export function parseArguments(args: string): string[] {
  if (!args || !args.trim()) return []

  // Use shell-quote for proper parsing
  const result = tryParseShellCommand(args, key => `$${key}`)
  if (!result.success) {
    // Fallback: simple whitespace split
    return args.split(/\s+/).filter(Boolean)
  }

  return result.tokens.filter((token): token is string => typeof token === 'string')
}
```

### 4.2 Argument Name Parsing

```typescript
// From argumentSubstitution.ts lines 50-68
export function parseArgumentNames(
  argumentNames: string | string[] | undefined,
): string[] {
  if (!argumentNames) return []

  const isValidName = (name: string): boolean =>
    typeof name === 'string' &&
    name.trim() !== '' &&
    !/^\d+$/.test(name)  // Reject numeric-only names (conflict with $0, $1)

  if (Array.isArray(argumentNames)) {
    return argumentNames.filter(isValidName)
  }
  if (typeof argumentNames === 'string') {
    return argumentNames.split(/\s+/).filter(isValidName)
  }
  return []
}
```

### 4.3 Substitution Logic

```typescript
// From argumentSubstitution.ts lines 94-145
export function substituteArguments(
  content: string,
  args: string | undefined,
  appendIfNoPlaceholder = true,
  argumentNames: string[] = [],
): string {
  if (args === undefined || args === null) return content

  const parsedArgs = parseArguments(args)
  const originalContent = content

  // 1. Replace named arguments ($foo, $bar)
  // Matches $name but not $name[...] or $nameXxx (word chars)
  for (let i = 0; i < argumentNames.length; i++) {
    const name = argumentNames[i]
    if (!name) continue
    content = content.replace(
      new RegExp(`\\$${name}(?![\\[\\w])`, 'g'),
      parsedArgs[i] ?? ''
    )
  }

  // 2. Replace indexed arguments ($ARGUMENTS[0], $ARGUMENTS[1], ...)
  content = content.replace(/\$ARGUMENTS\[(\d+)\]/g, (_, indexStr: string) => {
    const index = parseInt(indexStr, 10)
    return parsedArgs[index] ?? ''
  })

  // 3. Replace shorthand indexed arguments ($0, $1, ...)
  content = content.replace(/\$(\d+)(?!\w)/g, (_, indexStr: string) => {
    const index = parseInt(indexStr, 10)
    return parsedArgs[index] ?? ''
  })

  // 4. Replace $ARGUMENTS with full args string
  content = content.replaceAll('$ARGUMENTS', args)

  // 5. If no placeholders found, append arguments
  if (content === originalContent && appendIfNoPlaceholder && args) {
    content = content + `\n\nARGUMENTS: ${args}`
  }

  return content
}
```

### 4.4 Built-in Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `${argName}` | Named argument from frontmatter | Frontmatter `arguments:` field |
| `$ARGUMENTS` | Full arguments string | User input |
| `$ARGUMENTS[n]` | Indexed argument (0-based) | User input |
| `$n` | Shorthand for $ARGUMENTS[n] | User input |
| `${CLAUDE_SKILL_DIR}` | Skill's directory path | Resolved at load time |
| `${CLAUDE_SESSION_ID}` | Current session ID | `getSessionId()` |

---

## 5. Caching Strategy

### 5.1 Memoization Layers

```typescript
// From loadSkillsDir.ts lines 638-804
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    // ... loading logic
  }
)

// From markdownConfigLoader.ts lines 297-430
export const loadMarkdownFilesForSubdir = memoize(
  async function (subdir: ClaudeConfigDirectory, cwd: string): Promise<MarkdownFile[]> {
    // ... loading logic
  },
  // Custom cache key: combination of subdir and cwd
  (subdir: ClaudeConfigDirectory, cwd: string) => `${subdir}:${cwd}`
)
```

### 5.2 Cache Clearing

```typescript
// From loadSkillsDir.ts lines 806-816
export function clearSkillCaches() {
  getSkillDirCommands.cache?.clear?.()        // Clear skill commands
  loadMarkdownFilesForSubdir.cache?.clear?.() // Clear markdown files
  conditionalSkills.clear()                    // Clear conditional skills
  activatedConditionalSkillNames.clear()       // Clear activated set
}
```

### 5.3 File Identity-Based Deduplication

```typescript
// From loadSkillsDir.ts lines 118-124
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)  // Resolves symlinks to canonical path
  } catch {
    return null
  }
}

// Deduplication logic (lines 742-763)
const seenFileIds = new Map<string, SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'>()

for (let i = 0; i < allSkillsWithPaths.length; i++) {
  const entry = allSkillsWithPaths[i]
  const fileId = fileIds[i]

  if (fileId === null || fileId === undefined) {
    deduplicatedSkills.push(skill)
    continue
  }

  const existingSource = seenFileIds.get(fileId)
  if (existingSource !== undefined) {
    logForDebugging(`Skipping duplicate skill '${skill.name}' from ${skill.source}`)
    continue
  }

  seenFileIds.set(fileId, skill.source)
  deduplicatedSkills.push(skill)
}
```

---

## 6. File Watching with Chokidar

### 6.1 Watcher Configuration

```typescript
// From skillChangeDetector.ts lines 62-141
const USE_POLLING = typeof Bun !== 'undefined'  // Bun workaround for deadlock

export async function initialize(): Promise<void> {
  const paths = await getWatchablePaths()

  watcher = chokidar.watch(paths, {
    persistent: true,
    ignoreInitial: true,
    depth: 2,  // Skills use skill-name/SKILL.md format
    awaitWriteFinish: {
      stabilityThreshold: FILE_STABILITY_THRESHOLD_MS,  // 1000ms
      pollInterval: FILE_STABILITY_POLL_INTERVAL_MS,    // 500ms
    },
    ignored: (path, stats) => {
      if (stats && !stats.isFile() && !stats.isDirectory()) return true
      return path.split(platformPath.sep).some(dir => dir === '.git')
    },
    ignorePermissionErrors: true,
    usePolling: USE_POLLING,  // Bun workaround
    interval: POLLING_INTERVAL_MS,  // 2000ms for polling
    atomic: true,
  })

  watcher.on('add', handleChange)
  watcher.on('change', handleChange)
  watcher.on('unlink', handleChange)
}
```

### 6.2 Debounced Reload

```typescript
// From skillChangeDetector.ts lines 255-279
function scheduleReload(changedPath: string): void {
  pendingChangedPaths.add(changedPath)
  if (reloadTimer) clearTimeout(reloadTimer)

  reloadTimer = setTimeout(async () => {
    reloadTimer = null
    const paths = [...pendingChangedPaths]
    pendingChangedPaths.clear()

    // Execute ConfigChange hooks
    const results = await executeConfigChangeHooks('skills', paths[0]!)
    if (hasBlockingResult(results)) {
      logForDebugging(`ConfigChange hook blocked skill reload`)
      return
    }

    // Clear caches and notify
    clearSkillCaches()
    clearCommandsCache()
    resetSentSkillNames()
    skillsChanged.emit()
  }, RELOAD_DEBOUNCE_MS)  // 300ms debounce
}
```

### 6.3 Watchable Paths

```typescript
// From skillChangeDetector.ts lines 171-235
async function getWatchablePaths(): Promise<string[]> {
  const paths: string[] = []

  // User skills directory (~/.claude/skills)
  const userSkillsPath = getSkillsPath('userSettings', 'skills')
  if (userSkillsPath) {
    try {
      await fs.stat(userSkillsPath)
      paths.push(userSkillsPath)
    } catch { /* skip */ }
  }

  // User commands directory (~/.claude/commands)
  const userCommandsPath = getSkillsPath('userSettings', 'commands')
  // ... similar pattern

  // Project skills directory (.claude/skills)
  const projectSkillsPath = getSkillsPath('projectSettings', 'skills')
  // ... similar pattern

  // Additional directories (--add-dir)
  for (const dir of getAdditionalDirectoriesForClaudeMd()) {
    const additionalSkillsPath = platformPath.join(dir, '.claude', 'skills')
    // ... similar pattern
  }

  return paths
}
```

---

## 7. Shell Command Execution in Skills

### 7.1 Execution Patterns

Skills can embed shell commands that execute at load time:

- **Inline**: ``!`command` ``
- **Block**: `` ```! command ``` ``

### 7.2 Security Considerations

```typescript
// From promptShellExecution.ts lines 69-143
export async function executeShellCommandsInPrompt(
  text: string,
  context: ToolUseContext,
  slashCommandName: string,
  shell?: FrontmatterShell,
): Promise<string> {
  // Security: MCP skills are remote and untrusted — never execute inline shell
  if (loadedFrom !== 'mcp') {
    finalContent = await executeShellCommandsInPrompt(...)
  }

  // Patterns
  const BLOCK_PATTERN = /```!\s*\n?([\s\S]*?)\n?```/g
  const INLINE_PATTERN = /(?<=^|\s)!`([^`]+)`/gm

  // Check permissions before executing
  const permissionResult = await hasPermissionsToUseTool(shellTool, { command }, context, ...)
  if (permissionResult.behavior !== 'allow') {
    throw new MalformedCommandError(`Permission denied`)
  }

  // Execute and replace in content
  const { data } = await shellTool.call({ command }, context)
  // ... replace match with output
}
```

### 7.3 Shell Selection

```typescript
// Shell comes from frontmatter, never from user settings
const shellTool: PromptShellTool =
  shell === 'powershell' && isPowerShellToolEnabled()
    ? getPowerShellTool()
    : BashTool
```

---

## 8. Bundled Skills

### 8.1 Registration

```typescript
// From bundledSkills.ts lines 53-100
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition
  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    // Memoize extraction promise
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }

  const command: Command = {
    type: 'prompt',
    name: definition.name,
    description: definition.description,
    // ...
    loadedFrom: 'bundled',
  }
  bundledSkills.push(command)
}
```

### 8.2 File Extraction

```typescript
// From bundledSkills.ts lines 131-145
async function extractBundledSkillFiles(
  skillName: string,
  files: Record<string, string>,
): Promise<string | null> {
  const dir = getBundledSkillExtractDir(skillName)
  try {
    await writeSkillFiles(dir, files)
    return dir
  } catch (e) {
    logForDebugging(`Failed to extract bundled skill: ${e}`)
    return null
  }
}
```

---

## 9. MCP Skill Builders

### 9.1 Registration Pattern (Dependency Injection)

```typescript
// From mcpSkillBuilders.ts lines 1-44
import type { createSkillCommand, parseSkillFrontmatterFields } from './loadSkillsDir.js'

export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error('MCP skill builders not registered')
  }
  return builders
}

// Registration at module init in loadSkillsDir.ts (lines 1083-1086)
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

---

## 10. Key Design Patterns

### 10.1 Priority Loading Order

1. **Managed/Policy** (highest priority)
2. **User**
3. **Project** (deepest first)
4. **Additional directories**
5. **Legacy commands** (lowest priority)

### 10.2 Deduplication Strategy

- Uses `realpath()` to resolve symlinks
- First occurrence wins
- Logs duplicates for debugging

### 10.3 Security Measures

- Gitignored directories are skipped
- MCP skills cannot execute shell commands
- Permission checks before shell execution
- Path traversal protection for bundled skills
- Safe file writing with `O_NOFOLLOW | O_EXCL`

### 10.4 Performance Optimizations

- Memoization with `lodash-es/memoize`
- Parallel loading with `Promise.all`
- Lazy schema initialization
- Debounced file watching
- File stability threshold (1 second)

---

## Summary

The Skill Loading System is a sophisticated multi-source configuration loader with:

1. **Hierarchical discovery** from multiple directory sources
2. **YAML frontmatter parsing** with special character handling
3. **Zod schema validation** for type safety
4. **Variable substitution** for dynamic content
5. **Memoized caching** for performance
6. **File watching** with hot reload
7. **Security boundaries** for shell execution
