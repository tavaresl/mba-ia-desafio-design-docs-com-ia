# Architectural Analysis Report

Date: 2026-10-29
Scope: entire project root (no ignored folders)

## 1. Executive Summary

The project is a single-process, monolithic REST API ("order-management-api") for an Order Management System, written in TypeScript (ESM, Node >= 20) on Express 4, with MySQL 8 accessed through Prisma 5. It is organized as a modular monolith with a layered structure inside each business module (routes -> controller -> service -> repository -> Prisma), and a manual constructor-injection composition root in `src/app.ts`.

Business modules: auth, users, customers, products, orders. Cross-cutting infrastructure: configuration (zod-validated env), logging (pino), error model, request validation (zod), JWT authentication and role-based authorization.

Key findings:

- The Orders module is the architectural and business core. It concentrates the only multi-entity transactional logic (order creation, status transitions, stock debit/replenish, order number sequence) and is the largest file (`src/modules/orders/order.service.ts`, 255 lines) with direct Prisma access that bypasses its repository.
- Layering is inconsistent: `OrderService` receives both `OrderRepository` and a raw `PrismaClient`; `AuthService` depends on both `UserRepository` and `UserService`.
- Security-relevant design points: the public `/api/v1/auth/register` endpoint accepts a client-supplied `role` (including `ADMIN`); authorization (`requireRole`) is applied to only one route; there is no rate limiting, CORS, or security-header middleware.
- Stock consistency under concurrency relies on a read-then-write pattern inside a transaction without explicit row locking.
- Infrastructure is limited to a `docker-compose.yml` that runs MySQL only. There is no Dockerfile for the API, no CI/CD configuration, and no Kubernetes manifests.
- Only two test files exist (`tests/auth.test.ts`, `tests/orders.test.ts`) and run against a real database.
- Project documents (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/README.md`, `TRANSCRICAO.md`) exist; they were not used as a source of architectural truth beyond confirming scope. The analysis is based on code and configuration.

## 2. System Overview

```
project-root/
├── src/
│   ├── server.ts                 # Process bootstrap, graceful shutdown
│   ├── app.ts                    # Express app + composition root (manual DI)
│   ├── routes/index.ts           # API router aggregation (/api/v1)
│   ├── config/                   # env.ts (zod env validation), database.ts (PrismaClient singleton)
│   ├── middlewares/              # auth, validate, error, request-logger
│   ├── shared/
│   │   ├── errors/               # AppError hierarchy
│   │   ├── http/response.ts      # Pagination helpers
│   │   └── logger/index.ts       # pino logger with redaction
│   └── modules/
│       ├── auth/                 # routes, controller, service, schemas
│       ├── users/                # routes, controller, service, repository, schemas
│       ├── customers/            # routes, controller, service, repository, schemas
│       ├── products/             # routes, controller, service, repository, schemas
│       └── orders/               # routes, controller, service, repository, schemas, status (state machine)
├── prisma/                       # schema.prisma, migrations/, seed.ts
├── tests/                        # vitest + supertest integration tests, helpers, setup
├── docs/                         # PRD, RFC, FDD, TRACKER, ADR index, agent outputs
├── docker-compose.yml            # MySQL 8 only
├── .env.example, package.json, tsconfig*.json, vitest.config.ts, eslint/prettier config
```

Architectural patterns identified (documented separately):

1. Modular monolith with feature-based (vertical) modules under `src/modules`.
2. Layered architecture within each module: Routes -> Controller -> Service -> Repository -> Prisma (auth has no repository of its own and reuses the users repository).
3. Manual dependency injection via constructors, wired in `buildControllers` (`src/app.ts`); a composition root pattern.
4. Router/Controller factory pattern: routers are built by `build*Router(controller)` functions.
5. Middleware pipeline (Express chain): JSON parser -> request logger -> routes -> 404 -> error middleware.
6. State machine (lightweight) for order status in `src/modules/orders/order.status.ts`.
7. Centralized error handling via an `AppError` hierarchy mapped in `error.middleware.ts`.
8. Schema-first validation with zod at the HTTP boundary.
9. Transaction script style inside `OrderService` (interactive Prisma transactions); this deviates from the repository layering used elsewhere.

## 3. Critical Components Analysis

Coupling metrics: afferent coupling (Ca) is the number of other source modules (files) that import or depend on a component; efferent coupling (Ce) is the number of distinct modules (internal files and external packages) that a component imports. Values here were determined by manual inspection of import statements in `src/` (test files excluded unless stated) and are approximate counts at file or module-group level, not tool-generated.

| Component | Type | Location | Afferent Coupling | Efferent Coupling | Architectural Role |
|-----------|------|----------|-------------------|-------------------|-------------------|
| Application Bootstrap | Entry point | src/server.ts | 0 | 4 | Starts HTTP server, handles SIGINT/SIGTERM, disconnects Prisma |
| App Composition Root | Composition / Configuration | src/app.ts | 1 | 18 | Builds Express app, wires all repositories/services/controllers, global middleware, /health, 404 |
| API Router Aggregator | Routing | src/routes/index.ts | 1 | 10 | Mounts module routers under /api/v1 |
| Environment Config | Configuration | src/config/env.ts | 5 | 1 | Zod-validated environment; exits process on invalid config |
| Database Client | Infrastructure | src/config/database.ts | 2 (plus tests) | 2 | PrismaClient singleton |
| Prisma Schema and Migrations | Data model | prisma/schema.prisma, prisma/migrations/ | 5 repositories/services plus seed | 1 (MySQL) | Defines 7 entities, enums, indices; single source of persistence truth |
| Auth Middleware (authenticate, requireRole) | Security middleware | src/middlewares/auth.middleware.ts | 5 (route files) | 3 | JWT bearer verification and role check |
| Validation Middleware | Middleware | src/middlewares/validate.middleware.ts | 5 (route files) | 2 | Zod validation of body/query/params |
| Error Middleware | Middleware | src/middlewares/error.middleware.ts | 1 (app.ts) | 4 | Maps AppError, ZodError, Prisma errors to JSON responses |
| Request Logger Middleware | Middleware | src/middlewares/request-logger.middleware.ts | 1 (app.ts) | 2 | Request ID propagation and access logging |
| Error Model | Shared library | src/shared/errors/ | 8 | 0 | AppError hierarchy shared by middleware and services |
| Logger | Shared library | src/shared/logger/index.ts | 3 | 2 | pino logger with credential redaction |
| Pagination Helpers | Shared library | src/shared/http/response.ts | 3 (services) | 0 | Pagination response shape |
| Auth Module (AuthController, AuthService, routes, schemas) | Feature module | src/modules/auth/ | 2 (app.ts, routes index) | 8 | Registration, login, JWT issuing, /me |
| Users Module (UserController, UserService, UserRepository) | Feature module | src/modules/users/ | 3 (app.ts, auth service, routes index) | 6 | User creation (bcrypt), lookup, admin-only get-by-id |
| Customers Module (CustomerController, CustomerService, CustomerRepository) | Feature module | src/modules/customers/ | 2 (app.ts, routes index) | 6 | Customer CRUD with search and pagination |
| Products Module (ProductController, ProductService, ProductRepository) | Feature module | src/modules/products/ | 2 (app.ts, routes index) | 6 | Product catalog CRUD with SKU, active flag, stock quantity |
| Orders Module (OrderController, OrderService, OrderRepository, order.schemas) | Feature module (core domain) | src/modules/orders/ | 2 (app.ts, routes index) | 9 | Order creation, listing, status workflow, deletion; stock side effects |
| Order Status State Machine | Domain logic | src/modules/orders/order.status.ts | 1 (OrderService) | 1 | Allowed transitions and stock debit/replenish rules |
| Order Number Sequence | Data / domain mechanism | prisma/schema.prisma (OrderNumberSequence), src/modules/orders/order.service.ts (reserveOrderNumber) | 1 | 1 | Sequential ORD-NNNNNN numbering via single-row counter |
| Seed Script | Tooling | prisma/seed.ts | 0 | not inspected in detail | Database seeding |
| Test Suite | Quality infrastructure | tests/ | 0 | app, database, factories | Integration tests for auth and orders against real MySQL |
| MySQL Container | Infrastructure | docker-compose.yml | 1 (Prisma client at runtime) | 0 | Local database service |

## 4. Dependency Mapping

```
High-Level Dependencies:
server.ts -> app.ts -> routes/index.ts -> modules/*/routes -> middlewares (authenticate, validate)
modules/*/routes -> Controllers -> Services -> Repositories -> PrismaClient -> MySQL
app.ts (composition root) -> all Repositories, Services, Controllers
AuthService -> UserRepository, UserService (bcrypt, jsonwebtoken)
UserService -> UserRepository (bcrypt)
OrderService -> OrderRepository AND PrismaClient (direct transactions) -> order.status
Services -> shared/errors, shared/http/response
Middlewares -> shared/errors, shared/logger, config/env
shared/logger -> config/env
config/database -> config/env
```

Module-level view:

```
auth ------> users
orders ----> (customers, products, users data accessed via Prisma tx directly, not via their modules)
all modules --> shared/errors
customers, products, orders --> shared/http/response
```

Observations:

- Business modules are mostly independent of one another at the code level; the only inter-module import is auth -> users.
- Orders are coupled to customers, products, and users at the data level (foreign keys, and direct `tx.customer` / `tx.product` access in `OrderService`), but not through their service or repository classes. This creates implicit coupling to the Prisma schema of other modules.
- `src/config/env.ts` is a hub with process-exiting side effects at import time; every module that touches env (logger, database, auth) inherits this behavior, including in tests.
- No circular dependencies were observed.

## 5. Integration Points

| Integration | Type | Location | Purpose | Risk Level |
|-------------|------|----------|---------|------------|
| MySQL 8.0 | Database | docker-compose.yml, prisma/schema.prisma, src/config/database.ts | Sole persistent data store | Medium |
| Prisma ORM / Migrations | ORM / schema management | prisma/, package.json | Data access and migrations; shadow database configured via `SHADOW_DATABASE_URL` | Medium |
| JWT (jsonwebtoken, HS256 default) | Authentication mechanism | src/modules/auth/auth.service.ts, src/middlewares/auth.middleware.ts | Stateless access tokens signed with a shared secret | High |
| bcrypt | Library | src/modules/users/user.service.ts, src/modules/auth/auth.service.ts | Password hashing (10 rounds) | Low |
| pino / pino-http | Logging | src/shared/logger/index.ts | Structured logs to stdout; pino-pretty transport in development | Low |
| HTTP clients (inbound) | API consumers | src/routes/index.ts | REST API at /api/v1 plus /health | Medium |

No external third-party APIs (payments, email, messaging queues, caches, object storage) were found in the code.

## 6. Architectural Risks & Single Points of Failure

| Risk Level | Component | Issue | Impact | Details |
|------------|-----------|--------|--------|---------|
| High | MySQL (single instance) | Single point of failure | System-wide | All state resides in one MySQL instance; compose file defines one container with no replication. Every endpoint except /health depends on it. |
| High | OrderService stock handling | Read-check-then-decrement without row locks | Data integrity | `debitStock` reads stock and then decrements in the same transaction; under default MySQL isolation (REPEATABLE READ) concurrent transitions to PAID may both pass the check. The decrement is not conditional, so negative stock is possible. This is an inference from the code; no concurrency test was found. |
| High | OrderService | Bypasses repository layer; large transaction script | Maintainability, coupling | Direct `tx.customer`, `tx.product` access couples orders to other modules' tables; concentrates business rules in one 255-line class. |
| Medium | OrderNumberSequence | Single-row counter contended by every order creation | Throughput | All order creations upsert and increment the same row (id = 1) inside the transaction, serializing order creation. |
| Medium | Order status change | No optimistic concurrency / lock on order row | Consistency | Status is read and then updated in a transaction without version check; concurrent changes could both validate against the same prior status. |
| Medium | Config module | Process exit at import time | Testability, resilience | `env.ts` calls `process.exit(1)` on invalid configuration during module load. |
| Medium | Graceful shutdown (server.ts) | `process.exit(0)` invoked right after `prisma.$disconnect()` without awaiting server close | In-flight requests | `server.close` callback is not awaited, so in-flight requests may be cut off. |
| Medium | Listing endpoints | Offset pagination; `search` and filters on non-indexed patterns possible | Scalability | Orders list uses skip/take with count in a transaction; customers/products search implementation was not inspected for index usage. |
| Medium | Test coverage | Only auth and orders tested; no customers/products/users tests | Regression risk | Two test files exist. |
| Low | Inconsistent layering | AuthService depends on UserRepository and UserService simultaneously; auth module has no repository | Architectural consistency | Minor duplication of responsibility. |
| Low | Duplicate schemas | `registerSchema` and `createUserSchema` are identical | Drift risk | Defined in auth and users modules separately. |

Architectural debt summary: mixed layering in Orders, duplicated user-creation schema, authorization concentrated in a single route, and single-instance infrastructure.

## 7. Technology Stack Assessment

| Area | Technology | Version (package.json) | Notes |
|------|-----------|------------------------|-------|
| Language / runtime | TypeScript, Node.js | TS 5.6.3, Node >= 20 | ESM (`"type": "module"`), `.js` import suffixes |
| Web framework | Express | 4.21.1 | Express 4 (synchronous error handling semantics; controllers are assumed to forward async errors to `next`, not verified for each controller) |
| ORM | Prisma | 5.22.0 | MySQL provider, migrations, shadow DB |
| Database | MySQL | 8.0 (Docker image) | utf8mb4, `mysql_native_password` default authentication plugin |
| Validation | zod | 3.23.8 | Env and request validation |
| Auth | jsonwebtoken, bcrypt | 9.0.2, 5.1.1 | Stateless JWT, role claim in token |
| Logging | pino, pino-http, pino-pretty | 9.5.0, 10.3.0, 11.3.0 | `pino-http` is listed as a dependency but a custom request logger is used in code (no import of pino-http was found in src) |
| IDs | uuid, Prisma `uuid()` | 11.0.3 | Request IDs via uuid v4; entity IDs via Prisma default |
| Testing | vitest, supertest | 2.1.4, 7.0.0 | Integration tests against real DB |
| Tooling | tsx, eslint, prettier | 4.19.2, 8.57.1, 3.3.3 | ESLint 8 line |

Dependencies are pinned to exact versions. A dedicated dependency audit was not part of this analysis.

## 8. Security Architecture and Risks

Security boundaries and controls:

- Authentication: stateless JWT bearer tokens verified in `authenticate`; payload (sub, email, role) is trusted without database re-validation on each request.
- Authorization: `requireRole` exists, but it is applied only to `GET /api/v1/users/:id` (ADMIN). Customers, products, and orders routers require authentication only; any authenticated user (including OPERATOR) can create, update, and delete customers, products, and orders.
- Input validation: zod schemas on body/query/params through `validate`; JSON body limited to 1 MB.
- Error handling: internal errors return a generic 500 message; stack details are logged only.
- Logging: pino redaction for authorization header, cookies, password, passwordHash, token, accessToken.
- Secrets: loaded from environment; `.env` is git-ignored; `.env.example` contains placeholder credentials.

Risks:

| Severity | Area | Issue | Details |
|----------|------|-------|---------|
| Critical | Registration endpoint | Privilege escalation | `POST /api/v1/auth/register` is unauthenticated and its schema accepts `role` (`ADMIN` or `OPERATOR`), so any caller can create an ADMIN account. |
| High | Authorization model | Coarse RBAC | Role checks apply to one route only; destructive operations are open to all authenticated users. |
| High | JWT secret management | Shared symmetric secret | `.env.example` ships a predictable placeholder secret; validation only enforces a minimum length of 16 characters, so a placeholder value would pass. No key rotation or token revocation mechanism exists. |
| High | Authentication hardening | No rate limiting or lockout | Login and register endpoints have no throttling, enabling brute force and enumeration (register returns a distinct "Email already registered" conflict). |
| Medium | Token lifecycle | Long-lived access token, no refresh/revocation | Default `JWT_EXPIRES_IN` is 8h; role changes or user deletion are not reflected until expiry. |
| Medium | HTTP hardening | No security headers, CORS policy, or HTTPS termination in the codebase | Only `x-powered-by` is disabled; no helmet or CORS middleware was found. |
| Medium | Request ID handling | Client-supplied `x-request-id` is trusted and echoed into response headers and logs | Potential log forging or header injection vector depending on runtime validation. |
| Medium | Credentials in config | Default DB passwords in compose fallbacks and `.env.example`; MySQL port 3306 published to host | Intended for development; no separate production configuration exists. |
| Medium | Sensitive data | Customer `document` (identity document) and address stored in plaintext columns | No field-level encryption observed. |
| Low | Password policy | Minimum length 8, maximum 72 (bcrypt limit); no complexity rules | Acceptable bounds for bcrypt, limited strength policy. |

## 9. Infrastructure Analysis

Files present: `docker-compose.yml`, `.env.example`, `package.json` scripts, Prisma migrations.

- Containerization: only a MySQL 8.0 service (`oms-mysql`) with a named volume `oms_mysql_data`, restart policy `unless-stopped`, a mysqladmin-based healthcheck, and utf8mb4 settings. The application itself is not containerized (no Dockerfile).
- Runtime: the API runs on the host via `tsx watch` (dev) or `node dist/server.js` (production start), loading `.env` through Node's `--env-file` flag.
- Build: `tsc -p tsconfig.build.json` into `dist/`.
- Database lifecycle: `prisma migrate dev` / `reset`, single migration `20260519182739_init`, plus `prisma/seed.ts`.
- Health: `GET /health` returns a static `{status: "ok"}` without checking database connectivity.
- Not present: CI/CD pipelines, Kubernetes manifests, Makefile, IaC, reverse proxy, monitoring/metrics, or deployment scripts. Production deployment architecture cannot be determined from the repository.
- Scalability note: the API holds no in-process state beyond the Prisma connection pool, so the process model itself is stateless; the database and order-number counter are the shared contention points.
