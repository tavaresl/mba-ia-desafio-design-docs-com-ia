# Component Deep Analysis Report: Error Middleware

Component file: `src/middlewares/error.middleware.ts` (65 lines, single exported symbol `errorMiddleware`)
Analysis date: 2026-10-29 14:55:48

## 1. Executive Summary

The Error Middleware is the centralized Express 4 error-handling middleware (`ErrorRequestHandler`, four-argument signature) of the Order Management API. It is the last middleware registered in `src/app.ts:73`, after the router (`src/app.ts:67`) and a catch-all 404 forwarder (`src/app.ts:69-71`). Every controller catches exceptions and forwards them with `next(err)`; validation and auth middlewares also call `next(new XError(...))`. The Error Middleware converts any such error into a uniform JSON envelope `{ error: { code, message, details? } }` with an appropriate HTTP status.

Key findings:
- Four translation branches in strict order: `AppError` (domain/HTTP errors), `ZodError`, `Prisma.PrismaClientKnownRequestError` (only P2002 and P2025), and a generic 500 fallback.
- Only the 500 fallback logs the error (pino, with request id, method, path). Errors handled by the first three branches are not logged by this component (the request logger logs only status code on `finish`).
- Internal error details are never returned to the client for 500s (message is fixed), which limits information disclosure.
- Prisma errors other than P2002/P2025 (for example P2003 foreign key, P2000, P2034), `PrismaClientValidationError`, JSON body-parse errors from `express.json` (a `SyntaxError` with `status` 400) and payload-too-large errors (`status` 413) fall through to the 500 branch.
- There is no dedicated test for the middleware; it is exercised only indirectly through integration tests (`tests/auth.test.ts`, `tests/orders.test.ts`) that cover AppError subclasses and Zod-derived validation errors via `validate.middleware`.

## 2. Data Flow Analysis

```
1. Request enters Express app (src/app.ts:56-60): express.json (1mb limit) then requestLogger
   (assigns req.id from x-request-id header or uuid v4, sets X-Request-Id response header).
2. Route-level middlewares (validate, authenticate, authorize) and controllers run.
   - validate.middleware.ts:25-35 converts ZodError into ValidationError and calls next(err).
   - auth.middleware.ts calls next(UnauthorizedError / ForbiddenError).
   - Controllers wrap handlers in try/catch and call next(err) (e.g. customer.controller.ts:13-14).
   - Services throw NotFoundError, ConflictError, UnprocessableEntityError, InvalidStatusTransitionError,
     InsufficientStockError, ValidationError; repositories may surface Prisma errors.
3. Unmatched routes: app.ts:69-71 calls next(new NotFoundError("Route <METHOD> <url>")).
4. Express skips non-error middlewares and invokes errorMiddleware(err, req, res, _next) (app.ts:73).
5. Branch selection (error.middleware.ts):
   a. err instanceof AppError           -> status = err.statusCode, body from errorCode/message/details (L15-24)
   b. err instanceof ZodError           -> 400 VALIDATION_ERROR with formatted issues (L26-35)
   c. PrismaClientKnownRequestError
        P2002 -> 409 CONFLICT (target columns in message) (L38-47)
        P2025 -> 404 NOT_FOUND (L48-53)
        other codes -> fall through
   d. Fallback: resolve requestId (req.id or 'unknown'), logger.error with err, requestId, method,
      originalUrl (L56-60), respond 500 INTERNAL_SERVER_ERROR (L62-64).
6. Response is serialized as JSON; requestLogger's 'finish' listener logs http_request with the status code.
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Error mapping | Known application errors (`AppError` and subclasses) are returned with their own status code, error code, message and optional details | src/middlewares/error.middleware.ts:15-24 |
| Response contract | All error responses use the envelope `{ error: { code, message, details? } }`; `details` omitted when undefined | src/middlewares/error.middleware.ts:17-21 |
| Validation mapping | Raw `ZodError` is mapped to HTTP 400, code `VALIDATION_ERROR`, message `Validation failed`, details list of `{path, message}` | src/middlewares/error.middleware.ts:26-35 |
| Formatting | Zod issue path joined by `.`; empty path shown as `(root)` | src/middlewares/error.middleware.ts:7-12 |
| Persistence mapping | Prisma P2002 (unique violation) mapped to 409 `CONFLICT`, listing target fields when available | src/middlewares/error.middleware.ts:38-47 |
| Persistence mapping | Prisma P2025 (record not found) mapped to 404 `NOT_FOUND`, message `Resource not found` | src/middlewares/error.middleware.ts:48-53 |
| Fallback | Anything unrecognized yields 500 `INTERNAL_SERVER_ERROR` with generic message | src/middlewares/error.middleware.ts:62-64 |
| Observability | Only unhandled errors are logged at error level with requestId, method, path | src/middlewares/error.middleware.ts:56-60 |
| Security | Internal error message/stack never sent to client on 500 path; logger redacts sensitive keys | src/middlewares/error.middleware.ts:62-64; src/shared/logger/index.ts:4-11 |
| Ordering | Branch precedence: AppError, ZodError, Prisma; first match wins and returns | src/middlewares/error.middleware.ts:15-54 |
| Registration | Must be registered last, after routes and 404 forwarder | src/app.ts:69-73 |

### Detailed breakdown of the business rules

---

### Business Rule: AppError translation (domain errors to HTTP)

**Overview**:
Any thrown instance of `AppError` is serialized using the status code, error code, message and details that the error itself carries.

**Detailed description**:
`AppError` (src/shared/errors/app-error.ts:3-16) carries `statusCode`, `errorCode` and optional `details`. Subclasses in src/shared/errors/http-errors.ts define the catalog: BadRequestError (400 BAD_REQUEST), ValidationError (400 VALIDATION_ERROR), UnauthorizedError (401 UNAUTHORIZED), ForbiddenError (403 FORBIDDEN), NotFoundError (404 NOT_FOUND, message `<resource> not found`), ConflictError (409 CONFLICT by default), UnprocessableEntityError (422), InvalidStatusTransitionError (409 INVALID_STATUS_TRANSITION, details `{from,to}`) and InsufficientStockError (422 INSUFFICIENT_STOCK, details `{unavailable:[{sku,requested,available}]}`). The middleware does not know these subclasses; it relies on polymorphism through the base class, so new subclasses are handled automatically.

The business rules of the domain (stock check, order status machine, duplicate email/SKU, order deletion state) are enforced in services and surface to clients only through this translation. Custom codes used by services include EMAIL_ALREADY_USED (user.service.ts:24, customer.service.ts:32,48), SKU_ALREADY_USED (product.service.ts:33,50) and INVALID_ORDER_STATE_FOR_DELETE (order.service.ts:185, per tests/orders.test.ts:217).

The `details` key is included only when `err.details !== undefined` (line 20), so errors without details produce a body with just `code` and `message`. Messages of AppError are exposed verbatim, so developers must ensure they do not contain sensitive information (for example, the 404 route message echoes the method and original URL, app.ts:70).

**Rule workflow**:
```
err instanceof AppError?
  yes -> res.status(err.statusCode).json({error:{code:err.errorCode, message:err.message, [details]}}) ; return
  no  -> continue to next branch
```

---

### Business Rule: Zod validation error translation

**Overview**:
A `ZodError` reaching the middleware directly is mapped to a 400 response using the same shape as `ValidationError`.

**Detailed description**:
In the normal request pipeline `validate.middleware.ts:25-33` already converts `ZodError` into `ValidationError` with details of the same format (`path` joined with `.`, `(root)` for empty path). The middleware branch at lines 26-35 is therefore a safety net for ZodErrors thrown elsewhere, such as `schema.parse` calls in controllers or services (not confirmed whether any exist; no such call was verified in this analysis).

The format of the details array is `[{ path: string, message: string }]`. Path elements that are numbers (array indices) are joined using `.` (for example `items.0.quantity`). Only `path` and `message` are exposed; Zod issue `code`, `expected`, `received` are dropped. The formatting logic is duplicated in two places (formatZodIssues and validate.middleware.ts:27-30).

Because the code and message are literal (`VALIDATION_ERROR`, `Validation failed`), clients see a consistent result whichever path produced the error. This is confirmed by tests/auth.test.ts:36-38 (400, VALIDATION_ERROR, details is an array), which goes through validate.middleware and thus the AppError branch.

**Rule workflow**:
```
err instanceof ZodError?
  yes -> issues.map({path: issue.path.join('.') || '(root)', message}) -> 400 VALIDATION_ERROR ; return
```

---

### Business Rule: Prisma unique constraint violation (P2002) to 409

**Overview**:
Database unique violations that were not pre-empted by service-level checks are returned as 409 CONFLICT.

**Detailed description**:
When Prisma raises `PrismaClientKnownRequestError` with code `P2002`, the middleware reads `err.meta.target` (cast to `string[] | undefined`) and joins it with `, `. If present the message is `Unique constraint violation on: <fields>`, otherwise `Unique constraint violation`. This handles race conditions where two concurrent requests pass the service-level pre-check (for example email or SKU lookup before insert) and the database constraint rejects the second.

The error code is the generic `CONFLICT`, unlike the service-level pre-checks which return specific codes (EMAIL_ALREADY_USED, SKU_ALREADY_USED). Clients can thus receive different codes for the same logical conflict depending on timing. Exposing column names in the message reveals schema field names (for example `email`, `sku`) to clients. The cast of `meta.target` assumes an array; for some Prisma connectors the target may be a string, in which case `.join` would throw (not applicable to the PostgreSQL provider presumed from docker-compose; provider not verified here).

No test exercises this branch directly.

**Rule workflow**:
```
err is PrismaClientKnownRequestError and code == P2002
  -> target = meta.target?.join(', ')
  -> 409 {code: CONFLICT, message: target ? "Unique constraint violation on: <target>" : "Unique constraint violation"}
```

---

### Business Rule: Prisma record not found (P2025) to 404

**Overview**:
Operations on a missing record (update/delete of non-existent row) are returned as 404 NOT_FOUND.

**Detailed description**:
P2025 is raised by Prisma when an operation depends on records that do not exist, for example `update` or `delete` on a missing id. The middleware answers 404 with code `NOT_FOUND` and fixed message `Resource not found`, which does not identify the resource type (unlike `NotFoundError`, which yields `<Resource> not found`).

Services typically check existence first and throw NotFoundError (for example order.service.ts:46,136,183), so this branch is a fallback for races between existence check and write.

Other Prisma known error codes (for example P2003 foreign key violation, P2014 relation violation) do not match either `if` and fall through to the generic 500 path, including being logged as unhandled. No test covers this branch directly.

**Rule workflow**:
```
err is PrismaClientKnownRequestError and code == P2025
  -> 404 {code: NOT_FOUND, message: "Resource not found"} ; return
```

---

### Business Rule: Unhandled error fallback (500) and logging

**Overview**:
Unrecognized errors are logged with context and answered with an opaque 500 response.

**Detailed description**:
If no earlier branch returned, the middleware computes `requestId` from `(req as {id?: string}).id`, falling back to the string `'unknown'` (the id is normally set by requestLogger at request-logger.middleware.ts:6-7). It logs with `logger.error({ err, requestId, method, path: req.originalUrl }, 'Unhandled error in request')`. The pino logger serializes `err` using its standard error serializer (stack included) and applies redaction for paths such as `*.password`, `*.token`, `req.headers.authorization` (logger/index.ts:4-11). Note the redact path `req.headers.authorization` does not apply to the logged object because the middleware does not log a `req` property; redaction is relevant only to `*.xxx` keys one level deep.

The response is always `500 { error: { code: 'INTERNAL_SERVER_ERROR', message: 'Internal server error' } }`. Because the response status is fixed, errors that carry their own HTTP status but are not AppError (for example body-parser `SyntaxError` with status 400 from malformed JSON, or 413 payload too large with the 1mb limit at app.ts:59) are reported as 500 and logged as errors. This is observed from code; actual behavior was not executed.

The middleware does not check `res.headersSent`; if headers were already sent when an error occurs, calling `res.status().json()` would throw `ERR_HTTP_HEADERS_SENT` rather than delegating to Express's default handler via `next(err)`. The `_next` parameter is unused.

**Rule workflow**:
```
no previous branch matched
  -> requestId = req.id ?? 'unknown'
  -> logger.error({err, requestId, method, path}, 'Unhandled error in request')
  -> 500 {code: INTERNAL_SERVER_ERROR, message: "Internal server error"}
```

---

### Business Rule: Uniform error response contract

**Overview**:
Every error response from the API shares the structure `{ error: { code, message, details? } }`.

**Detailed description**:
All four branches build the body with a top-level `error` object holding a machine-readable `code` (upper snake case), a human-readable `message`, and optionally `details` (array of validation issues, object with transition data, or stock shortfall). Clients can branch on `code` rather than parsing messages. Tests assert this contract, for example `res.body.error.code` in tests/auth.test.ts:27,37,62 and tests/orders.test.ts:56,106,127,217, and `res.body.error.details.unavailable[0].sku` in tests/orders.test.ts:128.

The `ErrorDetails` type permits an object, array or undefined (app-error.ts:1). The Prisma branches never include details. The success-path envelope is separate (src/shared/http/response.ts, not analyzed in depth).

**Rule workflow**:
```
any error -> errorMiddleware -> JSON {error:{code,message,[details]}} with HTTP status by branch
```

---

### Business Rule: Registration order and 404 for unknown routes

**Overview**:
Unknown routes become a NotFoundError through a catch-all, then handled by this middleware.

**Detailed description**:
In `buildApp` the sequence is: json parser, requestLogger, `/health`, `/api/v1` router, 404 forwarder, errorMiddleware. Because Express error middlewares only receive errors from middlewares registered before them, errorMiddleware must come last. Errors raised by `express.json` and `requestLogger` (registered earlier) are also caught. The `/health` route and routes registered after would not be covered; none exist.

The 404 message is `Route <METHOD> <originalUrl> not found` (NotFoundError appends ` not found`), echoing user-supplied URL data in the response (JSON-encoded, so not an XSS vector in JSON context).

**Rule workflow**:
```
unmatched request -> app.use((req,_res,next)=>next(new NotFoundError(...))) -> errorMiddleware -> 404 NOT_FOUND
```

---

## 4. Component Structure

```
src/
├── app.ts                              # Registers errorMiddleware last (L73) and 404 forwarder (L69-71)
├── middlewares/
│   ├── error.middleware.ts             # COMPONENT: errorMiddleware + formatZodIssues (private)
│   ├── validate.middleware.ts          # Produces ValidationError (forwards to component)
│   ├── auth.middleware.ts              # Produces UnauthorizedError / ForbiddenError
│   └── request-logger.middleware.ts    # Sets req.id and X-Request-Id used by component
└── shared/
    ├── errors/
    │   ├── app-error.ts                # AppError base class, ErrorDetails type
    │   ├── http-errors.ts              # Error subclasses catalog
    │   └── index.ts                    # Barrel export
    └── logger/
        └── index.ts                    # pino logger with redaction
tests/
├── auth.test.ts                        # Indirect coverage (409, 400, 401)
├── orders.test.ts                      # Indirect coverage (404, 409, 422)
├── setup.ts                            # DB cleanup/connect
└── helpers/factories.ts
```

Component internals: one private helper `formatZodIssues` (L7-12) and one exported constant `errorMiddleware` (L14-65). No state.

## 5. Dependency Analysis

```
Internal Dependencies:
errorMiddleware -> AppError (src/shared/errors/index.ts -> app-error.ts)
errorMiddleware -> logger (src/shared/logger/index.ts -> config/env.ts)
errorMiddleware -> req.id (set by request-logger.middleware.ts; runtime coupling only, accessed via type cast)
app.ts -> errorMiddleware
Controllers/services/validate/auth middlewares -> AppError subclasses -> (handled by) errorMiddleware (runtime, via next(err))

External Dependencies:
- express 4.21.1 (ErrorRequestHandler type; error-handling middleware contract)
- @prisma/client 5.22.0 (Prisma.PrismaClientKnownRequestError)
- zod 3.23.8 (ZodError)
- pino 9.5.0 (via logger)
```

## 6. Afferent and Efferent Coupling

Counts are based on source imports/usages found in `src`.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| errorMiddleware | 1 (app.ts) compile-time; all handlers at runtime through next(err) | 5 (AppError, ZodError, Prisma, logger, express types) | High |
| formatZodIssues | 1 (errorMiddleware) | 1 (ZodError) | Low |
| AppError | 3 import sites in src/shared/errors/index.ts, http-errors.ts, error.middleware.ts | 0 | High |
| logger | Multiple (errorMiddleware, requestLogger, others) | 2 (pino, env) | Medium |
| validate (middleware) | Route files | 2 (zod, ValidationError) | Medium |

## 7. Endpoints

The component exposes no endpoints of its own. It indirectly defines the error responses of all endpoints and of unmatched routes.

Error response summary:

| Condition | HTTP Status | code |
|-----------|-------------|------|
| AppError subclass | Per error (400, 401, 403, 404, 409, 422) | Per error |
| ZodError | 400 | VALIDATION_ERROR |
| Prisma P2002 | 409 | CONFLICT |
| Prisma P2025 | 404 | NOT_FOUND |
| Anything else | 500 | INTERNAL_SERVER_ERROR |

(Per the report guidelines, no standard endpoint table is included since the component has no routes.)

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express error pipeline | Framework | Receives errors forwarded with next(err) | In-process | JS Error objects | Last-resort; no `headersSent` check |
| Prisma Client | Library (database layer) | Recognizes known request errors | In-process types | PrismaClientKnownRequestError (code, meta) | Only P2002, P2025 mapped |
| Zod | Library | Recognizes validation errors | In-process types | ZodError.issues | Mapped to 400 |
| pino logger | Internal shared module | Records unhandled errors | In-process; stdout (pino-pretty in development) | JSON log line | Sensitive-key redaction |
| HTTP clients | Consumers | Receive JSON error envelope | HTTP/JSON | `{error:{code,message,details?}}` | Fixed generic message on 500 |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Centralized error handling (Express error middleware) | `ErrorRequestHandler` registered last | src/middlewares/error.middleware.ts:14; src/app.ts:73 | Single place for error-to-response mapping |
| Exception hierarchy | AppError and subclasses | src/shared/errors/*.ts | Carry HTTP status and domain code with the exception |
| Chain of Responsibility / ordered type dispatch | Sequential instanceof branches with early return | error.middleware.ts:15-54 | Select translation by error type |
| Adapter / Anti-corruption | Prisma and Zod errors translated to API contract | error.middleware.ts:26-53 | Keep library errors out of client contract |
| Fail-safe default | Generic 500 | error.middleware.ts:62-64 | Avoid leaking internals |
| Dependency Injection | Not used; logger imported as module singleton | error.middleware.ts:5 | Noted as a testability consideration |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Fallback branch | Errors with their own HTTP status (body-parser SyntaxError 400, payload too large 413) and not derived from AppError are returned as 500 | Client errors misreported as server errors and logged as unhandled |
| Medium | Fallback branch | No `res.headersSent` check and `_next` unused | If headers are already sent, handler may throw instead of delegating to Express default handler |
| Medium | Prisma mapping | Only P2002 and P2025 mapped; others (e.g. P2003) become 500 | Inconsistent client-facing errors for DB constraint failures |
| Medium | Test coverage | No unit or direct integration tests for Zod, P2002, P2025, 500 fallback branches | Regressions undetected in untested branches |
| Low | Logging | Handled errors (4xx via AppError/Zod/Prisma mapped) are not logged by this component | Reduced diagnostic detail for 4xx beyond the status in http_request log |
| Low | Duplication | Zod issue formatting duplicated in validate.middleware.ts:27-30 and error.middleware.ts:7-12 | Divergence risk |
| Low | Information disclosure | P2002 message exposes column names; 404 route message echoes URL | Minor schema/URL disclosure |
| Low | Code consistency | P2002 returns generic CONFLICT while service-level checks return specific codes (EMAIL_ALREADY_USED, SKU_ALREADY_USED) | Different codes for same logical conflict depending on race timing |
| Low | Type safety | `meta.target` cast to `string[]`, `req` cast for `id` | Runtime assumption; possible throw if target is a string |
| Low | Logging config | `req.headers.authorization` redact path does not match the logged object shape | No impact currently since `req` is not logged |

## 11. Test Coverage Analysis

Test framework: Vitest 2.1.4 with Supertest 7.0.0 (package.json), config in vitest.config.ts (`tests/**/*.test.ts`, single fork, sequential, real database via tests/setup.ts). No coverage tool or threshold configuration found; numeric coverage cannot be stated. No test file targets `error.middleware.ts` directly. The project contains 17 `it`/`describe` entries in total across two test files.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| AppError branch | 0 | Yes, indirect: tests/auth.test.ts:26-27 (409 EMAIL_ALREADY_USED), :61-62 (401 UNAUTHORIZED); tests/orders.test.ts:55-56 (404 NOT_FOUND), :105-106 (409 INVALID_STATUS_TRANSITION), :127-128 (422 INSUFFICIENT_STOCK with details), :216-217 (409) | Not measured (branch covered by execution) | Asserts status, code and details; message text not asserted |
| ZodError branch (direct) | 0 | 0 (the validation test at tests/auth.test.ts:36-38 goes through ValidationError/AppError branch) | Likely not executed | Not tested |
| Prisma P2002 branch | 0 | 0 | Likely not executed | Not tested |
| Prisma P2025 branch | 0 | 0 | Likely not executed | Not tested |
| 500 fallback + logging | 0 | 0 | Likely not executed | Not tested |
| 404 unknown route (app.ts:69-71) | 0 | Not found in tests | Unknown | Not tested |

Test files located: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `vitest.config.ts`. Execution was not performed, so coverage statements are inferred from reading test assertions.
