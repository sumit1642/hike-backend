# Backend Phase 1 PRD: Foundation and Local Development Environment

**Product:** Hike backend (NestJS, Drizzle, PostgreSQL with PostGIS, Redis with BullMQ) **Phase:** 1 of 10 (see
`backend.md`) **Status:** ready to build. This document contains no code. It defines what must exist, how it must behave
and how we prove it. **Method:** test-driven development. For every requirement below, the test is written first and
seen to fail.

---

## 1. Summary

Phase 1 produces a small but complete backend skeleton: a NestJS application that boots from validated configuration,
connects to PostgreSQL (with PostGIS) and Redis, applies database migrations automatically and safely, exposes health
endpoints and an OpenAPI contract, answers every failure in one consistent error format, and is covered by a working
Jest and supertest test setup.

It adds **no product features**. Its job is to make every later phase small, safe and fast to build. If something here
is wrong, all ten phases inherit the problem, which is why it is specified in detail.

## 2. Goals and non-goals

### Goals

1. A new developer can go from a fresh clone to a running server and a green test suite with a short, documented set of
   commands.
2. Local development uses **two PostgreSQL databases**: `hike_dev` for development and `hike_test` for automated tests.
   They can never be mixed up by accident.
3. Database migrations are **automated, repeatable and safe** when two processes try to run them at once.
4. The database connection follows **least privilege**: a separate role changes the schema, and the running application
   can only read and write rows.
5. The transaction and concurrency foundation (`TransactionRunner` with bounded retry) exists and is proven by tests, so
   later phases never write their own transaction handling.
6. The application setup is shared between the real server and the tests, so e2e tests exercise the same pipeline as
   production.
7. Code style rules (readable code, small functions, no `any`, layer boundaries) are enforced by tooling from the first
   commit.
8. Health endpoints report truthfully when the database or Redis is unavailable.
9. An OpenAPI contract is generated and guarded against accidental drift.
10. Documentation explains the architecture, migration policy and testing approach in plain language.

### Non-goals (deliberately not built, YAGNI)

- No product tables (users, hikes, routes). Each feature phase owns its migrations.
- No authentication or authorization (Phase 3).
- No email, Mailpit or MinIO containers (added in Phases 4 and 8).
- No BullMQ queues or workers. Only the Redis connection and a connectivity check exist now.
- No CI pipeline, deployment, Docker image for the app, or cloud setup (after the MVP works locally).
- No metrics or tracing.

## 3. Users and stories

**Developer stories**

- As a developer, I run one command to start the local databases and Redis, and one command to start the server with
  migrations applied.
- As a developer, I run the test suite and it never touches my development data.
- As a developer, I get a clear, immediate error if my configuration is wrong, instead of a mysterious failure later.
- As a developer, I add a new endpoint in a later phase and the validation, error format, request ID and documentation
  work without extra wiring.
- As a developer, I get a lint error if I import infrastructure code from the domain layer.

**Future-pipeline stories** (supported now, built later)

- As a future CI job, I can run `db:check`, the tests and the OpenAPI export with no manual setup beyond starting
  containers.
- As a future deployment, I can run migrations as a separate step with the migrator role before the app starts.

## 4. Decisions and assumptions

These are decisions made for this phase. Each can be changed before building, and any change should be recorded in an
ADR.

| ID  | Decision                                                                                                                                                                                           | Reason                                                                                                                                   |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | One PostgreSQL container holds two databases, `hike_dev` and `hike_test`, created by an init script                                                                                                | One container is lighter on a laptop and still gives full isolation at the database level. Two containers remain an option if you prefer |
| D2  | Package manager is **npm**                                                                                                                                                                         | Fewest moving parts (KISS)                                                                                                               |
| D3  | Node.js **current LTS** at scaffold time, pinned in `.nvmrc` and `engines`. NestJS current major version. Every dependency is pinned to an exact version after checking its current stable release | Reproducible installs. Versions are verified at setup, not guessed in this document                                                      |
| D4  | Two database roles: `hike_migrator` (owns the schema, runs DDL) and `hike_app` (read and write rows only). The `postgres` superuser is used only by the init script and the migration smoke test   | Principle of Least Privilege                                                                                                             |
| D5  | Migrations run through an **explicit runner script**. npm pre-scripts invoke it for development and tests. The server never migrates itself in production                                          | Explicit over implicit. Safe for rolling deployments later                                                                               |
| D6  | Tests use Redis logical database **1**, development uses **0**                                                                                                                                     | Same isolation idea as the two PostgreSQL databases                                                                                      |
| D7  | Jest has three projects: `unit` (parallel), `integration` and `e2e` (both serial, one worker)                                                                                                      | Serial runs avoid shared-database collisions. Per-worker schemas can come later if speed demands it                                      |
| D8  | API base path is `/api` with URI versioning, so first routes live under `/api/v1`                                                                                                                  | Clear contract evolution                                                                                                                 |
| D9  | One error response shape for every failure (section 6.7)                                                                                                                                           | Predictable client handling                                                                                                              |
| D10 | Readiness endpoint exposes only up or down per dependency, never versions or connection details                                                                                                    | Principle of Least Knowledge, no information leakage                                                                                     |

## 5. Deliverables overview

At the end of Phase 1 the repository contains the following.

```
hike-backend/
  .nvmrc
  .env.example
  .env.development            (git-ignored, copied from example)
  .env.test                   (git-ignored, copied from example)
  package.json                (scripts listed in section 8)
  tsconfig files              (strict mode)
  eslint and prettier config
  jest config                 (projects: unit, integration, e2e)
  drizzle.config
  docker/
    docker-compose.yml        (postgres with postgis, redis)
    postgres/init/            (create roles, two databases, enable postgis)
  drizzle/                    (generated migrations, starting with the baseline)
  openapi/openapi.json        (exported contract, committed)
  src/
    main.ts
    create-application.ts
    app.module.ts
    config/
    shared/
      clock/
      database/
      errors/
      http/
    modules/
      health/
  test/
    e2e/
    helpers/
    fakes/
  docs/
    architecture.md
    migrations.md
    testing.md
    glossary.md
    adr/
  README.md
```

## 6. Functional requirements

Requirement IDs are used by the TDD build order in section 9 and the acceptance checklist in section 12.

### 6.1 Project scaffold and code standards

- **FR-1.1** The project is a NestJS application written in TypeScript with `strict` mode on, including strict null
  checks and no implicit `any`.
- **FR-1.2** The Node version is pinned in `.nvmrc` and `engines`. Installing on a different major version produces a
  clear warning.
- **FR-1.3** ESLint and Prettier are configured with these rules: no `any`; floating promises are errors; exported
  functions declare return types; maximum function length 40 lines; maximum cyclomatic complexity 8; maximum 4
  parameters per function; no nested ternaries; identifier names of at least 3 characters (with a short allow-list such
  as `id`).
- **FR-1.4** An import-boundary lint rule is active: files under any `domain` folder may not import from `@nestjs/*`,
  `drizzle-orm`, `pg`, `ioredis` or `bullmq`. The rule is written so that adding more folders and restrictions later is
  a configuration change.
- **FR-1.5** A pre-commit hook runs lint and formatting on changed files. A pre-push hook runs the unit tests.
- **FR-1.6** Every script that sets an environment variable uses `cross-env`, so scripts behave the same on Windows,
  macOS and Linux.
- **FR-1.7** The folder layout in section 5 exists, with only the folders needed now (YAGNI). Empty placeholder folders
  are not created.

### 6.2 Configuration

- **FR-2.1** All configuration is read in one place (`config`) and validated at startup with a Zod schema. No other file
  reads `process.env`.
- **FR-2.2** Invalid or missing required values stop the process before it listens on a port, with a message that names
  every offending variable (and never prints secret values).
- **FR-2.3** Configuration is exposed to the rest of the code as a typed, read-only object through dependency injection.
- **FR-2.4** `.env.example` documents every variable in section 7 with a safe sample value. Real `.env.*` files are
  git-ignored.
- **FR-2.5** **Database guard.** When `NODE_ENV` is `test`, the database name in `DATABASE_URL` must end with `_test`.
  When `NODE_ENV` is anything else, it must not. A violation stops startup with a clear message. The same guard applies
  to the Redis logical database (1 for test, 0 otherwise).
- **FR-2.6** In production mode, API documentation is disabled unless explicitly enabled, and verbose or pretty logging
  is refused.
- **FR-2.7** A `Clock` port (`now()` returning a date) exists with a system implementation and a controllable fake for
  tests. All later time-dependent code must use it.

### 6.3 Local infrastructure

- **FR-3.1** One Docker Compose file starts PostgreSQL with PostGIS and Redis. Containers have health checks, named
  volumes and ports bound to localhost only.
- **FR-3.2** The PostgreSQL init script (runs once on first start of an empty volume) creates: the roles `hike_migrator`
  and `hike_app`; the databases `hike_dev` and `hike_test`, both owned by `hike_migrator`; the PostGIS extension in both
  databases; and default privileges so that every table created later by `hike_migrator` is automatically readable and
  writable by `hike_app` (and sequences usable), with no schema-changing rights for `hike_app`.
- **FR-3.3** Passwords for local roles come from the environment files and are documented as local-only. They are never
  reused for any real environment.
- **FR-3.4** A documented `db:reset` workflow recreates the local containers and volumes from scratch (init scripts only
  run on an empty volume, and this is explained in the docs).
- **FR-3.5** Redis is configured for local use with persistence off (faster, disposable). The test logical database is
  flushed by the test setup.
- **FR-3.6** `docker compose up` followed by the documented commands works on a machine with only Docker and the pinned
  Node version installed.

### 6.4 Database layer

- **FR-4.1** A database module provides a Drizzle client over a node-postgres connection pool, injected through a named
  token. Application code never creates its own connection.
- **FR-4.2** The application connects with `DATABASE_URL` (the `hike_app` role). Pool size comes from
  `DATABASE_POOL_MAX`. Connection and idle timeouts are set explicitly.
- **FR-4.3** The pool is closed gracefully on shutdown.
- **FR-4.4** The Drizzle configuration lives at the project root. The schema directory exists but contains no product
  tables yet. The migrations output folder is `drizzle/`.
- **FR-4.5** Repository convention is documented now for later phases: repositories are classes behind an interface
  (port), and any method that can join a transaction takes the transaction handle as an **explicit argument**.

### 6.5 Automated migration pipeline

- **FR-5.1** A standalone **migration runner** (separate from the NestJS app) connects with `DATABASE_MIGRATION_URL`
  (the `hike_migrator` role) and applies all pending migrations in order.
- **FR-5.2** The runner takes a PostgreSQL **advisory lock** with a fixed project-specific key before migrating and
  releases it afterwards, including on failure. A second runner started at the same time waits for the first, then finds
  nothing left to apply.
- **FR-5.3** The runner logs how many migrations were applied, exits with code 0 on success and a non-zero code on any
  failure, and never leaves the lock held.
- **FR-5.4** The first migration is a hand-written **baseline** that ensures the PostGIS extension exists (safe to run
  when it already does). It proves the pipeline end to end without creating product tables.
- **FR-5.5** Scripts: `db:generate` creates a migration from schema changes. `db:migrate` runs the runner. `db:check`
  fails if the schema files and migration files disagree. These are listed in section 8.
- **FR-5.6** `predev`, `pretest`, and the integration and e2e test setups run `db:migrate` against the right database
  automatically. A migration failure stops them.
- **FR-5.7** **Migration smoke test.** An integration test creates a throwaway database (name ending in `_test`) using
  the admin connection, runs the runner against it, verifies PostGIS is installed and the migration history is recorded,
  then drops it.
- **FR-5.8** **Concurrency test.** The same test setup starts two runners simultaneously against a fresh throwaway
  database and proves both succeed and every migration is applied exactly once.
- **FR-5.9** `docs/migrations.md` states the policy: forward-only, never edit an applied migration, one concern per
  migration, descriptive names, expand-and-contract for destructive changes, review SQL before commit, and who runs
  migrations in each environment.

### 6.6 Transactions and concurrency foundation

- **FR-6.1** A `TransactionRunner` port with one method that runs a function inside a database transaction and returns
  its result. The function receives the transaction-bound database handle.
- **FR-6.2** If the function succeeds, the transaction commits. If it throws, the transaction rolls back and the
  original error is rethrown unchanged.
- **FR-6.3** The runner accepts an optional isolation level. The default is `READ COMMITTED`. `SERIALIZABLE` is allowed.
- **FR-6.4** A retry wrapper repeats the whole transaction function only for PostgreSQL error codes `40001`
  (serialization failure) and `40P01` (deadlock). Maximum attempts default to 3, with a short, increasing delay that is
  injectable so tests do not wait.
- **FR-6.5** Non-retryable errors are never retried. When retries are exhausted, the last error is thrown with the
  attempt count recorded in the log.
- **FR-6.6** Nested use is refused with a clear error (no hidden savepoint behavior). The rule is documented.
- **FR-6.7** Tests prove: commit, rollback, retry then success, retry exhaustion, no retry for ordinary errors, and
  correct totals when many concurrent callers increment one value using a row lock. The test table for this is created
  by the test setup and does not exist in migrations.

### 6.7 Application bootstrap and HTTP conventions

- **FR-7.1** A single `createApplication` function builds the configured NestJS application. Both `main.ts` and the test
  helpers call it, so tests run the production pipeline.
- **FR-7.2** Global prefix `/api`, URI versioning enabled, default version `1`.
- **FR-7.3** **Security headers** through `helmet`. **CORS** uses an allowlist from `CORS_ALLOWED_ORIGINS` (empty by
  default means no cross-origin access). Request body size is limited by `REQUEST_BODY_LIMIT` (default 1 MB).
- **FR-7.4** **Validation.** A global Zod-based validation pipe validates body, query and path parameters. Unknown
  fields are rejected or stripped consistently (decision recorded in the ADR and applied everywhere).
- **FR-7.5** **Request ID.** Each request gets an ID. A client-supplied `x-request-id` is accepted only if it matches a
  safe pattern (letters, digits, dash, underscore, at most 64 characters). Otherwise a new one is generated. The ID is
  returned in the `x-request-id` response header, is attached to every log line of that request, and appears in error
  responses.
- **FR-7.6** **Error contract.** Every failure returns JSON with these fields: `status` (HTTP code), `code` (stable
  machine-readable string, such as `VALIDATION_FAILED`, `NOT_FOUND`, `INTERNAL_ERROR`), `message` (human-readable),
  `requestId`, and optional `details` (for validation, a list of field path and message). This covers validation errors,
  unknown routes, wrong HTTP methods, malformed JSON, oversized bodies, thrown HTTP exceptions and unexpected errors.
- **FR-7.7** Unexpected errors return a generic `INTERNAL_ERROR` message with status 500. The real error and stack are
  logged with the request ID and never sent to the client.
- **FR-7.8** **Logging.** Structured JSON logs through pino. Authorization headers, cookies and any field named like a
  password or token are redacted. Pretty output only in development. Tests default to silent (`LOG_LEVEL=silent`). Each
  request is logged once with method, path, status and duration.
- **FR-7.9** **Graceful shutdown.** On `SIGTERM` and `SIGINT` the server stops accepting connections, finishes in-flight
  requests, closes the database pool and Redis connection, and exits within `SHUTDOWN_TIMEOUT_MS`.
- **FR-7.10** The application refuses to start if it cannot reach PostgreSQL or Redis within a limited number of retries
  (fail fast), with a message naming the failing dependency without printing credentials.

### 6.8 Health endpoints

- **FR-8.1** `GET /api/v1/health/live` returns 200 with a simple status as long as the process is running. It checks no
  dependencies.
- **FR-8.2** `GET /api/v1/health/ready` checks PostgreSQL (a trivial query using the app role), PostGIS availability,
  and Redis (a ping). Each check has a 2 second timeout. It returns 200 only when all pass, otherwise 503 using the
  standard error contract plus per-dependency up or down status (decision D10: no versions, hosts or error text).
- **FR-8.3** Each dependency check is a small class behind a common interface, so later phases can add checks (for
  example a queue check) without editing existing ones.
- **FR-8.4** Both endpoints are public (no authentication, which does not exist yet) and are marked in a way that Phase
  3 can keep them public explicitly.

### 6.9 OpenAPI contract

- **FR-9.1** OpenAPI documentation is generated from the code and the Zod schemas, including the shared error response
  schema and the health endpoints.
- **FR-9.2** The interactive documentation page is served at `/docs` when `ENABLE_API_DOCS` is true (default true in
  development, false in production).
- **FR-9.3** A script `openapi:export` writes the contract to `openapi/openapi.json`. The file is committed so the
  Android app can generate its client from it.
- **FR-9.4** A test builds the application, generates the contract in memory and fails if it differs from the committed
  file (contract drift guard). A second test validates the contract against the OpenAPI specification rules.

### 6.10 Testing infrastructure

- **FR-10.1** Jest runs three projects: `unit` (no I/O, parallel), `integration` (real PostgreSQL and Redis, one worker)
  and `e2e` (supertest against the full application, one worker).
- **FR-10.2** Naming convention: unit tests next to the code as `*.spec.ts`, integration tests as
  `*.integration.spec.ts`, e2e tests in `test/e2e` as `*.e2e-spec.ts`.
- **FR-10.3** Integration and e2e global setup: confirms `NODE_ENV=test`, enforces the database guard (FR-2.5), runs the
  migration runner on `hike_test`, and flushes Redis logical database 1.
- **FR-10.4** Helpers: `createTestApplication` (uses `createApplication` with test configuration), `resetTestDatabase`
  (empties every application table, keeps migration history and PostGIS system tables, and refuses to run unless the
  database name ends in `_test`), and a `FakeClock`.
- **FR-10.5** Coverage reporting is on, with an initial global floor of 80% for statements and functions, raised in
  later phases.
- **FR-10.6** The test commands in section 8 work from a clean checkout once the containers are running.
- **FR-10.7** A suite that leaves open handles (connections, timers) is a failure. The pool, Redis client and app are
  always closed.
- **FR-10.8** `docs/testing.md` explains the levels, fakes over mocks, builders, naming rules and how to write the first
  test for a new feature.

### 6.11 Documentation

- **FR-11.1** `README.md`: prerequisites, setup in numbered commands, how to run, how to test, how to reset the
  database, and troubleshooting for the five most likely problems (port in use, Docker not running, wrong Node version,
  init script not rerun, test guard error).
- **FR-11.2** `docs/architecture.md`: modular monolith, the four layers, the dependency rule and the extension recipe
  from `backend.md`.
- **FR-11.3** `docs/migrations.md` (FR-5.9) and `docs/testing.md` (FR-10.8).
- **FR-11.4** `docs/glossary.md`: starts with the terms hike, track point, route, waypoint, check-in, emergency contact,
  region pack, guest, session.
- **FR-11.5** ADRs for: modular monolith (0001), Zod for validation (0002), migration strategy and roles (0003), two
  local databases (0004), error contract and unknown-field policy (0005).

## 7. Environment variables

| Variable                     | Required               | Example                                                | Notes                                                                         |
| ---------------------------- | ---------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------- |
| `NODE_ENV`                   | yes                    | `development`, `test`, `production`                    | Drives the database guard and defaults                                        |
| `PORT`                       | yes                    | `3000`                                                 | Integer 1 to 65535                                                            |
| `LOG_LEVEL`                  | no                     | `info`                                                 | `silent`, `error`, `warn`, `info`, `debug`. Test default `silent`             |
| `DATABASE_URL`               | yes                    | `postgres://hike_app:...@localhost:5432/hike_dev`      | App role. Database name must follow the guard                                 |
| `DATABASE_MIGRATION_URL`     | yes for tooling        | `postgres://hike_migrator:...@localhost:5432/hike_dev` | Migrator role, used only by the runner and `drizzle-kit`                      |
| `DATABASE_ADMIN_URL`         | tests and tooling only | `postgres://postgres:...@localhost:5432/postgres`      | Used by the smoke test to create throwaway databases. Never set in production |
| `DATABASE_POOL_MAX`          | no                     | `10`                                                   | Default 10                                                                    |
| `REDIS_URL`                  | yes                    | `redis://localhost:6379/0`                             | Logical database 0 for development, 1 for tests                               |
| `CORS_ALLOWED_ORIGINS`       | no                     | (empty)                                                | Comma-separated list. Empty means none                                        |
| `REQUEST_BODY_LIMIT`         | no                     | `1mb`                                                  |                                                                               |
| `ENABLE_API_DOCS`            | no                     | `true`                                                 | Refused as `true` in production unless explicitly set                         |
| `SHUTDOWN_TIMEOUT_MS`        | no                     | `10000`                                                |                                                                               |
| `STARTUP_DEPENDENCY_RETRIES` | no                     | `5`                                                    | For FR-7.10                                                                   |

## 8. npm scripts

| Script                                                               | Purpose                                                                                  |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `infra:up`, `infra:down`, `infra:reset`                              | Start, stop, and fully recreate local PostgreSQL and Redis                               |
| `dev`                                                                | Start the server in watch mode (runs `predev` which migrates `hike_dev`)                 |
| `build`, `start`                                                     | Compile and run the compiled server                                                      |
| `db:generate`                                                        | Create a migration from schema changes                                                   |
| `db:migrate`                                                         | Apply pending migrations with the runner (reads the environment for the target database) |
| `db:migrate:test`                                                    | Migrate `hike_test` (used by test setup)                                                 |
| `db:check`                                                           | Fail on schema and migration disagreement                                                |
| `openapi:export`                                                     | Write `openapi/openapi.json`                                                             |
| `test`, `test:unit`, `test:integration`, `test:e2e`, `test:coverage` | Run all tests, or one project, or with coverage                                          |
| `lint`, `format`, `typecheck`                                        | Static checks                                                                            |

## 9. TDD build order

Every item is red first (test written and seen failing), then green (smallest code that passes), then refactor. Items
later in the list may rely on earlier ones.

1. **Config schema (unit):** rejects missing `DATABASE_URL`; rejects a non-numeric `PORT`; lists all invalid variables
   in one message; never prints secret values.
2. **Database guard (unit):** `test` plus a name not ending in `_test` fails; `development` plus a name ending in
   `_test` fails; Redis database mismatch fails.
3. **Clock (unit):** `FakeClock` returns the set time and can be advanced; `SystemClock` returns a current date.
4. **Error contract (unit then e2e):** each error category maps to the right `status` and `code`; unexpected errors hide
   internals; validation errors list field paths.
5. **Request ID (unit then e2e):** safe client ID is echoed; unsafe ID is replaced; ID appears in logs and error bodies.
6. **Database module (integration):** connects as `hike_app`; can read and write a test table; **cannot** create a table
   (least privilege proven).
7. **Migration runner (integration):** applies baseline on a throwaway database; PostGIS present; history recorded;
   second run applies nothing; failing migration returns a failure and releases the lock.
8. **Migration concurrency (integration):** two runners at once, exactly-once application, both succeed.
9. **TransactionRunner (integration):** commit, rollback, isolation option, nested use refused.
10. **Retry wrapper (unit and integration):** retries only `40001` and `40P01`; attempt limit; injected delay;
    concurrent row-lock increments end with the correct total.
11. **Test helpers (integration):** `resetTestDatabase` empties application tables, keeps migration history and PostGIS
    tables, refuses a non-test database.
12. **Application factory (e2e):** boots with test configuration; unknown route returns the error contract; malformed
    JSON returns the error contract; oversized body returns the error contract; unsupported method returns the error
    contract.
13. **Health (unit then e2e):** each check class reports up or down; liveness always up; readiness returns 200 when
    healthy; readiness returns 503 with per-dependency status when Redis is stopped; readiness returns 503 when the
    database is unreachable. Responses contain no versions or hostnames.
14. **Shutdown (e2e):** a close request lets an in-flight request finish and closes the pool and Redis (no open
    handles).
15. **OpenAPI (e2e):** contract includes health endpoints and the error schema; generated contract equals the committed
    file; contract is valid.
16. **Lint boundary (script test):** a sample file inside a `domain` folder importing `@nestjs/common` fails lint (kept
    as a fixture test).

## 10. Non-functional requirements

| Area                  | Requirement                                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Startup               | Server ready within 5 seconds on a developer laptop with warm containers                                                              |
| Test speed            | Unit suite under 10 seconds. Integration and e2e suites together under 60 seconds at the end of Phase 1                               |
| Reliability           | Migration runner is safe to run repeatedly and concurrently                                                                           |
| Security              | Least-privilege roles, no secrets in logs or error bodies, secure headers on, CORS closed by default, documentation off in production |
| Portability           | Same behavior on Windows, macOS and Linux (cross-env, no shell-specific scripts)                                                      |
| Maintainability       | Lint limits enforced. Every public module documented in `docs/`                                                                       |
| Accessibility of docs | A new developer completes setup using only the README                                                                                 |

## 11. Review focus: failure conditions and expected behavior

| Condition                                       | Expected behavior                                                  | Proven by                                   |
| ----------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------- |
| PostgreSQL is not running at startup            | Fail fast after limited retries, clear message, exit code non-zero | e2e startup test or documented manual check |
| Redis is down while the server runs             | Liveness still 200, readiness 503, server does not crash           | e2e (item 13)                               |
| Environment variable missing or malformed       | Startup stops, all problems listed, no secrets printed             | unit (item 1)                               |
| Tests started with the development database URL | Stops before any test runs                                         | unit and global setup (items 2, 11)         |
| Dev server started with the test database URL   | Stops at startup                                                   | unit (item 2)                               |
| Two migration runners start together            | One waits, migrations applied once, both exit 0                    | integration (item 8)                        |
| A migration fails halfway                       | Non-zero exit, lock released, a retry after fixing works           | integration (item 7)                        |
| Application role tries to alter the schema      | Permission denied                                                  | integration (item 6)                        |
| Request with huge body                          | 413-style response in the standard error shape                     | e2e (item 12)                               |
| Request with malformed JSON                     | 400 in the standard error shape                                    | e2e (item 12)                               |
| Client sends a hostile `x-request-id`           | Replaced with a generated ID, never reflected                      | e2e (item 5)                                |
| SIGTERM during a request                        | Request completes, connections closed, process exits               | e2e (item 14)                               |
| Init script not rerun after changing it         | Documented: it only runs on an empty volume, use `infra:reset`     | README troubleshooting                      |
| Port already in use                             | Clear error message on startup                                     | README troubleshooting                      |
| Developer commits a layer-violating import      | Lint fails                                                         | lint fixture (item 16)                      |

## 12. Acceptance criteria (Definition of Done)

- [ ] From a clean clone, the documented commands start PostgreSQL and Redis, migrate `hike_dev` and run the server.
- [ ] `hike_dev` and `hike_test` both exist, both have PostGIS, and both have the same roles and privileges.
- [ ] Migration runner exists, is idempotent, takes an advisory lock and passes the smoke and concurrency tests.
- [ ] `hike_app` can read and write rows but cannot change the schema (test proves it).
- [ ] `TransactionRunner` and retry wrapper pass all tests in FR-6.7.
- [ ] Health endpoints behave as specified, including under dependency failure.
- [ ] Every error path returns the standard error shape with a request ID.
- [ ] OpenAPI contract is exported, committed, valid and drift-guarded.
- [ ] `npm test` runs all three Jest projects green. Coverage floor met. No open handles.
- [ ] Lint, format and type checks pass with zero warnings. The import-boundary rule is proven by its fixture.
- [ ] README, architecture, migrations, testing, glossary and the five ADRs are written.
- [ ] A YAGNI log lists what was deliberately not built.
- [ ] Tests were written first for every requirement (reviewer can see failing-then-passing history in commits).

## 13. Risks and mitigations

| Risk                                                                             | Impact                                 | Mitigation                                                                            |
| -------------------------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------- |
| Drizzle and PostGIS extension handling is not fully expressed in the schema tool | Extension missing in some environments | Hand-written baseline migration plus init script, smoke test proves it                |
| Init scripts only run on an empty volume, so edits seem ignored                  | Confusing local failures               | `infra:reset` script and a README troubleshooting entry                               |
| Serial test projects slow down as the suite grows                                | Slow feedback                          | Keep unit tests I/O-free (parallel). Consider per-worker schemas only if needed later |
| Strict lint limits slow the first days                                           | Developer friction                     | Limits are configuration. Adjust deliberately through an ADR, never ad hoc            |
| Package versions drift between machines                                          | "Works on my machine"                  | Exact pins and a lockfile, Node version pinned                                        |
| Windows path and shell differences                                               | Broken scripts                         | `cross-env` everywhere and no shell-specific syntax                                   |

## 14. Handoff to Phase 2

Phase 2 (persistence and spatial kit) consumes from Phase 1:

- The Drizzle client provider and the migration pipeline (to add column helpers and spatial types).
- `TransactionRunner` and the retry wrapper (to build the optimistic-update helper on top).
- `resetTestDatabase`, `createTestApplication` and the test project layout.
- The error contract and error code list (to add `STALE_VERSION` and similar codes).
- The documented repository convention (explicit transaction handle).

Open items to confirm before building (change any decision in section 4 now rather than later):

1. Two databases inside one PostgreSQL container (D1), or two separate containers?
2. npm (D2), or pnpm?
3. Unknown request fields rejected or silently stripped (FR-7.4)? The plan assumes **rejected**, which catches client
   mistakes early.
