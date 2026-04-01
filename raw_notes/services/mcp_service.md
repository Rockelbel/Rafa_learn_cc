# MCP (Model Context Protocol) Service Analysis

## Overview

The MCP Service in Claude Code implements the Model Context Protocol, enabling integration with external MCP servers that expose tools, prompts, and resources. This service provides a complete client implementation for connecting to MCP servers via multiple transport mechanisms.

## Protocol Implementation

### Core Protocol Types (`types.ts`)

**Server Configuration Types:**
- `McpStdioServerConfig`: Local subprocess-based servers (command + args)
- `McpSSEServerConfig`: Server-Sent Events transport with OAuth support
- `McpHTTPServerConfig`: HTTP transport with OAuth support
- `McpWebSocketServerConfig`: WebSocket transport
- `McpSdkServerConfig`: In-process SDK-managed servers
- `McpClaudeAIProxyServerConfig`: Claude.ai connector proxies
- `McpSSEIDEServerConfig` / `McpWebSocketIDEServerConfig`: IDE extension transports

**Connection State Types:**
- `ConnectedMCPServer`: Active connection with client, capabilities, tools
- `FailedMCPServer`: Connection failed with error details
- `NeedsAuthMCPServer`: Requires OAuth authentication
- `PendingMCPServer`: Connection in progress with retry tracking
- `DisabledMCPServer`: User-disabled server

### Transport Implementations

**1. In-Process Transport (`InProcessTransport.ts`)**
- Linked transport pair for same-process MCP communication
- Uses `queueMicrotask` for async message delivery
- Prevents stack depth issues with synchronous request/response

**2. SDK Control Transport (`SdkControlTransport.ts`)**
- Bridges CLI process ↔ SDK process communication
- `SdkControlClientTransport`: CLI side - wraps MCP messages in control requests
- `SdkControlServerTransport`: SDK side - forwards messages to MCP server
- Enables MCP servers running in the SDK to communicate with CLI

**3. Standard MCP SDK Transports**
- `stdio`: Subprocess-based (most common)
- `sse`: Server-Sent Events for remote servers
- `http`: Direct HTTP connections
- `ws`: WebSocket connections

## Server Management

### Configuration Management (`config.ts`)

**Configuration Sources (in precedence order):**
1. Enterprise (`enterprise`): Managed enterprise configuration (exclusive control)
2. Local (`local`): Project-local settings
3. Project (`project`): `.mcp.json` files in current and parent directories
4. User (`user`): Global user configuration
5. Dynamic (`dynamic`): Command-line provided configs
6. Plugin (`plugin`): Plugin-provided MCP servers
7. Claude.ai (`claudeai`): Cloud connector configurations

**Key Features:**
- Environment variable expansion in configurations
- Deduplication of plugin vs manual servers by signature
- CCR proxy URL unwrapping for claude.ai connectors
- Policy filtering via allowlist/denylist

### Connection Lifecycle (`useManageMCPConnections.ts`)

**Initialization Flow:**
1. Load configurations from all sources
2. Initialize servers as `pending` state
3. Connect to enabled servers in parallel
4. Two-phase loading: Claude Code configs first, then claude.ai configs

**Reconnection Logic:**
- Exponential backoff for remote transports (SSE/HTTP/WS)
- Max 5 reconnection attempts
- Backoff: 1s → 2s → 4s → 8s → 16s (capped at 30s)
- Cancellation support for manual reconnect/quit

**State Management:**
- Batched updates via `setTimeout` (16ms window)
- Tracks clients, tools, commands, resources
- Stale client detection and cleanup on config changes

### Authentication (`auth.ts`)

**OAuth 2.0 Implementation:**
- `ClaudeAuthProvider`: Full OAuth client provider implementation
- PKCE (Proof Key for Code Exchange) flow
- Token refresh with proactive expiration handling
- Step-up authentication for scope escalation
- Secure token storage in OS keychain

**XAA (Cross-App Access):**
- Single IdP login for multiple MCP servers
- RFC 8693 token exchange + RFC 7523 JWT bearer
- Cached id_token enables silent re-authentication
- No browser pop after initial IdP authorization

**Token Management:**
- Lockfile-based refresh deduplication across processes
- Token revocation on logout (RFC 7009)
- Automatic retry with exponential backoff

## Tool Integration

### Tool Exposure (`client.ts` - referenced)

**Tool Discovery:**
- Fetches tools via `tools/list` MCP method
- Normalizes tool names: `mcp__{serverName}__{toolName}`
- Caches tools per client with memoization

**Command/Prompt Discovery:**
- Fetches prompts via `prompts/list`
- Converts to internal Command format
- MCP skills loaded from resources (if `MCP_SKILLS` feature enabled)

### Tool Execution

**Call Flow:**
1. Tool name parsed to extract server and original tool name
2. Request routed to appropriate MCP client
3. Input validated against schema
4. `tools/call` invoked on MCP server
5. Result truncated if exceeds token limit (`mcpValidation.ts`)
6. Response returned to Claude

**Output Handling:**
- Token-based truncation (default 25k tokens)
- Image compression for large image outputs
- Smart truncation preserving text over images when needed

## Resource Handling

### Resource Discovery
- Fetched via `resources/list` MCP method
- Annotated with source server name
- Cached per client

### Resource Access
- `resources/read` for fetching content
- URI-based identification
- Support for binary and text content

### Notifications
- `resources/list_changed`: Refresh resource cache
- `resources/updated`: Notify on resource updates

## Server Lifecycle

### Connection States

```
pending → connected → [tools/commands/resources available]
   ↓           ↓
failed    disconnected → [auto-reconnect for remote]
   ↑           ↓
needs-auth ← disabled (user toggle)
```

### Lifecycle Hooks

**On Connect:**
1. Initialize transport
2. Send `initialize` request
3. Fetch capabilities
4. Register notification handlers
5. Fetch tools/commands/resources
6. Update AppState

**On Disconnect:**
1. Clear server cache
2. Cancel pending reconnect timers
3. Update state to `failed` or `disabled`
4. Trigger auto-reconnect if enabled and remote transport

**On Config Change:**
1. Hash configs to detect changes
2. Disconnect stale servers (removed or modified)
3. Initialize new servers as pending
4. Connect to new enabled servers

## Channel Notifications (`channelNotification.ts`)

**Channel Push Feature:**
- MCP servers can push messages into conversation via `notifications/claude/channel`
- Requires `claude/channel` capability declaration
- Content wrapped in `<channel>` XML tags
- Enqueued for processing by SleepTool

**Permission Requests:**
- `notifications/claude/channel/permission` for approval flows
- Server parses user replies and emits structured responses
- Request ID correlation for matching pending permissions

**Gating Logic:**
- Capability check → Feature flag → Auth check → Policy check → Session opt-in → Allowlist check
- Plugin marketplace verification
- Admin-controlled allowlist for enterprise

## Advanced Features

### VSCode SDK MCP (`vscodeSdkMcp.ts`)
- Bidirectional communication with VSCode extension
- `file_updated` notifications for file changes
- `log_event` notification handler for telemetry
- Experiment gates synchronization

### Name Normalization (`normalization.ts`)
- Server names normalized to `^[a-zA-Z0-9_-]{1,64}$`
- Invalid characters replaced with underscores
- Claude.ai servers: consecutive underscores collapsed

### String Utilities (`mcpStringUtils.ts`)
- Tool name parsing: `mcp__{server}__{tool}` → `{server, tool}`
- Prefix generation for filtering
- Original name preservation for display

### Validation (`mcpValidation.ts`)
- MCP output token counting
- Truncation at configurable limit (env/GrowthBook/hardcoded)
- Image resizing for large outputs
- Content block handling (text + images)

### Enterprise Features

**Policy Controls:**
- `allowedMcpServers`: Name/command/URL-based allowlist
- `deniedMcpServers`: Block list with pattern matching
- `allowManagedMcpServersOnly`: Restrict to enterprise-managed servers

**Managed Configuration:**
- `managed-mcp.json` file in managed path
- Exclusive control when present (blocks user servers)
- Plugin-only mode for locked-down environments

## Security Considerations

1. **OAuth Security:**
   - PKCE for public clients
   - State parameter CSRF protection
   - Secure token storage (OS keychain)
   - Token revocation on logout

2. **Transport Security:**
   - HTTPS required for OAuth metadata
   - URL validation before redirects
   - Sensitive parameter redaction in logs

3. **Policy Enforcement:**
   - Multi-layer allowlist/denylist
   - Project server approval workflow
   - Enterprise exclusive control option

4. **Input Validation:**
   - Zod schema validation for configs
   - MCP tool input schema enforcement
   - Safe meta key validation for channels

## File Structure

```
services/mcp/
├── client.ts                    # MCP client implementation
├── types.ts                     # Core type definitions
├── config.ts                    # Configuration management
├── auth.ts                      # OAuth authentication
├── useManageMCPConnections.ts   # React hook for connection management
├── MCPConnectionManager.tsx     # React context provider
├── InProcessTransport.ts        # In-process transport
├── SdkControlTransport.ts       # SDK bridge transport
├── channelNotification.ts       # Channel push notifications
├── channelPermissions.ts        # Channel permission handling
├── channelAllowlist.ts            # Channel allowlist management
├── normalization.ts             # Name normalization utilities
├── mcpStringUtils.ts            # String parsing utilities
├── utils.ts                     # General MCP utilities
├── vscodeSdkMcp.ts              # VSCode integration
├── xaa.ts                       # Cross-App Access implementation
├── xaaIdpLogin.ts               # XAA IdP login flow
├── oauthPort.ts                 # OAuth callback port management
├── headersHelper.ts             # Header helper management
├── elicitationHandler.ts        # Elicitation (sampling) handler
├── officialRegistry.ts          # Official MCP registry
├── claudeai.ts                  # Claude.ai connector fetching
└── envExpansion.ts              # Environment variable expansion
```

## Integration Points

- **AppState**: MCP clients, tools, commands, resources stored centrally
- **Tool System**: MCP tools exposed alongside built-in tools
- **Command System**: MCP prompts exposed as slash commands
- **Permission System**: Tool use permissions for MCP tools
- **Analytics**: Comprehensive event logging for debugging
- **Notifications**: Toast notifications for auth/channel issues
- **Plugin System**: Plugin-provided MCP server integration
