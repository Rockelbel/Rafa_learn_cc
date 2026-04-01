# Claude Code /commit Command Implementation Analysis

## Overview

This document traces the implementation of the `/commit` and `/commit-push-pr` commands in the Claude Code codebase. These commands demonstrate a complete command workflow including validation, git operations, and PR creation.

## Source Files

1. `/home/claudeuser/ClaudeCode/commands/commit.ts` - Basic commit command
2. `/home/claudeuser/ClaudeCode/commands/commit-push-pr.ts` - Extended commit + push + PR flow
3. `/home/claudeuser/ClaudeCode/utils/attribution.ts` - Commit and PR attribution text generation
4. `/home/claudeuser/ClaudeCode/utils/undercover.ts` - Undercover mode for public repos
5. `/home/claudeuser/ClaudeCode/utils/promptShellExecution.ts` - Shell command execution in prompts
6. `/home/claudeuser/ClaudeCode/utils/git.ts` - Git utility functions
7. `/home/claudeuser/ClaudeCode/utils/git/gitFilesystem.ts` - Filesystem-based git operations

---

## 1. Command Structure (commit.ts)

### Command Type
The commit command is a **prompt-type command** (defined in `/home/claudeuser/ClaudeCode/types/command.ts`):

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  allowedTools?: string[]
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

### Allowed Tools (Tool Whitelist)

```typescript
const ALLOWED_TOOLS = [
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git commit:*)',
]
```

The tool whitelist uses glob patterns to restrict which tools can be called:
- `Bash(git add:*)` - Only `git add` commands
- `Bash(git status:*)` - Only `git status` commands
- `Bash(git commit:*)` - Only `git commit` commands

### Command Registration

Commands are registered in `/home/claudeuser/ClaudeCode/commands.ts`:

```typescript
import commit from './commands/commit.js'
import commitPushPr from './commands/commit-push-pr.js'

export const INTERNAL_ONLY_COMMANDS = [
  commit,
  commitPushPr,
  // ...
]
```

**Key Point**: These commands are marked as `INTERNAL_ONLY_COMMANDS` and only available when `process.env.USER_TYPE === 'ant'`.

---

## 2. Prompt Generation Flow

### Step 1: Build Prompt Content

The `getPromptContent()` function builds the prompt that will be sent to the model:

```typescript
function getPromptContent(): string {
  const { commit: commitAttribution } = getAttributionTexts()

  // Undercover mode prefix for public repos
  let prefix = ''
  if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    prefix = getUndercoverInstructions() + '\n'
  }

  return `${prefix}## Context

- Current git status: !\`git status\`
- Current git diff (staged and unstaged changes): !\`git diff HEAD\`
- Current branch: !\`git branch --show-current\`
- Recent commits: !\`git log --oneline -10\`

## Git Safety Protocol
...

## Your task
...
```

### Step 2: Shell Command Substitution

The prompt contains **inline shell commands** using the `!` syntax:
- `!\`git status\`` - Executes `git status` and substitutes output
- `!\`git diff HEAD\`` - Shows current changes
- `!\`git log --oneline -10\`` - Shows recent commit history

These are processed by `executeShellCommandsInPrompt()` in `/home/claudeuser/ClaudeCode/utils/promptShellExecution.ts`.

### Step 3: Permission Context Modification

```typescript
async getPromptForCommand(_args, context) {
  const promptContent = getPromptContent()
  const finalContent = await executeShellCommandsInPrompt(
    promptContent,
    {
      ...context,
      getAppState() {
        const appState = context.getAppState()
        return {
          ...appState,
          toolPermissionContext: {
            ...appState.toolPermissionContext,
            alwaysAllowRules: {
              ...appState.toolPermissionContext.alwaysAllowRules,
              command: ALLOWED_TOOLS,  // Inject allowed tools
            },
          },
        }
      },
    },
    '/commit',
  )

  return [{ type: 'text', text: finalContent }]
}
```

**Key Point**: The command modifies the tool permission context to allow only the whitelisted git commands.

---

## 3. Validation Flow

### Git Safety Protocol (in prompt)

The prompt includes explicit safety rules:

```markdown
## Git Safety Protocol

- NEVER update the git config
- NEVER skip hooks (--no-verify, --no-gpg-sign, etc) unless the user explicitly requests it
- CRITICAL: ALWAYS create NEW commits. NEVER use git commit --amend, unless the user explicitly requests it
- Do not commit files that likely contain secrets (.env, credentials.json, etc). Warn the user if they specifically request to commit those files
- If there are no changes to commit (i.e., no untracked files and no modifications), do not create an empty commit
- Never use git commands with the -i flag (like git rebase -i or git add -i) since they require interactive input which is not supported
```

### State Validation

The model is instructed to:
1. Check `git status` to see what files have changed
2. Check `git diff HEAD` to see the actual changes
3. Review recent commits to understand the commit message style
4. Verify there are actual changes to commit (no empty commits)

---

## 4. Commit Message Generation

### Attribution System

Attribution text is generated by `/home/claudeuser/ClaudeCode/utils/attribution.ts`:

```typescript
export type AttributionTexts = {
  commit: string  // Co-Authored-By line for commit
  pr: string      // Attribution for PR description
}

export function getAttributionTexts(): AttributionTexts {
  // Returns different attribution based on:
  // - Remote mode (session URL)
  // - Undercover mode (empty for public repos)
  // - User settings (custom attribution)
  // - Default (model name as Co-Authored-By)
}
```

### Default Commit Attribution

```typescript
const defaultAttribution = `🤖 Generated with [Claude Code](${PRODUCT_URL})`
const defaultCommit = `Co-Authored-By: ${modelName} <noreply@anthropic.com>`
```

### Undercover Mode

When contributing to public/open-source repos, `isUndercover()` returns true:

```typescript
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true
    // Auto: active unless in internal repo
    return getRepoClassCached() !== 'internal'
  }
  return false
}
```

Undercover mode:
- Strips all attribution from commits and PRs
- Removes model codenames and version numbers
- Hides any mention of "Claude Code"
- Writes commit messages like a human developer

---

## 5. Commit Execution

### HEREDOC Syntax for Commit Messages

The prompt instructs the model to use HEREDOC syntax:

```bash
git commit -m "$(cat <<'EOF'
Commit message here.
EOF
)"
```

This allows multi-line commit messages with proper attribution.

### Non-Interactive Operation

The prompt explicitly disables interactive operations:
- No `-i` flag on git commands
- No interactive rebase
- Hooks are NOT skipped by default (`--no-verify` is forbidden)
- Only runs git commands that work in non-interactive mode

---

## 6. /commit-push-pr Extended Flow

### Additional Allowed Tools

```typescript
const ALLOWED_TOOLS = [
  'Bash(git checkout --branch:*)',
  'Bash(git checkout -b:*)',
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git push:*)',
  'Bash(git commit:*)',
  'Bash(gh pr create:*)',
  'Bash(gh pr edit:*)',
  'Bash(gh pr view:*)',
  'Bash(gh pr merge:*)',
  'ToolSearch',
  // ... Slack tools
]
```

### PR Creation Flow

The prompt includes a multi-step workflow:

```markdown
## Your task

Analyze all changes that will be included in the pull request:

1. Create a new branch if on ${defaultBranch} (use SAFEUSER for branch name prefix)
2. Create a single commit with appropriate message using heredoc syntax
3. Push the branch to origin
4. If a PR already exists for this branch, update it with `gh pr edit`.
   Otherwise, create a PR with `gh pr create`
   - Keep PR titles short (under 70 characters)
   - Use body for details
```

### Branch Naming

```typescript
// Branch naming uses SAFEUSER or whoami as prefix
e.g., \`username/feature-name\`
```

### Enhanced PR Attribution

```typescript
export async function getEnhancedPRAttribution(
  getAppState: () => AppState,
): Promise<string> {
  // Calculates:
  // - Claude contribution percentage
  // - N-shotted count (user prompts)
  // - Model name
  // - Memory access count

  // Returns: "🤖 Generated with Claude Code (93% 3-shotted by claude-opus-4-5, 2 memories recalled)"
}
```

---

## 7. Git Utilities

### getDefaultBranch

Located in `/home/claudeuser/ClaudeCode/utils/git/gitFilesystem.ts`:

```typescript
async function computeDefaultBranch(): Promise<string> {
  const gitDir = await resolveGitDir()
  if (!gitDir) {
    return 'main'
  }
  // refs/remotes/ lives in commonDir
  const commonDir = (await getCommonDir(gitDir)) ?? gitDir

  // Try to read from origin/HEAD symref
  const branchFromSymref = await readRawSymref(
    commonDir,
    'refs/remotes/origin/HEAD',
    'refs/remotes/origin/',
  )
  if (branchFromSymref) {
    return branchFromSymref
  }

  // Fallback to checking main/master
  for (const candidate of ['main', 'master']) {
    const sha = await resolveRef(commonDir, `refs/remotes/origin/${candidate}`)
    if (sha) {
      return candidate
    }
  }
  return 'main'
}
```

### Git File Watcher

The `GitFileWatcher` class caches git state and watches for changes:

```typescript
class GitFileWatcher {
  // Watches:
  // - .git/HEAD (branch switches, detached HEAD)
  // - .git/config (remote URL changes)
  // - .git/refs/heads/<branch> (new commits)

  async get<T>(key: string, compute: () => Promise<T>): Promise<T> {
    // Returns cached value or recomputes if dirty
  }
}
```

### Cached Values

```typescript
export function getCachedBranch(): Promise<string>
export function getCachedHead(): Promise<string>
export function getCachedRemoteUrl(): Promise<string | null>
export function getCachedDefaultBranch(): Promise<string>
```

---

## 8. Error Handling

### Shell Command Errors

In `/home/claudeuser/ClaudeCode/utils/promptShellExecution.ts`:

```typescript
function formatBashError(e: unknown, pattern: string, inline = false): never {
  if (e instanceof ShellError) {
    if (e.interrupted) {
      throw new MalformedCommandError(
        `Shell command interrupted for pattern "${pattern}": [Command interrupted]`,
      )
    }
    const output = formatBashOutput(e.stdout, e.stderr, inline)
    throw new MalformedCommandError(
      `Shell command failed for pattern "${pattern}": ${output}`,
    )
  }
  // ...
}
```

### Permission Failures

```typescript
const permissionResult = await hasPermissionsToUseTool(...)
if (permissionResult.behavior !== 'allow') {
  throw new MalformedCommandError(
    `Shell command permission check failed for pattern "${match[0]}": ${permissionResult.message || 'Permission denied'}`,
  )
}
```

### Git Safety Errors

The prompt instructs the model to:
- Check for changes before committing (no empty commits)
- Skip committing files with likely secrets (.env, credentials.json)
- Warn users if they request destructive operations
- Never use `--no-verify` or `--no-gpg-sign` unless explicitly requested

---

## 9. Key Design Patterns

### 1. Prompt-Based Commands
Commands generate prompts that are sent to the model, which then decides which tools to use.

### 2. Tool Whitelisting
Allowed tools are explicitly declared via glob patterns, restricting what the model can execute.

### 3. Permission Context Injection
Commands modify the permission context to inject their allowed tool rules.

### 4. Shell Command Substitution
Inline shell commands (`!\`cmd\`` and `\`\`\`! cmd \`\`\``) provide dynamic context.

### 5. Attribution System
Flexible attribution that adapts to:
- Internal vs external repos
- Remote mode
- User preferences
- Undercover mode

### 6. Non-Interactive Operation
All operations must work without interactive input (no -i flags, no prompts).

### 7. Safety Protocols
Explicit safety rules embedded in prompts to guide model behavior.

---

## 10. Command Flow Summary

```
User types /commit
    ↓
Command.getPromptForCommand() called
    ↓
Build prompt with:
  - Git context (status, diff, log)
  - Safety protocols
  - Task instructions
    ↓
executeShellCommandsInPrompt() processes inline shell commands
    ↓
Prompt sent to model with tool whitelist injected
    ↓
Model analyzes changes and executes git commands
    ↓
HEREDOC commit created
    ↓
Result returned to user
```

For `/commit-push-pr`:
```
Additional steps:
  - Create feature branch
  - Push to origin
  - Check for existing PR (gh pr view)
  - Create or update PR (gh pr create/edit)
  - Optional: Post to Slack
```
