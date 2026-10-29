# Component Deep Analysis Report: Auth Middleware

Component: Auth Middleware (`authenticate`, `requireRole`)
Primary file: `src/middlewares/auth.middleware.ts` (61 lines)
Date: 2026-10-29 14:55:48

## 1. Executive Summary

The Auth Middleware is the single authentication and authorization gate of the Express 4.21.1 REST API (base path `/api/v1`). It exports two Express `RequestHandler` units:

- `authenticate` (`src/middlewares/auth.middleware.ts:27-47`): extracts a Bearer JWT from the `Authorization` header, verifies it with `jsonwebtoken` 9.0.2 and `env.JWT_SECRET`, and populates `req.user` (`id`, `email`, `role`) from the token claims, without a database lookup.
- `requireRole(...roles)` (`src/middlewares/auth.middleware.ts:49-61`): a factory that returns a handler. It checks that `req.user.role` is among the allowed roles (RBAC).

It also exports the `AuthUser` type and augments `express-serve-static-core`'s `Request` with `user?` and `id?` (lines 12-17).

Key findings:

- Stateless authentication: the identity and role come only from the signed token. A deleted or role-changed user keeps the old privileges until the token expires (default 8h, `src/config/env.ts:9`).
- All failures are delegated through `next(err)` to `errorMiddleware` and produce a 401 (`UNAUTHORIZED`) or 403 (`FORBIDDEN`) JSON envelope.
- `authenticate` is applied to every business router (customers, products, orders via `router.use`; `/auth/me` and `/users/:id` per route). `requireRole` is used in only one place (`GET /users/:id`, ADMIN only). Roles: `ADMIN`, `OPERATOR`.
- Notable risks: `POST /auth/register` accepts a client-supplied `role` (`src/modules/auth/auth.schemas.ts:7`), which allows self-registration as ADMIN (assuming the service does not override it; `UserService.createUser` was not inspected). The JWT algorithm is not pinned in `jwt.verify`. There is no direct unit test for `requireRole`, expired or tampered tokens, or the 403 path.

## 2. Data Flow Analysis

authenticate flow:

```
1. Request enters the router (e.g. /api/v1/orders) after app-level middleware (src/app.ts:67 mounts buildApiRouter at /api/v1)
2. router.use(authenticate) or per-route authenticate is invoked
3. Read req.headers.authorization (auth.middleware.ts:28)
4. Scheme check: header present and lowercased starts with "bearer " (line 29) -> else UnauthorizedError('Missing or invalid Authorization header')
5. Token = header.slice(7).trim() (line 34); empty -> UnauthorizedError('Missing bearer token')
6. jwt.verify(token, env.JWT_SECRET) (line 41); signature, exp, nbf checked by the library
7. On success: req.user = { id: payload.sub, email: payload.email, role: payload.role } (line 42); next()
8. On any thrown error: UnauthorizedError('Invalid or expired token') (line 45)
9. next(err) reaches errorMiddleware (src/middlewares/error.middleware.ts:14-24) -> HTTP 401 {error:{code:'UNAUTHORIZED',message}}
10. Downstream handlers read req.user (auth.controller.ts:32, order.controller.ts:30-41)
```

requireRole flow:

```
1. Executed after authenticate in the route chain (user.routes.ts:14-15)
2. If req.user is undefined -> UnauthorizedError() (default message 'Unauthorized') (lines 51-54)
3. If req.user.role not in roles -> ForbiddenError('Insufficient permissions') (lines 55-58)
4. Otherwise next()
5. Errors rendered by errorMiddleware: 401 UNAUTHORIZED or 403 FORBIDDEN
```

Token origin (outside the component, for context): `AuthService.login` signs `{ sub, email, role }` with `env.JWT_SECRET` and `expiresIn = env.JWT_EXPIRES_IN` (`src/modules/auth/auth.service.ts:47-50`), returned as `{ tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }`.

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authentication | Authorization header must exist and start with "Bearer " (case-insensitive scheme) | src/middlewares/auth.middleware.ts:29 |
| Validation | Bearer token must be non-empty after trimming | src/middlewares/auth.middleware.ts:34-38 |
| Authentication | Token must be a valid JWT signed with JWT_SECRET and not expired | src/middlewares/auth.middleware.ts:41,44-46 |
| Identity propagation | req.user is built from claims sub, email, role (no DB lookup) | src/middlewares/auth.middleware.ts:42 |
| Authorization | requireRole denies unauthenticated requests with 401 | src/middlewares/auth.middleware.ts:51-54 |
| Authorization | requireRole allows only roles in the allowed list; otherwise 403 | src/middlewares/auth.middleware.ts:55-58 |
| Domain constraint | Only roles ADMIN and OPERATOR exist | src/middlewares/auth.middleware.ts:9 |
| Configuration | JWT_SECRET min 16 chars; JWT_EXPIRES_IN default 8h | src/config/env.ts:8-9 |
| Policy application | All customers, products, orders endpoints require authentication; /users/:id requires ADMIN | src/modules/*/\*.routes.ts |
| Error contract | Failures map to 401 UNAUTHORIZED / 403 FORBIDDEN envelope | src/shared/errors/http-errors.ts:15-25; src/middlewares/error.middleware.ts:15-24 |

### Detailed breakdown of the business rules

---

### Business Rule: Bearer header requirement

**Overview**:
Every protected request must carry `Authorization: Bearer <token>`. Anything else is rejected with 401 before token parsing.

**Detailed description**:
The middleware reads `req.headers.authorization` (Node lowercases header names). If the header is absent or does not begin with `bearer ` after `toLowerCase()`, it calls `next(new UnauthorizedError('Missing or invalid Authorization header'))` (line 29-32). The scheme match is therefore case-insensitive ("Bearer", "bearer", "BEARER"), and requires exactly one space after the scheme in the prefix test. Other schemes (Basic, Token) are rejected.

The check covers the case where a client sends no credentials at all, and the case where a different scheme is used. The test `rejects /auth/me without token` (tests/auth.test.ts:79-83) exercises the missing-header path and asserts only status 401, not the message.

The rule protects all downstream handlers: because the middleware short-circuits with `return` after `next(err)`, controllers never execute without a valid scheme. Ambiguity: a header such as "Bearer" without trailing space also fails this branch (message "Missing or invalid Authorization header"), not the "Missing bearer token" branch.

**Rule workflow**:
```
header absent OR not startsWith('bearer ') -> 401 "Missing or invalid Authorization header"
else -> continue to token extraction
```

---

### Business Rule: Non-empty bearer token

**Overview**:
A header like "Bearer    " (scheme with only whitespace after) is rejected with a specific message.

**Detailed description**:
After passing the prefix check, the token is `header.slice(7).trim()` (line 34). Slicing 7 characters removes "Bearer " irrespective of its letter case. The trim removes surrounding whitespace, tolerating extra spaces between scheme and token.

If the result is the empty string, `UnauthorizedError('Missing bearer token')` is produced (lines 35-38). Note that HTTP servers typically trim trailing whitespace from header values, so this branch is reachable mainly with a value like "Bearer  " in non-normalizing clients. No test covers it.

The rule distinguishes malformed header from empty credential for diagnosability; both yield identical HTTP status and error code (401 UNAUTHORIZED).

**Rule workflow**:
```
token = header.slice(7).trim()
token === '' -> 401 "Missing bearer token"
else -> verify
```

---

### Business Rule: JWT signature and expiry verification

**Overview**:
The token must be a JWT verifiable with the shared secret and not expired; any failure yields one generic 401.

**Detailed description**:
`jwt.verify(token, env.JWT_SECRET)` (line 41) validates signature, `exp` and `nbf` claims. Algorithm is not specified in options, so jsonwebtoken's default allowed algorithms for a string secret (HMAC family HS256/HS384/HS512) apply. The result is cast to `JwtPayload` (`as`, no runtime schema validation of `sub`, `email`, `role`).

Any exception (malformed token, bad signature, `TokenExpiredError`, `NotBeforeError`) is swallowed by a bare `catch` and replaced by `UnauthorizedError('Invalid or expired token')` (lines 44-46). The original cause is neither logged nor distinguished, which prevents leaking information to clients but removes observability. Because `next()` is called inside the `try` (line 43), an exception thrown synchronously by downstream sync handlers invoked via `next()` could theoretically be caught and mislabeled as 401; in Express 4 the downstream error handling of `next()` is internally wrapped by Layer, so this is a low-probability observation (confidence: medium).

Tokens are issued by `AuthService.signToken` with the same secret and `JWT_EXPIRES_IN` (default 8h). No refresh, revocation or blacklist exists in the component. Secret strength is enforced at startup by `env.ts:8` (min 16 characters), exiting the process on invalid config.

**Rule workflow**:
```
jwt.verify OK -> req.user populated -> next()
jwt.verify throws -> 401 "Invalid or expired token"
```

---

### Business Rule: Identity propagation from claims

**Overview**:
The authenticated principal is reconstructed from the JWT claims only.

**Detailed description**:
On success, `req.user = { id: payload.sub, email: payload.email, role: payload.role }` (line 42). The `Request` interface is augmented (lines 12-17) so controllers have typed access to `req.user` (type `AuthUser`). Controllers rely on it: `AuthController.me` throws `UnauthorizedError` if absent and fetches the user by `req.user.id` (auth.controller.ts:30-34); `OrderController` passes `req.user.id` as actor for create and status changes (order.controller.ts:30-41), which feeds order status history.

Because no database lookup occurs, the role and identity are trusted as of token issue time. A user deleted or demoted after login remains authorized until expiry, except where a downstream handler loads the user (`/auth/me` returns not found via UserService, presumably). This is an implicit rule, confidence: high (code evidence), consequence: inferred.

The `id?: string` augmentation on Request is declared here but set elsewhere (request-logger middleware, not part of this component).

**Rule workflow**:
```
valid token -> req.user = {id: sub, email, role} -> next()
controllers: if (!req.user) -> UnauthorizedError (defensive check)
```

---

### Business Rule: Role-based access control (requireRole)

**Overview**:
`requireRole(...roles)` restricts a route to principals whose role is in the allowed list.

**Detailed description**:
It is a higher-order function returning a handler. First guard: if `req.user` is missing (authenticate was not run before it), it answers 401 with default message "Unauthorized" (lines 51-54), failing closed on misconfigured route chains. Second guard: `roles.includes(req.user.role)`; if false it yields `ForbiddenError('Insufficient permissions')` -> 403 `FORBIDDEN` (lines 55-58). Otherwise `next()`.

Roles are a closed union `'ADMIN' | 'OPERATOR'` (line 9), matching the Zod enum in registration (`auth.schemas.ts:7`). Roles are flat with no hierarchy: `requireRole('OPERATOR')` would not admit ADMIN. Calling `requireRole()` with no arguments denies everyone (empty list).

The only current usage is `GET /api/v1/users/:id` with `requireRole('ADMIN')`, ordered `authenticate -> requireRole -> validate -> controller` (user.routes.ts:12-18), so authorization precedes input validation (invalid id format by non-admin yields 403, not 400). Other modules apply no role restriction: any authenticated OPERATOR or ADMIN can create, update and delete customers, products and orders. Whether this is intended is not documented in code (see risks).

**Rule workflow**:
```
no req.user -> 401 "Unauthorized"
role not in allowed -> 403 "Insufficient permissions"
else next()
```

---

### Business Rule: Route protection policy

**Overview**:
Which endpoints demand authentication/authorization.

**Detailed description**:
Public endpoints: `POST /auth/register`, `POST /auth/login` (auth.routes.ts:10-11); health or other routes defined in `src/app.ts` and `src/routes/index.ts` were outside this component's scope. Authenticated: `GET /auth/me` (per route, auth.routes.ts:12); all of `/customers`, `/products`, `/orders` via `router.use(authenticate)` at router level (customer.routes.ts:14, product.routes.ts:14, order.routes.ts:14). ADMIN-only: `GET /users/:id`.

Because `router.use(authenticate)` is mounted at router level inside each module router, any route later added to those routers is protected by default. The users router, in contrast, applies middleware per route, so new routes there would be unprotected unless added explicitly.

**Rule workflow**:
```
/auth/register, /auth/login -> public
/auth/me, /customers/*, /products/*, /orders/* -> authenticate
/users/:id -> authenticate + requireRole('ADMIN')
```

---

### Business Rule: Uniform error contract

**Overview**:
Auth failures use the shared error classes and are formatted centrally.

**Detailed description**:
`UnauthorizedError` (401, code `UNAUTHORIZED`, default message "Unauthorized") and `ForbiddenError` (403, `FORBIDDEN`, default "Forbidden") extend `AppError` (src/shared/errors/http-errors.ts:15-25). The middleware never writes responses itself; it calls `next(err)`, and `errorMiddleware` serializes `{ error: { code, message } }` (error.middleware.ts:15-24). Details are omitted for auth errors.

This keeps the response format consistent with other domain errors and avoids exposing token internals. Auth errors are not logged (only unhandled 500s are logged at error.middleware.ts:57-60).

**Rule workflow**:
```
middleware -> next(AppError subclass) -> errorMiddleware -> res.status(statusCode).json({error:{code,message}})
```

---

## 4. Component Structure

```
src/
├── middlewares/
│   ├── auth.middleware.ts          # authenticate, requireRole, AuthUser type, Request augmentation (COMPONENT)
│   ├── error.middleware.ts         # Renders AppError (401/403) responses
│   ├── request-logger.middleware.ts# Sets req.id (augmented type shared)
│   └── validate.middleware.ts      # Runs after auth in user routes
├── config/env.ts                   # JWT_SECRET, JWT_EXPIRES_IN
├── shared/errors/                  # UnauthorizedError, ForbiddenError, AppError
└── modules/
    ├── auth/ (routes, controller, service)  # token issuance, /auth/me consumer
    ├── users/user.routes.ts                 # only requireRole consumer
    └── customers|products|orders/*.routes.ts # authenticate consumers
tests/
├── auth.test.ts                    # indirect coverage
├── orders.test.ts                  # indirect coverage (Bearer usage)
├── helpers/factories.ts            # createTestUser, loginAndGetToken, bootstrapAuthenticatedUser
└── setup.ts                        # DB cleanup
```

Single-file component with module-level pure functions; no state.

## 5. Dependency Analysis

```
Internal Dependencies:
auth.middleware.ts → config/env.ts (JWT_SECRET)
auth.middleware.ts → shared/errors/index.ts (UnauthorizedError, ForbiddenError)

Consumers (reverse):
auth.routes.ts, customer.routes.ts, product.routes.ts, order.routes.ts, user.routes.ts → auth.middleware.ts
(type consumers via Request.user: auth.controller.ts, order.controller.ts)

Runtime coupling (not imports):
AuthService.signToken must produce {sub, email, role} claims with the same JWT_SECRET
errorMiddleware must be registered to render AppError

External Dependencies:
- express 4.21.1 (types only: RequestHandler; module augmentation of express-serve-static-core)
- jsonwebtoken 9.0.2 (jwt.verify), @types/jsonwebtoken 9.0.7
```

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| authenticate | 5 route modules (auth, customers, products, orders, users) | 3 (env, UnauthorizedError, jsonwebtoken) | High |
| requireRole | 1 (user.routes.ts) | 1 (ForbiddenError) plus UnauthorizedError | Medium |
| AuthUser (type) | 0 explicit imports found; implicit via Request.user in 2 controllers | 0 | Medium |
| Request augmentation (user, id) | Controllers (auth, orders) | 0 | Medium |

Counts derived from import grep over `src/`; afferent figures are for modules, not call sites.

## 7. Endpoints

The component exposes no endpoints itself. It protects the following (per route files):

| Endpoint | Method | Protection |
|----------|--------|------------|
| /api/v1/auth/register | POST | Public |
| /api/v1/auth/login | POST | Public |
| /api/v1/auth/me | GET | authenticate |
| /api/v1/users/:id | GET | authenticate + requireRole('ADMIN') |
| /api/v1/customers (/ , /:id, POST, PATCH, DELETE) | all | authenticate |
| /api/v1/products (/ , /:id, POST, PATCH, DELETE) | all | authenticate |
| /api/v1/orders (/ , /:id, POST, PATCH, DELETE) | all | authenticate |

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| jsonwebtoken | Library | Token verification | In-process | JWT (HS*) | try/catch -> 401 generic |
| env config (JWT_SECRET) | Internal config | Verification secret | Process env | String (min 16) | Process exit on invalid config |
| AuthService (token issuer) | Internal module | Produces tokens consumed here | Shared claim contract | JWT claims sub/email/role | None; contract implicit |
| errorMiddleware | Internal middleware | Renders 401/403 | Express error chain | JSON | Centralized |
| Database | None | The component does not query the DB | - | - | - |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Chain of Responsibility / Middleware | authenticate, requireRole | src/middlewares/auth.middleware.ts | Request pipeline gating |
| Higher-order function (factory) | requireRole(...roles) | auth.middleware.ts:49 | Parameterized RBAC |
| Stateless token authentication | JWT bearer | auth.middleware.ts:41 | No server session |
| Declaration merging | Request.user augmentation | auth.middleware.ts:12-17 | Typed identity propagation |
| Centralized error handling | next(AppError) | auth.middleware.ts + error.middleware.ts | Uniform error contract |
| Fail-closed guard | requireRole checks req.user | auth.middleware.ts:51 | Safe misconfiguration behavior |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Registration (consumer of roles) | RegisterSchema accepts `role` from client (auth.schemas.ts:7); unless overridden in UserService (not verified), anyone can register as ADMIN | Privilege escalation bypassing requireRole |
| Medium | authenticate | No DB check on each request; role and existence trusted from token for up to 8h | Stale privileges after demotion or deletion; no revocation |
| Medium | Route policy | requireRole used on one route only; all authenticated users may mutate customers/products/orders and delete | Coarse authorization; intent undocumented |
| Medium | authenticate | `jwt.verify` without `algorithms` option; no issuer/audience checks | Relies on library defaults |
| Medium | authenticate | Payload cast with `as JwtPayload`, no runtime validation of claims (role could be arbitrary if secret signs it) | Unexpected role values deny at requireRole but pass authenticate |
| Low | authenticate | Bare catch discards error cause; auth failures not logged | Limited observability, no brute-force/audit trail |
| Low | authenticate | `next()` is invoked inside try block | Possible mislabeling of downstream sync errors (medium confidence) |
| Low | users router | Per-route middleware rather than router-level | New routes could be unprotected by omission |
| Low | Request augmentation | `id?` declared in auth middleware but unrelated to auth | Cohesion smell |
| Low | Rate limiting | No rate limiting or lockout visible for auth | Not part of component; noted as absence |

## 11. Test Coverage Analysis

Test files: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts` (Vitest 2.1.4, supertest 7.0.0, integration style against real DB; no dedicated `auth.middleware` test file). No coverage tooling or thresholds configured; coverage percentages are not measurable without execution (not run).

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| authenticate (happy path) | 0 | 2+ (auth.test.ts:65-77 GET /auth/me; all orders.test.ts calls with Bearer token) | Not measured; happy path covered | Good realistic flow |
| authenticate (missing header) | 0 | 1 (auth.test.ts:79-83, status only) | Partial | Does not assert code/message |
| authenticate (non-Bearer scheme, empty token) | 0 | 0 | None found | Gap |
| authenticate (invalid signature, malformed, expired token) | 0 | 0 | None found | Gap |
| orders without token (401) | 0 | 0 evident in grep | None found | Gap |
| requireRole (401 no user, 403 wrong role, 200 ADMIN) | 0 | 0 (no test references /users/:id) | None found | High-risk gap |

Test helpers: `createTestUser` (default role OPERATOR, factories.ts:27), `loginAndGetToken`, `bootstrapAuthenticatedUser` used to obtain real tokens through the login flow. Tests do not craft custom JWTs.
