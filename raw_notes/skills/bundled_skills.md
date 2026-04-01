# Bundled Skills System Analysis

## Overview

The Bundled Skills system allows Claude Code to ship built-in skills that are compiled into the CLI binary. These skills are registered programmatically at startup and are available to all users.

## Key Files

- `/home/claudeuser/ClaudeCode/skills/bundledSkills.ts` - Core registration API and file extraction logic
- `/home/claudeuser/ClaudeCode/skills/bundled/index.ts` - Initialization entry point that registers all bundled skills
- `/home/claudeuser/ClaudeCode/skills/bundled/*.ts` - Individual bundled skill implementations

## Registration Patterns

### 1. Startup Registration Flow

```typescript
// In main.tsx, line 1925
initBundledSkills();
```

The `initBundledSkills()` function in `/home/claudeuser/ClaudeCode/skills/bundled/index.ts` calls individual registration functions for each skill:

```typescript
export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()
  // ... feature-flagged skills
}
```

### 2. The registerBundledSkill() API

Located in `/home/claudeuser/ClaudeCode/skills/bundledSkills.ts` (lines 53-100):

```typescript
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  // Extracts embedded files if present
  // Creates a Command object
  // Adds to bundledSkills registry
}
```

**BundledSkillDefinition Interface** (lines 15-41):

```typescript
export type BundledSkillDefinition = {
  name: string                    // Skill command name (e.g., "verify")
  description: string             // Description shown in UI
  aliases?: string[]              // Optional command aliases
  whenToUse?: string              // Trigger guidance for model
  argumentHint?: string           // Argument placeholder text
  allowedTools?: string[]         // Tool permissions for this skill
  model?: string                  // Model override
  disableModelInvocation?: boolean
  userInvocable?: boolean         // Show in command palette
  isEnabled?: () => boolean       // Dynamic enablement check
  hooks?: HooksSettings
  context?: 'inline' | 'fork'     // Execution context
  agent?: string                  // Agent configuration
  files?: Record<string, string>  // Embedded reference files
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

## Embedded File Handling

### 1. File Structure

Skills can embed files using the `files` property:

```typescript
// From verifyContent.ts
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,
  'examples/server.md': serverMd,
}
```

### 2. Lazy Extraction

Files are extracted lazily on first skill invocation (lines 59-73):

```typescript
if (files && Object.keys(files).length > 0) {
  skillRoot = getBundledSkillExtractDir(definition.name)
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
```

**Key points:**
- Extraction happens once per process (memoized via closure)
- Uses promise memoization to prevent race conditions
- Files are written to a per-process nonce directory
- Base directory is prepended to the prompt so the model can Read/Grep files

### 3. Extraction Directory

```typescript
export function getBundledSkillExtractDir(skillName: string): string {
  return join(getBundledSkillsRoot(), skillName)
}

// getBundledSkillsRoot returns:
// /tmp/claude-{uid}/bundled-skills/{VERSION}/{16-byte-nonce}/
```

Security features:
- Per-process random nonce (16 bytes hex)
- Version-scoped to prevent stale extractions
- 0o700/0o600 permissions for owner-only access
- O_NOFOLLOW|O_EXCL flags to prevent symlink attacks
- Path traversal validation (no `..` allowed)

### 4. File Writing

The `writeSkillFiles` function (lines 147-167):
- Groups files by parent directory
- Creates directories recursively with 0o700 mode
- Writes files with safe flags (O_WRONLY|O_CREAT|O_EXCL|O_NOFOLLOW)
- Uses atomic file operations

## Example Skill Implementations

### Simple Skill (No Embedded Files)

**simplify.ts** - A straightforward skill with inline prompt:

```typescript
const SIMPLIFY_PROMPT = `# Simplify: Code Review and Cleanup
Review all changed files for reuse, quality, and efficiency...
`

export function registerSimplifySkill(): void {
  registerBundledSkill({
    name: 'simplify',
    description: 'Review changed code for reuse, quality, and efficiency...',
    userInvocable: true,
    async getPromptForCommand(args) {
      let prompt = SIMPLIFY_PROMPT
      if (args) {
        prompt += `\n\n## Additional Focus\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

### Skill with Embedded Files

**verify.ts** - Uses separate content file for embedded resources:

```typescript
import { SKILL_FILES, SKILL_MD } from './verifyContent.js'

export function registerVerifySkill(): void {
  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,
    userInvocable: true,
    files: SKILL_FILES,  // Embedded files
    async getPromptForCommand(args) {
      const parts: string[] = [SKILL_BODY.trimStart()]
      if (args) {
        parts.push(`## User Request\n\n${args}`)
      }
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
  })
}
```

**verifyContent.ts**:

```typescript
import cliMd from './verify/examples/cli.md'
import serverMd from './verify/examples/server.md'
import skillMd from './verify/SKILL.md'

export const SKILL_MD: string = skillMd
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,
  'examples/server.md': serverMd,
}
```

### Complex Skill with Dynamic Content

**claudeApi.ts** - Language detection and selective file inclusion:

```typescript
export function registerClaudeApiSkill(): void {
  registerBundledSkill({
    name: 'claude-api',
    description: 'Build apps with the Claude API...',
    allowedTools: ['Read', 'Grep', 'Glob', 'WebFetch'],
    userInvocable: true,
    async getPromptForCommand(args) {
      const content = await import('./claudeApiContent.js')  // Lazy import
      const lang = await detectLanguage()  // Dynamic detection
      const prompt = buildPrompt(lang, args, content)
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

### Skill with Conditional Registration

**stuck.ts** - ANT-only skill:

```typescript
export function registerStuckSkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // Skip registration for non-ANT users
  }

  registerBundledSkill({
    name: 'stuck',
    description: '[ANT-ONLY] Investigate frozen/stuck/slow Claude Code sessions...',
    userInvocable: true,
    async getPromptForCommand(args) {
      // ...
    },
  })
}
```

### Skill with Dynamic Enablement

**remember.ts** - Uses `isEnabled` callback:

```typescript
registerBundledSkill({
  name: 'remember',
  description: 'Review auto-memory entries...',
  userInvocable: true,
  isEnabled: () => isAutoMemoryEnabled(),  // Dynamic check
  async getPromptForCommand(args) {
    // ...
  },
})
```

## Skill Retrieval

Bundled skills are retrieved via `getBundledSkills()` in `/home/claudeuser/ClaudeCode/commands.ts` (line 375):

```typescript
const bundledSkills = getBundledSkills()
```

The function returns a copy of the registry to prevent external mutation (line 106-108):

```typescript
export function getBundledSkills(): Command[] {
  return [...bundledSkills]
}
```

## Summary

The Bundled Skills system provides:

1. **Compiled-in Skills** - Skills ship with the CLI binary, no external setup needed
2. **Lazy File Extraction** - Embedded files extracted on first use with security protections
3. **Flexible Registration** - Support for conditional registration, dynamic enablement, and feature flags
4. **Type Safety** - Full TypeScript definitions for skill definitions
5. **Security** - Path traversal protection, symlink attack prevention, and permission restrictions
