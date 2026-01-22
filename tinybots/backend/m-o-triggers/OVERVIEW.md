> **Branch:** develop
> **Last Commit:** bcb16de (Updated from n/a)
> **Last Updated:** Tue Nov 11 04:56:16 2025 +0000

# m-o-triggers Overview

## TL;DR
- Authenticated trigger management API and scheduler for TinyBots robots, built on `TinyDatabaseAppAuthenticated` with Awilix wiring.
- Persists `event_trigger` and `event_trigger_setting` rows in MySQL; caches event schemas and settings to keep scheduling fast.
- Cron-driven scheduler locks due triggers, enforces day/time/concurrency windows, and delivers payloads to robot-specific SQS queues.
- Kong headers and permission checks guard robot reads and admin setting updates; permission service calls flow through `PermissionAPIProvider`.

## Recent Changes Log
- Baseline rewrite: prior overview lacked metadata; document regenerated against HEAD `bcb16de` with current routes, scheduler rules, and config defaults.

## Repo Purpose & Bounded Context
- Exposes APIs for creating triggers from upstream events, reading robot-scoped trigger status, and configuring default/robot-specific trigger rules.
- Acts as the orchestration layer between `megazord-events` (incoming events), MySQL trigger tables, and downstream robot queues (SQS).
- Keeps trigger execution safe by validating schedule windows, concurrency, validity TTL, and day-of-week constraints before enqueueing work.

## Project Structure
- `src/cmd/app/main.ts` extends `TinyDatabaseAppAuthenticated`; loads configs via `loadConfigValue`, registers DI container entries, middleware, and routes; starts HTTP server and cron/timer infrastructure.
- `src/controllers/*` provide the HTTP layer: internal trigger creation, robot trigger reads, and admin trigger-setting upserts; `index.ts` exports for DI registration.
- `src/services/*` houses core logic: event schema cache (cron refreshed), trigger settings TTL cache, trigger lifecycle orchestration, scheduler/parser, and SQS dispatch.
- `src/repositories/*` encapsulate prepared queries and transactions for `event_schema`, `event_trigger_setting`, and `event_trigger` tables.
- `src/models/*` contains config DTOs (App, Scheduler, Cache, SQS, Polling, Permissions), request DTOs, and domain models with (de)serializers (day-of-week, HH:mm).
- `config/*.json` defines defaults and env overrides (port 8080, cron expressions, queue base URL, cache TTL, DB connection, permission service address).
- `test/**/*` includes controller integration tests, repository integration/unit tests, service logic tests, and DB fixture helpers via `tiny-testing`.

## Controllers & Public Surface
- `POST /internal/v1/triggers/triggers` (InternalEventTriggersController) — body validated against `CreateTriggerDto`; trusted internal caller creates a trigger that is immediately scheduled if eligible.
- `GET /v1/triggers/triggers/:triggerId` (EventTriggersController) — Kong header validation + robot validator + path DTO; returns trigger only when owned by the logged-in robot and not expired/failed.
- `PUT /v1/triggers/settings` (EventTriggerSettingsController) — Kong header + admin validator + permission `M_O_TRIGGERS_SETTING_WRITE_ALL` + `UpsertTriggerSettingDto` body; upserts default or robot override settings with time-of-day and concurrency rules.
- Error handling is centralized through `errorMiddleware`; request/response serialization handled via `serializerMiddleware`; context logging/callRef attached by `contextMiddleware`.

## Core Services & Logic
- **EventTriggersService**: `create` runs `TriggerSchedulerParser.pre` to choose default setting and compute `expectedExecutedAt`, writes the trigger, then asynchronously schedules it; `getByIdAndLoggedInRobot` hides foreign/failed/expired triggers; `upsertSetting` resolves event schema by name and writes settings.
- **TriggerSchedulerParser**: `pre` enforces presence of default setting and computes first allowed slot (time window + allowed days + reschedulable + max validity); `post` decides if execution should start now/next cycle based on `expectedExecutedAt`; `execute` wraps a transactional concurrency check (`countExecutedInWindowTimeTransaction`) before invoking executor, rejecting with REJECTED if over limits.
- **TriggerSchedulerService**: Maintains `TimerPool` capped by `scheduler.maxTimers`; cron `acquireTriggerCron` locks READY triggers into PICKED_UP, preloads settings into cache, and calls `prepare`; `releaseTriggerCron` recovers stuck triggers older than `releaseTriggerWindowMs`; `prepare` loads settings, validates via parser, and either executes immediately or schedules; `execute` sends SQS payloads to `${baseRobotQueueAddress}${robotId}` and unlocks to EXECUTED.
- **EventTriggerSettingsCacheService**: TTL cache (size/ttl from `cache` config) for settings; supports bulk preloading of locked trigger setting IDs; falls back to DB lookup and throws NotFound for missing IDs.
- **EventSchemasService**: In-memory cache of active `event_schema` rows; cron (from `pollingConfig.dbPolling`) refreshes updated schemas, removes deactivated ones, and indexes by name for UpsertTriggerSetting requests.

## External Dependencies & Cross-Service Contracts
- **MySQL**: primary store for event schemas, trigger settings, and triggers; prepared statements handled in repositories; DB connection limit increased by 3 in `createApp` to reserve cache connections.
- **AWS SQS**: messages pushed via `SQS.ContextSQS`; queue name rooted at `scheduler.baseRobotQueueAddress`; message includes `_appName`, robotId, and trigger payload with access link `/v1/triggers/triggers/:id`.
- **Kong / Identity**: Kong headers validated for robot/admin flows; `PermissionAPIProvider` calls external permission service at `permissionsProvider.address`.
- **Cron/Timers**: `cron` expressions from config drive acquisition/cleanup; `TimerPool` ensures max concurrent scheduled timers; `maxTriggerTTL` (default 300000ms) used to expire EXECUTED triggers on robot reads.
- **Caching & Utilities**: TTL cache (@isaacs/ttlcache), dayjs/datejs for time math, Awilix for DI, Winston for structured logging.
- **Config Overrides**: `config/custom-environment-variables.json` maps env vars (e.g., `BASE_ROBOT_QUEUE_ADDRESS`, AWS creds, `DB_RW_HOST`, `WONKERS_USER_ACCOUNT_ADDRESS`) for deployments.

## Testing & Quality Gates
- Test runner `yarn test` uses mocha + ts-node with nyc coverage; coverage gates: 95% statements/functions/lines, 70% branches; formatting enforced via eslint + dprint (`yarn lint:format`).
- **Controllers IT**: `test/controllers/*IT.ts` exercise request validation, permissions, robot scoping, and scheduling/rescheduling behavior against real DB fixtures.
- **Services**: `test/services/*Test.ts` covers schema cache refresh, settings cache behavior, scheduler poll/execute, parser concurrency/validity logic, and trigger service expiry handling.
- **Repositories**: `test/repositories/*` verify locking, recovery, setting writes/default disabling, and concurrency counting; includes both integration and unit variants.
- **Fixtures**: `test/helpers/DbSetup.ts` seeds/cleans `event_schema`, `event_trigger`, `event_trigger_setting` tables using shared `tiny-testing` helpers; adjust when adding new tables/columns.
