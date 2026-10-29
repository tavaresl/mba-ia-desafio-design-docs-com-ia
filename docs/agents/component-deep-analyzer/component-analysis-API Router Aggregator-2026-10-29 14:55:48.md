# Component Deep Analysis Report: API Router Aggregator (src/routes/index.ts)

Date: 2026-10-29 14:55:48

## 1. Executive Summary

The API Router Aggregator is a single 31-line TypeScript module (`src/routes/index.ts`) that composes the five module-level Express routers (auth, users, customers, products, orders) into one root `Router`. It exports:

- `Controllers` (type, lines 13-19): the contract of controller instances the aggregator requires.
- `buildApiRouter(controllers)` (function, lines 21-31): factory that creates an Express `Router`, mounts each module router under a fixed path prefix, and returns it.

It is consumed only by `src/app.ts:67`, which mounts the result at `/api/v1`. The component holds no business logic, validation, authentication, or I/O. Its only responsibilities are route-prefix assignment and dependency wiring of controllers to per-module router factories.

Key findings:

- Pure composition root for routing; stateless, side-effect free apart from building the router.
- Mount order: auth, users, customers, products, orders. Paths do not overlap, so order has no functional effect today.
- Authentication and authorization are not applied at the aggregator level. Each module router applies them itself (inconsistently in structure: `router.use(authenticate)` for customers/products/orders, per-route for users and auth).
- No direct automated tests target the aggregator. It is exercised indirectly through the supertest suites (`tests/auth.test.ts`, `tests/orders.test.ts`). Users, customers, and products routes have no tests.
- No 404 handling inside the aggregator; unmatched routes fall to the catch-all in `src/app.ts:69-71`.

## 2. Data Flow Analysis

Startup (wiring) flow:

```
1. src/server.ts creates the Prisma client and calls buildApp (src/app.ts:55)
2. buildApp calls buildControllers(prisma) (src/app.ts:66), which builds Repository -> Service -> Controller chains for each module (src/app.ts:26-53)
3. The Controllers object { auth, users, customers, products, orders } is passed to buildApiRouter (src/routes/index.ts:21)
4. buildApiRouter creates Router() (line 22)
5. Each controller is passed to its module factory (buildAuthRouter, buildUserRouter, buildCustomerRouter, buildProductRouter, buildOrderRouter)
6. Each returned sub-router is mounted with router.use(prefix, subRouter) (lines 24-28)
7. The root router is returned and mounted at /api/v1 (src/app.ts:67)
```

Request flow (runtime, through the aggregator):

```
1. HTTP request enters Express app: x-powered-by disabled, express.json (1mb), requestLogger (src/app.ts:58-60)
2. /health is handled before the aggregator (src/app.ts:62)
3. Path starting with /api/v1 enters the root router built by buildApiRouter
4. Express matches the prefix (/auth, /users, /customers, /products, /orders) and strips it, delegating to the sub-router
5. Sub-router applies its middleware chain (authenticate, requireRole, validate) then the controller method
6. Errors passed to next(err) skip to errorMiddleware (src/app.ts:73)
7. If no sub-router handles the path, the request falls through the aggregator to the NotFoundError catch-all (src/app.ts:69-71)
```

## 3. Business Rules & Logic

The component contains no domain business logic. The rules below are structural routing rules (all high confidence, explicit in code). Rules enforced by the mounted sub-routers are listed for context since they define effective behavior reached via the aggregator, but they are implemented outside this component.

## Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Routing | Auth module is mounted at `/auth` | src/routes/index.ts:24 |
| Routing | User module is mounted at `/users` | src/routes/index.ts:25 |
| Routing | Customer module is mounted at `/customers` | src/routes/index.ts:26 |
| Routing | Product module is mounted at `/products` | src/routes/index.ts:27 |
| Routing | Order module is mounted at `/orders` | src/routes/index.ts:28 |
| Composition | All five controllers are mandatory (non-optional type fields) | src/routes/index.ts:13-19 |
| Composition | Aggregator receives controllers by injection; it constructs none | src/routes/index.ts:21 |
| Versioning | `/api/v1` prefix is applied externally by the consumer | src/app.ts:67 |
| Delegated (context) | Auth: register/login public, `/me` requires authentication | src/modules/auth/auth.routes.ts:10-12 |
| Delegated (context) | Users: GET `/:id` requires authentication plus ADMIN role plus UUID param | src/modules/users/user.routes.ts:12-18 |
| Delegated (context) | Customers, products, orders: all routes require authentication | *.routes.ts:14 |
| Delegated (context) | Order status change is a sub-resource PATCH `/:id/status` | src/modules/orders/order.routes.ts:19-23 |

## Detailed breakdown of the business rules:
---

### Business Rule: Fixed Module Path Prefix Mapping

**Overview**:
Each domain module is exposed under a hard-coded, plural, lowercase resource prefix, except auth which uses a singular verb-like namespace.

**Detailed description**:
`buildApiRouter` calls `router.use(prefix, subRouter)` five times (src/routes/index.ts:24-28). The prefixes are string literals: `/auth`, `/users`, `/customers`, `/products`, `/orders`. Combined with the external `/api/v1` mount in src/app.ts:67, the full public URLs are `/api/v1/auth/...`, `/api/v1/users/...`, and so on. This is the single place where a module's public URL namespace is defined.

Because Express strips the mount prefix before delegating, each sub-router defines paths relative to its own root (for example `/:id`, `/register`). The sub-routers therefore have no knowledge of their public prefix, making the aggregator the sole authority of the URL layout. The tests confirm the resulting URLs: `/api/v1/auth/register`, `/api/v1/auth/login`, `/api/v1/auth/me` (tests/auth.test.ts:10,45,71) and `/api/v1/orders`, `/api/v1/orders/{id}/status` (tests/orders.test.ts:20,74).

The prefixes do not overlap, so first-match semantics of Express do not produce shadowing today. Adding a module requires adding an entry in `Controllers` and a `router.use` line here (and a controller in `buildControllers`).

**Rule workflow**:
```
Request /api/v1/<prefix>/<rest>
 -> app.use('/api/v1') strips /api/v1
 -> root router matches <prefix> in order (auth, users, customers, products, orders)
 -> prefix stripped, <rest> handled by module router
 -> no match: continues to app-level NotFoundError handler
```

---

### Business Rule: Mandatory Controller Contract (Controllers type)

**Overview**:
The aggregator can only be built if all five controller instances are supplied; the compiler enforces this.

**Detailed description**:
The `Controllers` type (src/routes/index.ts:13-19) declares five required properties typed with the concrete controller classes, imported with `import type` (lines 2-6) so no runtime dependency on controller modules exists. Omitting any controller is a compile-time error, so a route group cannot be silently unmounted.

The type is also the return type of `buildControllers` in src/app.ts:26, which means the type acts as the shared contract between the composition root (app.ts) and the routing layer. Typing against concrete classes rather than interfaces ties the aggregator to the specific controller implementations (relevant to testability; no tests substitute controllers).

There is no runtime validation: if a caller bypassed types and passed an undefined controller, the module router factories would throw or register handlers referencing undefined properties (the factories read methods such as `controller.register` at registration time, yielding undefined handlers, which Express rejects with an error at startup). This behavior is inferred, not explicit in this file.

**Rule workflow**:
```
buildControllers() -> Controllers object (type-checked)
 -> buildApiRouter(controllers)
 -> controllers.<module> passed to matching buildXRouter
```

---

### Business Rule: Authorization Responsibility Delegated to Module Routers

**Overview**:
The aggregator applies no global authentication; each sub-router declares its own access policy.

**Detailed description**:
The aggregator does not call `router.use(authenticate)`. Consequently, the public vs protected status of endpoints is decided in sub-routers. Auth routes: register and login are unauthenticated, `/me` uses `authenticate` (src/modules/auth/auth.routes.ts:10-12). Users: `GET /:id` chains `authenticate`, `requireRole('ADMIN')`, `validate` (user.routes.ts:12-18). Customers, products, and orders call `router.use(authenticate)` at the top of their factory (customer.routes.ts:14, product.routes.ts:14, order.routes.ts:14), protecting every route in the module, but note that no role restriction is applied to them (any authenticated user, ADMIN or OPERATOR).

Because the policy is distributed, a new module mounted via this aggregator is public by default unless its router adds authentication. This is a structural property of the design rather than an explicit rule in the code. Confidence: high for the description of present behavior, as read from the files.

An authenticated request is verified by `authenticate` (src/middlewares/auth.middleware.ts:27-47): Bearer header required, JWT verified with `env.JWT_SECRET`, `req.user` set. Failures produce UnauthorizedError; `requireRole` produces ForbiddenError.

**Rule workflow**:
```
Request -> aggregator prefix match -> sub-router
 -> [authenticate] -> [requireRole] -> [validate] -> controller
 -> failure at any step: next(error) -> errorMiddleware
```

---

### Business Rule: Unmatched Route Fallthrough

**Overview**:
Requests under `/api/v1` that match no sub-route are not answered by the aggregator; they fall through to a 404 handler.

**Detailed description**:
The aggregator defines no catch-all. If a request such as `/api/v1/unknown` or an unsupported method on a known path matches nothing in the sub-routers, Express continues out of the root router to the next app-level middleware, which raises `NotFoundError("Route METHOD originalUrl")` (src/app.ts:69-71), then rendered by `errorMiddleware`.

An important consequence involves the order of middleware inside the sub-routers: for customers, products, and orders, `router.use(authenticate)` runs for every request that reaches the sub-router prefix, including unknown subpaths such as `/api/v1/orders/x/y/z`. An unauthenticated request to an unknown path under these prefixes therefore yields 401 rather than 404, whereas an unknown path under `/api/v1/users` or `/api/v1/auth` yields 404. This is derived from the code and is not covered by tests.

**Rule workflow**:
```
Request under /api/v1 -> no sub-router route matches
 -> (customers/products/orders: authenticate may reject first with 401)
 -> otherwise app-level NotFoundError -> errorMiddleware
```

---

## 4. Component Structure

```
src/routes/
└── index.ts        # Router aggregator: Controllers type + buildApiRouter factory

Related (outside component boundary, consumed by it):
src/modules/
├── auth/auth.routes.ts            # buildAuthRouter
├── users/user.routes.ts           # buildUserRouter
├── customers/customer.routes.ts   # buildCustomerRouter
├── products/product.routes.ts     # buildProductRouter
└── orders/order.routes.ts         # buildOrderRouter
src/app.ts                         # Only consumer (imports at line 19, mounts at line 67)
```

## 5. Dependency Analysis

```
Internal Dependencies:
src/app.ts -> buildApiRouter (routes/index.ts) -> buildAuthRouter -> AuthController (type)
                                               -> buildUserRouter -> UserController (type)
                                               -> buildCustomerRouter -> CustomerController (type)
                                               -> buildProductRouter -> ProductController (type)
                                               -> buildOrderRouter -> OrderController (type)

External Dependencies:
- express (4.21.1) - Router factory (import at src/routes/index.ts:1)
```

Transitive (via sub-routers, not direct): zod 3.23.8 (validation), jsonwebtoken 9.0.2 (authentication), Prisma 5.22.0 (via controllers' services). Controller imports in the aggregator are type-only (erased at compile time).

## 6. Afferent and Efferent Coupling

Unit of analysis: exported functions, types, and imported symbols (TypeScript modules).

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| buildApiRouter | 1 (src/app.ts) | 6 (Router + 5 router factories) | High (sole entry to all API routes) |
| Controllers (type) | 2 (src/app.ts import and buildControllers return type; buildApiRouter parameter) | 5 (controller types) | Medium |
| src/routes/index.ts (module) | 1 | 11 import statements (express + 5 types + 5 factories) | High |
| buildAuthRouter | 1 | not measured here | Medium |
| buildUserRouter | 1 | not measured here | Medium |
| buildCustomerRouter | 1 | not measured here | Medium |
| buildProductRouter | 1 | not measured here | Medium |
| buildOrderRouter | 1 | not measured here | Medium |

Afferent counts are based on a repository-wide grep: `buildApiRouter` and `Controllers` are referenced only in src/app.ts and src/routes/index.ts.

## 7. Endpoints

The aggregator itself defines no handlers; it defines the prefixes. Resulting endpoints (all prefixed by `/api/v1`, defined in sub-routers, listed for completeness). Auth column describes sub-router middleware.

| Endpoint | Method | Description | Access (from sub-router) |
|----------|--------|-------------|--------------------------|
| /api/v1/auth/register | POST | Register a user (body validated by registerSchema) | Public |
| /api/v1/auth/login | POST | Login (loginSchema) | Public |
| /api/v1/auth/me | GET | Current authenticated user | Authenticated |
| /api/v1/users/{id} | GET | Get user by ID (UUID param) | ADMIN only |
| /api/v1/customers | GET | List customers (query validated) | Authenticated |
| /api/v1/customers/{id} | GET | Get customer | Authenticated |
| /api/v1/customers | POST | Create customer | Authenticated |
| /api/v1/customers/{id} | PATCH | Update customer | Authenticated |
| /api/v1/customers/{id} | DELETE | Delete customer | Authenticated |
| /api/v1/products | GET | List products | Authenticated |
| /api/v1/products/{id} | GET | Get product | Authenticated |
| /api/v1/products | POST | Create product | Authenticated |
| /api/v1/products/{id} | PATCH | Update product | Authenticated |
| /api/v1/products/{id} | DELETE | Delete product | Authenticated |
| /api/v1/orders | GET | List orders | Authenticated |
| /api/v1/orders/{id} | GET | Get order | Authenticated |
| /api/v1/orders | POST | Create order | Authenticated |
| /api/v1/orders/{id}/status | PATCH | Change order status | Authenticated |
| /api/v1/orders/{id} | DELETE | Delete order | Authenticated |

Outside the aggregator: `GET /health` at src/app.ts:62.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express Router | Framework | Route composition | In-process | N/A | None in component |
| src/app.ts | Internal consumer | Mounts router at /api/v1 | In-process call | Router object | App-level NotFoundError and errorMiddleware |
| Module router factories (5) | Internal | Provide sub-routers | In-process call | Router objects | Delegated to middleware (validate, authenticate) |
| Controllers | Internal | Request handlers | In-process | Express handlers | Delegated |

The component has no direct database, network, or external service integration.

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Factory Function | `buildApiRouter` | src/routes/index.ts:21 | Constructs the router from injected dependencies |
| Dependency Injection (constructor/parameter) | `Controllers` object parameter | src/routes/index.ts:13-21 | Decouples routing from controller construction |
| Composite / Aggregator (Router composition) | `router.use(prefix, subRouter)` | src/routes/index.ts:24-28 | Single root router built from modular sub-routers |
| Composition Root (external) | `buildControllers` | src/app.ts:26 | Wires Repository -> Service -> Controller |
| Modular / feature-based layout | `src/modules/*` | src/modules | Each module owns its router, schemas, controller |
| Type-only imports | `import type` | src/routes/index.ts:2-6 | Avoids runtime coupling to controller classes |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | Access policy distribution | No default authentication at aggregator; each sub-router must opt in | A newly mounted module is public unless it adds authenticate |
| Medium | Customers/products/orders routers | `router.use(authenticate)` runs on any subpath, so unknown paths return 401 for anonymous users and 404 for authenticated ones | Inconsistent error semantics, route-existence disclosure only to authenticated users |
| Low | Prefix literals | Prefixes hard-coded as string literals, no shared constants | Duplication risk if prefixes are referenced elsewhere; none found currently |
| Low | Controllers type | Typed to concrete classes, not interfaces | Lower substitutability for tests |
| Low | Versioning | `/api/v1` lives in src/app.ts, split from the aggregator | Route layout is defined in two files |
| Low | Role authorization | Customers, products, orders, accept any role (ADMIN or OPERATOR); only users GET is ADMIN-restricted | Possible intent mismatch; not determinable from code (ambiguity, no documented requirement found in this scope) |
| Medium | Test coverage | No dedicated test for the aggregator; users, customers, products routes untested | Mounting regressions would be caught only for auth and orders |

## 11. Test Coverage Analysis

Test files located in the project: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`. Configuration: `vitest.config.ts` (includes `tests/**/*.test.ts`, single fork, no parallelism, setup file `tests/setup.ts`). No coverage provider is configured, so coverage percentages are not available. No tests were executed.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| src/routes/index.ts (direct) | 0 | 0 | Not measured | No direct test |
| Auth mount (/api/v1/auth) | 0 | 7 (tests/auth.test.ts:6,19,30,41,54,65,79) | Not measured | Verifies prefix mapping via supertest on register, login, me |
| Orders mount (/api/v1/orders) | 0 | 8 (tests/orders.test.ts:12,42,59,89,109,134,163,197) | Not measured | Verifies prefix and sub-route `/:id/status`; covers create, list filter, status transitions, delete |
| Users mount (/api/v1/users) | 0 | 0 | Not measured | Untested; ADMIN-only rule not verified |
| Customers mount (/api/v1/customers) | 0 | 0 | Not measured | Untested |
| Products mount (/api/v1/products) | 0 | 0 | Not measured | Untested |
| Unmatched route fallthrough | 0 | 0 | Not measured | Untested (404 and anonymous 401 on unknown paths) |

Observation: tests use the real `buildApp` (full stack with database), so the aggregator is verified only through end-to-end behavior for two of five mounts.
