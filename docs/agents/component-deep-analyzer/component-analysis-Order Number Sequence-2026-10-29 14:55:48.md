# Component Deep Analysis Report: Order Number Sequence

Analysis date: 2026-10-29 14:55:48
Scope: project root, no ignored folders. Analysis only; no project files were modified.

## 1. Executive Summary

The Order Number Sequence is a single-row counter table (`order_number_sequence`, Prisma model `OrderNumberSequence`) that produces human-readable, monotonically increasing order identifiers in the format `ORD-NNNNNN` (zero-padded to 6 digits). It is consumed by the Orders module at order creation time (`OrderService.reserveOrderNumber`) and duplicated in the database seed script (`prisma/seed.ts`). The result is stored in `orders.orderNumber` (VARCHAR(20), UNIQUE).

Key findings:
- The counter is a singleton row (`id = 1`) incremented by a Prisma `upsert` inside the same database transaction that creates the order. Serialization comes from the row lock taken by the UPDATE, and a rolled-back order creation also rolls back the increment (gapless numbering, except for deleted orders).
- The only copy of the generation algorithm in application code is private to `OrderService`; a second, independent copy exists in `prisma/seed.ts` (duplication).
- Delete of an order does not return its number; numbers are never reused.
- The counter is not covered by direct tests. Only the output format is asserted (`tests/orders.test.ts:39`). Concurrency, rollover beyond 999999, and first-use paths are untested.
- The format is documented nowhere in docs/ (PRD, FDD, RFC, ADRs contain no mention of the `ORD-` format or the sequence); the rule is implicit in code (confidence: high for code behavior, none for documented intent).
- Behavior of Prisma 5.22.0 `upsert` against MySQL under concurrent first-ever use is a potential race (see risks). This is stated as a risk to verify, not a confirmed defect.

## 2. Data Flow Analysis

Order creation (`POST /api/v1/orders`):

```
1. Request enters via router.post('/') in src/modules/orders/order.routes.ts:18 (after authenticate middleware, line 14)
2. Body validated by createOrderSchema (validate middleware) in src/modules/orders/order.schemas.ts
3. OrderController.create delegates to OrderService.create (src/modules/orders/order.service.ts:50)
4. Input checks: non-empty items (line 51), aggregation of duplicate products (line 55)
5. prisma.$transaction opens (line 58)
6. Customer, product existence, active status, stock-independent price and discount checks (lines 59-91)
7. reserveOrderNumber(tx) called at line 93 (only after all validations pass)
8.   tx.orderNumberSequence.upsert({ where: {id: 1}, create: {id:1, nextValue:2}, update: {nextValue: {increment: 1}} }) (lines 246-251)
9.   current = returned nextValue - 1  (line 252)
10.  orderNumber = "ORD-" + current zero-padded to 6 digits (line 253)
11. tx.order.create with orderNumber, items and initial history (lines 95-120)
12. Transaction commits (row lock on sequence row released) or rolls back (counter increment undone)
13. Order with relations returned; serialized to JSON by the controller (orderNumber appears in the response)
```

Seed flow (`prisma/seed.ts`): `main()` deletes all data including `orderNumberSequence.deleteMany()` (line 187, resetting the counter), then for each planned order calls the local `reserveOrderNumber()` (line 285, defined lines 165-175), which wraps the same upsert in its own transaction, and then creates the order in a separate non-transactional call (line 289 onward).

Test flow: `tests/setup.ts:12` clears the sequence table before each test so each test starts at `ORD-000001`.

State of the counter row:
- Empty table: first reservation creates `{id:1, nextValue:2}` and yields number 1.
- Row present: `nextValue` is incremented and the returned (post-increment) value minus 1 is the number issued. `nextValue` therefore always holds the next number to be issued.

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Format | Order number format is `ORD-` plus the sequence value left-padded with zeros to 6 digits | src/modules/orders/order.service.ts:253; prisma/seed.ts:174 |
| Business Logic | Single global sequence (one row, id fixed to 1) shared by all orders | prisma/schema.prisma:134; src/modules/orders/order.service.ts:247-248 |
| Business Logic | First number issued is 1 (lazy initialization: row created with nextValue = 2) | src/modules/orders/order.service.ts:248-249 |
| Business Logic | Each reservation increments the counter by exactly 1 | src/modules/orders/order.service.ts:249 |
| Consistency | Number is reserved inside the order-creation transaction; a failed creation does not consume a number | src/modules/orders/order.service.ts:58, 93 |
| Consistency | Number is reserved only after all validations (customer, products, active, discount) pass | src/modules/orders/order.service.ts:59-93 |
| Integrity | `orders.orderNumber` is UNIQUE and VARCHAR(20) | prisma/schema.prisma:76; migration.sql:52,63 |
| Business Logic | Numbers are never reused or recycled; deleting an order does not decrement the counter | src/modules/orders/order.service.ts:181-192 (no sequence interaction) |
| Business Logic | Number is server-generated; client cannot supply it | src/modules/orders/order.schemas.ts (createOrderSchema has no orderNumber field) |
| Data Lifecycle | Seed and tests reset the counter by deleting the sequence row | prisma/seed.ts:187; tests/setup.ts:12 |
| Error Handling | A unique violation on orderNumber (P2002) is mapped to HTTP 409 CONFLICT | src/middlewares/error.middleware.ts:41-50 |

### Detailed breakdown of the business rules

---

### Business Rule: Order Number Format (ORD-NNNNNN)

**Overview**:
Every order receives a customer/operator-facing identifier built as the literal prefix `ORD-` followed by the numeric sequence value zero-padded to six digits (e.g. `ORD-000001`, `ORD-000042`).

**Detailed description**:
The format is produced by `String(current).padStart(6, '0')` prefixed with `ORD-` in `OrderService.reserveOrderNumber` (src/modules/orders/order.service.ts:253) and identically in the seed script (prisma/seed.ts:174). It exists in addition to the internal UUID primary key (`orders.id`, CHAR(36)), providing a short, readable reference for people while the UUID is used in routes (`/orders/:id`).

The only test assertion about the format is a regular expression `^ORD-\d{6}$` in tests/orders.test.ts:39. Note that `padStart` does not truncate: once the counter exceeds 999999 the number becomes 7 digits (`ORD-1000000`), which would no longer match the regex used in the test, although it still fits the VARCHAR(20) column. Also, the counter column is a 32-bit signed INTEGER (migration.sql:103-104), so the technical ceiling is 2,147,483,647; no explicit handling of overflow exists.

The prefix and width are hardcoded literals, not configuration (src/config/env.ts has no related setting). No documentation in docs/ describes the format; it is inferred from code and test (confidence: high that this is the implemented behavior; documented intent unknown).

**Rule workflow**:
```
sequenceValue (int) -> String() -> padStart(6,'0') -> prepend "ORD-" -> stored in orders.orderNumber
```

---

### Business Rule: Single Global Sequence (Singleton Row)

**Overview**:
One counter serves all orders. It is modelled as a table with exactly one logical row identified by `id = 1`.

**Detailed description**:
`OrderNumberSequence` (prisma/schema.prisma:133-138) has `id Int @id @default(1)` and `nextValue Int @default(1)`, mapped to table `order_number_sequence`. Application code always addresses the row with `where: { id: 1 }` and, on creation, explicitly sets `id: 1`. Nothing in the schema prevents additional rows with other ids (no CHECK constraint), but no code path reads or writes any other id, so the singleton nature is a convention enforced only by the code.

There is no partitioning by customer, date, year or tenant; all orders share one number space. This makes the sequence a global serialization point: every order creation takes a write lock on the same row until commit.

Note a naming/semantic detail: the Prisma default `nextValue = 1` suggests "the next number to issue is 1", while the code's lazy-create path stores `nextValue: 2` after issuing number 1. The two are consistent in meaning only because the code always issues `nextValue - 1` after incrementing (i.e. the field holds the next number to issue after the current reservation). The schema default of 1 is never exercised by application code (create always passes an explicit value).

**Rule workflow**:
```
Row missing -> create {id:1, nextValue:2} -> issue 1
Row present -> nextValue += 1 -> issue (new nextValue - 1)
```

---

### Business Rule: Lazy Initialization and Atomic Increment

**Overview**:
The counter row is not created by migration or seed; it is created on first use by an `upsert`. Subsequent uses increment it.

**Detailed description**:
The migration (prisma/migrations/20260519182739_init/migration.sql:102-108) only creates the table and inserts no row. `reserveOrderNumber` uses `tx.orderNumberSequence.upsert` with `create: { id: 1, nextValue: 2 }` and `update: { nextValue: { increment: 1 } }`, selecting `nextValue` back. Because the increment is expressed as a database-side atomic operation (`nextValue = nextValue + 1`) rather than read-modify-write in application code, concurrent transactions do not compute the same value once the row exists: the UPDATE acquires an InnoDB row lock held until the surrounding transaction commits or rolls back, so concurrent order creations are serialized on this row.

The returned `nextValue` is the value after increment; the code subtracts 1 to obtain the number assigned to this order. This is correct only when the returned value is read in the same statement/transaction as the increment, which is the case here.

For the very first reservation on an empty table, Prisma's upsert against MySQL is, to my understanding of Prisma 5.x behavior, not executed as a single native `INSERT ... ON DUPLICATE KEY UPDATE`, but as a lookup followed by create or update. Two transactions racing on an empty table could therefore both attempt to create `id = 1`; one would fail with a unique-constraint error (P2002), which the error middleware reports as HTTP 409 CONFLICT. This is documented here as an unverified behavior (confidence: medium) and as a risk; it was not executed or tested.

**Rule workflow**:
```
BEGIN (order creation)
  upsert order_number_sequence id=1
    exists?  -> UPDATE nextValue = nextValue + 1 (row lock)
    missing? -> INSERT (1, 2)
  read nextValue -> current = nextValue - 1
  format ORD-xxxxxx
  INSERT order
COMMIT / ROLLBACK
```

---

### Business Rule: Reservation Inside the Order Transaction (Gapless on Failure)

**Overview**:
The number is reserved within the same interactive transaction that persists the order, so any failure after reservation undoes the increment.

**Detailed description**:
`OrderService.create` wraps all work in `this.prisma.$transaction(async (tx) => ...)` (line 58) and calls `reserveOrderNumber(tx)` at line 93, passing the transaction client (`TxClient = Prisma.TransactionClient`, line 24). If `tx.order.create` (line 95) or any later step throws, the transaction rolls back, including the sequence update, so the number is not consumed. This differs from database AUTO_INCREMENT or a sequence object, which would leave gaps after rollbacks; it is the main reason a table-based counter is used here (rationale is inferred, not documented).

The reservation is placed deliberately late: after customer lookup (NotFoundError), product existence and active checks (NotFoundError, UnprocessableEntityError INACTIVE_PRODUCT) and the discount check (ValidationError). Because the row lock is taken only at line 93 and held until commit, the lock window is minimized to the order insert, item inserts and history insert; validation failures never touch the sequence.

The seed script behaves differently: its `reserveOrderNumber` runs in its own transaction that commits immediately, and the order is created afterward in a separate statement (prisma/seed.ts:285-289). A failure in order creation during seeding would therefore leave a gap. This is acceptable for seed data but is a divergence from the service behavior.

**Rule workflow**:
```
validations pass -> reserve (lock row) -> insert order/items/history -> commit (release lock)
any error after reserve -> rollback (counter restored)
```

---

### Business Rule: Uniqueness and Non-Reuse of Order Numbers

**Overview**:
`orderNumber` is unique across orders, and numbers are never recycled, even after deletion.

**Detailed description**:
`orders.orderNumber` has a unique index (`orders_orderNumber_key`, migration.sql:63; prisma/schema.prisma:76). This is the safety net against duplicates caused by any defect in the counter (for example manual edits to the sequence table or a reset of the counter while orders exist). Such a duplicate would surface as Prisma error P2002, converted by `errorMiddleware` to HTTP 409 with message "Unique constraint violation on: ..." (src/middlewares/error.middleware.ts:41-50). Note the transaction would roll back, so the failed request would consume no number; however, if the counter is lower than the highest existing number, every subsequent creation would fail until the counter passes it.

`OrderService.delete` (lines 181-192) permits deletion of orders in PENDING or CANCELLED status and calls `OrderRepository.deleteById`; it does not touch the sequence. Deleted orders therefore leave permanent gaps in the sequence. Order items and status history cascade on delete (schema onDelete: Cascade); the sequence row is unrelated.

Deleting the sequence row while orders exist (as `prisma/seed.ts:187` and `tests/setup.ts:12` do, but only after deleting orders first) would restart numbering at 1. In both of those scripts orders are deleted before the sequence, so uniqueness is preserved.

**Rule workflow**:
```
Create: reserve N -> insert orders(orderNumber=ORD-N) -> UNIQUE index enforces
Delete: remove order row only; counter unchanged; N never reissued
Duplicate detected -> P2002 -> HTTP 409 CONFLICT
```

---

### Business Rule: Server-Side Generation Only

**Overview**:
Clients cannot specify or alter an order number.

**Detailed description**:
`createOrderSchema` (src/modules/orders/order.schemas.ts) accepts customerId, items, discountCents and notes; there is no `orderNumber` field, and `OrderService.create` always builds the data object with the server-reserved value (order.service.ts:97). The status-change endpoint and delete endpoint only operate on UUIDs. There is no update path that changes `orderNumber` after creation, making it immutable in practice (by code convention; no database trigger or constraint prevents direct modification).

The webhook requirement summarized in docs/SUMMARY.md:37 (REQ-011) mentions that a status-change event payload should include `order_number`; no webhook implementation exists in the source tree analyzed, so this is a documented future consumer of the number, not an implemented one.

**Rule workflow**:
```
Client payload (no orderNumber) -> Zod validation -> service reserves number -> persisted -> returned in response
```

---

### Business Rule: Counter Reset in Non-Production Data Lifecycle

**Overview**:
The seed script and the test setup reset numbering by deleting the sequence row.

**Detailed description**:
`prisma/seed.ts:187` deletes the sequence table contents at the end of its cleanup sequence (after orders, items, history, products, customers and users). `tests/setup.ts:12` deletes it in `beforeEach` immediately after deleting orders. As a result every test and every seed run begins numbering at `ORD-000001`, which makes the number deterministic in tests (though only its pattern is asserted).

There is no application code path (endpoint, admin operation) to reset or adjust the counter. Production adjustments would require direct database access.

**Rule workflow**:
```
cleanup: delete history -> items -> orders -> (products/customers/users) -> sequence
next reservation -> lazy create -> ORD-000001
```

---

## 4. Component Structure

The component has no dedicated directory; it is spread across the schema, one service method, the seed and test setup.

```
prisma/
├── schema.prisma                         # model OrderNumberSequence (lines 133-138); Order.orderNumber (line 76)
├── migrations/20260519182739_init/
│   └── migration.sql                     # CREATE TABLE order_number_sequence (102-108); orders.orderNumber unique index (52, 63)
└── seed.ts                               # duplicated reserveOrderNumber (165-175); reset at 187; use at 285-289
src/
├── modules/orders/
│   └── order.service.ts                  # reserveOrderNumber (private, 245-254); call at line 93
├── middlewares/error.middleware.ts       # P2002 -> 409 mapping (41-50)
└── config/database.ts                    # PrismaClient singleton providing the transaction client
tests/
├── setup.ts                              # sequence reset before each test (line 12)
└── orders.test.ts                        # format assertion (line 39)
```

Data model:

| Table | Column | Type | Constraints |
|-------|--------|------|-------------|
| order_number_sequence | id | INTEGER | PK, default 1 |
| order_number_sequence | nextValue | INTEGER | NOT NULL, default 1 |
| orders | orderNumber | VARCHAR(20) | NOT NULL, UNIQUE |

## 5. Dependency Analysis

Internal Dependencies:
```
OrderController -> OrderService -> reserveOrderNumber (private) -> Prisma TransactionClient -> OrderNumberSequence model
OrderService -> Order model (orderNumber persisted)
errorMiddleware -> Prisma.PrismaClientKnownRequestError (P2002 handling for duplicate orderNumber)
seed.ts -> own reserveOrderNumber -> PrismaClient
tests/setup.ts -> prisma.orderNumberSequence.deleteMany
```

External Dependencies:
- @prisma/client 5.22.0 and prisma 5.22.0 - ORM, upsert and increment operators, transactions (package.json:26, 48)
- MySQL 8.0 (docker-compose.yml) - InnoDB row locking provides serialization
- vitest 2.1.4 and supertest 7.0.0 - tests

No dependency on Redis, queues, or external ID services.

## 6. Afferent and Efferent Coupling

Components are the model, class methods and scripts that touch the sequence.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| OrderNumberSequence (Prisma model/table) | 3 (OrderService.reserveOrderNumber, seed.ts, tests/setup.ts) | 0 | High |
| OrderService.reserveOrderNumber (private method) | 1 (OrderService.create) | 1 (OrderNumberSequence via TxClient) | High |
| seed.ts reserveOrderNumber | 1 (seed main) | 1 (OrderNumberSequence) | Low |
| Order.orderNumber (column) | 3 (OrderService.create, seed.ts, tests/orders.test.ts assertion) | 0 | Medium |
| errorMiddleware (P2002 branch) | Express pipeline | 1 (Prisma error types) | Low (for this component) |

## 7. Endpoints

The component exposes no endpoints of its own. It is used indirectly by one endpoint, and its output is returned in order payloads.

| Endpoint | Method | Relation to the component |
|----------|--------|---------------------------|
| /api/v1/orders | POST | Reserves a number and returns `orderNumber` |
| /api/v1/orders, /api/v1/orders/{id} | GET | Return `orderNumber` as part of Order records (no lookup by number exists) |

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (order_number_sequence table) | Database | Persistent counter | MySQL via Prisma | Relational rows | Transaction rollback; P2002 mapped to 409 |
| Orders module | Internal module | Number consumer | In-process call | string `ORD-NNNNNN` | Exceptions propagate and roll back transaction |
| Webhook (REQ-011 in docs/SUMMARY.md) | Planned consumer | Include order_number in event | Not implemented in code | JSON | Not applicable |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Counter table / database-backed sequence | Single-row table with atomic increment | prisma/schema.prisma:133-138 | Gapless-on-rollback, portable ID generation |
| Unit of Work / Transaction Script | Reservation joined to order creation transaction via TxClient | order.service.ts:58, 93 | Atomicity |
| Upsert (lazy initialization) | `upsert` create/update branches | order.service.ts:246-251 | No bootstrap row required |
| Singleton row | Fixed id = 1 | schema.prisma:134 | Single global counter |
| Surrogate vs natural key | UUID `id` plus business key `orderNumber` | schema.prisma:75-76 | Separates technical identity from readable reference |
| Pessimistic serialization (implicit) | InnoDB row lock held by UPDATE until commit | order.service.ts:249 | Prevent duplicate numbers |

Architecture notes: the generator is not abstracted behind its own class or repository; it is a private method in the service that accesses the Prisma transaction client directly, bypassing `OrderRepository` (which exposes no sequence method).

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Throughput | All order creations contend for one row lock held until the end of the transaction (which includes order, items and history inserts) | Serialized order creation; latency grows under concurrency |
| Medium | Initialization | Upsert on empty table under concurrent first requests may race (Prisma upsert on MySQL is not necessarily a native single statement; unverified) | Possible P2002 returned as 409 on the very first concurrent orders |
| Medium | Code duplication | `reserveOrderNumber` implemented twice (order.service.ts:245-254, prisma/seed.ts:165-175) | Format or algorithm drift between service and seed |
| Medium | Layering | Service accesses `tx.orderNumberSequence` directly rather than through a repository | Inconsistent with repository usage elsewhere; harder to isolate |
| Low | Format limit | `padStart(6)` does not cap; above 999999 the number is 7+ digits and fails the test regex `^ORD-\d{6}$` | Format inconsistency at scale (still fits VARCHAR(20)) |
| Low | Capacity | Counter is 32-bit INT | Overflow at 2,147,483,647 with no handling |
| Low | Schema semantics | Default `nextValue = 1` conflicts with the code's convention (`nextValue` stored as next-after-current; first row created with 2) | Confusion if rows are inserted manually via the default |
| Low | Singleton enforcement | No constraint prevents rows with id other than 1 | Extra rows are ignored silently |
| Low | Gaps | Deleted orders (PENDING/CANCELLED) leave permanent gaps | Number sequence not contiguous; may matter if gaplessness is a requirement (not documented) |
| Low | Documentation | Format, prefix and rationale absent from PRD/FDD/RFC/ADRs | Intent not traceable |
| Low | Configuration | Prefix and width hardcoded | Change requires code edit |
| Low | Seed | Seed reserves outside the order insert; a failure leaves gaps | Cosmetic in seed data |
| Low | Observability | No logging of reserved numbers or of reservation failures | Harder to diagnose duplicates |

Security: the number is sequential and therefore predictable/enumerable; however, all order endpoints require authentication (`router.use(authenticate)`, order.routes.ts:14) and routes use UUIDs, so predictability of the number alone does not grant access (confidence: high for the code reviewed).

## 11. Test Coverage Analysis

Test files located: tests/orders.test.ts, tests/setup.ts, tests/helpers/factories.ts, tests/auth.test.ts (unrelated). Configuration: vitest.config.ts (single fork, sequential files, setup file tests/setup.ts). No coverage provider or threshold is configured, and no coverage report was found; coverage percentages below are not measured.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| OrderService.reserveOrderNumber | 0 | 1 indirect (tests/orders.test.ts:12-40, regex at line 39); other create calls (lines 59+, 89+, 109+, 134+, 163+, 197+) exercise it without asserting | Not measured; happy path and lazy-create path only | Format asserted loosely by regex only; the exact value (e.g. ORD-000001), increments, uniqueness across orders, rollback behavior, concurrency, and rollover are not tested |
| OrderNumberSequence table | 0 | 0 direct | Not measured | Only reset in tests/setup.ts:12; the update (increment) branch of the upsert is exercised only in tests that create more than one order per test (e.g. the first test creates one order, so the update branch is exercised only if another test within the same test creates a second order; none was found that asserts it) |
| Order.orderNumber unique constraint | 0 | 0 | Not measured | No test for P2002 mapping on orderNumber |
| seed.ts reserveOrderNumber | 0 | 0 | Not measured | Not tested |

Highlighted risks: missing tests for concurrent creation, rollback not consuming a number, deletion not reusing numbers, and format overflow.

Note on the factories helper: tests/helpers/factories.ts does not create orders directly or touch the sequence.
