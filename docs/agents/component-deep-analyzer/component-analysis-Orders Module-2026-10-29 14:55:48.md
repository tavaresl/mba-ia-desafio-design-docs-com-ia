# Component Deep Analysis Report: Orders Module

Analysis date: 2026-10-29 14:55:48
Scope: `src/modules/orders` (OrderController, OrderService, OrderRepository, order.routes, order.schemas, order.status) plus the files that wire it and test it. No files were modified and no code or tests were executed. Coverage figures are not measured (no coverage tool is configured).

---

## 1. Executive Summary

The Orders Module is the core transactional domain of the "Order Management System REST API" (Node >= 20, TypeScript, Express 4.21.1, Prisma 5.22.0 on MySQL, Zod 3.23.8, JWT auth). It exposes five REST endpoints under `/api/v1/orders` (list, get, create, change status, delete). It owns:

- Order creation with server-side price snapshotting and total computation.
- A finite state machine for order status (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED).
- Stock debit/replenish side effects tied to specific status transitions.
- Sequential human-readable order numbers (`ORD-000001`).
- An append-only status audit trail (`order_status_history`).
- Conditional deletion.

Key findings:

- Layering is controller -> service -> repository, with manual constructor injection in `src/app.ts:42-44`. The service is not purely repository-based: `OrderService` receives both `OrderRepository` and a raw `PrismaClient`, and the two write paths (`create`, `changeStatus`) bypass the repository and use `tx` directly inside `prisma.$transaction`.
- Stock is debited only on PENDING -> PAID, not on creation. Stock is replenished on cancellation from PAID or PROCESSING only.
- Stock check and decrement in `debitStock` are a read-then-write sequence without row locks or conditional updates (`order.service.ts:204-231`), so concurrent PAID transitions on the same product can oversell (confidence: high for the code pattern; actual behavior depends on MySQL isolation, default REPEATABLE READ, which does not prevent this lost-update pattern for `decrement` with a stale check).
- No role-based authorization is applied to any order endpoint; `requireRole` exists (`src/middlewares/auth.middleware.ts:49`) but is not used in `order.routes.ts`. Any authenticated user (ADMIN or OPERATOR) can delete orders and change status.
- Test coverage is a single integration file (`tests/orders.test.ts`) with 8 tests against a real database; there are no unit tests for `order.status.ts` or the service.

---

## 2. Data Flow Analysis

Create order (`POST /api/v1/orders`):

```
1. Request enters Express router mounted at /api/v1/orders (src/routes/index.ts:28)
2. authenticate middleware verifies Bearer JWT, sets req.user (order.routes.ts:14; auth.middleware.ts:27-47)
3. validate({ body: createOrderSchema }) parses and replaces req.body, applies defaults (order.routes.ts:18; order.schemas.ts:11-16)
4. OrderController.create checks req.user, calls OrderService.create(body, user.id) (order.controller.ts:28-36)
5. OrderService.create: guard for empty items, aggregateItems merges duplicate productIds (order.service.ts:50-56, 194-202)
6. prisma.$transaction opens:
   a. tx.customer.findUnique -> NotFoundError('Customer')
   b. tx.product.findMany(ids) -> NotFoundError('Product') if count mismatch
   c. inactive products -> UnprocessableEntityError INACTIVE_PRODUCT
   d. unit price snapshot from product.priceCents; line total = unit * quantity
   e. subtotal = sum(lines); discount > subtotal -> ValidationError
   f. total = subtotal - discount
   g. reserveOrderNumber: upsert on order_number_sequence id=1 (order.service.ts:245-254)
   h. tx.order.create with nested items and an initial history row (PENDING, from=null, reason "order created"), including relations
7. Result returned; controller responds 201 with JSON
8. Any thrown error goes to next(err) and then errorMiddleware (error.middleware.ts:14-65) which formats { error: { code, message, details? } }
```

Change status (`PATCH /api/v1/orders/:id/status`):

```
1. authenticate, then validate({ params: orderIdParamSchema, body: updateOrderStatusSchema })
2. OrderController.changeStatus -> OrderService.changeStatus(id, body, user.id)
3. prisma.$transaction:
   a. tx.order.findUnique(include items) -> NotFoundError('Order')
   b. from === to -> ConflictError INVALID_STATUS_TRANSITION
   c. canTransition(from, to) false -> InvalidStatusTransitionError (409)
   d. shouldDebitStock(from,to) -> debitStock (availability check, then decrement per item)
   e. shouldReplenishStock(from,to) -> replenishStock (increment per item)
   f. tx.order.update status
   g. tx.orderStatusHistory.create (from, to, changedById, reason)
   h. re-read order with relations
4. 200 with refreshed order
```

List (`GET /api/v1/orders`): validate query -> `OrderService.list` computes `skip=(page-1)*pageSize` -> `OrderRepository.list` runs `findMany` (createdAt desc) and `count` in one `$transaction([...])` -> `paginated()` wraps `{ data, pagination }` (`src/shared/http/response.ts:22`).

Get by id: validate params -> `OrderRepository.findByIdWithRelations` (items with product summary, history ascending, customer summary) -> 404 if null.

Delete: validate params -> `OrderRepository.findById` -> status check -> `deleteById` (cascade removes items and history via FK `onDelete: Cascade`) -> 204.

---

## 3. Business Rules & Logic

### Overview of the business rules

| # | Rule Type | Rule Description | Location |
|---|-----------|------------------|----------|
| 1 | Authentication | All order endpoints require a valid Bearer JWT | order.routes.ts:14; auth.middleware.ts:27 |
| 2 | Validation | Order must have at least one item; quantity positive integer; UUIDs for ids | order.schemas.ts:6-16; order.service.ts:51 |
| 3 | Validation | discountCents non-negative integer, default 0; notes max 1000 chars | order.schemas.ts:14-15 |
| 4 | Business Logic | Duplicate product lines in one order are merged by summing quantity | order.service.ts:194-202 |
| 5 | Business Logic | Customer and all products must exist | order.service.ts:59-65 |
| 6 | Business Logic | Inactive products cannot be ordered | order.service.ts:66-73 |
| 7 | Business Logic | Unit price is snapshotted from product; totals computed on the server | order.service.ts:75-92 |
| 8 | Validation | Discount cannot exceed subtotal | order.service.ts:87-91 |
| 9 | Business Logic | New orders start in PENDING with an initial history entry | order.service.ts:95-114 |
| 10 | Business Logic | Sequential order number `ORD-` + 6 zero-padded digits | order.service.ts:245-254 |
| 11 | State Machine | Allowed status transitions matrix; DELIVERED and CANCELLED are terminal | order.status.ts:3-14 |
| 12 | Business Logic | Same-status transition rejected with 409 | order.service.ts:140-146 |
| 13 | Business Logic | Stock is debited only on PENDING -> PAID; fails with 422 if insufficient | order.status.ts:24-31; order.service.ts:151-153, 204-231 |
| 14 | Business Logic | Stock is replenished on cancellation from PAID or PROCESSING | order.status.ts:33-37; order.service.ts:154-156, 233-243 |
| 15 | Audit | Every status change writes a history record with actor and reason | order.service.ts:159-167 |
| 16 | Business Logic | Delete allowed only in PENDING or CANCELLED | order.service.ts:181-192 |
| 17 | Validation | List filters and pagination bounds (pageSize max 100, default 20) | order.schemas.ts:23-30 |
| 18 | Business Logic | List sorted by createdAt descending; date range inclusive | order.repository.ts:21-45 |
| 19 | Atomicity | Create and status change run in a single DB transaction | order.service.ts:58, 131 |

### Detailed breakdown of the business rules

---

### Business Rule: Authentication required on all order endpoints

**Overview**:
The router applies the `authenticate` middleware to every route, so no order operation is available anonymously.

**Detailed description**:
`buildOrderRouter` calls `router.use(authenticate)` before registering any route (`order.routes.ts:14`). The middleware requires an `Authorization` header starting with `bearer ` (case-insensitive), a non-empty token, and a token verifiable with `env.JWT_SECRET` (`auth.middleware.ts:27-47`). On success it populates `req.user` with `id` (from `sub`), `email` and `role`.

The controller adds a second, defensive guard: `create` and `changeStatus` throw `UnauthorizedError` if `req.user` is missing (`order.controller.ts:30, 40`). This is redundant given the middleware but protects against route misconfiguration. `list`, `getById` and `delete` have no such guard because they do not use the user identity.

The authenticated user's `id` is used as `createdById` on the order and `changedById` on history rows, giving attribution. The role is carried in the token but is never checked in this module: ADMIN and OPERATOR have identical permissions on orders. No ownership model exists either (any user can read or modify any order).

**Rule workflow**:
```
Request -> authenticate -> header check -> token present -> jwt.verify
  -> success: req.user set -> validate -> controller
  -> failure: 401 UNAUTHORIZED (Missing/invalid header, Missing token, Invalid or expired token)
```

---

### Business Rule: Input validation of order payloads

**Overview**:
Zod schemas validate and normalize body, params and query before the controller runs.

**Detailed description**:
`createOrderSchema` requires `customerId` as UUID, `items` as an array with at least one element (message "order must contain at least one item"), each item with `productId` UUID and `quantity` a positive integer, `discountCents` a non-negative integer defaulting to 0, and `notes` an optional string up to 1000 characters (`order.schemas.ts:6-16`). `updateOrderStatusSchema` requires `toStatus` to be a member of the Prisma `OrderStatus` enum and `reason` optional up to 500 characters (matching the `VarChar(500)` column). `orderIdParamSchema` requires a UUID `id`.

The `validate` middleware replaces `req.body` and `req.params` with parsed output and merges parsed query values into `req.query` (`validate.middleware.ts:14-23`). Failures become `ValidationError` (HTTP 400, code `VALIDATION_ERROR`) with a `details` array of `{ path, message }`.

The service repeats the empty-items check (`order.service.ts:51-53`), which is unreachable through HTTP because Zod rejects empty arrays first; it only matters if the service is called directly. There is no upper bound on the number of items or on quantity, and no bound on `discountCents` apart from the subtotal check done later.

**Rule workflow**:
```
raw input -> Zod parse -> ok: normalized input -> controller
                       -> ZodError: 400 VALIDATION_ERROR with details[]
```

---

### Business Rule: Duplicate items are aggregated

**Overview**:
If the same `productId` appears on several lines, quantities are summed into one order item.

**Detailed description**:
`aggregateItems` builds a `Map<productId, quantity>` and returns one entry per distinct product (`order.service.ts:194-202`). The resulting list drives the product lookup, the stock check later, and the item rows created. Consequently the stored order always has at most one `order_items` row per product, and the count comparison `products.length !== productIds.length` works on distinct ids.

This avoids false "product not found" results caused by duplicate ids in the `IN` query and guarantees stock checks are performed against total quantity per product. The response to the client contains merged lines, which may differ from what the client submitted (for example 2 lines becoming 1).

**Rule workflow**:
```
items[] -> Map accumulate by productId -> distinct list -> used for lookup and persistence
```

---

### Business Rule: Referenced customer and products must exist and be active

**Overview**:
Order creation fails if the customer or any product is missing, or if any product is inactive.

**Detailed description**:
Inside the transaction, the customer is loaded by id; absence yields `NotFoundError('Customer')` (404 `NOT_FOUND`, message "Customer not found"). Products are loaded with `findMany`; if the number returned differs from the number of distinct requested ids, `NotFoundError('Product')` is raised (`order.service.ts:59-65`). The error does not identify which product id is missing.

Products with `active === false` trigger `UnprocessableEntityError` (422, code `INACTIVE_PRODUCT`) with `details.skus` listing the inactive SKUs (`order.service.ts:66-73`). The check happens only at creation; if a product is deactivated afterwards, the PENDING -> PAID transition does not re-check `active` (only stock).

The customer check precedes product checks, so an invalid customer masks product errors. The only automated test covering this area verifies the non-existent product path (`tests/orders.test.ts:42-57`); customer-not-found and inactive-product paths are untested.

**Rule workflow**:
```
customer found? -no-> 404 Customer not found
products count == distinct ids? -no-> 404 Product not found
any inactive? -yes-> 422 INACTIVE_PRODUCT {skus}
else continue
```

---

### Business Rule: Server-side pricing, price snapshot and totals

**Overview**:
Prices are never accepted from the client; the unit price at order time is copied from the product and totals are computed server-side in integer cents.

**Detailed description**:
For each aggregated item, `unitPriceCents = product.priceCents` and `totalCents = unitPriceCents * quantity` (`order.service.ts:75-84`). These values are persisted on `order_items`, so later changes to the product price do not alter historical orders. `subtotalCents` is the sum of line totals; `totalCents = subtotalCents - discountCents`.

All amounts are integers (cents), avoiding floating point. The schema columns are `Int` (32-bit in MySQL), so very large orders could overflow; no guard exists (potential risk, unverified at runtime). The discount is an absolute amount; no percentage or coupon logic exists, and there are no taxes or shipping.

The integration test asserts subtotal, discount and total are computed correctly for a two-product order with a discount (`tests/orders.test.ts:12-40`).

**Rule workflow**:
```
product.priceCents -> unitPriceCents (snapshot) -> line total -> subtotal -> minus discount -> total
```

---

### Business Rule: Discount cannot exceed subtotal

**Overview**:
A discount larger than the computed subtotal is rejected.

**Detailed description**:
After computing the subtotal, if `input.discountCents > subtotalCents` a `ValidationError` (400, `VALIDATION_ERROR`) is thrown with details `[{ path: 'discountCents', message: 'Discount exceeds subtotal' }]` (`order.service.ts:87-91`). A discount equal to the subtotal is allowed and yields a total of 0.

This guarantees that `totalCents` is never negative. Because the check needs product prices, it can only occur in the service and not in the Zod schema. The error is raised inside the transaction, so no partial writes occur (the order number sequence is reserved only after this check, so no number is consumed). This path has no automated test.

**Rule workflow**:
```
subtotal computed -> discount > subtotal? -yes-> 400 -> no -> total = subtotal - discount
```

---

### Business Rule: Initial status and initial history entry

**Overview**:
Every order is created as PENDING and with one history row recording creation.

**Detailed description**:
`status` is hard-coded to `OrderStatus.PENDING` irrespective of input (`order.service.ts:99`); clients cannot choose the initial status. Nested creation adds a history record with `fromStatus: null`, `toStatus: PENDING`, `changedById: userId` and `reason: 'order created'` (`order.service.ts:106-113`). Items and history are created atomically with the order.

The history thus always has at least one entry, and the status timeline can be reconstructed completely from `order_status_history` (ordered by `changedAt` ascending when read). The test verifies exactly one history entry with `toStatus` PENDING (`tests/orders.test.ts:37-38`). Creation does not reserve or debit stock.

**Rule workflow**:
```
create -> status=PENDING -> history[0] = (null -> PENDING, "order created", user)
```

---

### Business Rule: Sequential order number generation

**Overview**:
Orders receive a unique number formatted `ORD-NNNNNN`, derived from a single-row sequence table.

**Detailed description**:
`reserveOrderNumber` upserts the row `order_number_sequence` with `id = 1`: on first use it creates `nextValue: 2`, otherwise increments `nextValue` by 1 and selects it. The assigned number is `nextValue - 1`, padded to 6 digits (`order.service.ts:245-254`). The first order is therefore ORD-000001.

Because the upsert runs inside the creation transaction, the sequence row is locked by the update until commit, serializing concurrent creations and avoiding duplicates; a rollback also rolls back the increment, so numbers are gapless under failures (confidence: high for InnoDB behavior; the first-insert race on a missing row could raise a unique conflict mapped to 409 by `errorMiddleware` P2002 handling, confidence: medium). `orderNumber` also has a unique index in the schema. After 999999 orders, the number grows beyond 6 digits while the column is `VarChar(20)`, which remains valid but breaks the fixed-width format.

The test asserts the format via regex only (`tests/orders.test.ts:39`); sequential increments are not asserted. `prisma/seed.ts:165` contains its own copy of this logic (duplicated outside the module).

**Rule workflow**:
```
upsert sequence(id=1) -> nextValue -> current = nextValue-1 -> "ORD-" + pad6(current)
```

---

### Business Rule: Order status state machine

**Overview**:
Status changes are allowed only along a fixed transition graph.

**Detailed description**:
The transitions are defined in `order.status.ts:3-10`:

| From | Allowed To |
|------|-----------|
| PENDING | PAID, CANCELLED |
| PAID | PROCESSING, CANCELLED |
| PROCESSING | SHIPPED, CANCELLED |
| SHIPPED | DELIVERED |
| DELIVERED | (none, terminal) |
| CANCELLED | (none, terminal) |

`canTransition` checks membership. Cancellation is not allowed once SHIPPED. Backward moves (for example PAID -> PENDING) are impossible. `allowedTransitions` and `isTerminal` are exported helpers but are not used by the service or controller (no endpoint exposes allowed transitions).

Violations throw `InvalidStatusTransitionError` (409, code `INVALID_STATUS_TRANSITION`, details `{ from, to }`) (`http-errors.ts:45-53`). The test covers PENDING -> SHIPPED (`tests/orders.test.ts:89-107`); other illegal edges and terminal states are not tested. The `toStatus` enum is validated by Zod, so unknown values yield 400 before the state machine is consulted.

**Rule workflow**:
```
load order -> from -> same as to? -> 409 (rule below)
           -> canTransition(from,to)? -no-> 409 INVALID_STATUS_TRANSITION
           -> yes: apply side effects, update, record history
```

---

### Business Rule: Same-status transition is rejected

**Overview**:
Requesting the status the order already has returns a conflict.

**Detailed description**:
Before consulting the matrix, the service compares `from === to` and throws `ConflictError` with message "Order is already in {to} status", code `INVALID_STATUS_TRANSITION` and details `{ from, to }` (`order.service.ts:140-146`). This makes status updates non-idempotent: repeating a successful request returns 409 rather than 200.

The same error code as the matrix violation is used, so clients cannot distinguish "no-op" from "illegal edge" except through the message. Since no self-transitions exist in the matrix, the explicit check only refines the message. It also protects against double stock debit on a repeated PAID request (a second PAID request would otherwise fail the matrix anyway).

**Rule workflow**:
```
from === to -> 409 INVALID_STATUS_TRANSITION ("already in X status")
```

---

### Business Rule: Stock is debited on PENDING -> PAID

**Overview**:
Inventory is reserved/consumed at payment confirmation, with an availability check that can fail with 422.

**Detailed description**:
`shouldDebitStock(from, to)` is true only for PENDING -> PAID (`order.status.ts:24-31`). `debitStock` loads the products of the order items, compares `stockQuantity` with each requested quantity, and collects all shortfalls `{ sku, requested, available }`. If any exist, it throws `InsufficientStockError` (422, `INSUFFICIENT_STOCK`, `details.unavailable[]`). A missing product record is treated as zero availability with the product id as `sku` (`order.service.ts:204-224`). Otherwise it decrements each product with `stockQuantity: { decrement: quantity }`.

Because everything runs in the status-change transaction, a failure leaves stock untouched and the order stays PENDING (verified by test: stock stays 1, `tests/orders.test.ts:130-131`). The success path is verified too (stock 5 -> 2, `tests/orders.test.ts:85-86`).

The check and decrement are separate statements without `SELECT ... FOR UPDATE` or a conditional update; two concurrent orders for the same product could both pass the check. There is no database CHECK constraint preventing negative stock in the migration/schema (`stockQuantity Int @default(0)`; to be confirmed against migration SQL). Product `active` is not re-checked at payment time.

**Rule workflow**:
```
PENDING->PAID -> load products -> any qty > stock? -yes-> 422 INSUFFICIENT_STOCK (rollback)
              -> no: decrement each product -> continue transition
```

---

### Business Rule: Stock is replenished when cancelling a paid or processing order

**Overview**:
Cancelling an order that already consumed stock returns the quantities to inventory.

**Detailed description**:
`shouldReplenishStock` is true when `to === CANCELLED` and `from` is PAID or PROCESSING (`order.status.ts:33-37`). `replenishStock` increments each item's product stock by the order item quantity (`order.service.ts:233-243`). Cancelling from PENDING does not touch stock because none was debited. SHIPPED orders cannot be cancelled, so there is no replenishment after shipping.

Replenishment uses the quantities in the order items, not product current state, and does not verify the product still exists (the FK `product` relation has no cascade, so product deletion would be blocked by the database while items reference it). The test covers PAID -> CANCELLED restoring stock 6 -> 10 (`tests/orders.test.ts:134-161`); PROCESSING -> CANCELLED is untested.

Debit and replenish rules are coupled by the state machine: the only way to reach PROCESSING is via PAID, so debit has always occurred for orders eligible for replenishment, assuming no data created outside the service (the seed script creates orders directly in PAID/PROCESSING states; stock consistency of seeded data is unverified).

**Rule workflow**:
```
to=CANCELLED and from in {PAID, PROCESSING} -> increment stock per item -> continue transition
```

---

### Business Rule: Status change audit trail

**Overview**:
Every transition appends a history record that includes who changed it, from/to status, and an optional reason.

**Detailed description**:
After updating the order's status, the service inserts `order_status_history` with `orderId`, `fromStatus`, `toStatus`, `changedById` (authenticated user) and `reason` (optional, max 500, stored as null when absent) (`order.service.ts:158-167`). `changedAt` defaults to `now()`. History is returned with orders ordered ascending by `changedAt`.

The history is written in the same transaction as the status update and stock effects, so the audit trail cannot diverge from state. Rows cascade on order delete (`onDelete: Cascade`), so deleting a PENDING or CANCELLED order destroys its audit trail. Ordering by `changedAt` alone could be ambiguous for rows with equal timestamps (second/millisecond precision, MySQL `DATETIME(3)` by Prisma default; to be confirmed in the migration).

**Rule workflow**:
```
transition accepted -> order.update(status) -> history.create(from,to,user,reason) -> commit
```

---

### Business Rule: Deletion only for PENDING or CANCELLED orders

**Overview**:
Orders in any other state cannot be deleted.

**Detailed description**:
`OrderService.delete` loads the order via the repository (404 if missing) and rejects the deletion with `ConflictError` (409, code `INVALID_ORDER_STATE_FOR_DELETE`, details `{ status }`) when the status is not PENDING or CANCELLED (`order.service.ts:181-192`). Otherwise `deleteById` performs a hard delete; items and history are removed by FK cascade. The response is 204 with no body.

The operation is not wrapped in a transaction: the check and delete are separate statements, so a status change between them could allow deleting an order that has just become PAID (stock debited and then order removed without replenishment). Deletion of a missing row between check and delete would raise Prisma P2025, which `errorMiddleware` maps to 404. Deleting a CANCELLED order is safe for stock because replenishment already happened at cancellation. There is no soft delete.

The refusal path is tested for a PAID order (`tests/orders.test.ts:197-218`); successful deletion and 404 are untested.

**Rule workflow**:
```
find order -> missing? 404 -> status not in {PENDING, CANCELLED}? 409 -> hard delete (cascade) -> 204
```

---

### Business Rule: List filtering, pagination and ordering

**Overview**:
Orders can be listed with optional filters and pagination, newest first.

**Detailed description**:
Query parameters: `page` (int >= 1, default 1), `pageSize` (int 1..100, default 20), `status` (enum), `customerId` (UUID), `from` and `to` (coerced dates) (`order.schemas.ts:23-30`). The service converts to `skip = (page-1)*pageSize`, `take = pageSize`. The repository builds a `where` object, adding `createdAt.gte = from` and `createdAt.lte = to` when provided (inclusive on both ends), sorts by `createdAt desc`, and obtains page and total count inside a single `$transaction` of two queries for a consistent snapshot (`order.repository.ts:21-45`).

The response is `{ data, pagination: { page, pageSize, total, totalPages } }`. List items are plain `Order` rows without items, history or customer. There is no validation that `from <= to` (an inverted range returns empty results). Sorting by `createdAt` alone is not a stable tiebreaker across pages. Invalid date strings coerce to Invalid Date and fail Zod validation (400). The status filter is tested (`tests/orders.test.ts:163-195`); pagination, customer and date filters are untested.

**Rule workflow**:
```
query -> Zod coerce/defaults -> skip/take -> where(status, customer, createdAt range)
  -> findMany(desc) + count (one transaction) -> paginated response
```

---

### Business Rule: Transactional atomicity of writes

**Overview**:
Order creation and status change are executed in one database transaction each.

**Detailed description**:
`create` and `changeStatus` use `this.prisma.$transaction(async (tx) => ...)` (`order.service.ts:58, 131`). Any thrown `AppError` rolls back everything: order, items, history, sequence increment, and stock changes. This is what guarantees that a failed stock check leaves no partial change and that sequence numbers are not consumed by failed creations.

Interactive transactions use Prisma defaults (timeout 5s, MySQL default isolation REPEATABLE READ); no custom options are set. Business validation reads (customer, products, order) are executed inside the transaction but without locking. Delete and list do not use explicit transactions except the count+find pair in the repository.

**Rule workflow**:
```
$transaction begin -> validations -> writes -> commit | throw -> rollback -> errorMiddleware
```

---

## 4. Component Structure

```
src/modules/orders/
├── order.controller.ts   # OrderController: HTTP handlers (list, getById, create, changeStatus, delete); maps results to status codes
├── order.service.ts      # OrderService: business logic, transactions, stock side effects, order number sequence
├── order.repository.ts   # OrderRepository: Prisma reads (list with filters, find with relations, findById) and deleteById; types OrderListFilters, OrderWithRelations
├── order.routes.ts       # buildOrderRouter: route table, authenticate + validate middleware wiring
├── order.schemas.ts      # Zod schemas and inferred input types
└── order.status.ts       # State machine transition table and stock-effect predicates

Wiring and related files outside the module:
src/app.ts:16-18,42-44             # Manual dependency injection
src/routes/index.ts:11,28          # Mount at /orders
src/middlewares/{auth,validate,error}.middleware.ts
src/shared/errors/http-errors.ts   # InvalidStatusTransitionError, InsufficientStockError, etc.
src/shared/http/response.ts        # paginated()
prisma/schema.prisma               # Order, OrderItem, OrderStatusHistory, OrderNumberSequence, OrderStatus
prisma/seed.ts                     # Seeds orders (own reserveOrderNumber copy at line 165)
tests/orders.test.ts, tests/helpers/factories.ts, tests/setup.ts, vitest.config.ts
```

Data model (prisma/schema.prisma): `Order` (unique `orderNumber`, indexes on customerId, status, createdAt, createdById), `OrderItem` (cascade on order, product FK without cascade), `OrderStatusHistory` (cascade on order, `fromStatus` nullable), `OrderNumberSequence` (single row id=1).

---

## 5. Dependency Analysis

Internal dependencies:

```
Router(order.routes) -> authenticate, validate, order.schemas, OrderController
OrderController -> OrderService, UnauthorizedError, ListOrdersQuery (type)
OrderService -> OrderRepository, PrismaClient (direct), order.status, order.schemas (types), shared/errors, shared/http/response
OrderRepository -> PrismaClient
order.schemas -> zod, @prisma/client (OrderStatus enum)
order.status -> @prisma/client (OrderStatus enum)
Composition root: src/app.ts constructs OrderRepository -> OrderService(repo, prisma) -> OrderController
```

Runtime cross-module data dependency: `OrderService` reads and writes `customer`, `product` (stock) and `orderNumberSequence` tables directly through `tx`, bypassing `CustomerRepository` and `ProductRepository` (no import of those modules).

External dependencies:
- express 4.21.1 (routing, RequestHandler types)
- @prisma/client 5.22.0 (ORM, enums, transactions) with MySQL
- zod 3.23.8 (validation)
- jsonwebtoken 9.0.2 (via authenticate middleware)
- pino (via error middleware logging)
- Test only: vitest 2.1.4, supertest 7.0.0, bcrypt 5.1.1 (test factories)

---

## 6. Afferent and Efferent Coupling

Counts are based on import statements and constructor usage within the repository (static analysis, approximate).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| OrderService | 2 (OrderController, app.ts) | 9 (OrderRepository, PrismaClient, order.status, order.schemas, ConflictError, InsufficientStockError, InvalidStatusTransitionError, NotFoundError, UnprocessableEntityError, ValidationError counted as error module, response helper) | High |
| OrderRepository | 2 (OrderService, app.ts) | 1 (PrismaClient) | Medium |
| OrderController | 2 (order.routes, app.ts) | 3 (OrderService, order.schemas type, UnauthorizedError) | Low |
| order.status (functions/constants) | 1 (OrderService) | 1 (OrderStatus enum) | High (encodes core state machine and stock triggers) |
| order.schemas | 3 (order.routes, order.controller type, order.service type) | 2 (zod, OrderStatus) | Medium |
| buildOrderRouter | 2 (src/routes/index.ts) | 5 (authenticate, validate, schemas, OrderController type, express Router) | Low |

Notes: Afferent figures for `OrderService` and `OrderRepository` count the composition root. The test suite couples to the HTTP surface and to Prisma models, not to these classes directly.

---

## 7. Endpoints

Base path: `/api/v1/orders` (mounted in `src/routes/index.ts:28`; the `/api/v1` prefix is inferred from the tests, defined in `src/app.ts`). All require `Authorization: Bearer <JWT>`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/orders | GET | List orders with filters (status, customerId, from, to) and pagination (page, pageSize); 200 |
| /api/v1/orders/{id} | GET | Get order with items (product summary), history and customer; 200 or 404 |
| /api/v1/orders | POST | Create order from customerId, items[], discountCents, notes; 201 |
| /api/v1/orders/{id}/status | PATCH | Change status (toStatus, reason); 200 |
| /api/v1/orders/{id} | DELETE | Delete order in PENDING or CANCELLED; 204 |

Error statuses used: 400 VALIDATION_ERROR, 401 UNAUTHORIZED, 404 NOT_FOUND, 409 INVALID_STATUS_TRANSITION / INVALID_ORDER_STATE_FOR_DELETE / CONFLICT, 422 INSUFFICIENT_STOCK / INACTIVE_PRODUCT, 500 INTERNAL_SERVER_ERROR. Error body shape: `{ error: { code, message, details? } }`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL via Prisma | Database | Persist orders, items, history, sequence; read customers and products; update stock | Prisma client over MySQL | Relational / typed models | Interactive transactions with rollback; Prisma P2002 -> 409, P2025 -> 404 in errorMiddleware; others -> 500 with logging |
| Auth middleware (JWT) | Internal | Identify the acting user | In-process | JWT (HS-style secret from env) | 401 UnauthorizedError |
| Customer and Product tables | Internal (data-level) | Validate existence, read price/active/stock, mutate stock | Direct Prisma access via tx | Rows | NotFoundError, UnprocessableEntityError, InsufficientStockError |
| Error middleware | Internal | Uniform error responses | In-process | JSON | Maps AppError, ZodError, Prisma errors |
| Pino logger | Internal | Logs unhandled errors only | In-process | JSON | n/a |

No external HTTP services, message brokers, events or payment gateways are called; the PAID status is set manually via API with no payment integration.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | Controller -> Service -> Repository | src/modules/orders/*.ts | Separation of HTTP, business, data concerns |
| Repository (partial) | OrderRepository | order.repository.ts | Read-side data access abstraction; writes bypass it |
| Dependency Injection (manual, constructor) | Composition in buildApp | src/app.ts:42-44 | Wiring and testability |
| State Machine (table-driven) | `transitions` record | order.status.ts:3-14 | Enforce legal status flow |
| Unit of Work / Transaction Script | `prisma.$transaction` in service methods | order.service.ts:58, 131 | Atomic multi-table changes |
| Schema validation / DTO | Zod schemas with inferred types | order.schemas.ts | Input contract and normalization |
| Middleware chain | authenticate, validate | order.routes.ts | Cross-cutting concerns |
| Domain exceptions + central error handler | AppError subclasses | shared/errors/http-errors.ts; error.middleware.ts | Consistent error contract |
| Audit log / event history | order_status_history | order.service.ts:159-167 | Traceability |
| Sequence table | order_number_sequence upsert | order.service.ts:245-254 | Human-readable sequential identifiers |
| Price snapshot | unitPriceCents on OrderItem | order.service.ts:75-84 | Historical price integrity |
| Factory function for router | buildOrderRouter(controller) | order.routes.ts:12 | Injectable routing |

Controller handlers are arrow-function properties to preserve `this` binding when passed to Express.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | OrderService.debitStock (order.service.ts:204-231) | Check-then-decrement without row lock or conditional update | Concurrent PAID transitions may oversell and produce negative stock |
| High | Authorization (order.routes.ts) | No role or ownership checks; `requireRole` unused; any authenticated user can delete or alter any order | Privilege and data-integrity exposure |
| Medium | OrderService.delete (order.service.ts:181-192) | Check and delete outside a transaction | Race with status change could delete a paid order without stock replenishment |
| Medium | History cascade delete (schema.prisma OrderStatusHistory) | Deleting an order erases its audit trail; hard delete only | Loss of traceability |
| Medium | OrderService write paths | Service uses raw PrismaClient/tx for customer, product, sequence, bypassing repositories | Inconsistent layering; business logic coupled to schema of other modules |
| Medium | Stock lifecycle | Stock debited at PAID, not at creation; product `active` not rechecked at PAID | Possible acceptance of orders that cannot be fulfilled; inactive product could be paid |
| Medium | Status transitions concurrency | Order read inside tx without lock; two concurrent transitions could both succeed (for example double PAID debit paths) | Inconsistent state or double stock effects (depends on isolation behavior) |
| Low | Redundant guard (order.service.ts:51-53) | Empty-items check unreachable via HTTP | Dead-path code |
| Low | Duplicate include blocks (order.service.ts:115-119, 171-175; repository.ts:50-58) | Same relation include repeated three times | Maintenance drift risk |
| Low | List endpoint | No from <= to validation; createdAt-only sort (unstable pagination); list rows lack relations | Minor correctness and usability |
| Low | Order number | Fixed 6-digit format overflows after 999999; duplicated logic in prisma/seed.ts:165 | Format inconsistency; duplicated rule |
| Low | Exported helpers `allowedTransitions`, `isTerminal` | Unused in module | Dead code |
| Low | Int overflow | totals are 32-bit Int with no upper bounds on quantity/items | Overflow on extreme inputs (unverified) |
| Low | Error specificity | "Product not found" does not name the missing id | Harder client diagnostics |
| Medium | Tests | Only one integration test file, no unit tests for state machine or service | Regressions in untested rules |

No Prisma `@@check` or DB-level constraint for non-negative stock was found in `prisma/schema.prisma`; migration SQL was not inspected line by line for CHECK constraints (unverified).

---

## 11. Test Coverage Analysis

Test files located in the project:
- `tests/orders.test.ts` (8 tests, integration via supertest against real MySQL through Prisma)
- `tests/helpers/factories.ts` (bootstrap user/token, customers, products, app builder)
- `tests/setup.ts` (connects Prisma; truncates order, product, customer, user tables before each test)
- `vitest.config.ts` (tests/**/*.test.ts, serial single fork, 30s timeouts)
- `tests/auth.test.ts` (authentication only, not orders)

No coverage provider or threshold is configured; the percentages below are not measured and are qualitative estimates of rule coverage.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| OrderController | 0 | 8 (indirect) | Not measured; create, changeStatus, list, delete(409 path) exercised; getById never called | Happy paths good; no 401 test on orders |
| OrderService.create | 0 | 2 | Not measured | Totals and number format asserted; missing: customer not found, inactive product, discount > subtotal, duplicate item aggregation, empty items, notes |
| OrderService.changeStatus | 0 | 4 | Not measured | PENDING->PAID, invalid PENDING->SHIPPED, insufficient stock, PAID->CANCELLED covered; missing: same-status, PROCESSING cancel, SHIPPED/DELIVERED, terminal states, 404, history reason |
| OrderService.delete | 0 | 1 | Not measured | Only 409 path; no successful delete, 404, or CANCELLED/PENDING delete |
| OrderService.list / OrderRepository.list | 0 | 1 | Not measured | Status filter and total asserted; missing: pagination, customer filter, date range, ordering |
| OrderRepository.findByIdWithRelations / findById | 0 | 0 direct (relations returned in create/changeStatus responses use inline queries) | Not measured | GET /orders/:id untested; repository relation query untested |
| order.status.ts (state machine) | 0 | 3 edges indirectly | Not measured | No exhaustive transition matrix test |
| order.schemas.ts | 0 | 0 explicit | Not measured | Validation failures (400) not asserted |
| Order number sequence | 0 | 1 (regex only) | Not measured | No sequential or concurrency test |
| Concurrency / race conditions | 0 | 0 | None | Stock and delete races untested |

Test quality observations: tests assert both HTTP responses and database side effects (stock quantities), which is strong. Each test bootstraps its own user, customer and products and global truncation in `beforeEach` provides isolation, at the cost of requiring a live database and serial execution. External dependencies are not mocked.

---
