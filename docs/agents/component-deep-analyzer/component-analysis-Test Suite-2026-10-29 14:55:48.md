# Component Deep Analysis Report: Test Suite (tests/)

Analysis date: 2026-10-29 14:55:48
Scope: `tests/`, `vitest.config.ts`, test-related entries in `package.json`. Ignored folders: none.

## 1. Executive Summary

The Test Suite is the automated verification layer of the OMS (Order Management System) REST API (Express + Prisma + MySQL). It is entirely black-box HTTP integration testing: Vitest 2.1.4 drives the real Express app (built through `buildApp`) using Supertest 7.0.0, against a real MySQL database reached through the real Prisma client. There are no unit tests, no mocks, and no test doubles.

Composition:
- 2 test files: `tests/auth.test.ts` (7 tests), `tests/orders.test.ts` (8 tests), 15 tests in total.
- 1 global setup file: `tests/setup.ts` (DB connect, per-test truncation, disconnect).
- 1 helper module: `tests/helpers/factories.ts` (app cache, user/customer/product factories, login helper).
- 1 runner config: `vitest.config.ts` (serial execution in a single forked process, 30s timeouts).

Key findings:
- Coverage is concentrated on the Auth and Orders modules. No tests exist for Customers, Products, Users modules, the `/health` endpoint, 404 handling, role-based authorization (ADMIN vs OPERATOR), or middleware in isolation.
- Orders tests cover the central domain rules: server-side total computation, PENDING history entry, order number format, status-transition validation, stock debit on PENDING -> PAID, stock replenishment on PAID -> CANCELLED, insufficient-stock rejection, listing filter, delete restriction.
- Several implemented rules have no tests (inactive product, discount exceeding subtotal, duplicate item aggregation, same-status transition, PROCESSING -> CANCELLED replenishment, expired/invalid JWT, pagination bounds).
- Tests depend on a live database (`DATABASE_URL`) and `JWT_SECRET` environment variables; the suite has no built-in provisioning of either (no env override in `vitest.config.ts`, no test-specific DB configuration). Running tests deletes all rows in the configured database (`tests/setup.ts:8-16`).
- No coverage tooling is configured (no `@vitest/coverage-*`, no `coverage` block), so quantitative coverage is not measurable from the repository; coverage figures below are qualitative estimates based on reading the tests against source.

## 2. Data Flow Analysis

Test execution flow:

```
1. `npm test` runs `vitest run` (package.json:17)
2. vitest.config.ts selects tests/**/*.test.ts, single fork, no file parallelism, node environment
3. tests/setup.ts is loaded before every test file (setupFiles)
4. beforeAll (setup.ts:4): prisma.$connect() on the real Prisma client from src/config/database.ts
   (importing it also triggers src/config/env.ts validation of process.env; invalid env calls process.exit(1))
5. beforeEach (setup.ts:8): delete all rows in FK-safe order:
   orderStatusHistory -> orderItem -> order -> orderNumberSequence -> product -> customer -> user
6. Test body obtains the Express app via getTestApp() (factories.ts:10), lazily built once with buildApp({ prisma })
7. Arrange: factories insert users (bcrypt hash cost 10), customers, products directly via Prisma
8. Authenticate: loginAndGetToken() calls POST /api/v1/auth/login via Supertest and extracts tokens.accessToken
9. Act: Supertest sends HTTP requests (in-process, no network port) with Authorization: Bearer <token>
10. Assert: status codes, response body shape, error.code values; for stock rules, the test re-reads the product via prisma.product.findUnique
11. afterAll (setup.ts:18): prisma.$disconnect()
```

Request path exercised inside the app under test (for context): Supertest -> Express JSON parser -> request logger -> `/api/v1` router -> auth middleware -> validate middleware (zod) -> controller -> service -> repository / Prisma transaction -> MySQL -> error middleware -> JSON response.

## 3. Business Rules & Logic

The Test Suite itself contains no production business logic; its "business rules" are (a) the behavioral rules it verifies in the application and (b) the testing rules/conventions embedded in the harness. Both are documented. Confidence is High for all rows since each is directly asserted in test code or configuration.

## Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Verified: Registration | Registration returns 201 with public user data only (no password or hash) | tests/auth.test.ts:6-17 |
| Verified: Registration | Duplicate email rejected with 409 `EMAIL_ALREADY_USED` | tests/auth.test.ts:19-28 |
| Verified: Validation | Invalid register payload rejected with 400 `VALIDATION_ERROR` and details array | tests/auth.test.ts:30-39 |
| Verified: Authentication | Valid login returns 200, Bearer access token and user | tests/auth.test.ts:41-52 |
| Verified: Authentication | Wrong password returns 401 `UNAUTHORIZED` | tests/auth.test.ts:54-63 |
| Verified: Authorization | `GET /auth/me` with valid token returns the authenticated user | tests/auth.test.ts:65-77 |
| Verified: Authorization | `GET /auth/me` without token returns 401 | tests/auth.test.ts:79-83 |
| Verified: Order creation | New order is PENDING, totals computed server-side, discount subtracted, one history row, `ORD-NNNNNN` number | tests/orders.test.ts:12-40 |
| Verified: Order creation | Non-existent product returns 404 `NOT_FOUND` | tests/orders.test.ts:42-57 |
| Verified: State machine / Stock | PENDING -> PAID allowed, decrements stock, appends history | tests/orders.test.ts:59-87 |
| Verified: State machine | PENDING -> SHIPPED returns 409 `INVALID_STATUS_TRANSITION` | tests/orders.test.ts:89-107 |
| Verified: Stock | PENDING -> PAID with insufficient stock returns 422 `INSUFFICIENT_STOCK`, stock unchanged | tests/orders.test.ts:109-132 |
| Verified: Stock | PAID -> CANCELLED restores stock | tests/orders.test.ts:134-161 |
| Verified: Listing | `GET /orders?status=` filters and reports pagination total | tests/orders.test.ts:163-195 |
| Verified: Deletion | Orders not PENDING/CANCELLED cannot be deleted (409 `INVALID_ORDER_STATE_FOR_DELETE`) | tests/orders.test.ts:197-218 |
| Harness | Database wiped before every test in FK-safe order | tests/setup.ts:8-16 |
| Harness | Serial execution, single fork, 30s test and hook timeouts | vitest.config.ts:8-17 |
| Harness | Single lazily-cached Express app instance shared across tests | tests/helpers/factories.ts:8-15 |
| Harness | Default fixtures: role OPERATOR, password `password123`, product price 10000 cents, stock 10, active true | tests/helpers/factories.ts:17-74 |

## Detailed breakdown of the business rules:

---

### Business Rule: Registration returns public data only

**Overview**:
Registering a user yields HTTP 201 and a body that includes email, name and role but never `password` or `passwordHash`.

**Detailed description**:
The test (`tests/auth.test.ts:6-17`) posts a valid payload (email, 12-character password, name, role OPERATOR) to `/api/v1/auth/register` and asserts `status 201`, `toMatchObject({ email, name, role })`, and the absence of `passwordHash` and `password` properties. This verifies that the application's `UserService.toPublic` projection (`src/modules/users/user.service.ts:42-51`) is applied at the boundary and that credentials are not leaked.

The registration path goes through `AuthService.register` -> `UserService.createUser`, which hashes the password with bcrypt (10 rounds) before persisting. The test does not verify the hash or that the stored password differs from plaintext; it only checks response redaction. It also does not verify that the persisted user can later log in, nor that `id`, `createdAt` or `updatedAt` are present.

Impact: this is the only test guarding against sensitive-field leakage in the registration response. Leakage through other endpoints (e.g. `/auth/me`, users module) is not tested for `passwordHash`.

**Rule workflow**:
```
POST /api/v1/auth/register {email, password, name, role}
 -> zod registerSchema -> UserService.createUser -> bcrypt.hash -> users.create
 -> toPublic -> 201 {id, email, name, role, createdAt, updatedAt}
Assert: 201, fields match, no password/passwordHash
```

---

### Business Rule: Email uniqueness on registration

**Overview**:
A second registration using an existing email must fail with 409 and code `EMAIL_ALREADY_USED`.

**Detailed description**:
The test (`tests/auth.test.ts:19-28`) creates a user directly through the factory with email `dup@test.local`, then calls register with the same email. The application checks `users.findByEmail` first and throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (`src/modules/users/user.service.ts:22-25`).

The test asserts the HTTP status and the machine-readable error code located at `res.body.error.code`, confirming the error envelope shape `{ error: { code, ... } }`. Case-insensitive or whitespace-variant email duplicates are not tested, and the race condition between concurrent registrations (check-then-insert) is not exercised.

Note the payload omits `role`, so the test also implicitly covers the default role path in the schema without asserting it.

**Rule workflow**:
```
Factory creates user dup@test.local
POST /auth/register with same email -> findByEmail hit -> ConflictError
Assert: 409, error.code == EMAIL_ALREADY_USED
```

---

### Business Rule: Registration payload validation

**Overview**:
Invalid registration input produces 400 `VALIDATION_ERROR` with an array of details.

**Detailed description**:
The test (`tests/auth.test.ts:30-39`) sends an invalid email (`not-an-email`), a too-short password (`short`, 5 characters versus the schema minimum of 8) and an empty name (schema minimum 2). The validate middleware converts the zod failure into a `ValidationError` with details. The assertions check status 400, code `VALIDATION_ERROR`, and that `error.details` is an array.

The test does not assert which fields failed or the content of each detail entry, so it would still pass if only one of the three fields were reported. Upper bounds (email 255, password 72, name 150), invalid roles, and malformed JSON are not covered. Login payload validation (`loginSchema`) is also untested.

**Rule workflow**:
```
POST /auth/register {bad email, short password, empty name}
 -> validate middleware (zod) fails -> ValidationError(400)
Assert: 400, VALIDATION_ERROR, details is array
```

---

### Business Rule: Login issues a Bearer access token

**Overview**:
Valid credentials return 200 with `tokens.accessToken` (string), `tokens.tokenType == 'Bearer'`, and the user's email.

**Detailed description**:
The test (`tests/auth.test.ts:41-52`) creates a user via factory (bcrypt-hashed password) and logs in over HTTP. `AuthService.login` (`src/modules/auth/auth.service.ts:31-45`) finds the user, compares via bcrypt, signs a JWT with claims `sub`, `email`, `role` and `JWT_EXPIRES_IN`, and returns the public user plus tokens.

The test verifies token presence and type but does not decode the token, check its claims, verify `expiresIn`, or verify that the token is accepted by protected routes in this particular test (that is covered indirectly by `/auth/me` and every order test, which use `loginAndGetToken`). Unknown-email login is not tested, although it takes the same 401 branch.

**Rule workflow**:
```
Factory user -> POST /auth/login -> findByEmail -> bcrypt.compare ok -> jwt.sign
Assert: 200, accessToken string, tokenType Bearer, user.email matches
```

---

### Business Rule: Wrong password is rejected without detail

**Overview**:
Incorrect password returns 401 with code `UNAUTHORIZED`.

**Detailed description**:
`tests/auth.test.ts:54-63` creates a valid user and logs in with `wrongpassword`. The service throws `UnauthorizedError('Invalid credentials')`, mapped to 401 `UNAUTHORIZED`. The same generic message is used for a missing user in the source, which prevents user enumeration, but only the wrong-password branch is tested; the unknown-user branch (and thus the non-enumeration property) is not verified.

No tests check rate limiting or lockout, as the application does not implement them.

**Rule workflow**:
```
POST /auth/login wrong password -> bcrypt.compare false -> UnauthorizedError
Assert: 401, error.code == UNAUTHORIZED
```

---

### Business Rule: Authenticated identity endpoint

**Overview**:
`GET /auth/me` returns the caller's identity when a valid Bearer token is presented and 401 when absent.

**Detailed description**:
Two tests cover this. `tests/auth.test.ts:65-77` logs in, then calls `/auth/me` with `Authorization: Bearer <token>` and asserts that `id` and `email` match the created user. `tests/auth.test.ts:79-83` calls the endpoint without a header and asserts 401.

Together they verify the auth middleware's two main branches (valid token, missing token). Malformed header, wrong scheme, tampered signature, expired token, token for a deleted user, and role-restricted routes are not tested. Since the database is wiped between tests, a "token for deleted user" scenario would be feasible but is absent.

**Rule workflow**:
```
login -> token -> GET /auth/me (Bearer) -> auth middleware verifies JWT -> UserService.getById
Assert: 200, id/email match
GET /auth/me (no header) -> 401
```

---

### Business Rule: Order creation, server-side totals, initial state and numbering

**Overview**:
Creating an order produces a PENDING order whose subtotal, total and order number are calculated by the server, with one initial history entry.

**Detailed description**:
`tests/orders.test.ts:12-40` creates a customer and two products (5000 cents x3, 1500 cents x2) and posts an order with a 1000 cent discount. The assertions confirm: status 201; `status == PENDING`; `subtotalCents == 18000`; `discountCents == 1000`; `totalCents == 17000`; 2 items; 1 history entry whose `toStatus` is PENDING; and `orderNumber` matching `/^ORD-\d{6}$/`.

In the source (`src/modules/orders/order.service.ts:50-124`) unit prices are read from the product table, not from the client, so the client cannot set prices; totals are computed inside a Prisma transaction; the order number comes from the single-row `OrderNumberSequence` upsert (`order.service.ts:245-254`) formatted with zero padding; and a history row with `fromStatus null` and reason `order created` is written. The test asserts format only, not sequential uniqueness or concurrency behavior, and `tests/setup.ts:12` resets the sequence between tests, which makes numbering deterministic but untested for continuity.

Unverified related rules: items for the same product are aggregated (`aggregateItems`, line 194-202); discount cannot exceed subtotal (lines 87-91); inactive products are rejected with 422 `INACTIVE_PRODUCT` (66-73); customer must exist (60); `notes` persistence.

**Rule workflow**:
```
POST /orders {customerId, items[], discountCents}
 -> zod createOrderSchema -> transaction: customer lookup, product lookup, price snapshot,
    subtotal/total, reserve ORD number, insert order+items+history
Assert: 201, PENDING, computed totals, item/history counts, number format
```

---

### Business Rule: Order creation requires existing products

**Overview**:
An order referencing a non-existent product is rejected with 404 `NOT_FOUND`.

**Detailed description**:
`tests/orders.test.ts:42-57` posts an item with a syntactically valid but unknown UUID. The service compares the number of found products against the number of requested distinct IDs (`order.service.ts:62-65`) and throws `NotFoundError('Product')`.

The test confirms status and code, not message or which product is missing. Because creation happens in a transaction, the absence of partial order rows is not asserted. The same applies to a missing customer (not tested) and malformed UUIDs (400 path, not tested).

**Rule workflow**:
```
POST /orders with unknown productId -> transaction -> product count mismatch -> NotFoundError
Assert: 404, NOT_FOUND
```

---

### Business Rule: PENDING -> PAID debits stock and records history

**Overview**:
Paying an order decrements product stock by ordered quantities and appends a PAID history entry.

**Detailed description**:
`tests/orders.test.ts:59-87` creates an order for 3 units of a product with stock 5, then PATCHes `/orders/:id/status` with `toStatus: PAID` and a reason. It asserts 200, `status == PAID`, history statuses equal `['PENDING','PAID']` in order, and, via a direct Prisma read, `stockQuantity == 2`.

The source rule (`order.status.ts:24-31`; `order.service.ts:151-153, 204-231`) debits stock only on the exact PENDING -> PAID transition, after checking availability for all items, within the same transaction as the status update and history insert. The test verifies the observable outcome (status, ordered history, persisted stock), which is the correct level to catch regressions. It does not verify the `reason` is stored, the `changedById` value, or that stock is not debited at creation time (the latter is only implied).

**Rule workflow**:
```
create order (stock 5, qty 3) -> PATCH status PAID
 -> tx: load order+items, canTransition, debitStock (check all then decrement), update status, insert history
Assert: 200, PAID, history [PENDING, PAID], DB stock == 2
```

---

### Business Rule: Invalid status transitions are rejected

**Overview**:
Transitions not allowed by the state machine return 409 `INVALID_STATUS_TRANSITION`.

**Detailed description**:
`tests/orders.test.ts:89-107` attempts PENDING -> SHIPPED. The state machine (`order.status.ts:3-10`) permits PENDING -> {PAID, CANCELLED}, PAID -> {PROCESSING, CANCELLED}, PROCESSING -> {SHIPPED, CANCELLED}, SHIPPED -> {DELIVERED}, and DELIVERED/CANCELLED are terminal. The service raises `InvalidStatusTransitionError` (a `ConflictError` subclass) with details `{from, to}`.

Only one illegal transition of the roughly 25 disallowed pairs is tested. The same-status case (`from === to`, `order.service.ts:140-146`), terminal-state exits, and the legal transitions PAID -> PROCESSING, PROCESSING -> SHIPPED, SHIPPED -> DELIVERED are not tested. The `error.details` payload is not asserted.

**Rule workflow**:
```
create order (PENDING) -> PATCH toStatus SHIPPED -> canTransition false -> InvalidStatusTransitionError
Assert: 409, INVALID_STATUS_TRANSITION
```

---

### Business Rule: Insufficient stock blocks payment

**Overview**:
Moving PENDING -> PAID when stock is lower than requested returns 422 `INSUFFICIENT_STOCK`, lists unavailable SKUs, and leaves stock untouched.

**Detailed description**:
`tests/orders.test.ts:109-132` creates an order for 5 units of a product with stock 1 (creation succeeds, demonstrating that stock is not validated at creation), then attempts PAID. The assertions: 422; code `INSUFFICIENT_STOCK`; `details.unavailable[0].sku` equals the product SKU; and the product stock remains 1.

The source (`order.service.ts:204-224`) collects all unavailable items before throwing `InsufficientStockError`, then relies on transaction rollback so that no partial decrements happen. The test validates the single-item case; multi-item partial availability (one item sufficient, another not) and the order remaining PENDING with no new history row are not asserted. Concurrent payment of two orders competing for the same stock is not tested (the source reads stock then decrements without row locks, which is a risk area but outside this component).

**Rule workflow**:
```
create order (qty 5, stock 1) -> 201
PATCH PAID -> debitStock detects shortage -> InsufficientStockError -> tx rollback
Assert: 422, INSUFFICIENT_STOCK, unavailable[0].sku, DB stock still 1
```

---

### Business Rule: Cancelling a paid order replenishes stock

**Overview**:
PAID -> CANCELLED returns the reserved quantity to stock.

**Detailed description**:
`tests/orders.test.ts:134-161` creates an order of 4 units (stock 10), pays it (stock 6, asserted), then cancels with a reason and asserts 200 and stock back to 10. In source, `shouldReplenishStock` (`order.status.ts:33-37`) returns true when moving to CANCELLED from PAID or PROCESSING; the service increments stock per item inside the transaction.

The test covers PAID -> CANCELLED only. PROCESSING -> CANCELLED replenishment, PENDING -> CANCELLED (no replenishment because nothing was debited), and the intermediate PAID response status code are not asserted (the pay call result is not checked; success is inferred from the stock check of 6).

**Rule workflow**:
```
create (qty 4, stock 10) -> PAID (stock 6) -> CANCELLED (reason)
 -> replenishStock increment -> stock 10
Assert: intermediate stock 6, cancel 200, final stock 10
```

---

### Business Rule: Order listing filter and pagination metadata

**Overview**:
`GET /orders?status=X` returns only orders in that status with accurate `pagination.total`.

**Detailed description**:
`tests/orders.test.ts:163-195` creates three PENDING orders (stock 100), pays the first, and queries with `status=PAID` (expects one result, the first order's id) and `status=PENDING` (expects 2 results and `pagination.total == 2`). This validates the repository filter and the `paginated` response envelope (`data`, `pagination`).

Not tested: default pagination values (page 1, pageSize 20), `pageSize` max 100, `page` and `pageSize` navigation, `customerId`, `from`, `to` filters, ordering of results, invalid status query (400), and other pagination fields (page, pageSize, totalPages).

**Rule workflow**:
```
create 3 orders -> PAY one -> GET ?status=PAID -> 1 item
GET ?status=PENDING -> 2 items, pagination.total 2
```

---

### Business Rule: Order deletion restricted by status

**Overview**:
Only PENDING or CANCELLED orders can be deleted; others yield 409 `INVALID_ORDER_STATE_FOR_DELETE`.

**Detailed description**:
`tests/orders.test.ts:197-218` pays an order and tries `DELETE /orders/:id`. The service (`order.service.ts:181-192`) allows deletion only for PENDING and CANCELLED. The test covers only the rejection branch for PAID.

The successful delete path (PENDING and CANCELLED, expected 204 or equivalent), the 404 for a missing order, behavior for other states (PROCESSING, SHIPPED, DELIVERED), and cascading deletion of items/history are not tested. The call to PATCH PAID in the arrange phase is not asserted.

**Rule workflow**:
```
create -> PAID -> DELETE -> status not in {PENDING, CANCELLED} -> ConflictError
Assert: 409, INVALID_ORDER_STATE_FOR_DELETE
```

---

### Business Rule (Harness): Isolation through full table wipe before each test

**Overview**:
Every test starts from an empty database.

**Detailed description**:
`tests/setup.ts:8-16` (a `beforeEach` registered through `setupFiles`, so it applies to every test file) deletes rows from `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, and `user` in child-to-parent order to satisfy foreign keys. Because the connection comes from `DATABASE_URL` through `src/config/env.ts`, the wipe targets whatever database that variable points to, with no guard against pointing at a non-test database (including the development database defined in `.env.example` as `oms`).

This design makes tests independent and deterministic (order numbers restart at ORD-000001 each test) at the price of requiring serial execution; hence `fileParallelism: false` and `singleFork: true`. Factories rely on this wipe and use random suffixes for emails/SKUs but a fixed document `000.000.000-00` and fixed emails (`dup@test.local`) that would collide without it.

**Rule workflow**:
```
beforeAll connect -> [for each test: deleteMany x7 -> test body] -> afterAll disconnect
```

---

### Business Rule (Harness): Serial, single-process execution with extended timeouts

**Overview**:
Tests run one at a time in one forked process with 30s limits.

**Detailed description**:
`vitest.config.ts:4-18` sets `environment: 'node'`, `globals: false` (tests import `describe/it/expect` explicitly), `include: ['tests/**/*.test.ts']`, `testTimeout` and `hookTimeout` of 30000 ms, `fileParallelism: false`, `pool: 'forks'`, `singleFork: true`. This guarantees that the shared database is never touched by concurrent workers, which is required by the wipe strategy.

The generous timeouts accommodate real DB I/O and bcrypt hashing (cost 10) performed in factories and in login for each test. No test is tagged or split into fast and slow groups; there is no separate unit-test project configuration.

**Rule workflow**:
```
vitest run -> load config -> spawn single fork -> run files sequentially -> tests sequentially
```

---

### Business Rule (Harness): Factories and shared application instance

**Overview**:
Helpers create valid baseline data and share a cached Express app.

**Detailed description**:
`tests/helpers/factories.ts` exposes `getTestApp` (builds `buildApp({ prisma })` once, line 10-15), `createTestUser` (bcrypt cost 10, default role OPERATOR, default password `password123`, unique email by timestamp and random suffix, lines 17-31), `loginAndGetToken` (real login via HTTP, throws on non-200, lines 33-40), `createTestCustomer` (fixed Brazilian address and phone, lines 42-60), `createTestProduct` (default price 10000 cents, stock 10, active, lines 62-74), and `bootstrapAuthenticatedUser` (user plus token, lines 76-83).

Data is inserted directly via Prisma (bypassing the API) so arrange steps are fast and independent of module behavior, while authentication always goes through the real login endpoint, so any login regression fails all order tests as well (coupling between auth and orders suites). The helper `bootstrapAuthenticatedUser('ADMIN')` supports role selection, but no test uses ADMIN, so role-based access control is never exercised.

**Rule workflow**:
```
getTestApp (cache) ; createTestUser -> loginAndGetToken -> token ; createTestCustomer/Product -> rows
```

---

## 4. Component Structure

```
tests/
├── setup.ts                 # Global Vitest setup: Prisma connect, per-test DB wipe, disconnect
├── helpers/
│   └── factories.ts         # App builder cache, user/customer/product factories, login helper
├── auth.test.ts             # 7 integration tests: register, login, /auth/me
└── orders.test.ts           # 8 integration tests: create, status transitions, stock, list, delete

Related configuration outside the folder:
vitest.config.ts             # Runner config (include glob, serial fork, timeouts, setupFiles)
package.json                 # Scripts: "test": "vitest run" (line 17), "test:watch": "vitest" (line 18);
                             # devDependencies: vitest 2.1.4, supertest 7.0.0, @types/supertest 6.0.2
```

No fixtures files, snapshots, mocks, `__mocks__` folders, or additional test projects were found anywhere in the repository (search for `*test*`, `*spec*` paths outside `node_modules` returned only the files above).

## 5. Dependency Analysis

```
Internal Dependencies:
tests/setup.ts            -> src/config/database.ts (prisma) -> src/config/env.ts
tests/helpers/factories.ts -> src/config/database.ts, src/app.ts (buildApp) -> all modules, middlewares, routes
tests/auth.test.ts        -> tests/helpers/factories.ts
tests/orders.test.ts      -> tests/helpers/factories.ts, src/config/database.ts
vitest.config.ts          -> tests/setup.ts (setupFiles)

External Dependencies:
- vitest 2.1.4 - test runner, assertions (`describe`, `it`, `expect`, hooks)
- supertest 7.0.0 (+ @types/supertest 6.0.2) - in-process HTTP assertions
- @prisma/client - direct DB access for fixtures and verification
- bcrypt - password hashing in factories
- express (types) - Express type for the cached app
- MySQL 8.0 (docker-compose.yml, service oms-mysql) - persistence used by tests
Runtime environment requirements: DATABASE_URL, JWT_SECRET (min 16 chars); optional JWT_EXPIRES_IN, NODE_ENV, LOG_LEVEL
```

Note: tests import source files with `.js` extensions (ESM/NodeNext style) while the files are `.ts`; this is resolved by Vitest.

## 6. Afferent and Efferent Coupling

Unit of analysis: TypeScript modules/exported functions within the suite. Counts are of distinct in-suite/internal references found in imports.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| tests/helpers/factories.ts (module) | 2 (auth.test, orders.test) | 4 (src/config/database, src/app, bcrypt/prisma types, supertest) | High |
| getTestApp | 3 (loginAndGetToken, auth.test, orders.test) | 1 (buildApp) | High |
| loginAndGetToken | 2 (auth.test, bootstrapAuthenticatedUser) | 2 (getTestApp, supertest) | Medium |
| bootstrapAuthenticatedUser | 1 (orders.test) | 2 (createTestUser, loginAndGetToken) | Medium |
| createTestUser | 2 (auth.test, bootstrapAuthenticatedUser) | 2 (prisma, bcrypt) | Medium |
| createTestCustomer | 1 (orders.test) | 1 (prisma) | Low |
| createTestProduct | 1 (orders.test) | 1 (prisma) | Low |
| tests/setup.ts | 1 (vitest.config.ts, loaded globally) | 2 (vitest, src/config/database) | High |
| tests/auth.test.ts | 0 | 2 (factories, supertest) | Low |
| tests/orders.test.ts | 0 | 3 (factories, src/config/database, supertest) | Low |
| src/config/database.ts (prisma singleton, as used by tests) | 3 (setup, factories, orders.test) | 1 (env) | High |

## 7. Endpoints

The Test Suite does not expose endpoints. Endpoints it exercises (for reference) are:

| Endpoint | Method | Tested in |
|----------|--------|-----------|
| /api/v1/auth/register | POST | auth.test.ts:9, 22, 32 |
| /api/v1/auth/login | POST | auth.test.ts:44, 57; factories.ts:35 |
| /api/v1/auth/me | GET | auth.test.ts:70, 81 |
| /api/v1/orders | POST | orders.test.ts (multiple) |
| /api/v1/orders | GET (status filter) | orders.test.ts:183, 190 |
| /api/v1/orders/{id}/status | PATCH | orders.test.ts (multiple) |
| /api/v1/orders/{id} | DELETE | orders.test.ts:212 |

Not exercised: `/health`, `GET /api/v1/orders/{id}`, all `/customers`, `/products`, `/users` routes, and the 404 fallback.

(The section above is included only as a coverage reference; the suite itself has no endpoints.)

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 via Prisma | Database (real) | Fixtures, cleanup, state verification | MySQL via Prisma Client | Prisma models | Connect in beforeAll; failures surface as test/hook failures (30s timeout) |
| Express app (in-process) | Internal system under test | HTTP behavior verification | HTTP via Supertest (no port) | JSON | Assertions on status and `error.code` |
| JWT (jsonwebtoken, in app) | Internal library | Obtain valid Bearer tokens | Authorization header | JWT | `loginAndGetToken` throws with status and body if login fails (factories.ts:36-38) |
| bcrypt | Library | Hash fixture passwords | In-process | Hash string | None |
| Environment (process.env) | Configuration | DB URL and JWT secret | Env variables | Strings | env.ts exits process on invalid config |

No external third-party services are called and none are mocked.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Integration (black-box API) testing | Supertest against `buildApp` | tests/*.test.ts | Verify end-to-end behavior including DB |
| Test Data Factory / Object Mother | `createTest*` functions with overrides | tests/helpers/factories.ts:17-83 | Concise, customizable fixtures |
| Lazy Singleton | `cachedApp` | tests/helpers/factories.ts:8-15 | Build the app once |
| Global Fixture Setup / Teardown | `beforeAll`/`beforeEach`/`afterAll` in setupFiles | tests/setup.ts | DB lifecycle and isolation |
| Dependency Injection use | `buildApp({ prisma })` | factories.ts:12 | Inject real Prisma client into the app |
| Arrange-Act-Assert | Within each test | tests/*.test.ts | Readability |
| State verification via side channel | Direct `prisma.product.findUnique` after HTTP calls | orders.test.ts:85, 130, 150, 159 | Assert persisted effects |
| Serial execution | `singleFork`, `fileParallelism: false` | vitest.config.ts:11-17 | Protect shared DB |

Architectural decision: tests choose real infrastructure over mocks, accepting environment dependency for higher realism. No separation between unit, integration and e2e levels exists.

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | tests/setup.ts:8-16 | Unconditional `deleteMany` on all core tables of whatever `DATABASE_URL` points to; no check for a dedicated test database | Possible destruction of development data if run against the dev DB |
| High | Coverage scope | No tests for Customers, Products, Users modules, `/health`, 404 fallback | Regressions in these modules undetected |
| High | Authorization | No test uses ADMIN role or verifies role-based restrictions; invalid/expired/tampered JWT untested | Access-control regressions undetected |
| Medium | Orders rules | Untested rules: inactive product (422), discount > subtotal, item aggregation, same-status transition, terminal states, PROCESSING -> CANCELLED replenishment, successful delete, missing customer | Core domain logic partially unprotected |
| Medium | Concurrency | No tests for concurrent stock debit or order number reservation | Race conditions invisible to the suite |
| Medium | Environment | No test-specific env or DB provisioning in config; suite requires pre-set `DATABASE_URL` and `JWT_SECRET`, and migrated schema | Suite fails or process exits when environment is missing; not self-contained |
| Medium | Coverage tooling | No coverage provider or thresholds configured; no CI configuration found | Coverage unmeasurable and unenforced |
| Medium | Assertion depth | Several tests omit asserting intermediate arrange calls (e.g., PAID call in delete and cancel tests) and error `details` content | Failures may surface in confusing places or pass vacuously |
| Low | Test coupling | All order tests depend on the real login endpoint via `loginAndGetToken` | An auth defect fails many unrelated tests |
| Low | Performance | bcrypt cost 10 for every created user and every login; serial execution only | Slower runs as suite grows |
| Low | Naming/Data | Fixed values (`dup@test.local`, document `000.000.000-00`) rely on the table wipe for uniqueness | Fragile if isolation strategy changes |
| Low | Documentation | README states tests are "not to be altered" for the challenge (README.md:68, 259); no test documentation of how to run against a DB | Context only; not a defect |

## 11. Test Coverage Analysis

Test file locations: `tests/auth.test.ts`, `tests/orders.test.ts` (plus support files `tests/setup.ts`, `tests/helpers/factories.ts`). No test files exist in other folders of the project. No coverage report or coverage configuration exists, so percentages are qualitative estimates from reading tests against source (not measured; code was not executed).

| Component (module under test) | Unit Tests | Integration Tests | Coverage | Test Quality |
|-------------------------------|------------|-------------------|----------|--------------|
| Auth module (register, login, me) | 0 | 7 | Approx. 70% of behavior (estimate) | Good status/code assertions; missing unknown-email login, token claims, invalid/expired token, loginSchema validation |
| Orders module (create, status, list, delete) | 0 | 8 | Approx. 60% of behavior (estimate) | Good persisted-state verification of stock; missing get-by-id, delete success, inactive product, discount rule, aggregation, other transitions, filters |
| Order status state machine (`order.status.ts`) | 0 | Indirect (3 transitions: PENDING->PAID, PENDING->SHIPPED rejected, PAID->CANCELLED) | Approx. 30% of transition matrix (estimate) | No direct unit tests of `canTransition`, `isTerminal`, `allowedTransitions` |
| Users module | 0 | Indirect only through register/me | Low | No direct tests |
| Customers module | 0 | 0 (used only as fixtures) | None | Untested |
| Products module | 0 | 0 (used only as fixtures) | None | Untested |
| Middlewares (auth, validate, error, request logger) | 0 | Indirect | Partial (auth: missing token and valid token; validate: one case; error: several codes) | No isolated tests; JWT failure modes missing |
| Config (env, database) | 0 | Indirect | Minimal | Env validation failure path untested |
| Health endpoint and 404 fallback | 0 | 0 | None | Untested |

Totals: 15 integration tests (7 + 8), 0 unit tests, 0 mocks. Test quality summary: readable Arrange-Act-Assert integration tests with meaningful business assertions and database verification; gaps are in breadth (modules, negative paths, authorization) and in isolation from environment.
