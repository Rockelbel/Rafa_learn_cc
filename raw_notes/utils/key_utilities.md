# Claude Code Utility Modules Analysis

An analysis of five key utility modules in the Claude Code codebase, focusing on patterns, error handling, configuration management, and logging strategies.

---

## 1. Authentication (`utils/auth.ts`)

### Overview
A comprehensive authentication module handling multiple auth sources: OAuth tokens, API keys, AWS/GCP credentials, and secure storage integration.

### Key Patterns

#### **Hierarchical Auth Source Resolution**
```typescript
// Auth sources are checked in priority order:
// 1. Bare mode (API-key-only, hermetic)
// 2. Environment variables (ANTHROPIC_AUTH_TOKEN, CLAUDE_CODE_OAUTH_TOKEN)
// 3. File descriptor tokens (for CCR/subprocess scenarios)
// 4. apiKeyHelper (user-configured script)
// 5. macOS Keychain / Config file
// 6. Claude.ai OAuth tokens
```

#### **Sync/Async Dual APIs with Memoization**
```typescript
// Sync version for hot paths (memoized)
export const getClaudeAIOAuthTokens = memoize((): OAuthTokens | null => { ... })

// Async version for non-blocking keychain reads
export async function getClaudeAIOAuthTokensAsync(): Promise<OAuthTokens | null> { ... }
```

#### **SWR (Stale-While-Revalidate) Caching Pattern**
```typescript
// For apiKeyHelper - returns stale value immediately, refreshes in background
if (_apiKeyHelperCache) {
  if (Date.now() - _apiKeyHelperCache.timestamp < ttl) {
    return _apiKeyHelperCache.value
  }
  // Stale — return stale value now, refresh in the background
  if (!_apiKeyHelperInflight) {
    _apiKeyHelperInflight = { promise: _runAndCache(...), startedAt: null }
  }
  return _apiKeyHelperCache.value
}
```

#### **Workspace Trust Security Guard**
```typescript
// All project-configured commands check trust before execution
if (isApiKeyHelperFromProjectOrLocalSettings()) {
  const hasTrust = checkHasTrustDialogAccepted()
  if (!hasTrust && !isNonInteractiveSession) {
    // Log security event and refuse execution
    logEvent('tengu_apiKeyHelper_missing_trust11', {})
    return null
  }
}
```

#### **Epoch-Based Cache Invalidation**
```typescript
let _apiKeyHelperEpoch = 0

export function clearApiKeyHelperCache(): void {
  _apiKeyHelperEpoch++
  _apiKeyHelperCache = null
  _apiKeyHelperInflight = null
}

// In-flight operations check epoch before touching state
if (epoch !== _apiKeyHelperEpoch) return value
```

### Reusable Approaches

1. **Multi-source auth with clear precedence** - Each source is checked in order, with early returns
2. **Managed vs unmanaged context detection** - `isManagedOAuthContext()` prevents settings leakage into CCR/Desktop sessions
3. **Token refresh with deduplication** - `pending401Handlers` Map prevents concurrent refresh storms
4. **Cross-process cache invalidation** - File mtime watching for credential file changes

---

## 2. Configuration (`utils/config.ts`)

### Overview
Sophisticated configuration management with file watching, atomic writes, corruption recovery, and project-scoped settings.

### Key Patterns

#### **Two-Tier Config Structure**
```typescript
type GlobalConfig = { /* user-level settings */ }
type ProjectConfig = { /* per-project settings like allowedTools, MCP servers */ }

// Projects keyed by normalized path
projects?: Record<string, ProjectConfig>
```

#### **Factory Function for Defaults**
```typescript
// Returns fresh objects (no deep clone cost for empty containers)
function createDefaultGlobalConfig(): GlobalConfig {
  return {
    numStartups: 0,
    theme: 'dark',
    // ... all defaults inline
  }
}
```

#### **Write-Through Caching with Freshness Watcher**
```typescript
let globalConfigCache: { config: GlobalConfig | null; mtime: number }

// Fast path: pure memory read
if (globalConfigCache.config) return globalConfigCache.config

// Background watcher detects external writes
watchFile(file, { interval: CONFIG_FRESHNESS_POLL_MS }, curr => {
  if (curr.mtimeMs <= globalConfigCache.mtime) return
  // Re-read and update cache
})
```

#### **Atomic Write with Lockfile**
```typescript
function saveConfigWithLock<A>(file: string, createDefault: () => A, mergeFn: (current: A) => A): boolean {
  const release = lockfile.lockSync(file, { lockfilePath: `${file}.lock` })
  try {
    // Re-read after acquiring lock (check for stale write)
    const currentConfig = getConfig(file, createDefault)
    // Auth-loss guard: refuse to write if we'd lose OAuth/onboarding state
    if (wouldLoseAuthState(currentConfig)) return false
    // Write with backup creation
  } finally {
    release()
  }
}
```

#### **Corruption Recovery with Backup**
```typescript
catch (error) {
  if (error instanceof ConfigParseError) {
    // Backup corrupted file
    const corruptedBackupPath = join(backupDir, `${fileBase}.corrupted.${Date.now()}`)
    fs.copyFileSync(file, corruptedBackupPath)
    // Notify user about backup location
    process.stderr.write(`Config corrupted, backed up to: ${corruptedBackupPath}`)
    return createDefault()
  }
}
```

#### **Trust Inheritance (Directory Traversal)**
```typescript
export function checkHasTrustDialogAccepted(): boolean {
  // Check from cwd up to root
  let currentPath = normalizePathForConfigKey(getCwd())
  while (true) {
    if (config.projects?.[currentPath]?.hasTrustDialogAccepted) return true
    const parentPath = normalizePathForConfigKey(resolve(currentPath, '..'))
    if (parentPath === currentPath) break
    currentPath = parentPath
  }
  return false
}
```

### Reusable Approaches

1. **Config guard prevents early access** - `configReadingAllowed` flag throws if modules read config during init
2. **Auth-loss prevention** - Compare fresh read against cached state before writing
3. **Backup strategy** - Timestamped backups, deduplication (60s interval), cleanup (keep 5)
4. **Migration pattern** - `migrateConfigFields()` handles field renames/transforms
5. **Re-entrancy guard** - `insideGetConfig` prevents `logEvent` -> `getGlobalConfig` recursion

---

## 3. Error Handling (`utils/errors.ts`)

### Overview
Comprehensive error utilities with custom error classes, type guards, and safe extraction functions.

### Key Patterns

#### **Custom Error Hierarchy**
```typescript
export class ClaudeError extends Error {
  constructor(message: string) {
    super(message)
    this.name = this.constructor.name  // Auto-set name from class
  }
}

export class ShellError extends Error {
  constructor(
    public readonly stdout: string,
    public readonly stderr: string,
    public readonly code: number,
    public readonly interrupted: boolean,
  ) { super('Shell command failed') }
}
```

#### **Telemetry-Safe Error (Explicit Verification)**
```typescript
// Long intentional name forces code review consideration
export class TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS extends Error {
  readonly telemetryMessage: string
  constructor(message: string, telemetryMessage?: string) {
    super(message)
    this.name = 'TelemetrySafeError'
    this.telemetryMessage = telemetryMessage ?? message
  }
}
```

#### **Type Guards for Unknown Errors**
```typescript
// Normalize unknown to Error
export function toError(e: unknown): Error {
  return e instanceof Error ? e : new Error(String(e))
}

// Safe message extraction
export function errorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e)
}

// Errno code extraction without casting
export function getErrnoCode(e: unknown): string | undefined {
  if (e && typeof e === 'object' && 'code' in e && typeof e.code === 'string') {
    return e.code
  }
  return undefined
}
```

#### **Abort Error Detection (Multi-Source)**
```typescript
export function isAbortError(e: unknown): boolean {
  return (
    e instanceof AbortError ||
    e instanceof APIUserAbortError ||
    (e instanceof Error && e.name === 'AbortError')
  )
}
// Note: Uses instanceof for SDK errors because minified builds mangle names
```

#### **Axios Error Classification**
```typescript
export type AxiosErrorKind = 'auth' | 'timeout' | 'network' | 'http' | 'other'

export function classifyAxiosError(e: unknown): {
  kind: AxiosErrorKind
  status?: number
  message: string
} {
  // Dependency-free check (looks for isAxiosError marker)
  if (!e || typeof e !== 'object' || !('isAxiosError' in e)) {
    return { kind: 'other', message: errorMessage(e) }
  }
  // Classify by status/code
  if (status === 401 || status === 403) return { kind: 'auth', ... }
  if (err.code === 'ECONNABORTED') return { kind: 'timeout', ... }
}
```

#### **Stack Trace Truncation (Token Conservation)**
```typescript
export function shortErrorStack(e: unknown, maxFrames = 5): string {
  if (!(e instanceof Error)) return String(e)
  if (!e.stack) return e.message
  const lines = e.stack.split('\n')
  const header = lines[0] ?? e.message
  const frames = lines.slice(1).filter(l => l.trim().startsWith('at '))
  if (frames.length <= maxFrames) return e.stack
  return [header, ...frames.slice(0, maxFrames)].join('\n')
}
```

### Reusable Approaches

1. **Explicit verification naming** - Long descriptive names force security review
2. **Dependency-free type detection** - Check marker properties instead of using `instanceof` for external libs
3. **Consolidated error classification** - Single function replaces ~20-line switch chains
4. **Safe property access** - Type-check before accessing `code`, `path` on unknown errors

---

## 4. Logging (`utils/log.ts`)

### Overview
Multi-destination logging with sink pattern, queued events, privacy controls, and structured log loading.

### Key Patterns

#### **Sink Pattern with Queue Drainage**
```typescript
export type ErrorLogSink = {
  logError: (error: Error) => void
  logMCPError: (serverName: string, error: unknown) => void
  logMCPDebug: (serverName: string, message: string) => void
  getErrorsPath: () => string
  getMCPLogsPath: (serverName: string) => string
}

let errorLogSink: ErrorLogSink | null = null
const errorQueue: QueuedErrorEvent[] = []

export function attachErrorLogSink(newSink: ErrorLogSink): void {
  if (errorLogSink !== null) return  // Idempotent
  errorLogSink = newSink
  // Drain queued events immediately
  if (errorQueue.length > 0) {
    const queuedEvents = [...errorQueue]
    errorQueue.length = 0
    for (const event of queuedEvents) { ... }
  }
}
```

#### **Privacy-Aware Error Logging**
```typescript
export function logError(error: unknown): void {
  // Check if error reporting should be disabled
  if (
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    process.env.DISABLE_ERROR_REPORTING ||
    isEssentialTrafficOnly()
  ) {
    return
  }
  // ... log to multiple destinations
}
```

#### **In-Memory Error Ring Buffer**
```typescript
const MAX_IN_MEMORY_ERRORS = 100
let inMemoryErrorLog: Array<{ error: string; timestamp: string }> = []

function addToInMemoryErrorLog(errorInfo: { error: string; timestamp: string }): void {
  if (inMemoryErrorLog.length >= MAX_IN_MEMORY_ERRORS) {
    inMemoryErrorLog.shift()  // Remove oldest
  }
  inMemoryErrorLog.push(errorInfo)
}
```

#### **API Request Capture (Selective)**
```typescript
export function captureAPIRequest(params: BetaMessageStreamParams, querySource?: QuerySource): void {
  // Only capture for main thread REPL sessions
  if (!querySource || !querySource.startsWith('repl_main_thread')) return

  // Store params WITHOUT messages to avoid retaining entire conversation
  const { messages, ...paramsWithoutMessages } = params
  setLastAPIRequest(paramsWithoutMessages)

  // Full messages only for internal 'ant' users
  setLastAPIRequestMessages(process.env.USER_TYPE === 'ant' ? messages : null)
}
```

#### **Hard Fail Mode (Testing)**
```typescript
const isHardFailMode = memoize((): boolean => process.argv.includes('--hard-fail'))

export function logError(error: unknown): void {
  if (feature('HARD_FAIL') && isHardFailMode()) {
    console.error('[HARD FAIL] logError called with:', err.stack || err.message)
    process.exit(1)
  }
  // ... normal logging
}
```

### Reusable Approaches

1. **Sink pattern decouples initialization** - Events can be logged before sink is attached
2. **Circular buffer for recent errors** - Bounded memory usage, useful for bug reports
3. **Environment-based disable** - Multiple flags can disable reporting (3P services, essential traffic)
4. **Selective data retention** - Different retention for different user types

---

## 5. Debugging (`utils/debug.ts`)

### Overview
Multi-level debug logging with filtering, buffering, file output, and runtime enablement.

### Key Patterns

#### **Leveled Logging with Filtering**
```typescript
export type DebugLogLevel = 'verbose' | 'debug' | 'info' | 'warn' | 'error'

const LEVEL_ORDER: Record<DebugLogLevel, number> = {
  verbose: 0, debug: 1, info: 2, warn: 3, error: 4
}

export function logForDebugging(message: string, { level }: { level: DebugLogLevel } = { level: 'debug' }) {
  if (LEVEL_ORDER[level] < LEVEL_ORDER[getMinDebugLogLevel()]) return
  if (!shouldLogDebugMessage(message)) return
  // ... write output
}
```

#### **Buffered vs Immediate Mode**
```typescript
function getDebugWriter(): BufferedWriter {
  if (!debugWriter) {
    debugWriter = createBufferedWriter({
      writeFn: content => {
        if (isDebugMode()) {
          // immediateMode: sync write for --debug
          getFsImplementation().appendFileSync(path, content)
        } else {
          // Buffered path: async write chain for background logging
          pendingWrite = pendingWrite.then(appendAsync.bind(null, ...)).catch(noop)
        }
      },
      flushIntervalMs: 1000,
      maxBufferSize: 100,
      immediateMode: isDebugMode(),
    })
  }
}
```

#### **Debug Filter Pattern Matching**
```typescript
// Support --debug=pattern syntax
export const getDebugFilter = memoize((): DebugFilter | null => {
  const debugArg = process.argv.find(arg => arg.startsWith('--debug='))
  if (!debugArg) return null
  const filterPattern = debugArg.substring('--debug='.length)
  return parseDebugFilter(filterPattern)
})

// Usage in shouldLogDebugMessage
const filter = getDebugFilter()
return shouldShowDebugMessage(message, filter)
```

#### **Runtime Enablement**
```typescript
let runtimeDebugEnabled = false

export function enableDebugLogging(): boolean {
  const wasActive = isDebugMode() || process.env.USER_TYPE === 'ant'
  runtimeDebugEnabled = true
  isDebugMode.cache.clear?.()
  return wasActive
}
// Called via /debug command mid-session
```

#### **User-Specific Logging (Ant-Only)**
```typescript
export function logAntError(context: string, error: unknown): void {
  if (process.env.USER_TYPE !== 'ant') return
  if (error instanceof Error && error.stack) {
    logForDebugging(`[ANT-ONLY] ${context} stack trace:\n${error.stack}`, { level: 'error' })
  }
}
```

#### **Symlink to Latest Log**
```typescript
const updateLatestDebugLogSymlink = memoize(async (): Promise<void> => {
  try {
    const debugLogPath = getDebugLogPath()
    const debugLogsDir = dirname(debugLogPath)
    const latestSymlinkPath = join(debugLogsDir, 'latest')
    await unlink(latestSymlinkPath).catch(() => {})
    await symlink(debugLogPath, latestSymlinkPath)
  } catch {
    // Silently fail if symlink creation fails
  }
})
```

### Reusable Approaches

1. **Dual-mode writing** - Sync for interactive debug, async buffered for background
2. **Pattern-based filtering** - Users can filter with `--debug=auth,shell`
3. **Memoized one-time setup** - Symlink update memoized to run once per session
4. **Silent failures** - Debug logging should never crash the application

---

## Cross-Module Patterns Summary

### **Memoization Strategy**
All modules use `lodash-es/memoize.js` for caching:
- **auth.ts**: `getClaudeAIOAuthTokens`, `isDebugMode`, `getDebugFilter`
- **config.ts**: `getProjectPathForConfig`
- **debug.ts**: `getMinDebugLogLevel`, `isDebugMode`, `getDebugFilter`

Cache invalidation uses `.cache.clear?.()` pattern with optional chaining.

### **Error Handling Philosophy**
1. **Fail safe, not loud** - Most logging errors are caught and silently ignored
2. **Explicit verification** - Security-sensitive errors have long descriptive names
3. **Type guards over casting** - `getErrnoCode(e)` instead of `(e as NodeJS.ErrnoException).code`

### **Configuration Safety**
1. **Guard against auth loss** - Config writes check wouldLoseAuthState before proceeding
2. **Lockfile protection** - Cross-process safety for config writes
3. **Backup strategy** - Automatic corrupted file backup with timestamps

### **Privacy & Compliance**
1. **Multiple disable switches** - Error reporting can be disabled via env vars
2. **3P service detection** - Bedrock/Vertex/Foundry automatically disable features
3. **Selective data retention** - Different data kept for internal vs external users

### **Performance Patterns**
1. **SWR caching** - Return stale immediately, refresh in background
2. **Epoch invalidation** - Bumped on clear, checked by in-flight operations
3. **Write-through caching** - Cache updated immediately on write, not re-read

---

## Files Analyzed

| File | Lines | Purpose |
|------|-------|---------|
| `/home/claudeuser/ClaudeCode/utils/auth.ts` | 2002 | Authentication, OAuth, API keys, AWS/GCP creds |
| `/home/claudeuser/ClaudeCode/utils/config.ts` | 1818 | Global/project config, trust management, migrations |
| `/home/claudeuser/ClaudeCode/utils/errors.ts` | 239 | Custom errors, type guards, error classification |
| `/home/claudeuser/ClaudeCode/utils/log.ts` | 363 | Error logging, API capture, in-memory ring buffer |
| `/home/claudeuser/ClaudeCode/utils/debug.ts` | 269 | Debug logging, filtering, buffered output |
