# Component Deep Analysis Report: Customers Module

Analysis date: 2026-10-29 14:55:48
Scope: `src/modules/customers/` (CustomerController, CustomerService, CustomerRepository, plus routes and schemas files in the same folder) and its direct collaborators.
Ignored folders: none.

---

## 1. Executive Summary

The Customers Module is the CRUD module for customer records in the "Order Management API" (Node >= 20, TypeScript, Express 4.21.1, Prisma 5.22.0 on MySQL, Zod 3.23.8). It follows a four-layer structure shared by the other modules (routes, controller, service, repository). All endpoints are mounted under `/customers` (full path `/api/v1/customers`, as seen in the test URLs) and sit behind JWT authentication.

Key findings:

- Five REST endpoints (list with search and pagination, get by id, create, partial update, delete). Every request is authenticated, and no role check is applied. ADMIN and OPERATOR have identical rights on customers (src/modules/customers/customer.routes.ts:14).
- The only domain rules enforced in code are: email uniqueness (pre-check in the service plus a DB unique index), existence checks before get, update and delete, and field-level format and length validation in Zod.
- The `document` field (CPF/CNPJ style, seed uses `123.456.789-00`) has only a length check (11 to 20 chars). It has no checksum or format validation and no uniqueness constraint, only a non-unique index (prisma/schema.prisma:52).
- Deleting a customer that has orders is blocked at the database level (`ON DELETE RESTRICT`, prisma/migrations/20260519182739_init/migration.sql:110). The module has no explicit handling for this. The error middleware only maps Prisma codes P2002 and P2025 (src/middlewares/error.middleware.ts:38-53), so a Prisma P2003 error would fall through to a generic 500. This is an inference from code reading. It was not executed.
- There are no automated tests for the customers module. Customers are only exercised indirectly as fixtures in `tests/orders.test.ts`.

---

## 2. Data Flow Analysis

### List (GET /api/v1/customers)
```
1. Request enters Express router /customers (src/routes/index.ts:26)
2. authenticate middleware verifies the Bearer JWT, sets req.user (src/middlewares/auth.middleware.ts:27)
3. validate({ query: listCustomersQuerySchema }) coerces page/pageSize, trims search (customer.schemas.ts:25-29)
   - parsed values are merged back into req.query via Object.assign (validate.middleware.ts:19)
4. CustomerController.list casts req.query to ListCustomersQuery (customer.controller.ts:10)
5. CustomerService.list computes skip = (page-1)*pageSize, take = pageSize (customer.service.ts:15-19)
6. CustomerRepository.list builds the OR "contains" filter and runs findMany + count in one $transaction (customer.repository.ts:24-36)
7. Service wraps result with paginated() -> { data, pagination } (response.ts:22)
8. Controller returns 200 JSON
```

### Get by id (GET /:id)
```
1. authenticate -> validate({ params: uuid }) -> CustomerController.getById
2. CustomerService.getById -> CustomerRepository.findById (findUnique)
3. null -> NotFoundError('Customer') -> errorMiddleware -> 404 NOT_FOUND
4. Otherwise 200 with the customer row
```

### Create (POST /)
```
1. authenticate -> validate({ body: createCustomerSchema }) (req.body replaced by parsed body; unknown keys stripped)
2. CustomerController.create -> CustomerService.create
3. CustomerRepository.findByEmail; if found -> ConflictError 409 EMAIL_ALREADY_USED
4. CustomerRepository.create (explicit field whitelist) -> Prisma insert
5. Controller returns 201 with the created row
```

### Update (PATCH /:id)
```
1. authenticate -> validate({ params, body: updateCustomerSchema (partial) })
2. CustomerService.update -> getById (404 if missing)
3. If email present: findByEmail; conflict if owned by a different id -> 409 EMAIL_ALREADY_USED
4. Build patch with only defined fields -> CustomerRepository.update
5. Controller returns 200 with the updated row
```

### Delete (DELETE /:id)
```
1. authenticate -> validate({ params })
2. CustomerService.delete -> getById (404 if missing) -> CustomerRepository.delete
3. Controller returns 204 with no body
4. If orders reference the customer, the DB rejects the delete (FK RESTRICT); see Technical Debt
```

Error path for all flows: each controller method has try/catch that calls `next(err)`; `errorMiddleware` formats `{ error: { code, message, details? } }`.

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authentication | All customer routes require a valid Bearer JWT | customer.routes.ts:14; auth.middleware.ts:27-47 |
| Authorization | No role restriction; any authenticated role (ADMIN, OPERATOR) may perform all operations | customer.routes.ts:14-24 |
| Validation | `id` path param must be a UUID | customer.schemas.ts:3-5 |
| Validation | name 2-150 chars | customer.schemas.ts:16 |
| Validation | email valid format, max 255 | customer.schemas.ts:17 |
| Validation | phone 8-32 chars (no format check) | customer.schemas.ts:18 |
| Validation | document 11-20 chars (no format or checksum) | customer.schemas.ts:19 |
| Validation | address object with street (1-200), number (1-20), city (1-100), state exactly 2 chars, zipCode 5-15 | customer.schemas.ts:7-13 |
| Validation | Update accepts any subset of create fields (all optional) | customer.schemas.ts:23 |
| Validation | List query: page >= 1 (default 1), pageSize 1-100 (default 20), search trimmed and non-empty if present | customer.schemas.ts:25-29 |
| Business Logic | Email must be unique on create | customer.service.ts:30-33 |
| Business Logic | Email must be unique on update unless it belongs to the same customer | customer.service.ts:45-50 |
| Business Logic | Customer must exist for get, update, delete (404) | customer.service.ts:23-27, 44, 61 |
| Business Logic | Update applies only fields explicitly provided | customer.service.ts:51-57 |
| Business Logic | Search matches name, email or document by substring | customer.repository.ts:12-22 |
| Business Logic | Listing sorted by createdAt descending | customer.repository.ts:29 |
| Business Logic | Pagination offset computed as (page-1)*pageSize; totalPages = ceil(total/pageSize) | customer.service.ts:17; response.ts:13-20 |
| Data Integrity | Email is unique at DB level | prisma/schema.prisma:43 |
| Data Integrity | Customers with orders cannot be deleted (FK RESTRICT) | migration.sql:110 |
| Data Integrity | Order creation requires an existing customer (enforced in Orders module) | src/modules/orders/order.service.ts:59-60 |

### Detailed breakdown of the business rules

---

### Business Rule: Authentication required on all customer routes

**Overview**:
The router applies `authenticate` to every route via `router.use`, so no customer data is reachable anonymously.

**Detailed description**:
`buildCustomerRouter` registers `authenticate` before any route handler (customer.routes.ts:14). The middleware reads the `Authorization` header, requires the `Bearer ` prefix (case-insensitive), extracts the token, and verifies it with `jsonwebtoken` using `env.JWT_SECRET` (auth.middleware.ts:27-47). On success the decoded payload is copied to `req.user` as `{ id, email, role }`.

Failure modes each produce an `UnauthorizedError` (HTTP 401, code UNAUTHORIZED): missing or non-bearer header, empty token, or invalid or expired token. The customer layers never read `req.user`, so the authenticated identity is not used for auditing or ownership. The rule only gates access.

Because the check is at router level, adding a route to this router would be protected by default. Validation middleware runs after authentication, so unauthenticated callers never receive validation error details.

**Rule workflow**:
```
Request -> header present and "Bearer"? -> token non-empty? -> jwt.verify ok?
  no at any step -> 401 UNAUTHORIZED
  yes -> req.user set -> next handler
```

---

### Business Rule: No role-based authorization

**Overview**:
Both defined roles can create, update and delete customers. `requireRole` exists but is not used here.

**Detailed description**:
`AuthUser.role` is `'ADMIN' | 'OPERATOR'` (auth.middleware.ts:9) and a `requireRole` helper is available (auth.middleware.ts:49-61). The customer router does not use it (customer.routes.ts). Therefore an OPERATOR can delete a customer the same as an ADMIN.

This is documented as the observed behavior. Whether the absence is intentional is not stated anywhere in the module (ambiguous, confidence: high that the behavior is as described, unknown intent).

**Rule workflow**:
```
Authenticated user (any role) -> any customer operation allowed
```

---

### Business Rule: Request input validation (Zod schemas)

**Overview**:
Every route validates params, query or body with Zod before the controller runs. Invalid input yields 400 VALIDATION_ERROR.

**Detailed description**:
The `validate` middleware (validate.middleware.ts:11-37) parses the relevant request parts. On `ZodError` it creates a `ValidationError` whose details are a list of `{ path, message }` (path is dotted, or `(root)`). The parsed body replaces `req.body`, which also strips unknown keys (Zod default object behavior), so clients cannot inject extra fields. Parsed query values are merged into `req.query`; parsed params replace `req.params`.

Field constraints (customer.schemas.ts): `name` 2-150 chars; `email` valid format and at most 255; `phone` 8-32 chars; `document` 11-20 chars; `address` is a required nested object with `street` 1-200, `number` 1-20 (string), `city` 1-100, `state` exactly 2 characters, `zipCode` 5-15. All fields are required on create. The limits for name, email, phone and document mirror the DB column sizes (VarChar 150, 255, 32, 20; prisma/schema.prisma:42-45). The address is stored as a JSON column with no DB-level structure, so the Zod schema is the sole enforcer of its shape.

The update schema is `createCustomerSchema.partial()` (customer.schemas.ts:23). Note that `.partial()` is shallow: the `address` object, when supplied, must still be complete (all five sub-fields required), so partial address updates are not possible and the address is replaced whole. An empty body `{}` passes validation and results in an update with no fields (Prisma update with empty data, which still touches `updatedAt` behavior per Prisma; not verified).

The `document` and `phone` fields have only length checks. No CPF/CNPJ checksum, digit-only or phone format rule exists in the module (ambiguity: seed and test data use formatted CPF such as `123.456.789-00` and `000.000.000-00`, so formatted strings are accepted and not normalized).

**Rule workflow**:
```
Request -> validate(schemas) -> parse OK -> controller
                              -> ZodError -> ValidationError(400, details[]) -> errorMiddleware
```

---

### Business Rule: Email uniqueness

**Overview**:
Two customers cannot share an email. Enforced in service (friendly 409) and in the database (unique index).

**Detailed description**:
On create, the service calls `findByEmail` and, if a row exists, throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (customer.service.ts:30-33), resulting in HTTP 409 with that code. On update, the check runs only if `input.email` is provided; a hit on a different customer id causes the same conflict, while a hit on the same id (customer resubmitting its own email) is allowed (customer.service.ts:45-50).

The check-then-write sequence is not atomic. Under concurrent requests, both can pass the pre-check, and the second insert fails on the DB unique constraint (`email @unique`, prisma/schema.prisma:43). That failure is a Prisma P2002 error, which `errorMiddleware` maps to 409 with generic code `CONFLICT` and message `Unique constraint violation on: <target>` (error.middleware.ts:38-47). So the client sees the same status but a different error code in the race case.

Email comparison is exact (`findUnique`). Case sensitivity depends on the MySQL column collation, which is not set explicitly in the schema (default behavior not verified here).

**Rule workflow**:
```
create/update(email) -> findByEmail
   found (and different id on update) -> 409 EMAIL_ALREADY_USED
   not found -> write -> (race) DB unique violation -> P2002 -> 409 CONFLICT
```

---

### Business Rule: Existence check before get, update, delete

**Overview**:
Operations on a specific customer first confirm the customer exists, returning 404 otherwise.

**Detailed description**:
`getById` throws `NotFoundError('Customer')` (message "Customer not found", HTTP 404, code NOT_FOUND) when the repository returns null (customer.service.ts:23-27). `update` and `delete` call `getById` first (customer.service.ts:44, 61), so they inherit this behavior and produce a consistent error before any write.

The existence check and the write are two separate queries. If the row is removed between them, Prisma raises P2025, which the error middleware maps to 404 `Resource not found` (error.middleware.ts:48-53). Thus both paths end at 404, with slightly different messages.

IDs are validated as UUIDs before reaching the service, so malformed ids return 400 rather than 404.

**Rule workflow**:
```
id -> uuid valid? no -> 400
   -> findById -> null -> 404 NOT_FOUND
   -> proceed with update/delete
```

---

### Business Rule: Partial update semantics

**Overview**:
PATCH only changes the fields that were sent.

**Detailed description**:
The service builds the Prisma update payload by conditionally spreading each field when `!== undefined` (customer.service.ts:51-57). This acts as an explicit whitelist: id, createdAt and any unknown property cannot be written through this path (Zod also strips unknown keys). `updatedAt` is managed by Prisma `@updatedAt`.

Because the check is `!== undefined` for most fields but the email conflict check uses truthiness (`if (input.email)`, customer.service.ts:45), an empty-string email cannot reach this point (Zod `.email()` rejects it), so there is no practical inconsistency.

Address is replaced as a whole JSON value, not merged.

**Rule workflow**:
```
PATCH body -> for each of name,email,phone,document,address: include if defined -> repository.update
```

---

### Business Rule: Search, ordering and pagination

**Overview**:
Listing supports substring search over name, email and document, newest first, with page-based pagination and a maximum page size of 100.

**Detailed description**:
`listCustomersQuerySchema` coerces `page` (integer >= 1, default 1) and `pageSize` (integer 1-100, default 20) from query strings, and trims `search`, rejecting an empty string after trimming (customer.schemas.ts:25-29). The service converts to `skip` and `take` (customer.service.ts:15-19).

The repository builds `OR` of `contains` on name, email and document when `search` is given; otherwise no filter (customer.repository.ts:12-22). It executes `findMany` (ordered `createdAt desc`) and `count` with the same `where` in a single Prisma `$transaction`, so items and total are consistent (customer.repository.ts:26-34). The response is `{ data, pagination: { page, pageSize, total, totalPages } }`. A page beyond the last returns an empty `data` array with correct totals. Search wildcard characters `%` and `_` are passed to Prisma `contains`, which escapes them in Prisma 5 (not verified here). Case sensitivity depends on MySQL collation.

The sort uses only `createdAt`, with no tiebreaker, so ordering among rows with identical timestamps is not deterministic (observation).

**Rule workflow**:
```
query -> validate/coerce -> skip/take -> repo.list(where OR contains, createdAt desc) -> {items,total} -> paginated()
```

---

### Business Rule: Customer deletion blocked by existing orders

**Overview**:
A customer referenced by any order cannot be deleted. The restriction lives only in the database.

**Detailed description**:
The orders table has FK `orders_customerId_fkey` with `ON DELETE RESTRICT` (migration.sql:110; relation without `onDelete` in prisma/schema.prisma:87 uses the default). `CustomerService.delete` checks only existence and then calls `repository.delete` (customer.service.ts:60-63), with no check for related orders.

When orders exist, MySQL rejects the delete. Prisma reports this as a known request error with code P2003 (foreign key constraint failed). `errorMiddleware` has branches only for P2002 and P2025 (error.middleware.ts:37-54). Other codes reach the final fallback that logs the error and returns 500 INTERNAL_SERVER_ERROR (error.middleware.ts:56-64). Conclusion (inferred from code, not executed): attempting to delete a customer with orders likely yields a 500 instead of a business-meaningful 409. The module contains no soft delete or archive mechanism.

Conversely, deleting a customer without orders succeeds permanently and returns 204.

**Rule workflow**:
```
DELETE -> exists? no -> 404
       -> repository.delete -> no orders -> 204
                             -> orders exist -> FK violation (P2003) -> unmapped -> 500
```

---

### Business Rule: Customer must exist to place an order (cross-module)

**Overview**:
The Orders module depends on customer records; this rule is implemented outside the module but defines the customer's role.

**Detailed description**:
`OrderService.create` looks up the customer inside its transaction using the Prisma client directly (`tx.customer.findUnique`) and throws `NotFoundError('Customer')` when absent (src/modules/orders/order.service.ts:59-60). It does not go through `CustomerRepository` or `CustomerService`.

Orders expose a read-only projection of the customer `{ id, name, email }` in responses (order.repository.ts:57; order.service.ts:118,174). Customers can also be used as an order list filter via `customerId` (order.schemas.ts:27; order.repository.ts:24).

**Rule workflow**:
```
POST /orders(customerId) -> tx.customer.findUnique -> null -> 404 Customer not found
```

---

## 4. Component Structure

```
src/modules/customers/
├── customer.routes.ts       # Express router factory; applies authenticate and validate per route
├── customer.controller.ts   # HTTP handlers (arrow-function properties), status codes, next(err)
├── customer.service.ts      # Business rules: existence, email uniqueness, partial update, pagination mapping
├── customer.repository.ts   # Prisma data access: list/search, findById, findByEmail, create, update, delete
└── customer.schemas.ts      # Zod schemas and inferred input types

Related (outside the module):
prisma/schema.prisma:40-54                         # Customer model (table "customers")
prisma/migrations/20260519182739_init/migration.sql # DDL and orders FK (RESTRICT)
prisma/seed.ts                                     # Seed customers
src/app.ts:34-36                                   # Manual dependency wiring
src/routes/index.ts:26                             # Mount at /customers
src/middlewares/{auth,validate,error}.middleware.ts
src/shared/errors/, src/shared/http/response.ts
tests/helpers/factories.ts:42                      # createTestCustomer fixture
```

Data model (prisma/schema.prisma:40-54): `id` CHAR(36) UUID default, `name` VARCHAR(150), `email` VARCHAR(255) unique, `phone` VARCHAR(32), `document` VARCHAR(20) indexed (non-unique), `address` Json, `createdAt`, `updatedAt`, relation `orders Order[]`.

---

## 5. Dependency Analysis

```
Internal Dependencies:
customer.routes  -> authenticate (middlewares/auth.middleware)
customer.routes  -> validate (middlewares/validate.middleware)
customer.routes  -> customer.schemas
customer.routes  -> CustomerController (type only)
CustomerController -> CustomerService (type only) -> CustomerRepository (type only)
CustomerController -> ListCustomersQuery (type)
CustomerService  -> ConflictError, NotFoundError (shared/errors)
CustomerService  -> paginated, PaginatedResponse (shared/http/response)
CustomerService  -> CreateCustomerInput, UpdateCustomerInput, ListCustomersQuery (schemas)
CustomerRepository -> PrismaClient, Prisma, Customer (@prisma/client)

Wiring: src/app.ts:34-36 constructs Repository(prisma) -> Service(repo) -> Controller(service)
Mount:  src/routes/index.ts:26

External Dependencies:
- express 4.21.1 - HTTP routing (RequestHandler, Router)
- zod 3.23.8 - input validation
- @prisma/client 5.22.0 - ORM, MySQL (inferred from migration DDL syntax)
- jsonwebtoken 9.0.2 - via authenticate middleware
- MySQL - persistence
```

Inbound dependents of the module: `src/app.ts`, `src/routes/index.ts`, `tests/helpers/factories.ts` (Customer type and Prisma model, not module classes). Orders module depends on the Customer table, not on module classes.

---

## 6. Afferent and Efferent Coupling

Counts are at class and function level, within the repository (type-only imports counted).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| CustomerController | 2 (routes, app.ts) | 3 (CustomerService, ListCustomersQuery, express types) | Low |
| CustomerService | 2 (controller, app.ts) | 5 (CustomerRepository, ConflictError, NotFoundError, paginated, schemas types) | Medium |
| CustomerRepository | 2 (service, app.ts) | 2 (PrismaClient, Prisma types) | Medium |
| buildCustomerRouter | 1 (routes/index.ts) | 4 (authenticate, validate, schemas, controller) | Low |
| customer.schemas | 3 (routes, service, controller) | 1 (zod) | Medium |
| Customer table (Prisma model) | 4 (CustomerRepository, OrderService, OrderRepository, tests/seed) | 0 | High |

---

## 7. Endpoints

Base path: `/api/v1/customers` (mount in src/routes/index.ts:26; the `/api/v1` prefix is taken from the test URLs in tests/orders.test.ts:20). All require `Authorization: Bearer <JWT>`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/customers | GET | List customers; query: page (default 1), pageSize (default 20, max 100), search. Returns 200 `{ data, pagination }` |
| /api/v1/customers/{id} | GET | Get a customer by UUID. 200, or 404 |
| /api/v1/customers | POST | Create a customer. 201, or 400 (validation), 409 (email in use) |
| /api/v1/customers/{id} | PATCH | Partially update a customer. 200, or 400, 404, 409 |
| /api/v1/customers/{id} | DELETE | Delete a customer. 204, or 404 (500 likely if orders exist; see rule) |

Error body format: `{ "error": { "code": string, "message": string, "details"?: any } }`. Common codes: UNAUTHORIZED (401), VALIDATION_ERROR (400), NOT_FOUND (404), EMAIL_ALREADY_USED (409), CONFLICT (409).

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL via Prisma | Database | Customer persistence | Prisma client (MySQL wire protocol) | Rows / JSON column for address | P2002 -> 409, P2025 -> 404, others -> 500 via errorMiddleware |
| JWT auth middleware | Internal | Request authentication | In-process | JWT Bearer | 401 UnauthorizedError |
| Orders module | Internal (data coupling) | Customer lookup and projection for orders | Shared DB table (direct Prisma access) | Rows | NotFoundError in OrderService |
| Pino logger | Internal | Logs unhandled errors (via errorMiddleware) | In-process | JSON | n/a |

No external HTTP services, queues or caches are used by the module.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | Routes -> Controller -> Service -> Repository | src/modules/customers/* | Separation of HTTP, business and data concerns |
| Repository Pattern | CustomerRepository | customer.repository.ts | Data access abstraction over Prisma |
| Constructor dependency injection (manual) | Wiring in buildApp | src/app.ts:34-36 | Decoupled construction, testability |
| Router factory | buildCustomerRouter(controller) | customer.routes.ts:12 | Controller injected into routes |
| Middleware chain / Schema validation | validate({ body, query, params }) with Zod | customer.routes.ts:16-24 | Declarative validation |
| Centralized error handling | AppError subclasses plus errorMiddleware | src/shared/errors, error.middleware.ts | Uniform error contract |
| Typed error codes | ConflictError('...', 'EMAIL_ALREADY_USED') | customer.service.ts:32 | Machine-readable errors |
| Query-object / specification-like builder | buildWhere | customer.repository.ts:12 | Dynamic search filter |
| Unit of work (read consistency) | $transaction([findMany, count]) | customer.repository.ts:26 | Consistent page and total |
| Response envelope | paginated() | src/shared/http/response.ts | Uniform pagination format |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Test coverage | No tests for any customer endpoint or rule | Regressions in uniqueness, validation, delete behavior go undetected |
| High | CustomerService.delete / errorMiddleware | FK RESTRICT violation (Prisma P2003) is not mapped; likely returns 500 (inferred, not executed) | Misleading error for a normal business situation; noisy error logs |
| Medium | Authorization | No role check; OPERATOR can delete or modify any customer | Possible excess privilege depending on intent |
| Medium | Email uniqueness | Check-then-write race; fallback returns generic CONFLICT code instead of EMAIL_ALREADY_USED | Inconsistent error code under concurrency |
| Medium | `document` field | Length-only validation, non-unique index, no normalization or checksum | Duplicate or invalid documents possible; inconsistent formatting |
| Medium | Data protection | Document (national ID), phone, email and address returned and stored in plaintext; no PII masking or audit trail; `req.user` unused for audit | Privacy and compliance exposure (assumption of LGPD context is not stated in code) |
| Low | Address handling | Address is JSON with Zod-only shape validation; update replaces whole object | No DB-level integrity; no partial address update |
| Low | Update with empty body | `{}` passes validation | No-op write |
| Low | Listing | Sort by createdAt only; substring search over three columns with `contains` (leading wildcard, no full-text index) | Non-deterministic order on ties; scan cost grows with table size (document has index but unused by contains) |
| Low | Duplicated error mapping | Controller try/catch boilerplate repeated in all handlers | Repetition |
| Low | Cross-module coupling | OrderService reads the customers table directly, bypassing CustomerRepository | Customer access logic spread across modules |
| Low | Type safety | `req.query as unknown as ListCustomersQuery` cast; `req.params.id!` non-null assertions | Relies on validate middleware having run |
| Low | Deletion | Hard delete only, no soft delete | Irreversible data loss for customers without orders |

---

## 11. Test Coverage Analysis

Test infrastructure: Vitest 2.1.4 + Supertest, `tests/**/*.test.ts`, single fork, sequential, setup at `tests/setup.ts` (cleans tables, including `customer.deleteMany()` at tests/setup.ts:14). No coverage tooling or report is configured or present, so numeric coverage is unavailable.

Test files located in the project: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`. There is no `tests/customers.test.ts` and no tests co-located under `src/`.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| CustomerController | 0 | 0 | 0% direct (not measured) | None |
| CustomerService | 0 | 0 | 0% direct (not measured) | None; email conflict, 404, partial update untested |
| CustomerRepository | 0 | 0 | Indirect only | Search, pagination untested |
| customer.schemas (validation) | 0 | 0 | Not measured | None |
| Customer fixture use (tests/orders.test.ts) | 0 | 8 tests use `createTestCustomer` (orders.test.ts:15,45,62,92 and others) | Customer rows created via Prisma directly, bypassing the module | Not a test of this module |

Observations:
- `createTestCustomer` (tests/helpers/factories.ts:42-60) inserts through Prisma directly, so none of the module code runs in tests.
- The orders test for a missing customer is not present; the only 404 tested is for a non-existent product (tests/orders.test.ts:42-57).
- The delete-with-orders scenario, email conflict, validation errors, pagination and search have no tests. This is flagged as a risk.
