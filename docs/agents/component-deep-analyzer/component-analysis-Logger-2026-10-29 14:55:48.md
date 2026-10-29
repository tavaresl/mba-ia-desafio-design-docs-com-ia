# Component Deep Analysis Report: Logger (src/shared/logger)

Date: 2026-10-29 14:55:48

## 1. Executive Summary

The Logger component is a single-file module (`src/shared/logger/index.ts`, 32 lines) that builds and exports a process-wide singleton structured logger based on `pino` 9.5.0. It reads `LOG_LEVEL` and `NODE_ENV` from the validated environment (`src/config/env.ts`), applies a fixed redaction list for sensitive fields, attaches static base fields (`service`, `env`), formats timestamps as ISO 8601, and switches to the `pino-pretty` transport in development.

Role in the system: it is the only logging facility of the Order Management API. It is consumed by the server bootstrap (`src/server.ts`), the HTTP request-logging middleware (`src/middlewares/request-logger.middleware.ts`) and the global error middleware (`src/middlewares/error.middleware.ts`).

Key findings:
- Redaction is configured through wildcard paths that only match one nesting level (`*.password`), so top-level or deeper-nested secrets are not covered (see Technical Debt).
- `pino-http` 10.3.0 is declared as a dependency but is not used; request logging is a hand-written middleware.
- `pino-pretty` is a devDependency but is referenced as transport only when `NODE_ENV === 'development'`.
- The logger is created at module import time (side effect) and is imported directly (no dependency injection), so it is not substitutable in tests.
- No test file references the logger; there are no unit or integration tests for it.
- The service name `order-management-api` is hardcoded.
- Only the 500 path in the error middleware logs the error; AppError, ZodError and handled Prisma errors are not logged.

## 2. Data Flow Analysis

Initialization flow:
```
1. Module `src/shared/logger/index.ts` is imported (by server.ts, request-logger.middleware.ts, error.middleware.ts)
2. `env` is loaded and validated by Zod in src/config/env.ts (LOG_LEVEL enum, default 'info'; NODE_ENV enum, default 'development')
3. `createLogger()` (index.ts:13) builds the pino instance with level, redact, base, timestamp and transport
4. The instance is exported as the singleton `logger` (index.ts:32)
```

Runtime log flow (HTTP request):
```
1. Request enters Express; requestLogger is registered after express.json (app.ts:60)
2. requestLogger reads `x-request-id` header or generates a UUID v4 (request-logger.middleware.ts:6)
3. req.id is set and X-Request-Id response header is written (lines 7-8)
4. A high-resolution start timestamp is captured (line 10)
5. Handlers / auth / controllers run (auth middleware adds req.user)
6. On the response 'finish' event, duration is computed and logger.info(...,'http_request') is emitted (lines 12-25)
7. pino applies level filter, redaction, base fields, ISO timestamp
8. Output goes to stdout as JSON (non-development) or through the pino-pretty worker transport (development)
```

Runtime log flow (unhandled error):
```
1. Error propagates to errorMiddleware (error.middleware.ts:14)
2. AppError, ZodError, Prisma P2002/P2025 are answered without logging
3. Any other error: requestId taken from req.id or 'unknown' (line 56)
4. logger.error({err, requestId, method, path}, 'Unhandled error in request') (line 57); pino's default `err` serializer renders the error
5. A generic 500 JSON response is returned
```

Lifecycle flow (server.ts): `server_started` (info, line 10), `shutdown_initiated` (info, line 14), `http_server_closed` (info, line 15), `bootstrap_failed` (fatal, line 25).

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Configuration | Log level is driven by LOG_LEVEL (fatal, error, warn, info, debug, trace; default info) | src/shared/logger/index.ts:15, src/config/env.ts:6 |
| Security | Sensitive fields are replaced with "[REDACTED]" | src/shared/logger/index.ts:4-11, 16-19 |
| Observability | Every log line carries service name and environment | src/shared/logger/index.ts:20 |
| Format | Timestamps are ISO 8601 | src/shared/logger/index.ts:21 |
| Environment | Pretty human-readable output only in development; JSON otherwise | src/shared/logger/index.ts:22-28 |
| Singleton | One shared logger instance per process | src/shared/logger/index.ts:32 |
| Traceability | Request ID is propagated from x-request-id or generated, and echoed in response | src/middlewares/request-logger.middleware.ts:6-8 |
| Observability | Each completed HTTP request logs one `http_request` info entry with method, path, status, duration and user id | src/middlewares/request-logger.middleware.ts:12-25 |
| Error handling | Only unexpected errors are logged at error level with request context | src/middlewares/error.middleware.ts:56-60 |
| Lifecycle | Server start, shutdown and fatal bootstrap failure are logged | src/server.ts:10,14,15,25 |

### Detailed breakdown of the business rules

---

### Business Rule: Configurable log level

**Overview**:
The minimum severity emitted is set from the `LOG_LEVEL` environment variable.

**Detailed description**:
`createLogger` passes `env.LOG_LEVEL` to pino's `level` option (index.ts:15). The value is validated at startup by Zod in `src/config/env.ts:6` as an enum of `fatal`, `error`, `warn`, `info`, `debug`, `trace` with default `info`. `.env.example:3` sets `LOG_LEVEL=info`. An invalid value is rejected by env validation before the logger is created (behavior of env.ts failure handling is outside this component and was not analyzed in depth).

The level is fixed at process start. There is no runtime level change, no per-module child loggers, and no per-request level override. All consumers share the same threshold.

Practical effect: in the current code only `info`, `error` and `fatal` calls exist, so `debug` and `trace` produce no additional output; setting `warn` would suppress `http_request` and lifecycle info logs, leaving only unhandled errors and fatal bootstrap failures.

**Rule workflow**:
```
env.LOG_LEVEL (validated) -> pino level -> calls below threshold are dropped
```

---

### Business Rule: Sensitive data redaction

**Overview**:
Fields matching a fixed path list are replaced with the literal `[REDACTED]` before output.

**Detailed description**:
`redactPaths` (index.ts:4-11) contains: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`. The censor string is `[REDACTED]` (index.ts:18). In pino/fast-redact semantics, `*.password` matches the key `password` one level below any top-level key (for example `{ user: { password } }` or `{ body: { password } }`), but not a top-level `password` and not deeper nesting such as `a.b.password`.

The `req.headers.*` paths only take effect if a log object contains a `req` key with `headers`. The current callers do not log a `req` object: request-logger logs flat scalar fields, and error middleware logs `err`, `requestId`, `method`, `path`. Thus the header redactions are presently latent protections (relevant if `pino-http` or a raw request were logged). The `*.` patterns would apply for example to `err.token`-style properties if present on error objects, but no current caller logs credentials intentionally. Confidence: high for the pino semantics described, based on the library's documented wildcard behavior; not verified by running code.

Redaction relates to the domain: the system has JWT authentication (`jsonwebtoken`) and bcrypt password hashes (`passwordHash`), so the listed names mirror the User model and auth tokens. Fields such as `refreshToken`, `secret`, `JWT_SECRET`, `x-api-key`, and top-level `password` are not in the list.

**Rule workflow**:
```
log object -> match against redactPaths -> matched values replaced with '[REDACTED]' -> serialized
```

---

### Business Rule: Static base context fields

**Overview**:
Every entry includes `service: 'order-management-api'` and `env: <NODE_ENV>`.

**Detailed description**:
The `base` option (index.ts:20) overrides pino's default base (`pid`, `hostname`), so `pid` and `hostname` are not emitted. The service name is a hardcoded literal that matches `package.json` `name`, but is not read from it. The `env` value comes from validated `NODE_ENV` (development, test, production; default development).

This supports filtering in log aggregation by service and environment. Note that `server.ts:10` also logs `env` explicitly in the `server_started` payload, which duplicates the base field.

Because `pid` and `hostname` are dropped, multi-instance deployments cannot distinguish instances from log content alone (inference; no deployment config exists in the repository besides docker-compose.yml, not analyzed for logging).

**Rule workflow**:
```
pino instance creation -> base fields merged into every log line
```

---

### Business Rule: ISO timestamp format

**Overview**:
Timestamps use `pino.stdTimeFunctions.isoTime`.

**Detailed description**:
index.ts:21 replaces the default epoch-milliseconds `time` with an ISO 8601 UTC string under the key `time`. This makes logs human-readable and ISO-parseable by aggregators.

In development, pino-pretty additionally reformats it with `translateTime: 'HH:MM:ss.l'` (index.ts:26), so developers see local-style short time without date.

**Rule workflow**:
```
log call -> time = ISO string -> (dev only) pino-pretty translates to HH:MM:ss.l
```

---

### Business Rule: Environment-dependent output format

**Overview**:
Development uses pino-pretty; every other environment emits newline-delimited JSON to stdout.

**Detailed description**:
If `env.NODE_ENV === 'development'` (index.ts:23), a transport is configured with target `pino-pretty` and options `colorize: true`, `singleLine: false`, `translateTime: 'HH:MM:ss.l'`. Otherwise `transport` is `undefined`, and pino writes JSON synchronously-buffered to stdout (default destination).

Because `NODE_ENV` defaults to `development` in env.ts:4, an unset variable results in pretty output. `pino-pretty` is a devDependency (package.json), so an installation that omits dev dependencies while running with `NODE_ENV=development` (or unset) would fail to resolve the transport target; this is an inference from the dependency declaration, not tested. Using a `transport` makes pino run formatting in a worker thread in development.

The `test` environment uses JSON output, meaning tests emit JSON log lines (including `http_request`) to stdout unless LOG_LEVEL is set to a silent-compatible level; `silent` is not an allowed enum value.

**Rule workflow**:
```
NODE_ENV === 'development' ? pino-pretty transport (worker thread) : default JSON stdout
```

---

### Business Rule: Singleton shared logger

**Overview**:
A single `logger` constant is created at import time and shared by all modules.

**Detailed description**:
`export const logger: Logger = createLogger();` (index.ts:32). ES module caching guarantees one instance per process. `createLogger` is also exported, which allows creating additional instances, but no code in the project does this.

Consumers import the singleton directly with a relative path (`../shared/logger/index.js`), meaning there is no injection through `AppDependencies` (app.ts), unlike `prisma`. Importing the module therefore triggers env validation and transport setup as side effects, including in tests that import `app.ts` (middlewares import the logger).

**Rule workflow**:
```
first import -> createLogger() -> exported const reused by all importers
```

---

### Business Rule: Request correlation ID

**Overview**:
Each request gets a correlation ID, reused from the `x-request-id` header or newly generated, attached to `req.id` and the response.

**Detailed description**:
request-logger.middleware.ts:6 uses the incoming `x-request-id` header if present (cast to string, without validation of format or length) and otherwise `uuidv4()`. The ID is stored on `req.id` (typed through the module augmentation in auth.middleware.ts:13-17) and returned as `X-Request-Id` (line 8).

Because the header value is trusted as-is, a client can supply arbitrary strings (including very long strings or, if repeated headers are sent, Node joins them in a comma-separated value). Node rejects invalid header characters when setting response headers, which could throw for malformed values (inference from Node behavior; not verified here).

The ID is included in the `http_request` log and in the unhandled-error log (via `req.id ?? 'unknown'`), so both entries for the same request can be correlated. Other logs (lifecycle) have no request ID. The logger itself does not carry request context (no child loggers or AsyncLocalStorage).

**Rule workflow**:
```
header x-request-id ?? uuidv4() -> req.id -> response header -> included in log payloads
```

---

### Business Rule: HTTP access logging

**Overview**:
One `info`-level `http_request` entry is written per completed response.

**Detailed description**:
On the response `finish` event (line 12), duration is computed from `process.hrtime.bigint()` difference converted to milliseconds and rounded to two decimals (lines 13, 20). Payload fields: `requestId`, `method`, `path` (`req.originalUrl`, which includes the query string), `statusCode`, `durationMs`, `userId` (`req.user?.id`, populated only on authenticated routes; undefined fields are omitted by JSON serialization).

Observations: the log level is always `info` regardless of status code (4xx and 5xx also log at info). Because the `finish` event is used, requests aborted before completion (client disconnect, `close` without `finish`) are not logged. `path` includes query strings, so query parameters containing sensitive values would be logged unredacted (redaction paths do not apply). The middleware is registered after `express.json` (app.ts:60), so requests rejected by the JSON body parser (e.g., malformed JSON or oversize body above 1mb) fail before this middleware runs and are not access-logged and have no request ID (inference from middleware order; the parser error is handled by the later error middleware, where `requestId` becomes 'unknown'). The `/health` endpoint is also logged.

**Rule workflow**:
```
request start -> hrtime start -> response 'finish' -> compute durationMs -> logger.info(payload, 'http_request')
```

---

### Business Rule: Unexpected error logging

**Overview**:
Only errors not mapped to a known type are logged, at `error` level, before a generic 500 response.

**Detailed description**:
In error.middleware.ts the early branches for `AppError`, `ZodError`, and Prisma codes P2002 and P2025 return without logging (lines 15-54). Anything else, including other Prisma errors and unknown exceptions, reaches line 57 where `logger.error({ err, requestId, method, path }, 'Unhandled error in request')` is emitted. The response body hides internals (`INTERNAL_SERVER_ERROR`).

The `err` key uses pino's standard error serializer (type, message, stack). Since the stack is emitted to logs, not to clients, internals stay server-side. Note that a Prisma error with an unhandled code is logged with its message, which may contain query details (inference).

Consequence: expected domain failures (4xx) leave only the `http_request` info line with status code as a trace; no warn-level logging exists for them.

**Rule workflow**:
```
error -> known type? respond without log : logger.error(err+context) -> 500 generic response
```

---

### Business Rule: Application lifecycle logging

**Overview**:
Startup, shutdown, and fatal bootstrap failure events are logged with fixed message keys.

**Detailed description**:
server.ts logs `server_started` with port and env (line 10) after listen, `shutdown_initiated` with signal on SIGINT/SIGTERM (line 14), `http_server_closed` in the `server.close` callback (line 15), and `bootstrap_failed` at `fatal` with the error (line 25) followed by `process.exit(1)`.

Message names use snake_case event identifiers (`server_started`, `http_request`) except the error message `'Unhandled error in request'`, which is free text (naming inconsistency). In `shutdown`, `process.exit(0)` runs right after `prisma.$disconnect()` without waiting for the `server.close` callback, so `http_server_closed` may not be emitted before exit (inference from control flow, lines 15-17). In development the pino-pretty transport runs in a worker thread, so entries logged immediately before `process.exit` may be lost (inference from pino transport behavior).

**Rule workflow**:
```
listen -> server_started; signal -> shutdown_initiated -> server.close(cb: http_server_closed) -> prisma disconnect -> exit(0)
bootstrap rejection -> fatal bootstrap_failed -> exit(1)
```

---

## 4. Component Structure

```
src/shared/logger/
└── index.ts                  # createLogger() factory, redaction list, exported singleton `logger`

Consumers (outside the component boundary):
src/server.ts                                  # lifecycle logging
src/middlewares/request-logger.middleware.ts   # http_request access log, request ID
src/middlewares/error.middleware.ts            # unhandled error logging
src/config/env.ts                              # LOG_LEVEL / NODE_ENV source (dependency)
.env.example                                   # NODE_ENV=development, LOG_LEVEL=info
```

Configuration: no dedicated config file; per-environment behavior is limited to the development/non-development transport switch.

## 5. Dependency Analysis

```
Internal Dependencies:
logger/index.ts -> config/env.ts (LOG_LEVEL, NODE_ENV)

Reverse (consumers):
server.ts -> logger
request-logger.middleware.ts -> logger
error.middleware.ts -> logger

External Dependencies:
- pino 9.5.0 - Structured logging (runtime dependency)
- pino-pretty 11.3.0 - Dev-only pretty transport (devDependency, loaded by name via transport.target)
- pino-http 10.3.0 - Declared but unused anywhere in src (no import found)
```

## 6. Afferent and Efferent Coupling

Units are modules/functions/constants (TypeScript, functional style).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| logger (const singleton) | 3 (server.ts, request-logger.middleware.ts, error.middleware.ts) | 1 internal (createLogger) | Medium |
| createLogger | 1 (internal, index.ts:32) | 2 (pino, env) | Low |
| redactPaths | 1 (createLogger) | 0 | Medium |
| env (config/env.ts, as used by Logger) | 1 from logger (plus others in project) | 0 | Medium |
| requestLogger (consumer) | 1 (app.ts) | 3 (uuid, logger, req augmentation) | Medium |
| errorMiddleware (consumer) | registered in app.ts | 4 (logger, AppError, zod, Prisma) | Medium |

Package-level: Logger afferent = 3 modules; efferent = 2 (pino, config/env).

## 7. Endpoints

Not applicable: the Logger component exposes no endpoints. (It sets the `X-Request-Id` response header through the consumer middleware, but defines no routes.)

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| stdout | Process stream | Log sink in non-development | Process stdout | NDJSON | None defined; pino default |
| pino-pretty | In-process worker transport | Developer-readable output | Worker thread | Text (colorized) | None defined |
| Environment (config/env.ts) | Internal config | LOG_LEVEL, NODE_ENV | Module import | Zod-validated strings | Validation in env.ts |
| Log aggregation / external shipper | Not present | No integration code found | N/A | N/A | N/A |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Singleton (module-level) | `export const logger` | src/shared/logger/index.ts:32 | One shared instance |
| Factory function | `createLogger()` | src/shared/logger/index.ts:13 | Encapsulate configuration |
| Structured logging | pino JSON with message keys | whole component | Machine-parseable logs |
| Middleware | requestLogger | src/middlewares/request-logger.middleware.ts | Cross-cutting access log |
| Correlation ID | `x-request-id` / uuid | request-logger.middleware.ts:6-8 | Request tracing |
| Redaction / data masking | pino `redact` | index.ts:16-19 | Prevent secret leakage |
| Shared kernel | `src/shared/` folder | src/shared/logger | Cross-module utility |

Architectural notes: configuration is centralized via env module; no dependency injection for logging; no child-logger or async context propagation.

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Redaction | Wildcard `*.x` paths match one level only; top-level `password`/`token` and deeper nesting not covered; list is static | Possible secret leakage if such shapes are logged |
| Medium | Request logging | `path` uses `originalUrl` including query string, not redacted | Sensitive query values may be logged |
| Medium | Request ID | Incoming `x-request-id` is trusted without validation | Log injection / header errors / unbounded size |
| Medium | Lifecycle | `process.exit(0)` not awaiting `server.close` callback or transport flush | Lost or missing shutdown logs |
| Low | Dependencies | `pino-http` declared but unused | Unused dependency |
| Low | Dependencies | `pino-pretty` is a devDependency but referenced when NODE_ENV is development (the default) | Potential resolution failure in environments without dev deps and unset NODE_ENV |
| Low | Config | Service name hardcoded; `pid`/`hostname` removed from base | Reduced instance identification |
| Low | Levels | All HTTP requests logged at info, including 4xx/5xx; handled errors not logged | Reduced signal for failures |
| Low | Naming | Mixed snake_case event names and free-text message | Inconsistent log query keys |
| Low | Testability | Import-time singleton, no injection, no tests | Hard to assert or silence logs in tests |
| Low | Coverage of events | Body-parser failures occur before requestLogger | Missing access logs and request ID for those requests |
| Low | Duplication | `env` logged in `server_started` and in base | Redundant field |
| Low | Typing | `req.id` augmentation lives in auth.middleware.ts; error middleware casts `req` instead | Hidden coupling between modules |

## 11. Test Coverage Analysis

Test files located: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/`. Search for "logger" in `tests/` and `vitest.config.ts` returned no matches. Vitest runs with `tests/**/*.test.ts`; no coverage provider or threshold is configured in `vitest.config.ts`.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| createLogger / logger | 0 | 0 | 0% direct (no coverage tooling configured; numeric value is not measured) | No assertions on level, redaction, transport selection or base fields |
| requestLogger | 0 | Indirect only: exercised when integration tests call the app via supertest (not asserted) | Not measured | No assertions on X-Request-Id header or log content found |
| errorMiddleware logging branch | 0 | Indirect at most | Not measured | No assertions on log output |
| server.ts lifecycle logs | 0 | 0 | Not measured | Not tested (entrypoint not imported by tests) |

Risk: absence of tests for redaction (security-relevant) is highlighted as a gap.
