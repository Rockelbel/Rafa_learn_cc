# Claude Code File Tools Analysis

## Overview

This document analyzes five core file tools in the Claude Code codebase:
1. **FileReadTool** (`FileReadTool.ts`)
2. **FileEditTool** (`FileEditTool.ts`)
3. **FileWriteTool** (`FileWriteTool.ts`)
4. **GlobTool** (`GlobTool.ts`)
5. **GrepTool** (`GrepTool.ts`)

## Architecture: The Tool Interface

All tools implement the `Tool` interface defined in `/home/claudeuser/ClaudeCode/Tool.ts`. The `ToolDef` type provides a partial definition that `buildTool()` fills with defaults.

### Core Tool Structure

```typescript
export type Tool<Input extends AnyObject, Output, P extends ToolProgressData> = {
  name: string
  call(args: z.infer<Input>, context: ToolUseContext, ...): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  inputSchema: Input  // Zod schema
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number  // Size limit before persisting to disk
  strict?: boolean  // Enables strict API mode

  // Lifecycle methods
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  backfillObservableInput?(input): void
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>

  // Classification and metadata
  toAutoClassifierInput(input): unknown
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }

  // UI rendering
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseProgressMessage?(...): React.ReactNode
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  userFacingName(input): string

  // Result mapping
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  extractSearchText?(out): string
}
```

## Schema Design with Zod

All tools use Zod v4 for input/output validation with `lazySchema()` for lazy initialization.

### Pattern: Lazy Schema Initialization

```typescript
import { lazySchema } from '../../utils/lazySchema.js'

const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file'),
    content: z.string().describe('The content to write'),
  })
)
```

### Key Schema Features

1. **`z.strictObject()`**: Prevents extra properties (strict validation)
2. **`semanticNumber()`**: Wrapper for numbers with semantic boolean handling (accepts "5" or 5)
3. **`semanticBoolean()`**: Wrapper for booleans that handles semantic strings

Example from GrepTool:

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The regex pattern to search for'),
    path: z.string().optional().describe('Directory to search in'),
    glob: z.string().optional().describe('Glob pattern to filter files'),
    output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    head_limit: semanticNumber(z.number().optional()).describe('Limit output to first N lines'),
    offset: semanticNumber(z.number().optional()).describe('Skip first N lines'),
    multiline: semanticBoolean(z.boolean().optional()).describe('Enable multiline mode'),
  })
)
```

## Tool Factory: buildTool

The `buildTool()` function in `Tool.ts` provides fail-closed defaults:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,  // Fail-closed: assume not safe
  isReadOnly: (_input?) => false,         // Fail-closed: assume writes
  isDestructive: (_input?) => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',        // Skip classifier by default
  userFacingName: () => '',
}
```

Tools are defined using `satisfies ToolDef<Input, Output>` for type safety:

```typescript
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  searchHint: 'find files by name pattern or wildcard',
  maxResultSizeChars: 100_000,
  // ... tool implementation
} satisfies ToolDef<InputSchema, Output>)
```

## Input Validation Patterns

### ValidationResult Type

```typescript
export type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number; behavior?: 'ask' }
```

### Pre-I/O Validation (No Filesystem Access)

Tools perform cheap validation before expensive filesystem operations:

**FileReadTool** (lines 418-495):
```typescript
async validateInput({ file_path, pages }, toolUseContext) {
  // 1. Validate pages parameter (pure string parsing)
  if (pages !== undefined) {
    const parsed = parsePDFPageRange(pages)
    if (!parsed) {
      return { result: false, message: "Invalid pages parameter", errorCode: 7 }
    }
  }

  // 2. Path expansion + deny rule check (no I/O)
  const fullFilePath = expandPath(file_path)
  const denyRule = matchingRuleForInput(fullFilePath, appState.toolPermissionContext, 'read', 'deny')
  if (denyRule !== null) {
    return { result: false, message: "File is in a denied directory", errorCode: 1 }
  }

  // 3. UNC path check (no I/O) - defer filesystem operations
  const isUncPath = fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')
  if (isUncPath) {
    return { result: true }
  }

  // 4. Binary extension check (string check only)
  const ext = path.extname(fullFilePath).toLowerCase()
  if (hasBinaryExtension(fullFilePath) && !isPDFExtension(ext)) {
    return { result: false, message: "Cannot read binary files", errorCode: 4 }
  }

  // 5. Block device files that would hang
  if (isBlockedDevicePath(fullFilePath)) {
    return { result: false, message: "Cannot read device file", errorCode: 9 }
  }

  return { result: true }
}
```

### Post-I/O Validation (After Filesystem Check)

**FileEditTool** validates file existence, content matching, and staleness:

```typescript
async validateInput(input: FileEditInput, toolUseContext: ToolUseContext) {
  const { file_path, old_string, new_string } = input
  const fullFilePath = expandPath(file_path)

  // Check deny rules first (no I/O)
  const denyRule = matchingRuleForInput(fullFilePath, appState.toolPermissionContext, 'edit', 'deny')
  if (denyRule !== null) {
    return { result: false, message: "File is denied by permission settings", errorCode: 2 }
  }

  // UNC path security check
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
    return { result: true }
  }

  // Check file size limit (prevents OOM)
  const { size } = await fs.stat(fullFilePath)
  if (size > MAX_EDIT_FILE_SIZE) {
    return { result: false, message: `File too large (${formatFileSize(size)})`, errorCode: 10 }
  }

  // Read file content
  let fileContent: string | null
  try {
    const fileBuffer = await fs.readFileBytes(fullFilePath)
    const encoding = detectEncoding(fileBuffer)
    fileContent = fileBuffer.toString(encoding).replaceAll('\r\n', '\n')
  } catch (e) {
    if (isENOENT(e)) fileContent = null
    else throw e
  }

  // File doesn't exist
  if (fileContent === null) {
    if (old_string === '') return { result: true }  // Creating new file
    return { result: false, message: "File does not exist", errorCode: 4 }
  }

  // File exists but old_string is empty (creation attempt on existing file)
  if (old_string === '' && fileContent.trim() !== '') {
    return { result: false, message: "Cannot create - file already exists", errorCode: 3 }
  }

  // Check if file has been read first (prevents blind writes)
  const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
  if (!readTimestamp || readTimestamp.isPartialView) {
    return { result: false, message: "File has not been read yet. Read it first.", errorCode: 6 }
  }

  // Check for file modification since last read
  if (readTimestamp) {
    const lastWriteTime = getFileModificationTime(fullFilePath)
    if (lastWriteTime > readTimestamp.timestamp) {
      return { result: false, message: "File has been modified since read", errorCode: 7 }
    }
  }

  // Find the actual string to replace (handles quote normalization)
  const actualOldString = findActualString(file, old_string)
  if (!actualOldString) {
    return { result: false, message: "String to replace not found", errorCode: 8 }
  }

  // Check for multiple matches
  const matches = file.split(actualOldString).length - 1
  if (matches > 1 && !replace_all) {
    return { result: false, message: `Found ${matches} matches but replace_all is false`, errorCode: 9 }
  }

  return { result: true, meta: { actualOldString } }
}
```

## Permission Checking Patterns

### Permission Decision Flow

1. **validateInput()** - Cheap checks (no I/O), returns error codes
2. **checkPermissions()** - Permission system integration
3. **preparePermissionMatcher()** - Hook pattern matching

### Read Permission Check

**GlobTool** and **GrepTool** (read-only tools):
```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(
    GlobTool,
    input,
    appState.toolPermissionContext,
  )
}
```

### Write Permission Check

**FileEditTool** and **FileWriteTool**:
```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkWritePermissionForTool(
    FileEditTool,
    input,
    appState.toolPermissionContext,
  )
}
```

### Permission Matcher for Hooks

```typescript
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```

## Security Considerations

### 1. UNC Path Protection

All file tools skip filesystem operations for UNC paths to prevent NTLM credential leaks:

```typescript
// SECURITY: Skip filesystem operations for UNC paths to prevent NTLM credential leaks.
// On Windows, fs.existsSync() on UNC paths triggers SMB authentication which could
// leak credentials to malicious servers.
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }
}
```

### 2. Blocked Device Paths

FileReadTool blocks device files that would hang:

```typescript
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero', '/dev/random', '/dev/urandom', '/dev/full',
  '/dev/stdin', '/dev/tty', '/dev/console',
  '/dev/stdout', '/dev/stderr',
  '/dev/fd/0', '/dev/fd/1', '/dev/fd/2',
])

function isBlockedDevicePath(filePath: string): boolean {
  if (BLOCKED_DEVICE_PATHS.has(filePath)) return true
  if (filePath.startsWith('/proc/') &&
      (filePath.endsWith('/fd/0') || filePath.endsWith('/fd/1') || filePath.endsWith('/fd/2')))
    return true
  return false
}
```

### 3. File Size Limits

Prevents OOM from large files:

```typescript
// FileEditTool
const MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024 // 1 GiB

if (size > MAX_EDIT_FILE_SIZE) {
  return {
    result: false,
    message: `File is too large to edit (${formatFileSize(size)}). Maximum editable file size is ${formatFileSize(MAX_EDIT_FILE_SIZE)}.`,
    errorCode: 10,
  }
}
```

### 4. Path Expansion

`expandPath()` normalizes paths (handles `~`, relative paths, Windows separators):

```typescript
backfillObservableInput(input) {
  // hooks.mdx documents file_path as absolute; expand so hook allowlists
  // can't be bypassed via ~ or relative paths.
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

### 5. Secret Detection

Team memory files are checked for secrets:

```typescript
const secretError = checkTeamMemSecrets(fullFilePath, new_string)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

## Error Handling Patterns

### Error Codes

Each validation error has a unique error code for telemetry and debugging:

| Code | Tool | Meaning |
|------|------|---------|
| 0 | FileEdit/Write | Secret detected in team memory |
| 1 | FileEdit | No changes (old_string == new_string) |
| 1 | FileWrite | Denied by permission settings |
| 1 | Glob | Directory does not exist |
| 1 | Grep | Path does not exist |
| 2 | FileEdit | Denied by permission settings |
| 2 | FileWrite | File not read yet |
| 3 | FileEdit | Creating file that exists |
| 3 | FileWrite | File modified since read |
| 4 | FileEdit | File does not exist |
| 4 | FileRead | Binary file blocked |
| 5 | FileEdit | Jupyter Notebook (use NotebookEditTool) |
| 6 | FileEdit | File not read yet |
| 7 | FileEdit | File modified since read |
| 7 | FileRead | Invalid pages parameter |
| 8 | FileEdit | String not found in file |
| 8 | FileRead | Pages range exceeds maximum |
| 9 | FileEdit | Multiple matches without replace_all |
| 9 | FileRead | Blocked device path |
| 10 | FileEdit | File too large |

### ENOENT Handling

Tools use `isENOENT()` for consistent error checking:

```typescript
import { isENOENT } from '../../utils/errors.js'

try {
  const stats = await fs.stat(fullFilePath)
} catch (e) {
  if (isENOENT(e)) {
    // File doesn't exist - provide helpful suggestions
    const similarFilename = findSimilarFile(fullFilePath)
    const cwdSuggestion = await suggestPathUnderCwd(fullFilePath)
    // ... build helpful error message
  }
  throw e
}
```

## Staleness Detection (Race Condition Prevention)

Tools track file modification times to prevent stale writes:

```typescript
// Before writing, verify file hasn't changed since last read
const lastWriteTime = getFileModificationTime(fullFilePath)
const lastRead = readFileState.get(fullFilePath)
if (!lastRead || lastWriteTime > lastRead.timestamp) {
  // On Windows, timestamps can change without content changes
  // Compare content for full reads to avoid false positives
  const isFullRead = lastRead && lastRead.offset === undefined && lastRead.limit === undefined
  const contentUnchanged = isFullRead && originalFileContents === lastRead.content
  if (!contentUnchanged) {
    throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
  }
}
```

## Result Size Limiting

Tools set `maxResultSizeChars` to control result persistence:

| Tool | maxResultSizeChars | Notes |
|------|-------------------|-------|
| FileReadTool | `Infinity` | Never persist (circular read loop risk) |
| FileEditTool | `100_000` | Standard limit |
| FileWriteTool | `100_000` | Standard limit |
| GlobTool | `100_000` | Standard limit |
| GrepTool | `20_000` | Tool result persistence threshold |

### Grep Result Limiting

GrepTool implements `head_limit` and `offset` for pagination:

```typescript
const DEFAULT_HEAD_LIMIT = 250

function applyHeadLimit<T>(items: T[], limit: number | undefined, offset: number = 0) {
  // Explicit 0 = unlimited escape hatch
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

## UI Rendering Patterns

### Tool Use Message Rendering

```typescript
export function renderToolUseMessage({ file_path }: Partial<Input>, { verbose }: { verbose: boolean }): React.ReactNode {
  if (!file_path) return null
  const displayPath = verbose ? file_path : getDisplayPath(file_path)
  return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>
}
```

### Tool Result Message Rendering

**FileReadTool** returns different output types:

```typescript
renderToolResultMessage(output: Output): React.ReactNode {
  switch (output.type) {
    case 'image':
      return <MessageResponse><Text>Read image ({formatFileSize(output.file.originalSize)})</Text></MessageResponse>
    case 'notebook':
      return <MessageResponse><Text>Read <Text bold>{output.file.cells.length}</Text> cells</Text></MessageResponse>
    case 'pdf':
      return <MessageResponse><Text>PDF read ({formatFileSize(output.file.originalSize)})</Text></MessageResponse>
    case 'text':
      return <Text>{output.file.content}</Text>
  }
}
```

**FileEditTool** shows diff:

```typescript
export function renderToolResultMessage({ filePath, structuredPatch, originalFile }: FileEditOutput): React.ReactNode {
  return <FileEditToolUpdatedMessage
    filePath={filePath}
    structuredPatch={structuredPatch}
    firstLine={originalFile.split('\n')[0] ?? null}
    fileContent={originalFile}
  />
}
```

### Error Message Rendering

```typescript
export function renderToolUseErrorMessage(result: ToolResultBlockParam['content']): React.ReactNode {
  const message = typeof result === 'string' ? result : extractTag(result, 'error') ?? 'Unknown error'
  return <FallbackToolUseErrorMessage message={message} />
}
```

## Search/Read Command Classification

Tools implement `isSearchOrReadCommand()` for UI collapsing:

```typescript
// GlobTool and GrepTool
isSearchOrReadCommand() {
  return { isSearch: true, isRead: false }
}

// FileReadTool
isSearchOrReadCommand() {
  return { isSearch: false, isRead: true }
}
```

This enables condensed display in the UI for search/read operations.

## Concurrency Safety

Tools declare concurrency safety:

```typescript
// GlobTool and GrepTool are concurrency-safe
isConcurrencySafe() {
  return true
}

// FileReadTool is concurrency-safe
isConcurrencySafe() {
  return true
}

// FileEditTool and FileWriteTool are NOT concurrency-safe (default)
```

## Read-Only vs Write Tools

```typescript
// Read tools
isReadOnly() {
  return true
}

// Write tools (default is false)
```

## Search Text Extraction

For transcript search indexing:

```typescript
// GrepTool - indexes content or filenames
extractSearchText({ mode, content, filenames }) {
  if (mode === 'content' && content) return content
  return filenames.join('\n')
}

// GlobTool - indexes filenames
extractSearchText({ filenames }) {
  return filenames.join('\n')
}

// FileReadTool - returns empty (content shown via UI, not transcript)
extractSearchText() {
  return ''
}
```

## Auto-Classifier Input

Tools provide input for security classification:

```typescript
// FileEditTool
toAutoClassifierInput(input) {
  return `${input.file_path}: ${input.new_string}`
}

// FileWriteTool
toAutoClassifierInput(input) {
  return `${input.file_path}: ${input.content}`
}

// GrepTool
toAutoClassifierInput(input) {
  return input.path ? `${input.pattern} in ${input.path}` : input.pattern
}

// GlobTool
toAutoClassifierInput(input) {
  return input.pattern
}

// FileReadTool
toAutoClassifierInput(input) {
  return input.file_path
}
```

## Tool Result Mapping

Tools map results to API format:

```typescript
mapToolResultToToolResultBlockParam(data: FileEditOutput, toolUseID) {
  const { filePath, userModified, replaceAll } = data
  const modifiedNote = userModified ? '. The user modified your proposed changes.' : ''

  if (replaceAll) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: `The file ${filePath} has been updated${modifiedNote}. All occurrences were successfully replaced.`,
    }
  }

  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: `The file ${filePath} has been updated successfully${modifiedNote}.`,
  }
}
```

## Key Lessons for Building Similar Tools

1. **Use `buildTool()` with `satisfies ToolDef<...>`** for type safety and defaults

2. **Implement `validateInput()`** with early returns and error codes
   - Do cheap checks first (path validation, permission rules)
   - Skip I/O for UNC paths
   - Use `isENOENT()` for consistent error handling

3. **Set appropriate `maxResultSizeChars`**
   - Use `Infinity` for tools that shouldn't persist results
   - Use `20_000` for search tools
   - Use `100_000` for standard tools

4. **Implement staleness detection** for write tools
   - Track file state in `readFileState`
   - Compare modification times before writing
   - Handle Windows timestamp quirks

5. **Implement proper UI rendering**
   - `renderToolUseMessage()` - shows what's being done
   - `renderToolResultMessage()` - shows results
   - `renderToolUseErrorMessage()` - shows errors
   - `renderToolUseRejectedMessage()` - shows rejection UI

6. **Implement `isSearchOrReadCommand()`** for proper UI collapsing

7. **Use `extractSearchText()`** for transcript search indexing

8. **Implement `toAutoClassifierInput()`** for security classification

9. **Set `strict: true`** for strict API mode compliance

10. **Use lazy schema initialization** with `lazySchema()` to avoid initialization overhead

## File Tool-Specific Patterns

### FileReadTool Special Features

- **Multiple output types**: text, image, notebook, pdf, parts, file_unchanged
- **Image resizing**: Token-budget-aware image compression
- **PDF handling**: Page extraction and text reading
- **Notebook support**: `.ipynb` files
- **Dedup logic**: Avoids re-reading unchanged files

### FileEditTool Special Features

- **Atomic operations**: Read-modify-write with content comparison
- **Quote normalization**: Handles curly/smart quotes
- **Multi-match detection**: Requires `replace_all` flag
- **LSP integration**: Notifies language servers of changes
- **Git diff**: Optional git diff computation

### FileWriteTool Special Features

- **Create vs Update**: Distinguishes new files from overwrites
- **Directory creation**: Ensures parent directories exist
- **LSP integration**: Notifies language servers
- **Git diff**: Optional git diff computation

### GlobTool Special Features

- **Pattern matching**: Glob patterns with ripgrep
- **Result limiting**: Configurable max results
- **Sorting**: By modification time
- **Truncation flag**: Indicates when results are limited

### GrepTool Special Features

- **Ripgrep integration**: Uses `ripGrep()` utility
- **Multiple modes**: content, files_with_matches, count
- **Context lines**: -A, -B, -C support
- **Pagination**: head_limit and offset
- **VCS exclusion**: Excludes .git, .svn, etc.
- **Type filtering**: --type flag support
