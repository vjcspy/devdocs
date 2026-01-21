> **Branch:** develop
> **Last Commit:** 3258c9d (Updated from N/A - metadata missing previously)
> **Last Updated:** Fri Nov 7 04:46:00 2025 +0000

# Sensara Adaptor Overview

## TL;DR
- Node.js/TypeScript adaptor linking Sensara resident telemetry to TinyBots via SSE ingestion, pollers, and admin APIs.
- Maintains resident<->robot bindings, posts TinyBots events for ADL, location, and notifications, and persists Sensara events and pilot reports into MySQL (tinybots + analytics).
- Long-running jobs keep Sensara streams live (main + test), restart pollers, and alert Slack when ingestion stalls or API calls fail.

## Recent Changes Log
- Previous overview lacked metadata; refreshed against commit 3258c9d.
- Added `/internal/v2/sensara/residents/location/register` with `RegisterLocationPollV2Dto` to fire specific `IN_*` TinyBots events from Sensara `LOCATION_*` labels via `LocationEventMapper`, alongside the legacy hearing-range poller.
- Location poller creation now accepts optional `event` mapping, letting pollers emit precise room-level events in addition to `CLIENT_IN_HEARING_RANGE`.

## Repo Purpose & Bounded Context
- Backend service that translates Sensara activity/location into TinyBots events, persists telemetry, and exposes admin/internal endpoints for resident onboarding, notifications, poller scheduling, and reporting.
- Built on `tiny-backend-tools` `TinyDatabaseAppAuthenticatedPermissions`, inheriting context/logging, validation, permissions, and DB wiring (`src/App.ts`).

## Project Structure
- `src/App.ts` sets DI container, middleware, routes, and start/stop lifecycle; resolves controllers/jobs and ties to MySQL + analytics pools.
- `src/server.ts` boots the app, runs `SensaraEventsJob` and `RestartPollerJobs`, then starts HTTP.
- `src/controller/` admin/internal controllers for residents, location/activity pollers, notifications, and reporting.
- `src/service/` core services (`ResidentService`, `LocationService`, `LocationEventMapper`) orchestrate repositories, pollers, and downstream clients.
- `src/jobs/` long-running workers: Sensara SSE ingestion, poller restarts, location/activity pollers.
- `src/eventsource/` custom EventSource fork with last-contact tracking plus Sensara-specific wrapper for registration, reload, and backoff.
- `src/sensara/` OAuth clients and REST/SSE API wrapper for Sensara environments.
- `src/repository/` MySQL access for residents, pollers, Sensara events, and pilot reports; `IntelligenceDatabase` targets the analytics pool.
- `src/model/` DTOs, domain models, and Sensara payload types.
- `config/` node-config defaults and env var mappings; `docs/` OpenAPI fragments/diagrams.
- `test/` integration and unit suites (Mocha + tsx + nock + ts-mockito) with DB fixtures under `test/util`.

## Controllers & Public Surface
- `PUT /v1/sensara/residents` (admin + `SENSARA_RESIDENT_WRITE_ALL`): upserts resident<->robot binding plus hearable locations; triggers SSE reload (`src/App.ts`).
- `DELETE /v1/sensara/residents/:residentId` (same guards): deletes binding, returns 404 on missing resident.
- `POST /internal/v1/sensara/notification`: body-validated Sensara notification; resolves resident by robot ID, sends via Sensara API.
- `POST /internal/v1/sensara/residents/location/register`: schedule hearing-range poller until timestamp; emits `CLIENT_IN_HEARING_RANGE` when last location/label matches stored hearable locations.
- `POST /internal/v2/sensara/residents/location/register`: schedule location poller that emits a specific `TinybotsEvent IN_*` when `LOCATION_*` label matches requested event.
- `POST /internal/v1/sensara/residents/activity/register`: schedule activity poller that emits `ACTIVITY` if recent activity seen within 120s.
- `POST /internal/v1/sensara/reports/pilot`: persist pilot script telemetry into analytics DB.
- `GET /internal/v1/sensara/reports/pilot`: fetch pilot reports, optional `createdSince`.
- `GET /internal/v1/sensara/events`: fetch persisted Sensara event report rows, optional `createdSince`.
- Cross-cutting: Kong header validator + admin/permission middleware on public admin routes; all routes use DTO validation and shared error middleware.

## Core Services & Logic
- **ResidentService/Repository:** transactional upsert of resident<->robot plus hearable locations; conflict detection on delete; helpers to fetch by resident/robot for routing controllers and SSE streams.
- **LocationService & Pollers:** writes poller rows, rebuilds pollers on boot, and constructs poller instances with Sensara API, EventService client, Slack notifier, and optional `LocationEventMapper` event mapping. Location poller polls every 3s until trigger/expiry, posts `CLIENT_IN_HEARING_RANGE` or requested `IN_*` event, and cleans DB row. Activity poller checks `LastLocation.timestamp` within 120s and posts `ACTIVITY`; both enqueue Slack errors and remove poller rows on failure.
- **SensaraEventsJob + SensaraEventSource:** registers SSE streams for main + test residents, resumes from last stored event ID, listens for ADL/notification/extramural events, persists each, maps selected events to TinyBots events, forwards via EventService, and heartbeats every 30s. Automatic reload on stalls or every 6h; Fibonacci backoff with Slack alerts when delays exceed notify threshold.
- **SensaraApiService & Authentication:** OAuth password-grant via per-env auth services; REST calls for notifications, stream registration/deletion, stream fetch with signed `Last-Event-ID`, and last location queries. Retries once for notification/stream operations; last-location retries disabled pending upstream fixes.
- **Repositories:** `LocationRepository` stores pollers (`LOCATION`/`ACTIVITY`) with optional `location_label`; `SensaraEventRepository` lazily ensures event schemas and exposes reports/resume IDs; `PilotReportRepository` writes analytics telemetry with optional `createdSince` filtering.

## SSE Streaming Architecture

### Event Flow: Sensara → TinyBots

```
Sensara API (V3)
    │
    │ SSE Stream (NotificationResponse, AdlEventResponse, StateExtramuralResponse)
    ▼
sensara-adaptor
├── server.ts → runEventsJob()
├── SensaraEventsJob → creates SensaraEventSource
├── SensaraApiService.registerStream() → POST /v3/streams/registrations
│   └── dataTypes: ['NotificationResponse', 'AdlEventResponse', 'StateExtramuralResponse']
├── SensaraEventSource._registerEvents() → addEventListener('NotificationResponse', ...)
├── SensaraEventsJob.handleEvent() → receives event
├── SensaraEvent.fromEvent() → extracts event type (e.g., notificationType for notifications)
└── SensaraEventsJob.convertEvent() → maps to TinybotsEvent
    │
    │ EventService.postEvent()
    ▼
megazord-events
├── Store incoming event
├── Fan out to subscriptions
└── Trigger if hasTrigger=true
```

### SSE Streaming Status

| Event Type | Description | Status |
|------------|-------------|--------|
| `NotificationResponse` | Sensara alarms/notifications (ST_*, LT_*, TA_*) | ✅ Implemented |
| `AdlEventResponse` | Activities of Daily Living (TOILETING, EATING, SLEEPING, etc.) | ✅ Implemented |
| `StateExtramuralResponse` | State changes (BedState, etc.) | ✅ Implemented |
| `LastLocationResponse` | Real-time location updates | ❌ Not implemented (uses polling) |

### Notification Type Mapping (SensaraEventsJob.convertEvent)

| Sensara NotificationType | TinybotsEvent |
|--------------------------|---------------|
| `ST_ACTIVITY_SHORT_INACTIVITY` | `SHORT_INACTIVITY` |
| `ST_ACTIVITY_INACTIVITY` | `SUSPICIOUS_INACTIVITY` |
| `ST_INOUT_OUT_OF_BED` | `EARLY_OUT_OF_BED` |
| `ST_NOT_RETURNED_TO_BED` | `NOT_RETURNED_TO_BED` |
| `ST_SLEEPING_AWAKE_DELAYED` | `LONGER_IN_BED_SHORT` |
| `ST_SLEEPING_AWAKE_LARGE_DELAY` | `LONGER_IN_BED_LONG` |

### ADL Event Type Mapping

| Sensara AdlEventType | TinybotsEvent |
|----------------------|---------------|
| `INSIDE` | `INSIDE_HOME` |
| `OUTSIDE` | `OUTSIDE_HOME` |
| `EATING_GENERAL` / `BREAKFAST` / `EATING` | `EATING_ACTIVITY` |
| `TOILETING` | `TOILET_ACTIVITY` |
| `BATHROOM_VISIT` | `BATHROOM_ACTIVITY` |
| `SLEEPING_ASLEEP` | `IN_BED` |
| `SLEEPING_AWAKE` | `OUT_OF_BED` |

## External Dependencies & Cross-Service Contracts
- Sensara REST/SSE hosts (main + test) for notifications, stream registration/data, and last-location lookups; OAuth credentials per environment.
- TinyBots Event Service (`services.eventServiceAddress`) for posting incoming events tied to robot IDs.
- Slack webhook (`tb-ts-slack-notification`) for API/stream/poller failures.
- MySQL `tinybots` for resident mappings, pollers, and Sensara events; MySQL `analytics` for pilot reports.
- Kong/permissions provider for admin route protection; relies on shared middleware from `tiny-backend-tools`.
- Internal libs: `tiny-internal-services` DTOs + EventService client; `tiny-backend-tools` for app harness, DB, validation, permissions, config.

## Testing & Quality Gates
- Mocha + tsx runners under `test/`; integration suites spin up real MySQL (robot/resident fixtures via `DbSetup`), use nock to mock Sensara/Event Service, and cover controller paths including V2 location registration, notifications, and SSE/event storage.
- Repository tests exercise upserts, poller filtering, schema creation, and pilot report retrieval; job tests cover SSE conversions/restarts and poller behavior.
- `yarn test` runs NYC-instrumented suite (excluding `src/eventsource/**`), generates HTML report, enforces coverage thresholds (statements 89, functions 85, branches 60, lines 89), then runs `yarn lint`.
- Required env vars mapped in `config/custom-environment-variables.json` (DB hosts, Sensara host/auth, event service, permissions provider, Slack hook, test resident); tests depend on accessible MySQL instances.
