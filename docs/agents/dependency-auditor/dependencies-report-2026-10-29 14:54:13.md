# Dependency Audit Report

Date: 2026-10-29 14:54:13
Project: order-management-api 1.0.0 (Node >=20, ESM, TypeScript)
Scope: entire project root. Ignored folders: none.
Ecosystem: single (npm). Manifest: package.json. Lockfile: package-lock.json (present; reproducibility risk is low).
Verification method: npm registry (npm view: latest dist-tag, license, deprecation flags, modification time) and `npm audit --package-lock-only` (GitHub Advisory Database). Context7 was not used for versions because the registry is the authoritative source; Firecrawl was not available. Direct dependencies only are catalogued; vulnerable transitive packages are cited only as the path through which a direct dependency is affected.

## 1. Summary

The project is a Node.js/Express 4 REST API using Prisma 5 (MySQL), zod, JWT, bcrypt and pino. It declares 8 runtime and 19 dev dependencies, all pinned to exact versions. Docker Compose uses mysql:8.0.

Main findings:
- `npm audit` reports 23 advisories against the lockfile (4 critical, 12 high, 7 moderate). Direct dependencies affected: express, zod, uuid, vitest, tsx.
- Every direct dependency is behind its latest release. Several are behind by a major version: express (4 vs 5), zod (3 vs 4), prisma and @prisma/client (5 vs 7), pino (9 vs 10), pino-http (10 vs 11), uuid (11 vs 14), vitest (2 vs 5), eslint (8 vs 10), bcrypt (5 vs 6), typescript (5 vs 7).
- eslint 8.57.1 is flagged deprecated by the registry (no longer supported). @types/uuid is a deprecated stub (uuid ships its own types). supertest 7.0.0 is flagged deprecated by the registry in favour of 7.1.3+.
- All licenses found are permissive (MIT or Apache-2.0).
- pino-http is declared but no import of it was found in src, prisma or tests (grep based).

Note on identifiers: npm audit returns GitHub Advisory IDs (GHSA). CVE identifiers were not returned by the source, so none are asserted here, except one CVE that appears in an advisory title (CVE-2026-14257, brace-expansion, transitive).

## 2. Critical Issues

| Dependency (direct) | Installed | Advisory | Issue | Fixed in |
|---|---|---|---|---|
| vitest | 2.1.4 | GHSA-9crc-q9x8-hgqq | Remote code execution when visiting a malicious website while the Vitest API server is listening | >4.1.10 |
| vitest | 2.1.4 | GHSA-5xrq-8626-4rwp | Arbitrary file read and execution when Vitest UI server is listening | >4.1.10 |
| vitest (via tinypool) | 2.1.4 | GHSA-5gmw-xhrv-c9v3, GHSA-85c8-ppgw-ccpr | Prototype pollution gadget leading to RCE (critical, tinypool <=2.1.1) | tinypool >2.1.1 |
| express (via proxy-addr) | 4.21.1 | GHSA-jqcg-44mw-7w3h | IP spoofing via IPv4-mapped IPv6 trust subnet (critical, proxy-addr 1.1.0-2.0.7); relevant only if `trust proxy` is configured (not set in src/app.ts) | 4.22.3 or 5.x per audit range |
| express (via body-parser) | 4.21.1 | GHSA-v422-hmwv-36x6, GHSA-27r3-jxmh-p828 | DoS: invalid limit silently disables size enforcement; DoS via trailing bytes after compressed body | express >4.22.2 |
| express (via path-to-regexp) | 4.21.1 | GHSA-rhx6-c78j-4q9w, GHSA-37ch-88jc-xwx2 | ReDoS in route parameter matching | express >4.22.2 |
| express (via qs) | 4.21.1 | GHSA-w7fw-mjwx-w883, GHSA-6rw7-vpxm-498p and others | DoS via arrayLimit bypass (moderate) | express >4.22.2 |
| zod | 3.23.8 | GHSA-mrhx-c7xc-m3w5 | Memory exhaustion validating oversized constrained collections (high); audit range is <4.6.6, so the 3.x line is flagged as affected | 4.6.6 or later |
| uuid | 11.0.3 | GHSA-w5hq-g745-h8pq | Missing buffer bounds check in v3/v5/v6 when `buf` is provided (moderate); the project uses only v4 without buf | 11.1.1 or later |
| tsx (via esbuild) | 4.19.2 | GHSA-67mh-4wv8-2f99 | esbuild dev server lets any website send requests and read responses (moderate, dev only) | tsx >4.19.2 |

Deprecated or end-of-support core dependencies:
- eslint 8.57.1: registry marks the version as no longer supported.
- @types/uuid 10.0.0 (registry marks latest 11.0.0 as a deprecated stub; types are bundled in uuid).
- supertest 7.0.0: registry deprecation notice recommends 7.1.3+.

Transitive-only criticals from build tooling (not direct): tar <=7.5.20 via bcrypt -> @mapbox/node-pre-gyp (install-time extraction of prebuilt binary; 12 advisories, e.g. GHSA-34x7-hfp2-rc4v). Fixed upstream in bcrypt 6.0.0 per registry latest; the bcrypt 5.1.1 lockfile resolves the vulnerable tar.

## 3. Dependencies

Runtime:

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| @prisma/client | 5.22.0 | 7.10.1 (npm "prev" tag; "latest" tag points to 8.0.0-rc.28 pre-release) | Outdated (2 majors behind stable) |
| bcrypt | 5.1.1 | 6.0.0 | Outdated (major) |
| express | 4.21.1 | 5.3.0 (4.x line: 4.22.3) | Outdated, Vulnerable |
| jsonwebtoken | 9.0.2 | 9.0.3 | Outdated (patch) |
| pino | 9.5.0 | 10.4.0 | Outdated (major) |
| pino-http | 10.3.0 | 11.0.0 | Outdated (major); apparently unused |
| uuid | 11.0.3 | 14.0.3 | Outdated, Vulnerable |
| zod | 3.23.8 | 4.6.7 (3.x line: 3.25.76) | Outdated, Vulnerable |

Dev:

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| @types/bcrypt | 5.0.2 | 6.0.0 | Outdated |
| @types/express | 4.17.21 | 5.0.6 | Outdated |
| @types/jsonwebtoken | 9.0.7 | 9.0.10 | Outdated |
| @types/node | 20.17.6 | 26.6.6 | Outdated (tracks Node 20 runtime; intentional lag is plausible) |
| @types/supertest | 6.0.2 | 7.2.1 | Outdated |
| @types/uuid | 10.0.0 | 11.0.0 | Deprecated (stub; uuid bundles types) |
| @typescript-eslint/eslint-plugin | 8.13.0 | 8.72.1 | Outdated |
| @typescript-eslint/parser | 8.13.0 | 8.72.1 | Outdated |
| eslint | 8.57.1 | 10.13.0 (8.x line final: 8.57.1) | Deprecated (end of support) |
| eslint-config-prettier | 9.1.0 | 10.1.8 | Outdated (major) |
| pino-pretty | 11.3.0 | 13.2.0 | Outdated (major) |
| prettier | 3.3.3 | 3.9.10 | Outdated (minor) |
| prisma (CLI) | 5.22.0 | 7.10.1 stable (npm "latest" tag: 8.0.0-rc.28) | Outdated (2 majors) |
| supertest | 7.0.0 | 7.3.1 | Outdated, Deprecated notice on 7.0.0 |
| tsx | 4.19.2 | 4.23.15 | Outdated, Vulnerable (dev) |
| typescript | 5.6.3 | 7.0.2 | Outdated (major) |
| vitest | 2.1.4 | 5.0.5 | Outdated, Vulnerable (critical) |

Infrastructure: mysql:8.0 Docker image. MySQL 8.0 reached upstream end of life in April 2026 (per Oracle's published lifecycle; not re-verified in this run, see section 5).

## 4. Risk Analysis

| Severity | Dependency | Issue | Details |
|---|---|---|---|
| Critical | vitest 2.1.4 | GHSA-9crc-q9x8-hgqq, GHSA-5xrq-8626-4rwp | RCE / file read when Vitest API or UI server listens; also tinypool prototype pollution RCE (GHSA-5gmw-xhrv-c9v3). Dev/test only, exposure limited to developer machines and CI running `vitest` in watch/UI mode |
| High | express 4.21.1 | body-parser, path-to-regexp, qs advisories | Request-handling DoS paths reachable through `express.json()` (src/app.ts) and route matching on a public API. Latest 4.x is 4.22.3 |
| High | zod 3.23.8 | GHSA-mrhx-c7xc-m3w5 | Memory exhaustion on oversized constrained collections; zod validates all request input (validate.middleware.ts) |
| High | eslint 8.57.1 | Deprecated | End of support; also pulls transitive js-yaml and brace-expansion advisories (high) |
| High | bcrypt 5.1.1 | Transitive tar critical advisories | Install-time native binary fetch via node-pre-gyp; supply-chain surface during `npm install` |
| Medium | prisma / @prisma/client 5.22.0 | Two majors behind | Prisma 5 is no longer the current line; fixes and features land in 7.x. Breaking changes between majors (client generation, driver adapters) expected |
| Medium | uuid 11.0.3 | GHSA-w5hq-g745-h8pq | Not exploitable via v4 usage in request-logger.middleware.ts; still a flagged version |
| Medium | tsx 4.19.2 | GHSA-67mh-4wv8-2f99 | esbuild dev server exposure during `npm run dev` |
| Medium | express 4 -> 5 | Major gap | Express 5 changes path-to-regexp syntax, async error handling and removes deprecated APIs; @types/express must move in step |
| Medium | pino 9 / pino-http 10 / pino-pretty 11 | One to two majors behind | Logging stack; pino-http declared but unused |
| Medium | mysql:8.0 image | Lifecycle | Upstream EOL (unverified, see section 5) |
| Low | supertest 7.0.0 | Registry deprecation notice | Upgrade guidance to 7.1.3+ |
| Low | @types/uuid | Deprecated stub | Redundant with uuid's own types |
| Low | jsonwebtoken 9.0.2 | Patch behind (9.0.3). Last modified 2026-07-08 | No advisory returned by audit |
| Low | typescript 5.6.3, @types/node 20.17.6, prettier, @typescript-eslint, eslint-config-prettier | Behind latest | No advisories; maintenance burden only |

Single points of failure: express (all 17 import sites, entire HTTP layer), @prisma/client (15 import sites, all repositories, enums and error mapping), zod (9 sites: env, all request schemas, error handling). A major version change in any of these touches most modules.

Maintenance burden: high. All versions are exact pins (no semver ranges), so no update arrives without a manual edit. Three core libraries (express, zod, prisma) each have a major release pending, and the toolchain (typescript, eslint, vitest) is also one to three majors behind.

License review: all direct dependencies are MIT, except Prisma (@prisma/client, prisma) and TypeScript, which are Apache-2.0. All are compatible with proprietary use of a private application (`"private": true`); Apache-2.0 carries a patent grant and NOTICE-file obligation on redistribution only. No copyleft licenses detected among direct dependencies. Transitive licenses were not enumerated.

## 5. Unverified Dependencies

| Dependency | Current Version | Reason Not Verified |
|---|---|---|
| mysql:8.0 (Docker image) | 8.0 | Not an npm package; image tag and EOL date not checked against Docker Hub or Oracle lifecycle in this run |
| Firecrawl / GitHub repository checks (all packages) | n/a | Firecrawl MCP was not available; official GitHub repository pages were not fetched. Versions, deprecation flags and modification dates come from the npm registry; vulnerabilities from npm audit (GitHub Advisory Database). Last-release dates other than "time.modified" were not independently confirmed |
| CVE identifiers | n/a | npm audit supplies GHSA IDs only; CVE mappings were not looked up and are not asserted |
| prisma / @prisma/client latest | 5.22.0 | Registry "latest" tag is a release candidate (8.0.0-rc.28) while the "prev" tag is 7.10.1; stable latest is reported as 7.10.1 on that basis |

## 6. Critical File Analysis

Only the files that depend most heavily on risky dependencies. Paths are relative to the project root.

1. `src/app.ts` - Builds the Express app, mounts `express.json({ limit: '1mb' })`. Directly exposed to the express/body-parser/qs/path-to-regexp advisories; entry point for every request.
2. `src/server.ts` - Process entry point that boots the Express app (via app.ts); any express/pino major change affects startup.
3. `src/middlewares/error.middleware.ts` - Imports express, zod and Prisma error classes. Maps ZodError and Prisma errors to HTTP responses; breaks if zod 4 or Prisma 7 change error shapes.
4. `src/middlewares/validate.middleware.ts` - Imports zod and express; every validated route passes through it. Concentrates the zod advisory (GHSA-mrhx-c7xc-m3w5) and zod 3 -> 4 API changes.
5. `src/middlewares/auth.middleware.ts` - Verifies JWTs with jsonwebtoken 9.0.2; security-critical gate for all protected routes.
6. `src/modules/auth/auth.service.ts` - Uses bcrypt 5.1.1 and jsonwebtoken for login and token signing; credential handling and the bcrypt native-binary supply chain.
7. `src/modules/orders/order.repository.ts` - Heaviest Prisma consumer (Order, OrderItem, OrderStatusHistory, Prisma namespace); core business data path, exposed to Prisma 5 -> 7 breaking changes.
8. `src/modules/orders/order.service.ts` - Order business logic using Prisma types and enums; central transactional workflow.
9. `src/modules/orders/order.schemas.ts` - Combines zod and Prisma `OrderStatus` enum; couples two libraries each facing major upgrades.
10. `src/config/env.ts` and `src/config/database.ts` (counted together as one configuration pair) - env.ts validates all environment variables with zod; database.ts instantiates PrismaClient. Both are initialization points whose failure stops the application.

Additional notable files: `src/middlewares/request-logger.middleware.ts` (uuid), `src/shared/logger/index.ts` (pino), `prisma/seed.ts` (bcrypt, Prisma, run via tsx), `tests/helpers/factories.ts` and `tests/*.test.ts` (vitest, supertest, bcrypt), `docker-compose.yml` (mysql:8.0).

## 7. Integration Notes

- express 4.21.1: HTTP framework; Router in every `*.routes.ts` (auth, customers, orders, products, users, routes/index.ts), RequestHandler types in all controllers and middlewares. 17 import sites. No `trust proxy` setting found.
- @prisma/client 5.22.0 and prisma 5.22.0: ORM for MySQL (prisma/schema.prisma, provider prisma-client-js); client instantiated in `src/config/database.ts`, injected into repositories; enums (OrderStatus, UserRole) and Prisma error classes used in schemas, services and error middleware. Migrations in prisma/migrations; seed via tsx.
- zod 3.23.8: environment validation, request schemas per module, `validate.middleware.ts`, ZodError handling.
- jsonwebtoken 9.0.2: signing in auth.service.ts, verification in auth.middleware.ts.
- bcrypt 5.1.1: password hashing in auth.service.ts, user.service.ts, seed and test factories (native module).
- uuid 11.0.3: request IDs (v4) in request-logger.middleware.ts. @types/uuid is redundant.
- pino 9.5.0: logger in `src/shared/logger/index.ts`. pino-http 10.3.0: declared, no import found. pino-pretty 11.3.0: dev log formatting, referenced as transport target in src/shared/logger/index.ts.
- vitest 2.1.4, supertest 7.0.0, @types/supertest: test runner and HTTP assertions in `tests/`.
- tsx 4.19.2: `dev` and `db:seed` scripts. typescript 5.6.3: build via `tsc`. eslint, @typescript-eslint/*, eslint-config-prettier, prettier: lint and format tooling.
- No forks, patches or vendored copies were found.

## 8. Report Storage

Saved to: docs/agents/dependency-auditor/dependencies-report-2026-10-29 14:54:13.md
