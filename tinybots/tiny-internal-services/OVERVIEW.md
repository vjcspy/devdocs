# tiny-internal-services — Repository Overview

**TL;DR**: Shared TypeScript SDK that centralizes Tinybots domain models and service clients (robots, Taas orders, subscriptions, Sensara, notifications). Downstream services import these classes to call other internal APIs with consistent DTO validation, logging, and error handling.

## Table of Contents
- [Repo Purpose & Interactions](#repo-purpose--interactions)
- [Repository Inventory](#repository-inventory)
- [Controllers / Public Surface](#controllers--public-surface)
- [Key Services & Logic](#key-services--logic)
- [Runtime / Request Flow](#runtime--request-flow)
- [External Dependencies & Cross-Service Contracts](#external-dependencies--cross-service-contracts)
- [Tests & Quality](#tests--quality)
- [Gaps & Risks](#gaps--risks)
- [Notes](#notes)

## Repo Purpose & Interactions
`tiny-internal-services` is *not* a standalone HTTP app. It is a “platform contracts” package that exposes:
1. Plain TypeScript domains/DTOs (robot accounts, Taas orders, Sensara payloads, Tinybots events, etc.).
2. Axios-backed service clients that other repos (dashboards, megazord-events, sensara-adaptor, Taas tooling) use to call the actual microservices.
3. A small runtime utility (`HardwareTypeLoader`) that loads DB-provided hardware types into strongly typed data classes via `tiny-backend-tools`.

Because all service calls funnel through this SDK, it enforces logging with `IRequestContext`, normalized `tb-ts-http-errors`, and shared enums (for example `TinybotsEvent`). It therefore sits in the middle of many flows: onboarding robots, linking Taas subscriptions, relaying Sensara alarms, running notification jobs, or manipulating dashboard accounts.

| Integration | Direction | Protocol & Paths | Notes |
| --- | --- | --- | --- |
| Dashboard Account Service | Outbound | `GET /internal/v3/admin/accounts/:accountId` | Used by `DashboardAccountService` to load admin/owner metadata. |
| Dashboard Robot Service | Outbound | `GET /internal/v4 admin/robots`, `GET /internal/v5/admin/robots/:id`, etc. | Powers robot search plus V5 hardware fetches (logs via `Logger`, requires `Call-Ref`). |
| Robot Account Service | Outbound | `PATCH /internal/v2/robots/accounts/:id`, invite CRUD, role assignments. | Handles pairing invites, user removal, and virtual robot creation. |
| Robot Settings Service | Outbound | `GET /internal/v3/settingsrobot/robots/:robotId` | Fetches `ReceivedRobotSettingsV3` and convenience accessors (language). |
| Robot Status Service | Outbound | `/internal/v1|v2/robot-status*` | Returns heartbeat/status DTOs and uses `class-transformer` for hydration. |
| Device Signals Service | Outbound | `/internal/v1/signals/robots/...` | Sends connect/disconnect/offline/online signals for hardware. |
| Event Service (Megazord) | Outbound | `/internal/v1/events/robots/:robotId/...` | Posts incoming events, manages (trigger) subscriptions, piped with context logging. |
| Sensara Adaptor Service | Outbound | `/internal/v1/sensara/*` | Sends notification payloads, pilot reports, and fetches Sensara event history. |
| Buy Subscription Service | Outbound | `/internal/v3/admin/subscription-chains` | Lists subscription chains for commerce flows. |
| Taas Service | Outbound | `/internal/v3|v4/admin/taas/*` | Links robots to Taas, creates V4 subscriptions, handles conflict detection. |
| Taas Order Service | Outbound | `/internal/v4|v5/taas-orders*` | Reads/updates orders, installations, and status transitions. |
| Taas Order Activation Service | Outbound | `/internal/v1/order-activation/job/run` | Triggers long-running activation job. |
| Notification Service | Outbound | `/internal/v1/notifications/:uuid/users/:id` | Sends templated notifications (welcome). Requires `Call-Ref`. |
| User Account Service | Outbound | `/internal/v3/users/robots/:robotId/users` | Lists users paired to a robot. |
| Script Service | Outbound | `/internal/v3/scripts/:id/step/types` | Fetches workflow step types per script reference. |

## Repository Inventory
```
tiny-internal-services/
├── lib/
│   ├── constants/RequestOrigin.ts           # Shared enums for request context tagging.
│   ├── model/                               # DTOs/domains (robots, Taas orders, events, Sensara).
│   └── services/                            # Axios-based clients + HardwareTypeLoader module.
├── dist/                                    # Compiled JS + type declarations (generated).
├── docs/                                    # Typedoc output of lib/ (generated).
├── package.json / tsconfig*.json            # Build configuration (yarn@3.8.7).
└── scripts (yarn build/docs/git-add)        # Build + doc generation helpers.
```
- `lib/model/events` provides DTOs for megazord-events (incoming events, subscription definitions, `TinybotsEvent` enums).
- `lib/model/sensara` contains notification/report DTOs that mirror Sensara APIs; also reused by sensara-adaptor.
- `lib/model/hardware*.ts` implements custom `BaseDomain` types so hardware IDs are type-safe and lazily cached.
- `lib/services/index` wraps each upstream HTTP API; every method catches Axios errors, logs, and throws semantic `tb-ts-http-errors`.
- `HardwareTypeLoader` (in `lib/services/HardwareTypeLoader.ts`) is the only module implementing `Modules.IAsyncModule` and needs a provider capable of returning hardware type rows at startup.

## Controllers / Public Surface
This package exports classes via `lib/index.ts` rather than Express controllers. Consumers instantiate the exported service objects with base URLs pulled from environment config. Core public surface:
- `BuySubscriptionService`, `DashboardAccountService`, `DashboardRobotService`, `RobotAccountService`, `RobotSettingsService`, `RobotStatusService`, `UserAccountService`, `DeviceSignalsService`, `NotificationService`, `ScriptService`.
- Taas-focused clients: `TaasService`, `TaasOrderService`, `TaasOrderActivationService`.
- Event bridge: `EventService` (incoming events & subscriptions) and `SensaraAdaptorService`.
- Domain exports: DTOs for Taas orders/subscriptions, relations, robot accounts, events, Sensara notifications, `RequestOrigin`, `HardwareType*`, etc.
- Runtime module: `HardwareTypeLoader` (exposed so host apps can preload DB hardware types and enforce validation).

## Key Services & Logic
### Dashboard, Account & Robot APIs
- `DashboardAccountService` (`lib/services/DashboardAccountService.ts`): GET `/internal/v3/admin/accounts/:id`. Throws `InternalServerError` when axios fails.
- `DashboardRobotService`: wraps multiple versions (v3/v4/v5) of dashboard robot endpoints. Methods support filtering (`getRobots`), fetching by hardware or account, and retrieving `DetailedRobotV4` with `Call-Ref` logging.
- `RobotAccountService`: large surface for robot lifecycle—status changes, lookups by serial/email, pairing invite CRUD, removing users, adapting roles, creating virtual/organizational robots. Side effects: HTTP PATCH/POST/DELETE; throws 500 unless 409 conflict is expected (virtual creation/new org robot rely on upstream validation).
- `RobotSettingsService`: fetches robot settings and exposes `getLanguage` helper that throws `ConflictError` if no default language exists.
- `RobotStatusService`: fetches robot heartbeat data (v1/v2). Uses `class-transformer` to hydrate into `RobotHeartBeat`/`RobotStatus` models, providing typed data for monitoring dashboards.

### Device telemetry & notifications
- `DeviceSignalsService`: centralizes connect/disconnect/order-status/online/offline POST calls for robot hardware IDs. Accepts either Taas IDs or subscription IDs when deregistering devices.
- `NotificationService`: currently only exposes `sendWelcomeNotification`, but is structured to hold a registry of more notification UUIDs. Logs request/response around axios POST.
- `ScriptService`: reads script step types for workflow orchestration (ensures script service remains the single source of truth).

### Events platform
- `EventService` (`lib/services/EventService.ts`): handles posting incoming events, listing events, and managing both general and trigger subscriptions. Every request expects an `IRequestContext` (to propagate `Call-Ref` header and structured logs) and surfaces `InternalServerError` semantics to clients. Private `request` helper centralizes axios method dispatch, logging, and error propagation.
- Event DTOs (`lib/model/events/*.ts`) define the request/response payloads used by megazord-events, ensuring API compatibility.

### Sensara integration
- `SensaraAdaptorService`: posts notifications, creates pilot report measurements, and fetches Sensara events/reports through `/internal/v1/sensara/*`. DTOs in `lib/model/sensara` (notification, pilot reports, event reports) shape the payloads.
- `TinybotsEvent` enum + request DTOs provide shared nomenclature for translating Sensara alarms to Tinybots events; downstream repos rely on this file when adding new event types.

### Taas lifecycle
- `TaasService`: orchestrates linking robots to Taas subscriptions (v3 + v4 API, with detailed `DashboardModels` DTOs). Uses conflict handling for duplicate links and emits `Call-Ref` logs.
- `TaasOrderService`: fetches Taas orders via robot hardware, Taas ID, client UUID, relation ID, or status. Also posts installation success/failure events and activates orders. Handles 409 conflicts for duplicate installations.
- `TaasOrderActivationService`: thin client to trigger the activation batch job.
- `LinkTaasRobotDto`, `V5TaasOrder`, `TaasSubscription`, and installation DTOs define the data surface consumed by the Taas services.

### Buy subscription & user services
- `BuySubscriptionService`: GETs subscription chains (with query params) and converts axios errors into `InternalServerError`.
- `UserAccountService`: fetches robot-user pairings from the user account service, returning typed `PairedUserAccount` entries.

### Hardware type registry
- `HardwareTypeLoader`: implements `Modules.IAsyncModule` so host services can resolve the module via their DI container and invoke `init()` at startup. It consumes `IHardwareTypeProvider.getAll()`, stores IDs into static maps, and populates `HardwareTypeDataType` caches used by DTOs for validation. Logs lifecycle events using `Logger` and honors `stop`.

## Runtime / Request Flow
1. **Configuration**: A consuming service loads base URLs for each upstream system (e.g., `process.env.ROBOT_ACCOUNT_SERVICE_ADDRESS`), instantiates the needed SDK service class, and optionally wires them into a DI container (Awilix is used in other repos).
2. **Context propagation**: For any method that requires logging (EventService, DashboardRobot v5, Taas v4, NotificationService), the caller passes an `IRequestContext` containing a `callRef`. The SDK sets the `Call-Ref` HTTP header and writes debug/error logs through `tiny-backend-tools` `Logger`.
3. **Data hydration**: Responses are cast into DTO classes (some via `class-transformer`) so downstream code can rely on strong typing (enum values, nested relations, statuses). For hardware-specific fields, `HardwareTypeLoader` must have been run so `HardwareTypeDataType` knows all IDs/names.
4. **Error handling**: Every client catches axios errors, logs them (sometimes with structured metadata), and rethrows `InternalServerError` or `ConflictError`. This ensures host services can respond with consistent HTTP semantics even though the SDK is just a library.
5. **Distribution**: `yarn build` compiles `lib/` into `dist/` and runs Typedoc into `docs/`, enabling both runtime consumption and API documentation hosting.

## External Dependencies & Cross-Service Contracts
- **Libraries**: `axios` for HTTP calls, `tb-ts-http-errors` for typed error classes, `tiny-backend-tools` for logging/context/modules, `tiny-specs` for shared Dashboard models, `class-transformer`/`class-validator` for DTO hydration & validation.
- **Authentication/headers**: Most calls rely on static credentials (handled by the network), but methods that accept `IRequestContext` always propagate a `Call-Ref` header for tracing. No other auth headers are set here, so consumers must ensure networking between services is trusted or inject interceptors.
- **Retry/idempotency**: The SDK itself does not retry requests. Some upstream APIs (e.g., linking Taas robots) return HTTP 409; the SDK surfaces this as `ConflictError`. Consumers should implement their own retry/backoff strategies if needed.
- **Data contracts**: DTOs intentionally mirror upstream payloads (see `lib/model/*.ts`). When upstream APIs evolve, both the model and corresponding service must be updated together; forgetting one leads to runtime mismatches because this library is the single source of truth for many repos.

## Tests & Quality
- `package.json`’s `test` script is a placeholder that immediately fails. There are no unit or integration tests in this repo, so correctness is inferred from TypeScript types and consumers’ tests.
- Quality relies on TypeScript compilation (`tsconfig.prod.json` targets `dist/`) and optional `typedoc`. There is no lint script, but the repo includes `dprint.json` and `neostandard` to keep code style consistent.
- Because `lib/` is published directly, any breaking change must also update `dist/` (use `yarn build` + `yarn git-add` helper) to keep the package installable.

## Gaps & Risks
- **Zero automated tests**: Changes to HTTP surface or DTOs cannot be validated here; regressions surface only after consumer services fail at runtime.
- **Manual error mapping**: Most methods catch-all and throw `InternalServerError`, losing upstream error details (except a few conflict cases). Troubleshooting can be painful without additional logging at call sites.
- **Duplicate exports**: `lib/index.ts` repeats certain exports (e.g., `EventQueryParams`, `IncommingEventDomain`, `V5TaasOrder`), increasing the chance of divergent future edits.
- **Runtime dependency on HardwareTypeLoader**: If consuming services forget to run `HardwareTypeLoader` before using `HardwareTypeDataType`, validations will fail silently or throw `invalidHardwareType`.
- **Coupling to upstream paths**: Any API version bump (e.g., `/internal/v6/...`) requires immediate SDK updates; otherwise dozens of downstream repos break simultaneously.

## Notes
- Run `yarn build` to refresh both `dist/` and `docs/`. CI/publish pipelines expect these folders to be kept in sync with `lib/`.
- `yarn docs` regenerates just the Typedoc output (watch the typo in `package.json`: missing space before `--out` on the docs script).
- Consumers typically import from `tiny-internal-services/lib/services/...` but should prefer the package entry point (`import { EventService } from 'tiny-internal-services'`) to benefit from type re-exports.
- When adding new enums (e.g., Tinybots events or Sensara notification types), ensure both the DTO and consuming services (megazord-events, sensara-adaptor) bump their dependency versions together.
