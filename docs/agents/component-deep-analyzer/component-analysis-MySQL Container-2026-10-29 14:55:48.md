# Component Deep Analysis Report: MySQL Container (docker-compose.yml)

Analysis date: 2026-10-29 14:55:48
Component boundary: `docker-compose.yml` (service `mysql`, container `oms-mysql`, volume `oms_mysql_data`), plus the configuration that parameterizes it (`.env.example`) and the contract files that bind consumers to it (`src/config/env.ts`, `src/config/database.ts`, `prisma/schema.prisma`, `prisma/migrations/`). Application logic that merely consumes the database is referenced only where it defines how the container is used.

## 1. Executive Summary

The MySQL Container is the single infrastructure component of the Order Management System (OMS). It is a Docker Compose definition that runs one `mysql:8.0` server (`docker-compose.yml:2-3`) named `oms-mysql`, publishes it on host port 3306, persists data in a local named volume, and exposes a health check. It is the only datastore of the project: all domain data (users, customers, products, orders, order items, status history, order-number sequence) is stored here and accessed through Prisma 5.22.0.

Key findings:

- The container has no custom initialization scripts. Schema creation is delegated to Prisma Migrate (`prisma/migrations/20260519182739_init/migration.sql`), not to `/docker-entrypoint-initdb.d`.
- All credentials have in-file defaults (`root_password`, `oms_user`/`oms_password`) through `${VAR:-default}` substitution (`docker-compose.yml:7-10`).
- The server is forced to `mysql_native_password`, `utf8mb4` and `utf8mb4_unicode_ci` (`docker-compose.yml:23-25`). The migration DDL repeats the same charset and collation on every table.
- The health check authenticates as root over `mysqladmin ping` with the root password interpolated into the command line (`docker-compose.yml:18`).
- Prisma uses a shadow database (`oms_shadow`) accessed as root (`.env.example:5`). Nothing in the container creates that database or grants the application user rights to it; Prisma Migrate creates it on demand with the root user (ambiguity noted in Section 10).
- The test suite connects to the same `DATABASE_URL` as development and deletes all rows from every table before each test (`tests/setup.ts:9-16`). No isolated test database is provisioned by the container.
- No backup, replication, TLS, resource limit or network isolation is configured.
- The business-relevant rules this component enforces are data-integrity rules realized in MySQL DDL (uniqueness, foreign keys, enums, cascade rules, charset) and the container's runtime policies (restart, persistence, health gating).

## 2. Data Flow Analysis

Provisioning and startup flow:

```
1. Developer copies .env.example to .env (variables MYSQL_ROOT_PASSWORD, MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD)
2. `docker compose up` reads docker-compose.yml and substitutes ${VAR:-default} values (docker-compose.yml:7-10)
3. Image mysql:8.0 starts container oms-mysql with command flags (auth plugin, utf8mb4, utf8mb4_unicode_ci)
4. On first start with an empty volume, the image entrypoint initializes /var/lib/mysql, sets the root password, creates database MYSQL_DATABASE and user MYSQL_USER with MYSQL_PASSWORD
5. Volume oms_mysql_data (driver local) persists the data directory across restarts
6. Healthcheck runs `mysqladmin ping -h localhost -uroot -p<root password>` every 5s, after a 30s start_period, up to 20 retries, 5s timeout
7. Port 3306 is published on the host so the application and Prisma CLI can connect
8. `npm run db:migrate` (prisma migrate dev) applies prisma/migrations/20260519182739_init/migration.sql using DATABASE_URL, and uses SHADOW_DATABASE_URL (root user, database oms_shadow) for drift detection
9. `npm run db:seed` runs prisma/seed.ts to insert sample data
```

Runtime data flow (application to container):

```
1. HTTP request reaches the Express app (src/app.ts)
2. Module service calls Prisma repository or $transaction (e.g., src/modules/orders/order.service.ts:58, :131)
3. PrismaClient (src/config/database.ts:4-8) opens TCP connection to localhost:3306 using DATABASE_URL (mysql://oms_user:oms_password@localhost:3306/oms)
4. MySQL executes statements as oms_user against database oms
5. Constraint violations return as Prisma errors (P2002 unique, P2025 not found), mapped to HTTP 409/404 in src/middlewares/error.middleware.ts:37-54
6. On SIGINT/SIGTERM, server.ts:16 calls prisma.$disconnect(); the container itself keeps running (restart: unless-stopped)
```

Test data flow:

```
1. vitest loads tests/setup.ts (vitest.config.ts:12), which imports the shared prisma client
2. beforeAll connects; beforeEach truncates, via deleteMany, orderStatusHistory, orderItem, order, orderNumberSequence, product, customer, user (child-first order)
3. Tests execute against the same MySQL instance and database defined by DATABASE_URL
```

## 3. Business Rules & Logic

The component is infrastructure, so its "business rules" are operational and data-integrity policies. Rules realized by the schema are in scope because the schema lives in the database hosted by this container. Confidence is High unless stated.

### Overview of the business rules:

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Runtime policy | Single MySQL 8.0 server, fixed container name `oms-mysql` | docker-compose.yml:3-4 |
| Runtime policy | Restart automatically unless explicitly stopped | docker-compose.yml:5 |
| Configuration | Credentials and DB name come from env vars with fallback defaults | docker-compose.yml:7-10 |
| Network | Publish MySQL on host port 3306 | docker-compose.yml:11-12 |
| Persistence | Data stored in named local volume `oms_mysql_data` | docker-compose.yml:13-14, 29-31 |
| Health | Container healthy only when `mysqladmin ping` as root succeeds (5s interval, 5s timeout, 20 retries, 30s start period) | docker-compose.yml:15-20 |
| Authentication | Force `mysql_native_password` default auth plugin | docker-compose.yml:22 |
| Encoding | Server charset utf8mb4, collation utf8mb4_unicode_ci | docker-compose.yml:23-24 |
| Data integrity | Unique email for users and customers; unique SKU; unique orderNumber | prisma/migrations/20260519182739_init/migration.sql:8,24,39,63 |
| Data integrity | Foreign keys RESTRICT on delete for customer, creator, product and changer references | migration.sql:98-110 |
| Data integrity | Items and status history cascade-delete with their order | migration.sql:104,110 |
| Data integrity | Enumerated roles and order statuses enforced at DB level | migration.sql:6,56,79-80 |
| Data integrity | Monetary values stored as integer cents | migration.sql:36,54-56,76-78 |
| Concurrency | Order numbers allocated from single-row sequence table `order_number_sequence` (id fixed to 1) | migration.sql:92-97; src/modules/orders/order.service.ts:245-253 |
| Environment | Shadow database is separate (`oms_shadow`), accessed as root | .env.example:5; prisma/schema.prisma:7 |
| Test isolation | Test suite wipes all tables before each test on the configured DB | tests/setup.ts:9-16 |

### Detailed breakdown of the business rules:

---

### Business Rule: Single Containerized MySQL 8.0 Instance

**Overview**:
The project runs exactly one database server as a Docker Compose service named `mysql`, using image `mysql:8.0` and container name `oms-mysql`.

**Detailed description**:
`docker-compose.yml:2-4` defines the whole infrastructure of the project as one service. The image tag `mysql:8.0` is a floating minor tag: any `docker compose pull` can move the server to a newer 8.0.x patch release. The fixed `container_name` guarantees a predictable name for `docker exec` and logs, but also prevents running two copies of this stack on the same Docker host, because container names must be unique.

Every consumer in the repository (Prisma client at `src/config/database.ts`, the seed script `prisma/seed.ts`, the Prisma CLI, and the Vitest suite) targets this one server. There is no read replica, no separate test database service and no cache or queue service. Project documentation (docs/SUMMARY.md, DEC-002, RES-004, SCL-007) records the team decision that new features, such as the planned outbox, reuse "the existing MySQL" rather than adding infrastructure. The container is therefore a shared dependency whose capacity and availability bound the whole system.

The application treats the server as always available on localhost. No retry or wait-for-ready logic exists in the application; readiness is only expressed by the container health check.

**Rule workflow**:
```
docker compose up -> pull mysql:8.0 (if absent) -> create container oms-mysql -> entrypoint init -> mysqld ready -> healthcheck passes -> app/Prisma can connect
```

---

### Business Rule: Automatic Restart Policy

**Overview**:
The container restarts automatically after failures or Docker daemon restarts, except when manually stopped (`restart: unless-stopped`, `docker-compose.yml:5`).

**Detailed description**:
With `unless-stopped`, Docker restarts the container if mysqld exits with a non-zero code, if the host reboots, or if the Docker daemon restarts. If a developer runs `docker compose stop` or `docker stop oms-mysql`, the container stays stopped across daemon restarts until started again.

This is a convenience for a local development database: the data volume keeps state, so the restarted server resumes with prior data. The policy does not interact with the health check, because plain Docker (non-Swarm) does not restart containers merely for being `unhealthy`; it only reports status.

For the application, a restart means a transient connection loss. Prisma's engine reconnects on the next query, but there is no application-level retry (ambiguity: reconnect behavior is Prisma's default, not configured in repository code).

**Rule workflow**:
```
mysqld crashes or daemon restarts -> Docker restarts oms-mysql (unless manually stopped) -> InnoDB recovery on volume -> healthcheck -> app queries succeed again
```

---

### Business Rule: Parameterized Credentials with Insecure Defaults

**Overview**:
Root password, database name, application user and password are read from environment variables, each with a default value if unset.

**Detailed description**:
`docker-compose.yml:7-10` uses `${MYSQL_ROOT_PASSWORD:-root_password}`, `${MYSQL_DATABASE:-oms}`, `${MYSQL_USER:-oms_user}` and `${MYSQL_PASSWORD:-oms_password}`. Compose resolves these from the shell or a `.env` file next to the compose file. `.env.example:9-12` supplies the same values as the defaults. `.gitignore` excludes `.env`, so real overrides are not committed.

Because defaults exist, the stack starts with no configuration at all and the credentials are then publicly known from the repository. The values must stay consistent with `DATABASE_URL` and `SHADOW_DATABASE_URL` in `.env.example:4-5`, which embed the same user, password, host and database name. The consistency is by convention only: nothing validates that `DATABASE_URL` matches the container variables. `src/config/env.ts:7` only requires `DATABASE_URL` to be a non-empty string.

The MySQL image applies these variables only when the data directory is empty (first initialization). Changing them later while `oms_mysql_data` exists has no effect on existing users or passwords; the volume must be removed to re-initialize.

**Rule workflow**:
```
.env / shell -> compose substitution (fallback to default) -> container env -> image entrypoint creates root password, database, user (first start only)
DATABASE_URL (manual, must match) -> Prisma connects as oms_user
```

---

### Business Rule: Host Port Publication on 3306

**Overview**:
MySQL's port 3306 is published to the host on all interfaces with the mapping `'3306:3306'` (`docker-compose.yml:11-12`).

**Detailed description**:
The mapping lets the application, Prisma CLI, seed script and tests run on the host and reach `localhost:3306`, as in `.env.example:4`. No bind address is specified, so Docker publishes on `0.0.0.0` (all host interfaces), exposing the database to the host's network in addition to localhost, subject to the host firewall.

The fixed host port means a local MySQL already listening on 3306 causes a start-up conflict. The mapping is not parameterized, unlike the credentials, so changing it requires editing the compose file and `DATABASE_URL` together.

No Docker network is declared; Compose creates the default project network. The application is not containerized in this repository, so the host mapping is the only access path.

**Rule workflow**:
```
host process -> localhost:3306 -> Docker port proxy -> container 3306 -> mysqld
```

---

### Business Rule: Persistent Named Volume

**Overview**:
Database files are stored in the named volume `oms_mysql_data`, mounted at `/var/lib/mysql`, using the `local` driver (`docker-compose.yml:13-14, 29-31`).

**Detailed description**:
Data survives `docker compose down` and container recreation. It is deleted only by `docker compose down -v`, `docker volume rm`, or equivalent. Project scripts `db:reset` (`package.json:15`, `prisma migrate reset --force`) reset the logical schema and data without touching the volume itself.

The volume name is project-prefixed by Compose (typically `<project>_oms_mysql_data`), so it is tied to the directory/project name. Running compose from a differently named directory creates a different volume and an apparently empty database.

Because credentials initialize only on an empty volume (see credentials rule), the volume is also the carrier of past credential state. No backup, snapshot or export mechanism is defined.

**Rule workflow**:
```
container start -> mount oms_mysql_data at /var/lib/mysql -> if empty: initialize; else: reuse existing data -> writes persist on host Docker storage
```

---

### Business Rule: Health Check Gating

**Overview**:
The container reports healthy only when `mysqladmin ping -h localhost -uroot -p<root password>` succeeds (`docker-compose.yml:15-20`).

**Detailed description**:
Parameters: interval 5s, timeout 5s, retries 20, start_period 30s. Failures during the 30s start period do not count toward the retry limit. After that, 20 consecutive failures (about 100 seconds) mark the container `unhealthy`. This tolerates the slow first-time initialization of MySQL 8.0.

The check authenticates as root and interpolates the root password into the command (`-p${MYSQL_ROOT_PASSWORD:-root_password}`), so the password is visible in `docker inspect` output and in the container's process list. Note that `mysqladmin ping` returns success even if the server is up but rejects the access in some conditions; it tests liveness of mysqld, not availability of the `oms` database, the application user, or the migrated schema (ambiguity: exact behavior depends on server version).

No other service declares `depends_on` with `condition: service_healthy` because there is no other service in the file. The health status is therefore informational for developers (`docker compose ps`) and for any external automation.

**Rule workflow**:
```
start_period 30s (failures ignored) -> every 5s run mysqladmin ping -> success: healthy; 20 consecutive failures: unhealthy
```

---

### Business Rule: Authentication Plugin Forced to mysql_native_password

**Overview**:
The server starts with `--default-authentication-plugin=mysql_native_password` (`docker-compose.yml:22`).

**Detailed description**:
MySQL 8.0 defaults to `caching_sha2_password`. Forcing `mysql_native_password` makes newly created users (root and `oms_user`, created by the image entrypoint) use the older plugin. This avoids client-compatibility problems for clients or drivers that lack support for the newer plugin or require RSA/TLS key exchange.

The flag is deprecated in MySQL 8.0 and the plugin is disabled by default in later MySQL versions (8.4 and above). The image tag `mysql:8.0` pins the major/minor, so the behavior currently works; an upgrade of the image tag would make this setting a compatibility concern. The reason for the choice is not documented in the repository (rationale unknown).

The application connects with a plain `mysql://` URL with no TLS parameters, consistent with a local-only development setup.

**Rule workflow**:
```
mysqld starts with flag -> entrypoint creates users with mysql_native_password -> clients authenticate with SHA1-based challenge
```

---

### Business Rule: utf8mb4 Character Set and Unicode Collation

**Overview**:
The server defaults to `utf8mb4` and `utf8mb4_unicode_ci` (`docker-compose.yml:23-24`), and the migration repeats it per table (`migration.sql` lines ending each CREATE TABLE).

**Detailed description**:
utf8mb4 supports the full Unicode range (including accents common in the seed data such as "Acácias", and emoji). The `_unicode_ci` collation is case-insensitive and accent-insensitive in comparisons, which affects unique indexes: for example, `users.email` and `customers.email` unique constraints treat `A@x.com` and `a@x.com` as duplicates. This is an effective business rule on top of application validation, surfacing as Prisma error P2002 and HTTP 409 CONFLICT (`src/middlewares/error.middleware.ts:37-46`).

Server-level defaults also apply to the `oms` database created by the entrypoint and to the shadow database. Because tables are explicitly created with the same charset in the migration, the schema is correct even on a server started without these flags, which gives the rule redundancy.

Index key lengths are within InnoDB limits with utf8mb4: the largest indexed column is `VARCHAR(255)` (`users.email`).

**Rule workflow**:
```
write email -> compare under utf8mb4_unicode_ci -> unique index match (case/accent-insensitive) -> P2002 -> HTTP 409
```

---

### Business Rule: Entity Uniqueness Constraints

**Overview**:
The database enforces uniqueness of `users.email`, `customers.email`, `products.sku` and `orders.orderNumber`.

**Detailed description**:
Defined as unique indexes in `migration.sql:8` (`users_email_key`), `:24` (`customers_email_key`), `:39` (`products_sku_key`) and `:63` (`orders_orderNumber_key`). They are the final safeguard against duplicate registration, duplicate catalog SKUs, and duplicate order numbers, independent of application checks, and they hold under concurrent requests where application pre-checks may race.

Violations raise Prisma `P2002`, translated by the error middleware to a 409 response with the violated target columns in the message. Tests cover the duplicated-user-email case (`tests/auth.test.ts:19`). Customer email, SKU and order number conflicts have no dedicated test.

Non-unique indexes exist on `customers.document`, `products.active`, `products.name`, `orders.customerId/status/createdAt/createdById`, `order_items.orderId/productId`, and `order_status_history.orderId/changedAt` to support lookups and filters. `customers.document` is indexed but not unique, so two customers may share a document (confidence High from DDL).

**Rule workflow**:
```
INSERT/UPDATE -> unique index check -> duplicate: P2002 -> error middleware -> 409 CONFLICT
```

---

### Business Rule: Referential Integrity and Cascade Policy

**Overview**:
Six foreign keys define which deletions are blocked (RESTRICT) and which cascade (CASCADE) (`migration.sql:99-110`).

**Detailed description**:
RESTRICT on delete: `orders.customerId -> customers.id`, `orders.createdById -> users.id`, `order_items.productId -> products.id`, `order_status_history.changedById -> users.id`. A customer, user or product referenced by an order or history record cannot be deleted at the database level. CASCADE on delete: `order_items.orderId -> orders.id` and `order_status_history.orderId -> orders.id`. Deleting an order removes its items and history. All foreign keys use ON UPDATE CASCADE.

Together, these preserve audit trails: status history and orders cannot lose their referenced actors, while history disappears only with its owning order. The application restricts deletion of orders to PENDING or CANCELLED (`tests/orders.test.ts:197`), so the cascade only applies to those orders.

The test cleanup (`tests/setup.ts:10-16`) deletes children before parents in the exact order dictated by these RESTRICT constraints, which makes the constraints an implicit contract for test infrastructure.

**Rule workflow**:
```
DELETE parent row -> RESTRICT: referenced rows exist -> DB error (Prisma P2003, not mapped specifically in error middleware, so handled as unhandled error); CASCADE: dependent rows deleted automatically
```

---

### Business Rule: Enumerated Values and Integer-Cents Money

**Overview**:
Roles and order statuses are MySQL ENUMs; money is stored as INTEGER cents.

**Detailed description**:
`users.role` is `ENUM('ADMIN','OPERATOR')` default `OPERATOR` (`migration.sql:6`). `orders.status`, `order_status_history.fromStatus` and `toStatus` are `ENUM('PENDING','PAID','PROCESSING','SHIPPED','DELIVERED','CANCELLED')`, with default `PENDING` for orders. The database therefore rejects invalid state names, but does not enforce which transitions are legal; the transition graph lives in the application (`src/modules/orders/order.status.ts`, out of this component's boundary).

Monetary columns (`priceCents`, `subtotalCents`, `discountCents`, `totalCents`, `unitPriceCents`) are 32-bit signed `INTEGER`, avoiding floating-point errors. The maximum is 2,147,483,647 cents (about 21.4 million currency units) per value. No CHECK constraints exist for non-negative prices, quantities or stock; non-negativity is only an application concern (confidence: no CHECK found in the DDL).

Adding a status or role requires a migration that alters the ENUM, which is a table-altering operation.

**Rule workflow**:
```
application writes status/role -> MySQL validates ENUM membership -> invalid value rejected; legal transition logic is applied before in the application
```

---

### Business Rule: Order Number Sequence Table

**Overview**:
Order numbers (`ORD-000001` format) are generated from a one-row table `order_number_sequence` (`id` default 1, `nextValue` default 1) (`migration.sql:92-97`).

**Detailed description**:
`src/modules/orders/order.service.ts:245-253` performs `upsert` on `id = 1` inside the order-creation transaction: it creates the row with `nextValue = 2` or increments `nextValue`, then derives the number `nextValue - 1`, zero-padded to six digits. Because the row is updated within the transaction, InnoDB takes a row lock on it; concurrent order creations serialize on this row, giving gap-free, unique numbers while serializing writes (a throughput bottleneck by design). If the transaction rolls back, the increment rolls back too.

The sequence lives in a normal table rather than an AUTO_INCREMENT column, which gives control over the format and over rollback behavior, at the cost of a hot row. The `orderNumber` column is `VARCHAR(20)` with a unique index as a second guard. Integer overflow of `nextValue` occurs after 2,147,483,647 orders; the six-digit padding does not truncate larger numbers.

The test setup deletes this table's rows before each test (`tests/setup.ts:13`), so each test starts numbering at `ORD-000001`.

**Rule workflow**:
```
BEGIN -> upsert order_number_sequence id=1 (row lock) -> read nextValue -> number = nextValue-1 -> insert order with number -> COMMIT (or ROLLBACK releases lock and number)
```

---

### Business Rule: Separate Shadow Database for Migrations

**Overview**:
Prisma Migrate uses `SHADOW_DATABASE_URL` pointing to database `oms_shadow` with the root user (`.env.example:5`, `prisma/schema.prisma:7`).

**Detailed description**:
`prisma migrate dev` creates and drops a temporary shadow database to detect drift and validate migrations. The application user `oms_user` is created by the image entrypoint with privileges only on `MYSQL_DATABASE` (standard image behavior), so it cannot create databases; hence the root user is used for the shadow URL.

Nothing in `docker-compose.yml` creates `oms_shadow`; Prisma handles it as needed (expected Prisma behavior, not verified by execution). The variable is required in the schema's datasource: `env("SHADOW_DATABASE_URL")` has no default, so any Prisma command that loads the schema without that variable set fails to resolve it (confidence Medium; depends on Prisma version behavior, which was not executed).

Using root credentials in a development `.env` file widens the privilege footprint of the tooling, but production does not need this variable except for `migrate dev`.

**Rule workflow**:
```
prisma migrate dev -> connect root -> create/reset oms_shadow -> replay migrations -> diff with schema -> apply to oms via DATABASE_URL
```

---

### Business Rule: Test Execution Against the Shared Database with Full Wipe

**Overview**:
Every test starts with all application tables emptied (`tests/setup.ts:9-16`), executed against whichever database `DATABASE_URL` references.

**Detailed description**:
`vitest.config.ts` runs tests serially in a single forked process (`fileParallelism: false`, `singleFork: true`) with 30s timeouts. This avoids cross-test interference in the shared database. The wipe uses `deleteMany` in child-to-parent order consistent with the foreign keys.

No separate test database or compose profile exists. If a developer runs `npm test` with the development `.env`, the development data in `oms` (including seed data) is destroyed. There is no guard (such as a check for a `_test` database name or `NODE_ENV=test`) in setup. `NODE_ENV` has a `test` value in the env schema (`src/config/env.ts:4`) but setup does not enforce it.

Tests require the container to be running, healthy and migrated; otherwise `prisma.$connect()` fails in `beforeAll`.

**Rule workflow**:
```
vitest start -> prisma.$connect (needs container) -> per test: deleteMany on 7 tables -> run test -> afterAll $disconnect
```

---

## 4. Component Structure

Files that constitute or directly configure the component:

```
.
├── docker-compose.yml                         # Component definition: service mysql, volume oms_mysql_data
├── .env.example                               # Template for MYSQL_* variables and DATABASE_URL / SHADOW_DATABASE_URL
├── .gitignore                                 # Excludes .env (real credentials)
├── prisma/
│   ├── schema.prisma                          # Datasource (provider mysql) and models mapped to tables
│   ├── seed.ts                                # Sample data loader (uses its own PrismaClient)
│   └── migrations/
│       ├── migration_lock.toml                # Locks provider to mysql
│       └── 20260519182739_init/migration.sql  # DDL for 7 tables, indexes, 6 foreign keys
├── src/config/
│   ├── env.ts                                 # Validates DATABASE_URL presence (non-empty string)
│   └── database.ts                            # PrismaClient factory/singleton
└── tests/setup.ts                             # Connect/wipe/disconnect lifecycle against the DB
```

docker-compose.yml internal layout:

```
services.mysql
├── image: mysql:8.0
├── container_name: oms-mysql
├── restart: unless-stopped
├── environment (4 variables with defaults)
├── ports: 3306:3306
├── volumes: oms_mysql_data:/var/lib/mysql
├── healthcheck (mysqladmin ping)
└── command (3 server flags)
volumes.oms_mysql_data (driver: local)
```

Not present: init scripts, custom my.cnf, Dockerfile, networks, secrets, resource limits, logging config, profiles, other services.

## 5. Dependency Analysis

```
Internal Dependencies (who depends on the container):
src/config/database.ts (PrismaClient) --DATABASE_URL--> MySQL Container
  <- src/server.ts, src/app.ts (injected), module services/repositories (customers, products, orders, users, auth)
prisma/seed.ts (own PrismaClient) --> MySQL Container
prisma CLI (migrate dev/reset) --DATABASE_URL, SHADOW_DATABASE_URL--> MySQL Container
tests/setup.ts --> src/config/database.ts --> MySQL Container
.env.example variables --> docker-compose.yml substitution; --> src/config/env.ts (DATABASE_URL)

External Dependencies:
- Docker Engine / Docker Compose - container runtime and orchestration
- Docker image mysql:8.0 (Docker Hub, floating tag) - database server
- Prisma 5.22.0 (@prisma/client and prisma, package.json:26,48) - client and migration tooling bound to provider "mysql"
- Host TCP port 3306 - must be free
- Local Docker volume driver - persistence
```

The container itself has no dependency on other services (no `depends_on`, no links).

## 6. Afferent and Efferent Coupling

Units are the declared elements of the component and its contract surface. Afferent: elements that depend on it. Efferent: elements it depends on. Counts derived from file inspection.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| Compose service `mysql` | 5 (PrismaClient in database.ts, seed.ts, Prisma CLI, tests/setup.ts via client, developers via `docker compose`) | 3 (image mysql:8.0, volume oms_mysql_data, .env variables) | High |
| Volume `oms_mysql_data` | 1 (service mysql) | 0 | High |
| `.env.example` MYSQL_* variables | 1 (service mysql) | 0 | Medium |
| `DATABASE_URL` / `SHADOW_DATABASE_URL` (contract with container) | 4 (schema.prisma, env.ts, Prisma CLI, PrismaClient) | 1 (implicit match with MYSQL_* values) | High |
| `prisma/schema.prisma` + migration.sql (schema hosted in container) | 6 (user, customer, product, order, auth modules, seed) | 1 (MySQL server) | High |
| `order_number_sequence` table | 1 (order.service.ts) | 0 | High |

Coupling note: the coupling between `MYSQL_*` variables and `DATABASE_URL` is implicit and manually synchronized.

## 7. Endpoints

Not applicable: the component does not expose HTTP, GraphQL or gRPC endpoints. Its only network surface is the MySQL wire protocol on TCP 3306 (see Integration Points).

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Prisma Client (src/config/database.ts) | Internal consumer | Application CRUD and transactions | MySQL wire protocol over TCP 3306, user oms_user | Prisma query engine to SQL | Prisma known errors P2002/P2025 mapped to 409/404 in error.middleware.ts:37-54; others logged and returned as 500; no retry/backoff in code |
| Prisma Migrate CLI | Internal tooling | Apply init migration, drift detection | MySQL wire protocol; main DB and shadow DB (root) | SQL DDL | CLI failure output; none in code |
| prisma/seed.ts | Internal tooling | Load sample customers/products/users/orders | MySQL wire protocol | SQL via Prisma | Script-level only |
| Vitest setup (tests/setup.ts) | Test harness | Reset tables per test | MySQL wire protocol | SQL via Prisma | Test failure on connection or constraint error |
| Docker Engine health subsystem | Infrastructure | Report container health | Local exec of mysqladmin inside container | Exit code | Marks unhealthy after 20 failures; no automatic remediation |
| Docker Hub (mysql:8.0) | External service | Image source | HTTPS (registry) | OCI image | Docker pull errors; no pinned digest |
| Docker volume driver (local) | Infrastructure | Persistence | Filesystem | InnoDB files | None configured |

Planned future integration (documented in docs, not implemented): an outbox table in this MySQL and a worker polling it with its own PrismaClient (docs/SUMMARY.md DEC-002, SCL-007).

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Infrastructure as Code | Compose YAML | docker-compose.yml | Reproducible local database |
| Externalized configuration (12-factor) | `${VAR:-default}` substitution and `.env` | docker-compose.yml:7-10; .env.example | Per-developer overrides without code changes |
| Health check / readiness probe | `mysqladmin ping` with start_period | docker-compose.yml:15-20 | Expose liveness state |
| Persistent volume | Named volume | docker-compose.yml:13-14, 29-31 | State survives container recreation |
| Database-per-application (single shared DB) | One schema `oms` | docker-compose.yml:8 | Single datastore |
| Schema migrations (versioned) | Prisma Migrate | prisma/migrations/ | Reproducible schema evolution |
| Shadow database | Prisma shadowDatabaseUrl | prisma/schema.prisma:7 | Migration validation |
| Sequence table (hi-lo style single counter) | order_number_sequence | migration.sql:92-97; order.service.ts:245 | Controlled order numbering |
| Transactional unit of work | prisma.$transaction | order.service.ts:58,131 | Atomic multi-table writes on InnoDB |
| Soft architectural decision | Reuse MySQL for outbox | docs/SUMMARY.md DEC-002 | Avoid new infrastructure |

Architectural note: the container is a pure infrastructure boundary; there is no init-time schema ownership, so schema responsibility is entirely in Prisma migrations.

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Credentials | Default root and user passwords hard-coded as fallbacks and committed in `.env.example`, compose and README-visible URL | Known credentials if run without override; combined with port exposure on all interfaces |
| High | Network | Port 3306 published on all host interfaces (no `127.0.0.1:` binding) | Database reachable from the host's network |
| High | Test isolation | Tests wipe all tables on the DB in `DATABASE_URL` with no safeguard or separate test DB | Accidental loss of development data |
| Medium | Health check | Root password in command line and visible in `docker inspect`; checks mysqld liveness only | Secret exposure; "healthy" does not imply schema or app user readiness |
| Medium | Image | Floating tag `mysql:8.0`, no digest | Non-reproducible server patch version |
| Medium | Auth plugin | `--default-authentication-plugin` is deprecated in 8.0 and removed or disabled in newer versions | Compatibility break when image is upgraded; rationale undocumented |
| Medium | Configuration drift | `MYSQL_*` and `DATABASE_URL` must be manually kept consistent; `env.ts` does not validate | Connection failures from mismatched values |
| Medium | Credential initialization | Variables only apply on empty volume | Silent no-op when changed with existing volume |
| Medium | Shadow DB | `SHADOW_DATABASE_URL` uses root; `oms_shadow` is not provisioned by the container; schema requires the variable | Setup friction; extra privileges in dev config |
| Medium | Concurrency | Single-row order number sequence serializes all order creation | Throughput bottleneck, lock contention under load |
| Medium | Data integrity | No CHECK constraints (non-negative price, quantity, stock) and `customers.document` not unique | Invalid data relies entirely on application validation |
| Medium | Data integrity | Foreign key RESTRICT violations (P2003) not mapped in error middleware | Would surface as 500 rather than a specific conflict response (confidence Medium; based on reading error.middleware.ts) |
| Low | Persistence | No backup/restore, no replication | Data loss on volume removal (acceptable for local dev) |
| Low | Security | No TLS, plain `mysql://` URL | Unencrypted local traffic |
| Low | Operations | Fixed `container_name` and fixed host port | Cannot run parallel stacks or coexist with a local MySQL |
| Low | Capacity | INTEGER cents capped near 21.4 million currency units per value; sequence INTEGER | Overflow on extreme values |
| Low | Resources | No memory/CPU limits, no log rotation | Unbounded resource use on shared hosts |

## 11. Test Coverage Analysis

Test files located in the project (no ignore folders): `tests/auth.test.ts` (7 tests, 84 lines), `tests/orders.test.ts` (8 tests, 219 lines), `tests/setup.ts` (lifecycle hooks), `tests/helpers/factories.ts` (data builders), `vitest.config.ts`. No tests exist for the container definition itself (compose file, health check, volume, ports, charset). Coverage percentages are not available: no coverage output exists in the repository, and tests were not executed per analysis rules.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| docker-compose.yml (service config, health check, volume, ports) | 0 | 0 | 0% (no validation) | No compose lint, smoke or health test; untested |
| Charset/collation behavior (utf8mb4_unicode_ci) | 0 | 0 | Not measured | No test for accent/case-insensitive uniqueness |
| Unique constraints | 0 | 1 (tests/auth.test.ts:19 duplicate email) | Partial | Only users.email covered; customer email, SKU, order number untested |
| Foreign key / cascade rules | 0 | 1 (tests/orders.test.ts:197 order deletion restriction, application-level) | Partial | DB-level RESTRICT/CASCADE not asserted directly |
| Order number sequence | 0 | 2 indirect (tests/orders.test.ts:12, :163 create orders) | Partial | No concurrency or rollback test of the sequence |
| Transactions / stock (use the DB) | 0 | 3 (tests/orders.test.ts:59, :109, :134) | Partial | Good status/stock assertions; no concurrent-transaction test |
| Test DB lifecycle (tests/setup.ts) | 0 | Implicit in all 15 tests | Full for wipe order | Depends on live container; no safeguard on target DB |
| Migrations / schema drift | 0 | 0 | 0% | No migration test; relies on prisma migrate manually |

All 15 tests are integration tests that require the running container with migrated schema; there are no unit tests that are isolated from the database.
