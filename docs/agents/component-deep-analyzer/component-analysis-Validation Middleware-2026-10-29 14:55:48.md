# Component Deep Analysis Report: Validation Middleware

Component: Validation Middleware (`src/middlewares/validate.middleware.ts`)
Analysis date: 2026-10-29 14:55:48
Scope: entire project root; no folders ignored. Analysis only; no project files were modified.

---

## 1. Executive Summary

The Validation Middleware is a single-file, 37-line Express middleware factory exposing one function, `validate(schemas)` (src/middlewares/validate.middleware.ts:11). It accepts an object with up to three optional Zod schemas (`body`, `query`, `params`) and returns an Express `RequestHandler` that validates the matching request parts. On success it replaces or merges the parsed (coerced, defaulted, transformed) values into the request and calls `next()`. On a `ZodError` it converts the issues into a `ValidationError` (HTTP 400, code `VALIDATION_ERROR`) with a `details` array of `{ path, message }` and forwards it to the central error handler. Any other exception is forwarded unchanged.

Role in the system: it is the input-validation gate on the HTTP layer. It is applied per route in the routers of the auth, customers, products, orders and users modules (18 call expressions across 5 route files). It sits after authentication middleware (where present) and before controllers.

Key findings:

- The component is stateless, small and cohesive, with a single responsibility.
- Body and params are replaced by the parsed output (unknown keys stripped, since Zod objects strip by default). Query is merged with `Object.assign`, so unknown query keys are retained (see Rule 4).
- Zod-issue formatting is duplicated in `src/middlewares/error.middleware.ts:7-12` and `:30-37`, which also handles a raw `ZodError`.
- Direct automated tests for the middleware do not exist. Coverage is indirect and limited to one invalid-payload test (tests/auth.test.ts:30-39) and a few incidental flows in tests/orders.test.ts.
- One user-module schema (`idParamSchema`) is declared inline in the route file and duplicates `userIdParamSchema` in `user.schemas.ts`.

---

## 2. Data Flow Analysis

```
1. HTTP request enters Express (src/app.ts:57); express.json({ limit: '1mb' }) parses the body (app.ts:59).
2. requestLogger runs (app.ts:60), then the request is routed to /api/v1/<module> (app.ts:67, routes/index.ts:24-28).
3. Route-level middlewares run in order. For protected routes: authenticate (and requireRole for users) run first.
   Auth routes /register and /login run validate directly with no authenticate.
4. validate(schemas) handler executes (validate.middleware.ts:12):
   a. body   -> schemas.body.parse(req.body)    -> req.body   = parsed result (line 14-16)
   b. query  -> schemas.query.parse(req.query)  -> Object.assign(req.query, parsed) (line 17-20)
   c. params -> schemas.params.parse(req.params)-> req.params = parsed result (line 21-23)
5a. All succeed: next() (line 24) -> controller method receives normalized, typed data.
5b. ZodError thrown by any parse (line 26): issues mapped to { path: issue.path.join('.') || '(root)', message }
    (line 27-30); next(new ValidationError('Validation failed', details)) (line 31); return.
5c. Any other error: next(err) (line 34).
6. errorMiddleware (src/middlewares/error.middleware.ts:14) catches the AppError subclass and responds
   400 { error: { code: 'VALIDATION_ERROR', message: 'Validation failed', details: [...] } }.
```

Order within a single call is fixed: body, then query, then params. Validation is fail-fast across sources: the first source that throws stops processing, so errors from later sources are not reported in the same response. Within one source, Zod reports all issues for that schema.

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation | Each request part (body, query, params) is validated only if a schema is supplied for it | src/middlewares/validate.middleware.ts:14, 17, 21 |
| Validation | Evaluation order is body, query, params; first failing source aborts | src/middlewares/validate.middleware.ts:14-23 |
| Data Normalization | Body is replaced by the schema-parsed output (coercion, defaults, stripping unknown keys) | src/middlewares/validate.middleware.ts:15 |
| Data Normalization | Query parsed output is merged into the existing req.query (not replaced) | src/middlewares/validate.middleware.ts:18-19 |
| Data Normalization | Params replaced by schema-parsed output | src/middlewares/validate.middleware.ts:22 |
| Error Handling | ZodError converted to ValidationError (400, VALIDATION_ERROR) with details | src/middlewares/validate.middleware.ts:26-33 |
| Error Handling | Empty issue path reported as "(root)"; nested paths joined with "." | src/middlewares/validate.middleware.ts:28 |
| Error Handling | Non-Zod errors are passed through unchanged to next(err) | src/middlewares/validate.middleware.ts:34 |
| Schema-defined (consumed) | Pagination: page >= 1 (default 1), pageSize 1..100 (default 20) | src/modules/customers/customer.schemas.ts:26-27; products/product.schemas.ts:17-18; orders/order.schemas.ts:24-25 |
| Schema-defined (consumed) | Identifiers must be UUIDs | customer.schemas.ts:4; product.schemas.ts:3; order.schemas.ts:4; user.schemas.ts:4; user.routes.ts:7 |
| Schema-defined (consumed) | Auth/user: email valid and <= 255, password 8..72 on register, name 2..150, role ADMIN or OPERATOR default OPERATOR | auth.schemas.ts:3-8; user.schemas.ts:7-12 |
| Schema-defined (consumed) | Order creation: at least one item, positive integer quantity, non-negative integer discountCents default 0 | order.schemas.ts:11-16 |
| Schema-defined (consumed) | Product: priceCents/stockQuantity non-negative integers; stockQuantity default 0; active default true | product.schemas.ts:5-12 |
| Schema-defined (consumed) | Partial update for customers and products (all fields optional) | customer.schemas.ts:23; product.schemas.ts:14 |

The first group (rows 1-8) is behavior owned by the middleware. The remaining rows are rules defined in module schemas that the middleware executes; they are documented because the middleware is the enforcement point.

### Detailed breakdown of the business rules

---

### Business Rule: Conditional, per-source validation

**Overview**:
The middleware validates only those request sources for which a schema was passed in the `schemas` argument.

**Detailed description**:
`validate` takes a `Sources` object (validate.middleware.ts:5-9) in which `body`, `query` and `params` are each optional. Inside the handler, each source is guarded by an `if (schemas.<source>)` check (lines 14, 17, 21). A route that supplies only `params` (for example `GET /customers/:id`, customer.routes.ts:17) never touches `req.body` or `req.query`; a route that supplies `{ params, body }` (for example `PATCH /orders/:id/status`, order.routes.ts:20-23) validates both.

This makes the middleware composable and low-cost: routes pay only for the validation they declare. It also means that a route with no schema for a source accepts arbitrary content in that source. For example, `GET /customers/:id` ignores any body, and routes that do not declare a `query` schema pass raw query strings to the controller. Routes such as `GET /auth/me` do not use `validate` at all (auth.routes.ts:12).

Use cases observed: list endpoints validate `query`; get-by-id and delete validate `params`; create validates `body`; update validates `params` plus `body`.

**Rule workflow**:
```
validate(schemas) called at route build time -> returns handler
request -> for each of [body, query, params]: schema present? -> yes: parse; no: skip -> next()
```

---

### Business Rule: Fixed evaluation order and fail-fast across sources

**Overview**:
Sources are validated in the order body, query, params, and the first Zod failure ends validation for the request.

**Detailed description**:
All three parse calls are inside one `try` block (lines 13-35). A `ZodError` thrown by `schemas.body.parse` jumps straight to the `catch` (line 25), so query and params are never evaluated for that request. Consequently a request that is invalid in both body and params gets only the body errors. After the client fixes the body, it receives the params error on the next request.

Within a single source, Zod's `parse` collects all issues for that schema, and they are all returned in `details`. For the register test payload `{ email: 'not-an-email', password: 'short', name: '' }`, multiple issues are expected in one response (tests/auth.test.ts:30-39 asserts only that `details` is an array).

Because params are validated last, a malformed `:id` on an `PATCH /:id` route will not be reported if the body is also invalid. The ordering is implicit (code order), not configurable, and undocumented in the code.

**Rule workflow**:
```
try { body.parse -> query.parse -> params.parse -> next() }
catch ZodError -> ValidationError(details) -> next(error)
```

---

### Business Rule: Body replaced with parsed output

**Overview**:
After successful validation, `req.body` is overwritten with the Zod result, so downstream code sees normalized data.

**Detailed description**:
Line 15 assigns `schemas.body.parse(req.body)` back to `req.body`. Zod's default object behavior strips keys not declared in the schema, applies `.default()` values, runs `.transform()`s and coerces where `z.coerce` is used. As a result, controllers only see declared fields. Examples: `registerSchema` defaults `role` to `OPERATOR` (auth.schemas.ts:7), `createOrderSchema` defaults `discountCents` to 0 (order.schemas.ts:14), and `createProductSchema` defaults `stockQuantity` to 0 and `active` to true (product.schemas.ts:10-11).

A security-relevant effect is that undeclared fields (for example, an attempt to inject `id` or `createdAt`) are silently dropped rather than rejected, since no schema uses `.strict()`. This is a "strip, don't reject" policy. Note that `registerSchema` accepts a client-supplied `role` of ADMIN or OPERATOR (auth.schemas.ts:7), so public registration can request the ADMIN role as far as validation is concerned; whether the service restricts it is outside this component (not verified in this analysis).

The type assertion `as z.infer<NonNullable<S['body']>>` (line 15) has no runtime effect, and the value is assigned to `req.body`, which is typed `any` in Express, so the inferred type is not propagated to controllers.

**Rule workflow**:
```
req.body (raw JSON) -> schemas.body.parse -> typed/normalized object -> req.body
```

---

### Business Rule: Query merged, not replaced

**Overview**:
Parsed query values are merged into the existing `req.query` using `Object.assign`.

**Detailed description**:
Lines 18-19 parse `req.query` and then `Object.assign(req.query, parsedQuery)`. This differs from body and params. The implementation avoids reassigning `req.query`, which is commonly done because in newer Express versions (5.x) `req.query` is a getter and cannot be assigned. The project pins Express 4.21.1 (package.json:28), where reassignment would work; the code is therefore compatible with both styles (inference; the author's intent is not documented).

The consequence of merging is that coerced values overwrite their string originals (for example `page: '2'` becomes number `2`, `from` becomes a `Date`, `active: 'true'` becomes boolean `true` in product list), defaults are added (page 1, pageSize 20), but unknown query keys remain on `req.query` in their original string form because they are not removed. Unlike body, the query is therefore not stripped. Controllers that read only declared keys are unaffected.

If a query key is an array (for example `?page=1&page=2`), Express yields an array; `z.coerce.number()` on an array yields NaN, resulting in a validation error (behavior of Zod coercion; not covered by tests).

**Rule workflow**:
```
req.query (strings) -> schemas.query.parse -> parsedQuery
Object.assign(req.query, parsedQuery) -> controller reads coerced values
```

---

### Business Rule: Params replaced by parsed output

**Overview**:
`req.params` is reassigned to the parsed value, ensuring identifiers are valid UUIDs before reaching controllers.

**Detailed description**:
Line 22 assigns the parse result to `req.params`. All param schemas in the project contain a single `id: z.string().uuid()` (customer.schemas.ts:3-5, product.schemas.ts:3, order.schemas.ts:4, user.schemas.ts:3-5, user.routes.ts:7). A non-UUID id (such as `abc`) therefore results in a 400 VALIDATION_ERROR rather than reaching the repository and producing a database error or 404.

Because `validate` is mounted at route level (after `router.get('/:id', ...)` matches), `req.params` is populated at that point. The middleware must be attached to a route with the path parameters; if it were mounted via `router.use` at a router level without the param, params would be empty. All current usages are route-level.

User routes duplicate the schema inline (`idParamSchema`, user.routes.ts:7) while `user.schemas.ts` exports `userIdParamSchema`, which is not used by any route in this analysis (no usage found by search).

**Rule workflow**:
```
req.params {id} -> schemas.params.parse (uuid check) -> req.params -> controller
```

---

### Business Rule: ZodError to ValidationError translation

**Overview**:
Validation failures are mapped to a standardized HTTP 400 error with per-field details.

**Detailed description**:
In the `catch` block (line 25), if the error is `instanceof ZodError` (line 26), the middleware maps `err.issues` to `{ path, message }` pairs (lines 27-30). `path` is the Zod issue path joined with `.` (for example `address.zipCode`, `items.0.quantity`); an empty path (error at the root, such as a body that is not an object) is rendered as the literal `(root)`. The message is Zod's own message, or the custom message supplied in a schema (for example `'order must contain at least one item'`, order.schemas.ts:13).

The middleware then calls `next(new ValidationError('Validation failed', details))` and returns (lines 31-32). `ValidationError` extends `AppError` with status 400 and code `VALIDATION_ERROR` (src/shared/errors/http-errors.ts:9-13). `errorMiddleware` serializes it as `{ error: { code, message, details } }` (error.middleware.ts:15-23). The client-facing contract is therefore: status 400, `error.code = "VALIDATION_ERROR"`, `error.message = "Validation failed"`, `error.details = [{ path, message }]`. Only the code, array shape and status are asserted in tests (tests/auth.test.ts:36-38).

The same mapping logic exists independently in error.middleware.ts (`formatZodIssues`, lines 7-12, and the ZodError branch, lines 30-37), providing an identical response for ZodErrors thrown elsewhere (for example in services or controllers). Zod messages may reveal expected formats, but contain no sensitive data; the raw submitted values are not echoed.

**Rule workflow**:
```
ZodError -> issues.map({ path: join('.') || '(root)', message }) -> ValidationError(400, VALIDATION_ERROR) -> next(error) -> errorMiddleware -> JSON response
```

---

### Business Rule: Non-validation errors pass through

**Overview**:
Any error that is not a `ZodError` is forwarded to the error pipeline unchanged.

**Detailed description**:
Line 34 calls `next(err)` for non-Zod exceptions. Examples include errors thrown by an async or custom `.transform()`/`.refine()` that throws a non-Zod error, or unexpected runtime failures (for example, `req.query` being unusable). These are handled by `errorMiddleware`: `AppError` instances with their own code, Prisma known errors, or a generic 500 with logging (error.middleware.ts:14-60).

The handler is synchronous. It uses `schema.parse` (not `parseAsync`), so schemas with asynchronous refinements or transforms would throw an error from Zod stating that async parse must be used; this would be forwarded through `next(err)` as a 500. No current schema uses async refinements.

This rule ensures the middleware never swallows errors and never sends a response itself; it only calls `next`.

**Rule workflow**:
```
catch (err) -> not ZodError -> next(err) -> errorMiddleware (AppError / Prisma / 500)
```

---

### Business Rules (consumed): Schema constraints enforced through the middleware

**Overview**:
The middleware enforces domain input constraints declared in module schemas.

**Detailed description**:
Authentication and users: `email` valid format, max 255; `password` 8..72 characters for register/create (72 matches bcrypt's input limit; the hashing algorithm is not verified in this analysis), `min(1)` for login; `name` 2..150; `role` enum ADMIN or OPERATOR default OPERATOR (auth.schemas.ts, user.schemas.ts).

Customers: `name` 2..150, `email` valid, `phone` 8..32, `document` 11..20, nested `address` (street 1..200, number 1..20, city 1..100, state exactly 2 characters, zipCode 5..15). Updates use `.partial()`, making all top-level fields optional (an empty body `{}` validates successfully; a provided `address` must still be complete since `partial()` is shallow). List query: page, pageSize, optional trimmed non-empty `search` (customer.schemas.ts).

Products: `sku` 2..64, `name` 2..200, `description` max 2000, `priceCents` non-negative integer, `stockQuantity` non-negative integer default 0, `active` boolean default true; partial updates; list query with `search` and `active` accepted as the literal strings 'true' or 'false' and transformed to boolean (product.schemas.ts:20-23).

Orders: `customerId` and each `productId` UUID; `items` array with at least one element; `quantity` positive integer; `discountCents` non-negative integer default 0; `notes` max 1000; status change takes `toStatus` (Prisma `OrderStatus` enum) and optional `reason` max 500; list query supports `status`, `customerId`, `from`, `to` (coerced to Date; invalid dates fail validation; no check that `from <= to`) (order.schemas.ts).

Cross-field and semantic rules (stock availability, status transition validity, uniqueness) are not in this component; they are enforced in services (for example `InvalidStatusTransitionError`, `InsufficientStockError` in src/shared/errors/http-errors.ts:49-69).

**Rule workflow**:
```
route declares validate({ body|query|params: <module schema> }) -> middleware executes schema -> controller receives normalized input
```

---

## 4. Component Structure

```
src/middlewares/
├── validate.middleware.ts          # Component under analysis: validate(schemas) factory (37 lines)
├── auth.middleware.ts              # Sibling: authenticate / requireRole (not part of component)
├── error.middleware.ts             # Sibling: central error handler; renders ValidationError / ZodError
└── request-logger.middleware.ts    # Sibling: request logging

Collaborating files (outside the component):
src/shared/errors/{app-error,http-errors,index}.ts   # ValidationError class
src/modules/*/ *.schemas.ts                           # Zod schemas supplied to validate()
src/modules/*/ *.routes.ts                            # Call sites of validate()
```

Internal organization of validate.middleware.ts:

- Lines 1-3: imports (express type, zod, ValidationError).
- Lines 5-9: `Sources` type.
- Lines 11-37: `validate` generic factory returning a `RequestHandler`.

No configuration, environment variables or feature flags are used by the component.

---

## 5. Dependency Analysis

```
Internal Dependencies:
validate (validate.middleware.ts) -> ValidationError (src/shared/errors/http-errors.ts via index.ts) -> AppError (app-error.ts)

Used by (call sites):
auth.routes.ts, customer.routes.ts, product.routes.ts, order.routes.ts, user.routes.ts
  -> validate -> Zod schemas from each module's *.schemas.ts
Downstream consumer of the produced error:
validate -> next(ValidationError) -> errorMiddleware (error.middleware.ts)

External Dependencies:
- zod 3.23.8 (package.json:33) - ZodError, ZodTypeAny, z.infer type, schema parsing
- express 4.21.1 (package.json:28) - RequestHandler type only (type-only import, erased at compile)
```

Build-time only: `@types/express` 4.17.21. Runtime: only `zod` and the internal error classes.

---

## 6. Afferent and Efferent Coupling

Counted over the component and its immediate collaborators (modules/files that import each symbol). Afferent = number of files importing the symbol; efferent = number of distinct project/external symbols it imports.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| validate (function) | 5 (auth, customer, product, order, user routes; 18 call expressions) | 3 (ZodError, ValidationError, express RequestHandler type; Zod types are type-only) | Medium |
| Sources (type, not exported) | 1 (internal) | 1 (ZodTypeAny) | Low |
| ValidationError | 2 (validate.middleware.ts; src/shared/errors/index.ts re-export) | 1 (AppError) | Medium |
| errorMiddleware (consumer of the output) | 1 (app.ts) | 5 (Prisma, ZodError, AppError, logger, express type) | High |

Call-expression counts: customers 5 (customer.routes.ts:16,17,18,21,24), products 5 (product.routes.ts:16,17,18,21,24), orders 5 (order.routes.ts:16,17,18,21,24), auth 2 (auth.routes.ts:10,11), users 1 (user.routes.ts:16); 18 in total.

Note: the counts above are manual and derived from text search; no tooling was run.

---

## 7. Endpoints

The component does not expose endpoints of its own, but it guards the following routes (prefix `/api/v1`, routes/index.ts:24-28):

| Endpoint | Method | Validated sources | Schema(s) |
|----------|--------|-------------------|-----------|
| /auth/register | POST | body | registerSchema |
| /auth/login | POST | body | loginSchema |
| /users/:id | GET | params | idParamSchema (inline; ADMIN only) |
| /customers | GET | query | listCustomersQuerySchema |
| /customers/:id | GET | params | customerIdParamSchema |
| /customers | POST | body | createCustomerSchema |
| /customers/:id | (update route; method on customer.routes.ts:19-22, not displayed in the search output) | params, body | customerIdParamSchema, updateCustomerSchema |
| /customers/:id | DELETE | params | customerIdParamSchema |
| /products | GET | query | listProductsQuerySchema |
| /products/:id | GET | params | productIdParamSchema |
| /products | POST | body | createProductSchema |
| /products/:id | (update route; method on product.routes.ts:19-22, not displayed) | params, body | productIdParamSchema, updateProductSchema |
| /products/:id | DELETE | params | productIdParamSchema |
| /orders | GET | query | listOrdersQuerySchema |
| /orders/:id | GET | params | orderIdParamSchema |
| /orders | POST | body | createOrderSchema |
| /orders/:id/status | PATCH | params, body | orderIdParamSchema, updateOrderStatusSchema |
| /orders/:id | DELETE | params | orderIdParamSchema |

Error response contract for failures: HTTP 400, `{ "error": { "code": "VALIDATION_ERROR", "message": "Validation failed", "details": [ { "path": "...", "message": "..." } ] } }`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express router pipeline | Internal framework | Run as route-level middleware | In-process | req/res objects | Errors forwarded via next() |
| Zod schemas (module *.schemas.ts) | Internal/external library | Parsing, coercion, validation | In-process | JS objects | ZodError converted to ValidationError |
| ValidationError / AppError | Internal | Typed 400 error | In-process | Error object with statusCode, errorCode, details | Rendered by errorMiddleware |
| errorMiddleware | Internal | Client-facing serialization | HTTP | JSON | Generic 500 fallback |
| Prisma (indirect) | External | order.schemas.ts imports `OrderStatus` enum from @prisma/client for `z.nativeEnum` | In-process | Enum | N/A (schema-level dependency, not of the middleware itself) |

The component performs no I/O, network or database calls.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Middleware / Chain of Responsibility | Express `RequestHandler` returned by `validate` | validate.middleware.ts:12 | Request pipeline stage |
| Factory function / Higher-order function | `validate(schemas)` returns a configured handler | validate.middleware.ts:11 | Per-route configuration via closure |
| Schema-based validation (declarative) | Zod schemas passed by callers | modules/*/*.schemas.ts | Separate validation rules from mechanism |
| Error translation (Adapter) | ZodError mapped to ValidationError | validate.middleware.ts:26-33 | Uniform API error contract |
| Generic constraint | `S extends Sources` | validate.middleware.ts:11 | Type inference from supplied schemas |

Architecture notes: validation is applied at route declaration (explicit, not global); the error response is rendered centrally. The component is a pure function over `req` with no state.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Test coverage | No direct unit tests for validate(); only one invalid-payload integration test | Behaviors such as query merge, params replacement, "(root)" path and ordering are unverified |
| Medium | Query handling | Unknown query keys are retained (Object.assign merge) while body/params keys are stripped | Inconsistent normalization semantics across sources |
| Medium | Registration schema | `role` accepted from client input with ADMIN allowed (auth.schemas.ts:7) | Potential privilege-assignment path if the service does not restrict it (service behavior not verified) |
| Low | Duplication | Zod issue formatting duplicated in validate.middleware.ts:27-30 and error.middleware.ts:7-12 | Divergence risk of error format |
| Low | Duplication | `idParamSchema` inline in user.routes.ts:7 duplicates `userIdParamSchema` (user.schemas.ts:3-5); the exported schema appears unused | Inconsistent schema source |
| Low | Type safety | `as z.infer<...>` casts (lines 15, 18, 22) have no effect on downstream types since `req.body`/`req.params` are not narrowed | Controllers rely on implicit shape |
| Low | Sync parsing | `parse` used instead of `parseAsync` | Async refinements would cause a 500 |
| Low | Fail-fast across sources | Errors from later sources hidden when an earlier source fails | Multiple round trips for clients |
| Low | Strip policy | No `.strict()`; unknown body fields silently dropped | Client typos go undetected |
| Low | Schema semantics | List `from`/`to` not cross-validated; `.partial()` allows empty update body; address in update must be complete | Domain edge cases pass validation |

---

## 11. Test Coverage Analysis

Test infrastructure: Vitest 2.1.4 with Supertest 7.0.0 (package.json:49,52); config in vitest.config.ts (includes `tests/**/*.test.ts`, setup tests/setup.ts, single fork, sequential). No coverage provider or coverage threshold is configured, and no coverage report exists in the repository, so numeric coverage cannot be determined (not executed per analysis rules).

Test files found: tests/auth.test.ts, tests/orders.test.ts, tests/setup.ts, tests/helpers/factories.ts. No tests exist for customers, products or users routes.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| validate() (direct) | 0 | 0 dedicated | Not measured | No dedicated tests |
| Body validation / error translation | 0 | 1 (tests/auth.test.ts:30-39, invalid register payload: asserts 400, `VALIDATION_ERROR`, `details` is an array) | Not measured | Weak assertions: does not check path/message content or "(root)" |
| Body validation success path + defaults | 0 | Incidental (tests/auth.test.ts:6, tests/orders.test.ts:12 create flows) | Not measured | Not asserted explicitly |
| Query validation / coercion | 0 | Incidental (tests/orders.test.ts:163 list orders filtered by status) | Not measured | Does not test invalid query, defaults or bounds |
| Params (UUID) validation | 0 | 0 explicit | Not measured | Untested negative case (non-UUID id) |
| Multi-source order / fail-fast | 0 | 0 | Not measured | Untested |
| Non-Zod error passthrough | 0 | 0 | Not measured | Untested |

Gaps: no tests for params rejection, query bounds (pageSize > 100), nested paths, orders with empty `items`, partial updates, and no tests at all for customers, products and users module validation.

---

## Analysis Notes and Ambiguities

- Line-level coupling counts were derived from text search of `validate`, `ValidationError` and imports; I did not execute tooling.
- The update-route HTTP method for customers and products (customer.routes.ts:19-22, product.routes.ts:19-22) was not displayed in the output reviewed; it is inferred to be an update route since it uses `{ params, body }` with an update schema.
- Hashing behavior behind the 72-character password limit and role restriction on registration are in the auth service and were not analyzed (outside component boundary).
