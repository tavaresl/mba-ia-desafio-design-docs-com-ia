# Component Deep Analysis Report: Database Client (src/config/database.ts)

Analysis date: 2026-10-29 14:55:48
Scope: project root, no ignored folders. Analysis only; no files were modified, no code or tests were executed.

## 1. Executive Summary

The Database Client is a single 10-line TypeScript module (`src/config/database.ts`) that owns creation of the Prisma ORM client (`@prisma/client` 5.22.0, MySQL provider) used by the Order Management System (OMS). It exports:

- `createPrismaClient(): PrismaClient` (src/config/database.ts:4-8), a factory that builds a client with a log level chosen from `env.NODE_ENV`.
- `prisma: PrismaClient` (src/config/database.ts:10), a module-level singleton created at import time.

Role in the system: it is the composition-root provider of the database connection. `src/server.ts` injects the singleton into `buildApp({ prisma })`, which hands it to every repository and to `OrderService` (for transactions). The test suite (`tests/setup.ts`, `tests/helpers/factories.ts`, `tests/orders.test.ts`) imports the same singleton directly.

Key findings:
- Very small, cohesive, single-responsibility module with one business-relevant decision: log verbosity per environment.
- The connection URL is not passed in code. Prisma reads `DATABASE_URL` from the environment via `prisma/schema.prisma:6`. `env.ts` validates that `DATABASE_URL` is non-empty, and because `database.ts` imports `env.js`, validation runs before the client is built.
- No explicit connection pool, timeout, retry, health-check or query-logging configuration. Defaults of Prisma are used.
- Shutdown is handled outside the module (`src/server.ts:16`, `prisma.$disconnect()`).
- Singleton is created eagerly at import time as a side effect; any importer triggers env validation (which calls `process.exit(1)` on invalid config).
- No dedicated unit test for the module; it is exercised only indirectly through integration tests that require a live MySQL instance.

## 2. Data Flow Analysis

Initialization flow:
```
1. Process starts; an entry point imports src/config/database.ts (src/server.ts:3, tests/setup.ts:2).
2. database.ts imports ./env.js (database.ts:2), which executes loadEnv() at import (env.ts:25).
3. env.ts validates process.env with Zod; on failure it logs issues and calls process.exit(1) (env.ts:15-21).
4. database.ts evaluates createPrismaClient() (database.ts:10).
5. PrismaClient is constructed with log: ['warn','error'] if NODE_ENV === 'development', otherwise ['error'] (database.ts:5-7).
6. Prisma lazily connects on first query (no explicit $connect in production code); DATABASE_URL is resolved by Prisma from the datasource block (prisma/schema.prisma:6).
7. The singleton is exported as `prisma`.
```

Runtime consumption flow:
```
1. src/server.ts:7 calls buildApp({ prisma }).
2. src/app.ts:26-66 buildControllers(prisma) constructs UserRepository, CustomerRepository, ProductRepository, OrderRepository with the client (app.ts:27,34,38,42) and OrderService(orderRepository, prisma) (app.ts:43).
3. Repositories issue queries (findUnique, findMany, create, update, delete, $transaction batch for list+count).
4. OrderService.create / transition use prisma.$transaction(async tx => ...) (order.service.ts:58,131) for atomic multi-table work.
5. Prisma errors propagate to error.middleware.ts, which maps P2002 -> 409 and P2025 -> 404 (error.middleware.ts:37-54); others are logged and returned as unhandled errors.
```

Shutdown flow:
```
1. SIGINT/SIGTERM handler in server.ts:19-20 calls shutdown().
2. HTTP server close is requested (non-awaited callback), then await prisma.$disconnect() (server.ts:16), then process.exit(0).
```

Test flow:
```
1. tests/setup.ts beforeAll: prisma.$connect().
2. beforeEach: deleteMany on orderStatusHistory, orderItem, order, orderNumberSequence, product, customer, user (child-first order).
3. afterAll: prisma.$disconnect().
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Configuration | Prisma log levels are ['warn','error'] in development and ['error'] otherwise (test, production) | src/config/database.ts:6 |
| Lifecycle | A single shared PrismaClient instance (singleton) is created at module load | src/config/database.ts:10 |
| Lifecycle | Client creation is exposed as a factory so additional clients can be built | src/config/database.ts:4 |
| Validation (indirect) | Environment must be valid (DATABASE_URL non-empty, etc.) before client is created, else process exits with code 1 | src/config/env.ts:6-21 (imported at database.ts:2) |
| Connection (indirect) | DATABASE_URL (and SHADOW_DATABASE_URL for migrations) supplied via environment to MySQL datasource | prisma/schema.prisma:4-8 |
| Lifecycle (external) | Client is disconnected on SIGINT/SIGTERM | src/server.ts:16-20 |
| Error mapping (external) | Prisma P2002 -> HTTP 409, P2025 -> HTTP 404 | src/middlewares/error.middleware.ts:37-54 |

Only the first three rules are implemented inside the component; the remaining rows are contextual rules that directly govern the component's behavior.

### Detailed breakdown of the business rules

---

### Business Rule: Environment-dependent Prisma logging

**Overview**:
The verbosity of Prisma's built-in logger depends on `NODE_ENV`. Development emits warnings and errors; every other environment emits errors only.

**Detailed description**:
At src/config/database.ts:6 the expression `env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error']` selects the `log` option for `PrismaClient`. `NODE_ENV` is validated by Zod as one of `development | test | production` with default `development` (src/config/env.ts:4). Consequently, if `NODE_ENV` is unset, the application behaves as development and the client logs warnings and errors.

In test and production the client logs only errors. Query-level logging (`query`, `info`) is never enabled in any environment, so SQL statements are not emitted by Prisma. This limits log volume and reduces the chance of leaking query parameters into logs. Prisma's log entries are written to stdout/stderr by default, since no `$on` event subscription or `emit: 'event'` configuration exists; they are therefore not routed through the project's structured logger (`src/shared/logger/index.ts`). This is an observation of the code; no integration was found between Prisma log events and the logger.

The rule affects test runs (quiet output, only errors) and operations (production only surfaces errors). Confidence: high (explicit code).

**Rule workflow**:
```
env.NODE_ENV
  -> 'development' ? log = ['warn','error']
  -> else (test | production) log = ['error']
  -> new PrismaClient({ log })
```

---

### Business Rule: Single shared client instance (singleton)

**Overview**:
One `PrismaClient` is created when the module is first imported and shared by the whole process.

**Detailed description**:
`export const prisma = createPrismaClient()` (src/config/database.ts:10) leverages ES module caching: the module body runs once per process, so all importers (`src/server.ts:3`, `tests/setup.ts:2`, `tests/helpers/factories.ts:3`, `tests/orders.test.ts:9`) receive the same instance. A single client manages one connection pool, which is the pattern Prisma recommends to avoid exhausting database connections.

The client is injected rather than imported inside repositories: `buildApp({ prisma })` (server.ts:7; app.ts:23,66) and constructor injection in repositories and `OrderService` (app.ts:27-43). This allows alternate clients to be supplied by callers, although no test in the repository does so; tests use the real singleton against a real database.

There is no hot-reload guard (e.g., storing the client on `globalThis`). Under the vitest configuration (`pool: 'forks'`, `singleFork: true`, vitest.config.ts:11-15) a single process is used, so one client exists. Confidence: high.

**Rule workflow**:
```
first import of database.ts
  -> env loaded and validated
  -> createPrismaClient()
  -> exported const prisma
subsequent imports -> cached module -> same instance
```

---

### Business Rule: Client factory exposure

**Overview**:
`createPrismaClient` is exported separately so that a new, identically configured client can be created on demand.

**Detailed description**:
The factory (database.ts:4-8) takes no parameters and returns a configured `PrismaClient`. It is used only to produce the singleton within the same file; a search across `src`, `tests`, and `prisma` found no other caller (`prisma/seed.ts:1` constructs its own `new PrismaClient()` directly instead of using the factory, so seed scripts do not share the log policy; its exact construction was not analysed in detail).

Because the factory reads `env.NODE_ENV` at call time and `env` is frozen at module load, every call yields the same configuration. The factory provides a seam for testing or scripts but is currently unused beyond the singleton. Confidence: high for the usage statement; medium for the seed script's construction details.

**Rule workflow**:
```
caller -> createPrismaClient() -> new PrismaClient({ log }) -> return
```

---

### Business Rule: Mandatory valid environment before client creation (contextual)

**Overview**:
The client cannot be built unless environment validation succeeds.

**Detailed description**:
`database.ts` imports `env` from `./env.js` (line 2). `env.ts` parses `process.env` with a Zod schema at module evaluation (env.ts:25). `DATABASE_URL` must be a non-empty string (env.ts:7), and `JWT_SECRET` must be at least 16 characters (env.ts:8). On any failure, the issues are printed and the process exits with status 1 (env.ts:15-21).

As a result, importing the Database Client in any context (server, tests) hard-fails if the environment is incomplete, including variables unrelated to the database. Note that `env.ts` checks only that `DATABASE_URL` is non-empty; it does not verify MySQL URL format. Prisma itself validates the URL protocol when connecting. `SHADOW_DATABASE_URL` is required by `prisma/schema.prisma:7` for migration commands but is not part of the Zod schema.

**Rule workflow**:
```
import database.ts -> import env.ts -> safeParse(process.env)
  -> invalid: console.error + process.exit(1)
  -> valid: continue to createPrismaClient()
```

---

### Business Rule: Graceful disconnect on shutdown (contextual, outside component)

**Overview**:
The shared client is disconnected when the process receives SIGINT or SIGTERM.

**Detailed description**:
`shutdown` in src/server.ts:12-18 logs, requests HTTP server close, awaits `prisma.$disconnect()`, then calls `process.exit(0)`. The disconnect is part of the component's lifecycle but implemented by the caller. `server.close` is invoked with a callback and not awaited, so in-flight requests may still be using the database when `$disconnect()` runs; and `process.exit(0)` follows immediately. This is a potential ordering concern (see Section 10). There is no handling of errors thrown by `$disconnect()` and no shutdown timeout.

**Rule workflow**:
```
SIGINT/SIGTERM -> shutdown(signal) -> log -> server.close(cb) -> await prisma.$disconnect() -> process.exit(0)
```

---

### Business Rule: Prisma error translation (contextual, outside component)

**Overview**:
Database errors raised by the client are translated into HTTP responses by the global error middleware.

**Detailed description**:
`error.middleware.ts:37-54` handles `Prisma.PrismaClientKnownRequestError`. Code `P2002` (unique constraint) returns 409 with code `CONFLICT` and the violated target columns when present. Code `P2025` (record not found) returns 404 `NOT_FOUND`. Other Prisma error codes (e.g., P2003 foreign key, P1001 connection failure) and other Prisma error classes (initialization, validation, unknown request) are not handled specifically and fall through to the generic logging path (error.middleware.ts:57-60) that logs with request id. The generic response body after that point was not part of the analysed range.

**Rule workflow**:
```
Prisma query throws -> error middleware
  -> KnownRequestError P2002 -> 409 CONFLICT
  -> KnownRequestError P2025 -> 404 NOT_FOUND
  -> otherwise -> logger.error + generic handling
```

---

## 4. Component Structure

```
src/config/
├── database.ts   # Prisma client factory (createPrismaClient) and singleton (prisma)  [component]
└── env.ts        # Zod-validated environment (consumed by database.ts)

Related (outside component boundary):
prisma/
├── schema.prisma # datasource (mysql, DATABASE_URL, SHADOW_DATABASE_URL), models, enums
├── migrations/   # 20260519182739_init
└── seed.ts       # separate PrismaClient usage
src/server.ts     # imports singleton, injects, disconnects on shutdown
src/app.ts        # DI wiring of the client into repositories/services
tests/setup.ts    # connect, cleanup, disconnect lifecycle
docker-compose.yml  # MySQL 8.0 service backing the client
.env.example        # DATABASE_URL, SHADOW_DATABASE_URL
```

## 5. Dependency Analysis

```
Internal Dependencies:
database.ts -> config/env.ts (env.NODE_ENV)

Consumers (depend on database.ts):
server.ts -> database.ts (prisma)
tests/setup.ts, tests/helpers/factories.ts, tests/orders.test.ts -> database.ts (prisma)

Indirect (via injection, type-only dependency on PrismaClient):
app.ts -> UserRepository / CustomerRepository / ProductRepository / OrderRepository / OrderService

External Dependencies:
- @prisma/client 5.22.0 - ORM client (package.json:26)
- prisma 5.22.0 (devDependency, package.json:48) - schema tooling, migrations, client generation
- MySQL 8.0 (docker-compose.yml) - data store
- zod (via env.ts) - environment validation
```

Runtime dependency note: `@prisma/client` requires `prisma generate` to have produced the client from `prisma/schema.prisma`; generation steps were not examined beyond package scripts.

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| prisma (singleton export) | 4 files (src/server.ts, tests/setup.ts, tests/helpers/factories.ts, tests/orders.test.ts); 1 production runtime consumer | 1 (PrismaClient instance from @prisma/client) | High |
| createPrismaClient (function) | 1 (internal, database.ts:10) | 2 (PrismaClient, env) | Low |
| config/env (as used by database.ts) | 1 from this component (many overall) | 1 (zod) | Medium |

Counts are based on a grep of import statements; type-only `PrismaClient` imports in repositories and app.ts are a separate, structural dependency on the Prisma type, not on this module. At runtime the singleton reaches all four repositories and OrderService through injection, which makes it a high-criticality dependency.

## 7. Endpoints

Not applicable. The component does not expose endpoints.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (DATABASE_URL) | External database | Persistent storage for users, customers, products, orders | MySQL wire protocol via Prisma query engine | Prisma models / SQL | Prisma known-error mapping for P2002, P2025 in error middleware; no retry or circuit breaker |
| MySQL shadow database (SHADOW_DATABASE_URL) | External database | Prisma migrate dev shadow DB | MySQL | SQL | Migration tooling only |
| Environment (process.env) | Configuration | NODE_ENV and DATABASE_URL | n/a | Strings | Zod validation, process.exit(1) |
| Express app (buildApp) | Internal | Dependency injection of client | In-process | TypeScript object | n/a |
| Process signals | Runtime | Disconnect on shutdown | OS signals | n/a | None for disconnect failure |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Singleton (module-scoped) | `export const prisma` | src/config/database.ts:10 | Single connection pool per process |
| Factory Function | `createPrismaClient()` | src/config/database.ts:4 | Encapsulate client configuration |
| Dependency Injection (constructor) | `buildApp({ prisma })`, repositories take PrismaClient | src/server.ts:7, src/app.ts:26-66 | Decouple consumers from module import |
| Repository Pattern (consumers) | `*Repository` classes | src/modules/*/ *.repository.ts | Data access abstraction over client |
| Unit of Work / transaction script | `prisma.$transaction` | src/modules/orders/order.service.ts:58,131 | Atomic multi-entity operations |
| Fail-fast configuration | Zod parse then exit | src/config/env.ts:13-24 | Reject invalid environment at boot |

Architectural decision: configuration of infrastructure lives under `src/config/`, wired once at the composition root (`server.ts`).

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Shutdown (server.ts:12-18) | `server.close` not awaited before `$disconnect()`; `process.exit(0)` follows immediately; no timeout or error handling | In-flight requests may fail on database access during shutdown |
| Medium | Connection configuration | No explicit pool size, connect or query timeouts, or retry; defaults only (connection_limit may only be set via URL, none observed in .env.example) | Behavior under load or DB outage is not controlled in code |
| Medium | Observability | Prisma log output not routed through structured logger; no query logging or slow-query visibility | Inconsistent log format; limited diagnostics |
| Low | Import side effects | Client created at import; importing triggers env validation and may call `process.exit(1)` | Harder to import module in isolation (e.g. tooling) |
| Low | Seed script | prisma/seed.ts uses its own `new PrismaClient()` rather than the shared factory | Divergent logging configuration (construction detail not fully verified) |
| Low | Env validation | `DATABASE_URL` only checked for non-empty; `SHADOW_DATABASE_URL` not validated | Malformed URL detected only when Prisma connects |
| Low | Startup | No explicit connection check at boot; connection is lazy | DB unavailability surfaces on first request rather than at startup |
| Low | Error mapping | Only P2002 and P2025 mapped specially | Other DB errors (e.g., connection failure, FK violation) handled generically |
| Low | Test isolation | Tests use the real singleton and a live database and delete all rows in beforeEach | Requires a dedicated test database; accidental execution against a non-test DB would erase data (no guard on NODE_ENV or DB name found in tests/setup.ts) |
| Low | Hot reload | No globalThis caching guard | Multiple clients possible if the module is re-evaluated in dev reloaders (no such reloader found in the project) |

Security observations: the module contains no credentials; connection strings come from the environment. `.env.example` contains sample credentials for local development only. Query logging is disabled, which avoids parameter leakage.

## 11. Test Coverage Analysis

Test files located: `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`; runner configuration `vitest.config.ts` (setupFiles `tests/setup.ts`, sequential single fork, 30 s timeouts). No coverage tool configuration or coverage report was found.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| createPrismaClient / log level rule | 0 | 0 direct | Unknown (no coverage report); the function executes in any test run, but the log-level branches are not asserted and only the non-development branch is likely exercised unless NODE_ENV is development | No assertions on log configuration |
| prisma singleton | 0 | Indirect: tests/setup.ts lifecycle (connect, cleanup, disconnect); tests/helpers/factories.ts (user, customer, product creation); tests/orders.test.ts:85,130,150,159 (stock verification via prisma.product.findUnique) | Indirect only | Real database used; no mocks; good realism but depends on MySQL availability |
| Shutdown disconnect | 0 | 0 | None | Not tested |
| Error mapping P2002 / P2025 | Not identified as dedicated tests (not exhaustively searched in tests/auth.test.ts) | Possibly indirect | Unknown | Not confirmed |

Gaps: no test for the environment-dependent log level, for env validation failure, for connection failure behavior, or for graceful shutdown.
