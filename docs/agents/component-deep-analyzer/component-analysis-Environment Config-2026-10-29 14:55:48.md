# Component Deep Analysis Report: Environment Config (src/config/env.ts)

Date: 2026-10-29 14:55:48

## 1. Executive Summary

Environment Config is a single-file module (`src/config/env.ts`, 28 lines) that validates `process.env` with a Zod schema at module-load time and exports a typed, effectively immutable-by-convention `env` object and an `Env` type. It is the sole typed gateway to runtime configuration for the Order Management API (service name `order-management-api`).

Key findings:
- Fail-fast: on invalid configuration the process prints all issues and calls `process.exit(1)` (src/config/env.ts:21-22).
- Six variables are validated: NODE_ENV, PORT, LOG_LEVEL, DATABASE_URL, JWT_SECRET, JWT_EXPIRES_IN.
- Validation happens as an import side effect (`export const env = loadEnv()` at src/config/env.ts:27); any file importing it triggers validation.
- The module does not load `.env` itself; loading is delegated to Node `--env-file=.env` in the npm scripts (package.json:11, 13).
- SHADOW_DATABASE_URL is consumed by Prisma (prisma/schema.prisma:8) but is not part of the schema; the Docker MYSQL_* variables are likewise outside the schema.
- No dedicated tests exist for this component.

## 2. Data Flow Analysis

```
1. Process starts (npm run dev/start use Node --env-file=.env; tests via vitest run with no env-file flag)
2. First import of config/env.js evaluates the module
3. envSchema.safeParse(process.env) (env.ts:15) applies defaults, coercion and constraints
4a. Failure: issues formatted as "  - <path>: <message>" (env.ts:17-19), written with console.error (env.ts:21), process.exit(1) (env.ts:22)
4b. Success: parsed.data returned (env.ts:24) and exported as `env` (env.ts:27)
5. Consumers read fields:
   - server.ts: PORT, NODE_ENV
   - shared/logger/index.ts: LOG_LEVEL, NODE_ENV
   - config/database.ts: NODE_ENV (Prisma log levels)
   - middlewares/auth.middleware.ts: JWT_SECRET (jwt.verify)
   - modules/auth/auth.service.ts: JWT_SECRET, JWT_EXPIRES_IN (jwt.sign, response expiresIn)
6. Prisma CLI separately reads DATABASE_URL and SHADOW_DATABASE_URL via env() in schema.prisma
```

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation / Default | NODE_ENV in {development, test, production}, default development | src/config/env.ts:4 |
| Validation / Default | PORT coerced to integer, positive, default 3000 | src/config/env.ts:5 |
| Validation / Default | LOG_LEVEL in pino levels, default info | src/config/env.ts:6 |
| Validation | DATABASE_URL required, non-empty string | src/config/env.ts:7 |
| Security Validation | JWT_SECRET minimum 16 characters | src/config/env.ts:8 |
| Default | JWT_EXPIRES_IN string, default 8h | src/config/env.ts:9 |
| Error Handling | Invalid config aborts process with exit code 1 and aggregated report | src/config/env.ts:14-23 |
| Lifecycle | Configuration validated once at import and exported as singleton | src/config/env.ts:27 |

### Detailed breakdown of the business rules

---

### Business Rule: Runtime Mode Selection (NODE_ENV)

**Overview**: NODE_ENV must be one of development, test, production; absent means development.

**Detailed description**: The enum restricts the application to three known modes. Any other value (for example "staging") fails validation and terminates startup. The default of development means a missing variable silently selects the most permissive mode.

NODE_ENV drives behavior in three consumers: the logger enables the pino-pretty transport only in development (src/shared/logger/index.ts:22-27) and adds the value to every log record's base fields (line 20); Prisma logs ['warn','error'] in development and only ['error'] otherwise (src/config/database.ts:6); server.ts logs the mode at startup (src/server.ts:10).

Note: the schema is an enum with default, so an empty string (e.g. `NODE_ENV=`) is not treated as undefined by Zod and would fail the enum check (inferred from Zod behavior; not exercised in code).

**Rule workflow**:
```
NODE_ENV undefined -> "development"
NODE_ENV in enum   -> accepted
otherwise          -> issue "NODE_ENV: Invalid enum value..." -> exit(1)
```

---

### Business Rule: HTTP Port (PORT)

**Overview**: PORT is coerced from string to number, must be a positive integer, default 3000.

**Detailed description**: `z.coerce.number()` converts the string env value via Number(). `.int()` rejects fractional values and `.positive()` rejects zero and negatives. There is no upper bound check (values above 65535 pass validation) and no minimum privileged-port check.

Coercion has an edge case: an empty string coerces to 0 and fails `.positive()`, while a non-numeric string coerces to NaN and fails as invalid number. The value is used only by `app.listen(env.PORT)` in src/server.ts:9.

**Rule workflow**:
```
PORT undefined -> 3000
PORT string -> Number() -> integer? -> > 0? -> accepted
failure at any step -> issue -> exit(1)
```

---

### Business Rule: Log Verbosity (LOG_LEVEL)

**Overview**: LOG_LEVEL must be a pino level (fatal, error, warn, info, debug, trace), default info.

**Detailed description**: The enum mirrors pino's standard levels, so the value can be passed directly to pino without further checking. Custom levels or "silent" are rejected; notably "silent", which pino supports, is not allowed by this schema.

The only consumer is createLogger (src/shared/logger/index.ts:15). Unrecognized values abort startup rather than falling back to a default.

**Rule workflow**:
```
LOG_LEVEL undefined -> "info"
in enum -> accepted -> pino level
else -> exit(1)
```

---

### Business Rule: Database Connection String Required (DATABASE_URL)

**Overview**: DATABASE_URL must be present and non-empty; there is no default.

**Detailed description**: The only constraint is `min(1)` with the custom message "DATABASE_URL is required". Format (mysql:// scheme, host, credentials) is not validated by this module; format errors surface later when Prisma connects.

Although declared in the schema, `env.DATABASE_URL` is not read by any TypeScript consumer in src/. Prisma reads the variable directly from the process environment via `env("DATABASE_URL")` (prisma/schema.prisma:7), and `new PrismaClient()` in src/config/database.ts passes no datasourceUrl. The validation therefore acts as a startup guard, not a value provider. Because database.ts imports env.js (line 2), the guard runs before the Prisma client is created.

**Rule workflow**:
```
DATABASE_URL missing/empty -> "DATABASE_URL: DATABASE_URL is required" -> exit(1)
DATABASE_URL non-empty -> accepted (format unchecked)
```

---

### Business Rule: JWT Secret Minimum Strength (JWT_SECRET)

**Overview**: JWT_SECRET is required with at least 16 characters; no default.

**Detailed description**: The length check enforces a minimal security floor, preventing trivially short HMAC signing keys. Absence yields a Zod "Required" issue; shorter values yield the custom message "JWT_SECRET must be at least 16 characters".

The secret is used to sign tokens (src/modules/auth/auth.service.ts:49) and verify them (src/middlewares/auth.middleware.ts:41). There is no entropy check, no maximum, and no rejection of placeholder values: the sample `.env.example` value ("change-me-in-production-please-use-a-long-random-secret") has more than 16 characters and would pass validation even in production.

**Rule workflow**:
```
JWT_SECRET undefined -> "Required" -> exit(1)
length < 16 -> custom message -> exit(1)
else accepted
```

---

### Business Rule: Token Lifetime (JWT_EXPIRES_IN)

**Overview**: JWT_EXPIRES_IN is any string, default "8h".

**Detailed description**: The schema only checks that the value is a string. Whether it is a valid duration (jsonwebtoken accepts forms like "8h", "15m", or a numeric string in seconds) is not verified here. An invalid value would pass startup and fail or misbehave at signing time.

The value is cast with `as SignOptions['expiresIn']` (src/modules/auth/auth.service.ts:48) and also returned verbatim to clients as `expiresIn` in the login response (line 43), so the raw string leaks into the API contract.

**Rule workflow**:
```
undefined -> "8h"
string -> accepted (not validated as a duration)
```

---

### Business Rule: Fail-Fast Aggregated Error Reporting

**Overview**: All validation issues are collected and reported together, then the process exits with code 1.

**Detailed description**: `safeParse` is used rather than `parse`, so Zod returns every issue, not just the first. Each issue is rendered as `  - <dotted path>: <message>` and joined by newlines, producing a single console.error message headed "Invalid environment configuration:" (src/config/env.ts:17-21).

The module uses console.error rather than the pino logger, which is consistent with the logger itself depending on env (circular dependency avoided). An eslint-disable comment for no-console is present (line 20). process.exit(1) happens synchronously at import, so importing modules (and test runners) terminate if variables are missing. Secret values are not echoed in messages (only path and message), though Zod messages for invalid enums list allowed options.

**Rule workflow**:
```
safeParse(process.env)
 success -> return data
 failure -> map issues -> console.error -> process.exit(1)
```

---

### Business Rule: Singleton Evaluated at Import

**Overview**: `env` is computed once at module load and shared by all importers.

**Detailed description**: ES module caching ensures loadEnv runs once per process. Subsequent changes to process.env are not reflected. The exported object is typed `Env` but not frozen (no Object.freeze or readonly), so mutation is possible by convention only.

Because the evaluation is an import side effect, any module that imports env (directly or transitively, including the logger, database client and auth middleware) forces validation. Test setup (tests/setup.ts:2) imports config/database.js, thereby triggering this validation in the test process.

**Rule workflow**:
```
first import -> loadEnv() -> cached export
later imports -> same object
```

---

## 4. Component Structure

```
src/config/
├── env.ts        # Zod schema, loadEnv(), exported `env` and `Env` type
└── database.ts   # Prisma client factory (consumer of env; adjacent, not part of the component)
.env.example      # Template of variables (includes vars outside the schema)
docker-compose.yml# MySQL service using MYSQL_* variables with defaults (outside schema)
```

## 5. Dependency Analysis

```
Internal Dependencies:
env.ts -> (none)

Dependents:
server.ts, shared/logger/index.ts, config/database.ts,
middlewares/auth.middleware.ts, modules/auth/auth.service.ts

External Dependencies:
- zod 3.23.8 (package.json) - schema validation
- Node.js process.env / process.exit / console
- Node --env-file flag (package.json:11, 13) - .env loading (no dotenv package used)
```

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| env (module export) | 5 modules (server.ts, logger/index.ts, database.ts, auth.middleware.ts, auth.service.ts) | 1 (zod) | High |
| envSchema (internal) | 1 (loadEnv) | 1 (zod) | Low |
| loadEnv (internal) | 1 (module export) | 3 (envSchema, process, console) | Low |
| Env type | 0 explicit imports found | 1 (zod infer) | Low |

Note: tests/ and prisma/ do not import env.ts directly.

## 7. Endpoints

Not applicable: the component exposes no endpoints.

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| process.env | Runtime | Source of configuration | In-process | String key/values | Aggregated validation, exit(1) |
| .env file | File | Variable source | Node --env-file | KEY=VALUE | Node-level; env.ts does not handle a missing file |
| Logger (pino) | Internal | LOG_LEVEL, NODE_ENV | In-process | Typed object | None needed |
| Prisma (database.ts, schema.prisma) | Internal/Tooling | NODE_ENV for log level; DATABASE_URL guard | In-process / CLI | String | Prisma reads DATABASE_URL independently |
| jsonwebtoken | Internal | JWT_SECRET, JWT_EXPIRES_IN | In-process | String | None at config level |

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Fail-fast validation | safeParse + process.exit(1) | src/config/env.ts:14-23 | Prevent starting with bad config |
| Schema as single source of truth | `z.infer<typeof envSchema>` | src/config/env.ts:12 | Types derived from runtime schema |
| Module singleton | Top-level `export const env` | src/config/env.ts:27 | Shared configuration |
| Twelve-factor config | Environment variables | whole module | Config separated from code |

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Medium | JWT_SECRET | Only length check; example placeholder passes in any NODE_ENV | Weak or default secret could reach production |
| Medium | Import side effect with process.exit | Exit at import time | Hard to unit test; terminates test runner |
| Medium | Test environment | tests script has no --env-file; DATABASE_URL/JWT_SECRET must exist in the shell | Test run exits with code 1 if not exported (inferred from package.json and vitest.config.ts) |
| Low | JWT_EXPIRES_IN | Not validated as a duration | Failure deferred to sign time; raw string in API response |
| Low | PORT | No upper bound | Invalid ports pass validation |
| Low | DATABASE_URL | Format not validated; value unused in TS | Guard only; Prisma reads env separately |
| Low | SHADOW_DATABASE_URL | Used by prisma/schema.prisma:8 but not in schema | Prisma commands fail without it, outside the validation |
| Low | LOG_LEVEL | "silent" not permitted | Cannot silence logs via config |
| Low | Env object | Not frozen | Mutable by convention |
| Low | NODE_ENV default | Defaults to development | Missing variable in deployment enables pretty logs and verbose Prisma logging |

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| env.ts | 0 | 0 direct | Not measured (no coverage config found in vitest.config.ts); only indirect load via imports | No tests for defaults, coercion, failure path or exit behavior |

Test files located: tests/auth.test.ts, tests/orders.test.ts, tests/setup.ts, tests/helpers/factories.ts. None reference env.ts or process.env. tests/setup.ts imports src/config/database.js, which transitively loads env.ts. Auth tests exercise JWT signing/verification indirectly (inferred; test bodies were not analyzed in detail for this component). vitest.config.ts uses setupFiles tests/setup.ts, single fork, 30s timeouts.
