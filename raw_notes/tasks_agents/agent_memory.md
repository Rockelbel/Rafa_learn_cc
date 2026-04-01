# Agent Memory System Analysis

## Overview

The Agent Memory System in Claude Code provides persistent storage for agent-specific knowledge across sessions. Agents can store learnings, preferences, and context that persists between conversations.

## Core Files

### 1. agentMemory.ts - Memory Persistence

**Location:** `/home/claudeuser/ClaudeCode/tools/AgentTool/agentMemory.ts`

**Key Concepts:**

- **AgentMemoryScope**: Three persistence scopes available:
  - `'user'`: Stored in `~/.claude/agent-memory/<agentType>/` - applies across all projects
  - `'project'`: Stored in `<cwd>/.claude/agent-memory/<agentType>/` - shared via version control
  - `'local'`: Stored in `<cwd>/.claude/agent-memory-local/<agentType>/` - machine-specific, not version controlled

**Memory Directory Structure:**
```
~/.claude/agent-memory/
  <agent-type>/
    MEMORY.md          # Entrypoint file
    topic-file.md      # Individual memory files
```

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `getAgentMemoryDir(agentType, scope)` | Returns the directory path for agent memory |
| `getAgentMemoryEntrypoint(agentType, scope)` | Returns path to MEMORY.md |
| `isAgentMemoryPath(absolutePath)` | Security check - validates if path is within agent memory |
| `loadAgentMemoryPrompt(agentType, scope)` | Loads memory and builds system prompt with scope-specific guidelines |

**Scope-Specific Guidelines:**
- User scope: "keep learnings general since they apply across all projects"
- Project scope: "tailor your memories to this project" (shared via VCS)
- Local scope: "tailor your memories to this project and machine" (not in VCS)

**Remote Memory Support:**
When `CLAUDE_CODE_REMOTE_MEMORY_DIR` is set, local-scope memory persists to a mount with project namespacing instead of the local filesystem.

---

### 2. agentMemorySnapshot.ts - Snapshot Mechanism

**Location:** `/home/claudeuser/ClaudeCode/tools/AgentTool/agentMemorySnapshot.ts`

**Purpose:** Enables project-level distribution of agent memory via version-controlled snapshots.

**Snapshot Structure:**
```
.claude/agent-memory-snapshots/
  <agent-type>/
    snapshot.json       # Metadata with timestamp
    *.md                # Memory files
```

**Key Files:**
- `snapshot.json`: Contains `{ updatedAt: string }` timestamp
- `.snapshot-synced.json`: Tracks last sync in local memory directory

**Snapshot Actions:**

| Action | Description |
|--------|-------------|
| `'none'` | No snapshot exists or already synced |
| `'initialize'` | First-time setup - copy snapshot to local |
| `'prompt-update'` | Newer snapshot exists - prompt user for update |

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `checkAgentMemorySnapshot(agentType, scope)` | Compares snapshot vs local, returns action needed |
| `initializeFromSnapshot(agentType, scope, timestamp)` | Copies snapshot to local for first use |
| `replaceFromSnapshot(agentType, scope, timestamp)` | Replaces local memory with snapshot (removes orphans) |
| `markSnapshotSynced(agentType, scope, timestamp)` | Updates sync metadata without changing memory |

**Initialization Flow:**
1. Check if snapshot exists and get its timestamp
2. Check if local memory exists
3. If no local memory: initialize from snapshot
4. If newer snapshot: set `pendingSnapshotUpdate` on agent definition
5. Track sync via `.snapshot-synced.json`

---

### 3. agentDisplay.ts - Display Patterns

**Location:** `/home/claudeuser/ClaudeCode/tools/AgentTool/agentDisplay.ts`

**Purpose:** Shared utilities for displaying agent information in CLI and interactive UI.

**Agent Source Groups (Ordered by Priority):**

```typescript
[
  { label: 'User agents', source: 'userSettings' },
  { label: 'Project agents', source: 'projectSettings' },
  { label: 'Local agents', source: 'localSettings' },
  { label: 'Managed agents', source: 'policySettings' },
  { label: 'Plugin agents', source: 'plugin' },
  { label: 'CLI arg agents', source: 'flagSettings' },
  { label: 'Built-in agents', source: 'built-in' },
]
```

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `resolveAgentOverrides(allAgents, activeAgents)` | Annotates agents with override info, deduplicates worktree duplicates |
| `resolveAgentModelDisplay(agent)` | Returns model alias or 'inherit' for display |
| `getOverrideSourceLabel(source)` | Human-readable label for override source |
| `compareAgentsByName(a, b)` | Case-insensitive alphabetical comparison |

**Override Resolution:**
- An agent is "overridden" when another agent with the same type from higher-priority source takes precedence
- Deduplication key: `${agentType}:${source}` - handles git worktree duplicates

---

### 4. agentColorManager.ts - Agent Identification

**Location:** `/home/claudeuser/ClaudeCode/tools/AgentTool/agentColorManager.ts`

**Purpose:** Assigns and manages colors for agent identification in the UI.

**Available Colors:**
```typescript
['red', 'blue', 'green', 'yellow', 'purple', 'orange', 'pink', 'cyan']
```

**Color Mapping:**
Colors map to theme-specific color keys (e.g., `'red'` → `'red_FOR_SUBAGENTS_ONLY'`)

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `getAgentColor(agentType)` | Returns theme color key if assigned, undefined for 'general-purpose' |
| `setAgentColor(agentType, color)` | Assigns or removes color for an agent |

**State Storage:**
- Colors stored in `STATE.agentColorMap: Map<string, AgentColorName>`
- Initialized in bootstrap at `/home/claudeuser/ClaudeCode/bootstrap/state.ts`

**Special Case:**
- `general-purpose` agent never gets a color (returns undefined)
- Colors are only applied if they exist in `AGENT_COLORS` array

---

## Integration Points

### Agent Loading (loadAgentsDir.ts)

**Memory Integration:**
- When loading custom agents, checks for memory snapshots if `AGENT_MEMORY_SNAPSHOT` feature is enabled
- Calls `initializeAgentMemorySnapshots()` for agents with `memory: 'user'` scope
- Injects memory prompt into system prompt via `loadAgentMemoryPrompt()`

**Color Integration:**
- After resolving active agents, initializes colors:
```typescript
for (const agent of activeAgents) {
  if (agent.color) {
    setAgentColor(agent.agentType, agent.color)
  }
}
```

**Memory-Aware Tool Injection:**
When memory is enabled, automatically injects file tools:
- `FileWriteTool`
- `FileEditTool`
- `FileReadTool`

This ensures agents can read/write their memory files.

### Memory Prompt Building (memdir/memdir.ts)

**Agent Memory Prompt Structure:**
1. Header with memory directory path
2. Scope-specific guidelines (user/project/local)
3. Memory taxonomy (user, feedback, project, reference)
4. What NOT to save (code patterns, derivable context)
5. How to save memories (frontmatter format)
6. MEMORY.md content (truncated if >200 lines or 25KB)

**Frontmatter Format:**
```yaml
---
name: "Memory Title"
description: "One-line description"
type: user | feedback | project | reference
---
Content here...
```

---

## Agent Resume from Memory

### Resume Process:

1. **Agent Definition Loading:**
   - Parse agent from markdown/JSON
   - Extract `memory` scope if specified
   - Store in `AgentDefinition.memory?: AgentMemoryScope`

2. **Memory Snapshot Check:**
   - If `AGENT_MEMORY_SNAPSHOT` feature enabled and auto-memory enabled
   - Call `checkAgentMemorySnapshot()` for user-scope agents
   - Initialize or prompt for update based on snapshot status

3. **System Prompt Construction:**
   - When `getSystemPrompt()` called, check if memory enabled
   - If enabled: append memory prompt via `loadAgentMemoryPrompt()`
   - Memory directory guaranteed to exist via `ensureMemoryDirExists()`

4. **Color Assignment:**
   - During agent loading, colors assigned via `setAgentColor()`
   - Retrieved during display via `getAgentColor()`

---

## Security Considerations

### Path Validation:
- `isAgentMemoryPath()` normalizes paths to prevent traversal attacks
- Checks against all three scope directories
- Used to validate file operations within agent memory

### Scope Isolation:
- User scope: Isolated to user's home directory
- Project scope: Isolated to current working directory
- Local scope: Isolated to project-local directory (may be remote mount)

### Agent Type Sanitization:
- `sanitizeAgentTypeForPath()` replaces colons with dashes
- Handles plugin-namespaced agents like `"my-plugin:my-agent"`

---

## Data Flow Summary

```
┌─────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  Agent Config   │────▶│  loadAgentsDir.ts   │────▶│  AgentDefinition │
│  (.md/.json)    │     │  (parse & validate) │     │  (with memory)   │
└─────────────────┘     └─────────────────────┘     └──────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
   ┌────────────────┐  ┌──────────────┐   ┌─────────────────┐
   │ Snapshot Check │  │ Color Assign │   │ Tool Injection  │
   │ (optional)     │  │              │   │ (if memory)     │
   └────────────────┘  └──────────────┘   └─────────────────┘
            │
            ▼
   ┌────────────────┐
   │ Memory Prompt  │◄──── loadAgentMemoryPrompt()
   │ (system prompt)│
   └────────────────┘
```

---

## Key Insights

1. **Hierarchical Scopes**: Memory scopes follow a clear hierarchy - user (broadest) → project → local (narrowest)

2. **Snapshot Distribution**: Enables teams to share agent configurations via version control while allowing personal modifications

3. **Lazy Color Assignment**: Colors are assigned at load time and stored in a simple Map, not persisted

4. **Fire-and-Forget Directory Creation**: `ensureMemoryDirExists()` is called synchronously but handles async creation; model can write immediately

5. **Automatic Tool Injection**: Memory-enabled agents automatically get file tools without explicit configuration

6. **Worktree Handling**: Display utilities deduplicate agents from git worktrees to avoid confusion
