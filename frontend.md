# Hike Frontend: MVP Plan (React Native, Android)

**Status:** planning only, no code written yet. **Companion files:** `backend.md` (API and server plan), `phase1.md`
(detailed PRD for backend Phase 1). **Source scope:** the MVP column of `core.md`, Android only. Web, watch and tablet
layouts come later.

---

## 1. Purpose and scope

The app is **offline-first**. A hiker must be able to record a hike, see their position on a map and use safety tools
with no mobile signal. The phone's local database is the working copy, and the backend is where data is backed up,
synced and used for safety alerts.

**In scope for the MVP app**

- Sign in with email/password or Google, guest mode, account recovery
- Reliable background GPS recording with live statistics and crash recovery
- Map with current position, recorded route and offline map regions
- Route planning, waypoints, GPX import and export, backtrack to start
- Hike history and summaries
- Emergency contacts, check-in timer and an SOS screen
- Profile, units, privacy controls
- Sync with the backend

**Out of scope for the MVP app**

Trail discovery, reviews and trail reports, photos and journals, group tracking, social features, achievements,
payments, AI features, watch app, tablet layouts, iOS.

---

## 2. Locked decisions and open spikes

| Area                 | Decision                                                                                                                                 | Notes                                                                                                                            |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Framework            | React Native with **Expo** (development builds, not Expo Go), TypeScript strict                                                          | Development builds allow native modules such as map and background location. Builds run locally and free with `expo run:android` |
| Navigation           | Expo Router with a bottom tab layout                                                                                                     | Tabs: Plan, Track, Journeys, Profile. Explore is added after the MVP, per `core.md`                                              |
| UI library           | React Native Paper (Material Design 3), with `react-native-reanimated`, `react-native-gesture-handler`, `react-native-safe-area-context` | Fits Android conventions. Outdoor-friendly large touch targets are a theme rule                                                  |
| Local database       | `expo-sqlite` with Drizzle ORM                                                                                                           | Same query style as the backend. Migrations run automatically at app start                                                       |
| Server state         | TanStack Query                                                                                                                           | Caching of API data, retries, invalidation                                                                                       |
| App state            | Zustand for small UI state, state machines for recording and auth                                                                        | No global store for data that belongs in SQLite or TanStack Query                                                                |
| Forms and validation | React Hook Form with Zod                                                                                                                 | Zod schemas also validate GPX import and API responses at the boundary                                                           |
| API client           | Typed client generated from the backend `openapi.json`                                                                                   | Generated, never hand-written. Contract changes show up as compile errors                                                        |
| Secure storage       | `expo-secure-store` for tokens                                                                                                           | Never AsyncStorage for secrets                                                                                                   |
| Maps                 | MapLibre (`@maplibre/maplibre-react-native`) with free OpenStreetMap-based data                                                          | Offline map packs come from the backend region catalog                                                                           |
| Location             | `expo-location` with a background task and foreground service                                                                            | **Spike in Phase 4** decides if this is reliable enough (see risks)                                                              |
| Google sign-in       | Native Google sign-in library using Android Credential Manager                                                                           | The ID token goes to the backend for verification                                                                                |
| Tests                | Jest (`jest-expo`) with React Native Testing Library, MSW for API doubles, Maestro for end-to-end flows                                  | Test-first                                                                                                                       |
| Linting              | ESLint and Prettier with the same rules as the backend (no `any`, function length, naming)                                               |                                                                                                                                  |
| Budget               | Free tools only                                                                                                                          | No paid background-location library unless the Phase 4 spike proves it necessary                                                 |

**Open spikes** (decided with evidence, not guesses)

1. Does `expo-location` background tracking survive a 5 hour hike on real mid-range Android devices? (Phase 4)
2. Does the chosen MapLibre version load PMTiles packs directly, or are MBTiles packs needed? The backend's pack format
   follows this answer. (Phase 6)
3. Which charting approach is simplest for the elevation profile? (Phase 8)

---

## 3. Architecture

### 3.1 Shape: feature modules with layered, testable code

The app is organized by **feature**, and each feature has the same layers. Screens contain no business logic.

| Layer                      | Contains                                                                                                                          | Rule                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `screens` and `components` | Pure UI built from React components                                                                                               | Display props, report user actions. No API or database calls                                 |
| `view-models` (hooks)      | One hook per screen, such as `useTrackScreenViewModel`                                                                            | Turns domain state into display values and exposes actions. This is the MVVM view-model      |
| `application`              | Use cases such as `startHike`, `finishHike`, `queueRouteForSync`                                                                  | Orchestrate repositories and services. Plain TypeScript functions or small classes, no React |
| `domain`                   | Types, pure calculations (distance, pace, elevation gain), state machines                                                         | No React, no Expo, no database imports                                                       |
| `infrastructure`           | SQLite repositories, API client wrapper, location adapter, secure storage adapter, file and share adapter, network status adapter | Implements ports defined by the domain or application layers                                 |

**Dependency rule:** UI depends on view-models, view-models on application, application on domain and ports.
Infrastructure implements ports and is connected only in the **composition root**, one file where real adapters are
wired together. Tests wire fake adapters instead.

### 3.2 Planned folder structure

```
hike-app/
  app/                         Expo Router routes (thin: each file renders a screen)
  src/
    composition-root.ts        the only place real adapters are created
    shared/
      ui/                      theme, reusable components
      database/                SQLite connection, migration runner
      network/                 API client wrapper, connectivity monitor
      logging/                 structured logger with redaction
      config/                  environment config validated with Zod
    features/
      auth/
      tracking/
      maps/
      routes/
      journeys/
      safety/
      profile/
      sync/
  test/
    helpers/                   render helpers, builders
    fakes/                     fake location, fake clock, fake API
  maestro/                     end-to-end flows
  docs/
```

Each feature folder contains `screens`, `components`, `view-models`, `application`, `domain`, `infrastructure`.

### 3.3 Offline-first rules

- The **local SQLite database is the source of truth** for hikes, routes and waypoints while using the app.
- Every write that must reach the backend is stored in the local table **and** in a local **outbox** table in one SQLite
  transaction. A sync worker later sends outbox items to the backend.
- Records get their **UUID on the device**, so the same upload can be retried safely (idempotent).
- Sync is retried with exponential backoff. Items that keep failing move to a failed list shown to the user, never
  silently dropped.
- Server rejections with `409` (stale version) follow a documented conflict rule: last write wins for simple records,
  with the user informed when it matters.
- The app shows clearly what is saved locally, what is synced and what needs a connection.

### 3.4 State machines (State pattern)

- **Recording state:** `idle` to `recording` to `paused` to `finished`, plus `recovering` after a crash. Transitions are
  pure functions, tested without a device.
- **Auth state:** `signed-out`, `guest`, `signed-in`, `refreshing`, each with defined allowed transitions.

### 3.5 Coding standards (same spirit as the backend)

- Small functions (aim for under 20 lines), early returns, no deep nesting, no nested ternaries.
- Plain syntax: `async`/`await`, `for...of`, simple `if`. No clever one-liners, no complex generic types.
- Descriptive names: `calculateDistanceInMeters`, `isRecordingPaused`. Components are nouns (`HikeSummaryCard`), hooks
  start with `use`, use cases are verbs (`finishHike`).
- No `any`. Data is treated as read-only and functions return new objects.
- Components stay small (aim for one screen concern per component, and extract when a file passes about 150 lines).
- Units are stored in metric (meters, seconds) and converted only for display.
- File naming: kebab-case with role suffixes (`.view-model.ts`, `.repository.ts`, `.use-case.ts`, `.test.ts`).

### 3.6 How a new feature is added

1. Write the use cases and glossary terms in plain language.
2. Write failing tests for the domain rules and use cases with fake adapters.
3. Add domain types and ports.
4. Add the local migration and repository with an integration test.
5. Write the view-model test, then the view-model, then the screen with a component test.
6. Register routes. Add a Maestro flow if the feature is user-critical.

---

## 4. Cross-cutting strategies

### 4.1 Testing and TDD

| Level                    | Scope                                                                                   | Tools                             |
| ------------------------ | --------------------------------------------------------------------------------------- | --------------------------------- |
| Unit                     | Domain calculations, state machines, use cases with fakes                               | Jest                              |
| Repository integration   | SQLite repositories against a real in-memory SQLite database (Node-side driver in Jest) | Jest                              |
| Component and view-model | Screens and hooks with fake adapters and MSW                                            | React Native Testing Library, MSW |
| Contract                 | Generated API types compile, and MSW handlers are typed from the OpenAPI file           | TypeScript, Jest                  |
| End-to-end               | Critical journeys on an emulator                                                        | Maestro                           |

**Rules:** test first, fakes over mocks, builders for test data, descriptive test names, a `FakeClock` and
`FakeLocationSource` for anything time or GPS related. The recording logic is validated by **replaying real recorded GPX
tracks** through the domain code.

### 4.2 Concurrency strategy

| Situation                                                  | Tool                                                                                    |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Several requests fail with an expired access token at once | **Single-flight token refresh**: one refresh runs and the others wait for its result    |
| Location updates arrive while a database write is running  | A **serial write queue** for track points, flushed in small batches                     |
| Same upload sent twice                                     | Client UUIDs and backend idempotency                                                    |
| Two devices edit the same record                           | Version number from the server, `409` handling                                          |
| Sync running while the user edits                          | Outbox items are claimed with a status field in a transaction, so no item is sent twice |
| App updated with database changes                          | Automated, ordered local migrations that run before the first screen                    |

### 4.3 Local database migrations

Local SQLite migrations are generated by `drizzle-kit`, committed, and **applied automatically at app start before any
screen renders**. If a migration fails, the app shows a clear recovery screen and does not touch the data. Migrations
are forward-only, and changes are made backward-compatible (**expand and contract**), because users update the app at
different times and may roll back.

### 4.4 Security and privacy baseline

Tokens in secure storage only, no secrets in the app bundle, permission requests made only when needed and explained
first (foreground location, then background location), private by default, no personal data or coordinates in logs, GPS
metadata handled carefully on export and sharing, clear data-deletion path.

### 4.5 Android-specific realities (designed in from the start)

Foreground service notification during recording, battery optimization and manufacturer background restrictions
(guidance screen for users), low-battery tracking mode, permission denied and revoked handling, process death and phone
restart recovery, Android storage limits for offline maps, and Play Store location permission declarations.

---

## 5. MVP phases

| #   | Phase                                    | Needs from backend                        | Main output                                                           |
| --- | ---------------------------------------- | ----------------------------------------- | --------------------------------------------------------------------- |
| 1   | Foundation and tooling                   | none                                      | Running app shell, theme, navigation, test harness                    |
| 2   | Local data layer and offline foundations | none                                      | SQLite and Drizzle, migrations, repositories, outbox                  |
| 3   | API client and authentication            | Backend phases 3, 4 (use MSW until ready) | Generated client, sign-in flows, guest mode, token handling           |
| 4   | Permissions and location foundation      | none                                      | Permission flow, location adapter, background recording spike         |
| 5   | Live hike tracking                       | none                                      | Recording state machine, live statistics, crash recovery              |
| 6   | Maps and navigation                      | Backend phase 8 for region downloads      | Map screen, breadcrumb, offline regions, backtrack                    |
| 7   | Route planning and GPX                   | Backend phase 7 for sync                  | Plan routes and waypoints, GPX import and export, follow a route      |
| 8   | Journeys and statistics                  | none                                      | History, hike detail, charts, summaries                               |
| 9   | Sync and safety tools                    | Backend phases 5 to 9                     | Outbox sync, emergency contacts, check-in, SOS screen                 |
| 10  | Profile, privacy, hardening and release  | Backend phase 10                          | Settings, accessibility, performance, end-to-end flows, release build |

Phases 1 to 5 and 8 need no backend, so frontend and backend work can proceed in parallel.

### Phase 1: Foundation and tooling

**Goal:** a runnable, tested, themed app shell. **Scope:** Expo project with TypeScript strict mode, development build
running on an emulator and one real device, lint and formatting rules, Jest with React Native Testing Library, MSW, Expo
Router tab shell with placeholder screens (Plan, Track, Journeys, Profile), React Native Paper theme (outdoor-friendly
contrast and large touch targets), validated environment config, structured logger with redaction, error boundary,
composition root, folder structure and import-boundary lint rule. **Tests first:** config validation fails fast on bad
values, tab navigation renders every placeholder screen, error boundary shows a recovery view, logger redacts tokens and
coordinates. **Exit:** fresh clone to green tests and a running app in a few documented commands.

### Phase 2: Local data layer and offline foundations

**Goal:** a reliable local database every feature can use. **Scope:** `expo-sqlite` with Drizzle, automatic migrations
at app start with a recovery screen, base tables for hikes, track points, routes, waypoints and the outbox, repository
ports and SQLite implementations, a transaction helper (unit of work), network status adapter, app settings storage,
soft-delete convention for sync tombstones. **Tests first:** migrations apply from zero and from every earlier version,
repository save and load round-trips, a failed transaction leaves no partial data, an outbox item is written together
with its record or not at all, large batch inserts stay fast. **Exit:** repository integration suite green and a
documented migration policy.

### Phase 3: API client and authentication

**Goal:** the user can have an account, or use the app as a guest. **Scope:** generated API client wrapper with request
IDs, error mapping to the standard error shape and a single-flight refresh interceptor; tokens in secure storage;
screens for sign-up, sign-in, verify email, forgot and reset password; Google sign-in; guest mode with later upgrade to
an account; auth state machine; sign-out and "sign out of all devices"; navigation guard. **Tests first:** concurrent
expired requests trigger exactly one refresh, a reused or rejected refresh token signs the user out cleanly, validation
messages are shown for each field, guest upgrade keeps local data, tokens are never written to logs. **Exit:** all auth
flows pass against MSW handlers typed from the OpenAPI file, then against the real backend when Phase 3 and 4 of the
backend are ready.

### Phase 4: Permissions and location foundation

**Goal:** prove reliable location access before building features on top of it. **Scope:** staged permission flow
(explain, request foreground, then background, handle "denied" and "revoked" states), location source port with a real
adapter and a fake, foreground service notification, GPS accuracy filtering, a minimal background recording prototype,
battery optimization guidance screen, spike report. **Tests first:** permission state machine covers every combination,
accuracy filter drops poor fixes and implausible jumps (replay of recorded tracks), fake location source drives the use
cases. **Spike exit (important):** a written report from at least one real-device test of a long background recording
(target five hours with the screen locked) and a decision: keep `expo-location`, or choose a different approach before
Phase 5.

### Phase 5: Live hike tracking

**Goal:** the core experience, a recording that never loses data. **Scope:** recording state machine (start, pause,
resume, finish, recover); points written locally in small batches through a serial write queue; live distance, elapsed
and moving time, pace, speed, elevation gain and loss, altitude and GPS accuracy; auto-pause; GPS signal loss and gap
detection; low-battery tracking mode; crash and restart recovery with a "resume or finish" prompt; the Track screen with
large, readable controls. **Tests first:** replaying a real GPX through the domain code gives the expected statistics,
every state transition is covered (including invalid ones), a simulated crash mid-recording recovers all stored points,
pause time is excluded from moving time. **Exit:** replay suite green and the Phase 4 device test repeated with the real
recording code.

### Phase 6: Maps and navigation

**Goal:** know where you are and how to get back, with and without signal. **Scope:** MapLibre map screen with current
position, heading and the recorded route (breadcrumb); basic map layer switch; backtrack to start; offline map region
browser and downloader using the backend region manifest (checksum verification, progress, resume, delete, storage
usage); clear "downloaded or online only" indicators; PMTiles or MBTiles decision from the spike. **Tests first:**
downloader resumes after interruption, a checksum mismatch discards the file, storage usage is calculated correctly,
backtrack produces the reverse of the recorded path, map screen state is driven by view-model tests (not by rendering
tiles). **Exit:** an offline region downloaded on Wi-Fi shows the map in airplane mode on a real device.

### Phase 7: Route planning and GPX

**Goal:** plan before leaving home and exchange routes with other tools. **Scope:** create a route by placing points on
the map; waypoints with notes and categories; distance and ascent estimates; elevation profile; GPX import (file picker)
and export (share sheet) validated at the boundary; follow a planned route with distance to destination and off-route
alert; planned versus actual comparison data. **Tests first:** GPX import then export then import round trip, hostile or
malformed files are rejected with a friendly message, off-route detection uses a distance threshold from configuration,
route edits create outbox entries. **Exit:** round-trip and malformed-file suites green, and a planned route can be
followed in the field.

### Phase 8: Journeys and statistics

**Goal:** turn recordings into a useful history. **Scope:** hike history list with cursor-style paging from SQLite, hike
detail screen with route map, statistics, elevation and pace charts, rename and delete, sync status per hike, empty and
loading states, charting approach decided by a short spike. **Tests first:** list ordering and paging, delete creates a
tombstone and outbox entry, statistics shown match the stored values, empty state is rendered for a new user. **Exit:**
a recorded hike appears with correct statistics and charts.

### Phase 9: Sync and safety tools

**Goal:** back up and protect the hiker. **Scope:** sync worker that drains the outbox with backoff, idempotent uploads,
conflict handling, pull of remote changes by cursor, sync status UI; emergency contacts management with consent state;
check-in timer (expected return time, extend, cancel, finish); local notification reminders; SOS screen with emergency
calling shortcut, current coordinates and battery and GPS status, reachable from every tab; clear statement of what the
app cannot do. **Tests first:** an item is never sent twice, failed items move to the failed list after the retry limit,
two-device conflict follows the documented rule, check-in reminder schedule is correct with a fake clock, SOS screen
works with no network and shows last known coordinates. **Exit:** record offline, reconnect, and see data appear on a
second device. Overdue alert verified against the real backend.

### Phase 10: Profile, privacy, hardening and release

**Goal:** a polished, safe first release candidate. **Scope:** profile and settings (units, language), privacy controls
(home-hiding zone, delete account, export data), accessibility pass (screen reader labels, contrast, text scaling, large
touch targets), performance review (startup, map, long-hike memory), crash reporting that is opt-in, Maestro end-to-end
flows for sign-in, record a hike, plan a route, check-in; release build and signing procedure; device and Android
version test matrix; store listing prerequisites (privacy policy, location permission declaration). **Tests first:**
accessibility checks for key screens, settings change display units everywhere, account deletion clears local data,
end-to-end flows pass on an emulator. **Exit:** release build installs and passes the end-to-end flows. Known issues are
written down.

---

## 6. Definition of Done (every phase)

- Tests were written first, all pass, coverage floor met.
- Lint, formatting and type checks pass with zero warnings.
- Local migrations apply from zero and from the previous version.
- Works on an emulator, and location-related work is checked on a real device.
- Accessibility basics checked for any new screen (labels, contrast, touch target size).
- Docs updated, including a short note of what was deliberately not built.

---

## 7. Principles and patterns catalog

This catalog is the code review reference. Every term from the requested list appears exactly once, in one of three
groups: **applies to the app** (with how), **server-side only** (covered in `backend.md`) or **deferred or not adopted**
(with the reason). Deferring is a YAGNI decision, and each item can be revisited when a real need appears.

### 7.1 Applies to the app

| Group                        | Terms                                                                                                                                                                                                                                                                                                                                                                                | How it applies                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Principles                   | SOLID, DRY, KISS, YAGNI, Separation of Concerns, Principle of Least Privilege, Composition over Inheritance, Dependency Inversion, High Cohesion, Low Coupling, Fail Fast, Explicit over Implicit, Convention over Configuration, Design by Contract, Immutability, Principle of Least Knowledge, Separation of Policy and Mechanism, Command-Query Separation, Dependency Injection | Same standards as the backend. React favors composition. Least privilege applies to Android permissions (ask for the minimum, at the moment of need). Dependency injection is done through the composition root and context providers. Policy (thresholds, limits) lives in configuration, mechanism (calculations) in pure functions                                                                                                                                                                                                                                                                                                                               |
| Object patterns              | Factory Method, Builder, Singleton, Adapter, Facade, Composite, Module Pattern, Strategy, Observer, Pub/Sub, Command, Chain of Responsibility, State, Specification, Memento                                                                                                                                                                                                         | Factory for choosing real or fake adapters. Builders for test data. Singleton only for the database connection and API client created once in the composition root. Adapters for location, storage, files, network. Facades are use cases. Composite is React component trees. ES modules are the module pattern. Strategy for unit formatting and map layers. Observer and Pub/Sub for location updates and connectivity changes. Outbox entries are commands. Request pipeline is a chain (auth header, refresh, error mapping). State machines for recording and auth. Specification for history filters. Memento for recording snapshots used in crash recovery |
| Data and layering            | Repository Pattern, Unit of Work, Data Mapper, DTO Pattern, Service Layer, Application Service, Domain Service, Domain Model, Transaction Script                                                                                                                                                                                                                                     | Repositories over SQLite. Transaction helper as unit of work. Mappers convert rows and API shapes into domain types. API types are the DTOs. Use cases are the application services. Domain model only where rules exist (recording, statistics). Simple CRUD stays a transaction script                                                                                                                                                                                                                                                                                                                                                                            |
| Architecture                 | MVVM, Layered Architecture, Clean Architecture, Onion Architecture, Hexagonal Architecture, Ports and Adapters, Modular Monolith, Domain-Driven Design, Event-Driven Architecture, Vertical Slice Architecture, Feature-Based Architecture, Pipeline Architecture                                                                                                                    | Screens are views, hooks are view-models. Dependency rule with ports and adapters. The app is one deployable with feature modules. Lightweight DDD with a glossary. Local events for recording updates. Location points flow through a processing pipeline (accuracy filter, jump filter, smoothing, statistics)                                                                                                                                                                                                                                                                                                                                                    |
| DDD                          | Aggregate Pattern, Aggregate Root, Bounded Context, Ubiquitous Language, Domain Events, Anti-Corruption Layer, ACL Pattern                                                                                                                                                                                                                                                           | Hike and Route are aggregate roots. Feature folders are bounded contexts. Shared glossary with the backend. Events like `HikeFinished`. API and GPX data are translated at the boundary so outside shapes never leak into the domain                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Reliability                  | Outbox Pattern, Transactional Outbox, Idempotency Pattern, Retry Pattern, Exponential Backoff, Retry with Backoff, Timeout Pattern, Throttling, Debouncing, Dead Letter Queue, Graceful Degradation, Reconciliation Pattern, Eventual Consistency, Strong Consistency, Optimistic Concurrency Control                                                                                | Client-side outbox written in the same SQLite transaction as the data. UUIDs make uploads idempotent. Backoff on sync and refresh. Timeouts on every request. Location UI updates are throttled, search inputs are debounced. Failed outbox items are parked in a failed list. Offline mode is the degraded mode. Sync reconciles local and remote state. Eventual consistency between devices, strong consistency inside the local database. Version numbers with `409` handling                                                                                                                                                                                   |
| Caching                      | Caching, Cache-Aside, TTL, Cache Invalidation                                                                                                                                                                                                                                                                                                                                        | TanStack Query cache with stale times and explicit invalidation                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Security                     | Token-Based Authentication, JWT Authentication, OAuth 2.0, OpenID Connect, Authorization Code Flow with PKCE, Refresh Token Rotation, Defense in Depth, Zero Trust, Secure by Default, Input Validation, Output Encoding                                                                                                                                                             | Tokens in secure storage and single-flight rotation. Google sign-in uses the native OAuth and OpenID Connect flow with PKCE, and the backend verifies the ID token. Validate everything crossing a boundary. React Native text rendering is safe by default, and WebViews are avoided                                                                                                                                                                                                                                                                                                                                                                               |
| Local data                   | Database Indexing, Keyset Pagination, Cursor Pagination, Database Transactions, Database Constraints, Database Migration Pattern, Expand-and-Contract Database Migration, Soft Delete, Read Model, Write Model                                                                                                                                                                       | SQLite indexes for history queries, key-based paging, constraints for integrity, automatic backward-compatible migrations, tombstones for sync, a stored hike summary as the read model                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Files                        | Content Validation                                                                                                                                                                                                                                                                                                                                                                   | GPX import checks size, structure and coordinate ranges before use                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Async work                   | Asynchronous Processing, Background Job Processing, Job Scheduling, Job Idempotency                                                                                                                                                                                                                                                                                                  | Background location task, sync worker, scheduled reminders. All safe to run twice                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Observability and operations | Structured Logging, Observability, Correlation ID, Request Context Propagation, Dependency Health Monitoring, Feature Flags                                                                                                                                                                                                                                                          | Logger with redaction. A request ID header sent with each API call. A status monitor for network, GPS, battery and permissions. Configuration-driven flags                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Testing                      | Unit Testing, Integration Testing, End-to-End Testing, Contract Testing, Test Doubles, Mocking, Dependency Isolation                                                                                                                                                                                                                                                                 | Section 4.1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Platform                     | Twelve-Factor App, Configuration Management, Secrets Management                                                                                                                                                                                                                                                                                                                      | Build-time environment config. No secrets in the bundle (the Google client ID is public, not secret)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

### 7.2 Server-side only (see `backend.md`)

Inbox Pattern, Rate Limiting, Distributed Locking, Leader Election, Health Check Pattern, Failover Pattern, Load
Balancing, Queue-Based Load Leveling, Idempotent Consumer, Competing Consumers, Message Deduplication, Pessimistic
Locking, API Gateway Pattern, Backend for Frontend, RBAC, PBAC, Policy-Based Authorization, Password Hashing, CSRF
Protection, CORS, Audit Logging, Presigned URL Pattern, Worker Pool Pattern.

### 7.3 Deferred or not adopted

| Terms                                                        | Reason                                                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| Abstract Factory, Prototype, Bridge, Flyweight, Proxy        | No matching problem. Add when one appears                                                   |
| Decorator                                                    | Hooks and component composition replace higher-order components                             |
| Template Method                                              | Conflicts with composition over inheritance                                                 |
| Mediator, Visitor, Interpreter, Iterator                     | No matching problem in this app                                                             |
| Active Record, DAO Pattern, Table Module                     | Repositories and data mappers are used instead                                              |
| MVC, MVP (the Model-View-Presenter pattern)                  | MVVM is the one UI pattern, to avoid mixing styles                                          |
| Microservices Architecture                                   | Not applicable to a mobile client                                                           |
| Serverless Architecture                                      | Not applicable to a mobile client                                                           |
| CQRS, Event Sourcing                                         | Needless complexity for local storage. A summary read model is enough                       |
| Plugin Architecture                                          | No third-party extension need                                                               |
| Strangler Pattern, Strangler Fig                             | No legacy system to replace                                                                 |
| Saga Pattern, Two-Phase Commit, Compensation Pattern         | No cross-system transactions                                                                |
| Circuit Breaker, Bulkhead Pattern, Bulkhead Isolation        | One backend and simple retries are enough                                                   |
| Read-Through Cache, Write-Through Cache, Write-Behind Cache  | TanStack Query cache-aside is enough                                                        |
| Session-Based Authentication, ABAC                           | Bearer tokens are used and authorization is server-side                                     |
| Geospatial Indexing, Spatial Query Optimization              | SQLite bounding-box columns are enough for the MVP. Real spatial queries stay on the server |
| Change Data Capture, Materialized View                       | No analytics need                                                                           |
| Staged Upload Pattern, Object Lifecycle Management           | Arrive with photos and media after the MVP                                                  |
| Distributed Tracing, Metrics Collection                      | Add with opt-in diagnostics after release                                                   |
| Blue-Green Deployment, Canary Deployment, Rolling Deployment | Play Store staged rollout covers this later                                                 |
| Infrastructure as Code                                       | Not applicable to the app repository. Backend deployment handles it                         |
