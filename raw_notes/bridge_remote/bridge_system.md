# Claude Code Bridge & Remote System Analysis

## Overview

The Bridge & Remote system in Claude Code provides bidirectional communication between the local CLI and claude.ai's remote CCR (Claude Code Remote) infrastructure. It enables:

- **Remote Control**: Control local Claude Code sessions from claude.ai web interface
- **Daemon Mode**: Headless session management for background tasks
- **CCR (Claude Code Remote)**: The server-side infrastructure for remote session management

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLAUDE CODE BRIDGE ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────┐     │
│  │   REPL/CLI   │     │  Bridge Core │     │    Remote (CCR) Server   │     │
│  │              │◄───►│              │◄───►│                          │     │
│  │  - bridge/   │     │  - Poll loop │     │  - Environment API       │     │
│  │  - commands/ │     │  - Transport │     │  - Session Ingress       │     │
│  │  - init.ts   │     │  - Session   │     │  - WebSocket API         │     │
│  └──────────────┘     └──────────────┘     └──────────────────────────┘     │
│                              │                                               │
│                    ┌─────────┴─────────┐                                     │
│                    ▼                   ▼                                     │
│              ┌──────────┐       ┌──────────┐                                │
│              │   v1     │       │   v2     │                                │
│              │Env-based │       │Env-less  │                                │
│              │Hybrid WS │       │SSE+CCR   │                                │
│              └──────────┘       └──────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
/home/claudeuser/ClaudeCode/bridge/
├── bridgeMain.ts           # Main bridge daemon loop and session management
├── bridgeApi.ts            # API client for environment registration/polling
├── bridgeConfig.ts         # Auth/URL resolution and dev overrides
├── bridgeEnabled.ts        # Feature gates and entitlement checks
├── bridgeMessaging.ts      # Message handling and protocol logic
├── bridgeUI.ts             # Status display and logging for bridge
├── initReplBridge.ts       # REPL initialization wrapper
├── remoteBridgeCore.ts     # v2 env-less bridge core (direct CCR)
├── replBridge.ts           # v1 env-based bridge core
├── replBridgeTransport.ts  # Transport abstraction (v1/v2 adapters)
├── sessionRunner.ts        # Child process session spawning
├── SessionsWebSocket.ts    # WebSocket client for remote sessions
├── types.ts                # Core type definitions
└── workSecret.ts           # Work credentials and JWT handling

/home/claudeuser/ClaudeCode/commands/bridge/
├── bridge.tsx              # /remote-control command UI
└── index.ts                # Command exports
```

## Remote Control Protocol

### Two Protocol Versions

#### v1: Environment-Based (Env-based)
- **Transport**: HybridTransport (WebSocket reads + HTTP POST writes)
- **Auth**: OAuth tokens via Session-Ingress
- **API**: `/v1/environments/*` polling-based work dispatch
- **Files**: `bridgeMain.ts`, `replBridge.ts`

#### v2: Environment-Less (Env-less)
- **Transport**: SSETransport (reads) + CCRClient (writes)
- **Auth**: Worker JWTs from `/bridge` endpoint
- **API**: Direct `/v1/code/sessions/*` endpoints
- **Files**: `remoteBridgeCore.ts`, `replBridgeTransport.ts`

### Protocol Flow (v1 - Environment-Based)

```
1. Register Environment
   POST /v1/environments/bridge
   → Returns: {environment_id, environment_secret}

2. Poll for Work
   POST /v1/environments/{id}/poll
   → Returns: WorkResponse with session credentials

3. Acknowledge Work
   POST /v1/environments/{id}/work/{work_id}/ack
   → Session token activated

4. Connect Session Ingress (HybridTransport)
   WebSocket: wss://.../v1/sessions/ws/{id}/subscribe
   + HTTP POST for writes

5. Bidirectional Message Flow
   - Inbound: WebSocket receives SDK messages
   - Outbound: HTTP POST sends user messages

6. Heartbeat
   POST /v1/environments/{id}/work/{work_id}/heartbeat

7. Cleanup
   POST /v1/environments/{id}/work/{work_id}/stop
   DELETE /v1/environments/{id} (on shutdown)
```

### Protocol Flow (v2 - Environment-Less)

```
1. Create Session
   POST /v1/code/sessions (OAuth)
   → Returns: session_id (cse_* format)

2. Fetch Bridge Credentials
   POST /v1/code/sessions/{id}/bridge
   → Returns: {worker_jwt, worker_epoch, api_base_url, expires_in}

3. Register Worker (CCR v2)
   POST /v1/code/sessions/{id}/worker/register
   → Returns: worker epoch confirmation

4. Connect SSE Stream
   GET /v1/code/sessions/{id}/worker/events/stream
   - Server-Sent Events for inbound messages
   - JWT authentication via Authorization header

5. Send Events (CCRClient)
   POST /v1/code/sessions/{id}/worker/events
   - SerialBatchEventUploader for reliable delivery
   - Automatic retry with exponential backoff

6. State Reporting
   PUT /v1/code/sessions/{id}/worker
   - Report worker state (idle/running/requires_action)
   - Report external metadata

7. Delivery Tracking
   POST /v1/code/sessions/{id}/worker/events/{event_id}/delivery
   - received → processing → processed

8. Teardown
   POST /v1/sessions/{compat_id}/archive
```

## Daemon Mode

### Bridge Main Loop (`bridgeMain.ts`)

The daemon mode runs a persistent bridge server that:

1. **Registers an environment** with the CCR backend
2. **Polls for work** in a continuous loop
3. **Spawns child sessions** when work arrives
4. **Manages multiple sessions** (up to 32 by default)
5. **Handles heartbeats** and health checks
6. **Provides status UI** with live updates

### Spawn Modes

```typescript
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

- **single-session**: One session in cwd, bridge tears down when it ends
- **worktree**: Persistent server, each session gets isolated git worktree
- **same-dir**: Persistent server, all sessions share cwd (can conflict)

### Session Spawning (`sessionRunner.ts`)

Child Claude Code processes are spawned with:
- `--sdk-url`: Session ingress URL
- `--session-id`: Session identifier
- `--input-format stream-json`: NDJSON protocol
- `--output-format stream-json`: NDJSON output
- `--replay-user-messages`: Enable message replay
- Environment variables for auth tokens

## Key Protocol Types

### Work Data (`types.ts`)

```typescript
type WorkData = {
  type: 'session' | 'healthcheck'
  id: string
}

type WorkResponse = {
  id: string
  type: 'work'
  environment_id: string
  state: string
  data: WorkData
  secret: string  // base64url-encoded JSON
  created_at: string
}

type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean  // CCR v2 selector
}
```

### Session Activity Tracking

```typescript
type SessionActivityType = 'tool_start' | 'text' | 'result' | 'error'

type SessionActivity = {
  type: SessionActivityType
  summary: string  // e.g., "Editing src/foo.ts"
  timestamp: number
}
```

## Transport Layer

### v1: HybridTransport

```typescript
// WebSocket for reads, HTTP POST for writes
HybridTransport {
  connect(): void
  write(msg: StdoutMessage): Promise<void>
  writeBatch(msgs: StdoutMessage[]): Promise<void>
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
}
```

### v2: SSE + CCRClient

```typescript
// Server-Sent Events for reads, CCRClient for writes
SSETransport {
  connect(): Promise<void>
  setOnData(callback: (data: string) => void): void
  setOnEvent(callback: (event: StreamClientEvent) => void): void
  getLastSequenceNum(): number  // For resume after reconnect
}

CCRClient {
  writeEvent(msg: StdoutMessage): Promise<void>
  reportState(state: SessionState): void
  reportMetadata(metadata: Record<string, unknown>): void
  reportDelivery(eventId: string, status: 'processing' | 'processed'): void
  flush(): Promise<void>
}
```

## Message Protocol

### Control Messages

```typescript
// Permission requests from CCR
type SDKControlPermissionRequest = {
  subtype: 'can_use_tool'
  tool_name: string
  input: Record<string, unknown>
  tool_use_id: string
}

// Control request wrapper
type SDKControlRequest = {
  type: 'control_request'
  request_id: string
  request: SDKControlPermissionRequest
}

// Control response
type SDKControlResponse = {
  type: 'control_response'
  response: {
    subtype: 'success' | 'error'
    request_id: string
    response: Record<string, unknown>
  }
}

// Interrupt signal
type SDKControlRequestInner = {
  subtype: 'interrupt'
}
```

### SDK Messages

SDK messages flow between local and remote:
- `user`: User messages sent to remote
- `assistant`: Assistant responses from remote
- `tool_use` / `tool_result`: Tool execution
- `stream_event`: Real-time streaming deltas

## Authentication

### OAuth Flow (v1)

1. User authenticates with claude.ai OAuth
2. OAuth tokens stored in secure keychain
3. Bridge uses OAuth token for environment API
4. Session ingress uses separate JWT tokens

### Worker JWT Flow (v2)

1. OAuth token exchanged for worker JWT via `/bridge`
2. JWT contains session_id claim and worker role
3. JWT refreshed proactively before expiry
4. Epoch tracking prevents stale worker conflicts

### Token Refresh

```typescript
// JWT refresh scheduler (jwtUtils.ts)
createTokenRefreshScheduler({
  refreshBufferMs: 300000,  // 5 min before expiry
  onRefresh: async (sessionId, oauthToken) => {
    // Fetch fresh credentials
    // Rebuild transport with new JWT
  }
})
```

## Error Handling & Resilience

### Reconnection Strategy

1. **Transient failures**: Exponential backoff (2s → 120s cap)
2. **401 errors**: OAuth token refresh + retry
3. **409 epoch mismatch**: Full transport rebuild
4. **Permanent errors**: Stop retrying (4001 session not found, 4003 unauthorized)

### Session Recovery

```typescript
// Reconnect session after bridge restart
reconnectSession(environmentId: string, sessionId: string)
  → Force-stop stale worker
  → Re-queue session for new worker pickup
```

### Message Deduplication

- UUID-based dedup for sent messages
- Bounded ring buffer (2000 UUIDs)
- Sequence number tracking for SSE resume

## Feature Gates

Controlled via GrowthBook:

- `tengu_ccr_bridge`: Enable Remote Control feature
- `tengu_bridge_repl_v2`: Use v2 env-less protocol
- `tengu_ccr_bridge_multi_session`: Multiple sessions per environment
- `tengu_cobalt_harbor`: Auto-connect at startup
- `tengu_ccr_mirror`: Mirror mode (outbound-only)

## Security Considerations

1. **Token isolation**: OAuth and session tokens kept separate
2. **Trusted device**: X-Trusted-Device-Token header for elevated auth
3. **Sandbox mode**: `CLAUDE_CODE_FORCE_SANDBOX` for restricted execution
4. **ID validation**: Safe ID pattern prevents path traversal
5. **Permission prompts**: User approval required for tool execution

## Related Files

### Remote Session Management
- `/home/claudeuser/ClaudeCode/remote/RemoteSessionManager.ts` - CCR session manager
- `/home/claudeuser/ClaudeCode/remote/SessionsWebSocket.ts` - WebSocket client
- `/home/claudeuser/ClaudeCode/remote/remotePermissionBridge.ts` - Permission forwarding

### Transport Layer
- `/home/claudeuser/ClaudeCode/cli/transports/HybridTransport.ts` - v1 transport
- `/home/claudeuser/ClaudeCode/cli/transports/SSETransport.ts` - v2 read transport
- `/home/claudeuser/ClaudeCode/cli/transports/ccrClient.ts` - v2 write transport

### Commands
- `/home/claudeuser/ClaudeCode/commands/bridge/bridge.tsx` - /remote-control command
