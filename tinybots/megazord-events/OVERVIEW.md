# Megazord Events — Repository Overview

> Megazord Events is the Tinybots backend that ingests robot events, keeps a catalog of event schemas/providers, and fan-outs notifications (SQS, HTTP triggers, external adaptors) to the rest of the platform.

## 1. Purpose & Scope
- Acts as the source of truth for `incoming_event`, `outgoing_event`, `event_schema`, `event_provider`, and `event_subscription` tables (see repositories under `src/repositories`).
- Provides internal APIs so other services can post robot activity (`/internal/v1/events/...`) and subscribe to event streams.
- Emits outgoing events whenever a new incoming event arrives, so downstream processors can react through:
  1. AWS SQS messages (`statusQueue`) for service subscriptions.
  2. HTTP calls to the Trigger Service for trigger subscriptions.
  3. Registration hooks to external sensors/adaptors (currently Sensara) so they know which robot/event pairs to send us.
- Relies heavily on `tiny-backend-tools` for HTTP middleware, database access, validation, permissions, cron jobs, and SQS wrappers.



## 2. Important term

### schema

- table `event_schema`

- typing: `src/models/domains/EventSchemaDomain.ts`

Ban đầu nó sẽ được load từ `schemas/events` files, sync từ trong này vào trong db if it doesn't already exist

Nó sẽ được reload liên tục trong `src/services/EventSchemasService.ts` dựa trên cronjob(cronjob sử dụng từ `tiny-backend-tool` repository) để cho vào cache mục đích để lấy cho nhanh

### provider

- table `event_provider`
- typing `src/models/domains/ProviderDomain.ts`

Similarly to `schema`, it is also cached and is reloaded into the cache periodically by cronjob

## 2. Controllers / Public Surface
### IncomingEventsController (`src/controllers/IncomingEventsController.ts`)
- `GET /internal/v1/events/robots/:robotId/incomings` — filter stored events by robot, event type (`event_name`), and `created_since`.
- `POST /internal/v1/events/robots/:robotId/incomings` — create an incoming event from an internal service (validates provider & event schema first).
- `POST /v1/events/robots/:robotId/incomings/simulate` — academy-only endpoint (enabled when `ENVIRONMENT=academy`) allowing dashboard users to simulate events if they own the robot (`userRobotAccessValidator`).

#### Flows

##### Create Incomming Event

Chủ yếu quan tâm đến việc tạo event. Hiện tại chưa rõ là bên nào is internal calling to create new event.

```typescript
export class CreateIncomingEventBodyDto {
  @Expose()
  @IsString()
  providerName!: string

  @Expose()
  @IsString()
  eventName!: string

  @Expose()
  @IsNumber()
  @IsOptional()
  // level can be optional in the DTO request but later will be filled from eventSchema via provide()
  level?: number

  @Expose()
  @IsString()
  referenceId?: string
 }
```



- Khi API nội bộ tạo sự kiện mới, service `IncomingEventsService` tra cứu `schema` & `provider` hợp lệ, ghi bản ghi vào bảng
  `incoming_event`, sau đó emit sự kiện vào event bus (src/services/IncomingEventsService.ts:63).
- EventSubscriptionsService đã đăng ký listener từ lúc init, nên mỗi incoming event mới sẽ dẫn tới
  `handleNewlyAddedIncomingEvent`, service này lấy tất cả subscription còn active của robot cho đúng eventTypeId (src/
  services/EventSubscriptionService.ts:132).
- Với từng subscription tìm thấy, hệ thống tạo một `outgoing_event` để theo dõi trạng thái gửi đi nhờ
  OutgoingEventsService.create, rồi attach thêm dữ liệu incoming event cho payload (`src/services/
  EventSubscriptionService.ts:173`, `src/services/OutgoingEventService.ts:32`).

Ý Nghĩa Khối switch Subscription
- SubscriptionType.SERVICE_SUBSCRIPTION: publish thông báo qua SQS tới status queue nội bộ bằng notify,
  payload là dữ liệu outgoing event đã chuẩn hóa kèm link để consumer truy ngược API nếu cần (`src/services/
  EventSubscriptionService.ts:187`).
- SubscriptionType.TRIGGER_SUBSCRIPTION: thay vì push lên queue, gọi TriggerService.sendTrigger để POST tới Trigger
  Service, truyền CreateTriggerDto với thông tin `robot`, `level`, `event type` nhằm kích hoạt `automation/trigger` downstream
  (`src/services/EventSubscriptionService.ts:196`, `src/services/TriggerService.ts:10`).
    - call to endpoint: `internal/v1/triggers/triggers`
- default: fallback trở lại SQS để đảm bảo những loại subscription chưa biết vẫn nhận thông báo như service
  subscription bình thường (src/services/EventSubscriptionService.ts:199).
- Hai nhánh SERVICE & default dùng chung notify, hàm này đóng gói message với metadata from, to, link,
  payload, rồi đẩy lên hàng đợi cấu hình tại statusQueueConfig.address qua SQS.ContextSQS (src/services/
  EventSubscriptionService.ts:322).

### EventSubscriptionController (`src/controllers/EventSubscriptionsController.ts`)
- `POST /internal/v1/events/robots/:robotId/subscriptions` — create or trigger-type subscriptions (takes `eventNames`, optional `until`, optional `isTriggerSubscription`).
- `DELETE /internal/v1/events/robots/:robotId/subscriptions/:subscriptionId` — soft-deactivate a subscription.
- `GET /internal/v1/events/robots/:robotId/subscriptions/:subscriptionId/outgoings/:outgoingEventId` — fetch an outgoing event after authorizing robot/subscription linkage.
- Trigger-specific endpoints (both internal and dashboard/admin versions):
  - `POST /internal|/v1/events/robots/:robotId/subscriptions/triggers` — single `eventName`, creates a trigger subscription, enforces uniqueness.
  - `GET /internal|/v1/events/robots/:robotId/subscriptions/triggers` — list active trigger subscriptions for a robot.
  - `DELETE /internal|/v1/events/robots/:robotId/subscriptions/:subscriptionId` or `/internal|/v1/events/robots/:robotId/subscriptions/triggers/:subscriptionId` — deactivate trigger subscriptions. Public (`/v1/...`) routes require admin auth + `M_O_TRIGGERS_SETTING_WRITE_ALL`.

These controllers are bound in `src/cmd/app/main.ts`, which wires validators (`ValidationMiddleware`, `robotValidator`, `userRobotAccessValidator`, admin/permission validators) provided by `tiny-backend-tools`.

## 3. Core Services & Responsibilities (all under `src/services`)
### EventSubscriptionsService
- Registers an event handler on `EventsName.INCOMING_EVENT_CREATED` during `init()`.
- `subscribe()` validates event names through `EventSchemasService`, enforces single active trigger subscription per `eventName`+robot, writes to `event_subscription` + `_detail`, and calls `AggregatedEventsAdaptorsService.register()` so external adaptors know about the subscription. Returns hydrated `SubscriptionDomain`.
- `handleNewlyAddedIncomingEvent()` loads active subscriptions for the event type + robot, creates `outgoing_event` records via `OutgoingEventsService`, and fan-outs per subscription type:
  - `SERVICE_SUBSCRIPTION` → enqueue SQS message (payload is the serialized `OutGoingEventDomain`) via `SQS.ContextSQS`, targeting `statusQueue.address`.
  - `TRIGGER_SUBSCRIPTION` → call `TriggerService.sendTrigger()` (HTTP POST to Trigger Service) with robot/event metadata.
- `unsubscribe()` deactivates DB rows and unregisters events from external adaptors.
- Tracks background tasks to coordinate graceful shutdown when the app stops.

### IncomingEventsService
- Responsable for validating (`eventSchemasService`, `eventProvidersService`) and persisting incoming events, hydrating them with schema/provider domains, and emitting `EventsName.INCOMING_EVENT_CREATED` so `EventSubscriptionsService` can react.
- `getAll()` builds repository filters from query params (`createdSince`, event name) and gracefully handles missing schemas/providers by logging errors.

### OutgoingEventsService
- Persists outgoing events and rehydrates them with the source incoming event (`IncomingEventsService.getByIdNoAuth()`) so consumers receive event metadata (schema/provider) even if they only have the outgoing ID.

### EventSchemasService & EventProvidersService
- Cache layers around their respective repositories. On `init()` they:
  - Warm the cache.
  - Start Cron jobs using `pollingConfig.dbPolling` to periodically reload rows updated since the last refresh.
- Provide O(1) lookups by id or name, throwing `NotFoundError` for invalid references (tests rely on these errors).

### EventSchemasLoader
- On startup, scans `schemas/events/*.json`, inserts/updates schemas, and logs successes/failures. Useful when adding new event types without manual SQL.

### AggregatedEventsAdaptorsService & SensaraEventsAdaptorService
- `AggregatedEventsAdaptorsService` is a registry of provider-specific adaptors (currently only Sensara). When subscriptions are (un)registered, it iterates over adaptors and forwards robot/event pairs.
- `SensaraEventsAdaptorService` maps Tinybots event names (from `tiny-internal-services` `TinybotsEvent` enum) to Sensara adaptor endpoints:
  - Location events → `/internal/v2/sensara/residents/location/register`.
  - Activity/hearing events → `/internal/v1/...`.
- Adds the request context's `Call-Ref` header for traceability.

### TriggerService
- Simple axios wrapper that POSTs `CreateTriggerDto` (`outgoingEventId`, `robotId`, `level`, `eventTypeId`) to `${TRIGGER_SERVICE_ADDRESS}/internal/v1/triggers/triggers`.

## 4. Persistence & Domain Models
- Domain objects live in `src/models/domains`. They all extend `BaseDomain`, have `provide()` methods to hydrate related entities, and expose computed getters (e.g., `SubscriptionDetailDomain.eventName`, `IncomingEventDomain.providerName`).
- Repositories in `src/repositories` wrap SQL for each table:
  - `IncomingEventsRepository`: insert/filter incoming events.
  - `OutgoingEventsRepository`: insert/outgoing lookups.
  - `EventSubscriptionsRepository`: transactional insert into `event_subscription` + `_detail`, filtering (with grouping), deactivation.
  - `EventSchemasRepository` & `ProvidersRepository`: maintain prepared statements + cached DB connections for efficient polling.
- Configuration classes under `src/models` (`AppConfig`, `PollingConfig`, `SQSConfig`, `ServicesConfig`, `PermissionsConfig`) are decorated with `class-validator` rules and loaded via `loadConfigValue()` in `createApp()`.

## 5. Runtime / Request Flow
1. **Bootstrap (`src/cmd/app/main.ts`)**
   - `App` extends `TinyDatabaseAppAuthenticated`, wiring Kong, MySQL, SQS, and Awilix DI.
   - Registers singletons for repositories/services/controllers, sets up middlewares (context, logging, serializer), and binds routes.
   - Registers the Sensara adaptor with the aggregated adaptor service so subscriptions automatically call out to Sensara.
2. **Incoming event ingestion**
   - HTTP request hits `IncomingEventsController.create()` → DTO validation → `IncomingEventsService.create()` writes DB row and emits the event.
3. **Fan-out**
   - `EventSubscriptionsService` reacts to the emitter, loads matching subscriptions, creates outgoing events, enqueues SQS messages, or POSTs to Trigger Service depending on `SubscriptionType`.
4. **Queries**
   - Consumers read events via controllers, which in turn rely on domain objects already hydrated with schema/provider data for consistent responses (`WebAppValidators.validate('Subscription', ...)` in tests).

## 6. External Dependencies & Cross-Service Contracts
- **tiny-backend-tools**: Provides the majority of infrastructure (Express middlewares, validators, Repository base class, Awilix wrapper, Cron, SQS producer, TinyDatabaseAppAuthenticated, permission checks).
- **tiny-specs**: Supplies shared HTTP schema validators used in tests (`WebAppValidators`).
- **tiny-internal-services**: Defines `TinybotsEvent` enum for Sensara mapping.
- **Kong ecosystem**: `KongFig`, `robotValidator`, `userRobotAccessValidator`, and headers like `x-consumer-id` enforce gateway policies.
- **Checkpoint / Prowl / Wonkers**: Configured via `config/*.json` as robot/user/permission services.
- **Trigger Service**: Receives trigger payloads for `SubscriptionType.TRIGGER_SUBSCRIPTION`.
- **Sensara adaptor service**: Receives subscription registrations/unregistrations so physical sensors know what to report.
- **AWS SQS / status queue**: `SQS.ContextSQS` publishes outgoing-event payloads to whatever queue URL/ARN is configured in `statusQueue.address`.

## 7. Test Suites Worth Reading
- `test/controllers/IncomingEventsControllerIT.ts` — end-to-end coverage for filtering, creation, and academy simulation flows (validations, authentication, errors).
- `test/controllers/EventSubscriptionControllerIT.ts` — full trigger/service subscription lifecycle, permissioning, Sensara adaptor expectations (via `nock`), SQS side effects, and edge cases (duplicate triggers, auth failures).
- `test/services/*.ts` — unit tests for caching services, loaders, adaptors, trigger service, outgoing-event creation, etc. These show expected behaviour of each service without HTTP.
- Helpers (`test/helpers/DbSetup.ts`, `PermissionDbSetup.ts`) show which tables/fixtures the system relies on when running locally.

## 8. Operational Notes
- Key env vars: `TRIGGER_SERVICE_ADDRESS`, `SENSARA_ADAPTOR_SERVICE_ADDRESS`, `STATUS_QUEUE_ADDRESS`, `WONKERS_USER_ACCOUNT_ADDRESS`, `WONKERS_INTERNAL_ADDRESS`, AWS creds for SQS, and `DB_RW_HOST`.
- `pollingConfig.dbPolling` determines cache refresh cadence (`*/2 * * * *` by default, faster in `default.dev.json`).
- When adding new event types, drop JSON definitions into `schemas/events` so `EventSchemasLoader` can persist them automatically.
- Graceful shutdown waits for `EventSubscriptionsService` background handlers to finish to avoid losing outgoing events.
