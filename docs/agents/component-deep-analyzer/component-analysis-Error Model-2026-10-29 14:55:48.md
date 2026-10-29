# Component Deep Analysis Report: Error Model (src/shared/errors)

Date: 2026-10-29 14:55:48

## 1. Executive Summary

The Error Model is a small, dependency-free TypeScript module (3 files, about 80 lines) that defines the application's typed error hierarchy. A base class `AppError` (src/shared/errors/app-error.ts:3) carries an HTTP status code, a machine-readable error code and optional structured details. Nine subclasses in src/shared/errors/http-errors.ts map common HTTP semantics (400, 401, 403, 404, 409, 422) and two domain-specific errors (`InvalidStatusTransitionError`, `InsufficientStockError`) to stable error codes. A barrel file (src/shared/errors/index.ts) is the single public entry point.

Role in the system: services, controllers and middlewares throw these errors. The central `errorMiddleware` (src/middlewares/error.middleware.ts:14) translates them into the JSON envelope `{ error: { code, message, details? } }`. The model is therefore the contract between the domain/service layer and the HTTP response format.

Key findings:
- The component is pure and has no external dependencies. It is imported from 10 consumer files.
- Domain errors are coupled to the Order domain (`InvalidStatusTransitionError`, `InsufficientStockError`) while living in a shared module.
- Error codes are string literals scattered across the component and its consumers; there is no central code registry or enum.
- The component has no dedicated unit tests. It is only covered indirectly via integration tests (tests/auth.test.ts, tests/orders.test.ts).
- `ValidationError` does not accept a `code`, while `BadRequestError` does. `UnauthorizedError` and `ForbiddenError` cannot carry details or a custom code.
- The mapping logic in error.middleware.ts lives outside the component boundary but is the sole consumer of `statusCode`/`errorCode`/`details`.

## 2. Data Flow Analysis

```
1. A failure condition is detected in a consumer (auth middleware, validate middleware,
   route fallback in app.ts, service methods, controllers).
2. The consumer instantiates a typed error from src/shared/errors/index.ts
   (e.g. new NotFoundError('Order')) and throws it (services/controllers) or
   passes it to next(err) (middlewares).
3. AppError constructor stores message, statusCode, errorCode, details and captures the stack trace
   (app-error.ts:8-15).
4. Express routes the error to errorMiddleware (src/middlewares/error.middleware.ts:14).
5. errorMiddleware checks `err instanceof AppError` (line 15) and responds with
   status = err.statusCode and body { error: { code, message, details? } };
   details is omitted when undefined (line 20).
6. Non-AppError errors (ZodError, Prisma P2002/P2025, unknown) are handled by fallback branches
   in the middleware (lines 26-64), outside the component, and produce the same envelope.
```

Specific flows:
- Validation: validate.middleware.ts:25-31 converts a ZodError into `ValidationError('Validation failed', [{path, message}])` and calls next.
- Authentication: auth.middleware.ts:30,36,45,52 emit `UnauthorizedError`; line 56 emits `ForbiddenError('Insufficient permissions')`.
- Unknown routes: app.ts:70 emits `NotFoundError('Route METHOD url')`.
- Order lifecycle: order.service.ts emits NotFound, Validation, UnprocessableEntity, Conflict, InvalidStatusTransition and InsufficientStock errors inside Prisma transactions, so throwing aborts the transaction (the throw occurs inside `$transaction` callbacks, lines 46-223).

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Contract | Every application error carries statusCode, errorCode and optional details | src/shared/errors/app-error.ts:3-15 |
| Mapping | BadRequestError -> HTTP 400, default code BAD_REQUEST | src/shared/errors/http-errors.ts:3-7 |
| Mapping | ValidationError -> HTTP 400, fixed code VALIDATION_ERROR | src/shared/errors/http-errors.ts:9-13 |
| Mapping | UnauthorizedError -> HTTP 401, code UNAUTHORIZED | src/shared/errors/http-errors.ts:15-19 |
| Mapping | ForbiddenError -> HTTP 403, code FORBIDDEN | src/shared/errors/http-errors.ts:21-25 |
| Message | NotFoundError -> HTTP 404, message "<resource> not found", code NOT_FOUND | src/shared/errors/http-errors.ts:27-31 |
| Mapping | ConflictError -> HTTP 409, default code CONFLICT, custom code allowed | src/shared/errors/http-errors.ts:33-37 |
| Mapping | UnprocessableEntityError -> HTTP 422, default code UNPROCESSABLE_ENTITY | src/shared/errors/http-errors.ts:39-43 |
| Domain | Invalid order status transition -> 409 INVALID_STATUS_TRANSITION with {from, to} | src/shared/errors/http-errors.ts:45-53 |
| Domain | Insufficient stock -> 422 INSUFFICIENT_STOCK with list of {sku, requested, available} | src/shared/errors/http-errors.ts:55-63 |
| Response contract | Details are only serialized when defined | src/middlewares/error.middleware.ts:20 |
| Public API | Only the barrel exports are the supported import surface | src/shared/errors/index.ts:1-13 |

### Detailed breakdown of the business rules

---

### Business Rule: Typed error contract (AppError)

**Overview**:
All errors intended to reach API clients extend `AppError`, which exposes `statusCode`, `errorCode` and `details`.

**Detailed description**:
`AppError` extends the native `Error` and sets `name` to the literal 'AppError' (app-error.ts:10). Note that subclasses do not override `name`, so every subclass reports `name === 'AppError'`; discrimination is expected to use `instanceof` or `errorCode` rather than `name`. All three extra fields are `readonly`, so an error cannot be altered after construction.

The `details` type (`ErrorDetails`, app-error.ts:1) is a union of a string-keyed record, an array, or undefined. This lets validation errors send an array of `{path, message}` objects while domain errors send objects such as `{from, to}`.

The constructor calls `Error.captureStackTrace?.(this, this.constructor)` (line 14), using optional chaining so it is safe on runtimes lacking the V8 API, and excludes the constructor frame from the stack. Because the central middleware branches on `instanceof AppError`, this class is the sole discriminator between expected (client-facing) errors and unexpected errors (which become HTTP 500 and are logged). The rule has high confidence.

**Rule workflow**:
```
throw new <Subclass>(...) -> AppError ctor sets fields -> errorMiddleware instanceof AppError
-> res.status(statusCode).json({error:{code,message,details?}})
```

---

### Business Rule: HTTP status mapping by error class

**Overview**:
Each subclass hard-codes an HTTP status and a default error code.

**Detailed description**:
The mapping is: BadRequest 400, Validation 400, Unauthorized 401, Forbidden 403, NotFound 404, Conflict 409, UnprocessableEntity 422. Subclasses with `message` defaults ('Bad request', 'Validation failed', 'Unauthorized', 'Forbidden') can be thrown without arguments, as done in controllers (`new UnauthorizedError()` at src/modules/orders/order.controller.ts:30, 40 and src/modules/auth/auth.controller.ts:32).

Customization is uneven. `BadRequestError`, `ConflictError` and `UnprocessableEntityError` accept a custom `code` and `details`. `ValidationError` accepts details but its code is fixed to VALIDATION_ERROR. `UnauthorizedError`, `ForbiddenError` and `NotFoundError` accept neither code nor details. `ConflictError` and `UnprocessableEntityError` require a message (no default), forcing callers to provide context, whereas the others have defaults.

Practical use: the 409 class is used for uniqueness collisions (EMAIL_ALREADY_USED in customer.service.ts:32,48 and user.service.ts:24; SKU_ALREADY_USED in product.service.ts:33,50) and state conflicts (INVALID_ORDER_STATE_FOR_DELETE at order.service.ts:185). The 422 class is used for semantically valid but unprocessable requests (INACTIVE_PRODUCT at order.service.ts:68). BadRequestError has no usages found in the project.

**Rule workflow**:
```
Caller picks class by failure category -> status fixed by class -> code default or caller-supplied
```

---

### Business Rule: NotFoundError message composition

**Overview**:
NotFoundError builds its message from a resource label.

**Detailed description**:
The constructor takes `resource` (default 'Resource') and produces "<resource> not found" with status 404 and code NOT_FOUND (http-errors.ts:27-31). Consumers pass entity names ('Order', 'Customer', 'Product', 'User') in services, and a descriptive string in the route fallback (`Route ${method} ${originalUrl}`, app.ts:70).

The code is identical across all entities, so clients cannot distinguish which resource was missing except by parsing the message. This is verified in tests/orders.test.ts:55-56 which asserts 404 and NOT_FOUND. Note also the order creation path throws a single 'Product' not-found when any requested product id is missing (order.service.ts:62-65) without identifying which ones.

The same code/status pair is produced by the middleware for Prisma P2025 (error.middleware.ts:48-52) with the generic message 'Resource not found', keeping the envelope consistent with this rule.

**Rule workflow**:
```
lookup returns null -> throw new NotFoundError('Entity') -> 404 {code:NOT_FOUND, message:'Entity not found'}
```

---

### Business Rule: Validation error format

**Overview**:
Request validation failures are reported as HTTP 400 with code VALIDATION_ERROR and a list of path/message issues.

**Detailed description**:
`ValidationError` defaults its message to 'Validation failed' and fixes the code (http-errors.ts:9-13). The validate middleware constructs it with details of shape `{ path: string, message: string }[]`, joining Zod issue paths with '.' and using '(root)' for empty paths (validate.middleware.ts:26-31). The service layer reuses the same class for business validation: an empty order ('Order must contain at least one item', order.service.ts:52) and a discount greater than subtotal, with a details entry `{path:'discountCents', message:'Discount exceeds subtotal'}` (order.service.ts:88-90).

The error middleware has a parallel path for raw ZodError (error.middleware.ts:26-35) producing an identical body, so the contract is the same whether validation is converted by the middleware or reaches the handler unconverted. Integration test tests/auth.test.ts:36-38 asserts 400, VALIDATION_ERROR and that details is an array. Note the order-empty case supplies no details, so details shape is not uniform (absent versus array).

**Rule workflow**:
```
Zod parse fails -> ValidationError(details[]) -> next(err) -> 400 {VALIDATION_ERROR, details}
Service rule fails -> throw ValidationError(msg[, details]) -> same response
```

---

### Business Rule: Authentication and authorization errors

**Overview**:
401 and 403 errors carry only a message and a fixed code.

**Detailed description**:
`UnauthorizedError` (401, UNAUTHORIZED) and `ForbiddenError` (403, FORBIDDEN) accept only a message. The auth middleware emits Unauthorized for a missing or malformed Authorization header, a missing bearer token, an invalid or expired token, and a missing user in the role check (auth.middleware.ts:30,36,45,52), and Forbidden for insufficient role permissions (line 56).

The auth service emits `UnauthorizedError('Invalid credentials')` for both unknown email and wrong password (auth.service.ts:34,38). Using the same message for both avoids user enumeration; this is an observed behavior, intent inferred (medium confidence). Controllers emit a bare `UnauthorizedError()` when `req.user` is absent.

Tests assert 401 and UNAUTHORIZED in tests/auth.test.ts:61-62 and 401 at line 82. No test asserting FORBIDDEN was found.

**Rule workflow**:
```
auth check fails -> next(new UnauthorizedError(msg)) / next(new ForbiddenError(msg)) -> 401/403 envelope
```

---

### Business Rule: Conflict errors with domain codes

**Overview**:
ConflictError returns 409 and lets callers define a specific code.

**Detailed description**:
`ConflictError` requires a message and defaults the code to CONFLICT (http-errors.ts:33-37). Consumers override the code for client-actionable conditions: EMAIL_ALREADY_USED (customers and users), SKU_ALREADY_USED (products), INVALID_STATUS_TRANSITION when the target status equals the current one (order.service.ts:141-145), and INVALID_ORDER_STATE_FOR_DELETE with details `{status}` (order.service.ts:185-189).

The generic CONFLICT code is also produced by the middleware for Prisma P2002 unique violations (error.middleware.ts:38-46), acting as a safety net if the pre-check races with a concurrent insert. The service-level pre-check codes therefore differ from the race-condition code for the same underlying condition.

Test coverage: tests/auth.test.ts:26-27 (EMAIL_ALREADY_USED) and tests/orders.test.ts:216-217 (INVALID_ORDER_STATE_FOR_DELETE). SKU_ALREADY_USED and the customer variant were not found in tests.

**Rule workflow**:
```
duplicate or bad state detected -> throw ConflictError(msg, CODE, details?) -> 409 envelope
```

---

### Business Rule: Invalid order status transition

**Overview**:
A disallowed order status change yields 409 INVALID_STATUS_TRANSITION with `{from, to}`.

**Detailed description**:
`InvalidStatusTransitionError` extends `ConflictError` and fixes the message to "Invalid status transition from X to Y", the code to INVALID_STATUS_TRANSITION and details to `{from, to}` (http-errors.ts:45-53). It is thrown in order.service.ts:148 when `canTransition(from, to)` is false. The same code is reused with a differently worded message through a plain `ConflictError` when from equals to (order.service.ts:141-145), so the code has two producers.

Because it subclasses ConflictError, `instanceof ConflictError` is also true for it, though no consumer relies on this. Since the throw occurs inside a Prisma transaction, no partial update is persisted. Test tests/orders.test.ts:89-106 asserts 409 and the code for PENDING to SHIPPED. The transition table itself (`canTransition`) is outside the component.

**Rule workflow**:
```
PATCH status -> load order -> from==to ? ConflictError : !canTransition ? InvalidStatusTransitionError -> 409
```

---

### Business Rule: Insufficient stock

**Overview**:
Lack of stock yields 422 INSUFFICIENT_STOCK with a list of unavailable products.

**Detailed description**:
`InsufficientStockError` extends `UnprocessableEntityError`, with fixed message 'One or more products do not have enough stock', code INSUFFICIENT_STOCK and details `{ unavailable: [{sku, requested, available}] }` (http-errors.ts:55-63). It is thrown at order.service.ts:223 after the service collects shortfalls (the loop that builds `unavailable` precedes it; exact aggregation logic is outside the component).

The error aggregates all failing SKUs rather than failing on the first, giving clients complete information in one response. Test tests/orders.test.ts:126-128 asserts 422, the code, and `details.unavailable[0].sku`. The type of the argument is declared inline rather than as a named exported type.

**Rule workflow**:
```
status change requiring stock -> collect shortfalls -> non-empty ? throw InsufficientStockError(list) -> 422
```

---

### Business Rule: Unprocessable entity for inactive products

**Overview**:
Ordering inactive products fails with 422 INACTIVE_PRODUCT.

**Detailed description**:
At order.service.ts:66-72 inactive products cause `UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', { skus })`. This shows the generic 422 class used with a custom code and details, as an alternative to defining a dedicated subclass (unlike InsufficientStock). No tests for this code were found. Confidence is high on behavior; the dedicated-versus-generic choice is inconsistent but unexplained.

**Rule workflow**:
```
products loaded -> filter !active -> any ? throw UnprocessableEntityError(..., 'INACTIVE_PRODUCT', {skus}) -> 422
```

---

### Business Rule: Conditional serialization of details and internal-error masking

**Overview**:
The envelope omits details when undefined; non-AppError errors are never leaked.

**Detailed description**:
In error.middleware.ts:20 `details` is spread into the body only when not undefined. Because `NotFoundError`, `UnauthorizedError` and `ForbiddenError` never set details, their responses contain only code and message. The constructor ensures undefined by default.

Any error that is not an AppError, ZodError or recognized Prisma error is logged with request id, method and path and answered with generic 500 INTERNAL_SERVER_ERROR (lines 56-64), so internal messages never reach clients. This rule resides in the consumer but is the behavioral consequence of the model: only errors extending `AppError` expose their message. Notably, `AppError.message` and `details` are sent verbatim, so callers are responsible for not including sensitive data.

**Rule workflow**:
```
err instanceof AppError ? serialize(code,message,details?) : (Zod/Prisma mapped or 500 generic)
```

---

## 4. Component Structure

```
src/shared/errors/
├── app-error.ts     # AppError base class and ErrorDetails type
├── http-errors.ts   # HTTP-mapped subclasses and two domain errors
└── index.ts         # Barrel: public exports
```

Related (outside boundary): src/middlewares/error.middleware.ts (translation to HTTP response).

## 5. Dependency Analysis

```
Internal Dependencies:
index.ts -> app-error.ts
index.ts -> http-errors.ts
http-errors.ts -> app-error.ts
(No dependency on any other project module)

External Dependencies:
- None (uses only the built-in Error class; Error.captureStackTrace is a V8 API used with optional chaining)
```

Dependents (afferent): auth.middleware.ts, validate.middleware.ts, error.middleware.ts, app.ts, auth.service.ts, auth.controller.ts, customer.service.ts, product.service.ts, user.service.ts, order.service.ts, order.controller.ts.

## 6. Afferent and Efferent Coupling

Afferent is the number of consumer files importing the class (from grep of src); efferent is the number of module-local classes it depends on.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| AppError | 1 direct file (error.middleware.ts) plus 9 subclasses | 0 | High |
| ErrorDetails (type) | 0 external (used by subclasses) | 0 | Low |
| BadRequestError | 0 | 1 | Low |
| ValidationError | 2 (validate.middleware.ts, order.service.ts) | 1 | Medium |
| UnauthorizedError | 4 (auth.middleware.ts, auth.service.ts, auth.controller.ts, order.controller.ts) | 1 | Medium |
| ForbiddenError | 1 (auth.middleware.ts) | 1 | Low |
| NotFoundError | 6 (app.ts, customer, product, user, order services) | 1 | Medium |
| ConflictError | 5 (customer, product, user, order services; base of InvalidStatusTransitionError) | 1 | Medium |
| UnprocessableEntityError | 1 (order.service.ts; base of InsufficientStockError) | 1 | Low |
| InvalidStatusTransitionError | 1 (order.service.ts) | 2 (ConflictError, AppError transitively) | Medium |
| InsufficientStockError | 1 (order.service.ts) | 2 (UnprocessableEntityError, AppError transitively) | Medium |

Note: ConflictError count includes order.service.ts, customer.service.ts, product.service.ts, user.service.ts (4 files) plus the internal subclass relationship; the table value of 5 counts that internal inheritance.

## 7. Endpoints

Not applicable: the component exposes no endpoints.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| errorMiddleware (src/middlewares/error.middleware.ts) | Internal consumer | Serialize AppError into HTTP responses | Express error handler | JSON `{error:{code,message,details?}}` | instanceof AppError, then Zod, Prisma, generic 500 |
| validate.middleware.ts | Internal producer | Convert ZodError into ValidationError | Express middleware | `{path,message}[]` | Non-Zod errors forwarded via next(err) |
| auth.middleware.ts | Internal producer | Emit 401/403 | Express middleware | Message string | next(err) |
| Service layer (auth, customer, product, user, order) | Internal producer | Emit domain errors | Function calls (throw) | Message, code, details | Throw inside Prisma transactions aborts them |
| Prisma errors (P2002, P2025) | Parallel external error source | Mapped to 409/404 by middleware, not by this component | In-process | Prisma error objects | Mapped to CONFLICT / NOT_FOUND |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Exception hierarchy / Custom error classes | AppError and subclasses | src/shared/errors/app-error.ts, http-errors.ts | Typed, discriminable errors |
| Facade / Barrel module | index.ts | src/shared/errors/index.ts | Single import surface |
| Centralized error handling | errorMiddleware | src/middlewares/error.middleware.ts:14 | One translation point to HTTP |
| Value object (immutable error data) | readonly fields | app-error.ts:4-6 | Immutability of error info |
| Specialization by inheritance | InvalidStatusTransitionError extends ConflictError; InsufficientStockError extends UnprocessableEntityError | http-errors.ts:45,55 | Domain errors reuse HTTP semantics |
| Error code convention | UPPER_SNAKE_CASE string codes | http-errors.ts | Stable client-facing identifiers |

Architectural note: the model combines transport semantics (HTTP status) with domain errors in one shared module, so the domain layer depends on HTTP status codes (ambiguous whether intentional layering choice; the code reflects it directly).

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Test coverage | No unit tests for the error classes (defaults, status, codes, inheritance) | Regressions in mapping detected only via integration tests |
| Medium | Error codes | Codes are free string literals across files (INVALID_STATUS_TRANSITION has two producers; EMAIL_ALREADY_USED, SKU_ALREADY_USED duplicated in several services) | Drift risk, no single registry |
| Medium | AppError.name | Subclasses do not set name; all report 'AppError' | Logs/stack traces do not show specific class name |
| Medium | Domain coupling | Order-domain errors reside in shared module; HTTP status embedded in domain-level errors | Shared module not domain-neutral |
| Low | API inconsistency | ValidationError has fixed code; Unauthorized/Forbidden/NotFound accept no code or details | Uneven extensibility |
| Low | Unused class | BadRequestError has no usages found | Dead code candidate |
| Low | Inconsistent pattern | Dedicated subclass for stock but generic 422 for INACTIVE_PRODUCT | Uneven modeling |
| Low | Details shape | details varies (array of path/message, object, absent) | Client parsing complexity |
| Low | Message exposure | message and details sent verbatim to clients | Depends on caller discipline |
| Low | Duplicate logic | Zod issue formatting duplicated in validate.middleware.ts:26 and error.middleware.ts:7 | Maintenance duplication |
| Low | NotFoundError | Single code for all resources; order creation reports 'Product' not found without identifying ids | Less precise client diagnostics |
| Low | Inline type | InsufficientStockError argument type is inline, not exported | Limited reuse |

## 11. Test Coverage Analysis

Test files located: tests/auth.test.ts, tests/orders.test.ts, tests/setup.ts, tests/helpers/factories.ts. Config: vitest.config.ts (include tests/**/*.test.ts; single fork; no coverage provider configured). No test files exist under src/shared/errors. No coverage report was found, so percentages are not available.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| AppError | 0 | Indirect | Not measured | Exercised through every error response; fields not directly asserted |
| ValidationError | 0 | 1 (tests/auth.test.ts:36-38) | Not measured | Asserts 400, code, details is array; order-level validation not tested |
| UnauthorizedError | 0 | 2 (tests/auth.test.ts:61-62, 82) | Not measured | Code asserted once |
| ForbiddenError | 0 | 0 found | Not measured | Gap |
| NotFoundError | 0 | 1 (tests/orders.test.ts:55-56) | Not measured | Message text not asserted |
| ConflictError | 0 | 2 (tests/auth.test.ts:26-27; tests/orders.test.ts:216-217) | Not measured | SKU_ALREADY_USED and customer email codes not tested |
| UnprocessableEntityError | 0 | 0 found (INACTIVE_PRODUCT) | Not measured | Gap |
| InvalidStatusTransitionError | 0 | 1 (tests/orders.test.ts:105-106) | Not measured | Asserts status and code; details {from,to} not asserted |
| InsufficientStockError | 0 | 1 (tests/orders.test.ts:126-128) | Not measured | Asserts details.unavailable[0].sku; requested/available not asserted |
| BadRequestError | 0 | 0 | Not measured | Unused and untested |
| Barrel (index.ts) | 0 | Indirect | Not measured | Verified implicitly by imports |
