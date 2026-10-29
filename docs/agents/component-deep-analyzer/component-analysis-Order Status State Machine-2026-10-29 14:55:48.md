# Component Deep Analysis Report: Order Status State Machine

Analyzed file: `src/modules/orders/order.status.ts` (37 lines). Consumers and tests were analyzed for context: `src/modules/orders/order.service.ts`, `src/modules/orders/order.schemas.ts`, `src/modules/orders/order.routes.ts`, `src/shared/errors/http-errors.ts`, `prisma/schema.prisma`, `tests/orders.test.ts`. Ignored folders: none.

## 1. Executive Summary

The Order Status State Machine is a stateless, pure-function module that defines the order lifecycle over the Prisma enum `OrderStatus` (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED). It owns three things:

1. The allowed-transition table (`src/modules/orders/order.status.ts:3-10`).
2. Query helpers: `canTransition`, `allowedTransitions`, `isTerminal` (lines 12-22).
3. Stock side-effect predicates, `shouldDebitStock` and `shouldReplenishStock`, with the constant `STOCK_DEBIT_TRANSITION` (lines 24-37).

The module performs no I/O. Its only consumer is `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), which enforces the rules inside a Prisma transaction.

Key findings:
- The transition graph is a linear happy path (PENDING -> PAID -> PROCESSING -> SHIPPED -> DELIVERED) with a cancellation branch from PENDING, PAID and PROCESSING. DELIVERED and CANCELLED are terminal.
- Stock is debited on PENDING -> PAID and replenished on PAID/PROCESSING -> CANCELLED. Cancelling from PENDING does not touch stock.
- `allowedTransitions` and `isTerminal` are exported but have no consumers in `src/` (grep result).
- The state machine itself has no unit tests. Coverage is indirect, through HTTP integration tests that cover only 3 of the 8 legal transitions and 1 illegal one.
- The delete rule (PENDING or CANCELLED only) is implemented separately in `order.service.ts:184`, not in this module.

## 2. Data Flow Analysis

```
1. PATCH /api/v1/orders/:id/status enters via order.routes.ts:19-23 (authenticate applied at router level, line 14)
2. validate() checks params (uuid) and body with updateOrderStatusSchema (toStatus via z.nativeEnum(OrderStatus), reason max 500)
3. OrderController.changeStatus calls OrderService.changeStatus(id, input, userId)
4. A prisma.$transaction opens (order.service.ts:131)
5. The order and its items are loaded; missing order -> NotFoundError (line 136)
6. from = order.status, to = input.toStatus
7. from === to -> ConflictError code INVALID_STATUS_TRANSITION (lines 140-146)
8. canTransition(from, to) false -> InvalidStatusTransitionError (409) (lines 147-149)
9. shouldDebitStock(from, to) -> debitStock: availability check for all items, then decrement (lines 151-153, 204-231); shortage -> InsufficientStockError (422)
10. shouldReplenishStock(from, to) -> replenishStock: increment per item (lines 154-156, 233-243)
11. order.status is updated (line 158)
12. A row is appended to orderStatusHistory (fromStatus, toStatus, changedById, reason) (lines 159-167)
13. The order is re-read with items, history and customer and returned (lines 169-177)
14. Any thrown error rolls back the whole transaction
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| State transition | Fixed transition table; only listed transitions are legal | src/modules/orders/order.status.ts:3-10, 12-14 |
| State transition | PENDING may go to PAID or CANCELLED | order.status.ts:4 |
| State transition | PAID may go to PROCESSING or CANCELLED | order.status.ts:5 |
| State transition | PROCESSING may go to SHIPPED or CANCELLED | order.status.ts:6 |
| State transition | SHIPPED may only go to DELIVERED (no cancellation) | order.status.ts:7 |
| State transition | DELIVERED and CANCELLED are terminal | order.status.ts:8-9, 20-22 |
| Business Logic | Stock is debited exactly on PENDING -> PAID | order.status.ts:24-31 |
| Business Logic | Stock is replenished on PAID/PROCESSING -> CANCELLED | order.status.ts:33-37 |
| Validation | Self-transition (from === to) is rejected as a conflict | order.service.ts:140-146 |
| Validation | All items must have sufficient stock when debiting; all-or-nothing | order.service.ts:204-224 |
| Audit | Every transition writes a history row with actor and optional reason | order.service.ts:159-167 |
| Atomicity | Status change, stock change and history are in one DB transaction | order.service.ts:131 |
| Validation | Target status must be a valid enum value; reason max 500 chars | order.schemas.ts:18-21 |
| Related (outside module) | Deletion allowed only in PENDING or CANCELLED | order.service.ts:184-190 |

### Detailed breakdown of the business rules

---

### Business Rule: Allowed Transition Graph

**Overview**:
An order's status can only move along edges of a fixed directed graph defined in the `transitions` constant. Any other change is refused with HTTP 409 `INVALID_STATUS_TRANSITION`.

**Detailed description**:
The table at `order.status.ts:3-10` is typed `Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>>`. Because it is a Record over the Prisma enum, TypeScript requires every enum member to have an entry, so adding a status to `prisma/schema.prisma:16-23` without updating the table is a compile-time error. The edges are: PENDING -> {PAID, CANCELLED}; PAID -> {PROCESSING, CANCELLED}; PROCESSING -> {SHIPPED, CANCELLED}; SHIPPED -> {DELIVERED}; DELIVERED -> {}; CANCELLED -> {}.

`canTransition(from, to)` (line 12) is a membership test on that table. It has no knowledge of actors, roles or reasons, so any authenticated user can request any legal transition; the router only applies `authenticate` (`order.routes.ts:14`) and no role check was found in the files analyzed. Backward moves (for example PAID -> PENDING), skips (PENDING -> SHIPPED, which the integration test exercises) and reopening of terminal orders are all illegal by omission.

Once an order is SHIPPED it can no longer be cancelled; the only exit is DELIVERED. Returns, refunds and reversal of delivery are not modeled. Whether the omission of cancellation after shipping is intentional is not documented in the source (assumption: it is a deliberate design choice, medium confidence).

**Rule workflow**:
```
changeStatus -> canTransition(from, to)
  true  -> continue to stock rules and persistence
  false -> InvalidStatusTransitionError(from, to) -> 409 INVALID_STATUS_TRANSITION, details {from, to}; transaction rolled back
```

---

### Business Rule: Terminal States

**Overview**:
DELIVERED and CANCELLED have empty transition lists and therefore are final.

**Detailed description**:
`isTerminal(status)` (lines 20-22) derives terminality from the table (`transitions[status].length === 0`) instead of a hard-coded list, so it follows table changes automatically. The function is exported but not called anywhere in `src/` or `tests/` according to a repository grep. Terminality is therefore enforced only implicitly, via `canTransition` returning false for every target.

For terminal orders, the only mutating operation that remains is deletion: a CANCELLED order can be deleted (`order.service.ts:184`), a DELIVERED order cannot. This means CANCELLED orders can be removed along with their audit trail (cascade behavior of history on delete was not verified in the schema excerpt reviewed; ambiguity noted).

**Rule workflow**:
```
status in {DELIVERED, CANCELLED} -> transitions[status] = [] -> any target fails canTransition -> 409
```

---

### Business Rule: Stock Debit on Payment

**Overview**:
Stock is subtracted from products only when an order moves from PENDING to PAID.

**Detailed description**:
`STOCK_DEBIT_TRANSITION` (lines 24-27) names the single trigger edge, and `shouldDebitStock` (lines 29-31) compares both endpoints against it. Order creation does not reserve or debit stock (the test at `tests/orders.test.ts:12-40` creates orders for quantities and a test at lines 109-132 creates an order for 5 units when only 1 is in stock, receiving 201). Stock availability is therefore validated at payment time, not at order time.

When triggered, `debitStock` (`order.service.ts:204-231`) loads all products for the items, builds a list of every item whose product is missing or whose `stockQuantity` is less than the requested quantity, and throws `InsufficientStockError` (422, code `INSUFFICIENT_STOCK`, details `unavailable[{sku, requested, available}]`) if the list is non-empty. Only if all items pass does it decrement each product. Because it runs inside the transaction, a failure leaves stock and order status unchanged (verified at `tests/orders.test.ts:130-131`).

The availability check and decrement are separate statements (read, then `decrement`) within a transaction. Whether the DB isolation level prevents concurrent oversell was not determined from the analyzed files; the decrement is not guarded by a conditional WHERE (see risks).

**Rule workflow**:
```
from == PENDING && to == PAID
  -> read products -> any shortage? -> yes: 422 INSUFFICIENT_STOCK, rollback
                                    -> no: decrement each item quantity
  -> update status, write history
```

---

### Business Rule: Stock Replenishment on Cancellation

**Overview**:
Cancelling an order that has already been paid (PAID or PROCESSING) returns its item quantities to stock.

**Detailed description**:
`shouldReplenishStock` (lines 33-37) returns true when the target is CANCELLED and the source is PAID or PROCESSING. These are exactly the states reachable after a debit, so the debit and replenishment rules are symmetric for cancellations. Cancelling from PENDING does not replenish because stock was never debited.

The check is expressed with explicit status comparisons and is independent of the transition table; it is not derived from `STOCK_DEBIT_TRANSITION`. If the transition table gained a new cancellable post-payment state, this predicate would need separate manual updating (coupling note).

`replenishStock` (`order.service.ts:233-243`) increments each product by the item quantity inside the same transaction. The integration test at `tests/orders.test.ts:134-161` verifies stock 10 -> 6 after payment of 4 units and back to 10 after cancellation. Replenishment uses the quantities from `order.items` at time of change; if product records had been deleted in the meantime, the update would throw (not analyzed or tested).

**Rule workflow**:
```
to == CANCELLED && from in {PAID, PROCESSING}
  -> for each item: product.stockQuantity += quantity
  -> update status, write history
from == PENDING && to == CANCELLED -> no stock change
```

---

### Business Rule: No-Op Transition Rejection

**Overview**:
Requesting the status the order already has is rejected with 409 and the same `INVALID_STATUS_TRANSITION` code.

**Detailed description**:
This check lives in the service (`order.service.ts:140-146`), not in the state machine module. It produces a specific message ("Order is already in X status") with details `{from, to}`. Without it, `canTransition` would already return false for every self-transition because no state lists itself, so the check only improves the message; the resulting HTTP outcome (409, same code) is identical.

**Rule workflow**:
```
from === to -> ConflictError(INVALID_STATUS_TRANSITION) -> 409
```

---

### Business Rule: Transition Audit Trail

**Overview**:
Every status change (and order creation) is recorded in `orderStatusHistory`.

**Detailed description**:
On creation, a history row with `fromStatus: null`, `toStatus: PENDING`, `reason: 'order created'` is written (`order.service.ts:106-112`). On every successful transition, a row with `fromStatus`, `toStatus`, `changedById` (authenticated user) and `reason` (optional, max 500 chars, null if absent) is created in the same transaction (lines 159-167). Failed transitions leave no history because the transaction rolls back. The response embeds history ordered by `changedAt` ascending. Tests assert the history sequence ['PENDING', 'PAID'] (`tests/orders.test.ts:80-83`).

**Rule workflow**:
```
successful transition -> tx.order.update -> tx.orderStatusHistory.create -> response with history
```

---

### Business Rule: Transactional Atomicity of Transitions

**Overview**:
Stock effects, status update and history insert succeed or fail together.

**Detailed description**:
`changeStatus` wraps everything in `this.prisma.$transaction` (`order.service.ts:131`). The state machine module's predicates are evaluated inside that transaction against the freshly read order, so the check uses the status as read within the transaction. Any error (invalid transition, insufficient stock, DB error) aborts and rolls back. This is confirmed for the insufficient-stock case by the test at `tests/orders.test.ts:109-132`.

**Rule workflow**:
```
BEGIN -> read order -> validate -> stock effects -> update status -> insert history -> read -> COMMIT (any throw -> ROLLBACK)
```

---

### Business Rule: Status Input Validation

**Overview**:
The request body must contain `toStatus` as a valid enum member, with an optional reason of at most 500 characters.

**Detailed description**:
`updateOrderStatusSchema` (`order.schemas.ts:18-21`) uses `z.nativeEnum(OrderStatus)`, tying API validation to the same Prisma enum the state machine uses. Unknown values are rejected by the validation middleware before reaching the service. The order id must be a UUID (`orderIdParamSchema`). The list filter `status` uses the same enum (line 26).

**Rule workflow**:
```
request -> validate(params, body) -> invalid: validation error response; valid: controller -> service
```

---

### Business Rule: Deletion Restricted by Status (related, outside the module)

**Overview**:
Orders can be deleted only while PENDING or CANCELLED; otherwise 409 `INVALID_ORDER_STATE_FOR_DELETE`.

**Detailed description**:
Implemented in `order.service.ts:181-192` with hard-coded status comparisons, not through `order.status.ts`. It is conceptually part of the lifecycle policy but is not centralized with it. Test: `tests/orders.test.ts:197-218` (PAID order cannot be deleted). Deleting a PENDING order requires no stock handling since no debit occurred; deleting a CANCELLED order requires none either because replenishment already happened at cancellation.

**Rule workflow**:
```
delete(id) -> not found: 404; status not in {PENDING, CANCELLED}: 409; else deleteById
```

---

### Transition matrix (complete)

| From \ To | PENDING | PAID | PROCESSING | SHIPPED | DELIVERED | CANCELLED |
|-----------|---------|------|------------|---------|-----------|-----------|
| PENDING | no | yes (debit) | no | no | no | yes |
| PAID | no | no | yes | no | no | yes (replenish) |
| PROCESSING | no | no | no | yes | no | yes (replenish) |
| SHIPPED | no | no | no | no | yes | no |
| DELIVERED | no | no | no | no | no | no |
| CANCELLED | no | no | no | no | no | no |

## 4. Component Structure

```
src/modules/orders/
├── order.status.ts       # THE COMPONENT: transition table, predicates, stock-effect rules (37 lines)
├── order.service.ts      # Sole consumer; orchestrates transaction, stock, history
├── order.schemas.ts      # Zod schemas using the OrderStatus enum
├── order.routes.ts       # PATCH /:id/status route
├── order.controller.ts   # HTTP handlers
└── order.repository.ts   # Data access

prisma/schema.prisma:16-23           # OrderStatus enum (source of states)
src/shared/errors/http-errors.ts     # InvalidStatusTransitionError (45-53), InsufficientStockError (55-63)
tests/orders.test.ts                 # Integration tests (indirect coverage)
```

Exports of `order.status.ts`: `canTransition`, `allowedTransitions`, `isTerminal`, `STOCK_DEBIT_TRANSITION`, `shouldDebitStock`, `shouldReplenishStock`. Module-private: `transitions`.

## 5. Dependency Analysis

Internal Dependencies:
```
order.routes.ts -> order.controller.ts -> order.service.ts -> order.status.ts
order.service.ts -> order.repository.ts, http-errors.ts, Prisma tx client
order.status.ts -> (no internal imports)
```

External Dependencies:
- `@prisma/client` (runtime import of the `OrderStatus` enum object, `order.status.ts:1`). Version is not stated in the analyzed files.
- Zod (schemas only, not used by the module itself).
- MySQL or compatible DB via Prisma (the `@db.Char(36)` annotation suggests MySQL; not verified).

The module is pure: no I/O, no configuration, no environment variables.

## 6. Afferent and Efferent Coupling

Counts are based on source grep of `src/` and `tests/`.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| transitions (private table) | 3 (canTransition, allowedTransitions, isTerminal) | 1 (OrderStatus enum) | High |
| canTransition | 1 (OrderService) | 1 | High |
| shouldDebitStock | 1 (OrderService) | 1 (STOCK_DEBIT_TRANSITION) | High |
| shouldReplenishStock | 1 (OrderService) | 1 (OrderStatus) | High |
| STOCK_DEBIT_TRANSITION | 1 (internal) | 1 | Medium |
| allowedTransitions | 0 | 1 | Low |
| isTerminal | 0 | 1 | Low |
| order.status.ts module | 1 (order.service.ts) | 1 (@prisma/client) | High |

## 7. Endpoints

The module exposes none directly. It governs one endpoint of the orders module:

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/orders/{id}/status | PATCH | Request an order status transition; body `{toStatus, reason?}`; responses: 200 (updated order), 404, 409 INVALID_STATUS_TRANSITION, 422 INSUFFICIENT_STOCK, validation error |

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Prisma enum `OrderStatus` | Library / generated code | Source of state values | In-process import | TS enum object | Compile-time exhaustiveness via Record type |
| OrderService | Internal | Sole caller of predicates | In-process function calls | Enum values, booleans | Service throws typed HTTP errors |
| Database (products, orders, history) | Internal DB via service | Stock and status persistence | Prisma / SQL transaction | Relational | Transaction rollback on any error |

The module itself has no external integrations.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Finite State Machine (table-driven) | `transitions` Record | order.status.ts:3-10 | Declarative legal-transition definition |
| Pure functions / stateless module | Exported predicate functions | order.status.ts:12-37 | Deterministic, side-effect-free rules |
| Separation of policy from mechanism | Module decides, service performs | order.status.ts vs order.service.ts:151-156 | Keeps DB logic out of the rule module |
| Immutability | `Readonly`, `ReadonlyArray`, `as const` | order.status.ts:3, 24-27 | Prevent runtime mutation of rules |
| Unit of Work / Transaction Script | `$transaction` in changeStatus | order.service.ts:131 | Atomic multi-write change |
| Audit log / event history | `orderStatusHistory` | order.service.ts:159-167 | Traceability |
| Typed domain errors | InvalidStatusTransitionError, InsufficientStockError | src/shared/errors/http-errors.ts:45-63 | Map rule failures to HTTP codes |

Architecture note: the module is a domain-policy layer inside a modular layered architecture (routes -> controller -> service -> repository).

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Test coverage | No unit tests for order.status.ts; integration tests cover only a subset of transitions | Regressions in the transition table or predicates may go undetected |
| High | Stock debit (order.service.ts:204-231) | Availability check and decrement are separate statements with no conditional update visible | Possible oversell under concurrent payments depending on DB isolation (not verified) |
| Medium | Stock rules coupling | `shouldReplenishStock` hard-codes source states independently of the transition table and `STOCK_DEBIT_TRANSITION` | Table and stock rules can drift out of sync |
| Medium | Authorization | No role/permission check on transitions found in analyzed files; any authenticated user can perform any legal transition | Potential privilege issue (confidence: medium, other middleware not exhaustively reviewed) |
| Medium | Lifecycle policy fragmentation | Delete rule (order.service.ts:184) uses inline status comparisons outside the module | Inconsistent centralization of status policy |
| Medium | Data integrity | CANCELLED orders are deletable; effect on history rows depends on schema cascade (not verified) | Possible loss of audit trail |
| Low | Dead code | `allowedTransitions` and `isTerminal` have no callers | Unused API surface |
| Low | Redundant check | Self-transition check at order.service.ts:140 duplicates behavior of canTransition (message differs only) | Minor duplication |
| Low | Missing modeled flows | No return, refund or post-shipment cancellation | Business gap (intent undocumented) |
| Low | Stock at creation | Orders can be created for quantities exceeding stock; failure surfaces only at payment | Late failure; behavior confirmed by test at tests/orders.test.ts:109-119 |
| Low | Lookup | `transitions[from]` has no guard for values outside the enum | Runtime error only if DB contains invalid value |

## 11. Test Coverage Analysis

Test files located: `tests/orders.test.ts` (219 lines, Vitest + supertest, integration against a real DB through `getTestApp` and `prisma`); `tests/auth.test.ts` (unrelated). No unit test file exists for `order.status.ts`. No coverage report was found or executed, so percentages are not available.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| order.status.ts (overall) | 0 | 5 relevant (indirect) | Not measured | Assertions good on outcomes; no direct unit tests |
| Transition table | 0 | 4 legal/illegal cases exercised | 3 of 8 legal edges (PENDING->PAID, PAID->CANCELLED, plus PENDING->PAID in other tests); 1 illegal edge (PENDING->SHIPPED) | Missing: PAID->PROCESSING, PROCESSING->SHIPPED, SHIPPED->DELIVERED, PROCESSING->CANCELLED, PENDING->CANCELLED; terminal-state rejections; self-transition; backward moves |
| canTransition | 0 | tests/orders.test.ts:89-107 | Indirect | Only one negative case |
| shouldDebitStock | 0 | tests/orders.test.ts:59-87, 109-132 | Indirect | Success and shortage covered; multi-item and missing-product cases not covered |
| shouldReplenishStock | 0 | tests/orders.test.ts:134-161 | Indirect | PAID->CANCELLED covered; PROCESSING->CANCELLED and PENDING->CANCELLED (no replenish) not covered |
| allowedTransitions / isTerminal | 0 | 0 | 0% | Untested and unused |
| History audit | 0 | tests/orders.test.ts:37-38, 80-83 | Partial | Reason and changedById not asserted |
| Delete restriction | 0 | tests/orders.test.ts:197-218 | Partial | PAID case only; PENDING and CANCELLED deletion not tested |
| Concurrency | 0 | 0 | 0% | No concurrent payment test |
