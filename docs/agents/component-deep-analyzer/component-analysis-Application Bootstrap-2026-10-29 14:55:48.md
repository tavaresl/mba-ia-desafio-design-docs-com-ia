# Component Deep Analysis Report: Application Bootstrap (src/server.ts)

Date: 2026-10-29 14:55:48
Scope: `src/server.ts` (27 lines) plus the modules it directly composes at start-up (`src/app.ts`, `src/config/env.ts`, `src/config/database.ts`, `src/shared/logger/index.ts`).

## 1. Executive Summary

The Application Bootstrap is the process entry point of the `order-management-api` (Express 4 + Prisma + MySQL, ESM TypeScript). It is a single function, `bootstrap()` (src/server.ts:6-22), that:

1. Builds the Express application through the composition root `buildApp({ prisma })` (src/server.ts:7).
2. Binds the HTTP listener on `env.PORT` and logs `server_started` (src/server.ts:9-11).
3. Registers a graceful-shutdown handler for SIGINT and SIGTERM (src/server.ts:13-21).
4. Converts any start-up failure into a `fatal` log and exit code 1 (src/server.ts:24-27).

Key findings:
- The component is thin and delegates all wiring to `src/app.ts` and all configuration to `src/config/env.ts`. Environment validation is a side effect of module import (fail-fast, `process.exit(1)` at src/config/env.ts:22).
- Graceful shutdown is only partial: `server.close()` is not awaited, so `process.exit(0)` runs right after `prisma.$disconnect()` and can cut in-flight requests (src/server.ts:15-17).
- There is no handling of `unhandledRejection` / `uncaughtException`, no shutdown timeout, no guard against repeated signals, and no explicit DB connectivity check at start-up (Prisma connects lazily).
- `src/server.ts` is not covered by any automated test. Tests build the app via `getTestApp()` and bypass the bootstrap.

## 2. Data Flow Analysis

Start-up flow:

```
1. Process starts: `tsx watch --env-file=.env src/server.ts` (dev) or `node --env-file=.env dist/server.js` (prod) (package.json scripts)
2. ES imports are evaluated in order (src/server.ts:1-4):
   a. ./app.js        -> loads Express, middlewares, all module repositories/services/controllers, routes
   b. ./config/env.js -> zod parses process.env; on failure prints issues and process.exit(1) (src/config/env.ts:14-27)
   c. ./config/database.js -> instantiates singleton PrismaClient (log level depends on NODE_ENV) (src/config/database.ts:4-10)
   d. ./shared/logger/index.js -> creates pino logger (redaction, pino-pretty in development) (src/shared/logger/index.ts:13-32)
3. bootstrap() invoked (src/server.ts:24)
4. buildApp({ prisma }) (src/app.ts:55-76):
   - disables x-powered-by, JSON body parser (1mb), requestLogger
   - GET /health
   - buildControllers(prisma): manual dependency injection of repositories -> services -> controllers (src/app.ts:26-53)
   - mounts /api/v1 router, 404 handler (NotFoundError), errorMiddleware
5. app.listen(env.PORT, cb) -> cb logs { port, env } 'server_started' (src/server.ts:9-11)
6. Registers SIGINT / SIGTERM listeners (src/server.ts:20-21)
7. Runtime: requests are served by the Express app
```

Shutdown flow:

```
1. SIGINT or SIGTERM received
2. shutdown(signal) logs 'shutdown_initiated' (src/server.ts:14)
3. server.close(cb) stops accepting new connections; callback logs 'http_server_closed' (not awaited) (src/server.ts:15)
4. await prisma.$disconnect() (src/server.ts:16)
5. process.exit(0) (src/server.ts:17)
```

Failure flow:

```
bootstrap() rejects -> .catch logs logger.fatal({ err }, 'bootstrap_failed') -> process.exit(1) (src/server.ts:24-27)
```

## 3. Business Rules & Logic

The component holds operational (lifecycle) rules rather than domain rules. Confidence is High for all, since they are explicit in code.

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Configuration | Server listens on `env.PORT` (default 3000, positive integer) | src/server.ts:9, src/config/env.ts:5 |
| Configuration | Environment variables validated at import time; invalid config terminates the process with exit 1 | src/config/env.ts:14-27 |
| Configuration | `DATABASE_URL` required; `JWT_SECRET` min 16 chars; `NODE_ENV` in development/test/production | src/config/env.ts:4,7,8 |
| Lifecycle | Single PrismaClient instance shared with the app via dependency injection | src/server.ts:7, src/config/database.ts:10 |
| Lifecycle | Start-up emits structured log `server_started` with port and env | src/server.ts:10 |
| Lifecycle | Graceful shutdown on SIGINT and SIGTERM: stop HTTP, disconnect DB, exit 0 | src/server.ts:13-21 |
| Resilience | Any bootstrap failure logs fatal and exits 1 | src/server.ts:24-27 |
| Observability | Logger redacts auth header, cookie, password, passwordHash, token, accessToken | src/shared/logger/index.ts:4-11 |
| Observability | Log level from `LOG_LEVEL`; pretty transport only in development | src/shared/logger/index.ts:15,22-29 |
| Persistence | Prisma log verbosity: warn+error in development, error otherwise | src/config/database.ts:6 |

### Detailed breakdown of the business rules

---

### Business Rule: Fail-fast environment validation

**Overview**:
The process must not start with an invalid or incomplete configuration.

**Detailed description**:
`src/config/env.ts` defines a zod schema and parses `process.env` at module load (`export const env = loadEnv()`, line 27). Because `src/server.ts` imports this module (line 2), and indirectly so do `database.ts`, `logger/index.ts` and `app.ts` dependencies, validation happens before `bootstrap()` runs. The schema enforces: `NODE_ENV` enum with default `development`; `PORT` coerced to a positive integer with default 3000; `LOG_LEVEL` enum with default `info`; `DATABASE_URL` non-empty (no default, so mandatory); `JWT_SECRET` at least 16 characters (mandatory); `JWT_EXPIRES_IN` string defaulting to `8h`.

When validation fails, each issue is printed via `console.error` as `  - <path>: <message>` and `process.exit(1)` is called (lines 16-22). This happens through `console.error` and not the logger, because the logger itself depends on `env`. This means that the `bootstrap().catch` handler in server.ts never sees this error; the process dies earlier during module evaluation.

Env values come from `--env-file=.env` passed by the npm scripts (package.json "dev" and "start"); the `.env.example` documents the expected keys. Additional keys in `.env.example` (SHADOW_DATABASE_URL, MYSQL_*) are not part of the schema and are used by Prisma CLI and docker-compose.

**Rule workflow**:
```
process.env -> envSchema.safeParse -> success: export env
                                   -> failure: console.error(issues) -> exit(1)
```

---

### Business Rule: Composition and wiring of the application

**Overview**:
The bootstrap builds the app through `buildApp({ prisma })` and injects the shared Prisma client.

**Detailed description**:
`server.ts` does not construct controllers or routes. It passes the singleton `prisma` to `buildApp` (src/server.ts:7), which creates repositories, services and controllers manually (src/app.ts:26-53) and mounts `/health`, `/api/v1`, a catch-all 404 and the error middleware in that order (src/app.ts:62-73). Separating `buildApp` from `listen` allows tests to construct the app without opening a port (tests use `getTestApp()` from tests/helpers/factories.ts, which is referenced by tests/auth.test.ts:3; file content not analyzed in depth here as it is outside the component boundary).

Order matters: request logging precedes routes so every request is logged on `finish`; the 404 handler comes after the API router; the error middleware is last so it receives errors forwarded by `next(err)`.

**Rule workflow**:
```
prisma singleton -> buildApp -> buildControllers -> routers -> 404 -> errorMiddleware -> app.listen
```

---

### Business Rule: HTTP listener binding and start-up logging

**Overview**:
The server listens on the configured port and announces readiness.

**Detailed description**:
`app.listen(env.PORT, callback)` (src/server.ts:9) binds on all interfaces (no host argument). The callback logs `server_started` with `port` and `env` fields. The log is emitted by pino with base fields `service: order-management-api` and `env`. Note that the server is considered "started" when the socket is listening, not when the database is reachable: Prisma connects lazily on first query, so there is no readiness validation against MySQL. `GET /health` returns `{ status: 'ok' }` without checking the database (src/app.ts:62-64).

If the port is in use, Node emits an `error` event on the server. No `error` listener is registered in server.ts, so this surfaces as an uncaught exception (Node default behavior: crash, exit code 1). This is inferred from Node semantics, not from explicit code.

**Rule workflow**:
```
listen(PORT) -> callback -> logger.info('server_started', { port, env })
```

---

### Business Rule: Graceful shutdown on SIGINT / SIGTERM

**Overview**:
On termination signals the process stops the HTTP server, disconnects Prisma and exits with code 0.

**Detailed description**:
`shutdown(signal)` (src/server.ts:13-18) logs `shutdown_initiated`, calls `server.close()` with a callback logging `http_server_closed`, awaits `prisma.$disconnect()`, then calls `process.exit(0)`. Handlers are registered via `process.on('SIGINT' | 'SIGTERM', () => void shutdown(...))` (lines 20-21); `void` discards the promise, so a rejection from `$disconnect()` would become an unhandled rejection (and in modern Node, crash with a non-zero code).

Behavioral caveats: `server.close` only stops accepting new connections and waits for existing ones to end, but the function does not wait for its callback; `process.exit(0)` is called after the Prisma disconnect completes. Consequently the `http_server_closed` log may never be emitted and in-flight or keep-alive requests can be terminated abruptly. There is no timeout/force-kill path, no idempotency guard (a second signal triggers a second run), and exit code is always 0 even when shutdown is triggered by SIGINT. Orchestrators (SIGTERM, for example Docker) rely on this path for stopping the container. docker-compose.yml in the repository defines only MySQL (grep found no app service), so no container-level stop configuration exists for the API.

**Rule workflow**:
```
signal -> log shutdown_initiated -> server.close(cb, not awaited) -> await prisma.$disconnect() -> process.exit(0)
```

---

### Business Rule: Bootstrap failure handling

**Overview**:
Errors thrown within `bootstrap()` are logged as fatal and terminate the process with exit code 1.

**Detailed description**:
`bootstrap().catch(...)` (src/server.ts:24-27) logs `logger.fatal({ err }, 'bootstrap_failed')` and calls `process.exit(1)`. Since `bootstrap` is async but only contains synchronous operations before registering handlers (`buildApp`, `listen`), the catch covers mainly synchronous exceptions thrown in `buildApp` (for example during wiring). Errors from the listen operation (such as EADDRINUSE) are emitted asynchronously as events and are not caught by this handler. Errors during env validation also bypass it, as described above.

**Rule workflow**:
```
bootstrap() throws -> catch -> logger.fatal -> exit(1)
```

---

### Business Rule: Log redaction and level policy (shared with bootstrap logging)

**Overview**:
All logs, including bootstrap logs, pass through a pino logger that masks sensitive fields.

**Detailed description**:
`createLogger` (src/shared/logger/index.ts:13-30) sets level from `env.LOG_LEVEL`, redacts `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` with `[REDACTED]`, uses ISO timestamps, and attaches `service` and `env` base fields. In `development`, output goes through `pino-pretty`; otherwise JSON to stdout. The `fatal` log in bootstrap passes `{ err }`; pino's default serializer is not configured explicitly here (the `err` key is serialized by pino's standard `err` serializer by default).

**Rule workflow**:
```
logger.<level>(obj, msg) -> redact paths -> JSON (or pretty in development) -> stdout
```

---

### Business Rule: Prisma client lifecycle

**Overview**:
A single PrismaClient is created at import time, injected into the app and disconnected on shutdown.

**Detailed description**:
`src/config/database.ts:10` exports a singleton created by `createPrismaClient()`, with query-log level `['warn','error']` in development and `['error']` elsewhere. The bootstrap passes it to `buildApp` and disconnects it in `shutdown`. The same singleton is imported by `tests/setup.ts:2` for test-database cleanup, so tests and the runtime share the construction path. There is no explicit `$connect()` call in the bootstrap (tests do call `$connect()` in tests/setup.ts:5).

**Rule workflow**:
```
import -> new PrismaClient -> inject in buildApp -> lazy connect on first query -> $disconnect on shutdown
```

---

## 4. Component Structure

```
src/
├── server.ts                      # Entry point: bootstrap(), listen, signal handlers, fatal handler
├── app.ts                         # buildApp / buildControllers (composition root, invoked by server.ts)
├── config/
│   ├── env.ts                     # zod env validation, exits on invalid config
│   └── database.ts                # PrismaClient singleton
└── shared/
    └── logger/index.ts            # pino logger with redaction
Related build/run files:
├── package.json                   # scripts dev/build/start (entry: src/server.ts, dist/server.js)
├── tsconfig.build.json            # compiles src/**/*.ts to dist/
├── .env.example                   # expected environment variables
└── docker-compose.yml             # MySQL only (no API service)
```

## 5. Dependency Analysis

```
Internal Dependencies:
server.ts -> app.ts (buildApp) -> middlewares, modules (users, auth, customers, products, orders), routes/index.ts, shared/errors
server.ts -> config/env.ts
server.ts -> config/database.ts -> config/env.ts
server.ts -> shared/logger/index.ts -> config/env.ts

External Dependencies:
- express 4.21.1 - HTTP server (via app.listen, returns node http.Server)
- @prisma/client 5.22.0 - ORM / MySQL access, $disconnect
- pino 9.5.0 (+ pino-pretty 11.3.0 dev) - logging
- zod 3.23.8 - env validation (indirect)
- Node.js >= 20 (package.json engines): process signals, --env-file
- tsx 4.19.2 (dev runner), TypeScript 5.6.3 (build)
- MySQL (runtime data store, accessed lazily)
```

## 6. Afferent and Efferent Coupling

Units are modules/functions in this TypeScript codebase. Counts are based on static imports observed.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| server.ts (bootstrap) | 0 (entry point; invoked by npm scripts only) | 4 (app, env, database, logger) | Low coupling, High criticality (process lifecycle) |
| app.ts (buildApp) | 1 in src (server.ts) plus test helper (tests/helpers/factories.ts, inferred from getTestApp) | 20+ (middlewares, modules, routes, errors) | High |
| config/env.ts | Several (database.ts, logger, server.ts and others) | 1 (zod) | High |
| config/database.ts (prisma) | server.ts, tests/setup.ts, tests/orders.test.ts, others | 2 (env, @prisma/client) | High |
| shared/logger | server.ts, request-logger middleware and others | 2 (env, pino) | Medium |

## 7. Endpoints

The bootstrap itself defines no endpoints. It exposes the application built by `buildApp`; the one infrastructure endpoint relevant to process health is:

| Endpoint | Method | Description |
|----------|--------|-------------|
| /health | GET | Returns `{ status: 'ok' }` (src/app.ts:62-64); liveness only, no DB check |
| /api/v1/* | various | Business API mounted by `buildApp` (outside this component's scope) |

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL via Prisma | Database | Persistence, disconnect on shutdown | MySQL wire protocol | SQL/Prisma | None at bootstrap (lazy connect); `$disconnect` rejection unhandled |
| OS signals | Platform | SIGINT/SIGTERM trigger shutdown | POSIX signals | n/a | No timeout, no re-entrancy guard |
| Environment / .env file | Configuration | Runtime configuration | Process env, `--env-file` | key=value | zod validation, exit(1) |
| stdout (pino) | Logging | Structured logs | stdout | JSON / pretty | n/a |
| HTTP clients | Inbound | Serve API | HTTP/1.1 | JSON | errorMiddleware (in app.ts) |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Composition Root | `buildApp` / `buildControllers` with manual constructor injection | src/app.ts:26-76 | Centralised wiring, testable app |
| Application factory | `buildApp(deps)` separated from `listen` | src/app.ts:55, src/server.ts:7-9 | Allows tests without opening a port |
| Singleton | `env`, `prisma`, `logger` module-level exports | src/config/env.ts:27, src/config/database.ts:10, src/shared/logger/index.ts:32 | Shared instances |
| Fail-fast configuration | zod schema validation at import | src/config/env.ts:14-27 | Prevent running misconfigured |
| Graceful shutdown (partial) | Signal handlers calling close/disconnect | src/server.ts:13-21 | Resource cleanup |
| Top-level error boundary | `bootstrap().catch` | src/server.ts:24-27 | Fatal logging and exit |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Shutdown | `server.close()` callback not awaited before `process.exit(0)` (src/server.ts:15-17) | In-flight requests may be dropped; `http_server_closed` may never log |
| Medium | Shutdown | No timeout or forced termination; no guard against repeated signals | Possible hang with keep-alive connections or duplicate shutdown runs |
| Medium | Shutdown | Shutdown promise discarded with `void`; `$disconnect` rejection is unhandled (src/server.ts:20-21) | Unhandled rejection, inconsistent exit code |
| Medium | Process-level errors | No `unhandledRejection` / `uncaughtException` handlers, no server `error` listener (for example EADDRINUSE) | Crash without structured fatal log |
| Medium | Readiness | No DB connectivity check at start-up; `/health` does not probe the DB | Service reports healthy while DB is unreachable |
| Low | Exit code | Always 0 on shutdown, including after errors | Reduced signal for orchestrators |
| Low | Configuration | Env validation executes as an import side effect and calls `process.exit` | Hard to test; bypasses `bootstrap().catch` |
| Low | Network | `listen` without host binding | Binds all interfaces by default |
| Low | Deployment | No API service in docker-compose.yml; no Dockerfile found | Container stop behavior undefined |
| Low | Unused dependency | `pino-http` is listed in package.json but a custom `requestLogger` is used (grep of imports in analyzed files did not show it) | Possible dead dependency (not fully verified) |

## 11. Test Coverage Analysis

Test files located in the project: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`. Runner: vitest (vitest.config.ts: node environment, single fork, sequential, 30s timeouts, `setupFiles: tests/setup.ts`). No coverage provider or threshold is configured, so coverage percentages are not available.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| src/server.ts bootstrap | 0 | 0 | 0% (never imported by tests) | No tests for listen, signals, or fatal path |
| Env validation (src/config/env.ts) | 0 | Implicit (module loaded when tests run with valid env) | Not measured | No negative tests for invalid env |
| Prisma singleton (src/config/database.ts) | 0 | Used by tests/setup.ts (connect, cleanup, disconnect) | Not measured | Exercised indirectly only |
| buildApp (src/app.ts) | 0 | Indirectly through supertest in tests/auth.test.ts and tests/orders.test.ts via `getTestApp()` | Not measured | HTTP-level only; `/health` and 404 not verified in the files inspected |
| Graceful shutdown | 0 | 0 | 0% | Untested; risk noted in section 10 |

## Assumptions and ambiguities

- `tests/helpers/factories.ts` was not read in full; the claim that `getTestApp()` uses `buildApp` is inferred from naming and from the absence of server.ts imports in tests.
- EADDRINUSE and unhandled rejection behaviors are derived from Node.js semantics, not from code in the repository.
- Whether `pino-http` is unused was checked only against the files read.
