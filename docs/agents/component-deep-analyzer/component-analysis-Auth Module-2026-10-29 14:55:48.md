# Component Deep Analysis Report: Auth Module

Scope: `src/modules/auth` (AuthController, AuthService, routes, schemas) plus the directly coupled `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, and the user collaborators (`UserService`, `UserRepository`) that the module delegates to. Ignored folders: none. Analysis only; no files modified, no code or tests executed.

## 1. Executive Summary

The Auth Module provides user registration, credential login (bcrypt + JWT HS256 access token) and a "current user" endpoint for the Order Management System (Express 4-style routers, Prisma/MySQL, Zod, Vitest). It is stateless: the only persistence is the `users` table, accessed through `UserRepository`. Token verification and role authorization are implemented in a shared middleware (`src/middlewares/auth.middleware.ts`) that every other module uses (customers, products, orders, users).

Key findings:
- Three endpoints under `/api/v1/auth`: `POST /register`, `POST /login`, `GET /me`.
- `register` is public and accepts a client-supplied `role` (`ADMIN` or `OPERATOR`, default `OPERATOR`) (auth.schemas.ts:7), so any anonymous caller can create an ADMIN account. This is a privilege-escalation risk.
- No refresh tokens, revocation, logout, rate limiting, or lockout. Access token lifetime defaults to 8h (env.ts:9).
- Login does not run bcrypt for unknown emails, so timing differs between "unknown email" and "wrong password" (user enumeration via timing); messages are identical ("Invalid credentials").
- Test coverage covers 6 happy/negative paths via HTTP; no tests for expired/malformed tokens, requireRole, or token claims.

## 2. Data Flow Analysis

Register (`POST /api/v1/auth/register`):
1. Request enters Express (`express.json` 1mb limit, requestLogger) in src/app.ts:59-60, routed by `buildApiRouter` (src/routes/index.ts:24) to `buildAuthRouter`.
2. `validate({ body: registerSchema })` parses and replaces `req.body` (validate.middleware.ts:14-16); ZodError becomes `ValidationError` (400, VALIDATION_ERROR, details array).
3. `AuthController.register` (auth.controller.ts:12) calls `AuthService.register`.
4. `AuthService.register` delegates to `UserService.createUser` (auth.service.ts:27-29).
5. `UserService.createUser` checks `UserRepository.findByEmail`; conflict raises `ConflictError` EMAIL_ALREADY_USED (user.service.ts:22-25).
6. Password hashed with bcrypt, 10 rounds (user.service.ts:16,26).
7. `UserRepository.create` inserts via Prisma (user.repository.ts:14-16).
8. Result mapped by `UserService.toPublic` (no hash) and returned with 201.
9. Any error flows to `errorMiddleware` (error.middleware.ts:14), which maps AppError to `{error:{code,message,details?}}`; Prisma P2002 maps to 409 CONFLICT.

Login (`POST /api/v1/auth/login`):
1. `validate({ body: loginSchema })`.
2. `AuthController.login` -> `AuthService.login` (auth.service.ts:31).
3. `UserRepository.findByEmail`; null -> `UnauthorizedError('Invalid credentials')`.
4. `bcrypt.compare(password, passwordHash)`; false -> same error.
5. `signToken` creates JWT with claims `sub, email, role`, secret `JWT_SECRET`, `expiresIn = JWT_EXPIRES_IN` (auth.service.ts:47-50).
6. Response 200: `{ user: PublicUser, tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }`.

Me (`GET /api/v1/auth/me`):
1. `authenticate` middleware reads Authorization header, verifies JWT, sets `req.user` (auth.middleware.ts:27-47).
2. `AuthController.me` checks `req.user`, calls `UserService.getById` (a DB lookup, not token claims) and returns PublicUser; missing user -> NotFoundError 404.

Downstream use: other routers call `authenticate` (customer.routes.ts:14, product.routes.ts:14, order.routes.ts:14) and `GET /users/:id` additionally uses `requireRole('ADMIN')` (user.routes.ts:12-18).

## 3. Business Rules & Logic

## Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation | Email must be valid format, max 255 chars | src/modules/auth/auth.schemas.ts:4,11 |
| Validation | Register password 8 to 72 chars | src/modules/auth/auth.schemas.ts:5 |
| Validation | Login password 1 to 72 chars | src/modules/auth/auth.schemas.ts:12 |
| Validation | Name 2 to 150 chars | src/modules/auth/auth.schemas.ts:6 |
| Business Logic | Role is ADMIN or OPERATOR, default OPERATOR (client-selectable) | src/modules/auth/auth.schemas.ts:7 |
| Business Logic | Email must be unique (409 EMAIL_ALREADY_USED) | src/modules/users/user.service.ts:22-25; prisma/schema.prisma:27 |
| Security | Passwords stored as bcrypt hash, 10 rounds | src/modules/users/user.service.ts:16,26 |
| Security | Never return passwordHash (public projection) | src/modules/users/user.service.ts:42-51 |
| Business Logic | Login failure responses are generic "Invalid credentials" (401) | src/modules/auth/auth.service.ts:33-39 |
| Business Logic | JWT payload sub/email/role, lifetime from env (default 8h) | src/modules/auth/auth.service.ts:47-50; src/config/env.ts:9 |
| Configuration | JWT_SECRET required, min 16 chars, fail-fast on boot | src/config/env.ts:8,14-25 |
| Security | Bearer header required and token verified for protected routes | src/middlewares/auth.middleware.ts:27-47 |
| Authorization | Role-based access via requireRole | src/middlewares/auth.middleware.ts:49-61 |
| Business Logic | /me returns fresh DB user; 404 if user deleted | src/modules/auth/auth.controller.ts:30-38 |

## Detailed breakdown of the business rules

---

### Business Rule: Registration input validation

**Overview**:
Registration payloads are validated by Zod before reaching any service. Email must be a valid address up to 255 characters, password 8 to 72 characters, name 2 to 150 characters, role one of the two enum values.

**Detailed description**:
The schema (auth.schemas.ts:3-8) is applied by the `validate` middleware, which replaces `req.body` with the parsed result. This means defaults are applied (role becomes OPERATOR when omitted) and unknown keys are stripped (Zod object default behavior, so extra fields never reach the service). Failures are collected as an array of `{path, message}` and returned as HTTP 400 with code VALIDATION_ERROR, as verified by tests/auth.test.ts:30-39.

The upper bound of 72 characters on passwords aligns with bcrypt's 72-byte input limit; bytes beyond 72 are ignored by bcrypt, and note the check is on characters, not bytes, so multi-byte characters could exceed 72 bytes (implicit behavior, medium confidence on intent). The 255 and 150 limits match the column definitions in prisma/schema.prisma (`VarChar(255)` email, `VarChar(150)` name).

There is no password complexity rule (only length), no email normalization (no lowercasing or trimming), so emails differing by case may be treated as distinct depending on MySQL collation (the unique index relies on database collation; not verified at runtime).

**Rule workflow**:
Body -> registerSchema.parse -> success: req.body replaced, continue; ZodError: ValidationError 400 with details.

---

### Business Rule: Login input validation

**Overview**:
Login requires a valid-format email (max 255) and a non-empty password (max 72).

**Detailed description**:
loginSchema (auth.schemas.ts:10-13) deliberately uses `min(1)` for the password, so users whose passwords predate stricter policies can still attempt login, and avoids leaking the password policy. The max of 72 rejects oversize inputs early, limiting bcrypt cost abuse from huge payloads (the JSON body is also capped at 1mb, app.ts:59).

An invalid email format yields 400 VALIDATION_ERROR rather than 401, so the response differs for malformed versus unknown emails, which is a minor information difference but not an account-existence leak.

**Rule workflow**:
Body -> loginSchema.parse -> success: AuthService.login; failure: 400.

---

### Business Rule: Unique email on registration

**Overview**:
Two users cannot share an email; duplicates return 409 with code EMAIL_ALREADY_USED.

**Detailed description**:
`UserService.createUser` performs a read-then-write: `findByEmail` and, if found, throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (user.service.ts:22-25). The database also enforces uniqueness (`email @unique`, schema.prisma:27). Under concurrent registrations of the same email the check can race; the loser would hit a Prisma P2002 error, which `errorMiddleware` maps to 409 with generic code CONFLICT (error.middleware.ts:37-47) rather than EMAIL_ALREADY_USED. So the response code is inconsistent between the two paths.

The explicit conflict response confirms account existence to anonymous callers (user enumeration through registration). Verified by tests/auth.test.ts:19-28.

**Rule workflow**:
findByEmail -> exists? 409 EMAIL_ALREADY_USED : continue to hash and insert; DB unique violation (race) -> 409 CONFLICT.

---

### Business Rule: Role assignment at registration

**Overview**:
The registering client chooses `ADMIN` or `OPERATOR`; default is OPERATOR.

**Detailed description**:
`role: z.enum(['ADMIN','OPERATOR']).default('OPERATOR')` (auth.schemas.ts:7) is passed unchanged through `AuthService.register` and `UserService.createUser` into the user row. The `/register` route has no `authenticate` or `requireRole` guard (auth.routes.ts:10). Consequently any anonymous caller may self-register as ADMIN, gaining access to `GET /users/:id` (ADMIN only) and any other ADMIN-gated behavior. It is not documented in the code whether this is intentional (e.g. bootstrap or development convenience); the intent is ambiguous. The test at tests/auth.test.ts:11 sends `role: 'OPERATOR'` explicitly; no test covers ADMIN self-registration.

Roles are also baked into the JWT at login (auth.service.ts:49), so role changes would not take effect until token expiry (no user-update endpoint exists in the module, so this is currently latent).

**Rule workflow**:
Body.role (default OPERATOR) -> persisted in users.role (enum UserRole, default OPERATOR) -> embedded in JWT at login -> read by requireRole.

---

### Business Rule: Password hashing and non-disclosure

**Overview**:
Passwords are stored only as bcrypt hashes (10 salt rounds) and never returned in responses.

**Detailed description**:
`BCRYPT_ROUNDS = 10` is a constant in user.service.ts:16 (not configurable by env). Hash is stored in `passwordHash VarChar(255)` (schema.prisma:28). All responses pass through `UserService.toPublic` (user.service.ts:42-51), which whitelists id, email, name, role, createdAt, updatedAt. The test at tests/auth.test.ts:15-16 asserts the absence of `passwordHash` and `password` in the register response.

Login uses `bcrypt.compare` (auth.service.ts:36), which is constant-time with respect to the hash comparison, but is skipped when the email is unknown, creating an observable timing difference.

**Rule workflow**:
plain password -> bcrypt.hash(10) -> persisted; login: bcrypt.compare(plain, hash).

---

### Business Rule: Credential verification and generic failure message

**Overview**:
Login fails with 401 UNAUTHORIZED and the message "Invalid credentials" for both unknown email and wrong password.

**Detailed description**:
`AuthService.login` looks up the user by email and throws `UnauthorizedError('Invalid credentials')` if absent (auth.service.ts:33-35); if present, it compares the password and throws the same error on mismatch (auth.service.ts:37-39). Using identical status, code, and message prevents explicit enumeration of accounts through login responses. Test coverage exists only for the wrong-password branch (tests/auth.test.ts:54-63); the unknown-email branch is untested.

There is no failed-attempt counter, delay, lockout, or rate limiter anywhere in the module or in app.ts, so brute force is not mitigated at the application level.

**Rule workflow**:
findByEmail -> null: 401; compare -> false: 401; true: issue token.

---

### Business Rule: Access token issuance

**Overview**:
On successful login a JWT access token is issued containing `sub` (user id), `email`, `role`, signed with `JWT_SECRET`, expiring per `JWT_EXPIRES_IN` (default `8h`).

**Detailed description**:
`signToken` (auth.service.ts:47-50) calls `jwt.sign({sub,email,role}, env.JWT_SECRET, {expiresIn})`. The algorithm is not specified, so jsonwebtoken 9.0.2 defaults to HS256. The response includes `expiresIn` as the configured string (for example "8h") and `tokenType: 'Bearer'` (auth.service.ts:43), so clients receive a duration string rather than seconds or an absolute timestamp. The cast `as SignOptions['expiresIn']` bypasses type checks for the string; an invalid env value (for example "abc") is not validated by `envSchema` (env.ts:9 accepts any string) and would fail at sign time during login with a 500.

Only an access token is issued: no refresh token, no `jti`, no issuer/audience claims, hence no revocation mechanism. Tests verify only that accessToken is a string and tokenType is Bearer (tests/auth.test.ts:48-51).

**Rule workflow**:
Verified credentials -> jwt.sign(payload, secret, expiresIn) -> `{user, tokens}` 200.

---

### Business Rule: JWT secret configuration

**Overview**:
`JWT_SECRET` must be present and at least 16 characters, or the process exits at startup.

**Detailed description**:
`envSchema` (env.ts:8) enforces `min(16)`. `loadEnv` (env.ts:14-25) logs the issues and calls `process.exit(1)` on failure, so the application never runs with a missing or short secret. `.env.example:8` ships a placeholder value ("change-me-in-production-please-use-a-long-random-secret") that satisfies the length rule, so an unchanged example would pass validation; no rule rejects the placeholder. `JWT_EXPIRES_IN` defaults to `8h`.

**Rule workflow**:
process.env -> envSchema.safeParse -> failure: console.error + exit(1); success: exported `env`.

---

### Business Rule: Bearer token authentication (protected routes)

**Overview**:
Protected routes require `Authorization: Bearer <jwt>`; invalid, missing, or expired tokens yield 401 UNAUTHORIZED.

**Detailed description**:
`authenticate` (auth.middleware.ts:27-47) checks the header exists and starts with `bearer ` (case-insensitive), then slices the token at index 7 and trims it; empty token gives "Missing bearer token". `jwt.verify(token, env.JWT_SECRET)` validates signature and expiry (algorithms are not restricted explicitly; with a string secret, jsonwebtoken 9 allows HMAC algorithms only). Any verification error collapses to "Invalid or expired token". On success, `req.user` is populated from claims `{id: sub, email, role}` without any database check; therefore a deleted user's token remains valid for protected routes until expiry (except `/auth/me`, which does query the DB and returns 404).

The payload is cast to `JwtPayload` without runtime shape validation; a validly signed token lacking the expected claims would yield `req.user` with undefined fields.

The middleware is applied globally with `router.use(authenticate)` in customers, products and orders routers and per-route in users and auth. Tests exercise only the missing-token branch (tests/auth.test.ts:79-83) and the valid-token path indirectly.

**Rule workflow**:
Header present and bearer? -> token non-empty? -> jwt.verify -> set req.user -> next; otherwise next(UnauthorizedError).

---

### Business Rule: Role-based authorization

**Overview**:
`requireRole(...roles)` allows access only if `req.user.role` is among the allowed roles, otherwise 403 FORBIDDEN "Insufficient permissions"; 401 if no user.

**Detailed description**:
`requireRole` (auth.middleware.ts:49-61) is a factory returning a handler. It must follow `authenticate`. The only current use is `GET /users/:id` with `ADMIN` (user.routes.ts:15). Other modules authenticate but do not restrict by role, so ADMIN and OPERATOR have equal access there. Combined with open role selection at registration, the ADMIN restriction is effectively not a security boundary. No tests cover `requireRole` (a grep of tests shows no 403 assertion).

**Rule workflow**:
req.user missing -> 401; role not in list -> 403; else next.

---

### Business Rule: Current user retrieval (/auth/me)

**Overview**:
Returns the public profile of the authenticated user by reading the database.

**Detailed description**:
`AuthController.me` (auth.controller.ts:30-38) rethrows 401 if `req.user` is absent (defensive, since `authenticate` precedes it) and then calls `userService.getById(req.user.id)`, producing a fresh profile (rather than echoing token claims). A user deleted after token issuance yields 404 NOT_FOUND ('User not found'). Verified by tests/auth.test.ts:65-77.

**Rule workflow**:
authenticate -> getById -> toPublic -> 200; not found -> 404.

---

## 4. Component Structure

```
src/modules/auth/
├── auth.controller.ts   # HTTP handlers: register, login, me (arrow-function RequestHandlers)
├── auth.routes.ts       # Router factory buildAuthRouter(controller); wires validate/authenticate
├── auth.schemas.ts      # Zod schemas registerSchema, loginSchema and inferred input types
└── auth.service.ts      # AuthService: register (delegating), login, private signToken

Closely related (outside module folder):
src/middlewares/auth.middleware.ts      # authenticate, requireRole, AuthUser type, Express Request augmentation
src/middlewares/validate.middleware.ts  # Zod validation middleware
src/modules/users/user.service.ts       # createUser, getById, toPublic (PublicUser)
src/modules/users/user.repository.ts    # findById, findByEmail, create
src/config/env.ts                       # JWT_SECRET, JWT_EXPIRES_IN
src/shared/errors/                      # UnauthorizedError, ForbiddenError, ConflictError, etc.
src/app.ts                              # Composition root (manual DI): lines 26-32
src/routes/index.ts                     # Mounts /auth
tests/auth.test.ts                      # HTTP integration tests
tests/helpers/factories.ts, tests/setup.ts
```

## 5. Dependency Analysis

Internal Dependencies:
```
auth.routes -> validate.middleware, auth.middleware, auth.schemas, AuthController(type)
AuthController -> AuthService (type), UserService (type), UnauthorizedError
AuthService -> UserRepository (type), UserService (value, for static toPublic), env, UnauthorizedError, auth.schemas (types)
AuthService.register -> UserService.createUser -> UserRepository -> PrismaClient
auth.middleware -> env, UnauthorizedError, ForbiddenError
Composition: src/app.ts:31-32 builds AuthService(userRepository, userService), AuthController(authService, userService)
Consumers of auth.middleware: customers, products, orders, users, auth routers
```

Note: `AuthService.register` takes `RegisterInput` and passes it to `createUser(CreateUserInput)`; the structural compatibility depends on user.schemas.ts (not part of this module).

External Dependencies (package.json):
- bcrypt 5.1.1 - password hashing (auth.service.ts:1, user.service.ts:1)
- jsonwebtoken 9.0.2 - sign/verify JWT (auth.service.ts:2, auth.middleware.ts:2)
- zod - schema validation and env parsing
- express - routing (Router, RequestHandler)
- @prisma/client / MySQL - persistence of users (via UserRepository)
- supertest, vitest - tests

## 6. Afferent and Efferent Coupling

Counts are based on source references found in `src` (type and value imports; test files excluded).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| AuthService | 2 (AuthController, app.ts) | 7 (bcrypt, jsonwebtoken, env, UnauthorizedError, UserRepository, UserService, auth.schemas) | Medium |
| AuthController | 2 (auth.routes, app.ts/routes index) | 3 (AuthService, UserService, UnauthorizedError) | Low |
| buildAuthRouter | 1 (routes/index.ts) | 5 (validate, authenticate, 2 schemas, AuthController) | Low |
| registerSchema / loginSchema | 1 (auth.routes) plus type use in AuthService | 1 (zod) | Low |
| authenticate (middleware) | 5 route modules (auth, users, customers, products, orders) | 3 (jsonwebtoken, env, errors) | High |
| requireRole (middleware) | 1 (users routes) | 2 (errors) | Medium |

## 7. Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/auth/register | POST | Create a user (public, no auth); body: email, password, name, role?; 201 PublicUser; 400, 409 |
| /api/v1/auth/login | POST | Authenticate; body: email, password; 200 `{user, tokens{accessToken, expiresIn, tokenType}}`; 400, 401 |
| /api/v1/auth/me | GET | Current user profile; requires Bearer token; 200 PublicUser; 401, 404 |

Error envelope: `{ "error": { "code", "message", "details"? } }` (error.middleware.ts:16-22).

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL (users table) via Prisma | Database | Persist/lookup users | Prisma client / SQL | Prisma models | P2002 -> 409, P2025 -> 404, others -> 500 with logging |
| UserService / UserRepository | Internal module | User creation and retrieval | In-process calls | TypeScript objects | AppError subclasses |
| Other routers (customers, products, orders, users) | Internal consumers | Request authentication and authorization | Express middleware | `req.user` | UnauthorizedError / ForbiddenError |
| Environment (`JWT_SECRET`, `JWT_EXPIRES_IN`) | Configuration | Token signing | process.env | strings | Process exit on invalid |

No external third-party identity providers or network services.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered (controller / service / repository) | AuthController -> AuthService -> UserRepository | src/modules/auth, src/modules/users | Separation of concerns |
| Constructor Dependency Injection (manual) | Composition root `buildControllers` | src/app.ts:26-53 | Wiring and testability |
| Router factory | `buildAuthRouter(controller)` | auth.routes.ts:7 | Injectable routing |
| Middleware / Chain of Responsibility | validate -> authenticate -> handler | auth.routes.ts:10-12 | Cross-cutting concerns |
| Higher-order function | `requireRole(...roles)` | auth.middleware.ts:49 | Parameterized authorization |
| Facade/Delegation | `AuthService.register` delegates to `UserService.createUser` | auth.service.ts:27-29 | Reuse of user creation |
| Data Transfer / Projection | `PublicUser`, `UserService.toPublic` | user.service.ts:7-51 | Hide sensitive fields |
| Centralized error handling | AppError hierarchy + errorMiddleware | src/shared/errors, error.middleware.ts | Uniform error responses |
| Fail-fast configuration | Zod env schema + exit | src/config/env.ts | Safe startup |
| Declaration merging | Express `Request.user` augmentation | auth.middleware.ts:12-17 | Typed request context |

Architecture note: stateless token-based authentication (JWT), authorization data embedded in token.

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Registration (auth.schemas.ts:7, auth.routes.ts:10) | Public endpoint accepts `role: ADMIN` | Anonymous privilege escalation to ADMIN |
| High | Login (auth.routes.ts:11, auth.service.ts:31) | No rate limiting, lockout, or throttling | Brute-force and credential-stuffing exposure |
| Medium | authenticate (auth.middleware.ts:41-42) | Claims trusted without DB check; no revocation, no logout, 8h lifetime | Deleted/demoted users retain access until expiry |
| Medium | Login timing (auth.service.ts:33-39) | bcrypt skipped for unknown emails | Timing-based account enumeration |
| Medium | Registration (user.service.ts:22-25) | Email enumeration via 409; check-then-insert race yields different code (CONFLICT vs EMAIL_ALREADY_USED) | Information disclosure; inconsistent API contract |
| Medium | JWT config (auth.service.ts:48, env.ts:9) | `JWT_EXPIRES_IN` unvalidated string; algorithm not pinned in sign/verify; no iss/aud/jti claims | Runtime 500 on bad config; weaker token hardening |
| Medium | Secret management (.env.example:8, env.ts:8) | Placeholder secret passes the 16-char check; HS256 shared secret | Weak secret could be deployed unnoticed |
| Medium | Role authorization (src/modules) | `requireRole` used only once; other modules grant equal access to OPERATOR and ADMIN | Limited authorization granularity (intent unclear) |
| Low | Email handling | No normalization (case/whitespace) | Possible duplicate or mismatched accounts depending on collation |
| Low | Password policy | Length only; 72 char limit counted in characters not bytes | Weak passwords accepted; bcrypt truncation for multibyte |
| Low | Hash cost | Hardcoded `BCRYPT_ROUNDS = 10` | Not tunable per environment |
| Low | Cohesion | `AuthController` depends on `UserService` directly; `AuthService.register` is a pass-through; `AuthService` imports `UserService` as value for static method | Blurred boundary between auth and user modules |
| Low | JwtPayload typing (auth.middleware.ts:41) | Unchecked cast of verified payload | Malformed-but-signed token yields undefined user fields |
| Low | Auth middleware location | Lives in `src/middlewares`, outside the module folder | Module boundary spans two locations |
| Low | Audit logging | No logging of login success/failure events in the module | Limited security observability |

## 11. Test Coverage Analysis

Test files located: `tests/auth.test.ts` (Auth flow, 6 tests), supporting `tests/helpers/factories.ts` (createTestUser, loginAndGetToken, bootstrapAuthenticatedUser), `tests/setup.ts` (DB cleanup per test), `vitest.config.ts` (single fork, sequential, 30s timeouts). `tests/orders.test.ts` uses authentication helpers indirectly (not analyzed for auth rules). No unit tests exist for AuthService, middlewares, or schemas. No coverage tooling or thresholds are configured (no coverage block in vitest.config.ts), so numeric coverage is not available.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| AuthController.register / AuthService.register | 0 | 3 (tests/auth.test.ts:6, 19, 30) | Not measured; success, duplicate, invalid payload covered | Asserts status, code, no hash leak; missing ADMIN role, defaults, boundary lengths |
| AuthService.login | 0 | 2 (tests/auth.test.ts:41, 54) | Not measured; success and wrong password covered | Unknown email, token claims/expiry, 400 on invalid body not tested |
| AuthController.me | 0 | 2 (tests/auth.test.ts:65, 79) | Not measured | No deleted-user 404 case |
| authenticate middleware | 0 | Indirect (missing token only) | Partial | Malformed header, bad signature, expired token untested |
| requireRole | 0 | 0 | None | No 403 or 401 test; high risk given sole ADMIN guard |
| Zod schemas (auth.schemas.ts) | 0 | 1 indirect (invalid payload) | Partial | Boundary values untested |
| env JWT config | 0 | 0 | None | Startup validation untested |

Tests hit a real database through Prisma (not mocked), which makes them true integration tests but dependent on a running MySQL instance.
