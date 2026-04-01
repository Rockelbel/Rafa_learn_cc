# Claude Code API Service Analysis

## Overview

The API Service in Claude Code provides a robust, multi-provider abstraction layer for interacting with Anthropic's LLM APIs. It handles authentication, retry logic, rate limiting, error classification, and streaming responses across multiple deployment modes (First-Party, AWS Bedrock, GCP Vertex, Azure Foundry).

---

## File Structure

| File | Purpose |
|------|---------|
| `client.ts` | Anthropic SDK client factory with multi-provider support |
| `claude.ts` | Main API query layer, message handling, request building |
| `withRetry.ts` | Retry logic with exponential backoff and circuit breaker patterns |
| `errors.ts` | Error classification, message generation, error handling utilities |
| `errorUtils.ts` | Connection error extraction, SSL/TLS error handling |
| `claudeAiLimits.ts` | Rate limit tracking and quota management |

---

## 1. API Client Architecture (`client.ts`)

### Multi-Provider Support

The client factory supports four deployment modes via environment variables:

```
CLAUDE_CODE_USE_BEDROCK   → AWS Bedrock (AnthropicBedrock SDK)
CLAUDE_CODE_USE_VERTEX    → GCP Vertex AI (AnthropicVertex SDK)
CLAUDE_CODE_USE_FOUNDRY   → Azure Foundry (AnthropicFoundry SDK)
(default)                 → First-Party Anthropic API
```

### Authentication Patterns

| Provider | Auth Method |
|----------|-------------|
| First-Party | OAuth tokens (Claude AI subscribers) or API key |
| Bedrock | AWS credentials (env vars, profile, or IAM) |
| Vertex | GCP service account or ADC (Application Default Credentials) |
| Foundry | Azure AD token provider or API key |

### Key Features

- **Client Request ID Injection**: Generates `x-client-request-id` for request tracing (first-party only)
- **Custom Headers**: Supports `ANTHROPIC_CUSTOM_HEADERS` env var for proxy configuration
- **Timeout Configuration**: Configurable via `API_TIMEOUT_MS` (default: 600s)
- **Proxy Support**: Integrated via `getProxyFetchOptions()`

---

## 2. Retry Logic (`withRetry.ts`)

### Retry Configuration

```typescript
const DEFAULT_MAX_RETRIES = 10
const BASE_DELAY_MS = 500
const MAX_529_RETRIES = 3  // Server overload threshold
```

### Retry Strategy

The retry system uses an **exponential backoff with jitter**:

```typescript
delay = min(BASE_DELAY_MS * 2^(attempt-1), maxDelayMs) + random(0, 0.25 * baseDelay)
```

### Retryable Errors

| Error Type | Status | Retry Behavior |
|------------|--------|----------------|
| Rate Limit | 429 | Retry with backoff (except for Claude AI subscribers) |
| Server Overload | 529 | Retry up to MAX_529_RETRIES, then fallback model |
| Timeout | 408 | Always retry |
| Lock Timeout | 409 | Always retry |
| Auth Error | 401/403 | Clear cache and retry once |
| Connection Error | ECONNRESET/EPIPE | Disable keep-alive and retry |
| Context Overflow | 400 | Adjust max_tokens and retry |

### Special Retry Modes

**Persistent Retry Mode** (`CLAUDE_CODE_UNATTENDED_RETRY`):
- For unattended/ant-only sessions
- Retries 429/529 indefinitely with higher backoff (up to 5min)
- Sends periodic heartbeat messages to prevent idle-kill
- Caps at 6-hour reset window

**Fast Mode Fallback**:
- On 429/529 with short retry-after (<20s): wait and retry with fast mode active
- On long retry-after: enter cooldown (10-30min), switch to standard speed

### Circuit Breaker Patterns

- **Model Fallback**: After MAX_529_RETRIES consecutive 529s on Opus, falls back to Sonnet
- **Authentication Refresh**: On 401/403 token revocation, triggers OAuth token refresh
- **Credential Cache Clearing**: AWS/GCP credential errors clear their respective caches

---

## 3. Error Classification (`errors.ts`)

### Error Types

The system classifies errors into user-facing categories:

| Error Category | Examples |
|----------------|----------|
| `rate_limit` | 429 quota exceeded, overage rejected |
| `authentication_failed` | Invalid API key, token revoked, OAuth errors |
| `invalid_request` | Prompt too long, invalid model, tool_use errors |
| `server_error` | 500s, 529 overloaded |
| `connection_error` | Network failures, SSL errors |
| `billing_error` | Credit balance too low |

### Error Message Generation

The `getAssistantMessageFromError()` function provides contextual error messages:

- **Prompt Too Long**: Extracts token counts, suggests compaction
- **Media Size Errors**: Image/PDF size limits with specific hints
- **Auth Errors**: Differentiates OAuth vs API key issues
- **Rate Limits**: Shows time until reset, suggests model switch

### Error Classification Function

```typescript
classifyAPIError(error: unknown): string
```

Returns standardized error types for analytics:
- `rate_limit`, `server_overload`, `prompt_too_long`, `image_too_large`
- `pdf_too_large`, `pdf_password_protected`, `tool_use_mismatch`
- `duplicate_tool_use_id`, `invalid_model`, `credit_balance_low`
- `invalid_api_key`, `token_revoked`, `oauth_org_not_allowed`
- `auth_error`, `bedrock_model_access`, `ssl_cert_error`

---

## 4. Rate Limiting (`claudeAiLimits.ts`)

### Rate Limit Types

```typescript
type RateLimitType =
  | 'five_hour'      // Session limit (5-hour window)
  | 'seven_day'      // Weekly limit
  | 'seven_day_opus' // Opus-specific weekly limit
  | 'seven_day_sonnet' // Sonnet-specific weekly limit
  | 'overage'        // Extra usage (pay-as-you-go)
```

### Header-Based Rate Limit Extraction

The system processes rate limit headers from API responses:

```
anthropic-ratelimit-unified-status: allowed | allowed_warning | rejected
anthropic-ratelimit-unified-representative-claim: five_hour | seven_day | overage
anthropic-ratelimit-unified-reset: <unix_timestamp>
anthropic-ratelimit-unified-fallback: available
anthropic-ratelimit-unified-overage-status: allowed | allowed_warning | rejected
anthropic-ratelimit-unified-overage-disabled-reason: <reason_code>
```

### Early Warning System

Two-tier early warning for approaching limits:

1. **Header-based**: Server sends `surpassed-threshold` header
2. **Time-relative fallback**: Client-side calculation based on utilization vs time elapsed

Threshold configs:
- **5-hour**: Warn at 90% utilization if <72% through window
- **7-day**: Multiple thresholds (75%/60%, 50%/35%, 25%/15%)

### Quota Status Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    allowed      │────→│ allowed_warning │────→│    rejected     │
│  (normal usage) │     │ (approaching    │     │  (limit hit,    │
│                 │     │    limit)       │     │  wait for reset)│
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                                               │
         │                    ┌──────────────────────────┘
         │                    │
         │                    ▼
         │           ┌─────────────────┐
         │           │   overage       │
         └──────────→│  (extra usage   │
                     │   if enabled)   │
                     └─────────────────┘
```

---

## 5. Request Building (`claude.ts`)

### Message Flow

```
┌─────────────────┐
│  User Messages  │
└────────┬────────┘
         ▼
┌─────────────────┐
│  stripExcess    │  ← Remove >100 media items
│  MediaItems     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ normalizeMsgs   │  ← Convert to API format
│    ForAPI       │
└────────┬────────┘
         ▼
┌─────────────────┐
│ensureToolResult │  ← Fix tool_use/tool_result pairing
│    Pairing      │
└────────┬────────┘
         ▼
┌─────────────────┐
│ addCacheBreak   │  ← Insert cache_control markers
│    points       │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Build System   │  ← Add system prompts, betas
│     Prompt      │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Send to API    │  ← via withRetry
└─────────────────┘
```

### Beta Headers

Dynamic beta header injection based on:

| Beta Header | Trigger |
|-------------|---------|
| `prompt-caching-2025-01-01` | When prompt caching enabled |
| `fast-mode-2025-02-01` | Fast mode active (latched) |
| `context-management-2025-01-01` | Extended context window |
| `afk-mode-2025-01-01` | Auto-mode active |
| `cache-editing-2025-01-01` | Cached microcompact enabled |
| `advisor-20260301` | Advisor tool enabled |
| `structured-outputs-2025-01-01` | JSON output format requested |

### Caching Strategy

- **Ephemeral caching**: Standard prompt caching
- **1-hour TTL**: Extended cache lifetime for eligible users
- **Global cache scope**: When MCP tools not present
- **Cache editing**: For cached microcompact feature

---

## 6. Error Handling Patterns (`errorUtils.ts`)

### Connection Error Extraction

Walks the error cause chain to extract root cause:

```typescript
extractConnectionErrorDetails(error: unknown): {
  code: string
  message: string
  isSSLError: boolean
}
```

### SSL Error Codes Tracked

- Certificate verification: `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `CERT_HAS_EXPIRED`
- Self-signed: `DEPTH_ZERO_SELF_SIGNED_CERT`, `SELF_SIGNED_CERT_IN_CHAIN`
- Hostname: `ERR_TLS_CERT_ALTNAME_INVALID`, `HOSTNAME_MISMATCH`
- Chain errors: `CERT_CHAIN_TOO_LONG`, `PATH_LENGTH_EXCEEDED`

### HTML Sanitization

Strips HTML error pages (e.g., CloudFlare) from error messages:

```typescript
sanitizeAPIError(apiError: APIError): string
```

---

## 7. Streaming Architecture

### Streaming Flow

```typescript
// 1. Create streaming request
const stream = await anthropic.beta.messages.create(
  { ...params, stream: true },
  { signal, headers: { [CLIENT_REQUEST_ID_HEADER]: clientRequestId } }
).withResponse()

// 2. Process chunks
for await (const part of stream.data) {
  switch (part.type) {
    case 'message_start': // Initialize usage tracking
    case 'content_block_start': // New content block
    case 'content_block_delta': // Incremental content
    case 'content_block_stop': // Block complete
    case 'message_delta': // Usage updates
    case 'message_stop': // Stream complete
  }
}
```

### Streaming Protections

- **Idle Timeout Watchdog**: Aborts if no chunks for 90s (configurable)
- **Stall Detection**: Logs gaps >30s between chunks
- **Resource Cleanup**: Explicitly cancels response body on abort/error

### Fallback to Non-Streaming

On certain streaming failures, falls back to non-streaming request:
- Timeout: 120s for CCR mode, 300s otherwise
- Triggers after streaming 529 errors exceed threshold

---

## 8. Key Design Patterns

### Generator-Based Async Flow

The retry system uses async generators to yield progress:

```typescript
async function* withRetry<T>(...): AsyncGenerator<SystemAPIErrorMessage, T>
```

This allows:
- Yielding retry messages to UI while waiting
- AbortSignal propagation through generator chain
- Clean resource cleanup on abort

### Context Passing

Retry context carries mutable state across attempts:

```typescript
interface RetryContext {
  maxTokensOverride?: number  // Adjusted on context overflow
  model: string
  thinkingConfig: ThinkingConfig
  fastMode?: boolean          // May be disabled on overage rejection
}
```

### Error Aggregation

Errors are aggregated through:
1. **SDK-level**: Network/connection errors
2. **Retry-level**: Transient failures with retry logic
3. **Application-level**: User-facing error messages with recovery hints

---

## 9. Configuration Environment Variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Direct API authentication |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token authentication |
| `CLAUDE_CODE_USE_BEDROCK` | Enable AWS Bedrock mode |
| `CLAUDE_CODE_USE_VERTEX` | Enable GCP Vertex mode |
| `CLAUDE_CODE_USE_FOUNDRY` | Enable Azure Foundry mode |
| `CLAUDE_CODE_MAX_RETRIES` | Override default retry count |
| `API_TIMEOUT_MS` | Request timeout |
| `CLAUDE_CODE_UNATTENDED_RETRY` | Enable persistent retry mode |
| `CLAUDE_CODE_EXTRA_BODY` | Additional API request parameters |
| `ANTHROPIC_CUSTOM_HEADERS` | Custom headers for proxy support |
| `DISABLE_PROMPT_CACHING` | Disable prompt caching |

---

## Summary

The Claude Code API Service demonstrates sophisticated error handling and resilience patterns:

1. **Multi-layered retry strategy** with provider-specific handling
2. **Intelligent error classification** for both analytics and user experience
3. **Graceful degradation** through model fallbacks and mode switching
4. **Comprehensive rate limit handling** with early warnings and overage support
5. **Streaming robustness** via watchdogs, stall detection, and non-streaming fallback
6. **Authentication flexibility** supporting OAuth, API keys, and cloud provider IAM

The architecture prioritizes user experience by providing actionable error messages, automatic recovery where possible, and clear feedback when manual intervention is needed.
