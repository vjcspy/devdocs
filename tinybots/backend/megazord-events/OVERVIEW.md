> **Branch:** develop
> **Last Commit:** 097c2bf (Merged in feature/fix-yarn4-issue, PR #52)
> **Last Updated:** Tue Dec 16 03:01:31 2025 +0000

# Megazord Events

## TL;DR
- Ingests robot telemetry and stores structured incoming events, schemas, providers, and subscriptions; seeds schemas from `schemas/events/*.json` (now includes `SHORT_INACTIVITY`, `LONGER_IN_BED_SHORT/LONG`, updated `NO_TOILET_ACTIVITY_ALARM`).
- Fans out every incoming event into outgoing events per subscription; service subscriptions publish to the status queue (optionally suffixed per subscription), trigger subscriptions call the Trigger Service.
- Caches schemas/providers with cron-driven refreshes after a full startup load, keeping lookups fast and consistent with the database.
- Registers robot/event pairs with external adaptors (currently Sensara) whenever subscriptions change so hardware feeds stay aligned.

## Recent Changes
- Added optional `queue` field on subscription creation; when set, outgoing events send to `${statusQueue.address}-{queue}`. Validates alphanumeric/hyphen and length ≤128; integration/unit tests cover queue routing (PR #49).
- Added `SHORT_INACTIVITY` schema plus generator guard for existing files and `FORCE_GENERATE=true` regen path; exposed `yarn generate:schemas` script (PR #47).
- Added `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` schemas; removed unused Sensara adaptor mappings for these events (Nov 19 2025).
- `NO_TOILET_ACTIVITY_ALARM` schema added then tuned to level 10 with `hasTrigger=false` (PR #46).
- Hotfix ensures all schemas load before cache cron initialization to avoid stale caches on boot (commit 4a01277).
- Yarn toolchain bumped to 4.12 with CI fixes for Yarn 4 workspaces (PRs #50–52).

## Repo Purpose & Bounded Context
- Source of truth for event ingestion and routing across TinyBots: manages `incoming_event`, `outgoing_event`, `event_schema`, `event_provider`, and `event_subscription`.
- Exposes internal APIs for services to post robot events and manage subscriptions; exposes admin/dashboard trigger subscription endpoints gated by Kong/permission validators.
- Provides cross-platform notifications via SQS (status queue or per-subscription queue suffix) and the Trigger Service, and coordinates registration with Sensara through adaptor hooks.

## Project Structure
- `src/cmd/app/main.ts`: Awilix DI wiring, Kong/MySQL boot via `TinyDatabaseAppAuthenticated`, middleware setup, Sensara adaptor registration, and route binding.
- `src/controllers`: Incoming and subscription controllers exposing internal routes plus admin trigger routes; simulate endpoint enabled only when `ENVIRONMENT=academy`.
- `src/services`: Business logic layers (incoming events, subscriptions, outgoing events, schema/provider caches, schema loader, trigger HTTP client, aggregated adaptor registry, Sensara adaptor).
- `src/repositories`: SQL accessors for events, schemas, providers, and subscriptions; includes prepared statements and transactional inserts for subscription + detail rows.
- `src/models`: DTO validators, domain models with `provide()` hydration, and config classes for app, polling, SQS/status queue, services, and permissions.
- `schemas/`: `gen.ts` generates JSON schema seeds into `schemas/events`; loader pulls them into the database at startup.
- `config/`: Default and dev config for MySQL, Kong, service addresses, status queue, polling schedule, and permissions provider.
- `test/`: Integration tests for controllers and unit tests for services/repositories; helpers for DB and permission fixtures.
- `ci/`: Docker-compose based test harness wiring MySQL, typ-e schema service, Checkpoint, and Prowl.

## Controllers & Public Surface
- `GET /internal/v1/events/robots/:robotId/incomings`: Filter incoming events by robot, optional `event_name` and `created_since` Unix timestamp (validated via DTO).
- `POST /internal/v1/events/robots/:robotId/incomings`: Create an incoming event (validates provider/schema), logs request context.
- `POST /v1/events/robots/:robotId/incomings/simulate`: Academy-only when `ENVIRONMENT=academy`; requires Kong headers and `userRobotAccessValidator`.
- `POST /internal/v1/events/robots/:robotId/subscriptions`: Create service or trigger subscriptions (array of event names, optional `until`, optional `queue` suffix for service subscriptions; `queue` must be ≤128 chars, alphanumeric/hyphen).
- `DELETE /internal/v1/events/robots/:robotId/subscriptions/:subscriptionId`: Deactivate a subscription.
- `GET /internal/v1/events/robots/:robotId/subscriptions/:subscriptionId/outgoings/:outgoingEventId`: Fetch outgoing event after robot/subscription authorization.
- Trigger-specific routes (internal + admin): create/list/delete trigger subscriptions; `/v1/...` routes require admin validator plus `M_O_TRIGGERS_SETTING_WRITE_ALL`.

## Core Services & Logic
- **Bootstrap (`App` in `src/cmd/app/main.ts`)**: Registers all modules with Awilix, wires context/logging/serializer middleware, binds routes, and adds Sensara adaptor to the aggregated registry. `createApp()` loads config files via `loadConfigValue`.
- **EventSchemasLoader**: Locates `schemas/events`, reads JSON definitions (`eventName`, `level`, `hasTrigger`, `isActive`, `description`), and upserts `event_schema` records at startup.
- **EventSchemasService / EventProvidersService**: Warm caches on init and schedule cron refreshes using `pollingConfig.dbPolling`, polling rows updated since the previous refresh via prepared statements. Provide by-id/name lookups; throw `NotFoundError` on invalid references.
- **IncomingEventsService**: Validates schema/provider, maps DTO to repository request, writes `incoming_event`, hydrates it with domains, and emits `incoming_event_created` on the event emitter.
- **EventSubscriptionsService**: On init, subscribes to the emitter to react to new incoming events. `subscribe()` validates event names, rejects duplicate active trigger subscriptions, writes subscription + detail rows transactionally, and registers robot/event pairs with external adaptors (sensara). For each new incoming event, creates outgoing events and:
  - `SERVICE_SUBSCRIPTION` → sends SQS message to `statusQueue.address` or `${statusQueue.address}-{queue}` when a `queue` suffix is provided (payload is the outgoing event domain + link).
  - `TRIGGER_SUBSCRIPTION` → POSTs to Trigger Service with event metadata.
  - Default branch falls back to SQS.
  Tracks background tasks for graceful shutdown; `unsubscribe()` deactivates DB rows and unregisters adaptor registrations.
- **OutgoingEventsService**: Persists outgoing events with status `CREATED`, hydrates them with source incoming event for consumers, and supports list-by-subscription.
- **AggregatedEventsAdaptorsService / SensaraEventsAdaptorService**: Registry that forwards register/unregister calls to all adaptors. Sensara adaptor maps Tinybots events to sensor registration endpoints and forwards `Call-Ref` from request context.
- **TriggerService**: Minimal axios client posting `CreateTriggerDto` to `${TRIGGER_SERVICE_ADDRESS}/internal/v1/triggers/triggers` for trigger subscriptions.

## External Dependencies & Cross-Service Contracts
- Kong: `KongFig` plus validators (`robotValidator`, `userRobotAccessValidator`, admin/permission validators) wrap routes; relies on Checkpoint/Prowl/Wonkers addresses from config.
- AWS SQS: `SQS.ContextSQS` producer publishes outgoing-event payloads to `statusQueue.address` (or suffixed queue when provided); credentials/endpoint come from `sqsConfig`.
- Trigger Service: HTTP dependency for trigger subscriptions at `TRIGGER_SERVICE_ADDRESS`.
- Sensara adaptor service: HTTP client for registering robot/event subscriptions; uses `TinybotsEvent` mapping from `tiny-internal-services`.
- Database: MySQL backing all event tables; repositories use prepared statements and connection caching for schema/provider polling.
- Shared libs: `tiny-backend-tools` (DI, HTTP middleware, cron, DB repo base, validation), `tiny-specs` (HTTP schema validators in tests), `tiny-internal-services` (event enums).

## Gap Analysis
- Error handling for event fan-out is muted: `handleNewlyAddedIncomingEvent` swallows exceptions without logging; failures could silently drop notifications.
- TriggerService lacks context propagation, retries, and timeouts; transient failures could block trigger-type subscriptions without fallback or telemetry.
- Schema/provider loader runs only at startup; runtime changes rely solely on `updated_at` polling—ensure DB triggers or processes bump timestamps when schema/provider records change.
- Sensara adaptor registry currently hard-coded; future provider-specific behaviour needs configuration to avoid sending unsupported events.
- Per-subscription queue suffix assumes the target queue exists; missing queues will drop messages silently—consider queue existence/metric checks before publish.
