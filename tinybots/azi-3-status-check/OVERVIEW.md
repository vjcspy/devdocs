# azi-3-status-check Overview
>
> TL;DR: Node/TypeScript service that lets Tinybots robots kick off “status check” script steps, persists and evaluates them against historical/future Megazord events using configurable templates, and pushes updates back over SQS.

## Repo Purpose & Interactions

azi-3-status-check is the backend that owns “status checks” for robots. Robots call the HTTP API to start a check when they reach a `statusCheck` Micro Managers step. The service looks up the script definition, finds a matching YAML template, loads prior Megazord events to evaluate “past” conditions, subscribes to “future” events over Megazord, and eventually notifies the robot (through an SQS queue) which next script step to execute. The project is a Yarn 3 workspace with TypeScript, Express, Awilix DI, Tiny Backend Tools utilities, and generated template modules.

### Inventory

- `src/cmd/app/` – Bootstraps the Express app via `TinyDatabaseAppAuthenticated`, wires configs, Awilix container bindings, middleware, routes, and graceful start/stop (`main.ts`, `index.ts`).
- `src/controllers/StatusChecksController.ts` – HTTP handlers for create/get operations; fetches context and delegates to `StatusChecksService`.
- `src/services/` – Core business logic: `StatusCheckService`, `StatusCheckTeardownSchedulerService`, `StatusChecksSQSProducerService`, `StatusTemplatesLoaderService`, `StatusTemplatesCompiler`, plus Megazord/MicroManagers adapters and template parser utilities under `internal/`.
- `src/repositories/` – MySQL repositories (`StatusChecksRepository`, `StatusTemplateRepository`) and the `internal/StatusCheckActiveModel` aggregate that encapsulates DB persistence and locking semantics.
- `src/models/` – Configuration models, DTOs, and domain entities (status checks, templates, Megazord events, Micro Managers script representations, SQS message envelopes).
- `schemas/status-templates/` – Authoritative YAML templates that describe what events constitute PASS/FAIL/UNKNOWN for each status check; compiled JS lives under `schemas/generated/`.
- `config/` – `node-config` profiles, environment variable mappings, and scheduler/cache/SQS defaults used at runtime.
- `ci/` – Dockerfile, docker-compose stack (MySQL, Localstack SQS, Typ-e, external services), Concourse pipelines, entrypoints, and verification scripts.
- `test/` – Mocha/Nyc suites covering controllers, services, repositories, and helper DB fixture builder `helpers/DbSetup.ts`.
- `dprint.json`, `eslint.config.js`, `tsconfig*.json`, `Dockerfile` – Tooling for formatting, linting, builds, and containerization.

## Controllers / Public Surface

The app sets up context/logging/serialization middleware plus Kong header validation (`ValidationMiddleware.headerValidator(KongHeader, false, true, false)`) and robot authentication via `robotValidator` before hitting controllers (`src/cmd/app/main.ts`). Errors are funneled through Tiny Backend Tools’ `errorMiddleware`.

### POST /v1/status-check/check

- **Method/Path**: `POST /v1/status-check/check`.
- **Request Body**: `CreateStatusCheckBodyDto` (`stepId`, `scheduledId`, `scriptVersionId`, `plannedAt` ISO timestamp).
- **Validation & Guards**: Kong consumer headers must be present, requester must be a robot (`x-authenticated-userid`), payload validated via `class-validator` DTO, middleware attaches context/loggers.
- **Execution Flow**: `StatusChecksController.create` logs the request and calls `StatusChecksService.create`. The service reads the micro-managers step, resolves the template by name, computes polling windows, inserts a new status check row, performs the “past” evaluation immediately, registers SQS-based future subscriptions if needed, flushes DB aggregates, and returns the resulting domain entity.
- **Response Shape**: `201 Created` with `StatusCheckDomain` (id, robotId, script metadata, `result`, `nextStepId`, `history`, timestamps, computed `statusCheck` name). On validation errors an HTTP 400 with concatenated messages is returned automatically by middleware.
- **Side Effects**: Writes to `status_check` & `status_check_poller` tables, may subscribe to Megazord events, schedule teardown jobs, and emit messages to robot queues when evaluations finish.

### GET /v1/status-check/check/:statusCheckId

- **Method/Path**: `GET /v1/status-check/check/:statusCheckId`.
- **Validation & Guards**: Same Kong + robot checks; `statusCheckId` path param validated as integer via `StatusCheckIdParamDto`.
- **Execution Flow**: Controller fetches the logged-in robot and calls `StatusChecksService.getById`. The service loads the active model, ensures it belongs to the caller, pulls template details (to populate friendly `statusCheck` names/history), and returns the domain object.
- **Responses**: `200 OK` with `StatusCheckDomain` when found; `404 NotFoundError` if the ID does not exist or belongs to another robot; `400` when the path param fails validation.

## Key Services, Repositories & Logic

### StatusChecksService (`src/services/StatusCheckService.ts`)

- Implements `Modules.IAsyncModule`. During `init` it begins consuming the configured `statusQueue.address` via Tiny Backend Tools’ SQS client and spawns `handleStatusCheckSQSMessage` workers with `setImmediate`.
- `create()` orchestrates a new status check by: fetching the Micro Managers step (`MicroManagersService.getScriptStep`), extracting the template name and timing windows (`parseStatusCheckScriptName` / `parseStatusCheckScriptInterval`), loading the template via `StatusTemplatesLoaderService`, persisting the row with initial poller bounds, and executing a past-events validation if there is a `pollingSince` window.
- `executePastCheck()` pulls every required past event from Megazord (`MegazordEventsService.getIncomingEventsSince`), sorts them chronologically, maps them to `StatusCheckHistoryRecord` entries, runs the compiled template validator, writes DB records via the active model, and if a future window exists, registers subscriptions via `registerFutureCheck()`.
- `registerFutureCheck()` subscribes to each dependency in `template.resources.futureEvents.dependsOn` through Megazord. It transitions the phase to `FUTURE`, saves the subscription id, and leaves the status check in `PENDING` until events arrive.
- `handleStatusCheckSQSMessage()` deserializes Megazord outgoing event notifications (wrapping them in `OutgoingEventSQSMessage`), finds the matching status check by subscription, skips stale events (outside `pollingUntil`), executes future validations (`executeFutureCheck()`), and acknowledges or fails the SQS message.
- `executeFutureCheck()` merges new outgoing events into history, re-runs the template future validator, persists results, publishes robot notifications through `StatusChecksSQSProducerService`, and, when done, locks the poller and delegates to the teardown scheduler.
- `performValidation()` is the central state machine: appends history entries, chooses next phase/result/next step based on validator output and `nextStepsResource`, warns if a phase transition happens without a matching `nextStepId`, and keeps everything idempotent by relying on `StatusCheckActiveModel` flush semantics.
- `getById()` reloads the aggregate, ensures robot ownership, and rehydrates templates so the `statusCheck` getter works.

### StatusCheckTeardownSchedulerService (`src/services/StatusCheckTeardownSchedulerService.ts`)

- Also an async module. On `init` it sets up a `Pool.TimerPool` and two cron jobs (`SimpleContextCronJob`) to periodically acquire expiring pollers (`acquireStatusCheckTeardownCron`) and to clean up stale locks (`releaseStatusCheckTeardownCron`).
- `poll()` locks `status_check_poller` rows whose `UNTIL` is within the acquire window (`StatusChecksRepository.acquireByPollerUntil`), loads the matching active models, and schedules each through the timer pool.
- `schedule()` enqueues a timer that will call `execute()` at `pollingUntil`. The callback wraps errors, logs them, and emits application error details if needed.
- `execute()` reloads the status check, ensures the same lock signature, unsubscribes from Megazord events, marks unfinished checks as `FAILED`, derives the final result/next step from latest history, flushes DB state, notifies robots, and unlocks the poller.
- `cleanup()` releases stale locks by nullifying pollers whose `updated_at` is older than `releaseStatusCheckTeardownWindowMs`.

### StatusChecksSQSProducerService (`src/services/StatusCheckSQSProducer.ts`)

- Wraps an `SQS.ISQSProducer` and robot queue config. `notify()` sends a message per robot to `${robotQueue.address}${robotId}` with a `link` (`/v1/status-check/check/{id}`) that robots can call, plus metadata about the origin service (`from.serviceName = appName`).
- Uses hostname/app name for the “from” envelope and serializes the domain via `toPlain()`.

### StatusTemplatesLoaderService & Compiler (`src/services/StatusTemplatesLoaderService.ts`, `StatusTemplatesCompiler.ts`)

- Loader owns two caches: a process-wide map for precompiled templates (from `schemas/generated`) and an LRU TTL cache for runtime-loaded templates.
- `loadAllPrecompiled()` iterates through every YAML file, compiles them (if needed) with `StatusTemplatesCompiler.parseYaml()` and generated module code, then calls `syncTemplate()` so DB IDs stay aligned.
- `loadRuntimeTemplate()` fetches an “active” template from MySQL, runs it through the in-memory `internalTemplates.parse()` (EJS-based, streaming to `require-from-string`), caches it, and returns a hydrated domain.
- `syncTemplate()` ensures compiled templates exist in DB (creating or deactivating via `StatusTemplatesRepository`), attaches DB ids, and logs mismatches.
- Compiler (`StatusTemplatesCompiler`) locates `schemas/status-templates`, parses YAML via `yaml`, emits generated JS to `schemas/generated/{name}.js`, and exposes helper methods for path inspection and batch compilation. The `internal/templates/TemplatesParser.ts` file turns each EJS expression into exported functions plus two validators (past/future) that fall back to `PENDING` if no expression matches.

### MegazordEventsService (`src/services/MegazordEventsService.ts`)

- Axios-based client that injects the request context as `Call-Ref` header, logs both requests and responses, and wraps failures in `Utils.ApplicationError` + `UnprocessableEntityError`.
- Methods: `getIncomingEventsSince()` (GET `/internal/v1/events/robots/{robotId}/incomings?event_name={name}&created_since={unix}`); `subscribe()` (POST `/internal/v1/events/robots/{robotId}/subscriptions` with `eventNames` and `until` ISO string); `unsubscribe()` (DELETE the subscription id). Returns typed domain objects (IncomingEventDomain, SubscriptionDomain).

### MicroManagersService (`src/services/MicroManagerService.ts`)

- Similar Axios wrapper for `micro-managers`. `getScriptStep()` hits `/internal/v5/scripts/robots/{robotId}/scripts/steps/{stepId}` and hydrates a `DetailedScriptStepV5Domain` (step, node, next instructions).
- `parseStatusCheckScriptName()` ensures the node is `statusCheck` and returns the template name + script node.
- `parseStatusCheckScriptInterval()` derives `since` / `until` windows from script parameters or template defaults (supports infinite look-back when `pastInterval` is `-1`), and validates that “pass” and “fail” commands exist in the script’s `multipleChoice` next steps.

### Repositories & Active Models

- `StatusChecksRepository` handles raw SQL inserts/queries for `status_check`, `status_check_poller`, and record tables. Notable APIs: `create()`, `getById()`, `getByRobotIdAndSubscriptionId()`, `acquireByPollerUntil()`, `releaseLockedPoller()`. Inserts are wrapped in explicit transactions.
- `StatusTemplateRepository` manages template descriptions, versions, activation, and detail expressions. It creates the high-level `status_check_template` rows plus per-phase expressions stored in `status_check_template_detail`.
- `StatusCheckActiveModel` is the ORM-like aggregate that loads base rows, histories, pollers, and next-step resources in batches, tracks pending mutations (`addRecord`, `setResult`, `setNextStepId`, `setSubscriptionId`, `setNewPhase`, `lockPoller`, etc.), and flushes everything transactionally while holding row-level locks. It exposes domain-friendly getters and ensures the `StatusCheckDomain` always reflects the latest writes.

## Data & Domain Model

- **StatusCheckDomain / StatusCheckHistoryRecord / StatusCheckNextStepsResource**: Represent the persisted status check, its chronological history (each record tracks event ids, phase, result, timestamps), and the mapping from template results to script next step ids. `StatusCheckPhase` enumerates `CREATED`, `PAST`, `FUTURE`, `COMPLETED`, `FAILED` and is used to gate scheduler behavior.
- **StatusTemplateDomain**: YAML-driven entity that contains metadata (`name`, `version`, `description`, `isActive`, `pollUntilEnd`), optional poller defaults, and `resources` split into `pastEvents` and `futureEvents`. Each resource describes dependencies (event names) plus optional EJS expressions for PASS/FAIL/UNKNOWN outcomes.
- **Micro Managers Script Domains**: `DetailedScriptStepV5Domain` captures script step metadata, node definitions (including `ScriptNodeStatusCheck` parameters), and the set of `NextMultipleChoice` instructions that determine which commands map to next steps.
- **Megazord Domains**: `IncomingEventDomain` and `SubscriptionDomain` mirror Megazord’s API responses, enabling the service to translate event payloads into status-check records and to track subscription lifecycle.
- **SQSMessageDomain / OutgoingEvent**: Wrap SQS payloads describing outgoing events, including who sent the message (`from`), who should receive it (`to.robotId`), how to fetch enriched data (`link`), and the outgoing event body (with references to source incoming events, provider, subscription, etc.).
- **Configs & DTOs**: `AppConfig`, `CacheConfig`, `SchedulerConfig`, `SQSConfig`, `QueueConfig`, `SlackConfig`, and Kong DTOs describe runtime parameters and driver settings; DTOs under `src/models/DTOs/` enforce request validation.

## External Dependencies & Cross-Service Contracts

- **MySQL (tinybots schema)** – Primary persistence for status checks, pollers, templates, history, script metadata. Accessed through Tiny Backend Tools’ `Database` wrapper and custom SQL. Migrations are managed separately (e.g., via the `typ-e` service referenced in `ci/docker-compose.yml`).
- **Kong gateway** – All controller routes require Kong headers (`KongHeader` DTO) and authenticated robot identities. `robotValidator` middleware (from `tiny-backend-tools`) ensures only robot service accounts invoke the API.
- **Micro Managers service** – HTTP API (default base URL `http://MICRO_MANAGERS_ADDRESS`) for retrieving script steps. Requires `Call-Ref` header per request and returns JSON conforming to the `DetailedScriptStepV5Domain` schema.
- **Megazord Events service** – HTTP API (base URL `http://MEGAZORD_EVENTS_ADDRESS`) that exposes incoming events and subscription management. Used both synchronously (fetch history) and asynchronously (subscribe/unsubscribe). Event timestamps are compared to `pollingUntil` to decide when to ignore stale SQS messages.
- **AWS SQS / Localstack** – Two queues are involved: `statusQueue.address` is the shared queue where Megazord places outgoing events; `robotQueue.address` is a prefix for per-robot queues that receive `StatusCheckDomain` notifications. Tiny Backend Tools’ SQS clients handle polling, ack/fail, and sending messages. Local development uses Localstack (see `ci/docker-compose.yml`).
- **Tiny Backend Tools** – Provides the application base class, awilix integration (`Modules.AwilixWrapper`), request context propagation, logging, Cron helpers, SQS client, validation middleware, serializer, and error wrappers.
- **Slack notifications** – Slack config (`SlackConfig`) and `SlackService` are registered in the container for future alerting, although the service does not yet make Slack calls.

## Testing & Tooling

- **Unit/Integration tests**: `yarn test` runs `nyc mocha --require tsx` across `test/**/*.ts`, enforces coverage thresholds (statements/functions/lines ≥ 94%, branches ≥ 70%), emits HTML coverage, and runs ESLint afterwards (`package.json`).
  - `test/controllers/StatusCheckControllersIT.ts` spins up the full Express app against a local MySQL (through `Azi3StatusCheckDbSetup`), stubs Megazord/Micro Managers with `nock`, and exercises create/get flows plus validation failures.
  - Service-level suites (`test/services/*`) mock dependencies with `ts-mockito` to cover Megazord/MicroManagers adapters, template compiler/loader, SQS producer, teardown scheduler, and the main status-check workflow (including error paths and scheduler interactions).
  - Repository suites under `test/repositories/` hit a real MySQL database (credentials provided via env) to verify SQL queries, locking semantics, and template persistence.
  - `test/helpers/DbSetup.ts` extends `DashboardAccountDbSetup` to seed robots, subscriptions, events, and script metadata.
- **Dev workflows**: `yarn build` compiles TypeScript via `tsconfig.prod.json`. `yarn start` runs the compiled app (`dist/cmd/app`). `yarn lint` uses ESLint, while `yarn lint:format` also runs `dprint fmt` to enforce formatting.
- **Docker & CI**: The `Dockerfile` performs a two-stage build (install + build, then production runtime). `ci/docker-compose.yml` provisions Localstack, MySQL, Typ-e, Checkpoint, and Prowl for end-to-end testing; `ci/node-verify.sh` waits for dependencies and runs the test suite. Concourse pipeline files (`ci/build-*.yml`, `ci/test-*.yml`) orchestrate CI/CD.

## Gaps & Risks

- `package.json` exposes a `start:job` script that points to `dist/cmd/job`, but the repository only contains `src/cmd/app` (no `cmd/job`). Invocations of this script will fail unless a job entrypoint is added.
- Slack configuration and services are registered in `src/cmd/app/main.ts`, yet no code actually uses `SlackService`. This unused dependency increases boot time and can drift without tests or visibility.
- Credentials for local MySQL (including the root password) are hard-coded in `test/helpers/DbSetup.ts` and `ci/docker-compose.yml`, which is risky for credential hygiene and complicates secret rotation.
- `StatusChecksRepository.exist()` (and supporting SQL) is unused by `StatusChecksService`, so nothing prevents duplicate status checks per robot/template/result even though the repository has the ability to check. Consider enforcing deduplication before inserting.
- `StatusCheckActiveModel.saveNewRecords()` still contains a TODO about deadlocks and currently locks the whole status check row to avoid them. High concurrency or long-running flushes may still deadlock other writers unless the underlying issue is resolved.
- `StatusChecksService.poll()` dispatches `handleStatusCheckSQSMessage()` via `setImmediate` without awaiting or throttling. Errors rely on implicit promise rejection handling inside `handleStatusCheckSQSMessage`; under heavy load this pattern can drop rejections or overwhelm the process.
