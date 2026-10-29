# Component Deep Analysis Report: App Composition Root (src/app.ts)

Analysis date: 2026-10-29 14:55:48. Scope: `src/app.ts` (76 lines), plus its direct collaborators for contract analysis (`src/routes/index.ts`, `src/server.ts`, middlewares, tests). No project files were modified.

## 1. Executive Summary

`src/app.ts` is the composition root of the Express 4.21.1 / TypeScript API. It exports three symbols:

- `AppDependencies` (type, `src/app.ts:22-24`): `{ prisma: PrismaClient }`.
- `buildControllers(prisma)` (`src/app.ts:26-53`): manual constructor-injection wiring of Repository -> Service -> Controller for five modules (users, auth, customers, products, orders).
- `buildApp(deps)` (`src/app.ts:55-76`): creates the Express instance, registers global middleware, the `/health` endpoint, the `/api/v1` router, a 404 fallthrough, and the global error handler.

The module has no side effects at import time and does not listen on a port. `src/server.ts:7` (production) and `tests/helpers/factories.ts:12` (tests) are its only callers, which is what lets the same app be exercised with supertest without a network listener.

Key findings:
- Clean, explicit dependency injection with no DI container; the PrismaClient is the single injected infrastructure dependency.
- Middleware ordering is correct for the handlers involved (JSON parser, logger, routes, 404, error).
- `UserRepository` and `UserService` instances are shared between the users and auth modules; all other instances are module-private.
- No CORS, security headers (helmet), rate limiting, trust proxy, or readiness check is configured in this file. `/health` does not check the database.
- `buildControllers` is exported but only used internally (no external references found).
- Direct tests of `app.ts` do not exist; it is covered only indirectly via two integration test files.

## 2. Data Flow Analysis

Build-time (composition) flow:

```
1. server.ts bootstrap() (or tests getTestApp()) calls buildApp({ prisma })
2. buildApp creates express() instance
3. Disables x-powered-by header
4. Registers express.json({ limit: '1mb' })
5. Registers requestLogger middleware
6. Registers GET /health inline handler
7. buildControllers(prisma) builds repositories, services, controllers
8. buildApiRouter(controllers) is mounted at /api/v1
9. Registers catch-all 404 middleware (forwards NotFoundError)
10. Registers errorMiddleware (4-arg error handler)
11. Returns Express app
```

Runtime request flow through the root:

```
1. HTTP request enters Express
2. express.json parses body (JSON up to 1mb; oversize or malformed JSON raises an error handled in step 9)
3. requestLogger assigns req.id (x-request-id header or uuid v4), sets X-Request-Id response header, starts timer
4a. GET /health -> 200 {"status":"ok"} (ends here)
4b. /api/v1/* -> buildApiRouter -> per-module router -> authenticate/requireRole/validate -> Controller -> Service -> Repository -> Prisma
5. Unmatched route -> 404 middleware creates NotFoundError("Route <METHOD> <originalUrl>")
6. Any thrown/next(err) error -> errorMiddleware
7. errorMiddleware maps AppError / ZodError / Prisma P2002, P2025 / fallback 500 to JSON { error: { code, message, details? } }
8. On response "finish", requestLogger emits http_request log (requestId, method, path, status, durationMs, userId)
```

Note: errors raised by `express.json` (malformed body, payload too large) occur before `requestLogger`, so `req.id` is not set; `errorMiddleware` falls back to `'unknown'` for the request id (`src/middlewares/error.middleware.ts:56`). Those errors have no `AppError` type and would be handled by the 500 fallback (confidence: medium; body-parser errors carry a status but errorMiddleware does not read `err.status`).

## 3. Business Rules & Logic

The component holds infrastructure/wiring rules rather than domain rules. Domain rules live in module services and are out of scope.

## Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Security | Hide framework fingerprint (`x-powered-by` disabled) | src/app.ts:58 |
| Validation / Limit | JSON request bodies capped at 1 MB | src/app.ts:59 |
| Observability | Every request gets a correlation id and a completion log, registered before all routes | src/app.ts:60 |
| Operational | Unauthenticated liveness endpoint `GET /health` returning 200 `{status:'ok'}` | src/app.ts:62-64 |
| Routing | All business API is versioned under `/api/v1` | src/app.ts:67 |
| Wiring | Users repo/service shared with Auth; Orders service additionally receives raw PrismaClient | src/app.ts:27-44 |
| Error handling | Unknown routes produce 404 NOT_FOUND via the error pipeline | src/app.ts:69-71 |
| Error handling | Global error middleware is last in the chain | src/app.ts:73 |
| Isolation | Dependencies injected (no global prisma import in app.ts), one app per call | src/app.ts:22-24, 55-56 |

## Detailed breakdown of the business rules:
---

### Business Rule: Framework fingerprint suppression

**Overview**:
The `X-Powered-By: Express` header is disabled on every response.

**Detailed description**:
`app.disable('x-powered-by')` (`src/app.ts:58`) is the first statement after instance creation. Express would otherwise advertise itself on every response, including error and health responses, enabling trivial technology fingerprinting by scanners.

The setting is application-wide and applies to all routes and sub-routers, since it is an app setting rather than a middleware. It is therefore unaffected by registration order.

It is the only hardening measure in this file. Helmet-style headers, CORS and rate limiting are not present here (searched in `src/`; not found in app.ts). This is a partial hardening rather than a complete one.

**Rule workflow**:
```
buildApp -> app.disable('x-powered-by') -> all responses omit X-Powered-By
```

---

### Business Rule: Request body size limit (1 MB)

**Overview**:
JSON bodies larger than 1 MB are rejected by the parser.

**Detailed description**:
`express.json({ limit: '1mb' })` (`src/app.ts:59`) parses `application/json` bodies. It raises the limit above/below Express's default of 100kb to 1 MB, allowing moderately large payloads (for example orders with many items) while bounding memory use per request.

The parser is registered before the logger and routes, so every route receives `req.body` already parsed. Only JSON is parsed; urlencoded and multipart are not supported by this root.

When the limit is exceeded or JSON is malformed, body-parser calls `next(err)` with an error carrying `status`/`type` fields. `errorMiddleware` does not inspect those fields, so such errors would fall to the generic 500 branch rather than a 4xx (inferred from reading `error.middleware.ts`; the behavior is not covered by tests, confidence medium).

**Rule workflow**:
```
Request -> express.json -> (size ok, valid JSON) -> req.body populated -> next
                        -> (too large / malformed) -> next(err) -> errorMiddleware
```

---

### Business Rule: Request correlation and access logging precedes routing

**Overview**:
`requestLogger` is mounted globally before any route, including `/health`.

**Detailed description**:
`app.use(requestLogger)` (`src/app.ts:60`) ensures every request that reaches it receives `req.id` (taken from the inbound `x-request-id` header or generated with uuid v4), an `X-Request-Id` response header, and a structured `http_request` log on `finish` with method, path, status, duration and `userId` (`src/middlewares/request-logger.middleware.ts:5-28`).

Because it is mounted before `/health`, health probes are also logged, which may be noisy under frequent probing. Because it is mounted after `express.json`, body-parse failures are not logged by this middleware with a request id.

The 404 and error responses are also logged because the `finish` listener is attached early; `userId` is populated only if `authenticate` ran. The inbound header is trusted without validation (an observation, not a change recommendation).

**Rule workflow**:
```
Request -> requestLogger (set req.id, header, start timer) -> ... handler ... -> res "finish" -> logger.info('http_request')
```

---

### Business Rule: Liveness endpoint

**Overview**:
`GET /health` always answers 200 with `{ "status": "ok" }`.

**Detailed description**:
The handler is defined inline (`src/app.ts:62-64`) outside the `/api/v1` prefix and before `buildApiRouter`, so it is unversioned and bypasses `authenticate` and `validate`. It is intended for container/orchestrator liveness probing (docker-compose.yml exists in the repo; whether it references this endpoint was not verified).

It does not test database connectivity. A process with a broken Prisma connection will still answer 200. This is therefore liveness, not readiness.

It is subject to `express.json` and `requestLogger` since those are registered earlier.

**Rule workflow**:
```
GET /health -> express.json -> requestLogger -> handler -> 200 {"status":"ok"}
```

---

### Business Rule: API versioning prefix

**Overview**:
All module routers are mounted under `/api/v1`.

**Detailed description**:
`app.use('/api/v1', buildApiRouter(controllers))` (`src/app.ts:67`). `buildApiRouter` (`src/routes/index.ts:21-31`) then mounts `/auth`, `/users`, `/customers`, `/products`, `/orders`. The prefix is hardcoded in app.ts; the only other occurrence is in test code (`tests/*.ts`).

Authentication is not applied at this level. It is applied per router (`router.use(authenticate)` in customers, products, orders; per-route in auth `/me` and users `/:id` with `requireRole('ADMIN')`). `/auth/register` and `/auth/login` are public. A new version prefix would require a new mount in this file.

**Rule workflow**:
```
/api/v1/<module>/... -> buildApiRouter -> module router -> route-level middleware chain -> controller
```

---

### Business Rule: Dependency wiring and instance sharing

**Overview**:
Manual constructor injection builds the object graph per `buildApp` call.

**Detailed description**:
`buildControllers` (`src/app.ts:26-53`) instantiates, in order: `UserRepository(prisma)` -> `UserService(userRepository)` -> `UserController(userService)`; `AuthService(userRepository, userService)` -> `AuthController(authService, userService)`; `CustomerRepository` -> `CustomerService` -> `CustomerController`; `ProductRepository` -> `ProductService` -> `ProductController`; `OrderRepository(prisma)` -> `OrderService(orderRepository, prisma)` -> `OrderController`.

The users repository and service are singletons within one graph and are shared with auth (AuthService uses the repository for credential lookup and UserService for creation/public mapping, `src/modules/auth/auth.service.ts:22-33`). Orders is the only service that receives `PrismaClient` directly in addition to its repository (`src/modules/orders/order.service.ts:27-30`), which is consistent with it opening transactions at the service layer (the transaction usage itself was not analyzed here).

The returned `Controllers` object (`src/routes/index.ts:13-19`) is the typed contract: adding a module requires editing both `Controllers` and `buildControllers`. TypeScript enforces completeness of the returned object. Calling `buildApp` multiple times produces independent graphs sharing the same PrismaClient; tests cache one app (`tests/helpers/factories.ts:8-15`).

**Rule workflow**:
```
prisma -> *Repository -> *Service -> *Controller -> Controllers{auth,users,customers,products,orders} -> buildApiRouter
```

---

### Business Rule: Unknown routes return 404 through the error pipeline

**Overview**:
Unmatched requests yield `NotFoundError("Route <METHOD> <originalUrl>")`.

**Detailed description**:
The middleware at `src/app.ts:69-71` is registered after `/health` and `/api/v1`, so it only receives requests no route handled. It forwards `new NotFoundError(...)`, whose constructor yields message `"Route GET /x not found"`, status 404, code `NOT_FOUND` (`src/shared/errors/http-errors.ts:25-29`).

Because it uses `next(err)`, the response format is identical to all other API errors: `{ error: { code, message } }`, produced by `errorMiddleware`. It applies to any HTTP method and any path including outside `/api/v1`. The original URL, including query string, is echoed in the message (reflected in JSON only).

**Rule workflow**:
```
Unmatched request -> 404 middleware -> next(NotFoundError) -> errorMiddleware -> 404 JSON
```

---

### Business Rule: Centralized error mapping as terminal middleware

**Overview**:
`errorMiddleware` is registered last and normalizes all errors.

**Detailed description**:
`app.use(errorMiddleware)` (`src/app.ts:73`) must come after all routes for Express to route errors to it. Its mapping (`src/middlewares/error.middleware.ts:14-65`): `AppError` -> its statusCode/errorCode/details; `ZodError` -> 400 `VALIDATION_ERROR` with issue list; Prisma `P2002` -> 409 `CONFLICT`; Prisma `P2025` -> 404 `NOT_FOUND`; everything else (including other Prisma codes) -> logged and 500 `INTERNAL_SERVER_ERROR` with a generic message (no stack leakage to client).

Since the handler is added at the root, any module throwing is covered without per-module try/catch (async handlers must still forward rejections; handler wrapping is outside this component and was not verified, Express 4 does not catch async rejections natively).

**Rule workflow**:
```
error -> errorMiddleware -> AppError | ZodError | Prisma(P2002,P2025) | fallback 500 -> JSON
```

---

### Business Rule: Injected infrastructure, no import-time side effects

**Overview**:
`app.ts` receives `PrismaClient` as a parameter and does not import config singletons.

**Detailed description**:
`AppDependencies` (`src/app.ts:22-24`) keeps `app.ts` free of `env`/`database` imports (the type import of `PrismaClient` is `import type`). `server.ts` supplies the real client (`src/server.ts:3,7`), and tests supply the same one via `tests/helpers/factories.ts`. However, transitive imports (services, middlewares) such as `auth.service.ts` import `env` directly (`src/modules/auth/auth.service.ts:4`), so environment loading still occurs at import of `app.ts`. This limits true isolation.

Listening, graceful shutdown (SIGINT/SIGTERM with `prisma.$disconnect`) and process exit are in `server.ts`, not in this component.

**Rule workflow**:
```
server.ts / tests -> buildApp({prisma}) -> app (not listening)
```

---

## 4. Component Structure

```
src/
├── app.ts                         # Composition root: buildControllers, buildApp, AppDependencies
├── server.ts                      # Caller: listen, shutdown (outside component)
├── routes/
│   └── index.ts                   # Controllers type, buildApiRouter (mount points)
├── middlewares/
│   ├── request-logger.middleware.ts  # Registered by app.ts
│   └── error.middleware.ts           # Registered by app.ts
└── modules/{users,auth,customers,products,orders}/
    └── *.repository.ts / *.service.ts / *.controller.ts   # Instantiated by app.ts
```

`src/app.ts` is a single file with two functions and one type; no configuration of its own.

## 5. Dependency Analysis

```
Internal Dependencies:
buildApp -> buildControllers -> {User,Customer,Product,Order}Repository -> {..}Service -> {..}Controller
buildControllers -> AuthService -> (UserRepository, UserService); AuthController -> (AuthService, UserService)
OrderService -> PrismaClient (direct)
buildApp -> requestLogger, errorMiddleware, buildApiRouter, NotFoundError

Callers:
server.ts -> buildApp
tests/helpers/factories.ts -> buildApp

External Dependencies:
- express 4.21.1 - HTTP framework (app, json parser)
- @prisma/client 5.22.0 - type import only in app.ts (runtime via repositories)
- supertest 7.0.0 / vitest 2.1.4 - test-side consumers
```

Imports use ESM `.js` extension specifiers (`src/app.ts:3-20`).

## 6. Afferent and Efferent Coupling

Counts are for app.ts-level symbols and its direct wiring (static imports/usages found in `src` and `tests`).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| buildApp | 2 (server.ts, tests/helpers/factories.ts) | 4 (express, requestLogger, errorMiddleware, buildApiRouter) + NotFoundError + buildControllers | High |
| buildControllers | 1 (buildApp; exported, no external use found) | 15 concrete classes (5 repos/services/controllers minus auth repo, plus AuthService/AuthController) + PrismaClient | High |
| AppDependencies | 1 (buildApp) + 2 indirect callers | 1 (PrismaClient) | Low |
| app.ts module overall | 2 | 19 import statements (17 local modules/classes, express, @prisma/client) | High |

## 7. Endpoints

Endpoints defined directly in this component:

| Endpoint | Method | Description |
|----------|--------|-------------|
| /health | GET | Liveness check, returns 200 `{"status":"ok"}` |
| (any unmatched path) | ALL | 404 NOT_FOUND via NotFoundError |

Mounted (defined in module routers, summarized from `rg` of `*.routes.ts`):

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/auth/register, /login | POST | Public registration and login |
| /api/v1/auth/me | GET | Current user (authenticated) |
| /api/v1/users/:id | GET | ADMIN only |
| /api/v1/customers, /products, /orders | GET, POST | List/create (authenticated) |
| /api/v1/{customers,products,orders}/:id | GET, PATCH, DELETE | Read/update/delete (authenticated); the PATCH route definitions were not individually inspected |

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| PostgreSQL via PrismaClient | Database (injected) | Persistence for all modules | Prisma client | Typed objects | errorMiddleware maps P2002/P2025; other errors 500 |
| HTTP clients | Inbound | API consumers | HTTP/1.1 | JSON (1 MB) | JSON error envelope |
| Logger (pino-style `logger`) | Internal | Structured logs | In-process | JSON | n/a |
| Container/orchestrator probes | Inbound | Liveness | HTTP GET /health | JSON | None (always 200) |

The logger library identity was not verified in app.ts; it is imported from `src/shared/logger/index.js`.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Composition Root | `buildControllers` + `buildApp` | src/app.ts:26-76 | Single place where object graph is assembled |
| Constructor / Dependency Injection (manual) | Constructors receive repositories/services/PrismaClient | src/app.ts:27-44 | Testability, no container |
| Factory Function | `buildApp`, `buildControllers`, `buildApiRouter` | src/app.ts, src/routes/index.ts | App creation without listening |
| Layered Architecture | Controller -> Service -> Repository | src/modules/* | Separation of concerns |
| Chain of Responsibility (middleware pipeline) | Ordered `app.use` | src/app.ts:58-73 | Cross-cutting concerns |
| Centralized Error Handling | `errorMiddleware` last | src/app.ts:73 | Uniform error envelope |
| Router composition | `buildApiRouter` | src/routes/index.ts:21 | Modular routing |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | /health | Liveness only; no DB check | Probes may report healthy while DB is down |
| Medium | Middleware chain | No CORS, helmet, rate limiting, trust proxy in root | Reduced defense in depth (verify whether handled upstream/proxy) |
| Medium | express.json errors | Body-parser errors (413, malformed JSON) not mapped by errorMiddleware, occur before request id assignment | Likely 500 for client errors; no request id in logs |
| Low | requestLogger | Trusts inbound `x-request-id` without validation; /health logged | Log noise / log forging |
| Low | Wiring | `buildControllers` hand-maintained; `Controllers` type and function must change together | Omissions caught by compiler only for missing keys |
| Low | Isolation | `PrismaClient` injected but `env` imported transitively | Tests need env/DB present |
| Low | Exports | `buildControllers` exported without external consumers | Wider public surface than needed |
| Low | Reflected input | 404 message echoes `originalUrl` | Minor (JSON only) |

## 11. Test Coverage Analysis

Test files found: `tests/auth.test.ts` (7 tests), `tests/orders.test.ts` (8 tests), `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts`. No test imports or targets `app.ts` directly beyond `buildApp` via `getTestApp()`. No coverage tooling output or coverage config was found (no `coverage` block in `vitest.config.ts`), so percentages are not available.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| buildApp (assembly, /api/v1 mount, error middleware) | 0 | 15 indirect (auth.test.ts, orders.test.ts through supertest) | Not measured | Exercises routing, validation (400), 409/422 mapping via errorMiddleware |
| buildControllers | 0 | Indirect (auth and orders modules wired) | Not measured | Customers, products, users wiring not exercised by tests |
| GET /health | 0 | 0 | 0 (no test found) | Untested |
| 404 fallthrough | 0 | 0 (no test found) | 0 | Untested |
| x-powered-by, 1 MB limit, X-Request-Id | 0 | 0 | 0 | Untested |

Setup characteristics: Postgres-backed tests that truncate all tables in `beforeEach` (`tests/setup.ts:8-16`), single fork and no file parallelism (`vitest.config.ts:8-17`), 30s timeouts. The app instance is cached across tests (`tests/helpers/factories.ts:8-15`). The database is real, not mocked.
