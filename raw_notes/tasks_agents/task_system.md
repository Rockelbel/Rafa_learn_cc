# Claude Code Task System Analysis

## Overview

The Task System in Claude Code provides file-based task management with support for dependencies, concurrent access, and UI integration. It supports both legacy todo lists (v1) and the newer Task system (v2).

---

## Data Model

### Task Schema

Located in `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 76-89):

```typescript
interface Task {
  id: string;                    // Unique numeric ID (assigned sequentially)
  subject: string;               // Brief title for the task
  description: string;           // Detailed description of work to be done
  activeForm?: string;           // Present continuous for spinner display (e.g., "Running tests")
  owner?: string;                // Agent ID that claimed the task
  status: 'pending' | 'in_progress' | 'completed';
  blocks: string[];              // Task IDs this task blocks (dependencies)
  blockedBy: string[];           // Task IDs that block this task
  metadata?: Record<string, unknown>; // Arbitrary additional data
}
```

### Task Status States

| Status | Description |
|--------|-------------|
| `pending` | Task created but not yet started |
| `in_progress` | Task is actively being worked on |
| `completed` | Task is finished |

### Status Migration

The system includes migration logic for old status names (lines 319-331):
- `open` → `pending`
- `resolved` → `completed`
- Development statuses (`planning`, `implementing`, `reviewing`, `verifying`) → `in_progress`

---

## Storage Architecture

### File Structure

Tasks are stored as individual JSON files in a directory structure:

```
~/.claude/tasks/
  {taskListId}/
    .highwatermark          # Stores max task ID ever assigned
    .lock                   # Lock file for concurrent access
    1.json                  # Task #1
    2.json                  # Task #2
    ...
```

### Path Resolution

**Task List ID Resolution** (lines 199-210) - Priority order:
1. `CLAUDE_CODE_TASK_LIST_ID` environment variable
2. In-process teammate context (uses leader's team name)
3. `CLAUDE_CODE_TEAM_NAME` environment variable
4. Leader team name (set via `setLeaderTeamName()`)
5. Session ID (fallback)

**Path Sanitization** (lines 217-230):
- Path components are sanitized using regex `[^a-zA-Z0-9_-]` → replaced with `-`
- Prevents directory traversal attacks

### High Water Mark

The `.highwatermark` file (lines 91-131) prevents ID reuse after task deletion:

```typescript
const HIGH_WATER_MARK_FILE = '.highwatermark'

// On delete: update high water mark if deleted task ID is higher
async function writeHighWaterMark(taskListId: string, value: number): Promise<void>
async function readHighWaterMark(taskListId: string): Promise<number>
```

This ensures that even after deleting task #5, the next created task will be #6, not #5.

---

## Concurrency Control

### Lockfile Implementation

Uses `proper-lockfile` library via lazy-loaded wrapper (`/home/claudeuser/ClaudeCode/utils/lockfile.ts`):

```typescript
// Async lock with retry options
const LOCK_OPTIONS = {
  retries: {
    retries: 30,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

Budget sized for ~10+ concurrent swarm agents with ~50-100ms critical sections.

### Locking Patterns

**1. Task-Level Locking** (lines 370-391):
- Used for `updateTask()` - locks individual task file
- Check existence before locking (proper-lockfile throws if target doesn't exist)

**2. Task-List-Level Locking** (lines 504-523):
- Used for `createTask()` and `resetTaskList()`
- Locks the `.lock` file in the task list directory
- Required for operations that need to read all tasks (ID generation)

**3. Claim with Busy Check** (lines 618-692):
- Uses task-list-level lock for atomic "check agent busy + claim" operation
- Prevents TOCTOU (Time-Of-Check-Time-Of-Use) race conditions

### Lock Safety

- Always use try/finally to ensure lock release
- `updateTaskUnsafe()` variant for callers already holding a lock (prevents deadlock)
- Lock acquisition timeout: ~2.6s total wait (30 retries with exponential backoff)

---

## CRUD Operations

### Create Task (`createTask`)

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 284-308)

```typescript
export async function createTask(
  taskListId: string,
  taskData: Omit<Task, 'id'>,
): Promise<string>
```

**Flow**:
1. Ensure lock file exists
2. Acquire exclusive lock on task list
3. Read highest ID from disk (considering high water mark)
4. Generate new ID = highest + 1
5. Write task JSON to `{id}.json`
6. Emit `tasksUpdated` signal
7. Release lock

### Read Task (`getTask`)

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 310-350)

- Returns `null` if task not found (ENOENT)
- Includes migration logic for old status values
- Validates against TaskSchema using zod

### Update Task (`updateTask`)

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 370-391)

```typescript
export async function updateTask(
  taskListId: string,
  taskId: string,
  updates: Partial<Omit<Task, 'id'>>,
): Promise<Task | null>
```

- Acquires task-level lock
- Uses `updateTaskUnsafe()` internally after lock acquisition

### Delete Task (`deleteTask`)

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 393-441)

**Flow**:
1. Update high water mark before deletion (prevents ID reuse)
2. Delete the task file
3. Remove references from other tasks' `blocks` and `blockedBy` arrays
4. Emit `tasksUpdated` signal

### List Tasks (`listTasks`)

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 443-456)

- Reads all `.json` files in task directory
- Filters out non-task files (like `.highwatermark`)
- Returns array of validated Task objects

---

## Task Dependencies

### Blocking Relationships

**`blockTask()`** (lines 458-486):

```typescript
export async function blockTask(
  taskListId: string,
  fromTaskId: string,  // The task that blocks
  toTaskId: string,    // The task being blocked
): Promise<boolean>
```

Updates both directions:
- `fromTask.blocks.push(toTaskId)`
- `toTask.blockedBy.push(fromTaskId)`

### Claim Semantics with Blockers

**`claimTask()`** (lines 541-612):

A task can only be claimed if:
1. Task exists
2. Not already claimed by another agent
3. Status is not `completed`
4. **No unresolved blockers** - all tasks in `blockedBy` must have status `completed`

Returns detailed result type:

```typescript
type ClaimTaskResult = {
  success: boolean
  reason?: 'task_not_found' | 'already_claimed' | 'already_resolved' | 'blocked' | 'agent_busy'
  task?: Task
  busyWithTasks?: string[]    // When reason is 'agent_busy'
  blockedByTasks?: string[]   // When reason is 'blocked'
}
```

### Agent Busy Check

When `checkAgentBusy: true` is passed to `claimTask()`:
- Uses task-list-level lock for atomic operation
- Checks if agent owns any other unresolved tasks
- Prevents one agent from claiming multiple tasks simultaneously

---

## Signal-Based Notification System

### Signal Implementation

**File**: `/home/claudeuser/ClaudeCode/utils/signal.ts`

Tiny event primitive (~43 lines):

```typescript
export type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}

export function createSignal<Args extends unknown[] = []>(): Signal<Args>
```

### Task Update Notifications

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 17-67)

```typescript
const tasksUpdated = createSignal()

export const onTasksUpdated = tasksUpdated.subscribe

export function notifyTasksUpdated(): void {
  try {
    tasksUpdated.emit()
  } catch {
    // Ignore listener errors - task mutations must not fail
  }
}
```

**Usage Pattern**:
- UI components subscribe via `onTasksUpdated()`
- Any CRUD operation calls `notifyTasksUpdated()` after completion
- Listeners re-fetch tasks to refresh UI
- Wrapped in try/catch so notification failures never propagate

---

## UI Integration

### Task List Display Component

**File**: `/home/claudeuser/ClaudeCode/components/TaskListV2.tsx`

React component using Ink for terminal UI:

**Key Features**:
- **Status Icons** (lines 220-241):
  - `completed`: tick (✓) in success color
  - `in_progress`: squareSmallFilled (◼) in claude color
  - `pending`: squareSmall (◻) dim

- **Display Prioritization** (lines 139-169):
  - Recent completed (within 30s TTL)
  - In-progress tasks
  - Pending tasks (unblocked first)
  - Older completed tasks
  - Max 10 tasks displayed (adaptive based on terminal rows)

- **Blocked Task Display**: Shows "blocked by #1, #2" with blocker IDs
- **Owner Display**: Shows `@ownername` with color coding for teammates
- **Activity Display**: Shows current activity for in-progress tasks

### Tasks Command

**File**: `/home/claudeuser/ClaudeCode/commands/tasks/tasks.tsx`

Simple command entry point:

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
): Promise<React.ReactNode> {
  return <BackgroundTasksDialog toolUseContext={context} onDone={onDone} />
}
```

Opens `BackgroundTasksDialog` for interactive task management.

---

## Tool Integration

### TaskCreateTool

**File**: `/home/claudeuser/ClaudeCode/tools/TaskCreateTool/TaskCreateTool.ts`

**Input Schema**:
```typescript
{
  subject: string;           // Brief title
  description: string;       // What needs to be done
  activeForm?: string;       // Spinner text (e.g., "Running tests")
  metadata?: Record<string, unknown>;
}
```

**Behavior**:
- Creates task with `status: 'pending'`
- Executes task creation hooks (for validation/policy)
- Auto-expands task list in UI via `setAppState()`
- If hooks return blocking errors, deletes the task and throws

**Enabled via**: `isTodoV2Enabled()` - returns true in interactive sessions

### TodoWriteTool (Legacy)

**File**: `/home/claudeuser/ClaudeCode/tools/TodoWriteTool/TodoWriteTool.ts`

- Legacy tool for session-scoped todo lists
- Stored in `AppState.todos[sessionId]`
- Disabled when `isTodoV2Enabled()` returns true
- Includes verification agent nudge for 3+ completed tasks without verification step

---

## Team/Swarm Integration

### Leader Team Name

**File**: `/home/claudeuser/ClaudeCode/utils/tasks.ts` (lines 25-47)

```typescript
let leaderTeamName: string | undefined

export function setLeaderTeamName(teamName: string): void
export function clearLeaderTeamName(): void
```

- Set by `TeamCreateTool` when a team is created
- Causes leader's tasks to be stored under team name (shared with teammates)
- Changing team name triggers `notifyTasksUpdated()`

### Agent Status Tracking

**`getAgentStatuses()`** (lines 763-798):

```typescript
export async function getAgentStatuses(
  teamName: string,
): Promise<AgentStatus[] | null>

type AgentStatus = {
  agentId: string
  name: string
  agentType?: string
  status: 'idle' | 'busy'
  currentTasks: string[]
}
```

- Reads team members from team config file
- Cross-references with task ownership
- Agent is "busy" if they own any unresolved tasks

### Task Unassignment on Teammate Exit

**`unassignTeammateTasks()`** (lines 818-860):

- Called when teammate is killed or gracefully shuts down
- Unassigns all open tasks from the exiting teammate
- Resets task status to `pending`
- Returns notification message for leader

---

## Environment Controls

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_ENABLE_TASKS` | Force-enable Task tools (even in non-interactive mode) |
| `CLAUDE_CODE_TASK_LIST_ID` | Explicit task list ID override |
| `CLAUDE_CODE_TEAM_NAME` | Set when running as process-based teammate |

---

## Key Design Decisions

1. **File-per-task vs Single File**: File-per-task enables granular locking and reduces merge conflicts in concurrent scenarios

2. **High Water Mark**: Prevents ID confusion when tasks are deleted and new ones created

3. **Signal vs Store**: Signals notify "something happened" without storing state - UI re-fetches fresh data

4. **Two-Level Locking**: Task-level for updates, list-level for ID generation and agent busy checks

5. **Lazy Lockfile Loading**: proper-lockfile adds ~8ms startup cost - lazy loaded to avoid impacting `--help`

6. **Graceful Degradation**: All file operations have try/catch; missing files return null rather than throwing
