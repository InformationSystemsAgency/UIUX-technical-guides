# Logging and Error Handling Guide

[Back to README](./README.md)

This document describes the structured logging and error handling systems used across the [NewMoon](https://github.com/InformationSystemsAgency/newmoon) monorepo. Both `apps/api` (Hono) and `apps/web` (Nuxt/Nitro) produce machine-readable JSON logs via [Pino](https://github.com/pinojs/pino) and use shared error types for consistent error responses.

**These principles are mandatory for every project using our base stack.**
---

## Table of Contents

### Logging
1. [Design Principles](#1-design-principles)
2. [Log Format](#2-log-format)
3. [Logger Configuration](#3-logger-configuration)
4. [Event Constants](#4-event-constants)
5. [Event Type Classifications](#5-event-type-classifications)
6. [Request ID Propagation](#6-request-id-propagation)
7. [Severity Level Guidelines](#7-severity-level-guidelines)
8. [Per-App Logging Details](#8-per-app-logging-details)
9. [Adding a New Log Event](#9-adding-a-new-log-event)

### Error Handling
10. [Error Handling Architecture](#10-error-handling-architecture)
11. [AppError — Shared Error Class](#11-apperror--shared-error-class)
12. [API Error Handler](#12-api-error-handler)
13. [Web Error Handler](#13-web-error-handler)
14. [Error Response Format](#14-error-response-format)
15. [Error Types and When to Use Them](#15-error-types-and-when-to-use-them)
16. [Adding Error Handling to a New Module](#16-adding-error-handling-to-a-new-module)

### Reference
17. [Rules](#17-rules)

---

## 1. Design Principles

- **Structured JSON only** — No `console.log`, `console.error`, or `console.warn` in server-side code. The only exception is `apps/api/src/env.ts` which runs before the logger is initialised and terminates the process on failure.
- **Machine-readable** — Every log line is a single JSON object, parseable by log aggregators (ELK, CloudWatch, Datadog, etc.).
- **Correlated** — Every request carries a `requestId` (UUID v4) from the web layer through to the API layer, included in every log entry and every error response.
- **Classified** — Every log entry carries an `event` (what happened) and `event_type` (what kind of thing it is), enabling filtering, dashboards, and compliance reporting.
- **ISO 8601 timestamps** — All entries use `pino.stdTimeFunctions.isoTime` for unambiguous, timezone-aware timestamps.

---

## 2. Log Format

Every JSON log line contains these base fields:

```json
{
  "level": 30,
  "time": "2025-04-10T12:34:56.789Z",
  "name": "newmoon-api",
  "env": "production",
  "version": "1.3.3",
  "msg": "Human-readable message"
}
```

| Field     | Source                        | Description                                          |
|-----------|-------------------------------|------------------------------------------------------|
| `level`   | Pino                         | Numeric severity (10=trace, 20=debug, 30=info, 40=warn, 50=error, 60=fatal) |
| `time`    | Pino (`isoTime`)             | ISO 8601 timestamp                                   |
| `name`    | Logger config                | `newmoon-api` or `newmoon-web`                       |
| `env`     | `base` config                | `NODE_ENV` / `NEWMOON_API_NODE_ENV`                  |
| `version` | `base` config                | `APP_VERSION` env var, defaults to `"unknown"`        |
| `msg`     | Log call                     | Human-readable message                               |

Application-specific fields are added per log call:

```json
{
  "event": "auth.success",
  "event_type": "security",
  "requestId": "c8008a30-474a-4f38-a26b-930411de8ebe",
  "method": "POST",
  "url": "/v1/auth/callback",
  "clientIp": "127.0.0.1",
  "userId": "user-123",
  "status": 200,
  "responseTime": 47
}
```

---

## 3. Logger Configuration

### API — `apps/api/src/lib/utils/logging.ts`

```typescript
import pino from 'pino';
import Env from '@/env';

export const logger = pino({
  name: 'newmoon-api',
  level: Env.NEWMOON_API_LOG_LEVEL,
  timestamp: pino.stdTimeFunctions.isoTime,
  base: {
    env: Env.NEWMOON_API_NODE_ENV,
    version: process.env.APP_VERSION ?? 'unknown',
  },
  serializers: {
    err: pino.stdSerializers.err,
  },
});
```

The API logger is integrated into Hono via `hono-pino`. Each request gets a child logger accessible as `c.var.logger`, which automatically includes `requestId`, `method`, `url`, `clientIp`, `userId`, `status`, and `responseTime`.

Configuration in `apps/api/src/lib/middlewares/logger.ts`:

```typescript
pinoLogger({
  pino: logger,
  http: {
    referRequestIdKey: 'requestId',
    onReqBindings: (c) => ({
      method: c.req.method,
      url: c.req.path,
      clientIp: c.get('clientIp'),
      userId: c.get('jwtPayload')?.sub,
    }),
    onResBindings: (c) => {
      const stats = c.get('requestStats');
      const bindings: Record<string, unknown> = { status: c.res.status };
      if (stats?.cacheHit) bindings.cacheHit = stats.cacheHit;
      if (stats?.coalesced) bindings.coalesced = stats.coalesced;
      return bindings;
    },
  },
});
```

The response bindings include two optional counters that are only emitted when non-zero:

- **`cacheHit`** — number of times the request was served from the `createCoalescedFetcher` TTL cache (per call site).
- **`coalesced`** — number of times the request joined an existing in-flight promise instead of issuing its own upstream call.

Both are populated by the coalescer reading a mutable `RequestStats` object stashed on the per-request `AsyncLocalStorage` store (see [Section 6](#6-request-id-propagation)). A single request can produce a log line like `{ "status": 200, "cacheHit": 3, "coalesced": 1 }` when several `coalescedContent(...)` / `coalescedLayout(...)` calls participated.

### Web — `apps/web/server/utils/logger.ts`

```typescript
import pino from 'pino';

export const logger = pino({
  name: 'newmoon-web',
  level: process.env.LOG_LEVEL ?? 'info',
  timestamp: pino.stdTimeFunctions.isoTime,
  base: {
    env: process.env.NODE_ENV ?? 'production',
    version: process.env.APP_VERSION ?? 'unknown',
  },
  serializers: {
    err: pino.stdSerializers.err,
  },
});
```

The web logger is a standalone Pino instance imported directly in Nitro server middleware, plugins, and route handlers. There is no per-request child logger — `requestId` is passed explicitly from `event.context.requestId`.

**Cache-hit tracking on `HTTP_REQUEST`** — Nitro handlers that wrap their work with `cacheManager.withCache(...)` can pass an `onResult({ fromCache })` callback that writes to `event.context.cacheHit`. The request logger middleware (`server/middleware/request-logger.ts`) emits the `cacheHit` field on the `HTTP_REQUEST` log line whenever it is defined, so it only appears for routes that participate in this protocol. Current participants:

- `server/api/layout/[[...path]].ts` — sets `cacheHit` from the layout cacheManager lookup.
- `server/api/code-highlighter.post.ts` — sets `cacheHit` from the code-highlighter cacheManager lookup.
- `server/api/v1/[...path].ts` — always sets `cacheHit = false` because the proxy never caches at the Nitro layer (the API's own coalescer handles caching on the other side).

Unlike the API's counter-style `cacheHit` (number of cached sub-calls inside one request), the web's `cacheHit` is a single boolean: did this Nitro response come from the in-memory cacheManager or was it regenerated. Routes that don't use `cacheManager.withCache` simply omit the field.

---

## 4. Event Constants

All event names live in `packages/shared/src/constants/log-fields.ts` and are exported as `LogEvent`. Never hardcode event strings — always import from `@newmoon/shared` (or `@/lib/utils/log-fields` in the API).

Format: `<domain>.<action>` — lowercase, dot-separated.

| Constant | Value | Used for |
|----------|-------|----------|
| **HTTP request lifecycle** | | |
| `HTTP_REQUEST` | `http.request` | Every incoming web request (method, path, status, duration) |
| **System lifecycle** | | |
| `SYSTEM_STARTUP` | `system.startup` | Server start, env validation |
| `SYSTEM_SHUTDOWN` | `system.shutdown` | Graceful shutdown (SIGTERM/SIGINT) |
| `SYSTEM_DEPENDENCY_READY` | `system.dependency_ready` | Dependency connection validated on startup |
| `SYSTEM_DEPENDENCY_FAILED` | `system.dependency_failed` | Dependency connection failed on startup |
| **Scheduled tasks** | | |
| `TASK_START` | `task.start` | Scheduled task begins |
| `TASK_SUCCESS` | `task.success` | Scheduled task completes |
| `TASK_ERROR` | `task.error` | Scheduled task fails |
| **Authentication** | | |
| `AUTH_INITIATE` | `auth.initiate` | OAuth flow started |
| `AUTH_SUCCESS` | `auth.success` | User authenticated |
| `AUTH_FAILURE` | `auth.failure` | Auth callback failed (missing cookies, token exchange error) |
| `AUTH_LOGOUT` | `auth.logout` | User logged out |
| `AUTH_JWT_INVALID` | `auth.jwt_invalid` | JWT validation failed on optional auth route |
| `AUTH_MOCK_TOKEN_ISSUED` | `auth.mock_token_issued` | Mock token issued in test mode |
| **Data access** | | |
| `ACCESS_SERVICE_SUBMISSION` | `access.service_submission` | Service form submitted |
| `ACCESS_SERVICE_SUBMISSION_ERROR` | `access.service_submission_error` | Service submission failed |
| `ACCESS_SEARCH` | `access.search` | Search query executed |
| `ACCESS_DATA_DISPLAY_ERROR` | `access.data_display_error` | Data display fetch failed |
| **Administrative** | | |
| `ADMIN_CACHE_CLEAR` | `admin.cache_clear` | Cache cleared by needle |
| `ADMIN_CACHE_STATS` | `admin.cache_stats` | Cache stats requested |
| `ADMIN_CACHE_MANUAL_CLEAR` | `admin.cache_manual_clear` | Full cache clear |
| `ADMIN_SEARCH_INDEX_REBUILD` | `admin.search_index_rebuild` | Search index rebuilt |
| `ADMIN_SEARCH_INDEX_UPDATE` | `admin.search_index_update` | Search index updated |
| `ADMIN_SEARCH_INDEX_DELETE` | `admin.search_index_delete` | Search index entry deleted |
| **Application errors** | | |
| `APP_ERROR` | `app.error` | General application error (the catch-all) |
| `APP_CODE_HIGHLIGHT_ERROR` | `app.code_highlight_error` | Code highlighter endpoint failed |
| `APP_SITEMAP_FAILED` | `app.sitemap_failed` | Sitemap generation failed |
| **Integration errors** | | |
| `INTEGRATION_ERROR` | `integration.error` | Generic external service failure |
| `INTEGRATION_MEILISEARCH_ERROR` | `integration.meilisearch_error` | Meilisearch call failed |
| `INTEGRATION_CMS_ERROR` | `integration.cms_error` | Directus CMS call failed |
| `INTEGRATION_YESEM_TOKEN_EXCHANGE_ERROR` | `integration.yesem_token_exchange_error` | YesEm token exchange failed |
| `INTEGRATION_YESEM_DISCOVERY_ERROR` | `integration.yesem_discovery_error` | YesEm OIDC discovery failed |
| `INTEGRATION_GIP_REQUEST_FAILED` | `integration.gip_request_failed` | GIP request failed |
| `INTEGRATION_DIRECTUS_REQUEST` | `integration.directus_request` | Outbound Directus HTTP call (method, url, status, durationMs) |
| `INTEGRATION_MEILISEARCH_REQUEST` | `integration.meilisearch_request` | Outbound Meilisearch HTTP call (method, url, status, durationMs) |
| `INTEGRATION_HEALTH_CHECK_DEGRADED` | `integration.health_check_degraded` | Health check dependency down |
| `INTEGRATION_API_PROXY_ERROR` | `integration.api_proxy_error` | Web→API proxy error |
| `INTEGRATION_LAYOUT_FETCH_ERROR` | `integration.layout_fetch_error` | Layout data fetch failed |
| `INTEGRATION_RSS_FETCH_ERROR` | `integration.rss_fetch_error` | RSS feed fetch failed |
| `INTEGRATION_SITEMAP_FETCH_ERROR` | `integration.sitemap_fetch_error` | Sitemap data fetch failed |
| `INTEGRATION_REALTIME_ERROR` | `integration.realtime_error` | Realtime/WebSocket connection error |
| **Realtime / WebSocket** | | |
| `REALTIME_CONNECTION_OPEN` | `realtime.connection_open` | WebSocket client connected |
| `REALTIME_CONNECTION_CLOSE` | `realtime.connection_close` | WebSocket client disconnected (with `code`, `reason`) |
| `REALTIME_MESSAGE_RECEIVED` | `realtime.message_received` | WebSocket message received (debug, with `size`) |
| **Memory monitoring** | | |
| `MEMORY_MONITOR` | `memory.monitor` | Memory threshold warning |
| `MEMORY_STATS` | `memory.stats` | Periodic memory statistics |
| `MEMORY_CACHE_CLEANUP` | `memory.cache_cleanup` | Expired cache entries removed |
| `MEMORY_CACHE_EVICTION` | `memory.cache_eviction` | Cache entry evicted (debug) |
| **Client-side** | | |
| `CLIENT_ERROR_REPORT` | `client.error_report` | Client-side error reported via `POST /v1/client-errors` |

---

## 5. Event Type Classifications

Every log entry includes an `event_type` field classifying what kind of event it is. These are defined in `LogEventType` in `packages/shared/src/constants/log-fields.ts`.

| Type | Value | When to use |
|------|-------|-------------|
| `OPERATIONAL` | `operational` | System lifecycle, HTTP requests, scheduled tasks, memory stats |
| `SECURITY` | `security` | Authentication events (login, logout, JWT validation, token issuance) |
| `ACCESS` | `access` | User-initiated data access (search, form submissions) |
| `ADMINISTRATIVE` | `administrative` | Admin actions (cache clear, search index management) |
| `APPLICATION_FAILURE` | `application-failure` | Errors — application bugs, integration failures, unhandled exceptions |
| `AUDIT` | `audit` | Compliance-sensitive events requiring audit trail |

These classifications enable:
- Filtering security events for incident response
- Separating operational noise from application errors
- Compliance reporting on administrative and audit events

---

## 6. Request ID Propagation

A UUID v4 `requestId` is generated at the web layer and propagated end-to-end:

```
Browser → Nuxt/Nitro (generate requestId)
              │
              ├─ SSR rendering (requestId in event.context)
              │
              └─ Proxy to API (/api/v1/*, /api/layout/*)
                    │
                    └─ X-Request-Id header
                           │
                           └─ Hono API (reads header via requestId() middleware)
                                  │
                                  ├─ c.get('requestId') — available in all handlers
                                  ├─ c.var.logger — child logger with requestId bound
                                  └─ Error responses include requestId in JSON body
```

### Web layer

**Middleware** (`apps/web/server/middleware/request-id.ts`):
- Reads `X-Request-Id` from incoming request header (if present, e.g. from a load balancer)
- Falls back to generating `crypto.randomUUID()`
- Sets `event.context.requestId` and `X-Request-Id` response header

**Proxy routes** (`apps/web/server/api/v1/[...path].ts`, `apps/web/server/api/layout/[[...path]].ts`):
- Forward `X-Request-Id` header on all outgoing fetch calls to the API

### API layer

**Hono middleware** (`hono/request-id`):
- Reads `X-Request-Id` from incoming request header
- Falls back to generating `crypto.randomUUID()`
- Sets `c.var.requestId` and `X-Request-Id` response header

**hono-pino** (`referRequestIdKey: 'requestId'`):
- Automatically includes `reqId` in every log entry from the per-request child logger

### Upstream forwarding (API → external services)

The API propagates `requestId` to all outbound HTTP calls via `AsyncLocalStorage`:

**Request context** (`apps/api/src/lib/utils/request-context.ts`):
- An `AsyncLocalStorage<{ requestId: string }>` store populated by middleware in `create-app.ts`
- Runs after `requestId()` middleware so the ID is always available

**Instrumented clients** read the store and inject `X-Request-Id` on every outbound request:

| Client | File | Header injected |
|--------|------|----------------|
| Directus SDK | `src/modules/directus/directus.service.ts` | `X-Request-Id` via custom `globals.fetch` |
| Meilisearch | `src/modules/search/search.service.ts` | `X-Request-Id` via custom `httpClient` |
| GIP | `src/lib/utils/gip.ts` | `X-Request-Id` in fetch headers |

This means a single `requestId` flows from the browser through the web proxy, the API, and into Directus/Meilisearch/GIP — enabling full end-to-end trace correlation.

### Error responses

All API error responses include `requestId` in the JSON body:

```json
{
  "code": "NOT_FOUND",
  "message": "Route not found.",
  "requestId": "c8008a30-474a-4f38-a26b-930411de8ebe"
}
```

This allows users/support to report the `requestId` for correlation with server logs.

---

## 7. Severity Level Guidelines

| Level | Pino value | When to use |
|-------|------------|-------------|
| `fatal` | 60 | Process is about to crash — never recovered |
| `error` | 50 | 5xx responses, unhandled exceptions, integration failures that block the request |
| `warn` | 40 | 4xx client errors, recoverable integration issues, degraded dependencies, JWT validation failures |
| `info` | 30 | Successful operations: startup, shutdown, auth success, search executed, cache cleared, HTTP requests |
| `debug` | 20 | Cache eviction details, outbound HTTP timing (Directus/Meilisearch), verbose diagnostics (off in production by default) |
| `trace` | 10 | Not currently used |

**Rule of thumb for HTTP status → log level:**
- `2xx`/`3xx` → `info`
- `4xx` → `warn`
- `5xx` → `error`

---

## 8. Per-App Logging Details

### API (`apps/api`)

| Component | File | What it logs |
|-----------|------|-------------|
| Server lifecycle | `src/index.ts` | `SYSTEM_STARTUP`, `SYSTEM_SHUTDOWN`, `TASK_START/SUCCESS/ERROR` |
| HTTP request/response | `src/lib/middlewares/logger.ts` | Automatic via hono-pino (method, url, status, responseTime, reqId). Response side adds optional `cacheHit` / `coalesced` counters when non-zero. |
| Error handler | `src/lib/errors/utils.ts` | `APP_ERROR` with errorCode, requestId, stack |
| Auth handlers | `src/modules/auth/auth.handlers.ts` | `AUTH_INITIATE`, `AUTH_SUCCESS`, `AUTH_FAILURE`, `AUTH_LOGOUT`, `AUTH_MOCK_TOKEN_ISSUED` |
| Auth middleware | `src/lib/middlewares/auth.ts` | `AUTH_JWT_INVALID` |
| Auth service | `src/modules/auth/auth.service.ts` | `INTEGRATION_YESEM_*` |
| Search handlers | `src/modules/search/search.handlers.ts` | `ACCESS_SEARCH`, `ADMIN_SEARCH_INDEX_*` |
| Search service | `src/modules/search/search.service.ts` | `INTEGRATION_MEILISEARCH_ERROR` on query/filter failures with structured context (`indexName`, `query`, `filters`) |
| Services handlers | `src/modules/services/services.handlers.ts` | `ACCESS_SERVICE_SUBMISSION` |
| Services service | `src/modules/services/services.service.ts` | `ACCESS_SERVICE_SUBMISSION_ERROR`, `INTEGRATION_CMS_ERROR` on Directus submission failures |
| Feed service | `src/modules/feed/feed.service.ts` | `INTEGRATION_CMS_ERROR` on feed/RSS upstream failures with requestId and duration |
| Data display | `src/modules/data-display/data-display.service.ts` | `APP_ERROR` on fetch failure |
| Layout globals | `src/modules/layout/layout-globals.ts` | `INTEGRATION_CMS_ERROR` |
| Realtime events | `src/modules/realtime/realtime.events.ts` | `REALTIME_CONNECTION_OPEN`, `REALTIME_CONNECTION_CLOSE`, `REALTIME_MESSAGE_RECEIVED`, `INTEGRATION_REALTIME_ERROR` |
| GIP client | `src/lib/utils/gip.ts` | `INTEGRATION_GIP_REQUEST_FAILED` |
| Health routes | `src/modules/health/health.routes.ts` | `INTEGRATION_HEALTH_CHECK_DEGRADED` |
| Scheduled publish | `src/modules/scheduled-publish/scheduled-publish.service.ts` | `TASK_*` events |
| Directus client | `src/modules/directus/directus.service.ts` | `INTEGRATION_DIRECTUS_REQUEST` (debug, warn on failure) |
| Directus validation | `src/modules/directus/directus.validation.ts` | `SYSTEM_DEPENDENCY_READY`, `SYSTEM_DEPENDENCY_FAILED` |
| Meilisearch client | `src/modules/search/search.service.ts` | `INTEGRATION_MEILISEARCH_REQUEST` (debug, warn on failure) |
| Meilisearch validation | `src/modules/directus/directus.validation.ts` | `SYSTEM_DEPENDENCY_READY`, `SYSTEM_DEPENDENCY_FAILED` |
| Client error reporting | `src/modules/client-errors/client-errors.routes.ts` | `CLIENT_ERROR_REPORT` (warn) |

### Web (`apps/web`)

| Component | File | What it logs |
|-----------|------|-------------|
| Request logging | `server/middleware/request-logger.ts` | `HTTP_REQUEST` (method, path, status, durationMs, requestId, optional `cacheHit` when the handler opted in) |
| Error handler | `server/plugins/02.error-handler.ts` | `APP_ERROR` on unhandled errors |
| Cache control (legacy handler) | `server/cache-control/cache-control-api-handler.ts` | `ADMIN_CACHE_CLEAR`, `ADMIN_CACHE_MANUAL_CLEAR`, `ADMIN_CACHE_STATS` |
| Cache clear endpoint | `server/api/cache-clear.post.ts` | `ADMIN_CACHE_MANUAL_CLEAR` at info on success, warn on validation failure, error on exception (with `requestId`, pattern/target) |
| Cache stats endpoint | `server/api/cache-stats.get.ts` | `ADMIN_CACHE_STATS` at info on successful stat dump, error on exception |
| Cache manager | `server/cache-control/cache-manager.ts` | `MEMORY_CACHE_CLEANUP`, `MEMORY_CACHE_EVICTION` |
| Memory monitor | `server/utils/memory-monitor.ts` | `MEMORY_MONITOR`, `MEMORY_STATS` |
| RSS | `server/routes/rss.xml.ts` | `INTEGRATION_RSS_FETCH_ERROR` |
| Sitemap | `server/routes/sitemap.xml.ts` | `INTEGRATION_SITEMAP_FETCH_ERROR`, `APP_SITEMAP_FAILED` |
| Code highlighter | `server/api/code-highlighter.post.ts` | `APP_CODE_HIGHLIGHT_ERROR` on invalid body, missing language, and highlighter failure (each with `requestId`, `language`, `theme`) |
| Env validation | `server/utils/env-schema.ts` | `SYSTEM_STARTUP` on validation failure |
| Client error sink | `server/api/client-log.post.ts` | `CLIENT_ERROR_REPORT` at the reported level (info/warn/error) with `source: 'browser'`, `clientUrl`, `clientMethod`, `statusCode`, `stack`, `userAgent`, `route`, `context`, `requestId` |
| Client error helper | `app/utils/client-logger.ts` | No direct logging — `reportClientError()` forwards browser errors to `/api/client-log` via `sendBeacon` (fallback: `fetch({ keepalive: true })`); SSR path logs to `console.error` |
| Auth store | `app/stores/auth.ts` | `reportClientError()` on SSO init / token refresh / logout failures |
| Special store | `app/stores/special.ts` | `reportClientError()` on content fetch failures |
| Service store | `app/stores/service.ts` | `reportClientError()` on service form submission failures |
| Auth middleware | `app/middleware/auth.ts` | `reportClientError()` on SSO preflight failure |
| useSearch composable | `app/composables/useSearch.ts` | `reportClientError()` on client-initiated search failures |
| useContentDisplay composable | `app/composables/useContentDisplay.ts` | `reportClientError()` on data-display fetch failures |

### Outbound HTTP Instrumentation

Every outbound call to Directus and Meilisearch is automatically instrumented with timing and `X-Request-Id` forwarding. This is done via custom fetch wrappers injected into the SDK clients at instantiation time.

**Directus** (`apps/api/src/modules/directus/directus.service.ts`):
- Uses `globals.fetch` in `createDirectus()` options
- Logs `INTEGRATION_DIRECTUS_REQUEST` at `debug` level (includes method, url, status, durationMs)
- Logs at `warn` level if the request fails

**Meilisearch** (`apps/api/src/modules/search/search.service.ts`):
- Uses `httpClient` in `MeiliSearch` config
- Logs `INTEGRATION_MEILISEARCH_REQUEST` at `debug` level (includes method, url, status, durationMs)
- Logs at `warn` level if the request fails

**GIP** (`apps/api/src/lib/utils/gip.ts`):
- Injects `X-Request-Id` header from `AsyncLocalStorage` context

Set `NEWMOON_API_LOG_LEVEL=debug` to see outbound HTTP timing in development.

### Startup Readiness Logging

On startup, the API validates connections to external dependencies before accepting traffic. Results are logged with structured events:

**Directus** — `validateDirectusConnection()` in `apps/api/src/modules/directus/directus.validation.ts`:
- Calls `GET /server/health` on the Directus instance
- Logs `SYSTEM_DEPENDENCY_READY` (info) on success with `durationMs`
- Logs `SYSTEM_DEPENDENCY_FAILED` (error) on failure with error details

**Meilisearch** — `validateMeilisearchConnection()` in `apps/api/src/modules/directus/directus.validation.ts`:
- Calls `GET /health` on the Meilisearch instance (5s timeout)
- Logs `SYSTEM_DEPENDENCY_READY` (info) on success with `durationMs`
- Logs `SYSTEM_DEPENDENCY_FAILED` (error) on failure with error details

Both are called from `apps/api/src/index.ts` at server startup.

### Client-Side Error Reporting

The API exposes `POST /v1/client-errors` for browser-side error reporting. This allows frontend JavaScript errors to be captured in the same structured logging pipeline as server-side errors.

**Route**: `apps/api/src/modules/client-errors/client-errors.routes.ts`

**Request body** (validated by Zod):

| Field | Type | Required | Max length |
|-------|------|----------|------------|
| `message` | `string` | Yes | 2000 |
| `source` | `string` | No | 500 |
| `stack` | `string` | No | 5000 |
| `url` | `string` | No | 2000 |
| `userAgent` | `string` | No | 500 |
| `metadata` | `Record<string, unknown>` | No | — |

**Response**: `204 No Content`

**Logging**: Each report is logged at `warn` level with event `CLIENT_ERROR_REPORT` and event type `APPLICATION_FAILURE`. The full error body is included in the `clientError` field of the structured log entry.

### Web-side Client Error Reporting

The Nuxt app ships its own client-error pipeline that stays within the web process instead of hopping over to the API. Browser code calls `reportClientError()` which sends a structured payload to a Nitro sink at `/api/client-log`; the sink logs it under `CLIENT_ERROR_REPORT` using the same pino instance as the rest of the server-side logs. Use this pipeline for frontend issues whose surrounding context (route, store state, composable) belongs to the web app — not the API.

```
Browser (Vue code — stores / composables / middleware / plugins)
    │
    ▼
reportClientError(entry)   ← app/utils/client-logger.ts
    │
    ├── sendBeacon('/api/client-log', <JSON blob>)        (preferred)
    └── fetch('/api/client-log', { method, body, keepalive: true })  (fallback)
            │
            ▼
Nitro: POST /api/client-log   ← server/api/client-log.post.ts
            │
            ▼
logger[level]({ event: LogEvent.CLIENT_ERROR_REPORT, event_type: APPLICATION_FAILURE, … }, message)
```

**Helper** — `apps/web/app/utils/client-logger.ts`:

- `reportClientError(entry: ClientLogEntry)` — call from any client-side code. On SSR it falls back to `console.error` (so it's safe to invoke from universal code). On the client it mirrors the message to the devtools console at the matching level, then fires the beacon. Reporting failures are swallowed — this helper must never throw.
- `describeError(err: unknown): { message, stack? }` — normalizes arbitrary thrown values, VueUse fetch error refs, and `Error` instances into a consistent `{ message, stack }` shape for the entry.
- `ClientLogEntry` fields: `level` (`'error' | 'warn' | 'info'`, default `'error'`), `message`, `stack`, `statusCode`, `url`, `method`, `context`. The helper auto-attaches `userAgent` (`navigator.userAgent`) and `route` (`window.location.pathname`).

**Sink** — `apps/web/server/api/client-log.post.ts`:

- Reads JSON body; on parse failure logs `CLIENT_ERROR_REPORT` at `warn` and returns a 400.
- Clamps the level to `['error', 'warn', 'info']` (default `'error'`) and falls back to `'Client error reported'` if `message` is absent.
- Emits a single `logger[level](payload, message)` call with `event: CLIENT_ERROR_REPORT`, `event_type: APPLICATION_FAILURE`, `source: 'browser'`, and `requestId` pulled from `event.context.requestId`. Structured fields: `clientUrl`, `clientMethod`, `statusCode`, `stack`, `userAgent` (body-supplied or `User-Agent` header fallback), `route`, `context`.
- Returns `{ ok: true }`.

**Why two client-error sinks exist**: the API's `POST /v1/client-errors` is the right sink when the browser has already authenticated with the API and the failure belongs to an API call — it keeps the error stream next to the request it describes. The web sink is the right choice for purely frontend failures (router guards, store hydration, composables that haven't hit the API yet) where we don't want a cross-service hop and `requestId` is the web request, not the API request.

---

## 9. Adding a New Log Event

1. **Add the constant** to `packages/shared/src/constants/log-fields.ts` under the appropriate category:
   ```typescript
   // Integration errors
   INTEGRATION_NEW_SERVICE_ERROR: 'integration.new_service_error',
   ```

2. **Rebuild shared**:
   ```bash
   pnpm build
   ```

3. **Use it in your code**:
   ```typescript
   // In API (via re-export shim)
   import { LogEvent, LogEventType } from '@/lib/utils/log-fields';

   // In web (direct import)
   import { LogEvent, LogEventType } from '@newmoon/shared';

   logger.error(
     {
       event: LogEvent.INTEGRATION_NEW_SERVICE_ERROR,
       event_type: LogEventType.APPLICATION_FAILURE,
       requestId,
       err,
     },
     'New service call failed',
   );
   ```

4. **Choose the right `event_type`** — see [section 5](#5-event-type-classifications).

5. **Choose the right severity** — see [section 7](#7-severity-level-guidelines).

---

## 10. Error Handling Architecture

Error handling is layered across the monorepo:

```
packages/shared
  └─ AppError class          (shared error type with codes + status mapping)
  └─ ErrorCode constants      (BAD_REQUEST, NOT_FOUND, etc.)

apps/api
  └─ handleError()            (global Hono error handler — catches all thrown errors)
  └─ handleZodError()         (OpenAPI validation hook — catches Zod schema errors)
  └─ ErrorCodes / statusToCode (API-specific error code enum + status mapping)

apps/web
  └─ 02.error-handler.ts     (Nitro plugin — catches unhandled server errors)
  └─ createError()            (H3 built-in — used in Nitro routes/middleware)
```

The design ensures:
- Every error produces a structured JSON response with `code`, `message`, and `requestId`
- Every error is logged with severity based on HTTP status (4xx = warn, 5xx = error)
- Error types are consistent between API and web via the shared `AppError` class
- Stack traces are captured in logs but never leaked to the client

---

## 11. AppError — Shared Error Class

Defined in `packages/shared/src/errors/app-error.ts`. This is the preferred way to throw errors in business logic.

### Error Codes

| Code | HTTP Status | Human Name |
|------|-------------|------------|
| `BAD_REQUEST` | 400 | Bad Request |
| `UNAUTHORIZED` | 401 | Unauthorized |
| `FORBIDDEN` | 403 | Forbidden |
| `NOT_FOUND` | 404 | Not Found |
| `METHOD_NOT_ALLOWED` | 405 | Method Not Allowed |
| `CONFLICT` | 409 | Conflict |
| `UNPROCESSABLE_ENTITY` | 422 | Unprocessable Entity |
| `TOO_MANY_REQUESTS` | 429 | Too Many Requests |
| `INTERNAL_SERVER_ERROR` | 500 | Internal Server Error |
| `BAD_GATEWAY` | 502 | Bad Gateway |
| `SERVICE_UNAVAILABLE` | 503 | Service Unavailable |
| `GATEWAY_TIMEOUT` | 504 | Gateway Timeout |

### Usage

```typescript
import { AppError } from '@newmoon/shared';

// Static factories (preferred — concise and readable)
throw AppError.badRequest('Missing required field: email');
throw AppError.unauthorized('Session expired');
throw AppError.forbidden('Insufficient permissions');
throw AppError.notFound('Article not found');
throw AppError.conflict('Username already taken');
throw AppError.internal('Unexpected processing error');
throw AppError.serviceUnavailable('CMS is temporarily unavailable');
throw AppError.gatewayTimeout('Upstream service did not respond');

// Manual construction (for codes without a static factory)
throw new AppError('METHOD_NOT_ALLOWED', 'PATCH is not supported on this resource');
throw new AppError('UNPROCESSABLE_ENTITY', 'Cannot publish article without a title');
throw new AppError('TOO_MANY_REQUESTS', 'Rate limit exceeded');
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `code` | `ErrorCodeValue` | Machine-readable code (e.g. `'NOT_FOUND'`) |
| `status` | `number` | HTTP status (e.g. `404`) |
| `errorName` | `string` | Human-readable name (e.g. `'Not Found'`) |
| `message` | `string` | Custom error message passed at construction |

### Helper: `statusToErrorCode(status)`

Converts a numeric HTTP status back to an `ErrorCodeValue`. Used when wrapping errors from external services:

```typescript
import { statusToErrorCode } from '@newmoon/shared';

const code = statusToErrorCode(res.status); // e.g. 404 → 'NOT_FOUND'
```

---

## 12. API Error Handler

Located in `apps/api/src/lib/errors/utils.ts`. Registered as `app.onError(handleError)` in `apps/api/src/lib/create-app.ts`.

### `handleError(err, c)`

The global catch-all for every error thrown in a Hono handler or middleware. It determines the error type, logs it, and returns a structured JSON response.

**Processing order:**

| Priority | Error Type | Status | Log Level | Notes |
|----------|-----------|--------|-----------|-------|
| 1 | `ZodError` | 400 | `warn` | Validation failures from Zod schemas. Issues are flattened into a readable message. |
| 2 | `AppError` | From code | `warn` (4xx) / `error` (5xx) | The preferred error type. Status is determined by `ErrorCode`. |
| 3 | `HTTPException` | From exception | `warn` (4xx) / `error` (5xx) | Hono's built-in error type. Kept for backward compatibility. |
| 4 | Unknown `Error` | 500 | `error` | Catch-all for anything else. Treated as an internal server error. |

**Every error log includes:**
```json
{
  "event": "app.error",
  "event_type": "application-failure",
  "errorCode": "NOT_FOUND",
  "requestId": "c8008a30-...",
  "err": { "type": "AppError", "message": "...", "stack": "..." }
}
```

**Every error response includes:**
```json
{
  "code": "NOT_FOUND",
  "message": "Article not found",
  "requestId": "c8008a30-..."
}
```

### `handleZodError(result, c)`

Registered as the `defaultHook` on the OpenAPI router. Catches validation errors from `@hono/zod-openapi` route schemas (query params, path params, request body) before the handler runs.

Returns the same `{ code, message, requestId }` format with status 400.

### `createErrorSchema(code)`

A Zod schema factory for OpenAPI documentation. Used in route definitions to describe error responses:

```typescript
import { createErrorSchema } from '@/lib/errors';

// In openapi route definition
responses: {
  404: {
    description: 'Not found',
    content: {
      'application/json': { schema: createErrorSchema('NOT_FOUND') },
    },
  },
}
```

### `isErrorSchema(error)`

Type guard to check if an unknown value matches the error response shape. Useful when handling upstream API responses:

```typescript
if (isErrorSchema(response)) {
  // response has { code, message, requestId }
}
```

---

## 13. Web Error Handler

Located in `apps/web/server/plugins/02.error-handler.ts`.

Hooks into Nitro's error lifecycle via `nitroApp.hooks.hook('error', ...)`. Catches any unhandled error from Nitro routes, middleware, or server plugins.

```typescript
nitroApp.hooks.hook('error', (error, { event }) => {
  const requestId = event?.context?.requestId;
  const status = error?.statusCode ?? 500;
  const logLevel = status >= 500 ? 'error' : 'warn';

  logger[logLevel]({
    event: LogEvent.APP_ERROR,
    event_type: LogEventType.APPLICATION_FAILURE,
    requestId,
    statusCode: status,
    err: error,
  }, `Unhandled server error: ${error.message}`);
});
```

For intentional errors in Nitro routes, use H3's `createError()`:

```typescript
// In a Nitro route handler
throw createError({ statusCode: 400, message: 'Missing required parameter' });
throw createError({ statusCode: 404, message: 'Page not found' });
```

---

## 14. Error Response Format

### API responses

All API errors return this JSON structure:

```json
{
  "code": "NOT_FOUND",
  "message": "Article not found",
  "requestId": "c8008a30-474a-4f38-a26b-930411de8ebe"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `code` | `string` | Machine-readable error code from `ErrorCodes` enum |
| `message` | `string` | Human-readable explanation |
| `requestId` | `string` | UUID for log correlation |

The `code` field is one of: `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `METHOD_NOT_ALLOWED`, `CONFLICT`, `UNPROCESSABLE_ENTITY`, `TOO_MANY_REQUESTS`, `INTERNAL_SERVER_ERROR`, `PAYMENT_REQUIRED`.

### Web proxy errors

When the web proxy (`/api/v1/*`) receives a non-OK response from the API, it throws an H3 error with the API's status code and message. The original `requestId` is preserved since both layers share the same `X-Request-Id`.

---

## 15. Error Types and When to Use Them

### Decision tree

```
Is this a domain/business rule violation?
  └─ Yes → throw AppError.xxx('message')
  
Is this a validation error caught by Zod?
  └─ Handled automatically by handleError / handleZodError

Is this in a Nitro route (web server)?
  └─ Yes → throw createError({ statusCode, message })

Is this an external service failure?
  └─ Log with INTEGRATION_* event, then throw AppError.serviceUnavailable / .gatewayTimeout / .internal

Is this in legacy code using HTTPException?
  └─ Works fine — handled by handleError. Prefer AppError for new code.
```

### When to use `AppError` vs `HTTPException`

| | `AppError` | `HTTPException` |
|---|-----------|----------------|
| **Package** | `@newmoon/shared` | `hono/http-exception` |
| **Typed codes** | Yes (`ErrorCodeValue`) | No (raw number) |
| **Status mapping** | Automatic from code | Manual |
| **Static factories** | `.notFound()`, `.badRequest()`, etc. | None |
| **Use in web** | Yes (shared package) | No (Hono-only) |
| **Recommendation** | Preferred for new code | Acceptable in existing code |

### Common patterns

**Service not found:**
```typescript
const service = await fetchService(id);
if (!service) {
  throw AppError.notFound(`Service with ID ${id} not found`);
}
```

**Authentication required:**
```typescript
if (!token) {
  throw AppError.unauthorized('Authentication is required for this service');
}
```

**External service failure:**
```typescript
try {
  const data = await externalApi.fetch();
} catch (err) {
  logger.error({
    event: LogEvent.INTEGRATION_ERROR,
    event_type: LogEventType.APPLICATION_FAILURE,
    err,
  }, 'External API call failed');
  throw AppError.serviceUnavailable('External service is temporarily unavailable');
}
```

**API key validation:**
```typescript
if (body.apiKey !== Env.NEWMOON_API_SYSTEM_TOKEN) {
  throw AppError.forbidden('Invalid API key');
}
```

---

## 16. Adding Error Handling to a New Module

When creating a new API module, follow this pattern:

### 1. Service layer — throw errors

```typescript
// modules/my-feature/my-feature.service.ts
import { AppError } from '@newmoon/shared';

export async function getItem(id: string) {
  const item = await fetchFromCMS(id);
  if (!item) {
    throw AppError.notFound(`Item ${id} not found`);
  }
  return item;
}
```

### 2. Handler — let errors propagate

Handlers should not try/catch unless they need to add context. The global `handleError` catches everything:

```typescript
// modules/my-feature/my-feature.handlers.ts
export const getItemHandler: ApiRouteHandler<typeof getItemRoute> = async (c) => {
  const { id } = c.req.valid('param');
  const item = await getItem(id);  // AppError.notFound propagates to handleError
  return c.json(itemResponse.parse(item), 200);
};
```

### 3. OpenAPI route — document error responses

```typescript
// modules/my-feature/my-feature.openapi.ts
import { createErrorSchema } from '@/lib/errors';

export const getItemRoute = createRoute({
  method: 'get',
  path: '/v1/items/{id}',
  responses: {
    200: { /* success schema */ },
    404: {
      description: 'Item not found',
      content: {
        'application/json': { schema: createErrorSchema('NOT_FOUND') },
      },
    },
  },
});
```

### 4. Integration errors — log and re-throw

When calling external services, log the integration error with a specific event, then throw an appropriate `AppError`:

```typescript
try {
  return await directusClient.request(readItem('items', id));
} catch (err) {
  logger.error({
    event: LogEvent.INTEGRATION_CMS_ERROR,
    event_type: LogEventType.APPLICATION_FAILURE,
    err,
  }, `CMS fetch failed for item ${id}`);
  throw AppError.internal('Failed to fetch item from CMS');
}
```

---

## 17. Rules

### Logging rules

1. **Never use `console.log/warn/error`** in server-side code. Import `logger` from the appropriate utility module. The only exception is code that runs before the logger is initialised (e.g. `apps/api/src/env.ts`).

2. **Never hardcode event strings.** Always use `LogEvent.SOME_EVENT` from `@newmoon/shared`.

3. **Always include `event` and `event_type`** in the structured data object (first argument to the logger call).

4. **Always include `requestId`** when it is available. In API handlers, it's automatic via the hono-pino child logger. In web server code, pass it explicitly from `event.context.requestId`.

5. **Pass errors as `err`** in the structured data object so Pino's error serialiser captures the stack trace:
   ```typescript
   logger.error({ event: LogEvent.APP_ERROR, event_type: LogEventType.APPLICATION_FAILURE, err }, 'Something failed');
   ```

6. **Log level matches HTTP status**: `info` for success, `warn` for 4xx, `error` for 5xx.

7. **Keep messages short and action-oriented.** The structured fields carry the data; the message is for human scanning:
   - Good: `'Auth callback failed'`
   - Bad: `'The authentication callback endpoint encountered an error while processing the OAuth code exchange'`

8. **Env variable `LOG_LEVEL`** (web) / **`NEWMOON_API_LOG_LEVEL`** (API) controls the minimum log level. Defaults to `info`. Set to `debug` for verbose output during development.

9. **Env variable `APP_VERSION`** should be set at deploy time (e.g. git SHA or package version) so logs can be correlated with specific releases.

10. **Client-side Vue code reports errors via `reportClientError()`.** In stores, composables, middleware, and plugins inside `apps/web/app/`, import `reportClientError` and `describeError` from `~/utils/client-logger` instead of calling `console.error` or reinventing `sendBeacon` plumbing. The helper forwards the entry to the Nitro sink at `/api/client-log`, which logs it under `CLIENT_ERROR_REPORT` with the full request context. Server-side Nitro code (`server/**`) should continue to use the standard `logger` import — only browser-origin code should go through `reportClientError()`.

### Error handling rules

11. **Use `AppError` for new code.** It provides typed codes, automatic status mapping, and works in both API and web.

12. **Never leak stack traces to clients.** The global error handler logs the full stack but only returns `{ code, message, requestId }` to the client.

13. **Don't catch errors in handlers unless you need to add context.** Let them propagate to `handleError`. If you do catch, always re-throw:
    ```typescript
    // Bad — swallows the error
    try { await doThing(); } catch (e) { return c.json({ error: 'failed' }, 500); }

    // Good — logs context, re-throws
    try { await doThing(); } catch (e) {
      logger.error({ event: LogEvent.INTEGRATION_ERROR, event_type: LogEventType.APPLICATION_FAILURE, err: e }, 'Context');
      throw AppError.internal('Human-readable message');
    }
    ```

14. **Log integration errors before re-throwing.** When an external call fails, log it with a specific `INTEGRATION_*` event so the integration failure is visible in logs separately from the generic `APP_ERROR` that `handleError` will also produce.

15. **Document error responses in OpenAPI.** Use `createErrorSchema()` in route definitions so the API documentation reflects possible error codes.

16. **Use `createError()` in Nitro routes (web).** This is H3's built-in error creator — it integrates with Nitro's error handling and the error handler plugin.
