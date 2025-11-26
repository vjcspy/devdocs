> **Branch:** develop
> **Last Commit:** 12f692a
> **Last Updated:** Thu May 15 11:02:27 2025 +0000

## Micro-Manager TL;DR
- Express + TypeScript service that owns robot script lifecycle (create, version, execute, archive) with v2/v3/v5 APIs for users, robots, and internal automation.
- Converts legacy v1 scripts and schedules into v2 formats, recalculates durations, writes new records, and notifies schedulers.
- Relies on MySQL (`mysql2`), Kong auth (`kong-ts`), and downstream TinyBots services for scheduling, reporting, permissions, robot status/settings, dashboards, and speech interactions.
- OpenAPI lives in `docs/micro-manager.yaml` with example payloads; DI is via Awilix; error handling is centralized in `buildExpressApp`.
- Tests use Mocha + ts-node + NYC with Standardx lint; UT/IT suites cover services, controllers, repositories, and conversion paths.

## Repo Purpose & Bounded Context
Micro-Manager is the scripts domain service for TinyBots. It stores, validates, converts, and serves executable robot scripts; coordinates scheduled executions; enforces permissions; and surfaces script data to robots, users, and internal automation. It is the migration bridge from v1 scripts to current v2/v3/v5 models and integrates with scheduling, reporting, and robot state services.

## Project Structure
### Runtime & Boot
- `src/server.ts` starts Express on port 8080 using DI container.
- `src/buildExpressApp.ts` wires middleware (morgan, JSON body parsing, v1 + v2/v3/v5 routes) and healthcheck.
- `src/buildContainer.ts` registers controllers, services, repositories, MySQL pool, config, and healthcheck with Awilix.

### HTTP Surface
- `src/routes/routes.ts` holds v2/v3/v5 REST endpoints for users, robots, conversion, internal admin, and execution search.
- `src/v1/routes/routes.js` and supporting controllers/services maintain legacy script endpoints.

### Domain Services
- `src/services/*` implements core behaviors: script CRUD, execution logging, duration calc, scheduling coordination, conversion, permissions, reporting, status/settings lookups, speech interaction cleanup, multiple-choice logic, cron helpers.

### Data Access & Persistence
- `src/repository/*` encapsulates MySQL access (scripts, nodes, steps, next transitions, executions, archives, conversions, users).
- `src/connections/dbConnection.ts` and `src/repository/Database.ts` provide pooled MySQL connections using `config.database`.

### Validation & Schemas
- `src/schemas/**` contains Joi and class-validator DTOs for params, query, headers, and bodies (script versions, executions, conversions, reports, capability versions).
- `src/middleware/validation/*` applies request validation and Kong-based access control.

### Documentation & Tooling
- OpenAPI spec in `docs/micro-manager.yaml` plus examples and schema snippets under `docs/schemas` and `docs/examples`.
- CI and dev scripts in `ci`, `ci-dev`, Dockerfile for containerized runs; Jest config present but Mocha is active test runner.

## Controllers & Public Surface
- **User script APIs (v2/v3/v5):** create/update scripts, list with filters, fetch versions, archive, validate command similarity; routes gated by Kong user headers and `checkUserAccess`.
- **Robot-facing APIs (v2/v3/v5):** fetch scripts/versions and upsert executions; v5 adds version lookups by ID; guarded by `checkRobotAccess`.
- **Internal admin APIs:** manage default/robot scripts, fetch step types and specific steps, list supported defaults, and delete archives.
- **Conversion APIs:** convert v1 scripts for user or robot contexts; internal and user-facing variants.
- **Execution search:** `/internal/v4/scripts/executions/search` and `/internal/v4/scripts/versions/search` for querying executions and versions.
- **Healthcheck:** `/healthcheck` via `express-healthcheck` using DI-provided handler.

## Core Services & Logic
- **Script lifecycle:** `ScriptService`, `ScriptExecutionService`, `ScriptDurationService`, and `ScriptDatabaseService` manage script creation, versioning, duration calculation, and execution persistence.
- **Conversion pipeline:** `ScriptConversionService` pulls v1 scripts, converts commands via `CommandConversionService`, builds v2 structures, recalculates durations, writes new versions, maps scheduled tasks through `TaskConversionService`, and marks conversion status in `ScriptConversionRepository`.
- **Scheduling coordination:** `ScheduleScriptService` talks to schedule service to read/write planned tasks, notify robots, and verify if scripts are scheduled; `CronService` computes next occurrences and validates schedule end times.
- **Reporting:** `ReportingService` assembles execution report bodies from nodes and posts to reporting service; handles validation of report forms.
- **Permissions & robot state:** `ScriptPermissionService` checks required step permissions; `RobotStatusService` and `RobotSettingsService` fetch status and language/settings; `DashboardRobotService` fetches hardware metadata.
- **Speech interactions & cleanup:** `SpeechInteractionService` deletes stale speech interactions tied to scripts.
- **Validation & access:** `kongAuthentication` + `checkUserAccess`/`checkRobotAccess` enforce consumer identity and robot ownership per Kong headers.
- **Legacy support:** `src/v1/*` retains prior script models and repositories for backward compatibility.

## External Dependencies & Cross-Service Contracts
- **MySQL** (`tinybots` DB, user `micro-manager-rw`) via `mysql2` pool configured by `DB_RW_HOST`, `DB_PORT`, `maxPoolSize`, etc.
- **Kong** (`kong-ts`) validates `x-consumer-username` and JWT-derived identities for users (`tinybots-users`) and robots (`tinybots-robots`).
- **Schedule service (EVE)** at `EVE_ADDRESS` for planned tasks CRUD, notifications, and schedule queries.
- **Speech interaction service (WADSWORTH)** at `WADSWORTH_ADDRESS` for deleting speech interactions.
- **Robot settings service (COMMANDER_DATA)** at `COMMANDER_DATA_ADDRESS` for language and settings validation.
- **Robot status service (SIGMUND)** at `SIGMUND_ADDRESS` for heartbeat/state lookups.
- **Permission service (ROBOCOP)** at `ROBOCOP_ADDRESS` to validate step permissions before accepting scripts.
- **Dashboard robot service (WONKERS_ROBOTS)** at `WONKERS_ROBOTS_ADDRESS` for robot hardware metadata.
- **Reporting service** at `REPORTING_ADDRESS` for report validation and submission.
- **Kong upstreams for robot/user services** specified in `config.kong`.
- **Public bot ID** via `PUBLIC_BOT_ID` for default scripts (referenced in controllers/services).

## Testing & Quality Gates
- Test runner: Mocha with ts-node, coverage via NYC; lint via Standardx (`yarn test` chains lint → mocha → coverage with 90/85/80/90 thresholds).
- Suites: `test/services` for service UTs, `test/controllers` for ITs across versions, `test/IT/repositoryIT` for DB-heavy integration, `test/middleware` for validators, `test/v1` for legacy coverage.
- Fixtures under `test/fixtures` and `test/v1/fixtures`; setup utilities in `test/setup` manage DB bootstrap/teardown.
- CI scripts in `ci` and `ci-dev` run lint/tests inside containers; docker-compose files provision MySQL and dependencies for local/CI runs.
