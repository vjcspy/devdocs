# Sensara Adaptor Backend – Onboarding Brief

## TL;DR

- Bridges Sensara’s resident telemetry (notifications, ADL events, last-known location) with Tinybots robots by translating, storing, and forwarding events plus managing resident↔robot mappings.
- Exposes a small admin/public surface for resident onboarding and internal endpoints for notifications, poller orchestration, pilot-report storage, and event backfills; everything else runs via background jobs and pollers started on boot.
- Stateful data lives in MySQL (`tinybots`) and the analytics database, while long-running jobs keep Sensara Server-Sent Events (SSE) streams healthy and raise Slack alerts on drift.

## Repo Purpose & Interactions

Sensara Adaptor is a Node.js/TypeScript backend that keeps Tinybots robots in sync with Sensara’s care platform. It stores which robot listens to which resident, listens to Sensara SSE streams, propagates activity/location triggers to the internal Event Service, and offers supporting endpoints for pilot reporting and manual notifications (`src/App.ts:281-334`). The app bootstraps via `TinyDatabaseAppAuthenticatedPermissions`, so it inherits context/logging middleware, admin auth, and permission checks for the public admin API (`src/App.ts:257-305`).

### Sensara REST + SSE
- Direction: Bi-directional.
- Surface: HTTPS `/v3/notifications`, `/v3/streams`, `/v3/hardware/measurements/last-location` plus a custom SSE client.
- Purpose: Register resident streams, pull last-known locations, and push alert notifications.
- Auth & resilience: OAuth password grant via `SensaraAuthenticationService`; `SensaraApiService` retries once then Slack-notifies and throws (`src/sensara/SensaraApiService.ts:62-193`).
- Code: `src/sensara/**`, `src/jobs/SensaraEventsJob.ts`, pollers.

### Tinybots Event Service
- Direction: Outbound.
- Surface: HTTPS `/internal/v1/events/robots/:id/incomings` via `EventService`.
- Purpose: Emit Tinybots events such as hearing-range transitions, ADL mappings, and activity bursts.
- Auth & resilience: No explicit retry in adapters; upstream client handles HTTP errors.
- Code: `src/jobs/LocationPoller.ts:106-113`, `src/jobs/ActivityPoller.ts:97-104`, `src/jobs/SensaraEventsJob.ts:167-175`.

### Slack Webhooks
- Direction: Outbound.
- Surface: `tb-ts-slack-notification`.
- Purpose: Surface Sensara API/stream failures and poller issues.
- Auth & resilience: Fire-and-forget; most callers swallow errors after notifying.
- Code: `src/jobs/LocationPoller.ts:62-91`, `src/jobs/SensaraEventsJob.ts:84-87`, `src/sensara/SensaraApiService.ts:76-119`.

### MySQL `tinybots`
- Direction: Inbound storage.
- Surface: Tables `sensara_resident_robot`, `sensara_location_polling`, `tessa_hearable_location`, `sensara_event*`.
- Purpose: Persist resident bindings, scheduled pollers, and ingested events.
- Auth & resilience: Connections via `tiny-backend-tools` `Database`; transactional writes with rollbacks.
- Code: `src/repository/ResidentRepository.ts:13-154`, `src/repository/LocationRepository.ts:5-64`, `src/repository/SensaraEventRepository.ts:6-79`.

### MySQL `analytics`
- Direction: Inbound storage.
- Surface: Table `sensara_pilot_report`.
- Purpose: Capture pilot script telemetry and expose a reporting API.
- Auth & resilience: Separate pool via `IntelligenceDatabase`.
- Code: `src/repository/PilotReportRepository.ts:6-55`.

### Kong / Dashboard Permissions
- Direction: Inbound protection.
- Surface: Kong headers plus the Tinybots permission service.
- Purpose: Enforce admin access and `SENSARA_RESIDENT_WRITE_ALL` on public endpoints.
- Auth & resilience: `ValidationMiddleware.headerValidator` combined with `usePermissionValidatorMiddleware`.
- Code: `src/App.ts:281-299`.

## Repository Inventory

- `src/App.ts` – Express bootstrap, Awilix DI container registration, middleware stack, and route wiring (`src/App.ts:120-337`).
- `src/server.ts` – Entry point that creates the app, runs background jobs (events stream + poller restarts) before listening (`src/server.ts:1-9`).
- `src/controller/` – HTTP controllers for residents, locations, notifications, and internal information aggregation.
- `src/service/` – `ResidentService` and `LocationService` orchestrate repositories, pollers, and downstream services.
- `src/jobs/` – Long-running workers (`SensaraEventsJob`, `RestartPollerJobs`) plus `LocationPoller`/`ActivityPoller` implementations.
- `src/eventsource/` – Custom fork of `eventsource` that tracks last contact time and wraps Sensara SSE logic.
- `src/sensara/` – API + authentication clients and environment-specific wrappers.
- `src/repository/` – Data mappers for MySQL (`ResidentRepository`, `LocationRepository`, `SensaraEventRepository`, `PilotReportRepository`, `IntelligenceDatabase`).
- `src/model/` – DTOs for validation, domain objects (e.g., `ResidentRobot`, Sensara event models), config classes.
- `test/` – Integration tests (`controller`, `repository`) that hit real MySQL, plus unit tests for jobs, models, and Sensara clients.
- `config/` – `node-config` JSON files for MySQL, Kong, Sensara OAuth, Slack.
- `docs/` – OpenAPI fragments and diagrams describing the public admin API.
- `ci/` & `Dockerfile` – Build/publish assets; Dockerfile performs Yarn 3 build, compiles TypeScript, and runs `yarn start`.

## Data & Integration Map

### `ResidentRobot` (`sensara_resident_robot`)
- Description: Stores `{id, residentId, robotId}` per resident↔robot binding; hearable locations live alongside in `tessa_hearable_location` (`robot_id`, `location`).
- Usage: Managed transactionally so registration inserts/updates both tables (`src/repository/ResidentRepository.ts:66-104`). Controllers rely on conflict detection to guard poller creation.

### `LocationPollConfig` (`sensara_location_polling`)
- Description: Persists `{id, robot_id, type, until}` when a poll is scheduled; rows join back to `sensara_resident_robot` so restarted pollers can resolve resident IDs.
- Usage: `LocationRepository.getPollers()` returns active rows for `RestartPollerJobs` and watchers (`src/repository/LocationRepository.ts:28-65`).

### `SensaraEvent` + `sensara_event_schema`
- Description: Stores events with `sensara_id`, `resident_robot_id`, payload, and schema metadata; schema rows are lazily created per event name/type.
- Usage: `SensaraEventsJob` writes every SSE event before forwarding downstream, and `InternalInformationController.getSensaraEvents` queries via `SensaraEventRepository.getEventReport` with optional `createdSince` filtering (`src/repository/SensaraEventRepository.ts:42-79`).

### `SensaraEventReport`
- Description: DTO representing joined event data with resident + robot context, returned verbatim from `/internal/v1/sensara/events`.
- Usage: Served by `InternalInformationController.getSensaraEvents` (`src/controller/InternalInformationController.ts:54-68`).

### `PilotReport` (`sensara_pilot_report` in analytics DB)
- Description: Stores script execution telemetry (serial, robot, script version/execution IDs, planned/executed timestamps, measurement/message).
- Usage: Inserted through `/internal/v1/sensara/reports/pilot` and fetched for reporting/backfills (`src/repository/PilotReportRepository.ts:30-55`).

### DTOs (`ResidentRegistrationDto`, `RegisterLocationPollDto`, `SensaraNotificationDto`, etc.)
- Description: Class-validator DTOs ensuring incoming payloads contain required types.
- Usage: Applied via `ValidationMiddleware` on every route (`src/App.ts:281-334`).

## Controllers / Public Surface

### `PUT /v1/sensara/residents`
- Request: `ResidentRegistrationDto` (`residentId`, `robotId`, `hearableLocations[]`).
- Response: 200 JSON `ResidentRobotRegistration`.
- Guards: Kong header validator + admin check + `SENSARA_RESIDENT_WRITE_ALL` + DTO validation (`src/App.ts:281-289`).
- Behavior: `ResidentController.registerResident` logs with context, upserts the resident + hearable locations transactionally, returns the mapping, then reloads the events stream to include the new resident (`src/controller/ResidentController.ts:23-44`). Tested via `test/controller/ResidentControllerIT.ts:61-110`.

### `DELETE /v1/sensara/residents/:residentId`
- Request: `ResidentIdPathDto` path param.
- Response: 204 on success, 404 if resident missing.
- Guards: Same Kong/admin/permission middleware as the PUT route (`src/App.ts:290-299`).
- Behavior: Calls `ResidentService.deleteResident` and converts `ResidentRobotNotFoundError` to HTTP 404 (`src/controller/ResidentController.ts:46-68`). Tests in `test/controller/ResidentControllerIT.ts:112-148`.

### `POST /internal/v1/sensara/notification`
- Request: `SensaraNotificationDto` (`robotId`, `notificationType`, `timeStamp`, `message`).
- Response: 204.
- Guards: Body validation only (internal endpoint) (`src/App.ts:300-304`).
- Behavior: Controller resolves the resident via `ResidentService`, converts to `SendNotificationDto`, and delegates to `SensaraApiService.sendNotification`; unknown residents raise 409 (`src/controller/NotificationController.ts:22-49`). Tests mock Sensara/Event Service with `nock` (`test/controller/NotificationControllerIT.ts:50-140`).

### `POST /internal/v1/sensara/residents/location/register`
- Request: `RegisterLocationPollDto` (`robotId`, ISO `until`).
- Response: 201 with empty body.
- Guards: Body validation only (`src/App.ts:306-310`).
- Behavior: Looks up the resident, persists a poll config, creates a `LocationPoller`, responds immediately, and starts polling asynchronously (`src/controller/LocationController.ts:23-46`). Conflict thrown for unknown robots. Integration tests verify validation, conflict handling, and event emission (`test/controller/LocationControllerIT.ts:53-141`).

### `POST /internal/v1/sensara/residents/activity/register`
- Request: `RegisterLocationPollDto`.
- Response: 201.
- Guards: Same as the location register route (`src/App.ts:312-315`).
- Behavior: Mirrors `pollLocation` but instantiates an `ActivityPoller` that checks recent activity timestamps (`src/controller/LocationController.ts:48-71`). Tests in `test/controller/LocationControllerIT.ts:143-229`.

### `POST /internal/v1/sensara/reports/pilot`
- Request: `PilotReportDto` (script metadata; `executedAt` required).
- Response: 201.
- Guards: Body validation (`src/App.ts:318-321`).
- Behavior: Persists telemetry in the analytics DB via `PilotReportRepository.addReportItem`; controller only logs and stores the payload (`src/controller/InternalInformationController.ts:20-35`). Tests cover validation errors + happy path (`test/controller/InternalInformationControllerIT.ts:75-126`).

### `GET /internal/v1/sensara/reports/pilot`
- Request: `PilotReportQueryDto` with optional `createdSince`.
- Response: 200 JSON array sorted ascending.
- Guards: Query validation (`src/App.ts:324-328`).
- Behavior: Fetches results from analytics; consumers can supply an ISO timestamp for incremental pulls (`src/controller/InternalInformationController.ts:37-52`). Tests exercise filtering semantics (`test/controller/InternalInformationControllerIT.ts:130-186`).

### `GET /internal/v1/sensara/events`
- Request: `EventQueryDto` optional `createdSince`.
- Response: 200 JSON array of `SensaraEventReport`.
- Guards: Query validation (`src/App.ts:330-334`).
- Behavior: Serves persisted Sensara events for analytics/backfill consumers (`src/controller/InternalInformationController.ts:54-69`). Tested in `test/controller/InternalInformationControllerIT.ts:188-239`.

Cross-cutting concerns: every controller extracts a `context` for structured logging via `tiny-backend-tools` and relies on the global `errorMiddleware` for consistent 4xx/5xx responses (`src/App.ts:257-337`).

## Key Services & Logic

- **ResidentService & ResidentRepository** – Thin wrapper that upserts resident↔robot rows and hearable locations transactionally, deletes residents, and resolves mappings by resident or robot (`src/service/ResidentService.ts:15-27`, `src/repository/ResidentRepository.ts:66-154`). Errors: throws `ResidentRobotNotFoundError` on delete; other DB errors bubble up after a rollback. No retries; natural idempotency because duplicate registrations overwrite.
- **LocationService + Pollers** – Registers polling jobs in DB and instantiates poller classes with all dependencies (`src/service/LocationService.ts:32-89`). `LocationPoller` normalizes Sensara location/label strings, posts `CLIENT_IN_HEARING_RANGE` events when the resident enters configured locations, and deletes the DB poller once triggered or when `until` expires. Errors trigger Slack notifications and poller cleanup (`src/jobs/LocationPoller.ts:54-116`). `ActivityPoller` looks for last activity within 120 seconds to emit `ACTIVITY` events (`src/jobs/ActivityPoller.ts:52-108`). Neither poller retries API calls beyond Slack-notified failures; they rely on `SensaraApiService` raising `InternalServerError`.
- **SensaraApiService & Authentication** – Wraps two OAuth clients (main/test environment) and exposes helpers to send notifications, register/delete streams, fetch SSE connections with signed fetch, and query last locations (`src/sensara/SensaraApiService.ts:62-193`). Retries once by default (configurable via constants) before notifying Slack and throwing `InternalServerError`. Tokens cached by `SensaraAuthenticationService`, which refreshes 60 seconds before expiry (`src/sensara/SensaraAuthenticationService.ts:20-65`).
- **SensaraEventsJob & SensaraEventSource** – On boot, fetches all residents, registers two SSE streams (main + test) through `SensaraEventSource`, and schedules both heartbeat checks (every 30s) and periodic restart (every 6h) (`src/jobs/SensaraEventsJob.ts:65-128`). `SensaraEventSource` manages connecting, toggles between pending/current streams, and notifies Slack if retry backoff exceeds configured timeout (`src/eventsource/SensaraEventSource.ts:69-225`). Incoming events are persisted and mapped onto Tinybots events via `convertEvent`, covering ADL, Sensara notifications, and extramural state transitions, before posting to the Event Service (`src/jobs/SensaraEventsJob.ts:139-221`). Test residents are isolated via `sensaraTestResident`.
- **RestartPollerJobs** – Reads outstanding pollers from DB and restarts either activity or location pollers when the process boots (`src/jobs/RestartPollerJobs.ts:11-46`). All creation exceptions are logged; no retry/backoff beyond initial attempt.
- **Internal Data Services** – `PilotReportRepository` writes to analytics; `SensaraEventRepository` lazily registers event schemas; `LocationRepository` ensures only future pollers are returned; `IntelligenceDatabase` is a thin subclass pointing to the analytics pool (`src/repository/*.ts`).

## Runtime / Request Flow

1. **Boot** – `createApp()` loads configs (MySQL, Kong, permissions, Sensara OAuth, Slack) via `loadConfigValue`, registers dependencies in Awilix, and applies middleware (`src/App.ts:394-438`). `server.ts` then calls `runEventsJob()` and `restartPollers()` before awaiting `start()` so background processes are already running when the HTTP server comes up (`src/server.ts:1-9`).
2. **Inbound admin requests (resident CRUD)** – Kong headers are validated, admin/tokens checked, and DTOs parsed. `ResidentController` writes to MySQL through `ResidentService` and immediately triggers `SensaraEventsJob.reload()` so SSE subscriptions include the new resident (`src/controller/ResidentController.ts:23-44`). Deletes cascade to both resident and hearable location tables.
3. **Internal poller registration** – Internal services call `/internal/v1/sensara/residents/{location|activity}/register`. After validation, controllers look up the resident, create a poller row, respond 201, and start the poller. Pollers fetch Sensara → post Tinybots events → call `LocationRepository.removePoller`. Slack notifications fire on API failure.
4. **Sensara SSE ingestion** – `SensaraEventsJob` obtains resident lists, registers SSE streams via `SensaraApiService.getStream`, and wires event listeners for `AdlEventResponse`, `NotificationResponse`, and `StateExtramuralResponse` (`src/eventsource/SensaraEventSource.ts:96-143`). `handleEvent` hydrates resident context, persists events, converts them to Tinybots semantics, and posts to the Event Service (`src/jobs/SensaraEventsJob.ts:139-177`). Heartbeats monitor `timeOfLastContact`, and `reload` re-establishes streams if the gap exceeds 120 seconds.
5. **Pilot reports & telemetry dump** – Internal tooling pushes script execution reports via `/internal/v1/sensara/reports/pilot`, which simply stores records to analytics. Consumers can page through aggregated reports or persisted events via `GET /internal/v1/sensara/{reports|events}` for backfills or audits.

## Tests & Quality Signals

- **Controller IT suites** (`test/controller/*.ts`) spin up the full app against real MySQL schemas using helper `DbSetup` and `PermissionDbSetup`. They assert validation errors, permission enforcement, conflict handling, successful 2xx responses, and integration behavior (e.g., `LocationControllerIT` mocks Sensara + Event Service with `nock` and checks that events were posted, `test/controller/LocationControllerIT.ts:75-228`).
- **Repository IT suites** hit live tables to prove persistence logic (upsert semantics, poller filtering, schema creation). See `test/repository/ResidentRepositoryIT.ts`, `LocationRepositoryIT.ts`, `SensaraEventRepositoryIT.ts`, and `PilotRepositoryIT.ts`.
- **Job & Service unit tests** rely on `ts-mockito` and `tiny-internal-services-mocks` to assert poller retry/cleanup behavior, event conversions, and SSE reconnect logic (`test/jobs/*.ts`). `SensaraEventsJobTest.ts` covers ADL, notification, and extramural translations plus stream restarts.
- **Sensara client tests** (`test/sensara/*.ts`) mock OAuth/token endpoints and REST APIs to verify retry thresholds, Slack notification triggers, and token caching semantics.
- **Model tests** confirm `SensaraEvent.fromEvent` conversions (`test/model/sensara/SensaraEventTest.ts`).
- **Coverage & lint gating** – `yarn test` runs Mocha via `ts-node` with NYC instrumentation, checks coverage thresholds (89/85/60/89), and runs `yarn lint` afterwards (`package.json:7-18`). Tests currently require environment variables pointing to MySQL hosts and event service URLs; root DB passwords are hardcoded in the suites, so CI must provide isolated databases.

Green test suite confidence: resident onboarding/offboarding, notification delivery, poller registration/execution, telemetry ingestion, and SSE persistence all have either integration or unit coverage. No automated tests cover the internal `/internal/v1/sensara/events` consumer beyond retrieval.

## External Dependencies & Cross-Service Contracts

- **`tiny-backend-tools`** – Provides the app base class, DI helpers, middleware (context logging, serializer, validation), config loaders, DB abstractions, and permission middleware. Controllers depend on context + logging from here (`src/App.ts:257-337`).
- **`awilix`** – Backing DI container used through `Modules.AwilixWrapper` to resolve controllers/services/jobs on demand (`src/App.ts:108-255`).
- **`tiny-internal-services` + `tiny-internal-services-mocks`** – Supplies the `EventService` HTTP client, DTOs (e.g., `IncomingEventBodyDto`, `TinybotsEvent`, `SensaraNotificationType`), and test doubles for event posting (`src/jobs/LocationPoller.ts:2-15`, `test/jobs/LocationPollerTest.ts:1-108`).
- **`tb-ts-http-errors` & `tb-ts-slack-notification`** – Unified error helpers and Slack webhook client; used to surface 4xx/5xx to callers and ops (`src/controller/LocationController.ts:37`, `src/jobs/LocationPoller.ts:62`).
- **`kong-js`** – Supplies `KongHeader` schema for validating Kong-provided headers on admin routes (`src/App.ts:281-299`).
- **`tiny-backend-testing-tools`** – Integration-test helpers for provisioning admin accounts/permissions and DB fixtures (`test/controller/ResidentControllerIT.ts:1-58`, `test/util/DbSetup.ts`).
- **Shared Config Contracts** – `config/default*.json` defines required secrets/environment variables (DB hosts, Sensara OAuth credentials, Slack hook, etc.), and `custom-environment-variables.json` maps them to env var names. Deployments must deliver both `mysql` and `mysql-intelligence` creds plus `sensara`/`testSensara` OAuth pairs.

## Gaps & Risks

- **Duplicate Slack registration** – `App.extendContainer` registers `ContainerNames.SLACK_SERVICE` twice (`src/App.ts:149-157`), so later bindings may overwrite the first or mask configuration errors; confirm DI state or remove the duplicate.
- **Internal endpoints lack auth** – `/internal/v1/sensara/...` routes only perform validation and accept unauthenticated requests (`src/App.ts:300-334`). If exposed beyond trusted networks, they could trigger arbitrary pollers, notifications, or data writes; consider at least Kong key auth or shared-secret middleware.
- **Last-location retry disabled** – `SensaraApiService.getLastLocation` logs an error and immediately throws without retry, even though a retry strategy is commented out (`src/sensara/SensaraApiService.ts:166-193`). Transient Sensara outages will therefore tear down pollers quickly and spam Slack.
- **Unused helper** – `SensaraEventJobUtils` is only referenced from tests (`src/jobs/SensaraEventJobUtils.ts`, `test/jobs/SensaraEventJobUtilsTest.ts`), suggesting dead production code or missing integration. Decide whether to wire it into heartbeat monitoring or delete.
- **Controller fire-and-forget pollers** – `LocationController` sends 201 before `poller.startPolling()` finishes, and any immediate exception (e.g., Sensara outage) only logs to console/Slack with no HTTP feedback (`src/controller/LocationController.ts:39-71`). Consider awaiting first execution or persisting failure status so callers can re-request confidently.
- **Hardcoded root DB credentials in tests** – Integration tests embed plaintext root passwords/hosts (`test/controller/ResidentControllerIT.ts:17-46`, `test/controller/InternalInformationControllerIT.ts:25-44`), which is risky if the repo becomes public and complicates running tests without dedicated sandboxes.
- **Startup error handling** – `server.ts` calls `theApp.stop()` inside the catch even if `createApp()` rejected before assigning `theApp`, which would cause a crash masking the original error (`src/server.ts:4-9`). Add a guard before stopping.
