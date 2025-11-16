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

| Repo | Purpose | Key Integrations | Notes |
| --- | --- | --- | --- |
| `megazord-events` (`devdocs/tinybots/megazord-events/OVERVIEW.md`) | Source-of-truth for `incoming_event`, `event_subscription`, `outgoing_event` tables. Accepts `/internal/v1/events/robots/:robotId/incomings` posts, validates schemas/providers, persists, and fan-outs through AWS SQS, Trigger Service calls, and adaptor registrations. | - Consumes Tinybots HTTP requests from robot apps or adaptors.<br>- Publishes to `statusQueue` (SQS), Trigger Service (`/internal/v1/triggers/triggers`), and Sensara adaptor registration endpoints. | Uses `tiny-backend-tools` middleware stack, Awilix DI, Cron-based schema/provider caches, and `AggregatedEventsAdaptorsService` to inform external sensors about subscriptions. |
| `m-o-triggers` (`devdocs/tinybots/m-o-triggers/OVERVIEW.md`) | Trigger scheduler and API living on `TinyDatabaseAppAuthenticated`. Handles creation (`POST /internal/v1/triggers/triggers`), robot-facing reads, and admin trigger-setting management. Schedules triggers via Cron, concurrency windows, and per-robot SQS queues. | - Receives trigger requests from `megazord-events` (`TriggerService`).<br>- Emits robot-specific jobs to `${baseRobotQueueAddress}${robotId}` on SQS.<br>- Reads/writes `event_trigger` and `event_trigger_setting` tables. | `TriggerSchedulerParser` enforces day-of-week/time windows, TTL, and `maxConcurrentTriggers`. `EventTriggerSettingsCacheService` caches DB rows to keep scheduling cheap. |
| `sensara-adaptor` (`devdocs/tinybots/sensara-adaptor/OVERVIEW.md`) | Bridges Sensara resident telemetry to Tinybots. Registers SSE streams via `SensaraEventSource`, persists events (`sensara_event*` tables), manages resident↔robot bindings, and exposes internal endpoints for pollers, notifications, and pilot reports. | - Pulls/pushes to Sensara REST/SSE (`/v3/streams`, `/v3/notifications`).<br>- Emits Tinybots events by calling `EventService.postEvent` (the `megazord-events` ingestion endpoint).<br>- Uses Slack webhooks for alerting. | Includes long-running jobs: `SensaraEventsJob` (SSE + heartbeats) and pollers for last-known location + activity. Admin routes require Kong + permission middleware. |

### Experience & Business Apps

| Repo | Purpose | Key Surfaces | Notes |
| --- | --- | --- | --- |
| `wonkers-api` | Customer dashboard REST API (“Wonkers”). Lets users view robot subscriptions/status, proxies to Odoo when needed, and enforces Kong auth. | `/v1/...` REST endpoints documented at https://api.tinybots.academy/docs/Wonkers.html | Handles user dashboards; relies on env vars for Odoo credentials and uses `ci/localtest.sh` for combined test suites. |
| `wonkers-accounts` | Manages dashboard accounts, permissions, login flows, and group membership. Express + Awilix service with controllers for `Account`, `Group`, `Login`, `Permission`. | Internal `/internal/v1/accounts/*` style endpoints (see `src/controller`). | Uses `tiny-backend-tools` (`tiny-tools`) for DB access, `kong-js` for auth header validation, MySQL via `mysql2`. |
| `wonkers-robots` | Hardware management service for dashboard admins (robot inventory, metadata). Legacy robot endpoints remain in `wonkers-api`. | Robot CRUD/admin endpoints (authenticated via Kong). | Acts as system of record for robot hardware when interacting with dashboard UI. |
| `wonkers-taas-orders` | Workflow engine for TaaS (tinybots-as-a-service) orders: create/update orders, email flows, integrate with fulfillment. | Controllers under `src/` expose order CRUD, email notifications, and internal admin endpoints. | Coordinates with `wonkers-taas-order-activation` for long-running activation jobs. |
| `wonkers-taas-order-activation` | Batch/cron-style service that runs activation jobs for TaaS orders. | Background job entrypoints triggered via `/internal/v1/order-activation/job/run`. | Also has an HTTP surface for manual job runs; interacts with TaaS order data. |
| `wonkers-graphql` (`devdocs/tinybots/wonkers-graphql/report_overview.md`) | Apollo GraphQL gateway consolidating data from dashboard DB (`wonkers-db`) and Tinybots DB (`typ-e`). Provides reports like Robot Usage, retention KPIs, etc. | GraphQL schema built with Nexus; resolvers query Prisma models across two DB connections. | Handles heavy aggregation (time filters, multi-permission checks). Gradually replacing older REST data sources. |

## Shared Libraries, Tooling & Schemas

| Repo / Path | Role in the Platform |
| --- | --- |
| `tiny-backend-tools` | Foundational toolkit exported via `lib/index.ts`: base `TinyDatabaseApp*` classes, middleware (context, serializer, validation, error handling), logging, Awilix modules, Cron + SQS providers, DB abstractions, permission providers, and email/TOTP helpers. Every service above extends these base apps to inherit Kong auth, context logging, and config loading (`loadConfigValue`). |
| `tiny-internal-services` (`devdocs/tinybots/tiny-internal-services/OVERVIEW.md`) | Shared SDK with DTOs (`TinybotsEvent`, Sensara payloads, Taas orders, robots) and axios-based HTTP clients for internal services (Event Service, Sensara adaptor, Notification Service, Robot Accounts, Taas). Ensures consistent logging + error mapping via `tb-ts-http-errors`. |
| `tiny-internal-services-mocks` | Companion package exporting mocks/stubs (RobotAccountServiceMock, EventServiceMocks, SensaraAdaptorServiceMocks, etc.) so unit tests in other repos can avoid real HTTP calls. |
| `tiny-specs` | Codegen pipeline that turns OpenAPI/Swagger specs (`specs/`) into TypeScript validators/types. `yarn all` regenerates `dist/` plus docs; used by multiple services to share contract-accurate DTOs. |
| `typ-e` | Java tool (Maven project) that manages the Tinybots operational MySQL schema (“organizes our database schema”). Used to evolve tables for robot accounts, scripts, schedules, etc. |
| `wonkers-db` | Similar Java schema management project but targeted at the dashboard (TaaS/order) database. Keeps DDL for `taas_order`, `dashboard_robot`, relations, etc. |
| `db/docker-compose.yaml` | Local infrastructure bootstrap (MySQL containers, etc.) to run services end-to-end during development. |
| `devdocs/` | Markdown knowledge base for every repo, release runbook, and onboarding instructions (e.g., per-service `OVERVIEW.md`, investigations, KPI definitions). |
| `AGENTS.md` | Playbook explaining how AI agents should behave inside this workspace (command policies, style guides, etc.). |

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
