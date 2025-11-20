# Tinybots Platform — Project Overview

## TL;DR

- Tinybots is a family of Node.js/TypeScript services plus database tooling that powers robot automation, telemetry ingestion, customer dashboards, and fulfillment (TaaS) workflows.
- Runtime workloads fall into three domains: **Automation Core** (event ingestion, triggers, Sensara adaptor), **Experience/Business Apps** (Wonkers suite, GraphQL reporting, TaaS order flows), and **Shared Libraries/Tooling** (tiny-backend-tools, tiny-internal-services, schema generators).
- Services communicate over HTTPS (internal `/internal/v1|v2/...` surfaces), AWS SQS queues, MySQL (Tinybots + Dashboard schemas), and external APIs like Sensara SSE. Kong acts as the common gateway/auth layer.
- Documentation for individual repos lives under `devdocs/tinybots/<repo>/OVERVIEW.md`; this document links the moving pieces so new engineers or AI agents can orient quickly.

## Table of Contents

- [Platform Purpose & Landscape](#platform-purpose--landscape)
- [Core Runtime Services](#core-runtime-services)
  - [Automation Core](#automation-core)
  - [Experience & Business Apps](#experience--business-apps)
- [Shared Libraries, Tooling & Schemas](#shared-libraries-tooling--schemas)
- [Cross-Service Data Flows](#cross-service-data-flows)
- [Operational Notes & Testing](#operational-notes--testing)

## Platform Purpose & Landscape

Tinybots builds social robots plus the surrounding SaaS. The workspace root (`/Users/kai/work/tinybots`) contains:

- **Runtime services** (Node.js/TypeScript, mostly Express + Awilix) that expose REST or GraphQL APIs, orchestrate queues, or keep long-running Sensara connections alive.
- **Data-access and tooling repos** (`typ-e`, `wonkers-db`, `tiny-specs`) that manage schemas and code generation for MySQL + OpenAPI contracts.
- **Shared libraries** (`tiny-backend-tools`, `tiny-internal-services`, `tiny-internal-services-mocks`) that provide app scaffolding, DTOs, HTTP clients, and mocks so services stay consistent.
- **Docs & agent instructions** (`devdocs`, `AGENTS.md`) to bootstrap humans and AI agents.

Everything assumes Yarn 3 workspaces, MySQL backends, Kong for auth headers, and AWS SQS for fan-out.

## Core Runtime Services

### Automation Core

#### `megazord-events` (`devdocs/tinybots/megazord-events/OVERVIEW.md`)

- **Purpose**: Acts as the source-of-truth for `incoming_event`, `event_subscription`, and `outgoing_event` tables. Receives `/internal/v1/events/robots/:robotId/incomings`, validates schemas/providers, persists, and fans events out.
- **Key Integrations**: Consumes Tinybots HTTP posts from robot apps/adaptors, publishes to AWS SQS `statusQueue`, calls Trigger Service (`/internal/v1/triggers/triggers`), and hits Sensara adaptor registration endpoints.
- **Notes**: Built on `tiny-backend-tools` with Awilix DI, Cron caches for schemas/providers, and `AggregatedEventsAdaptorsService` so external sensors know which robots/events to track.

#### `m-o-triggers` (`devdocs/tinybots/m-o-triggers/OVERVIEW.md`)

- **Purpose**: Trigger scheduler/API extending `TinyDatabaseAppAuthenticated`. Handles trigger creation (`POST /internal/v1/triggers/triggers`), robot reads, and admin trigger-setting management with Cron scheduling plus concurrency windows.
- **Key Integrations**: Receives trigger requests from `megazord-events` via `TriggerService`, emits robot-specific jobs to `${baseRobotQueueAddress}${robotId}` (SQS), and reads/writes `event_trigger` + `event_trigger_setting` tables.
- **Notes**: `TriggerSchedulerParser` enforces day-of-week/time windows, TTL, and `maxConcurrentTriggers`; `EventTriggerSettingsCacheService` caches DB rows for fast scheduling.

#### `sensara-adaptor` (`devdocs/tinybots/sensara-adaptor/OVERVIEW.md`)

- **Purpose**: Bridges Sensara resident telemetry to Tinybots. Maintains SSE streams through `SensaraEventSource`, persists events in `sensara_event*`, manages resident↔robot mappings, and exposes internal poller/notification/report endpoints.
- **Key Integrations**: Talks to Sensara REST/SSE (`/v3/streams`, `/v3/notifications`), emits Tinybots events via `EventService.postEvent` (feeding `megazord-events`), and notifies Slack on failures.
- **Notes**: Long-running `SensaraEventsJob` handles SSE + heartbeat restarts; pollers fetch last location/activity. Admin routes use Kong + permission middleware.

#### `azi-3-status-check` (`devdocs/tinybots/azi-3-status-check/OVERVIEW.md`)

- **Purpose**: Owns “status check” workflows for robots: exposes `/v1/status-check/check` APIs so Micro Managers steps can start checks, evaluates YAML-defined templates against past/future Megazord events, and reports next script steps/results.
- **Key Integrations**: Reads/writes `status_check*` tables, loads historical/future events from `megazord-events`, subscribes to robot queues via `StatusChecksSQSProducerService`, and authenticates robots through Kong header validation middleware.
- **Notes**: Built on `TinyDatabaseAppAuthenticated` + Awilix, compiles templates under `schemas/status-templates/`, and orchestrates lifecycle via `StatusCheckService`, teardown scheduler cron, and Micro Managers adapters/tests in `ci/` + `test/`.

### Experience & Business Apps

#### `wonkers-api`

- **Purpose**: Customer dashboard REST API for viewing robot subscriptions/status and proxying limited Odoo calls, all behind Kong auth.
- **Key Surfaces**: `/v1/...` REST endpoints, documented at <https://api.tinybots.academy/docs/Wonkers.html>.
- **Notes**: Depends on Odoo credentials and bundles lint+unit+integration tests via `ci/localtest.sh`.

#### `wonkers-accounts`

- **Purpose**: Handles dashboard accounts, permissions, login flows, and groups with controllers for Account, Group, Login, Permission.
- **Key Surfaces**: Internal `/internal/v1/accounts/*` style routes exposed from `src/controller`.
- **Notes**: Uses `tiny-backend-tools` (published as `tiny-tools`), `kong-js` header validation, and `mysql2` for persistence.

#### `wonkers-robots`

- **Purpose**: Admin-facing robot hardware management; manages metadata/inventory outside the legacy `wonkers-api`.
- **Key Surfaces**: Robot CRUD/admin endpoints protected by Kong.
- **Notes**: Serves as the canonical source for robot details consumed by dashboards.

#### `wonkers-taas-orders`

- **Purpose**: Workflow engine for Tinybots-as-a-Service orders (CRUD, status updates, email notifications, fulfillment hooks).
- **Key Surfaces**: Controllers under `src/` offering internal admin endpoints for order lifecycle actions.
- **Notes**: Coordinates with `wonkers-taas-order-activation` for background activation jobs and hosts email templates under `email/`.

#### `wonkers-taas-order-activation`

- **Purpose**: Cron/batch service that executes activation jobs for TaaS orders.
- **Key Surfaces**: Background job API at `/internal/v1/order-activation/job/run` plus optional manual triggers.
- **Notes**: Reads the same order data as `wonkers-taas-orders` and focuses on long-running job orchestration.

#### `wonkers-graphql` (`devdocs/tinybots/wonkers-graphql/report_overview.md`)

- **Purpose**: Apollo GraphQL gateway aggregating Dashboard DB (`wonkers-db`) + Tinybots DB (`typ-e`) for reports (Robot Usage, retention KPIs, etc.).
- **Key Surfaces**: Nexus-based GraphQL schema/resolvers querying via Prisma across two database connections.
- **Notes**: Enforces multiple permission checks, heavy filtering, and gradually replaces legacy REST data sources.

## Shared Libraries, Tooling & Schemas

#### `tiny-backend-tools` (`devdocs/tinybots/tiny-backend-tools/OVERVIEW.md`)

- **Role**: Provides base `TinyApp*`/`TinyDatabaseApp*` classes, Express middleware (context logging, serializer, validation, Slack-backed error handling), logging, Awilix modules, cron/SQS providers, DB abstractions, permission providers, OAuth/email/password/TOTP utilities, and GraphQL helpers. Every runtime service extends these base apps to inherit Kong auth + request-context tracing.
- **Docs**: The dedicated overview now details the inventory, middleware contracts, async modules (cron, SQS, pools), config loaders, and testing strategy—read it before touching shared scaffolding because changes cascade to all repos.

#### `tiny-internal-services` (`devdocs/tinybots/tiny-internal-services/OVERVIEW.md`)

- **Role**: Shared SDK exporting DTOs (`TinybotsEvent`, Sensara payloads, Taas orders) and axios-based HTTP clients (Event Service, Sensara adaptor, Notification Service, Robot Accounts, Taas). Enforces consistent logging and error handling via `tb-ts-http-errors`.

#### `tiny-internal-services-mocks`

- **Role**: Ships mocks/stubs (RobotAccountServiceMock, EventServiceMocks, SensaraAdaptorServiceMocks, etc.) for unit tests across repos, preventing real HTTP calls.

#### `tiny-specs`

- **Role**: Generates TypeScript validators/types from OpenAPI specs (`specs/`). `yarn all` refreshes code + docs; services consume the generated artifacts for contract-accurate DTOs.

#### `typ-e`

- **Role**: Java/Maven project managing the Tinybots operational MySQL schema (robot accounts, schedules, scripts). Keeps schema evolution consistent.

#### `wonkers-db`

- **Role**: Schema management tool for the dashboard/TaaS database (`taas_order`, `dashboard_robot`, relations). Mirrors `typ-e` but for the business-facing schema.

#### `db/docker-compose.yaml`

- **Role**: Local infrastructure definition (MySQL containers, etc.) enabling end-to-end testing across services.

#### `devdocs/`

- **Role**: Markdown knowledge base per repo (e.g., `devdocs/tinybots/<repo>/OVERVIEW.md`), investigations, KPI memos, and onboarding guides.

#### `AGENTS.md`

- **Role**: Playbook describing how AI agents should work in this workspace (command policies, editing rules, style guides).

## Cross-Service Data Flows

1. **Telemetry → Events → Automation**
   - Sensara SSE/REST feeds flow into `sensara-adaptor` via `SensaraEventSource`. Events are stored in `sensara_event*` tables and published to `megazord-events` through the shared `EventService` client (`tiny-internal-services`).
   - `megazord-events` writes the canonical `incoming_event` row, notifies Slack/SQS subscribers, and, for trigger subscriptions, calls `TriggerService.sendTrigger` (hits `m-o-triggers`).
   - `m-o-triggers` evaluates scheduling windows, persists `event_trigger`, and eventually enqueues robot-specific SQS jobs that robots consume to perform actions.

2. **Subscriptions & Sensor Registration**
   - When a client registers or removes event subscriptions via `megazord-events`, `AggregatedEventsAdaptorsService` invokes `sensara-adaptor` endpoints (`/internal/v1|v2/sensara/residents/.../register`). This ensures the adaptor only tracks the residents needed for those robots.

3. **Customer Dashboards & Reporting**
   - Dashboard frontends call `wonkers-api` (REST) or `wonkers-graphql` (GraphQL). These services read/write via `wonkers-accounts`, `wonkers-robots`, and the MySQL schemas managed by `typ-e` (Tinybots) and `wonkers-db` (Dashboard/TaaS).
   - Reporting resolvers inside `wonkers-graphql` combine data across both databases (robot usage, TaaS statuses) and respect Kong-based permission checks. Outputs often power KPIs or admin tooling.

4. **TaaS Order Lifecycle**
   - Orders are created/updated in `wonkers-taas-orders`, persisted in the dashboard DB, and may trigger activation jobs executed by `wonkers-taas-order-activation`. Email templates under `wonkers-taas-orders/email` notify stakeholders as statuses change.

5. **Shared SDK Usage**
   - All services import DTOs/clients from `tiny-internal-services` ensuring common enums like `TinybotsEvent`, `SensaraNotificationType`, or TaaS structures stay synchronized. Unit tests rely on `tiny-internal-services-mocks` for deterministic interactions.

## Operational Notes & Testing

- **Language & Tooling**: Node.js 18+, TypeScript, Yarn 3 (`packageManager` fields). Java/Maven is used for schema repos (`typ-e`, `wonkers-db`).
- **CI**: Each service ships `ci/` scripts (often `localtest.sh`) combining lint + unit/integration tests. Many integration tests hit local MySQL (see `db/docker-compose.yaml`).
- **Auth & Context**: Kong headers + permission validators from `tiny-backend-tools` protect both admin and internal APIs. `Call-Ref` headers propagate through HTTP clients for traceability.
- **Queues & Jobs**: AWS SQS is the default queueing layer (status queue for events, per-robot queues for triggers). Cron jobs (from `tiny-backend-tools` providers) refresh caches (schemas/providers) and restart long-lived streams.
- **Docs**: Always check `devdocs/tinybots/<repo>/OVERVIEW.md` before diving into a service; they capture endpoints, data models, tests, and known risks (e.g., Sensara SSE drift alerts, unauthenticated internal routes).

With this map, you can jump into any repo knowing how it feeds Tinybots’ telemetry, automation, and customer-facing surfaces. Use the shared tooling repos for scaffolding and builders, and follow the documented flows when stitching services together.
