# Component Deep Analysis Report: Seed Script (prisma/seed.ts)

Analysis date: 2026-10-29. Scope: entire project root. Ignored folders: none.

## 1. Executive Summary

The Seed Script (`prisma/seed.ts`, 335 lines) is a standalone, one-shot TypeScript program that populates the MySQL database of the Order Management System (OMS) API with deterministic demonstration data. It is not part of the runtime API (`src/`), and no application code imports it. It is invoked via `npm run db:seed` (`package.json:17`, `tsx --env-file=.env prisma/seed.ts`) or automatically by Prisma after `prisma migrate reset` through the `prisma.seed` hook (`package.json:24-26`, `tsx prisma/seed.ts`).

Its behavior in order:
1. Wipes all seven tables (destructive, unconditional).
2. Creates two users (one ADMIN, one OPERATOR) with bcrypt-hashed, hardcoded passwords.
3. Creates 10 customers and 20 products from in-file literal arrays.
4. Creates 26 orders across all six statuses, with items, price snapshots, an occasional 5% discount, sequential order numbers, and a synthetic status history trail.

Key findings:
- The script re-implements, rather than reuses, application logic: order-number generation (`prisma/seed.ts:165-176` mirrors `src/modules/orders/order.service.ts:245-254`), the bcrypt cost factor (10, mirrors `src/modules/users/user.service.ts:16`), and a status path table (`prisma/seed.ts:255-262`) derived from, but not shared with, `src/modules/orders/order.status.ts`.
- Seed data does not run through the service layer, so it bypasses stock debit/replenish rules. Stock quantities are the initial values from the product literals, regardless of the PAID/PROCESSING/SHIPPED/DELIVERED orders that reference them.
- Credentials (`admin@oms.local` / `admin123`, `operador@oms.local` / `operator123`) are hardcoded, with no environment guard.
- No automated tests cover the script; no production-environment safeguard exists before `deleteMany()` calls.

## 2. Data Flow Analysis

```
1. Process start: `tsx` executes prisma/seed.ts; module-level `new PrismaClient()` (seed.ts:4) is created; DATABASE_URL comes from .env (--env-file) or Prisma's own loading when run by `prisma migrate reset`.
2. main() invoked (seed.ts:327).
3. Cleanup (seed.ts:181-187): deleteMany on orderStatusHistory, orderItem, order, product, customer, user, orderNumberSequence (child tables first).
4. Users (seed.ts:191-210): bcrypt.hash(ADMIN_PASSWORD, 10) and bcrypt.hash(OPERATOR_PASSWORD, 10); prisma.user.create for admin and operator.
5. Customers (seed.ts:214-216): Promise.all over customersSeed (10 records) -> prisma.customer.create (address stored as JSON).
6. Products (seed.ts:220-222): Promise.all over productsSeed (20 records) -> prisma.product.create.
7. Orders loop (seed.ts:264-321), sequential for each of 26 plan entries:
   a. Pick customer round-robin (i % 10) and author (admin if i % 3 === 0 else operator).
   b. Build items: unit price snapshot from product.priceCents; line total = unit * qty.
   c. subtotal = sum of line totals; discount = floor(5% of subtotal) when i % 7 === 0; total = subtotal - discount.
   d. reserveOrderNumber() (seed.ts:165-176): interactive transaction, upsert order_number_sequence row id=1, derive "ORD-" + 6-digit zero-padded number.
   e. prisma.order.create with nested items.create (seed.ts:287-305).
   f. Build status path = flowFromStatus[status] + [status]; create one orderStatusHistory row per step (seed.ts:307-320), fromStatus null for the first.
8. Log "Seed completed." (seed.ts:324).
9. On error: console.error and process.exit(1) (seed.ts:328-332). In all cases, prisma.$disconnect() (seed.ts:333-335).
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Data Reset | All tables are fully cleared before seeding, children first | prisma/seed.ts:181-187 |
| Security / Auth | Two fixed users: ADMIN and OPERATOR, with fixed credentials | prisma/seed.ts:6-7, 194-210 |
| Security | Passwords hashed with bcrypt, cost factor 10 | prisma/seed.ts:191-192 |
| Reference Data | 10 Brazilian customers (8 individuals with CPF, 2 companies with CNPJ), JSON address | prisma/seed.ts:9-140 |
| Reference Data | 20 products with SKU, price in cents, initial stock | prisma/seed.ts:142-163 |
| Distribution | Customer assigned round-robin (i % 10) | prisma/seed.ts:266 |
| Distribution | Order author alternates: admin when i % 3 === 0, otherwise operator | prisma/seed.ts:267 |
| Pricing | Unit price snapshot copied from product at order time; line total = unit * qty | prisma/seed.ts:269-279 |
| Pricing | Discount of 5% (floored to integer cents) on orders with index i % 7 === 0 | prisma/seed.ts:281-283 |
| Numbering | Order number ORD-NNNNNN from a single-row sequence table (id=1) inside a transaction | prisma/seed.ts:165-176, 285 |
| Order Status | 26 orders distributed across PENDING (4), PAID (5), PROCESSING (4), SHIPPED (4), DELIVERED (6), CANCELLED (3) | prisma/seed.ts:226-253 |
| Audit Trail | Synthetic status history built from a fixed path per final status | prisma/seed.ts:255-262, 307-320 |
| Stock | Stock is not adjusted by seeded orders | prisma/seed.ts (absence), contrast order.service.ts:228 |
| Error Handling | Any failure logs error and exits with code 1; client always disconnected | prisma/seed.ts:327-335 |

### Detailed breakdown of the business rules

---

### Business Rule: Full Data Reset Before Seeding

**Overview**:
Every execution deletes all rows from all domain tables before inserting new data, making the script idempotent in outcome (same end state on every run) but destructive.

**Detailed description**:
`main()` begins by calling `deleteMany()` on `orderStatusHistory`, `orderItem`, `order`, `product`, `customer`, `user` and `orderNumberSequence` in that order (`prisma/seed.ts:181-187`). The order respects foreign key dependencies: history and items reference orders, orders reference customers and users, and items reference products. (The schema also declares `onDelete: Cascade` from order to items and history, `prisma/schema.prisma`, but the script deletes explicitly anyway.)

There is no confirmation prompt, no check of `NODE_ENV`, and no check of the database name or host. Whatever `DATABASE_URL` points to will be fully wiped. Because the deletions are separate statements and not wrapped in a transaction, a failure midway leaves a partially cleaned database.

The sequence table is also cleared, so the first reserved order number after a reset is always `ORD-000001`, which is what makes the generated order numbers deterministic across runs. The same cleanup list is used by the test suite setup (`tests/setup.ts:7-13`, with a slightly different order), meaning running tests against the same database also erases seeded data.

**Rule workflow**:
```
start -> delete history -> delete items -> delete orders -> delete products
      -> delete customers -> delete users -> delete order number sequence -> proceed to seeding
```

---

### Business Rule: Default Users and Roles

**Overview**:
Two accounts are provisioned so the API can be exercised immediately: an ADMIN and an OPERATOR.

**Detailed description**:
The script defines `ADMIN_PASSWORD = 'admin123'` and `OPERATOR_PASSWORD = 'operator123'` (`prisma/seed.ts:6-7`). It creates `admin@oms.local` ("Administrador do Sistema", role `UserRole.ADMIN`) and `operador@oms.local` ("Operador Padrao", role `UserRole.OPERATOR`) (`prisma/seed.ts:194-210`). The login flow in `src/modules/auth/auth.service.ts:36` compares credentials with `bcrypt.compare`, so these accounts work for authentication once seeded.

The passwords are plain literals in source, are not read from environment variables, and are not rotated. The hash cost of 10 matches `BCRYPT_ROUNDS` in `src/modules/users/user.service.ts:16`, but the value is duplicated rather than shared. Emails use the `.local` pseudo-domain, which is non-routable, indicating development-only intent; this is an inference, since no comment documents it.

The created `admin` and `operator` records are later used as authors (`createdById`) of orders and (`changedById`) of status history rows, so the user step is a prerequisite for order seeding.

**Rule workflow**:
```
hash admin password (cost 10) -> hash operator password (cost 10)
 -> create admin user (ADMIN) -> create operator user (OPERATOR)
 -> keep records in memory for order authorship
```

---

### Business Rule: Customer Reference Data

**Overview**:
Ten customers with Brazilian-formatted data are inserted verbatim from a static array.

**Detailed description**:
`customersSeed` (`prisma/seed.ts:9-140`) holds name, unique email, E.164-style phone (`+55` prefix), a `document` and a JSON `address` (street, number, city, state, zipCode). Eight records use CPF format (`NNN.NNN.NNN-NN`) and two (`Logistica Atlas Comercial Ltda`, `Distribuidora Nordeste S.A.`) use CNPJ format (`NN.NNN.NNN/NNNN-NN`), so both individual and corporate customers are represented. Cities span ten Brazilian states (SP, RJ, MG, PR, RS, BA, PE, GO, CE; SP appears twice).

The values satisfy schema constraints: email is `@unique` (all ten are distinct), `document` is VarChar(20) (max length used is 18), `phone` VarChar(32). The documents are visibly fabricated placeholder values (e.g. ending in `-00`); the script performs no CPF/CNPJ check-digit validation, and no such validation was found in the seed. Customers are created concurrently with `Promise.all` (`prisma/seed.ts:214-216`), so insertion order, and therefore `createdAt` ordering, is not guaranteed, although the returned array preserves the input order, which the order loop depends on.

**Rule workflow**:
```
for each customersSeed entry (parallel) -> prisma.customer.create -> array of created customers in input order
```

---

### Business Rule: Product Catalog and Pricing in Cents

**Overview**:
Twenty products are inserted with integer prices in cents and starting stock quantities.

**Detailed description**:
`productsSeed` (`prisma/seed.ts:142-163`) defines SKU (unique), name, description, `priceCents` and `stockQuantity`. Monetary values are stored as integer cents (e.g. 459900 = R$ 4,599.00), consistent with the schema (`priceCents Int`). The catalog covers computers, peripherals, furniture and office supplies. `active` is not set, so it takes the schema default `true`.

Stock ranges from 12 to 300 units. The seeded order quantities (the largest being 50 units of index 17, `PAP-A4-001`, stock 300) never exceed the starting stock, but because seeded orders do not debit stock (see the stock rule below), this is incidental. Products are created with `Promise.all`, and the returned array order is relied upon by index in the order plan (`products[idx]`).

One data detail: product `PEN-PLT-001` has the name "Caneta Pilot G2 Pretra" (likely a typo for "Preta"); this is cosmetic.

**Rule workflow**:
```
for each productsSeed entry (parallel) -> prisma.product.create (active defaults to true) -> array indexed 0..19
```

---

### Business Rule: Order Plan and Status Distribution

**Overview**:
A fixed plan of 26 orders defines items, quantities and final status so that every status is represented for demonstration and testing of listing/filtering.

**Detailed description**:
`ordersPlan` (`prisma/seed.ts:226-253`) is an array of `{ status, itemIdxs, quantities }`. Distribution: PENDING 4, PAID 5, PROCESSING 4, SHIPPED 4, DELIVERED 6, CANCELLED 3. Item indexes refer to positions in `productsSeed` (valid range 0-19; the maximum used is 19), and `quantities` is parallel to `itemIdxs`. The non-null assertions (`plan.quantities[j]!`, `products[idx]!`) mean a mismatch in array lengths or an out-of-range index would throw at runtime or insert `undefined`-derived data, with no validation.

Each order is assigned a customer by `customers[i % customers.length]` (round-robin across 10 customers, so orders 0-9 map to customers 0-9 and 10-19, 20-25 wrap) and an author by `i % 3 === 0 ? admin : operator` (9 orders by admin: indexes 0,3,6,...,24; 17 by operator) (`prisma/seed.ts:266-267`). Orders are created sequentially in a `for` loop, which keeps order numbers aligned with plan position (plan index i -> `ORD-` number i+1).

Orders are inserted directly in their final status; the script does not invoke the application state machine. The final status is simply written to `order.status`.

**Rule workflow**:
```
for i in 0..25: select customer (i mod 10), author (admin if i mod 3 == 0) -> build items -> compute totals
 -> reserve order number -> create order (status = plan.status) with nested items -> create history rows
```

---

### Business Rule: Price Snapshot, Totals and Discount

**Overview**:
Order items capture the product price at creation, and some orders receive a 5% discount.

**Detailed description**:
For each plan item, `unitPriceCents` is copied from `product.priceCents`, and `totalCents = unitPriceCents * quantity` (`prisma/seed.ts:269-279`). This mirrors the snapshot-pricing approach for order items in the schema (`OrderItem.unitPriceCents`). The order `subtotalCents` is the sum of line totals (`prisma/seed.ts:281`).

A discount is applied when the plan index satisfies `i % 7 === 0` (indexes 0, 7, 14, 21, i.e. four orders): `discountCents = Math.floor(subtotalCents * 0.05)`, otherwise 0 (`prisma/seed.ts:282`). `totalCents = subtotalCents - discountCents` (`prisma/seed.ts:283`). The discount is therefore tied to loop position, not to any customer, product or business attribute; it exists purely to produce a mix of discounted and non-discounted orders. The 5% rate is a magic number.

The application layer enforces that a discount cannot exceed the subtotal (`src/modules/orders/order.service.ts:87-92`); the seed always satisfies this (5% < 100%) but does not call that validation. The floor operation guarantees integer cents, in line with the Int schema columns.

**Rule workflow**:
```
unit = product.priceCents -> lineTotal = unit * qty -> subtotal = sum(lineTotal)
 -> discount = (i mod 7 == 0) ? floor(subtotal * 0.05) : 0 -> total = subtotal - discount
```

---

### Business Rule: Sequential Order Number Generation

**Overview**:
Order numbers have the format `ORD-` plus a zero-padded 6-digit counter, taken from a single-row table via an atomic upsert.

**Detailed description**:
`reserveOrderNumber()` (`prisma/seed.ts:165-176`) runs in `prisma.$transaction`. It upserts `OrderNumberSequence` with `id: 1`: if absent, it creates `{ nextValue: 2 }`, otherwise increments `nextValue` by 1, and selects the stored `nextValue`. The reserved number is `nextValue - 1`, formatted with `padStart(6, '0')`. Starting from an empty table (guaranteed by the cleanup rule), the first call returns `ORD-000001` and the 26th `ORD-000026`.

This is a verbatim duplicate of the private method `reserveOrderNumber(tx)` in `src/modules/orders/order.service.ts:245-254`; the seed version opens its own transaction and uses the global client, while the service version receives a transaction client from the caller so number reservation shares a transaction with order creation. In the seed, the number reservation is committed independently from the order insert, so a failure between them would leave a gap in the sequence (not relevant for demo data, as the table is cleared on each run).

The test `tests/orders.test.ts:39` asserts the format `^ORD-\d{6}$` for API-created orders, which is the contract the seed follows.

**Rule workflow**:
```
BEGIN -> upsert sequence(id=1; create nextValue=2 | increment nextValue) -> read nextValue
 -> current = nextValue - 1 -> "ORD-" + pad6(current) -> COMMIT
```

---

### Business Rule: Synthetic Status History Trail

**Overview**:
For each order the script writes a history chain from creation up to the final status, authored by the order's author.

**Detailed description**:
`flowFromStatus` (`prisma/seed.ts:255-262`) maps each final status to the list of predecessor statuses: PENDING `[]`; PAID `[PENDING]`; PROCESSING `[PENDING, PAID]`; SHIPPED `[PENDING, PAID, PROCESSING]`; DELIVERED `[PENDING, PAID, PROCESSING, SHIPPED]`; CANCELLED `[PENDING]`. The path is `[...flowFromStatus[status], status]` (`prisma/seed.ts:307`). One `orderStatusHistory` row is created per step, with `fromStatus` null for the first row (`reason` null) and, for later rows, `reason = "seed transition <prev>-><next>"` (`prisma/seed.ts:309-320`). All rows use `changedById = author.id` and rely on the schema default `changedAt = now()`, so timestamps are creation-time and nearly identical, not historically spaced.

The paths are consistent with the permitted transitions in `src/modules/orders/order.status.ts:3-10` (PENDING->PAID/CANCELLED, PAID->PROCESSING/CANCELLED, PROCESSING->SHIPPED/CANCELLED, SHIPPED->DELIVERED). Seeded CANCELLED orders always show a single transition from PENDING, even though the state machine would also allow cancellation from PAID or PROCESSING. The table is hand-maintained, not derived from `order.status.ts`, so drift is possible if the state machine changes.

History rows are inserted one at a time with `await` (sequentially), producing up to 5 inserts for DELIVERED orders. Total history rows: PENDING 4x1, PAID 5x2, PROCESSING 4x3, SHIPPED 4x4, DELIVERED 6x5, CANCELLED 3x2 = 4+10+12+16+30+6 = 78 rows.

**Rule workflow**:
```
path = predecessors[status] + status ; prev = null
for s in path: create history(orderId, fromStatus=prev, toStatus=s, changedBy=author, reason=prev?"seed transition prev->s":null); prev = s
```

---

### Business Rule: Stock Is Not Affected by Seeded Orders

**Overview**:
Seeded orders in PAID or later states do not decrement product stock.

**Detailed description**:
In the application, stock is debited on the PENDING->PAID transition and replenished when a PAID/PROCESSING order is cancelled (`src/modules/orders/order.status.ts:24-34`; stock operations at `src/modules/orders/order.service.ts:214-240`). The seed writes orders directly with the final status and never touches `stockQuantity`. Consequently, the seeded database is not consistent with the application rules: for example, 19 orders in PAID/PROCESSING/SHIPPED/DELIVERED states exist while all products keep their initial stock. (This is documented as an observation; no seed comment states whether this is intentional.)

Confidence: high that stock is not adjusted (no code path does so); unknown whether it is an intentional simplification.

**Rule workflow**:
```
seed order insert (any status) -> product.stockQuantity unchanged
```

---

### Business Rule: Failure Handling and Resource Cleanup

**Overview**:
The script fails fast with a non-zero exit code and always releases the DB connection.

**Detailed description**:
`main()` is invoked with `.catch` logging the error via `console.error` and calling `process.exit(1)`, followed by `.finally` calling `prisma.$disconnect()` (`prisma/seed.ts:327-335`). Note that `process.exit(1)` inside `.catch` terminates the process before `.finally` executes, so the disconnect is effectively skipped on failure (the process exit closes the connection anyway). On success, `.finally` disconnects normally.

There is no retry, no rollback of partial data, and no wrapping transaction; a failure leaves a partially seeded database, which is repaired by simply re-running the script (cleanup rule). ESLint `no-console` is disabled per-line for each log call.

**Rule workflow**:
```
main() -> success: log "Seed completed." -> $disconnect
       -> error: console.error -> exit(1)
```

---

## 4. Component Structure

```
prisma/
├── schema.prisma          # Data model consumed by seed (PrismaClient types, enums UserRole/OrderStatus)
├── seed.ts                # Seed Script (single-file component, 335 lines)
│   ├── constants          # ADMIN_PASSWORD, OPERATOR_PASSWORD (6-7)
│   ├── customersSeed      # 10 customer literals (9-140)
│   ├── productsSeed       # 20 product literals (142-163)
│   ├── reserveOrderNumber # sequence helper (165-176)
│   ├── main               # orchestration (178-325)
│   └── bootstrap          # main().catch().finally() (327-335)
└── migrations/
    ├── 20260519182739_init   # schema creation
    └── migration_lock.toml
```

Related wiring: `package.json:17` (`db:seed` script), `package.json:24-26` (`prisma.seed` hook), `package.json:18` (`db:reset` script, `prisma migrate reset --force`, which triggers the seed hook).

## 5. Dependency Analysis

```
Internal Dependencies (compile-time):
seed.ts -> @prisma/client generated types (PrismaClient, OrderStatus, UserRole)
seed.ts -> prisma/schema.prisma (indirectly, models: user, customer, product, order, orderItem,
           orderStatusHistory, orderNumberSequence)
seed.ts does NOT import anything from src/ (logic duplicated, see rules)

Logical (non-import) couplings:
seed.ts reserveOrderNumber  <-> src/modules/orders/order.service.ts:245 (duplicated)
seed.ts flowFromStatus      <-> src/modules/orders/order.status.ts:3 (hand-mirrored)
seed.ts bcrypt cost 10      <-> src/modules/users/user.service.ts:16
seed.ts credentials         <-> src/modules/auth/auth.service.ts (login)

External Dependencies:
- @prisma/client 5.22.0 - ORM client
- bcrypt 5.1.1 - password hashing
- tsx 4.19.2 (devDependency) - TypeScript runner for the script
- prisma 5.22.0 (devDependency) - CLI that triggers the seed hook
- MySQL 8.0 (docker-compose.yml, service oms-mysql) - persistence
- Node >= 20 with --env-file (package.json engines/script)
- Environment: DATABASE_URL (via .env; not read directly by the script)
```

Note: `bcrypt` and `tsx` placement: `bcrypt` is a production dependency, `tsx` is a devDependency; the script is dev tooling.

## 6. Afferent and Efferent Coupling

Components in this context are the module-level units (functions/constants) of a procedural TypeScript script.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| main() | 1 (bootstrap chain) | 6 (PrismaClient, bcrypt, customersSeed, productsSeed, reserveOrderNumber, flowFromStatus/ordersPlan) | High |
| reserveOrderNumber() | 1 (main) | 1 (PrismaClient, orderNumberSequence) | Medium |
| prisma (PrismaClient instance) | 3 (main, reserveOrderNumber, bootstrap .finally) | 1 (@prisma/client) | High |
| customersSeed | 1 (main) | 0 | Low |
| productsSeed | 1 (main) | 0 | Low |
| ordersPlan / flowFromStatus (local to main) | 1 (main loop) | 1 (OrderStatus enum) | Medium |
| ADMIN_PASSWORD / OPERATOR_PASSWORD | 1 (main) | 0 | Medium (credentials) |
| Seed Script as a whole | 0 imports from other project files; invoked by npm scripts and Prisma CLI | 2 external packages + Prisma client | Medium |

## 7. Endpoints

Not applicable: the component exposes no endpoints. (Section intentionally omitted per guidelines; this is a command-line script.)

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (oms-mysql) | Database | Wipe and populate domain tables | MySQL wire protocol via Prisma engine | Prisma models / SQL | catch -> console.error -> exit(1); no retry |
| Prisma CLI (`migrate reset`) | Tooling | Auto-run seed after reset (`prisma.seed`) | Process spawn | n/a | Exit code propagated |
| npm script `db:seed` | Tooling | Manual invocation with `.env` | Process spawn | n/a | Exit code 1 on failure |
| .env file | Configuration | Supplies DATABASE_URL | Node `--env-file` | KEY=VALUE | Missing variable surfaces as Prisma error |
| bcrypt | Library | Password hashing for seeded users | In-process | Hash string (VarChar 255) | Unhandled rejection reaches the main catch |
| OMS API auth (`src/modules/auth`) | Internal consumer (indirect) | Seeded users log in via `/auth` flow | Via database | bcrypt hashes | n/a |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Script / Transaction Script | Linear procedural `main()` | prisma/seed.ts:178-325 | One-shot data load |
| Data-driven fixtures (static fixtures) | `customersSeed`, `productsSeed`, `ordersPlan` arrays | prisma/seed.ts:9-163, 226-253 | Deterministic seed data |
| Counter/sequence table | `OrderNumberSequence` upsert in a transaction | prisma/seed.ts:165-176 | Sequential order numbers |
| Lookup table | `flowFromStatus` Record | prisma/seed.ts:255-262 | Derive status history path |
| Clear-then-load (reset) | deleteMany cascade of tables | prisma/seed.ts:181-187 | Repeatable end state |
| Fail-fast bootstrap | `.catch(exit 1).finally(disconnect)` | prisma/seed.ts:327-335 | Process lifecycle |
| Module-level singleton | `const prisma = new PrismaClient()` (separate from `src/config/database.ts`) | prisma/seed.ts:4 | DB access |

Architectural decision: the script is deliberately decoupled from `src/` (no imports), accessing data directly through Prisma instead of through services, which avoids side effects such as stock movement and state-machine validation. Whether this decoupling was a conscious choice is not documented.

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Data reset | Unconditional `deleteMany` on all tables with no environment/host guard or confirmation | Total data loss if run against a non-disposable database |
| High | Credentials | Hardcoded known passwords (`admin123`, `operator123`) for an ADMIN account, with no environment restriction | Known-credential admin account if seeded outside local development |
| Medium | Logic duplication | Order number generation, bcrypt cost, and status path table duplicated from `src/` | Drift between seed and application behavior |
| Medium | Data consistency | Seeded orders in PAID+ states do not debit stock; history written directly | Seed data violates application invariants (stock vs. orders) |
| Medium | Atomicity | Cleanup and inserts are not in a transaction; order number reservation separate from order insert | Partially seeded database on failure; sequence gaps |
| Medium | Test isolation | `tests/setup.ts` wipes the same tables | Seeded data destroyed when tests run against the same DB |
| Low | Error handling | `process.exit(1)` in `.catch` precludes `.finally` disconnect | Connection released only by process termination |
| Low | Performance | Order loop is fully sequential (about 26 transactions, 26 order creates, 78 history inserts, plus bcrypt) | Slow only marginally; negligible at this scale |
| Low | Robustness | Non-null assertions on array indexes without validation | Runtime errors on a malformed plan |
| Low | Data quality | Placeholder CPF/CNPJ values not validated; typo "Pretra"; parallel creates give no timestamp order | Cosmetic / demo-only inaccuracies |
| Low | Temporal data | All history `changedAt` timestamps equal creation time | Unrealistic audit timelines |
| Low | Documentation | No README/docs reference found for the seed credentials in the files searched beyond package.json | Discoverability |

## 11. Test Coverage Analysis

Test files located in the project: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`; configuration `vitest.config.ts` (includes `tests/**/*.test.ts`, single fork, sequential files).

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| Seed Script (prisma/seed.ts) | 0 | 0 | 0% (no tests reference the seed; no coverage tool configured) | No automated verification of counts, totals, order numbers, history paths or credentials |
| reserveOrderNumber (seed copy) | 0 | 0 (the service copy is indirectly checked by `tests/orders.test.ts:39` format assertion) | 0% for the seed copy | Format contract only tested via API |
| Related: tests/helpers/factories.ts | n/a | Test-side fixture helpers (bcrypt cost 10 at line 21) independent from the seed | n/a | Separate fixture mechanism; duplicates seed responsibilities for tests |

Observations: `tests/setup.ts` clears all tables before each test, so no test relies on seeded data. The seed is only exercisable manually. This is flagged as a risk (untested component).
