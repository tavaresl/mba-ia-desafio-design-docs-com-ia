# mba-ia-desafio-design-docs-com-ia (order-management-api) - Project Overview

**Generated on**: 2026-10-29 15:00:30

## Summary

The project is a single-process REST API for an Order Management System, written in TypeScript (ESM, Node >= 20) on Express 4 with MySQL 8 accessed through Prisma 5. It is a modular monolith with five business modules (auth, users, customers, products, orders). Each module is layered as routes, controller, service, repository and Prisma, and wired by hand in a composition root (`src/app.ts`). The Orders module is the core domain: it holds the only multi-entity transactional logic (order creation, status transitions, stock debit and replenishment, order number sequence).

The architectural analysis identified 23 components, and all 23 were analyzed in depth. Infrastructure is limited to a `docker-compose.yml` that runs MySQL only. The repository has no Dockerfile for the API, no CI/CD configuration and no Kubernetes manifests. No external third-party integrations (payments, email, queues, caches) were found in the code.

The dependency audit covers 8 runtime and 19 dev dependencies, all pinned to exact versions, in a single npm ecosystem. `npm audit` reports 23 advisories against the lockfile (4 critical, 12 high, 7 moderate). Every direct dependency is behind its latest release. The automated test suite has 15 integration tests, covering only the auth and orders modules. No coverage tooling is configured, so no coverage percentages were measured in any report.

## Architecture Overview

- **Style:** modular monolith with feature-based modules under `src/modules`. Each module is layered as routes, controller, service, repository, Prisma. The auth module has no repository of its own and reuses the users repository.
- **Wiring:** manual constructor injection in `buildControllers` and `buildApp` (`src/app.ts`). Routers are built by `build*Router(controller)` factories and aggregated under `/api/v1` in `src/routes/index.ts`.
- **Request pipeline:** `x-powered-by` disabled, `express.json` (1 MB limit), request logger, `GET /health`, `/api/v1` router, 404 `NotFoundError`, error middleware.
- **Cross-cutting pieces:** Zod-validated environment (exits the process on invalid configuration), pino logger with redaction, an `AppError` hierarchy mapped by the error middleware, Zod request validation, and JWT bearer authentication with a role check.
- **Orders:** a table-driven status state machine (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, plus CANCELLED). Stock is debited on PENDING to PAID and replenished on cancellation from PAID or PROCESSING. Order numbers (`ORD-NNNNNN`) come from a single-row counter table reserved inside the order transaction.
- **Data:** 7 tables and a single `init` migration. Integrity rules in the database are four unique keys, money in integer cents, restrictive foreign keys and cascades from orders to items and history. The status-transition matrix, stock rules and delete restriction exist only in application code. The database has no CHECK constraints.
- **Layering exception:** `OrderService` receives both `OrderRepository` and a raw `PrismaClient`, and accesses customer and product tables directly in its transactions.
- **Module coupling:** the only inter-module import is auth to users. No circular dependencies were observed.
- **Deployment:** the API runs on the host via `tsx watch` (dev) or `node dist/server.js`, loading `.env` through `--env-file`. The production deployment architecture cannot be determined from the repository.

## Dependencies Health

Critical issues only:

- **vitest 2.1.4 (critical, dev):** remote code execution and file-read advisories when the Vitest API or UI server is listening (GHSA-9crc-q9x8-hgqq, GHSA-5xrq-8626-4rwp), plus a tinypool prototype-pollution RCE (GHSA-5gmw-xhrv-c9v3, GHSA-85c8-ppgw-ccpr).
- **express 4.21.1 (critical/high):** advisories via proxy-addr, body-parser, path-to-regexp and qs. The proxy-addr advisory matters only if `trust proxy` is configured, which `src/app.ts` does not do.
- **zod 3.23.8 (high):** GHSA-mrhx-c7xc-m3w5, memory exhaustion on oversized constrained collections.
- **bcrypt 5.1.1 (transitive critical):** a vulnerable `tar` through `@mapbox/node-pre-gyp`, relevant at install time.
- **Deprecated:** eslint 8.57.1 (end of support), `@types/uuid` (stub) and supertest 7.0.0 (registry deprecation notice).
- **Major versions behind:** express (4 vs 5), zod (3 vs 4), Prisma (5 vs 7), pino (9 vs 10), uuid (11 vs 14), vitest (2 vs 5), eslint (8 vs 10), bcrypt (5 vs 6) and TypeScript (5 vs 7).
- **Unused:** `pino-http` is declared but not imported anywhere.
- **Licenses:** all direct dependencies are MIT or Apache-2.0.
- **Unverified in the audit:** the `mysql:8.0` image tag and its end-of-life date, CVE mappings (only GHSA IDs were returned), and GitHub repository maintenance details. Prisma's npm `latest` tag points to a release candidate, so the audit used 7.10.1 as the stable latest.

## Components Analyzed

| # | Component | Summary |
|---|-----------|---------|
| 1 | [Application Bootstrap](./component-deep-analyzer/component-analysis-Application%20Bootstrap-2026-10-29%2014:55:48.md) | `src/server.ts` builds the app, listens on `env.PORT` and handles SIGINT/SIGTERM. `server.close()` is not awaited before `process.exit(0)`, there are no `unhandledRejection` or `uncaughtException` handlers, and `/health` does not probe the database. |
| 2 | [App Composition Root](./component-deep-analyzer/component-analysis-App%20Composition%20Root-2026-10-29%2014:55:48.md) | The 76-line `src/app.ts` wires repositories, services and controllers for five modules and sets the middleware order. No CORS, helmet or rate limiting is configured, and body-parser errors are probably not mapped (inferred, not tested). |
| 3 | [API Router Aggregator](./component-deep-analyzer/component-analysis-API%20Router%20Aggregator-2026-10-29%2014:55:48.md) | `src/routes/index.ts` mounts the five module routers under `/api/v1`. It applies no default authentication, so each sub-router must add its own. |
| 4 | [Environment Config](./component-deep-analyzer/component-analysis-Environment%20Config-2026-10-29%2014:55:48.md) | A Zod schema validates six variables at import time and calls `process.exit(1)` on failure. `JWT_SECRET` has only a 16-character minimum, and `JWT_EXPIRES_IN` is not checked as a duration. |
| 5 | [Database Client](./component-deep-analyzer/component-analysis-Database%20Client-2026-10-29%2014:55:48.md) | A 10-line PrismaClient factory and singleton. Prisma log levels depend on the environment, there is no pool or timeout configuration, and Prisma logs bypass the structured logger. |
| 6 | [Prisma Schema and Migrations](./component-deep-analyzer/component-analysis-Prisma%20Schema%20and%20Migrations-2026-10-29%2014:55:48.md) | 7 tables on MySQL 8 with a single `init` migration. Prisma error P2003 (foreign-key violation) is not mapped by the error middleware, and the database has no CHECK constraints. |
| 7 | [Auth Middleware](./component-deep-analyzer/component-analysis-Auth%20Middleware-2026-10-29%2014:55:48.md) | `authenticate` verifies a Bearer JWT and trusts the claims without a database lookup. `requireRole` is used only on `GET /users/:id`. Invalid, expired and non-Bearer tokens are untested. |
| 8 | [Validation Middleware](./component-deep-analyzer/component-analysis-Validation%20Middleware-2026-10-29%2014:55:48.md) | A Zod middleware factory used in 18 call expressions across 5 route files. It maps `ZodError` to a 400 `VALIDATION_ERROR`, and Zod issue formatting is duplicated in the error middleware. |
| 9 | [Error Middleware](./component-deep-analyzer/component-analysis-Error%20Middleware-2026-10-29%2014:55:48.md) | Handles `AppError`, `ZodError`, Prisma P2002 (409) and P2025 (404), then a generic 500. Other Prisma codes and body-parser errors fall to the 500 branch, and only the `AppError` branch has indirect test coverage. |
| 10 | [Request Logger Middleware](./component-deep-analyzer/component-analysis-Request%20Logger%20Middleware-2026-10-29%2014:55:48.md) | Reuses the inbound `x-request-id` or generates a UUID v4, and emits one `http_request` log per finished response. The client-supplied request ID is not validated, and aborted requests are not logged. |
| 11 | [Error Model](./component-deep-analyzer/component-analysis-Error%20Model-2026-10-29%2014:55:48.md) | An `AppError` base class with nine subclasses and a barrel file. Subclasses do not override `name`, `BadRequestError` is unused, and two order-domain errors live in the shared module. |
| 12 | [Logger](./component-deep-analyzer/component-analysis-Logger-2026-10-29%2014:55:48.md) | A pino 9.5.0 singleton with redaction. Redaction wildcards match only one nesting level, and `pino-pretty` is a devDependency used when `NODE_ENV` is development. |
| 13 | [Pagination Helpers](./component-deep-analyzer/component-analysis-Pagination%20Helpers-2026-10-29%2014:55:48.md) | A pure module exporting `buildPagination` and `paginated`, used by the customer, product and order services. The `skip` calculation and the `page`/`pageSize` Zod fields are duplicated across modules. |
| 14 | [Auth Module](./component-deep-analyzer/component-analysis-Auth%20Module-2026-10-29%2014:55:48.md) | Register, login and `/auth/me` endpoints, with bcrypt (10 rounds) and an HS256 JWT valid for 8h by default. Registration accepts a client-supplied `role`, and login has no rate limiting. |
| 15 | [Users Module](./component-deep-analyzer/component-analysis-Users%20Module-2026-10-29%2014:55:48.md) | One endpoint (`GET /users/:id`, ADMIN only). User creation is reached through `/auth/register`, enforces a unique email and strips `passwordHash` from responses. The email uniqueness check is read-then-insert. |
| 16 | [Customers Module](./component-deep-analyzer/component-analysis-Customers%20Module-2026-10-29%2014:55:48.md) | Five CRUD endpoints with search and pagination, open to any authenticated role. `document` and `phone` get only length checks, and deleting a customer with orders probably returns 500 (inferred). |
| 17 | [Products Module](./component-deep-analyzer/component-analysis-Products%20Module-2026-10-29%2014:55:48.md) | Five CRUD endpoints with SKU uniqueness (409 `SKU_ALREADY_USED`), open to any authenticated role. A partial update might reset defaults such as `stockQuantity` (medium confidence, unverified). |
| 18 | [Orders Module](./component-deep-analyzer/component-analysis-Orders%20Module-2026-10-29%2014:55:48.md) | Five endpoints and 19 documented business rules, covering creation, listing, status workflow and deletion. Stock debit is check-then-decrement without row locks, and no endpoint has role or ownership checks. |
| 19 | [Order Status State Machine](./component-deep-analyzer/component-analysis-Order%20Status%20State%20Machine-2026-10-29%2014:55:48.md) | A table-driven transition map with PENDING to PAID stock debit and replenishment on cancellation. `allowedTransitions` and `isTerminal` are unused, and only some legal transitions are tested. |
| 20 | [Order Number Sequence](./component-deep-analyzer/component-analysis-Order%20Number%20Sequence-2026-10-29%2014:55:48.md) | A single-row counter (`ORD-NNNNNN`) reserved with an upsert inside the order transaction. It serializes order creation, leaves gaps on deletion, and its generation code is duplicated in the seed script. |
| 21 | [Seed Script](./component-deep-analyzer/component-analysis-Seed%20Script-2026-10-29%2014:55:48.md) | Wipes all 7 tables unconditionally, then seeds users, customers, products and orders. It hardcodes `admin123` and `operator123`, duplicates logic from `src/`, and has no tests. |
| 22 | [Test Suite](./component-deep-analyzer/component-analysis-Test%20Suite-2026-10-29%2014:55:48.md) | 15 integration tests (7 auth, 8 orders) with Vitest and Supertest against a real MySQL database. `tests/setup.ts` deletes all rows in the database named by `DATABASE_URL`. |
| 23 | [MySQL Container](./component-deep-analyzer/component-analysis-MySQL%20Container-2026-10-29%2014:55:48.md) | One `mysql:8.0` container with a named volume and a healthcheck, no init scripts, and port 3306 published on all interfaces. Credentials have hardcoded fallbacks and the image tag floats. |

## Critical Findings

### Security Risks

- **Privilege escalation (critical):** the unauthenticated `POST /api/v1/auth/register` accepts a client-supplied `role`, including `ADMIN`. Reported by the architectural, Auth Module, Users Module, Auth Middleware and Validation Middleware analyses.
- **Coarse authorization (high):** `requireRole` is applied to one route only, so any authenticated user can create, update and delete customers, products and orders.
- **No brute-force protection (high):** login and register have no rate limiting or lockout. Unknown-email login skips bcrypt, which makes timing reveal which emails exist. Registration returns a distinct 409 for existing emails.
- **JWT handling (high/medium):** claims are trusted for up to 8h with no revocation or refresh, `jwt.verify` does not pin the algorithm, and `.env.example` ships a placeholder secret that passes the 16-character minimum.
- **Missing HTTP hardening (medium):** no helmet, CORS policy or security headers. The client-supplied `x-request-id` is trusted and echoed into response headers and logs.
- **Exposure and credentials in config (medium):** MySQL port 3306 is published on all host interfaces, default database passwords appear in compose fallbacks and `.env.example`, the healthcheck passes the root password on the command line, and the seed script hardcodes account passwords.
- **Sensitive data (medium):** customer `document` and address are stored in plaintext columns.
- **Vulnerable dependencies:** the advisories for vitest, express, zod and bcrypt's transitive `tar` listed under Dependencies Health.
- **Destructive test and seed setup:** `tests/setup.ts` and `prisma/seed.ts` delete all rows in the database named by `DATABASE_URL`, with no guard against a non-test database.

### Technical Debt

- **Orders layering:** `OrderService` bypasses its repository and the customer and product repositories, concentrating business rules in a 255-line transaction script.
- **Error mapping gaps:** the error middleware maps only Prisma P2002 and P2025. Foreign-key violations (P2003) from deleting customers or products that have orders, and body-parser errors, are likely to return 500 (inferred, not tested).
- **Duplication:** `registerSchema` and `createUserSchema`, the inline `idParamSchema` and `userIdParamSchema`, pagination `skip` logic, Zod issue formatting, and order-number, bcrypt-cost and status-path logic in the seed script.
- **Unused code:** `pino-http`, `BadRequestError`, `allowedTransitions`, `isTerminal`, `ProductRepository.findManyByIds` and the exported `buildControllers`.
- **Test coverage:** only auth and orders are tested. There are no tests for customers, products, users, `/health`, the 404 fallback, `requireRole`, or most error-middleware branches. No coverage tooling exists.
- **Lifecycle and observability gaps:** the graceful shutdown does not await `server.close()`, `/health` is liveness only, and Prisma logs bypass the structured logger.
- **Documentation gap:** the `ORD-` order-number format and the sequence table are not described in `docs/`.

### Single Points of Failure

- **MySQL:** a single container and instance with no replication. Every endpoint except `/health` depends on it.
- **Order number counter:** every order creation locks the same row (`id = 1`) until commit, which serializes creation under concurrency.
- **Stock consistency:** the check-then-decrement without row locks, together with no version check on status changes, could oversell or double-apply concurrent transitions (inferred from code, no concurrency test).
- **Library concentration:** express (17 import sites), `@prisma/client` (15) and zod (9). A major version change in any of them touches most modules.
- **Configuration hubs:** `src/config/env.ts` exits the process at import time and is imported by the logger, database client and auth code, including in tests.
- **Process model:** the API is stateless apart from the Prisma connection pool, so the database and the order number counter are the shared contention points.

## Reports Index

See [MANIFEST.md](./MANIFEST.md) for complete list of all generated reports.
