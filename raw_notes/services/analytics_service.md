# Analytics Service Analysis

## Overview

The Analytics Service is located at `/home/claudeuser/ClaudeCode/services/analytics/` and provides comprehensive event tracking, telemetry collection, and feature flag management for Claude Code. It uses a multi-sink architecture supporting both first-party (1P) event logging and third-party analytics (Datadog).

## File Structure

```
services/analytics/
├── index.ts                    # Public API for event logging, AnalyticsSink interface
├── config.ts                   # Analytics configuration and enablement checks
├── firstPartyEventLogger.ts    # 1P event logging with OpenTelemetry
├── firstPartyEventLoggingExporter.ts  # Custom exporter to Anthropic API
├── growthbook.ts              # Feature flags and experiments (GrowthBook integration)
├── sink.ts                    # Analytics sink implementation (routes to Datadog + 1P)
├── sinkKillswitch.ts          # Per-sink killswitch controls
├── datadog.ts                 # Datadog integration
└── metadata.ts                # Event metadata enrichment
```

---

## 1. Event System

### Core Event API (`index.ts`)

**Public API:**
- `logEvent(eventName, metadata)` - Synchronous event logging
- `logEventAsync(eventName, metadata)` - Asynchronous event logging
- `attachAnalyticsSink(sink)` - Attaches the backend sink during startup

**Design Principles:**
- Zero-dependency module to avoid import cycles
- Events queued until sink is attached (no events lost during startup)
- Type-safe metadata: only `boolean | number | undefined` values allowed (no strings to prevent accidental PII logging)

**PII Protection:**
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` - Marker type requiring explicit verification
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` - For values routed to privileged proto columns
- `stripProtoFields()` - Removes `_PROTO_*` keys from general-access backends

**Queued Event Processing:**
```typescript
// Events logged before sink attachment are queued
type QueuedEvent = {
  eventName: string
  metadata: LogEventMetadata
  async: boolean
}

// Queue drained asynchronously via queueMicrotask when sink attaches
```

### Event Sampling (`firstPartyEventLogger.ts`)

Dynamic event sampling via GrowthBook config `tengu_event_sampling_config`:

```typescript
type EventSamplingConfig = {
  [eventName: string]: {
    sample_rate: number  // 0-1
  }
}

// Sampling logic:
// - No config = 100% rate (no sampling)
// - rate >= 1 = 100% rate
// - rate <= 0 = drop everything
// - 0 < rate < 1 = random sampling
```

---

## 2. Telemetry

### Metadata Enrichment (`metadata.ts`)

**Core Metadata Structure:**
```typescript
type EventMetadata = {
  model: string
  sessionId: string
  userType: string
  betas?: string
  envContext: EnvContext
  entrypoint?: string
  agentSdkVersion?: string
  isInteractive: string
  clientType: string
  processMetrics?: ProcessMetrics
  sweBenchRunId?: string
  agentId?: string
  parentSessionId?: string
  agentType?: 'teammate' | 'subagent' | 'standalone'
  teamName?: string
  subscriptionType?: string
  rh?: string  // Hashed repo remote URL
  kairosActive?: true
  skillMode?: 'discovery' | 'coach' | 'discovery_and_coach'
  observerMode?: 'backseat' | 'skillcoach' | 'both'
}
```

**Environment Context:**
- Platform, arch, node version
- Package managers and runtimes
- CI/CD detection (GitHub Actions, etc.)
- Remote environment detection
- WSL version, Linux distro info
- VCS detection

**Process Metrics:**
- Memory usage (rss, heap, external)
- CPU usage and percentage
- Uptime

**PII Sanitization:**
- `sanitizeToolNameForAnalytics()` - Redacts MCP tool names (user-specific configs)
- `extractMcpToolDetails()` - Only logs details for official/whitelisted MCP servers
- `getFileExtensionForAnalytics()` - Safe file extension extraction (no paths)
- Tool input truncation: strings >512 chars, JSON >4KB, nested depth >2

### 1P Event Logging Pipeline

**Architecture:**
```
Event -> logEventTo1P() -> OpenTelemetry Logger -> BatchLogRecordProcessor -> FirstPartyEventLoggingExporter -> POST /api/event_logging/batch
```

**Features:**
- Separate LoggerProvider from customer OTLP telemetry
- Batched export (configurable: default 200 events, 10s interval)
- Disk-backed retry queue for failed events
- Quadratic backoff retry (max 8 attempts)
- Short-circuit on first failure to avoid hammering unhealthy endpoints
- Auth fallback: retries without auth on 401 errors

**Resilience Mechanisms:**
1. Failed events written to disk (`~/.claude/telemetry/1p_failed_events.{sessionId}.{uuid}.json`)
2. Retry on startup for previous batch failures
3. Killswitch support via `isSinkKilled('firstParty')`

**Event Types:**
- `ClaudeCodeInternalEvent` - Standard internal events
- `GrowthbookExperimentEvent` - A/B test exposure events

### Datadog Integration (`datadog.ts`)

**Configuration:**
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Flush interval: 15s (configurable via `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`)
- Max batch size: 100 events

**Allowed Events:**
Whitelist of 60+ event names (e.g., `tengu_api_success`, `tengu_tool_use_success`, `chrome_bridge_*`)

**Cardinality Reduction:**
- MCP tool names normalized to "mcp"
- Model names canonicalized
- Dev versions truncated to base + date
- User bucketing: 30 buckets via SHA256 hash (privacy-preserving unique user estimation)

**Tag Fields:**
`arch`, `clientType`, `errorType`, `http_status`, `model`, `platform`, `provider`, `toolName`, `userBucket`, `version`

---

## 3. Feature Flags (GrowthBook)

### GrowthBook Integration (`growthbook.ts`)

**Client Configuration:**
- Remote evaluation enabled (`remoteEval: true`)
- Cache key attributes: `id`, `organizationUUID`
- Refresh interval: 6 hours (external), 20 minutes (ant)
- Auth headers included for authenticated requests

**User Attributes:**
```typescript
type GrowthBookUserAttributes = {
  id: string              // Device ID
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string // For enterprise proxy targeting
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

### Feature Flag API

**Cached (Non-blocking) - Preferred:**
```typescript
getFeatureValue_CACHED_MAY_BE_STALE<T>(feature: string, defaultValue: T): T
getDynamicConfig_CACHED_MAY_BE_STALE<T>(configName: string, defaultValue: T): T
checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate: string): boolean
```

**Blocking (awaits init):**
```typescript
getFeatureValue_DEPRECATED<T>(feature: string, defaultValue: T): Promise<T>
getDynamicConfig_BLOCKS_ON_INIT<T>(configName: string, defaultValue: T): Promise<T>
```

**Security Gates:**
```typescript
checkSecurityRestrictionGate(gate: string): Promise<boolean>
checkGate_CACHED_OR_BLOCKING(gate: string): Promise<boolean>
```

### Experiment Logging

**Exposure Tracking:**
- Deduplicated per session (`loggedExposures` Set)
- Logged to 1P event pipeline as `GrowthbookExperimentEvent`
- Deferred logging for features accessed before init

**Experiment Data:**
```typescript
type StoredExperimentData = {
  experimentId: string
  variationId: number
  inExperiment?: boolean
  hashAttribute?: string
  hashValue?: string
}
```

### Config Overrides (Ant-only)

**Environment Variable Overrides:**
- `CLAUDE_INTERNAL_FC_OVERRIDES` - JSON object for eval harnesses
- Takes precedence over remote values

**Config File Overrides:**
- `/config` Gates tab in UI
- Stored in `growthBookOverrides` in global config
- Survives restarts

---

## 4. Privacy Controls

### Analytics Disablement (`config.ts`)

**Analytics is disabled when:**
1. `NODE_ENV === 'test'`
2. Third-party cloud providers: `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`
3. `isTelemetryDisabled()` returns true (privacy level check)

**Privacy Levels (from `privacyLevel.ts`):**
- `no-telemetry` - All telemetry disabled
- `essential-traffic` - Essential traffic only

### Sink Killswitch (`sinkKillswitch.ts`)

**Dynamic Per-Sink Control:**
```typescript
type SinkName = 'datadog' | 'firstParty'

isSinkKilled(sink: SinkName): boolean
```

- Config name: `tengu_frond_boric` (GrowthBook dynamic config)
- Fail-open: missing/malformed config = sink stays on
- Checked per-event dispatch (not at enablement time)

### PII Handling

**Data Classification:**
- **General-access (Datadog):** `_PROTO_*` keys stripped
- **Privileged (1P only):** `_PROTO_skill_name`, `_PROTO_plugin_name`, `_PROTO_marketplace_name` → hoisted to proto fields

**MCP Tool Name Sanitization:**
- Custom/user-configured MCPs: logged as `mcp_tool`
- Official/whitelisted: full name allowed (Cowork, claude.ai-proxied, official registry)

---

## 5. Configuration Summary

| Config Name | Purpose | Source |
|-------------|---------|--------|
| `tengu_event_sampling_config` | Event sampling rates | GrowthBook |
| `tengu_1p_event_batch_config` | Batch processor settings | GrowthBook |
| `tengu_frond_boric` | Sink killswitches | GrowthBook |
| `tengu_log_datadog_events` | Datadog logging gate | GrowthBook |
| `CLAUDE_INTERNAL_FC_OVERRIDES` | Feature flag overrides | Env var (ant-only) |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Datadog flush interval | Env var |
| `OTEL_LOGS_EXPORT_INTERVAL` | 1P export interval | Env var |
| `OTEL_LOG_TOOL_DETAILS` | Enable detailed tool logging | Env var |

---

## Key Design Decisions

1. **Multi-Sink Architecture:** Events route to both Datadog (general access) and 1P (privileged) with different PII filtering
2. **Caching Strategy:** Disk cache survives restarts; memory cache for hot paths
3. **Remote Eval:** Server-side feature evaluation reduces client-side logic
4. **Backpressure Handling:** Quadratic backoff with disk queue prevents data loss during outages
5. **Privacy-First:** No strings in metadata by default; explicit verification markers required
6. **Separation of Concerns:** Customer OTLP telemetry kept completely separate from internal 1P event logging
