# Component Deep Analysis Report: Request Logger Middleware

Component file: `src/middlewares/request-logger.middleware.ts` (28 lines, single exported symbol `requestLogger`).
Analysis date: 2026-10-29. Ignore-folders: none. Analysis only; no project files were modified.

## 1. Executive Summary

The Request Logger Middleware is an Express `RequestHandler` (`src/middlewares/request-logger.middleware.ts:5`) that provides request correlation and access logging for the Order Management API. It performs three responsibilities:

1. Resolves a request identifier, reusing the inbound `x-request-id` header or generating a UUID v4, then stores it on `req.id` and echoes it in the `X-Request-Id` response header (lines 6-8).
2. Starts a high-resolution timer (`process.hrtime.bigint()`, line 10).
3. On the response `finish` event, emits one structured `info` log entry with message `http_request` containing requestId, method, path, statusCode, durationMs and userId (lines 12-25).

It is registered globally in `src/app.ts:60`, after `express.json` and before `/health`, the `/api/v1` router, the 404 handler and `errorMiddleware`.

Key findings:
- The inbound `x-request-id` header is trusted without validation (no length, format or character check).
- `req.id` is consumed by `src/middlewares/error.middleware.ts:56` for error log correlation; the type augmentation of `req.id` and `req.user` lives in `auth.middleware.ts:11-16`, not in this component (hidden type coupling).
- `userId` is only populated when authentication ran before the `finish` event; since `finish` fires after the whole chain, authenticated routes do log the user id (the `req.user` read is lazy, at finish time).
- Only the `finish` event is handled; aborted connections (`close` without `finish`) are not logged.
- No tests exist for this component.
- `pino-http` 10.3.0 is declared in `package.json:31` but not used by this component (custom implementation instead).

## 2. Data Flow Analysis

```
1. Request enters Express app; express.json (src/app.ts:59) parses the body first.
2. requestLogger invoked (src/app.ts:60).
3. Header x-request-id read; fallback to uuidv4() (request-logger.middleware.ts:6).
4. req.id assigned (line 7); X-Request-Id response header set (line 8).
5. Start timestamp captured via process.hrtime.bigint() (line 10).
6. 'finish' listener registered on res (line 12).
7. next() called; request continues to routes, 404 handler, errorMiddleware (line 27).
8. Downstream handlers (authenticate may set req.user, auth.middleware.ts:42) produce the response.
9. On res 'finish': duration computed in ms (line 13), rounded to 2 decimals (line 20).
10. logger.info (pino instance, src/shared/logger/index.ts:32) writes the structured entry with message 'http_request'.
11. Pino applies redaction paths and, in development, routes output through pino-pretty.
```

Note: if `express.json` rejects an oversized/malformed body (it runs before the middleware), the logger is never reached and that request is not logged by this component (inference from middleware order in `src/app.ts:59-60`; the error passes to `errorMiddleware`, which logs only for unhandled errors).

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Correlation | Reuse client-supplied `x-request-id` if present, otherwise generate UUID v4 | request-logger.middleware.ts:6 |
| Propagation | Request id stored on `req.id` for downstream consumers | request-logger.middleware.ts:7 |
| Propagation | Request id echoed in `X-Request-Id` response header | request-logger.middleware.ts:8 |
| Observability | Duration measured with monotonic high-resolution clock, in ms, 2 decimals | request-logger.middleware.ts:10,13,20 |
| Observability | One access log per completed response, at `info` level, message `http_request` | request-logger.middleware.ts:12-24 |
| Observability | Log fields: requestId, method, path (originalUrl), statusCode, durationMs, userId | request-logger.middleware.ts:15-22 |
| Identity | `userId` logged from `req.user?.id` (undefined for unauthenticated requests) | request-logger.middleware.ts:21 |
| Security (inherited) | Sensitive fields redacted by shared logger configuration | src/shared/logger/index.ts:4-11 |
| Flow control | Middleware never blocks or fails a request; always calls `next()` | request-logger.middleware.ts:27 |

### Detailed breakdown of the business rules

---

### Business Rule: Request ID Resolution and Propagation

**Overview**:
Every request receives a correlation id, taken from the `x-request-id` header when supplied or generated as UUID v4, and exposed on both the request object and response header.

**Detailed description**:
At line 6 the middleware reads `req.headers['x-request-id']`, cast to `string | undefined`. The nullish coalescing operator (`??`) means only `undefined`/`null` trigger generation; an empty string header value would be accepted as-is, and Node may deliver a repeated header as a joined string (not an array for this header name), so the cast is generally safe, though the type assertion bypasses compile-time checking. This supports distributed tracing scenarios where an upstream gateway or client supplies the id.

The id is assigned to `req.id` (line 7) so downstream code can correlate. The only found consumer is `src/middlewares/error.middleware.ts:56`, which reads it via an inline cast and falls back to `'unknown'`. The `Request` type is augmented with `id?: string` in `src/middlewares/auth.middleware.ts:11-16`; this component depends on that declaration being included in the compilation. The id is also set as the `X-Request-Id` response header (line 8), allowing clients to quote it when reporting issues. The header is set before `next()`, so it is present on all responses, including 404 and 500 responses produced by later handlers.

Because the client-provided value is not validated, a caller can inject arbitrary strings (long values, values that Node's `setHeader` rejects such as invalid characters would throw `ERR_INVALID_CHAR`, which would surface as an Express error). Also, a client can choose ids that collide with other requests' ids, undermining trace uniqueness. These are observations; the code states no explicit policy.

**Rule workflow**:
```
read x-request-id header
  -> present (non-null)? use it : uuidv4()
  -> req.id = requestId
  -> res.setHeader('X-Request-Id', requestId)
```

---

### Business Rule: Request Duration Measurement

**Overview**:
Latency of each request is measured from middleware entry to response `finish`, in milliseconds with two decimals.

**Detailed description**:
A `bigint` nanosecond timestamp from `process.hrtime.bigint()` is captured at line 10. Using a monotonic clock avoids distortion from system clock adjustments. At `finish` (line 13) the difference is converted with `Number(...) / 1_000_000` to milliseconds, then `toFixed(2)` and `Number()` produce a numeric value rounded to two decimals (line 20).

The measured interval starts after `express.json` has parsed the body (middleware order, `src/app.ts:59-60`), so body parsing time is excluded. It ends on `finish`, which fires when the response has been handed to the OS, so it includes handler, database (Prisma) and serialization time but not client network transit.

The duration appears only inside the access log; it is not exposed as a header or metric.

**Rule workflow**:
```
startedAt = hrtime.bigint()
... request processing ...
on finish: durationMs = round2((hrtime.bigint() - startedAt) / 1e6)
```

---

### Business Rule: Structured Access Logging on Response Completion

**Overview**:
One `info`-level structured entry (`http_request`) is written per response that emits `finish`.

**Detailed description**:
A listener on `res.on('finish')` (line 12) logs the object `{ requestId, method, path, statusCode, durationMs, userId }`. `path` uses `req.originalUrl` (line 18), meaning the full URL including query string and the `/api/v1` mount prefix, not the stripped `req.url`. Query strings are therefore logged verbatim; the shared redaction paths (`src/shared/logger/index.ts:4-11`) cover only `req.headers.authorization`, `req.headers.cookie` and `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` at one nesting level, so sensitive values in query strings would not be redacted (observation).

The log level is fixed at `info`, regardless of status code: 4xx and 5xx responses are also logged at `info` by this component (the error middleware separately logs unhandled errors at `error`, `error.middleware.ts:57`). The effective output depends on `LOG_LEVEL` (`src/config/env.ts:6`, default `info`); at `warn` or higher these entries are suppressed. In development (`NODE_ENV=development`) the logger uses `pino-pretty` transport (`src/shared/logger/index.ts:22-28`).

Only `finish` is observed. If the client disconnects before the response completes, `close` fires without `finish` and no entry is logged (observation from code).

**Rule workflow**:
```
res 'finish' -> compute duration -> logger.info({fields}, 'http_request')
```

---

### Business Rule: Authenticated User Attribution

**Overview**:
The log entry includes the authenticated user's id when available.

**Detailed description**:
Line 21 reads `req.user?.id` at `finish` time. `req.user` is set by the `authenticate` middleware (`src/middlewares/auth.middleware.ts:42`) which runs later, inside route chains. Because the read is deferred until `finish`, the value reflects authentication that occurred during the request. For public routes (`/health`, login) and requests rejected as unauthorized, `userId` is `undefined` and pino omits the key.

Only the id is logged; email and role are not. Whether the user id is considered personal data is not addressed in code.

**Rule workflow**:
```
at finish: userId = req.user?.id (set by authenticate if it ran) -> included in log
```

---

### Business Rule: Non-Blocking Pass-Through

**Overview**:
The middleware never rejects, alters status or short-circuits a request.

**Detailed description**:
`next()` is always called synchronously at line 27 with no arguments. There is no try/catch around the logging callback. An exception thrown inside the `finish` listener (for example from the logger) would occur outside the Express error pipeline, as an event-emitter exception, with potential for an uncaught exception; nothing in the code guards this (risk, low probability).

**Rule workflow**:
```
set up id + timer + listener -> next()
```

---

## 4. Component Structure

```
src/
├── middlewares/
│   └── request-logger.middleware.ts   # The component: requestLogger RequestHandler
│   ├── auth.middleware.ts             # (related) declares Request.id / Request.user augmentation
│   └── error.middleware.ts            # (related) consumes req.id
├── shared/logger/index.ts             # pino logger used by the component
├── config/env.ts                      # LOG_LEVEL, NODE_ENV
└── app.ts                             # registers the middleware (line 60)
```

Single-file component, one exported const, no classes, no configuration of its own.

## 5. Dependency Analysis

```
Internal Dependencies:
requestLogger -> shared/logger/index.ts (logger) -> config/env.ts
requestLogger -> (type only) Request augmentation in middlewares/auth.middleware.ts (req.id, req.user)
app.ts -> requestLogger
errorMiddleware -> req.id (runtime data contract, not an import)

External Dependencies:
- express (4.21.1) - RequestHandler type (type-only import)
- uuid (11.0.3) - v4 id generation
- pino (9.5.0) - logging backend (via shared logger)
- pino-pretty (11.3.0) - development transport (via shared logger)
- Node.js built-ins: process.hrtime.bigint
- Not used: pino-http (10.3.0, declared in package.json:31)
```

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| requestLogger | 1 (app.ts import; plus implicit runtime contract with errorMiddleware via req.id) | 3 (logger, uuid, express types) plus implicit dependency on the Request type augmentation in auth.middleware.ts | Medium |
| logger (shared) | 2+ (this component, error.middleware.ts, possibly others) | 1 (env) | Medium |
| Request.id augmentation (auth.middleware.ts) | 1 implicit (this component writes) and 1 (error.middleware reads via cast) | 0 | Low |

## 7. Endpoints

Not applicable as a standalone endpoint provider. The middleware applies to all routes (global `app.use`, `src/app.ts:60`) and adds the `X-Request-Id` response header to every response, including `/health`, `/api/v1/*` and 404s.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Client / upstream gateway | Inbound header | Receive correlation id | HTTP header `x-request-id` | String | None; unvalidated |
| Client | Outbound header | Return correlation id | HTTP header `X-Request-Id` | String | None |
| Shared pino logger | Internal library | Emit access log | In-process / stdout (pino-pretty in dev) | JSON (pino) | None in component; relies on pino |
| Express response lifecycle | Framework event | Detect response completion | Node EventEmitter `finish` | n/a | None |
| errorMiddleware | Internal | Correlate error logs via `req.id` | Property on request | String | Falls back to `'unknown'` |
| authenticate middleware | Internal | Provide `req.user` | Property on request | AuthUser | `userId` undefined if absent |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Middleware / Chain of Responsibility | Express `RequestHandler` | request-logger.middleware.ts:5 | Cross-cutting request handling |
| Correlation ID | `x-request-id` reuse or UUID | request-logger.middleware.ts:6-8 | Request tracing across logs and clients |
| Observer (event listener) | `res.on('finish')` | request-logger.middleware.ts:12 | Deferred logging after response |
| Closure state | `requestId`, `startedAt` captured per request | request-logger.middleware.ts:6,10 | Per-request state without globals |
| Module singleton | Imported `logger` | src/shared/logger/index.ts:32 | Shared logging instance |
| Declaration merging | `Request` interface augmentation (in auth middleware) | auth.middleware.ts:11-16 | Typed `req.id`/`req.user` |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Request id handling | Client-supplied `x-request-id` is not validated or length-limited; invalid header characters would make `setHeader` throw | Log injection/pollution, id collisions, possible 500 on malformed ids |
| Medium | Logging of `path` | `originalUrl` logged with query string; redaction does not cover URLs | Potential leakage of sensitive query values to logs |
| Low | Event handling | Only `finish` handled; no `close`/abort logging | Aborted requests invisible in access logs |
| Low | Log level | Fixed `info` for all status codes (including 5xx) | Errors not distinguishable by level in access log |
| Low | Type coupling | `req.id`/`req.user` augmentation declared in `auth.middleware.ts`, not in this file; error middleware uses an inline cast | Hidden dependency; fragile typing |
| Low | Error safety | No try/catch in `finish` listener | Logger failure would be an unhandled event exception |
| Low | Unused dependency | `pino-http` declared but custom logging implemented | Dependency redundancy |
| Low | Ordering | Registered after `express.json`; body-parse failures are not access-logged by this component | Gaps in access log |
| Medium | Test coverage | No tests for this component | Regressions undetected |

## 11. Test Coverage Analysis

Test files located in the project: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`; configuration `vitest.config.ts` (include `tests/**/*.test.ts`). A search for `request-logger`, `requestLogger`, `X-Request-Id`, `x-request-id` and `logger` across `tests/` found no references. No coverage tooling configuration (thresholds or reporters) was found in `vitest.config.ts`; coverage was not measured and tests were not executed.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| requestLogger | 0 | 0 dedicated (the middleware is exercised implicitly if integration tests in `tests/auth.test.ts` and `tests/orders.test.ts` build the app through `buildApp`; not verified) | Not measured; no assertions on id, header, or log output | None: no assertions on `X-Request-Id`, generated vs supplied id, duration, log fields, or `userId` |

Untested behaviors: header reuse vs generation, response header presence, log field content, userId attribution, behavior with invalid header values, aborted requests.
