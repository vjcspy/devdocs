> **Branch:** master
> **Last Commit:** e0a35b4
> **Last Updated:** Tue Nov 18 08:14:29 2025 +0000

# tiny-internal-services-mocks Overview

## TL;DR
- TypeScript helper library exporting mocks for Tiny Internal Services clients; uses ts-mockito and nock to stub HTTP calls while shaping DTOs with class-transformer.
- Source lives in `lib/mocks`, compiled output in `dist`, typedoc in `docs`, and mocha/nyc tests in `test` to keep mocks aligned with client behavior.
- Depends on `tiny-internal-services` (1.23.0) and `tiny-specs` for shared DTOs covering robots, dashboard, Taas orders/subscriptions, signals, sensara, scripts, and notifications.

## Repo Purpose & Bounded Context
- Provides reusable test doubles for internal service clients so downstream services can run integration/unit tests without hitting live endpoints.
- Covers robot accounts/status/settings, dashboard resources, event subscriptions, device signals, sensara adaptor flows, script metadata, notifications, and Taas subscription/order lifecycles using the same DTOs as production clients.

## Project Structure
- `lib/index.ts`: Re-exports all mock classes plus input interfaces for Taas orders/robot accounts.
- `lib/mocks/*.ts`: Per-service mocks combining ts-mockito stubs and nock interceptors (dashboard robots/accounts, robot accounts/status/settings, device signals, events, sensara adaptor, scripts, notifications, Taas services/orders/activation, user accounts).
- `test/*.ts`: Mocha specs exercising each mock against the real client classes to verify nocked vs non-nocked behavior.
- `dist/`: Compiled JS and d.ts outputs.
- `docs/`: Typedoc output generated from `lib`.
- `coverage/`, `.nyc_output/`: nyc coverage artifacts; `.vscode/`, `.idea/` hold local tooling configs; `.yarnrc.yml` pins Yarn 3.

## Controllers & Public Surface
- Exported classes: `DashboardRobotServiceMock`, `RobotSettingsServiceMock`, `RobotStatusServiceMock`, `UserAccountServiceMock`, `DashboardAccountServiceMock`, `ScriptServiceMock`, `TaasOrderServiceMock` (with `TaasOrderMockInput`, `ClientUuidTaasOrderMockInput`, `TaasIdTaasOrderMockInput`), `DeviceSignalsServiceMock`, `RobotAccountServiceMock` (with `RobotAccountMockInput`), `TaasServiceMock` (with `TaasMockInput`, `TaasMockInputV4`), `TaasOrderActivationServiceMock`, `EventServiceMocks`, `SensaraAdaptorServiceMocks`, `NotificationServiceMock`.
- Each exposes `getMock()`/`getInstance()` plus paired `mock*` (ts-mockito expectations) and `nock*` (HTTP interceptors) methods targeting `/internal/*` service routes.

## Core Services & Logic
- DashboardRobotServiceMock: Builds V3/V4/V5 dashboard robots (class-transformer) and stubs lookups by account id, hardware id, or relation filters; nocks `/internal/v4/dashboard/robots/accounts/:id/robot`, `/internal/v5/dashboard/robots/accounts/:id/robot`, `/internal/v3/admin/robots/:serial`, `/internal/v5/admin/robots/:id`, and V4 admin filters.
- RobotAccountServiceMock: Stubs searches by serial/email/id, pairing invite CRUD, user removal/role updates, and virtual/organizational robot creation; nocks `/internal/v2/robots/accounts/search`, `/internal/v2/robots/accounts/:id`, `/internal/v3/robots/accounts/:id/pairing-invites`, `/robots/accounts/:id/users/*`, etc.
- RobotStatusServiceMock: Mocks heartbeats/status lists and online queries; nocks `/internal/v1/robot-status` POST (single/multi) and `/internal/v2/robot-status/search`/`/online` GET endpoints.
- RobotSettingsServiceMock: Returns `ReceivedRobotSettingsV3` and nocks `/internal/v3/settingsrobot/robots/:robotId`.
- DeviceSignalsServiceMock: Covers connect/disconnect/update/online/offline flows for hardware/subscription/taas IDs; nocks `/internal/v1/signals/robots/hardware/:serialId/*` and `/internal/v1/signals/robots/accounts/:robotId/*`.
- EventServiceMocks: Builds subscriptions/trigger subscriptions and stubs post/get/delete for events and triggers; nocks `/internal/v1/events/robots/:id/incomings|subscriptions|subscriptions/triggers`.
- SensaraAdaptorServiceMocks: Stubs sensara notifications, pilot reports, and event feeds with optional query filters; nocks `/internal/v1/sensara/notification`, `/reports/pilot`, and `/events`.
- ScriptServiceMock: Stubs script step type lookups via `/internal/v3/scripts/:scriptReferenceId/step/types`.
- NotificationServiceMock: Stubs welcome notifications using `NotificationsRegistry` UUIDs via `/internal/v1/notifications/:notificationUuid/users/:userId`.
- TaasServiceMock: Builds Taas subscription payloads, stubs get by id, link robot (v3/v4), patch, and create flows including failure cases; nocks `/internal/v3/admin/taas/:id` and `/internal/v4/admin/taas` link/create/patch routes.
- TaasOrderServiceMock: Uses class-transformer to compose `V5TaasOrder` and stubs searches by hardware, relation, status, clientUuid, or taasId; also activation and install success/failure; nocks `/internal/v4/taas-orders` queries and `/internal/v5/taas-orders/installations*` endpoints.
- TaasOrderActivationServiceMock: Stubs job runs via `/internal/v1/order-activation/job/run`.
- DashboardAccountServiceMock: Stubs account lookup via `/internal/v3/admin/accounts/:id`.
- UserAccountServiceMock: Builds paired user accounts per robot and nocks `/internal/v3/users/robots/:robotId/users`.

## External Dependencies & Cross-Service Contracts
- `tiny-internal-services@1.23.0` provides the DTOs and HTTP clients the mocks wrap, keeping mocked payloads aligned with production contracts.
- `tiny-specs@1.4.0` supplies dashboard/taas model enums and DTO typings used in mock builders.
- Mocking stack: `ts-mockito` for class doubles, `nock` for HTTP interception, `class-transformer` (and `class-validator` compatibility) for DTO shaping.
- Tooling: TypeScript 5.1.3, ts-node, mocha, chai, nyc, standardx lint, typedoc; Yarn 3.8.7 declared via `packageManager`.

## Testing & Quality Gates
- `npm test` (declared script) runs standardx lint on `lib/**/*.ts`, mocha via ts-node, nyc coverage with HTML output under `coverage/`, and enforces 50% thresholds for statements/functions/branches/lines.
- Tests cover each mock class to confirm nocked HTTP calls resolve and real clients throw when not mocked (e.g., DeviceSignals connect/disconnect, RobotAccount search/pairing, Event subscription flows, Taas order retrieval/activation).
- `npm run docs` generates typedoc docs into `docs/`; `npm run build` runs tests, cleans `dist`, emits compiled JS/typings, and regenerates docs.
- No CI config present; align mock updates with `tiny-internal-services`/`tiny-specs` version bumps to avoid contract drift.
