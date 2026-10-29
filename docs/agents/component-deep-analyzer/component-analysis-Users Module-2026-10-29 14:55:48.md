# Component Deep Analysis Report: Users Module

Analysis date: 2026-10-29 14:55:48
Scope: `src/modules/users` (UserController, UserService, UserRepository, user.routes.ts, user.schemas.ts) plus its consumers and tests across the project. No folders ignored.

## 1. Executive Summary

The Users Module manages system operators (roles `ADMIN` and `OPERATOR`) of the Order Management API (Express 4, Prisma 5.22 / MySQL-style schema, Zod, bcrypt, JWT). It follows the project layered layout (routes, controller, service, repository, schemas).

Role in the system:
- Exposes one HTTP endpoint, `GET /api/v1/users/:id`, restricted to `ADMIN`.
- Provides the user creation logic (`UserService.createUser`) that is reused by the Auth module's `POST /auth/register`. The module has no create endpoint of its own.
- Provides `UserService.getById` (used by `GET /auth/me`), `UserService.toPublic` (used by login), and `UserRepository.findByEmail` (used by login).

Key findings:
- Credentials never leave the module: `toPublic` is an explicit allow-list that excludes `passwordHash`.
- `createUserSchema` and `userIdParamSchema` in `user.schemas.ts` are defined but unused by the module's own routes. `user.routes.ts` declares its own duplicate `idParamSchema` (user.routes.ts:7). `createUserSchema` is a duplicate of `registerSchema` in the Auth module (auth.schemas.ts:3-8).
- Email uniqueness is checked with a read-then-write sequence (user.service.ts:22-32) with no explicit handling of the database unique-constraint violation (race window).
- Public self-registration with a client-supplied `role` (auth.schemas.ts:7) allows any caller to create an `ADMIN`. This is an Auth-module exposure of the Users Module's `createUser`, which accepts the role without authorization checks.
- There are no dedicated tests for the Users Module. It is exercised indirectly through `tests/auth.test.ts`. No test covers `GET /users/:id`.

## 2. Data Flow Analysis

Flow A: `GET /api/v1/users/:id`
```
1. Request enters router mounted at /users (src/routes/index.ts:25)
2. authenticate middleware validates Bearer JWT, sets req.user (auth.middleware.ts:27-47)
3. requireRole('ADMIN') checks req.user.role (auth.middleware.ts:49-61)
4. validate({ params }) parses :id as UUID with Zod (user.routes.ts:16)
5. UserController.getById calls UserService.getById(req.params.id) (user.controller.ts:9)
6. UserService.getById calls UserRepository.findById (user.service.ts:37)
7. UserRepository runs prisma.user.findUnique({ where: { id } }) (user.repository.ts:7)
8. If null, NotFoundError('User') -> 404 NOT_FOUND (user.service.ts:38)
9. UserService.toPublic strips passwordHash (user.service.ts:42-51)
10. Controller responds 200 with JSON (user.controller.ts:10); errors go to next(err) and the global error middleware
```

Flow B: user creation (invoked via Auth module `POST /auth/register`)
```
1. AuthController.register receives validated body (registerSchema, auth.routes.ts:10)
2. AuthService.register delegates to UserService.createUser (auth.service.ts:27-29)
3. UserRepository.findByEmail checks for existing email (user.service.ts:22)
4. If present, ConflictError 409 EMAIL_ALREADY_USED
5. bcrypt.hash(password, 10) (user.service.ts:26)
6. UserRepository.create persists email, passwordHash, name, role (user.service.ts:27-32)
7. toPublic result returned; AuthController responds 201
```

Flow C: reuse by Auth
- `GET /auth/me`: AuthController calls `userService.getById(req.user.id)` (auth.controller.ts:33).
- `POST /auth/login`: AuthService uses `UserRepository.findByEmail` and `UserService.toPublic` (auth.service.ts:32, 42).

## 3. Business Rules & Logic

## Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authorization | Only ADMIN can read a user by id | src/modules/users/user.routes.ts:14-15 |
| Authentication | Valid Bearer JWT required for user lookup | src/modules/users/user.routes.ts:14, src/middlewares/auth.middleware.ts:27 |
| Validation | `:id` must be a UUID | src/modules/users/user.routes.ts:7,16 |
| Business Logic | Email must be unique at creation (409 EMAIL_ALREADY_USED) | src/modules/users/user.service.ts:22-25 |
| Security | Passwords hashed with bcrypt, 10 rounds | src/modules/users/user.service.ts:16,26 |
| Data Exposure | Responses expose only the public user fields | src/modules/users/user.service.ts:42-51 |
| Business Logic | Missing user yields 404 | src/modules/users/user.service.ts:38 |
| Validation | Creation input constraints (email, password 8-72, name 2-150, role enum) | src/modules/users/user.schemas.ts:7-12 (mirrored in src/modules/auth/auth.schemas.ts:3-8, which is the one enforced) |
| Default | Role defaults to OPERATOR | user.schemas.ts:11, prisma/schema.prisma:30 |
| Data Constraint | DB-level email uniqueness and column lengths | prisma/schema.prisma:25-38 |

## Detailed breakdown of the business rules:

---

### Business Rule: Admin-only user lookup

**Overview**:
Reading a user by id requires a valid token and the `ADMIN` role.

**Detailed description**:
The route `GET /:id` chains `authenticate`, `requireRole('ADMIN')`, `validate`, then the controller (user.routes.ts:12-18). `authenticate` rejects missing or non-Bearer headers and empty tokens with 401 `UNAUTHORIZED`, and rejects tokens failing `jwt.verify` (bad signature or expired) with 401 "Invalid or expired token" (auth.middleware.ts:28-46).

`requireRole` returns 401 when `req.user` is absent and 403 `FORBIDDEN` "Insufficient permissions" when the role is not in the allowed list (auth.middleware.ts:50-57). An `OPERATOR` therefore cannot read other users' records, including their own through this route; operators use `/auth/me` instead.

The role comes from the JWT payload, not from the database. A role change or deletion of a user does not affect already-issued tokens until expiry. No user update or delete operations exist in the module, so this is currently latent. Ordering matters: authorization runs before parameter validation, so a non-admin sees 403 regardless of id format.

**Rule workflow**:
```
Request -> authenticate (401 on failure) -> requireRole ADMIN (403 on failure) -> validate UUID (400) -> controller
```

---

### Business Rule: Path parameter must be a UUID

**Overview**:
`:id` is validated with `z.string().uuid()` before reaching the service.

**Detailed description**:
`validate({ params: idParamSchema })` parses `req.params` and replaces it with the parsed result (validate.middleware.ts:21-23). Failure produces a `ValidationError` (400, code `VALIDATION_ERROR`) with details listing `path` and `message` (validate.middleware.ts:26-31).

The schema is declared locally in user.routes.ts:7. An identical `userIdParamSchema` exists in user.schemas.ts:3-5 and is not referenced by the routes (duplication). Prisma's `id` column is `Char(36)` with `uuid()` default (schema.prisma:26), consistent with the UUID check.

The validation prevents malformed values from reaching the database, but note that the check is syntactic only.

**Rule workflow**:
```
params.id -> Zod uuid() -> valid: continue | invalid: 400 VALIDATION_ERROR with details
```

---

### Business Rule: Unique email on user creation

**Overview**:
A user cannot be created with an email already registered.

**Detailed description**:
`createUser` first calls `findByEmail` and throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (HTTP 409) if a row exists (user.service.ts:22-25). This is covered by tests/auth.test.ts:20-28.

The database also enforces `@unique` on `email` (schema.prisma:27). Because the check is a separate read before the insert, two concurrent requests with the same email could both pass the check and the second insert would fail with a Prisma unique-constraint error (P2002). The module does not catch it, so that case would surface through the global error handler (its mapping of Prisma errors was not part of this component and was not verified here). Email comparison is exact as passed; case-insensitivity depends on database collation and no normalization (lowercasing or trimming) is performed in the module.

**Rule workflow**:
```
createUser(input) -> findByEmail -> exists? 409 EMAIL_ALREADY_USED : continue to hash and persist
```

---

### Business Rule: Password hashing

**Overview**:
Plain-text passwords are hashed with bcrypt (cost 10) before persistence.

**Detailed description**:
`BCRYPT_ROUNDS = 10` is a module-level constant (user.service.ts:16), not configurable through `src/config/env.ts` (not verified in env content, only that the service does not read env). The hash is stored in `passwordHash` (`VarChar(255)`). The plain password is never stored or returned.

The Zod schema bounds the password to 8-72 characters (user.schemas.ts:9; effective at auth.schemas.ts:5). The 72 maximum matches bcrypt's 72-byte input limit, although the limit is in bytes and Zod counts characters, so multi-byte characters could exceed 72 bytes and be silently truncated by bcrypt (observation, not a verified defect). Login comparison occurs in the Auth module, not here.

Seed and test helpers use the same 10 rounds (prisma/seed.ts:191, tests/helpers/factories.ts:21), consistent with the service.

**Rule workflow**:
```
input.password -> bcrypt.hash(10) -> passwordHash -> repository.create
```

---

### Business Rule: Public user projection (no credential exposure)

**Overview**:
Every user returned by the module is mapped through `toPublic`.

**Detailed description**:
`UserService.toPublic` builds a new object with `id, email, name, role, createdAt, updatedAt` (user.service.ts:42-51). It is an allow-list so newly added columns are not exposed automatically. It is used by `createUser`, `getById`, and externally by `AuthService.login` (auth.service.ts:42).

The repository itself returns full `User` rows including `passwordHash` (user.repository.ts:6-12); safety relies on callers using the service. `AuthService` receives `UserRepository` directly (auth.service.ts:23) for credentials checks, which is the legitimate need for the hash. Test tests/auth.test.ts:15-16 asserts `passwordHash` and `password` are absent from register responses.

**Rule workflow**:
```
User (Prisma row) -> toPublic -> PublicUser (without passwordHash) -> HTTP response
```

---

### Business Rule: User not found

**Overview**:
Lookup of a non-existent id returns 404.

**Detailed description**:
`getById` throws `NotFoundError('User')`, which yields status 404, code `NOT_FOUND`, message "User not found" (http-errors.ts:26-30). The controller forwards errors with `next(err)` to the global error middleware (user.controller.ts:11-13).

For `/auth/me`, a valid token whose user no longer exists also yields 404 from this rule. Because the existence check is on the database, it is the only point where token-vs-database consistency is checked for that endpoint.

**Rule workflow**:
```
findById -> null ? NotFoundError(404) : toPublic
```

---

### Business Rule: Creation input constraints and role default

**Overview**:
Creation payload: valid email up to 255 chars, password 8-72, name 2-150, role in {ADMIN, OPERATOR} defaulting to OPERATOR.

**Detailed description**:
`createUserSchema` (user.schemas.ts:7-12) defines the constraints and exports `CreateUserInput`, which is the type used by `UserService.createUser` (user.service.ts:5, 21). However, no route in the Users Module applies this schema; request validation for creation actually happens in the Auth module via `registerSchema`, which has identical fields (auth.schemas.ts:3-8). The two schemas must be kept in sync manually.

The role defaults to `OPERATOR` both in Zod and in the Prisma model (schema.prisma:30). The role is accepted from the client without any authorization check, so an unauthenticated caller of `/auth/register` can specify `ADMIN`. This is documented as an observed behavior; whether it is intended (for example bootstrap) is not stated in the code. Column limits (email 255, name 150) align with the Prisma schema.

**Rule workflow**:
```
body -> registerSchema (Auth) -> createUser(CreateUserInput) -> persisted with role (default OPERATOR)
```

---

## 4. Component Structure

```
src/modules/users/
├── user.controller.ts    # UserController.getById (HTTP handler)
├── user.service.ts       # UserService: createUser, getById, toPublic; PublicUser type
├── user.repository.ts    # UserRepository: findById, findByEmail, create (Prisma)
├── user.routes.ts        # buildUserRouter: GET /:id with auth chain
└── user.schemas.ts       # Zod schemas: userIdParamSchema, createUserSchema, CreateUserInput

Related (outside the module boundary):
src/app.ts:27-29                      # Manual dependency wiring of repository, service, controller
src/routes/index.ts:25                # Mounts router at /users
src/middlewares/auth.middleware.ts    # authenticate, requireRole
src/middlewares/validate.middleware.ts
prisma/schema.prisma:25-38            # User model, table "users"
prisma/seed.ts:190-210                # Seeds admin@oms.local (ADMIN) and an operator
```

## 5. Dependency Analysis

```
Internal Dependencies:
user.routes -> authenticate, requireRole, validate (middlewares)
user.routes -> UserController (type)
UserController -> UserService
UserService -> UserRepository
UserService -> ConflictError, NotFoundError (shared/errors)
UserService -> CreateUserInput (user.schemas)
UserRepository -> PrismaClient (@prisma/client)

Consumers of the module:
AuthService -> UserRepository, UserService (createUser, toPublic)
AuthController -> UserService (getById)
app.ts -> UserRepository, UserService, UserController (composition)
routes/index.ts -> buildUserRouter

External Dependencies:
- express 4.21.1 - routing and handler types
- zod 3.23.8 - validation
- bcrypt 5.1.1 - password hashing
- @prisma/client 5.22.0 - persistence (users table)
- jsonwebtoken 9.0.2 - used indirectly through authenticate
```

## 6. Afferent and Efferent Coupling

Counts are based on direct import/construction references found in `src/` and `tests/`.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| UserService | 3 (UserController, AuthService, AuthController, plus app.ts composition) | 3 (UserRepository, ConflictError, NotFoundError) plus bcrypt | High |
| UserRepository | 3 (UserService, AuthService, app.ts) | 1 (PrismaClient) | High |
| UserController | 2 (app.ts, routes/index.ts type) | 1 (UserService) | Low |
| buildUserRouter | 1 (routes/index.ts) | 4 (authenticate, requireRole, validate, UserController type) plus zod | Low |
| user.schemas (createUserSchema, userIdParamSchema) | 0 runtime references (CreateUserInput type used by UserService) | 1 (zod) | Low |

## 7. Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/users/{id} | GET | Get a user by UUID; requires Bearer JWT and ADMIN role; returns PublicUser |

Responses: 200 (PublicUser), 400 VALIDATION_ERROR (non-UUID id), 401 UNAUTHORIZED, 403 FORBIDDEN, 404 NOT_FOUND. The `/api/v1` prefix is inferred from test usage (tests/auth.test.ts) and mounting in src/routes/index.ts.

Indirect exposure of module logic: `POST /api/v1/auth/register` (user creation) and `GET /api/v1/auth/me` (user read), both implemented in the Auth module.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Database (Prisma `users` table) | Internal datastore | Persist and read users | Prisma Client | Rows / JS objects | No try/catch; errors propagate to global error middleware |
| Auth module | Internal module | Registration, login, profile | In-process calls | TypeScript objects | AppError subclasses propagate |
| Auth middleware | Internal middleware | AuthN/AuthZ for route | In-process | JWT (HS-based via `env.JWT_SECRET`) | 401/403 AppErrors |
| Orders module | Indirect (FK) | `User` referenced by `Order.createdBy` and `OrderStatusHistory` changedBy | DB foreign keys | n/a | n/a |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | Route -> Controller -> Service -> Repository | src/modules/users/* | Separation of concerns |
| Repository Pattern | UserRepository | user.repository.ts | Persistence abstraction over Prisma |
| Constructor Dependency Injection (manual) | Wiring in buildApp | src/app.ts:27-29 | Composition root without a DI container |
| Factory function for router | buildUserRouter(controller) | user.routes.ts:9 | Inject controller into router |
| DTO / Projection | PublicUser + toPublic | user.service.ts:7-14,42-51 | Hide sensitive fields |
| Middleware chain | authenticate, requireRole, validate | user.routes.ts:12-18 | Cross-cutting concerns |
| Typed error hierarchy | ConflictError, NotFoundError | src/shared/errors | Uniform HTTP error mapping |
| Arrow-function handler properties | `getById: RequestHandler` | user.controller.ts:7 | Preserve `this` binding |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Auth/Users integration | `role` accepted from unauthenticated register payload (auth.schemas.ts:7, user.service.ts:31) | Any caller can create an ADMIN user (privilege escalation) |
| Medium | UserService.createUser | Check-then-insert uniqueness without handling DB unique violation (user.service.ts:22-32) | Race condition may yield an unmapped DB error instead of 409 |
| Medium | Tests | No tests for GET /users/:id, role enforcement, or UserService directly | Authorization and 404 behavior unverified |
| Low | user.schemas.ts | `createUserSchema` and `userIdParamSchema` unused by module routes; duplicated in user.routes.ts:7 and auth.schemas.ts | Drift risk between duplicated schemas |
| Low | UserService | BCRYPT_ROUNDS hardcoded (user.service.ts:16) | Not environment-configurable |
| Low | Email handling | No normalization (case/trim) before lookup or insert | Possible duplicate accounts differing in case, depending on DB collation |
| Low | UserRepository | Returns full rows including passwordHash | Relies on callers to project via toPublic |
| Low | Authorization | Role read from JWT only, not revalidated against DB | Stale role until token expiry (latent; no update/delete exists) |
| Low | Password length | Zod 72-character max vs bcrypt 72-byte limit | Multi-byte passwords may be truncated |
| Info | Module scope | No list, update, delete, or create endpoints | Limited user management capability (observation) |

## 11. Test Coverage Analysis

Test files found: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts` (includes `tests/**/*.test.ts`). There is no `tests/users*` file and no unit test for the module. No coverage tooling or report is configured or present in the repository; the Coverage column is therefore unmeasured. Test counts below were not executed, only read.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| UserService.createUser | 0 | Indirect via tests/auth.test.ts (register success :6-17, duplicate email :19-28, invalid payload :30-38 which exercises Auth schema) | Not measured | Asserts status, code, and absence of passwordHash; no hash verification or role-default test |
| UserService.getById / toPublic | 0 | Indirect via tests/auth.test.ts (`GET /auth/me`, :65-77; login :43-52) | Not measured | Verifies id and email returned; no not-found case |
| UserRepository | 0 | Indirect only | Not measured | No direct tests |
| UserController.getById / GET /users/:id | 0 | 0 | 0% (inferred) | Untested: 200, 400, 401, 403, 404 paths |
| requireRole('ADMIN') on users route | 0 | 0 in this route | Not measured | No 403 test observed in the files reviewed |

Test infrastructure: `tests/helpers/factories.ts` provides `createTestUser` (bcrypt 10 rounds, direct Prisma insert, default role OPERATOR) and `loginAndGetToken`; `tests/setup.ts:15` clears the users table via `prisma.user.deleteMany()`. Tests run against a real database (no mocks), sequentially in a single fork (vitest.config.ts:10-15).
