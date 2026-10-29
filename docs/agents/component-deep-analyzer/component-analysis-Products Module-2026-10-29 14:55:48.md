# Component Deep Analysis Report: Products Module

Analysis date: 2026-10-29. Scope: `src/modules/products` (5 files) plus the integration surface in `src/app.ts`, `src/routes/index.ts`, `src/middlewares/*`, `src/shared/*`, `prisma/schema.prisma`, `src/modules/orders/order.service.ts` and `tests/*`. No files were modified and no code or tests were executed. Ignored folders: none.

## 1. Executive Summary

The Products Module is the product catalog CRUD slice of an Express + TypeScript + Prisma (MySQL) order-management REST API. It follows a Controller / Service / Repository layering with Zod schemas for request validation. It exposes five endpoints under `/api/v1/products`, all behind JWT authentication.

Key findings:

- The module has a single explicit domain rule beyond field validation: SKU uniqueness, checked in the service (`product.service.ts:31-34`, `:47-52`) and backed by a database unique index.
- Any authenticated user (ADMIN or OPERATOR) can create, update and delete products. `requireRole` exists but is not used in this module (`product.routes.ts:14-25`).
- Stock (`stockQuantity`) and activation (`active`) are owned by this module's data model, but stock is mutated by the Orders module directly through the Prisma transaction client, bypassing `ProductRepository` (`order.service.ts:62`, `:208-241`).
- Product deletion is a hard delete. The FK `order_items_productId_fkey` is `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:119`). Deleting a product that appears in an order is not handled in the service. The resulting Prisma error (probably P2003, an inference) is not mapped in `error.middleware.ts`, so it most likely surfaces as HTTP 500.
- There are no tests dedicated to products. The module is only exercised indirectly via `createTestProduct` in the orders tests.
- `ProductRepository.findManyByIds` is defined but no caller was found in `src` (confidence: high, based on grep).

## 2. Data Flow Analysis

Create (POST /api/v1/products):

```
1. express.json (limit 1mb) parses body (src/app.ts:59)
2. requestLogger middleware (src/app.ts:60)
3. Router mount /api/v1 -> /products (src/routes/index.ts:27, src/app.ts:67)
4. authenticate: Bearer JWT verified, req.user populated (auth.middleware.ts:27-47)
5. validate({ body: createProductSchema }): Zod parse, defaults applied, req.body replaced (validate.middleware.ts:14-16)
6. ProductController.create -> ProductService.create (product.controller.ts:27-34)
7. ProductService: ProductRepository.findBySku -> ConflictError(SKU_ALREADY_USED) if found (product.service.ts:31-34)
8. ProductRepository.create -> prisma.product.create (product.repository.ts:51-53)
9. Controller responds 201 with the Prisma entity (raw, no DTO mapping)
10. Any thrown error -> next(err) -> errorMiddleware (error.middleware.ts:14-72)
```

List (GET /):

```
1. authenticate
2. validate({ query }): coerces page/pageSize, parses active 'true'|'false' to boolean, trims search; result merged into req.query via Object.assign (validate.middleware.ts:17-20)
3. ProductController.list casts req.query to ListProductsQuery (product.controller.ts:10)
4. ProductService.list computes skip=(page-1)*pageSize, take=pageSize (product.service.ts:15-20)
5. ProductRepository.list: buildWhere + prisma.$transaction([findMany orderBy createdAt desc, count]) (product.repository.ts:25-37)
6. paginated() builds { data, pagination{page,pageSize,total,totalPages} } (response.ts:13-24)
7. 200 JSON
```

Get / Update / Delete follow the same prefix (authenticate -> validate params/body). Update performs: getById (404) -> SKU conflict check (if sku provided) -> repository.update with only defined fields -> 200. Delete performs: getById (404) -> repository.delete -> 204 with empty body.

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authentication | All product routes require a valid Bearer JWT | product.routes.ts:14; auth.middleware.ts:27-47 |
| Authorization | No role restriction (ADMIN and OPERATOR equal) | product.routes.ts:14-25 |
| Validation | Resource id must be a UUID | product.schemas.ts:3 |
| Validation | sku: string, 2-64 chars | product.schemas.ts:6 |
| Validation | name: string, 2-200 chars | product.schemas.ts:7 |
| Validation | description: optional, max 2000 chars | product.schemas.ts:8 |
| Validation | priceCents: integer >= 0 (zero allowed) | product.schemas.ts:9 |
| Validation | stockQuantity: integer >= 0, default 0 | product.schemas.ts:10 |
| Validation | active: boolean, default true | product.schemas.ts:11 |
| Validation | Update accepts any subset of create fields (partial) | product.schemas.ts:14 |
| Validation | Pagination: page >= 1 (default 1), pageSize 1-100 (default 20) | product.schemas.ts:17-18 |
| Validation | Filters: search trimmed non-empty; active only 'true' or 'false' | product.schemas.ts:19-23 |
| Business Logic | SKU must be unique on create | product.service.ts:31-34 |
| Business Logic | SKU must be unique on update, excluding the product itself | product.service.ts:47-52 |
| Business Logic | Update/Delete/Get of a missing product returns 404 | product.service.ts:24-27, 46, 64 |
| Business Logic | Update is a partial patch; undefined fields are not touched | product.service.ts:53-60 |
| Business Logic | description defaults to null on create | product.service.ts:38 |
| Query Logic | Search is substring match on name OR sku | product.repository.ts:16-21 |
| Query Logic | Active filter applied only when provided | product.repository.ts:15 |
| Query Logic | Order by createdAt descending | product.repository.ts:30 |
| Data Integrity | sku unique index; FK from order_items RESTRICT on delete | schema.prisma:58; migration.sql:119 |
| Cross-module | Inactive products cannot be ordered | order.service.ts:66-72 |
| Cross-module | Stock decrements/increments performed by Orders | order.service.ts:208-241 |
| Cross-module | Order unit price taken from product.priceCents at order time | order.service.ts:77 |

### Detailed breakdown of the business rules

---

### Business Rule: Authenticated access to the catalog

**Overview**:
Every product endpoint requires a valid JWT in the `Authorization: Bearer` header. No role-based distinction is made.

**Detailed description**:
The router applies `router.use(authenticate)` before any route (`product.routes.ts:14`). The middleware rejects requests lacking a header starting with "bearer " (case-insensitive) or having an empty token with 401 UNAUTHORIZED, and verifies the token with `jsonwebtoken` using `env.JWT_SECRET` (`auth.middleware.ts:28-45`). On success `req.user` receives `{id, email, role}` from the payload. Any verification failure (bad signature, expiry) is collapsed into a single 401 "Invalid or expired token".

The module does not call `requireRole`, so ADMIN and OPERATOR have identical permissions, including creation, price changes and deletion. This is an observed fact; whether it is intentional is not documented in the code (ambiguity). The role claim is trusted from the token without a database lookup, so a user deactivated or demoted after token issuance keeps access until expiry.

Use cases: a front-office operator listing the catalog; an admin creating products. Both go through the same path, and unauthenticated callers never reach validation or the service.

**Rule workflow**:
```
Request -> authenticate -> (missing/invalid header | invalid token) -> 401
                        -> valid -> req.user set -> validate -> controller
```

---

### Business Rule: Input validation (create, update, params, list query)

**Overview**:
Zod schemas constrain all inputs before the service runs. Failures return 400 VALIDATION_ERROR with per-field details.

**Detailed description**:
Create requires `sku` (2-64), `name` (2-200) and `priceCents` (non-negative integer). It accepts optional `description` (max 2000), `stockQuantity` (non-negative integer, default 0) and `active` (default true). Prices are integer cents, which avoids floating-point issues; a zero price is valid. Because Zod objects strip unknown keys by default, extra properties are silently discarded (default Zod behavior; not configured as strict in this file).

Update uses `createProductSchema.partial()`. Note that `.partial()` on fields with `.default()` yields optional fields whose defaults are still applied by Zod when the key is absent. In Zod, a `.default()` inside `.partial()` remains active, so a PATCH with an empty body or one omitting `stockQuantity`/`active` would parse to `stockQuantity: 0` and `active: true`, which the service then writes because it checks `!== undefined` (`product.service.ts:58-59`). This is an inference from Zod semantics and has no test evidence; confidence: medium. If correct, a PATCH of only `name` would reset stock to 0 and re-activate the product. This should be verified against the installed Zod version.

The id param must be a UUID (`product.schemas.ts:3`). List query coerces `page` and `pageSize`, caps pageSize at 100, trims `search` and rejects empty search, and accepts `active` only as the literal strings 'true'/'false' (case-sensitive), transforming to boolean. The validate middleware merges parsed query into `req.query` with `Object.assign` and reassigns `req.params` and `req.body` (`validate.middleware.ts:14-23`). Errors are mapped to `{path, message}` details.

**Rule workflow**:
```
validate(body|query|params) -> Zod parse
  success -> normalized values attached -> next()
  ZodError -> ValidationError(400, VALIDATION_ERROR, details[])
```

---

### Business Rule: SKU uniqueness

**Overview**:
No two products may share a SKU. Violations return 409 with code SKU_ALREADY_USED.

**Detailed description**:
On create the service looks up `findBySku` and throws `ConflictError('SKU already in use', 'SKU_ALREADY_USED')` if a record exists (`product.service.ts:31-34`). On update, if `input.sku` is provided, the service looks it up and conflicts only when the found product has a different id (`:47-52`), allowing a product to be re-saved with its own SKU.

The check is a read-then-write sequence without a transaction, so concurrent requests can both pass the check. The database `@unique` on `sku` (`schema.prisma:58`) is the final guard; in that race the Prisma P2002 error is mapped by the error middleware to 409 with code CONFLICT and a generic "Unique constraint violation on: ..." message (`error.middleware.ts:38-47`). Clients can therefore see two different 409 codes for the same logical conflict.

SKU comparison depends on the MySQL column collation (case sensitivity not determinable from the code; the schema does not specify a collation). SKU is not normalized (no trim/uppercase) in the schema.

**Rule workflow**:
```
create/update(sku) -> findBySku
   exists (and id differs for update) -> 409 SKU_ALREADY_USED
   else -> write -> (race) DB unique violation P2002 -> 409 CONFLICT
```

---

### Business Rule: Existence check on read, update and delete

**Overview**:
Operations on a nonexistent product id return 404 NOT_FOUND "Product not found".

**Detailed description**:
`getById` throws `NotFoundError('Product')` when `findById` returns null (`product.service.ts:24-27`). `update` and `delete` call `getById` first (`:46`, `:64`) so that a clean 404 is returned before any write. Since the id is validated as a UUID, malformed ids fail earlier with 400.

There is a time-of-check/time-of-use gap: if the record is deleted between the check and the write, Prisma raises P2025, which the error middleware maps to 404 with code NOT_FOUND and message "Resource not found" (`error.middleware.ts:48-53`). Each update/delete costs an extra read query.

**Rule workflow**:
```
getById/update/delete -> findById -> null -> 404 NOT_FOUND
                                  -> found -> continue
```

---

### Business Rule: Partial update semantics

**Overview**:
PATCH only writes fields that are not `undefined`.

**Detailed description**:
The service builds the update payload by conditionally spreading each field (`product.service.ts:53-60`). Thus `description` can be set to a new string, but there is no way to clear it to null: the schema permits only `string | undefined` for `description`, so null is rejected by validation (400). Likewise on create, an omitted description is stored as `null` (`:38`).

The updatedAt timestamp is maintained by Prisma `@updatedAt`. The repository's `update` accepts `Prisma.ProductUncheckedUpdateInput`, so the service is the only place restricting writable fields. Stock can be changed directly through PATCH (`stockQuantity`), in parallel to Orders' increment/decrement; a PATCH is an absolute overwrite and can race with concurrent order stock changes (no optimistic locking or version column).

See also the validation rule regarding `.partial()` and defaults.

**Rule workflow**:
```
PATCH -> getById -> (sku conflict check) -> build payload from defined fields -> prisma.update -> 200
```

---

### Business Rule: Listing, search, filtering and pagination

**Overview**:
Paged list ordered newest first with optional text search and active filter.

**Detailed description**:
`page`/`pageSize` are converted to `skip`/`take` (`product.service.ts:17-19`). The repository runs `findMany` and `count` with the same `where` inside one `$transaction` for a consistent total (`product.repository.ts:27-35`). Search uses `contains` on `name` and `sku` joined by OR; the filter on `active` is exact. The ordering is `createdAt desc` only, with no secondary tiebreaker, so rows with equal timestamps may paginate non-deterministically.

The response envelope is `{ data: Product[], pagination: { page, pageSize, total, totalPages } }` (`response.ts`). A page beyond the last returns an empty `data` array with 200. `contains` becomes SQL LIKE '%term%', which cannot use the `name` index efficiently (inference about MySQL behavior). Wildcard characters in the term are handled by Prisma's escaping (not verified).

**Rule workflow**:
```
query -> skip/take -> where{active?, OR[name~,sku~]} -> [findMany, count] -> paginated envelope
```

---

### Business Rule: Hard deletion and referential protection

**Overview**:
Delete removes the row permanently; products referenced by order items cannot be deleted at the database level.

**Detailed description**:
`ProductService.delete` checks existence then calls `prisma.product.delete` (`product.service.ts:63-66`), returning 204. There is no soft-delete; the `active` flag serves the "disable" purpose and is honored by the Orders module.

The FK from `order_items.productId` is `ON DELETE RESTRICT` (`migration.sql:119`), so deleting a product with order history fails at the DB. The service performs no pre-check for order items and `error.middleware.ts` handles only P2002 and P2025; a foreign key violation (Prisma P2003, by inference from Prisma behavior) would fall through to the generic 500 branch and be logged as an unhandled error. This behavior is not covered by tests.

**Rule workflow**:
```
DELETE -> getById (404) -> prisma.delete
   no order items -> 204
   has order items -> DB FK violation -> unmapped error -> 500 INTERNAL_SERVER_ERROR (inferred)
```

---

### Business Rule: Product state consumed by Orders (inactive products, price snapshot, stock)

**Overview**:
Fields managed by this module drive order creation rules in the Orders module.

**Detailed description**:
At order creation, within a transaction, products are fetched by id; a count mismatch raises 404 Product not found; any product with `active = false` raises 422 INACTIVE_PRODUCT (`order.service.ts:62-72`). The unit price is copied from `product.priceCents` into the order item (`:77`), making later price edits non-retroactive for existing orders.

Stock availability is validated (`product.stockQuantity < quantity` -> InsufficientStockError 422) and decremented in the same transaction (`:208-229`); cancellation or related transitions increment it back (`:235-243`). These writes use `tx.product` directly rather than `ProductRepository`. Rules are documented here because they define the semantic contract of `active`, `priceCents` and `stockQuantity`, but they are implemented in the Orders module and belong to its analysis.

**Rule workflow**:
```
Order create -> load products -> missing? 404 -> inactive? 422 -> stock insufficient? 422 -> decrement stock -> snapshot priceCents
```

---

## 4. Component Structure

```
src/modules/products/
├── product.controller.ts   # HTTP handlers (list, getById, create, update, delete); try/catch -> next(err)
├── product.service.ts      # Business logic: SKU uniqueness, existence checks, partial update, pagination mapping
├── product.repository.ts   # Prisma data access: list(+count in transaction), findById, findBySku, findManyByIds, create, update, delete
├── product.routes.ts       # Router factory buildProductRouter; auth + validate wiring
└── product.schemas.ts      # Zod schemas and inferred DTO types

Related (outside boundary):
src/app.ts                      # Manual DI composition (lines 38-40, 50)
src/routes/index.ts             # Mounts /products (line 27)
src/middlewares/{auth,validate,error}.middleware.ts
src/shared/http/response.ts     # paginated()
src/shared/errors/              # NotFoundError, ConflictError
prisma/schema.prisma            # model Product (lines 56-72)
prisma/seed.ts                  # productsSeed (line 142)
tests/helpers/factories.ts      # createTestProduct (lines 62-74)
```

## 5. Dependency Analysis

Internal dependencies:

```
buildProductRouter -> authenticate, validate, product.schemas, ProductController(type)
ProductController  -> ProductService (type), ListProductsQuery (type)
ProductService     -> ProductRepository (type), ConflictError, NotFoundError, paginated, schema types
ProductRepository  -> PrismaClient, Prisma, Product (@prisma/client)
Composition: src/app.ts new ProductRepository(prisma) -> new ProductService(repo) -> new ProductController(service)
Reverse: routes/index.ts -> buildProductRouter/ProductController; orders.service -> product table (via tx, not via the module)
```

External dependencies (versions in package.json not re-verified here):

- express (routing, RequestHandler)
- zod (validation)
- @prisma/client / Prisma ORM with MySQL (persistence)
- jsonwebtoken (via authenticate middleware)

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| ProductController | 2 (routes/index.ts, app.ts; product.routes by type) | 2 (ProductService, schema type) | Low |
| ProductService | 1 (ProductController; app.ts for construction) | 5 (ProductRepository, ConflictError, NotFoundError, paginated, schema types) | Medium |
| ProductRepository | 1 (ProductService; app.ts for construction) | 2 (PrismaClient, Prisma types) | Medium |
| buildProductRouter | 1 (routes/index.ts) | 5 (authenticate, validate, 4 schemas, controller type) | Low |
| product.schemas | 3 (routes, service, controller) | 1 (zod) | Medium |
| Product table (data model) | 3 modules (products, orders service, order repository/seed) | n/a | High |

Counts are manual import-level approximations. The Product table is the highest effective coupling point because Orders accesses it outside the module boundary.

## 7. Endpoints

Base path: `/api/v1/products` (`src/app.ts:67`, `src/routes/index.ts:27`). All require `Authorization: Bearer <JWT>`.

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/products | GET | Paginated list; query: page, pageSize (max 100), search, active. Returns 200 `{data, pagination}` |
| /api/v1/products/{id} | GET | Get product by UUID. 200, 404 |
| /api/v1/products | POST | Create product. 201, 400, 409 SKU_ALREADY_USED |
| /api/v1/products/{id} | PATCH | Partial update. 200, 400, 404, 409 |
| /api/v1/products/{id} | DELETE | Hard delete. 204, 404 (FK-protected deletes are unmapped, see risks) |

Error body format: `{ error: { code, message, details? } }`. Common codes: UNAUTHORIZED (401), VALIDATION_ERROR (400), NOT_FOUND (404), SKU_ALREADY_USED / CONFLICT (409), INTERNAL_SERVER_ERROR (500).

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL via Prisma | Database | Product persistence (table `products`) | Prisma Client / MySQL | Relational rows | P2002 -> 409, P2025 -> 404, others -> 500 via errorMiddleware |
| Orders module | Internal module (shared table) | Reads active/price/stock; mutates stock | Direct Prisma tx access | Prisma entities | Domain errors 404/422 in Orders |
| Auth middleware (JWT) | Internal | Caller authentication | HTTP Bearer / JWT | JWT claims | 401 UnauthorizedError |
| Database seed | Tooling | Initial catalog data | Prisma script | TS objects | n/a |

No external third-party services are called.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | Controller -> Service -> Repository | src/modules/products/* | Separation of HTTP, rules, persistence |
| Repository | ProductRepository | product.repository.ts | Data access abstraction over Prisma |
| Dependency Injection (manual constructor) | app.ts composition root | src/app.ts:38-40 | Wiring without a container |
| Router factory | buildProductRouter(controller) | product.routes.ts:12 | Injectable routing |
| Schema-based validation / DTO inference | Zod + z.infer | product.schemas.ts | Runtime validation and static types |
| Middleware chain | authenticate, validate | product.routes.ts | Cross-cutting concerns |
| Centralized error handling | AppError hierarchy + errorMiddleware | shared/errors, error.middleware.ts | Uniform error contract |
| Query builder method | buildWhere | product.repository.ts:13 | Reusable filter composition |
| Envelope / pagination wrapper | paginated() | shared/http/response.ts | Consistent list responses |
| Arrow-function handlers as properties | Controller class fields | product.controller.ts | Preserves `this` binding |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Delete flow | No handling of FK RESTRICT (products with order items); only P2002/P2025 mapped (inferred 500) | Unclear client error, noisy error logs |
| High | Schemas (PATCH) | `.partial()` on schema with `.default()` may re-apply stockQuantity=0 and active=true on partial updates (inferred, needs verification) | Silent stock reset / reactivation |
| Medium | Authorization | No role checks on mutating routes | Any operator can alter prices or delete products |
| Medium | Stock ownership | Orders mutates stock via tx.product bypassing repository; PATCH can overwrite stock absolutely | Lost updates, split ownership of data |
| Medium | SKU check | Read-then-write race; inconsistent 409 codes (SKU_ALREADY_USED vs CONFLICT) | Inconsistent client contract |
| Medium | Tests | No product-specific tests | Regressions undetected |
| Low | Controller | Query cast via `as unknown as`; `req.params.id!` non-null assertions | Type safety bypass |
| Low | Repository | `findManyByIds` unused; list ordering has no tiebreaker | Dead code, unstable pagination |
| Low | Update | Extra read before every update/delete; description cannot be cleared | Performance, functional limitation |
| Low | Search | LIKE '%term%' on name/sku | Scan cost on large catalogs (inferred) |
| Low | Responses | Raw Prisma entity returned (no DTO) | Schema changes leak to API |
| Low | Data | Hard delete with no audit trail | Loss of history |

## 11. Test Coverage Analysis

Test files located: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts` (cleans products at line 13), `tests/helpers/factories.ts` (`createTestProduct`, lines 62-74). Runner config: `vitest.config.ts` (tests/**/*.test.ts, single fork, sequential, DB-backed integration tests via supertest). No coverage configuration or reports were found; percentages cannot be determined.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| ProductController | 0 | 0 direct | Unknown (none measured) | No endpoint test hits /api/v1/products |
| ProductService | 0 | 0 direct | Unknown | SKU conflict, 404, partial update, pagination untested |
| ProductRepository | 0 | 0 direct | Unknown | Only indirectly via factory (uses prisma directly, not repository) |
| product.schemas | 0 | 0 | Unknown | Validation boundaries untested |
| Product behavior in Orders (tests/orders.test.ts) | 0 | Several (non-existent product line 42; stock decrement/insufficient lines 63-135; stock restore lines 138-160) | Indirect | Covers consumption of product stock; inactive-product rule not found in tests (not confirmed beyond grep) |

Missing coverage highlights: authentication on product routes, SKU uniqueness (create/update), PATCH default behavior, search/filter/pagination, delete with order references, UUID validation.
