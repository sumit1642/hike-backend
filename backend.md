# Hike Backend: MVP Plan

**Status:** planning only, no code written yet. **Companion files:** `frontend.md` (React Native app plan), `phase1.md`
(detailed PRD for backend Phase 1). **Source scope:** the MVP column of `core.md` (Phase 1: MVP, plus the four pillars:
reliable offline GPS tracking, useful maps, route planning, practical safety tools).

---

## 1. Purpose and scope

The backend exists to support an **offline-first Android app**. The phone records hikes locally and the backend is where
data is safely kept, synced, shared with trusted contacts and checked for safety.

**In scope for the MVP backend**

- Accounts and authentication (email/password, Google OAuth, guest mode, sessions, password reset, email verification)
- Profile, preferences, emergency contacts, privacy settings
- Hike storage with idempotent track-point upload and computed statistics
- Planned routes, waypoints, GPX import and export
- Offline sync (delta sync with tombstones) and a map-region catalog for offline map packs
- Safety check-ins with overdue alerts to emergency contacts (email only)
- Scheduled and background jobs
- OpenAPI contract for the Android client

**Out of scope for the MVP backend** (kept for later phases of the product)

Trail discovery, reviews and trail reports, photos and journals, group tracking, public profiles and social features,
payments and subscriptions, AI features, push notifications and SMS, web application, admin dashboard, deployment and
CI/CD (decided after the MVP works locally).

---

## 2. Locked decisions

| Area                 | Decision                                                                                             | Notes                                                             |
| -------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Language and runtime | TypeScript (strict mode) on the current Node LTS                                                     | Exact version pinned in `.nvmrc` and `engines`                    |
| Framework            | NestJS                                                                                               | Modules, dependency injection, guards, pipes, interceptors        |
| Database             | PostgreSQL with PostGIS                                                                              | Spatial data in SRID 4326                                         |
| ORM and migrations   | Drizzle ORM with `drizzle-kit`, node-postgres driver                                                 | Plain SQL migration files committed to git                        |
| Queue and cron       | Redis with BullMQ                                                                                    | Delayed jobs for check-ins, repeatable jobs for cron work         |
| Validation           | Zod through `nestjs-zod` and `drizzle-zod`                                                           | One schema gives validation, types and OpenAPI                    |
| API contract         | OpenAPI through `@nestjs/swagger`                                                                    | The exported file is committed so the app can generate its client |
| Authentication       | argon2id passwords, JWT access tokens, rotating refresh tokens, Google ID-token verification         | Details in Phases 3 and 4                                         |
| Email                | `MailProvider` port. Brevo SMTP in production, Mailpit locally, a fake in tests                      | Free tiers only                                                   |
| Logging              | `pino` through `nestjs-pino`, JSON output, secrets redacted                                          |                                                                   |
| Tests                | Jest and supertest, written test-first                                                               | Three Jest projects: unit, integration, e2e                       |
| Scripts              | `cross-env` in every script that sets environment variables                                          |                                                                   |
| Local infrastructure | Docker Compose: one PostGIS container holding **two databases** (`hike_dev`, `hike_test`) plus Redis | Redis logical database 0 for dev, 1 for tests                     |
| Cost                 | Free tools and free tiers only                                                                       | AWS and Azure credits are reserved for deployment later           |

---

## 3. Architecture

### 3.1 Shape: a modular monolith organized by feature

One deployable application. Each feature is a self-contained module with clear boundaries, so a module could be
extracted later without a rewrite.

**The dependency rule** (enforced by an ESLint import rule from Phase 1):

- `domain` imports nothing from NestJS, Drizzle, Redis or HTTP.
- `application` imports `domain` only, plus ports (interfaces).
- `infrastructure` implements ports and may import anything it needs.
- `http` calls `application` only, never `infrastructure` directly.
- A module may use another module only through that module's exported application service or through a domain event. It
  never imports another module's infrastructure or database tables.

### 3.2 Layers inside every module

| Layer            | Contains                                                                              | Rule                                                               |
| ---------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `http`           | Controllers, request and response DTOs (Zod schemas)                                  | Thin: parse input, call one application service, return the result |
| `application`    | Use-case services such as `RegisterUserService`, `FinishHikeService`                  | Orchestrate steps and transactions. Own no business rules          |
| `domain`         | Entities, value objects, pure functions, domain errors, repository interfaces (ports) | Plain TypeScript, fully unit-testable without any framework        |
| `infrastructure` | Drizzle repositories, Brevo adapter, Google verifier, BullMQ processors               | Implements ports. Maps database rows to domain objects             |

Simple CRUD features may keep a very small `domain` folder. We use the **Transaction Script** style where there are no
real rules (KISS) and a richer **Domain Model** only where rules exist (hike statistics, check-in timing, route
validation).

### 3.3 Planned folder structure

```
hike-backend/
  src/
    main.ts
    create-application.ts          shared by main.ts and e2e tests
    app.module.ts
    config/                        Zod env schema, typed config service
    shared/                        generic building blocks only, no business concepts
      clock/                       Clock port, SystemClock, FakeClock (tests)
      database/                    Drizzle provider, TransactionRunner, column helpers
      errors/                      error codes, exception filter
      http/                        request id, pipes, pagination helpers
      queue/                       BullMQ setup helpers
    modules/
      health/
      auth/
      users/
      emergency-contacts/
      hikes/
      routes/
      sync/
      map-regions/
      check-ins/
      mail/
  drizzle/                         generated SQL migrations (committed)
  openapi/openapi.json             exported API contract (committed)
  test/
    e2e/                           supertest suites
    helpers/                       test application, database reset, builders
    fakes/                         in-memory fakes used by unit tests
  docs/                            architecture, migrations, testing, glossary, ADRs
  docker/                          compose file and database init scripts
```

### 3.4 Coding standards ("no advanced syntax")

- Small functions (aim for under 20 lines, hard limit 40). One reason to change each.
- Early returns and guard clauses instead of deep nesting. No nested ternaries.
- Plain `async`/`await`, `for...of` loops and simple `if` statements. Avoid clever `reduce` chains, conditional or
  mapped types, deep generics, and decorators beyond what NestJS requires.
- Descriptive names, no abbreviations: `calculateElevationGain`, not `calcEG`. Booleans start with `is`, `has` or `can`.
  Errors are named after the situation: `EmailAlreadyRegisteredError`.
- No `any`. Public functions declare their return types. Data is `readonly` by default and functions return new objects
  instead of mutating inputs.
- Comments explain why, never what.
- File naming: kebab-case with a role suffix such as `.controller.ts`, `.service.ts`, `.repository.ts`, `.dto.ts`,
  `.spec.ts`.
- API coordinates always use explicit `latitude` and `longitude` fields. Conversion to GeoJSON order (longitude first)
  happens only inside adapters.

### 3.5 How a new feature is added (the extension recipe)

1. Write the glossary terms and the use cases in plain language.
2. Write failing unit tests for the domain rules and use-case services (with fakes).
3. Add the domain types and the repository port.
4. Add the Drizzle schema, generate a migration, write the repository integration test, then the repository.
5. Add DTO schemas and the controller, then the supertest e2e test.
6. Register the module. Export the OpenAPI file. Update docs.

New features add files. They should not require editing existing modules except to register themselves.

---

## 4. Cross-cutting strategies

### 4.1 Automated migration strategy

- **Source of truth:** Drizzle schema files. `drizzle-kit generate` produces SQL migration files that are reviewed and
  committed. Hand-written SQL migrations are allowed for things the schema cannot express (extensions, triggers, spatial
  indexes).
- **Forward-only:** applied migrations are never edited. A mistake is fixed by a new migration.
- **Automated, but explicit:**
    - Local development and tests run migrations automatically through npm pre-scripts (`predev`, `pretest`) using a
      standalone migration runner.
    - The application **never migrates itself on boot in production**. Migration is a separate step that runs before the
      new version starts (wired up in the deployment phase).
- **One runner at a time:** the runner takes a PostgreSQL advisory lock, so two processes starting together cannot
  corrupt the migration table. A test proves this.
- **Least privilege:** migrations run as `hike_migrator` (can change schema). The running application connects as
  `hike_app` (can read and write rows only).
- **Expand and contract:** destructive changes are split across releases (add new, copy data, switch code, remove old)
  so a new schema never breaks the previous app version during a rolling deployment.
- **Drift protection:** `db:check` fails when schema files and generated migrations disagree. Added to CI when CI
  exists.
- **Proven in tests:** a throwaway database is created, every migration is applied from zero, and PostGIS is verified.
- **Rules live in `docs/migrations.md`:** one concern per migration, descriptive names, no data deletes without a review
  note, large-table changes need a plan.

### 4.2 Concurrency strategy

Concurrency is handled by choosing the right tool for each situation, not one global solution.

| Situation                                            | Tool                                                                                                                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Two devices edit the same route, waypoint or profile | **Optimistic concurrency**: a `version` column. Updates apply only if the version matches. Otherwise the API returns `409 Conflict` with the current version |
| Retried or duplicated uploads (bad connectivity)     | **Idempotency**: client-generated UUIDs plus unique constraints and `INSERT ... ON CONFLICT DO NOTHING`                                                      |
| Hike state changes (pause, resume, finish)           | **Conditional updates** (`WHERE status = ...`) or a row lock (`SELECT ... FOR UPDATE`) inside a transaction                                                  |
| Refresh token used twice at once                     | **Atomic single-use update** (`UPDATE ... WHERE used_at IS NULL RETURNING`). A second use is treated as reuse and the session family is revoked              |
| Serialization failures and deadlocks                 | `TransactionRunner` with a bounded retry (maximum 3 attempts) on PostgreSQL codes `40001` and `40P01`                                                        |
| Two migration runners                                | Advisory lock                                                                                                                                                |
| Duplicate background jobs                            | Deterministic BullMQ job IDs and idempotent processors                                                                                                       |
| Cron running on several instances                    | BullMQ repeatable jobs (the queue deduplicates, no leader election needed)                                                                                   |

Default isolation is `READ COMMITTED`. `SERIALIZABLE` is used only where a test shows a real anomaly, always paired with
the retry helper. Repositories that take part in a transaction receive the transaction handle **as an explicit
argument** (no hidden context).

### 4.3 Testing and TDD

**Workflow:** red, green, refactor. No production code without a failing test first. Each pull request contains the
tests that justify its code.

| Level       | Scope                                                                                                | Tools            | Speed                          |
| ----------- | ---------------------------------------------------------------------------------------------------- | ---------------- | ------------------------------ |
| Unit        | Domain rules and application services with in-memory fakes. No network, no database                  | Jest             | Milliseconds, runs in parallel |
| Integration | Repositories against the real `hike_test` PostGIS database, BullMQ against real Redis (logical DB 1) | Jest             | Runs serially                  |
| End-to-end  | The whole Nest application over HTTP                                                                 | Jest + supertest | Runs serially                  |
| Contract    | The exported OpenAPI file matches the running app. Error shape is identical across endpoints         | Jest             | Fast                           |

**Rules**

- `hike_test` is never touched by development, and the dev database is never touched by tests. A fail-fast guard checks
  the database name against `NODE_ENV`.
- Tests prefer **fakes** (a `FakeClock`, `FakeMailProvider`, in-memory repositories) over heavy mocking. Mocks are used
  only at true boundaries.
- Test data comes from **builders** (`aUser().withEmail(...).build()`), never from copied literals.
- Test names describe behavior: `it('rejects a refresh token that was already used')`.
- Time-based behavior (check-in deadlines, token expiry) always uses `FakeClock`.
- Every bug fix starts with a failing regression test.
- Coverage thresholds start at 80% and rise as the project matures. Coverage is a floor, not a goal.

### 4.4 Security baseline

Secure by default (a global authentication guard, with `@Public()` as the explicit exception), least privilege (database
roles, ownership policies), defense in depth (validation, authorization, constraints), rate limiting with Redis storage,
`helmet`, strict CORS allowlist, request body size limits, secrets only in environment variables, no secrets or personal
data in logs, argon2id for passwords, hashed refresh tokens, audit records for security events.

### 4.5 API conventions

URI versioning (`/api/v1`), UUID identifiers created by the client for offline-created records, ISO 8601 UTC timestamps,
cursor (keyset) pagination, a single error contract for every failure (stable machine-readable `code`, human `message`,
optional `details`, `requestId`), `x-request-id` correlation on every response.

### 4.6 Observability (MVP level)

Structured JSON logs with request IDs, liveness and readiness endpoints, job failure logging with retained failed jobs
for inspection. Metrics and tracing are deferred.

---

## 5. MVP phases

| #   | Phase                                 | Depends on | Main output                                                                                 |
| --- | ------------------------------------- | ---------- | ------------------------------------------------------------------------------------------- |
| 1   | Foundation and local environment      | none       | Running skeleton, two databases, migration pipeline, health, OpenAPI, test harness          |
| 2   | Persistence and spatial kit           | 1          | Column conventions, geometry types, spatial helpers, optimistic-lock and pagination helpers |
| 3   | Authentication core                   | 2          | Register, login, JWT, refresh rotation, sessions                                            |
| 4   | Authentication extended               | 3          | Mail module, verification, reset, Google, guest mode, account export and deletion           |
| 5   | Profile, settings, emergency contacts | 3, 4       | Preferences, privacy settings, contacts with consent                                        |
| 6   | Hike tracking and ingestion           | 2, 3       | Hike lifecycle, idempotent point upload, statistics job                                     |
| 7   | Routes, waypoints and GPX             | 6          | Planned routes, waypoint CRUD, safe GPX import and export                                   |
| 8   | Offline sync and map regions          | 6, 7       | Delta sync, conflict rules, region catalog and manifest                                     |
| 9   | Safety check-ins and scheduled jobs   | 4, 5, 6    | Delayed overdue alerts, cron cleanup, reconciliation                                        |
| 10  | Hardening and release readiness       | all        | Security pass, load check, frozen API v1, deployment checklist                              |

Each feature owns its own tables and migrations (vertical slices), so Phase 2 delivers shared tools rather than a big
upfront schema. `phase1.md` holds the full detail for Phase 1. Phases 2 to 10 get their own `phaseN.md` just before each
is built, so decisions are made with fresh knowledge.

### Phase 1: Foundation and local environment

**Goal:** a runnable, tested, documented skeleton that every later phase builds on. **Scope:** project scaffold and lint
rules, validated configuration, Docker Compose (PostGIS with two databases and database roles, Redis), Drizzle
connection, automated migration pipeline with advisory lock, `TransactionRunner` with retry, application bootstrap
(validation, error contract, request IDs, helmet, graceful shutdown), health endpoints, OpenAPI export, Jest projects
and test helpers, documentation. **Tests first:** environment schema, database-name guard, error filter, request ID,
transaction commit/rollback/retry, migration runner concurrency, health under dependency failure, OpenAPI drift.
**Exit:** a fresh clone reaches a green `npm test` in a few commands. Details in `phase1.md`.

### Phase 2: Persistence and spatial kit

**Goal:** shared, tested database conventions so feature phases stay small and consistent. **Scope:** column helper set
(`id`, `created_at`, `updated_at`, `version`, `deleted_at`); Drizzle geometry column types for point and line string
(SRID 4326); spatial SQL helpers (distance, bounding-box filter, simplify, length); optimistic-update helper that raises
`StaleVersionError`; keyset pagination helper; soft-delete convention; test builders and seed tooling; spatial test
fixtures. **Tests first:** version mismatch is rejected, two concurrent updates produce exactly one winner, pagination
is stable under inserts, a point inside a bounding box is found and one outside is not, simplify keeps endpoints.
**Exit:** helpers documented and used by a small throwaway test table only (no product tables yet).

### Phase 3: Authentication core

**Goal:** secure email/password accounts and sessions. **Scope:** users and credentials tables; register and login;
argon2id; short-lived access JWT; refresh tokens stored hashed with rotation and reuse detection; session and device
list, single and all-device logout; global authentication guard and `@Public()`; roles scaffold (`user`, `admin`);
throttling and temporary lockout; security audit log table. **Tests first:** register rejects duplicate email regardless
of case, wrong password gives the same error as unknown email, expired access token rejected, a reused refresh token
revokes the session family, concurrent refresh with the same token yields one success. **Exit:** an authenticated test
route is protected by default and every auth path has an e2e test.

### Phase 4: Authentication extended

**Goal:** complete the account lifecycle the app needs. **Scope:** `MailProvider` port with SMTP adapter (Brevo in
production, Mailpit locally) and fake; mail queue with retries; email verification; forgot and reset password; change
password and email; Google ID-token login and safe account linking; guest accounts and upgrade to a full account without
losing data; account data export and account deletion. **Tests first:** reset token is single-use and expires, Google
token with an unverified email is refused for linking, guest data survives upgrade, deleting an account removes or
anonymizes everything it owns, mail failures never block registration (queued and retried). **Exit:** full auth flow
passes end to end with captured emails.

### Phase 5: Profile, settings and emergency contacts

**Goal:** user preferences and the trusted-contact data the safety features need. **Scope:** profile, units and
language, privacy settings (including the "hide start and end near home" zone stored for later use); emergency contact
CRUD with limits; contact consent confirmation by email; optimistic versioning on editable records. **Tests first:** a
user can never read or edit another user's records (ownership policy), contact must confirm before it can receive
alerts, stale version returns 409. **Exit:** an ownership-policy test suite that later phases reuse.

### Phase 6: Hike tracking and ingestion

**Goal:** store hikes recorded offline, safely and idempotently. **Scope:** hikes and track points tables; lifecycle
(start, pause, resume, finish) as an explicit state machine; batch point upload that is safe to retry; GPS jump and
drift detection as a chain of small filters; finish triggers a background job that computes distance, moving time,
elevation gain and loss (smoothed) and a simplified display line; hike list and detail with cursor pagination; draft
hikes that survive interruption. **Tests first:** uploading the same batch twice stores points once, an invalid state
transition is rejected, a recorded GPX replayed through the API produces known statistics within tolerance, implausible
jumps are flagged, two simultaneous finish calls result in one finished hike. **Exit:** replay test with a real recorded
track passes.

### Phase 7: Routes, waypoints and GPX

**Goal:** plan before the hike and exchange routes with other tools. **Scope:** routes and waypoints; distance and
ascent estimates from PostGIS; GPX import with strict limits (size, point count, coordinate ranges, XML entity attacks
blocked) and GPX export; data needed for planned-versus-actual comparison and "backtrack to start". **Tests first:**
import then export then import gives equivalent geometry, malformed and hostile files are rejected with the standard
error shape, ownership applies to routes and waypoints. **Exit:** round-trip and hostile-file suites pass.

### Phase 8: Offline sync and map regions

**Goal:** make offline edits safe and serve offline map packs. **Scope:** delta sync using a change cursor, tombstones
for deletions, last-write-wins per record with the rule documented and stale versions reported; map region catalog
(PostGIS polygons) and manifest endpoint (version, size, checksum, download URL); a command-line script that builds
PMTiles packs and uploads them to object storage (local MinIO now, S3-compatible storage later); MinIO added to Docker
Compose in this phase. **Tests first:** an offline edit sequence converges after reconnect, a deletion on one device
reaches another, repeated sync calls change nothing, manifest checksum matches the stored file. **Exit:** simulated
two-device scenario passes.

### Phase 9: Safety check-ins and scheduled jobs

**Goal:** the first real safety net. **Scope:** trip check-in with an expected return time; a delayed BullMQ job that
alerts confirmed emergency contacts by email with the last known location if the hiker has not checked in or finished;
extend and cancel endpoints; cron jobs for expired tokens and sessions, abandoned drafts, stale guest accounts; a
reconciliation job that finds overdue check-ins missing a job (covers lost jobs). **Tests first:** with `FakeClock`, the
alert fires once, never fires after check-in or finish, survives a worker restart, and the reconciliation job repairs a
missing job. Failed jobs are retained. **Exit:** all timing scenarios pass without real waiting.

### Phase 10: Hardening and release readiness

**Goal:** a trustworthy v1 API ready to be deployed. **Scope:** per-endpoint rate limits, payload limits, dependency
audit; OpenAPI polish and frozen v1; typed client generation check for the Android app; ingestion load test; `EXPLAIN`
review of spatial and listing queries; Dockerfile; runbook; deployment requirements checklist (the start of deployment
and CI/CD planning). **Tests first:** abuse scenarios (rapid logins, oversized uploads), contract snapshot, index usage
checks on realistic data volumes. **Exit:** green full suite, frozen contract, written deployment checklist.

---

## 6. Definition of Done (every phase)

- Tests were written first, all pass, coverage floor met.
- Lint, formatting and type checks pass with zero warnings.
- Migrations apply from zero and `db:check` is clean.
- OpenAPI file regenerated and committed.
- Docs updated (`docs/`, glossary, ADR for any significant decision).
- No new `TODO` without a linked follow-up.
- A short note on what was deliberately not built (YAGNI log).

---

## 7. Principles and patterns catalog

This catalog is the reference for code reviews. Every term from the requested list appears exactly once below, either as
**adopted for the MVP** (with how it shows up in this codebase) or **deferred or not adopted** (with the reason).
Deferring is a YAGNI decision, not a rejection, and each item can be revisited when a real need appears.

### 7.1 Design principles

| Adopted for the MVP                | How it shows up                                                                                                                                                |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SOLID                              | Single-purpose use-case services, behavior added through new classes (new strategy or adapter) rather than editing old ones, small interfaces, depend on ports |
| DRY                                | Shared helpers only after the second real repeat (rule of three for abstractions)                                                                              |
| KISS                               | Plain syntax, flat code, Transaction Script for simple CRUD                                                                                                    |
| YAGNI                              | Features own their tables and ports. Deferred items are listed in this catalog                                                                                 |
| Separation of Concerns             | Four layers per module                                                                                                                                         |
| Principle of Least Privilege       | Separate migrator and app database roles, ownership policies, minimal token scopes                                                                             |
| Composition over Inheritance       | Services are assembled from small collaborators. No deep class hierarchies                                                                                     |
| Dependency Inversion               | Application code depends on interfaces (ports). Adapters implement them                                                                                        |
| High Cohesion                      | Everything about a feature lives in its module folder                                                                                                          |
| Low Coupling                       | Modules talk through exported services and domain events                                                                                                       |
| Fail Fast                          | Environment validation at boot, database-name guard, guard clauses                                                                                             |
| Explicit over Implicit             | Transaction handles passed explicitly, migrations run by an explicit step, no magic globals                                                                    |
| Convention over Configuration      | Fixed folder layout, file suffixes, naming and column conventions                                                                                              |
| Design by Contract                 | Zod schemas at the boundary, preconditions as guard clauses, database constraints as the last line                                                             |
| Immutability                       | `readonly` data, functions return new objects                                                                                                                  |
| Principle of Least Knowledge       | Controllers know services only. Services do not reach through objects                                                                                          |
| Separation of Policy and Mechanism | Mechanism (compute statistics, filter points) is separate from policy (limits, privacy zone, thresholds kept in config)                                        |
| Command-Query Separation           | A method either changes state or returns data, never both (except returning the created identifier)                                                            |
| Dependency Injection               | NestJS constructor injection everywhere                                                                                                                        |

### 7.2 Design patterns (object level)

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Deferred or not adopted                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Factory Method**: choose mail provider by environment. **Builder**: test data builders. **Singleton**: NestJS default provider scope only, never hand-written singletons                                                                                                                                                                                                                                                                                                                                                       | **Abstract Factory** and **Prototype**: no need for families of objects or cloning                                                                                                                |
| **Adapter**: Brevo, Google verifier, clock, object storage, GPX parser. **Facade**: application services front the domain and repositories. **Decorator**: Nest guards, interceptors and decorators, retry wrapper. **Module Pattern**: Nest modules                                                                                                                                                                                                                                                                             | **Proxy**: no need beyond what NestJS does internally. **Composite**, **Bridge**, **Flyweight**: no matching problem yet                                                                          |
| **Strategy**: swappable GPS filters, mail providers, token rules. **Observer** and **Pub/Sub**: in-process domain events (Redis Pub/Sub only if multiple instances need it). **Command**: queue jobs are commands. **Chain of Responsibility**: Nest pipeline (middleware, guards, pipes, interceptors) and the GPS filter chain. **State**: hike lifecycle and check-in lifecycle. **Specification**: reusable filter rules for hike lists and GPX validation. **Iterator**: stream large point sets without loading everything | **Template Method**: conflicts with composition over inheritance. **Mediator**, **Visitor**, **Memento**, **Interpreter**: no matching problem on the server (draft recovery lives on the client) |

### 7.3 Data and layering patterns

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                              | Deferred or not adopted                                                                                                                          |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Repository Pattern**: one port per aggregate. **Unit of Work**: `TransactionRunner`. **Data Mapper**: Drizzle rows mapped to domain objects by explicit mapper functions. **DTO Pattern**: Zod request and response schemas. **Service Layer** and **Application Service**: use-case classes. **Domain Service**: rules spanning several entities (statistics calculation). **Domain Model**: only where real rules exist. **Transaction Script**: simple CRUD | **Active Record**: mixes persistence into entities. **DAO Pattern**: redundant next to repositories. **Table Module**: not a fit for this domain |

### 7.4 Architecture styles

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Deferred or not adopted                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Modular Monolith**, **Feature-Based Architecture** and **Vertical Slice Architecture**: modules by feature. **Layered Architecture**, **Clean Architecture**, **Onion Architecture**, **Hexagonal Architecture** and **Ports and Adapters**: adopted as the dependency rule (domain in the centre, adapters outside). **MVC**: controllers, DTO views and services for the HTTP layer. **Domain-Driven Design**: lightweight, with bounded contexts as modules and a glossary. **Event-Driven Architecture**: in-process domain events plus BullMQ jobs. **Pipeline Architecture**: GPS processing and GPX import as ordered steps. **Aggregate Pattern** and **Aggregate Root**: Hike and Route are roots, and outside code changes them only through the root. **Bounded Context**, **Ubiquitous Language** and **Domain Events**: glossary file, events like `HikeFinished`. **Anti-Corruption Layer** and **ACL Pattern**: Google, Brevo and GPX data are translated at the edge | **MVP** (the Model-View-Presenter pattern, not the minimum viable product) and **MVVM**: client-side patterns, see `frontend.md`. **Microservices Architecture**: premature, boundaries stay clean so extraction is possible later. **CQRS**: only the light form (separate query and command methods, summary read model) and no separate buses or stores. **Event Sourcing**: needless complexity. **Serverless Architecture**: revisit in deployment planning. **Plugin Architecture**: no third-party extension need. **Strangler Pattern**: relevant only when replacing an existing system. **Saga Pattern**: no distributed transactions inside a monolith |

### 7.5 Messaging, reliability and resilience

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Deferred or not adopted                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Idempotency Pattern**: client UUIDs and unique constraints. **Retry Pattern**, **Exponential Backoff** and **Retry with Backoff**: BullMQ attempts and backoff, transaction retry. **Timeout Pattern**: outbound calls and health checks. **Rate Limiting** and **Throttling**: Redis-backed throttler. **Asynchronous Processing**, **Background Job Processing**, **Worker Pool Pattern** and **Queue-Based Load Leveling**: BullMQ workers with set concurrency. **Competing Consumers**: several workers on one queue. **Idempotent Consumer**, **Message Deduplication** and **Job Idempotency**: deterministic job IDs and safe processors. **Dead Letter Queue**: failed jobs are retained for inspection. **Job Scheduling**: delayed and repeatable jobs. **Reconciliation Pattern**: cron repairs missing check-in jobs. **Graceful Degradation**: mail failures queue and retry, they never fail registration. **Health Check Pattern** and **Dependency Health Monitoring**: liveness and readiness. **Eventual Consistency**: statistics and sync converge. **Strong Consistency**: inside PostgreSQL transactions for auth and state changes. **Optimistic Concurrency Control** and **Pessimistic Locking**: version column, and row locks only for state transitions | **Outbox Pattern**, **Transactional Outbox** and **Inbox Pattern**: the reconciliation job covers the one place a lost event matters. Revisit if more event consumers appear. **Circuit Breaker** and **Bulkhead Pattern** and **Bulkhead Isolation**: one external dependency, retries suffice (separate queues per job type give light isolation). **Debouncing**: client-side concern. **Compensation Pattern** and **Two-Phase Commit**: no cross-system transactions. **Distributed Locking**: only the migration advisory lock. **Leader Election**: BullMQ repeatable jobs avoid it. **Failover Pattern**, **Load Balancing**: deployment concerns, application stays stateless to allow them |

### 7.6 Caching

| Adopted for the MVP                                                                                                                                                                                                                    | Deferred or not adopted                                                                   |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **TTL**: expiring Redis keys for throttling and short-lived data. **Caching**, **Cache-Aside** and **Cache Invalidation**: only for the map region manifest in Phase 8 and only after measuring, invalidated on catalog version change | **Read-Through Cache**, **Write-Through Cache**, **Write-Behind Cache**: no measured need |

### 7.7 Integration and gateway patterns

| Adopted for the MVP                     | Deferred or not adopted                                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Anti-Corruption Layer is covered in 7.4 | **API Gateway Pattern**, **Backend for Frontend**: one backend and one client. **Strangler Fig**: no legacy system |

### 7.8 Security

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Deferred or not adopted                                                                                                                                                                                                                                     |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Token-Based Authentication** and **JWT Authentication**: short-lived access tokens. **Refresh Token Rotation**: single-use refresh tokens with reuse detection. **Password Hashing**: argon2id. **OAuth 2.0**, **OpenID Connect** and **Authorization Code Flow with PKCE**: the Android Google SDK performs the flow and the backend verifies the resulting ID token. **RBAC**: `user` and `admin` roles. **Policy-Based Authorization** and **PBAC**: ownership policies per resource. **Defense in Depth**, **Zero Trust** and **Secure by Default**: global guard, every request authenticated and authorized, private by default. **CORS**: strict allowlist. **Input Validation**: Zod at every boundary. **Output Encoding**: response DTOs whitelist fields, email templates escape values | **ABAC**: add when rules depend on many attributes. **Session-Based Authentication** and **CSRF Protection**: we use bearer tokens, not cookies. Revisit if a cookie-based web app appears. Server-side session records exist only to manage refresh tokens |

### 7.9 Database and data

| Adopted for the MVP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Deferred or not adopted                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| **Database Indexing**, **Geospatial Indexing** and **Spatial Query Optimization**: GIST indexes, bounding-box filter before exact distance. **Keyset Pagination** and **Cursor Pagination**: all lists. **Database Transactions**: via `TransactionRunner`. **Database Constraints**: unique, foreign key, check constraints. **Database Migration Pattern** and **Expand-and-Contract Database Migration**: section 4.1. **Soft Delete**: tombstones for sync. **Audit Logging**: security events. **Read Model** and **Write Model**: hike summary row computed after finish versus raw points | **Change Data Capture**, **Materialized View**: no analytics need yet. Revisit for dashboards |

### 7.10 Files and uploads

| Adopted for the MVP                                               | Deferred or not adopted                                                                                                                                                                            |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Content Validation**: GPX size, structure and coordinate checks | **Presigned URL Pattern** and **Staged Upload Pattern**: arrive with photos and large media. Map packs use plain public URLs. **Object Lifecycle Management**: configured when storage is deployed |

### 7.11 Observability and operations

| Adopted for the MVP                                                                                                                                                                                                          | Deferred or not adopted                                                                                                                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Structured Logging**, **Correlation ID** and **Request Context Propagation**: request ID flows into every log line. **Observability**: logs, health and job visibility. **Feature Flags**: simple configuration flags only | **Distributed Tracing**, **Metrics Collection**: add when deployed. **Blue-Green Deployment**, **Canary Deployment**, **Rolling Deployment**: decided in the deployment phase (the app and migrations are designed to allow rolling) |

### 7.12 Testing

| Adopted for the MVP                                                                                                                                                                                                        | Deferred or not adopted                                    |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Unit Testing**, **Integration Testing**, **End-to-End Testing**: section 4.3. **Contract Testing**: OpenAPI snapshot guard. **Test Doubles**, **Mocking** and **Dependency Isolation**: fakes first, mocks at boundaries | Consumer-driven contract tools: not needed with one client |

### 7.13 Platform and configuration

| Adopted for the MVP                                                                                                                                                   | Deferred or not adopted                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Twelve-Factor App**, **Configuration Management** and **Secrets Management**: environment-driven config, nothing committed, `.env.example` documents every variable | **Infrastructure as Code**: Docker Compose is the local form. Cloud IaC arrives with deployment |
