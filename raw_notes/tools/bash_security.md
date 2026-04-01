# Bash Tool Security Analysis

## Overview

The Bash Tool security system in Claude Code is a multi-layered defense mechanism designed to:
1. Parse and validate shell commands before execution
2. Detect potentially dangerous patterns that could bypass security
3. Enforce permission rules (allow/ask/deny) based on user preferences
4. Integrate with sandboxing for additional isolation
5. Protect against command injection and parser differential attacks

## File Structure

### 1. BashTool.tsx - Main Tool Implementation
**Location:** `/home/claudeuser/ClaudeCode/tools/BashTool/BashTool.tsx`

This is the primary tool definition that:
- Defines the tool schema (input/output)
- Handles command execution via `runShellCommand`
- Integrates with the permission system via `bashToolHasPermission`
- Manages background task execution
- Handles sandbox annotation and result processing
- Provides UI rendering for tool use messages

**Key Security-Relevant Code:**
```typescript
// Schema includes sandbox override option
const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string(),
  timeout: semanticNumber(z.number().optional()),
  description: z.string().optional(),
  run_in_background: semanticBoolean(z.boolean().optional()),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()), // Security-critical field
  _simulatedSedEdit: z.object({...}).optional(), // Internal only, excluded from model schema
}));
```

**Security Note:** The `_simulatedSedEdit` field is explicitly omitted from the model-facing schema to prevent the model from bypassing permission checks by pairing an innocuous command with arbitrary file write capabilities.

---

### 2. bashSecurity.ts - Core Security Validation
**Location:** `/home/claudeuser/ClaudeCode/tools/BashTool/bashSecurity.ts`

This file contains the primary security validation logic with ~20+ individual validators.

#### Security Check Categories

**A. Command Substitution & Expansion Detection**
- `COMMAND_SUBSTITUTION_PATTERNS`: Detects `$()`, `<()`, `>()`, `=${}`, Zsh equals expansion
- `HEREDOC_IN_SUBSTITUTION`: Special handling for `$(cat <<'EOF'...)` patterns
- `validateSafeCommandSubstitution`: Allows safe heredoc patterns only

**B. Zsh-Specific Protections**
```typescript
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',      // Can load dangerous modules (zsh/mapfile, zsh/system, zsh/zpty)
  'emulate',       // With -c flag is eval-equivalent
  'sysopen',       // Raw file I/O via zsh/system
  'sysread',
  'syswrite',
  'zpty',          // Pseudo-terminal execution
  'ztcp',          // Network exfiltration
  'zf_rm',         // Builtin rm from zsh/files
  // ... and more
])
```

**C. Parser Differential Defenses**
The system protects against differences between shell-quote parsing and actual bash execution:

1. **validateCarriageReturn**: CR (`\r`) is treated as word separator by shell-quote but as literal by bash
2. **validateBackslashEscapedOperators**: `\;` normalizes to `;` in splitCommand, enabling double-parse attacks
3. **validateBraceExpansion**: `{a,b}` expansion bypass - git diff `{--upload-pack="touch /tmp/pwned",test}`
4. **validateQuotedNewline**: Newlines inside quotes followed by `#` lines hide arguments from line-based processing
5. **validateCommentQuoteDesync**: Quote characters in `#` comments desync quote trackers
6. **validateMidWordHash**: Mid-word `#` treated as comment-start by shell-quote but literal by bash

**D. Flag Obfuscation Detection**
- **validateObfuscatedFlags**: Comprehensive detection of quote-based flag hiding
  - ANSI-C quoting: `$'...'`
  - Locale quoting: `$"..."`
  - Empty quotes before dash: `''-` or `""-`
  - Quote concatenation: `"-""exec"` → `-exec`
  - 3+ consecutive quotes at word start

**E. Git Commit Protection**
- `validateGitCommit`: Special handling for `git commit -m "..."`
  - Checks for command substitution in commit messages
  - Validates remainder for shell operators after `-m`
  - Blocks messages starting with dash (obfuscation)

**F. JQ Command Protection**
- `validateJqCommand`: Blocks dangerous jq flags
  - `system()` function calls
  - File-reading flags: `-f`, `--from-file`, `--rawfile`, `--slurpfile`, `-L`

**G. Environment Variable Protections**
- **validateIFSInjection**: Blocks `$IFS` and `${...IFS...}` patterns
- **validateProcEnvironAccess**: Blocks `/proc/*/environ` access
- **validateDangerousVariables**: Variables in redirections/pipes

**H. Control Character Protections**
- **CONTROL_CHAR_RE**: Blocks non-printable characters (0x00-0x08, etc.)
- **validateUnicodeWhitespace**: Blocks Unicode whitespace that shell-quote treats as separators

#### Validation Pipeline

```typescript
// Early validators (can return 'allow' to short-circuit)
const earlyValidators = [
  validateEmpty,
  validateIncompleteCommands,
  validateSafeCommandSubstitution,
  validateGitCommit,
]

// Main validators (in order of execution)
const validators = [
  validateJqCommand,
  validateObfuscatedFlags,
  validateShellMetacharacters,
  validateDangerousVariables,
  validateCommentQuoteDesync,
  validateQuotedNewline,
  validateCarriageReturn,  // Misparsing concern
  validateNewlines,        // Non-misparsing
  validateIFSInjection,
  validateProcEnvironAccess,
  validateDangerousPatterns,  // $(), backticks, etc.
  validateRedirections,    // Non-misparsing
  validateBackslashEscapedWhitespace,
  validateBackslashEscapedOperators,  // Misparsing
  validateUnicodeWhitespace,
  validateMidWordHash,
  validateBraceExpansion,  // Misparsing
  validateZshDangerousCommands,
  validateMalformedTokenInjection,
]
```

**Critical Security Design:** Misparsing validators set `isBashSecurityCheckForMisparsing: true`, causing early blocking at the permission gate, while non-misparsing validators flow through standard permission prompts.

---

### 3. bashPermissions.ts - Permission System Integration
**Location:** `/home/claudeuser/ClaudeCode/tools/BashTool/bashPermissions.ts`

This file integrates command validation with the permission system.

#### Permission Rule Types

```typescript
// Three rule types supported
type ShellPermissionRule =
  | { type: 'exact'; command: string }
  | { type: 'prefix'; prefix: string }
  | { type: 'wildcard'; pattern: string }
```

#### Security-Critical Environment Variables

**SAFE_ENV_VARS** (can be stripped for matching):
- Go: `GOEXPERIMENT`, `GOOS`, `GOARCH`, `CGO_ENABLED`, `GO111MODULE`
- Rust: `RUST_BACKTRACE`, `RUST_LOG`
- Node: `NODE_ENV`
- Python: `PYTHONUNBUFFERED`, `PYTHONDONTWRITEBYTECODE`
- Locale: `LANG`, `LANGUAGE`, `LC_ALL`, `LC_CTYPE`, etc.
- Terminal: `TERM`, `COLORTERM`, `NO_COLOR`, `TZ`

**ANT_ONLY_SAFE_ENV_VARS** (internal use only):
- `KUBECONFIG`, `DOCKER_HOST`, `AWS_PROFILE`
- Various cluster selection vars
- Feature flags like `CI`, `SKIP_NODE_VERSION_CHECK`
- Credentials like `PGPASSWORD`, `GH_TOKEN`

**Security Note:** `ANT_ONLY_SAFE_ENV_VARS` is intentionally gated to `USER_TYPE === 'ant'` and must never ship to external users. Stripping `DOCKER_HOST` defeats prefix-based permission restrictions.

#### Wrapper Command Stripping

**SAFE_WRAPPER_PATTERNS** (stripped for permission matching):
```typescript
const SAFE_WRAPPER_PATTERNS = [
  /^timeout[ 	]+.../,   // With flag enumeration for GNU timeout
  /^time[ 	]+(?:--[ 	]+)?/,
  /^nice(?:[ 	]+-n[ 	]+-?\d+|[ 	]+-\d+)?[ 	]+(?:--[ 	]+)?/,
  /^stdbuf(?:[ 	]+-[ioe][LN0-9]+)+[ 	]+(?:--[ 	]+)?/,
  /^nohup[ 	]+(?:--[ 	]+)?/,
]
```

**Security Design:** Wrapper stripping is iterative (fixed-point) to handle interleaved patterns like `nohup FOO=bar timeout 5 claude`.

#### Permission Checking Flow

```typescript
export async function bashToolHasPermission(input, context): Promise<PermissionResult> {
  // 1. AST-based security parse (tree-sitter)
  //    - Returns 'simple' commands, 'too-complex', or 'parse-unavailable'
  //    - 'too-complex' respects deny rules then falls to ask

  // 2. Sandbox auto-allow check
  //    - If sandboxed AND auto-allow enabled AND no deny/ask rules

  // 3. Exact match check
  //    - Check exact command against allow/ask/deny rules

  // 4. Classifier checks (Haiku)
  //    - Parallel deny/ask classification
  //    - Deny takes precedence

  // 5. Command operator permissions
  //    - Handle pipes, redirections, compound commands

  // 6. Legacy misparsing gate
  //    - Run bashCommandIsSafeAsync when tree-sitter unavailable

  // 7. Subcommand processing
  //    - Split compound commands
  //    - Check each subcommand
  //    - Validate output redirections

  // 8. Permission decision
  //    - Deny if any subcommand denied
  //    - Ask if any subcommand asks
  //    - Allow if all allowed and no injection detected
}
```

#### Compound Command Security

**CD + Git Attack Prevention:**
```typescript
// Block compound commands with both cd AND git
// Prevents: cd /malicious/dir && git status
// where malicious directory contains bare git repo with core.fsmonitor
if (compoundCommandHasCd) {
  const hasGitCommand = subcommands.some(cmd => isNormalizedGitCommand(cmd.trim()))
  if (hasGitCommand) {
    return {
      behavior: 'ask',
      reason: 'Compound commands with cd and git require approval to prevent bare repository attacks'
    }
  }
}
```

#### Subcommand Fanout Protection

```typescript
// CC-643: Cap subcommands to prevent ReDoS and event loop starvation
export const MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50
export const MAX_SUGGESTED_RULES_FOR_COMPOUND = 5
```

---

### 4. destructiveCommandWarning.ts - Destructive Operation Detection
**Location:** `/home/claudeuser/ClaudeCode/tools/BashTool/destructiveCommandWarning.ts`

**Purpose:** Purely informational - displays warning strings in permission dialogs without affecting permission logic.

**Destructive Patterns Detected:**

| Pattern | Warning |
|---------|---------|
| `git reset --hard` | "Note: may discard uncommitted changes" |
| `git push --force` | "Note: may overwrite remote history" |
| `git clean -f` (not dry-run) | "Note: may permanently delete untracked files" |
| `git checkout .` | "Note: may discard all working tree changes" |
| `git stash drop/clear` | "Note: may permanently remove stashed changes" |
| `git branch -D` | "Note: may force-delete a branch" |
| `git commit/push/merge --no-verify` | "Note: may skip safety hooks" |
| `git commit --amend` | "Note: may rewrite the last commit" |
| `rm -rf` | "Note: may recursively force-remove files" |
| `DROP/TRUNCATE TABLE/DATABASE` | "Note: may drop or truncate database objects" |
| `kubectl delete` | "Note: may delete Kubernetes resources" |
| `terraform destroy` | "Note: may destroy Terraform infrastructure" |

---

### 5. shouldUseSandbox.ts - Sandbox Decision Logic
**Location:** `/home/claudeuser/ClaudeCode/tools/BashTool/shouldUseSandbox.ts`

**Decision Flow:**

```typescript
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  // 1. Check if sandboxing is enabled globally
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }

  // 2. Check explicit override (dangerouslyDisableSandbox)
  //    Only honored if unsandboxed commands allowed by policy
  if (input.dangerouslyDisableSandbox && SandboxManager.areUnsandboxedCommandsAllowed()) {
    return false
  }

  // 3. Check user-configured excluded commands
  //    Note: This is a convenience feature, NOT a security boundary
  if (containsExcludedCommand(input.command)) {
    return false
  }

  return true
}
```

**Excluded Commands Logic:**

Commands can be excluded from sandboxing via:
1. **Dynamic config** (ant users only): `tengu_sandbox_disabled_commands` feature flag
2. **User settings**: `sandbox.excludedCommands` in settings.json

**Pattern matching for exclusions:**
- Exact matches: `"npm run lint"`
- Prefix patterns: `"npm run test:*"`
- Wildcard patterns: `"npm run *test*"`

**Security Note:** The code explicitly documents: "excludedCommands is a user-facing convenience feature, not a security boundary. It is not a security bug to be able to bypass excludedCommands — the sandbox permission system (which prompts users) is the actual security control."

---

## Security Architecture Summary

### Defense Layers (Outside to Inside)

1. **Permission Mode** (`auto`/`plan`/`bypassPermissions`)
   - `auto`: Auto-approve with classifier
   - `plan`: User must approve each command
   - `bypassPermissions`: Skip all permission checks (dangerous)

2. **Explicit Rules** (deny/ask/allow)
   - Deny rules take highest precedence
   - Ask rules require user confirmation
   - Allow rules auto-approve matching commands

3. **Sandbox Auto-Allow**
   - When sandboxed AND auto-allow enabled
   - Respects explicit deny/ask rules
   - Blocks excluded commands

4. **Command Classification** (Haiku)
   - AI-based classification of command intent
   - Parallel deny/ask checks
   - High-confidence matches auto-decide

5. **Security Validators** (bashSecurity.ts)
   - Parser differential detection
   - Command substitution blocking
   - Flag obfuscation detection
   - Zsh dangerous command blocking

6. **Path Constraints**
   - Validates file paths in commands
   - Blocks access outside project boundaries
   - Validates output redirections

7. **Sandbox Execution**
   - Landlock/pledge/seccomp-based sandboxing
   - Restricts filesystem access
   - Network access controls

### Permission Result Types

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput; decisionReason }
  | { behavior: 'ask'; message; decisionReason; suggestions?; pendingClassifierCheck? }
  | { behavior: 'deny'; message; decisionReason }
  | { behavior: 'passthrough'; message? }  // Continue to next check
```

### Key Security Principles

1. **Fail-Closed**: When uncertain, ask the user (return 'ask')
2. **Deny Precedence**: Deny rules always take priority over allow rules
3. **Defense in Depth**: Multiple overlapping security checks
4. **Parser Differential Awareness**: Extensive protections against shell-quote vs bash differences
5. **User Control**: Users can set explicit allow/deny rules for fine-grained control
6. **Transparency**: Destructive commands show warnings, permission reasons are explained

## Known Attack Vectors Addressed

1. **Command Injection via Substitution**: `$()`, ``, `${}` detection
2. **Flag Obfuscation**: Quote-based hiding of dangerous flags
3. **Parser Differentials**: CR, backslash-escaped operators, brace expansion
4. **Zsh Module Loading**: zmodload and related builtins
5. **Heredoc Injection**: Careful validation of `$(cat <<'EOF')` patterns
6. **Git-Based Attacks**: Bare repository RCE via core.fsmonitor
7. **Path Traversal**: Comprehensive path validation
8. **Control Character Injection**: Non-printable character blocking
