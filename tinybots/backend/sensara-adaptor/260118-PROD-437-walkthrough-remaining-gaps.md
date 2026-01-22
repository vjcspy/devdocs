- **Repo:** `sensara-adaptor`
- **Branch:** `feature/PROD-437-sensara-endpoints`
- **Source:** `devdocs/tinybots/sensara-adaptor/260117-PROD-437-sensara-endpoints-review-and-update.md`

## 1) Ticket Requirements

### 1.1 Requirements

- Ignore authentication for now
- Resident endpoints can be read from the database directly
- For event triggers, use internal endpoints in `tiny-internal-services`
- Use `residentId` to resolve `robotId`

### 1.2 API Specification

**Resident Endpoints**

| Method | Path | Description |
|---|---|---|
| GET | `/v1/ext/sensara/residents` | Get all residents and their robot |
| GET | `/v1/ext/sensara/residents/{residentId}` | Get a resident and their robot |
| PATCH | `/v1/ext/sensara/residents/{residentId}` | Update a resident and their robot |

**Events/Triggers Endpoints**

| Method | Path | Description |
|---|---|---|
| POST | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | Create trigger subscription |
| GET | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | Get robot trigger subscriptions |
| DELETE | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers/{subscriptionId}` | Delete trigger subscription |

## 2) Current State vs. Requirements (Remaining Gaps)

### 2.1 Residents: `GET /v1/sensara/residents` (list residents by organization)

- **Current:** Implemented as `GET /v1/sensara/residents` and requires `x-relation-id` to scope results by organisation.
- **Compared to requirement:** Path does not match spec (`/v1/ext/sensara/residents`)

#### Need to change

- [ ] Confirm whether `x-relation-id` is OK?
- [ ] Add `/v1/ext/sensara/residents` route (or alias) to match the spec.
- [ ] Align implementation with “DB-direct” requirement?

### 2.2 Trigger subscriptions (internal vs external surface)

- **Current:** Implemented under `/internal/v1/events/residents/{residentId}/subscriptions/triggers` (POST/GET/DELETE).
- **Compared to requirement:** Path does not match spec (`/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers`).

#### Need to change

- [ ] Confirm whether trigger subscription endpoints must be exposed under `/v1/ext/...` or remain internal-only.
- [ ] Confirm expected auth posture for these endpoints (ignore auth vs guard).

### 2.3 Authentication posture (“Ignore authentication for now”)

- **Current:** Existing resident write endpoints (`PUT /v1/sensara/residents`, `DELETE /v1/sensara/residents/{residentId}`) still require `SENSARA_RESIDENT_WRITE_ALL`.

#### Need to change

- [ ] Confirm the precise meaning of “ignore authentication” 
- [ ] Update integration tests to reflect the decided behavior.

### 2.4 Missing resident endpoints from spec

- **Current:** `GET /v1/ext/sensara/residents/{residentId}` is not implemented.
- **Current:** `PATCH /v1/ext/sensara/residents/{residentId}` is not implemented.
- **Compared to requirement:** Both are spec-required.

#### Need to change

- [ ] Implement `GET /v1/ext/sensara/residents/{residentId}`.
- [ ] Implement `PATCH /v1/ext/sensara/residents/{residentId}`.
- [ ] Add integration tests for 200/404/400 cases once implemented.

### 2.5 “DB-direct” requirement for resident data

- **Current:** GET-all depends on downstream services (robots + robot accounts) and is therefore not DB-only.

#### Need to change

- [ ] Confirm whether “DB-direct” is strict (no downstream dependencies) or best-effort.
- [ ] If strict, refactor to fetch resident mappings directly from MySQL for the response shape.

### 2.6 Code quality / merge blockers

- **Current:** Review identified merge blockers: `describe.only` in tests, `console.log` in service code, missing DTO validation decorator, incorrect status code (500 vs 400), and potential behavior change (soft delete → hard delete).
- **Compared to requirement:** Not explicitly in ticket spec, but blocks safe merge and production readiness.

#### Need to change

- [ ] Remove `describe.only` from all tests.
- [ ] Replace `console.log` with structured logger usage.
- [ ] Add missing validation decorators on config/DTOs as required.
- [ ] Align invalid `subscriptionId` error response to the correct status code (and update tests).
- [ ] Confirm whether the delete behavior change (soft vs hard delete) is intended and correct.

### 2.7 Build status

- **Current:** `yarn run build` fails on this branch.

#### Need to change

- [ ] Resolve dependency type mismatches (`tiny-internal-services` vs `tiny-backend-tools`, missing `tiny-specs` types).
- [ ] Fix local TypeScript compilation errors in `App.ts`, `LocationController.ts`, `LocationService.ts` (missing ctx args, signature mismatch, required `eventMapper`).

#### Build Log (excerpt)

```text
/Users/kai/.nvm/versions/node/v22.20.0/bin/yarn run build
node_modules/tiny-internal-services/dist/model/hardware.d.ts:3:22 - error TS2305: Module '"tiny-backend-tools"' has no exported member 'ErrorType'.

3 import { BaseDomain, ErrorType, IRequestContext } from 'tiny-backend-tools';
                       ~~~~~~~~~

node_modules/tiny-internal-services/dist/model/hardwareV2.d.ts:2:10 - error TS2305: Module '"tiny-backend-tools"' has no exported member 'ErrorType'.

2 import { ErrorType } from 'tiny-backend-tools';
           ~~~~~~~~~

node_modules/tiny-internal-services/dist/services/DashboardRobotService.d.ts:2:33 - error TS2307: Cannot find module 'tiny-specs' or its corresponding type declarations.

2 import { DashboardModels } from 'tiny-specs';
                                  ~~~~~~~~~~~~

node_modules/tiny-internal-services/dist/services/TaasService.d.ts:2:38 - error TS2307: Cannot find module 'tiny-specs' or its corresponding type declarations.

2 import { type DashboardModels } from 'tiny-specs';
                                       ~~~~~~~~~~~~

src/App.ts:417:14 - error TS2554: Expected 1 arguments, but got 0.

417     eventJob.run()
                 ~~~

  src/jobs/SensaraEventsJob.ts:70:20
    70   public async run(ctx: IRequestContext) {
                          ~~~~~~~~~~~~~~~~~~~~
    An argument for 'ctx' was not provided.

src/App.ts:424:15 - error TS2554: Expected 1 arguments, but got 0.

424     pollerJob.run()
                  ~~~

  src/jobs/RestartPollerJobs.ts:14:20
    14   public async run(ctx: IRequestContext) {
                          ~~~~~~~~~~~~~~~~~~~~
    An argument for 'ctx' was not provided.

src/controller/LocationController.ts:67:7 - error TS2554: Expected 2 arguments, but got 3.

67       body.event
         ~~~~~~~~~~

src/service/LocationService.ts:59:39 - error TS2345: Argument of type '{ id: number; sensaraApi: SensaraApiService; eventService: EventService; slackService: SlackService; locationRepository: LocationRepository; residentRobot: ResidentRobot; hearableLocations: string[]; until: Date; }' is not assignable to parameter of type '{ id: number; sensaraApi: SensaraApiService; eventService: EventService; slackService: SlackService; locationRepository: LocationRepository; ... 5 more ...; specificLocation?: TinybotsEvent; }'.
  Property 'eventMapper' is missing in type '{ id: number; sensaraApi: SensaraApiService; eventService: EventService; slackService: SlackService; locationRepository: LocationRepository; residentRobot: ResidentRobot; hearableLocations: string[]; until: Date; }' but required in type '{ id: number; sensaraApi: SensaraApiService; eventService: EventService; slackService: SlackService; locationRepository: LocationRepository; ... 5 more ...; specificLocation?: TinybotsEvent; }'.

  59     const poller = new LocationPoller({
                                          ~
  60       id: config.id,
    ~~~~~~~~~~~~~~~~~~~~
...
  67       until: config.until
    ~~~~~~~~~~~~~~~~~~~~~~~~~
  68     })
    ~~~~~

  src/jobs/LocationPoller.ts:50:5
    50     eventMapper: EventMapper
           ~~~~~~~~~~~
    'eventMapper' is declared here.

Found 8 errors in 7 files.

Process finished with exit code 2
```

### 2.8 Test status and coverage (PROD-437 scope)

- **Current:** The repo has both unit tests (`*Test.ts`) and integration tests (`*IT.ts`) under `sensara-adaptor/test/`.

#### Need to change

- [ ] Remove `describe.only` from `test/controller/ResidentControllerIT.ts`.
- [ ] Remove `describe.only` from `test/service/ResidentServiceTest.ts`.
- [ ] Add IT coverage for `/v1/ext/sensara/...` once routes are implemented/aliased.
- [ ] Add IT coverage for missing spec endpoints (`GET/PATCH /v1/ext/sensara/residents/{residentId}`) once implemented.
- [ ] Align tests with the decided meaning of “ignore authentication”.
- [ ] Add controller tests for invalid trigger payloads (e.g., missing `eventName`) if validation is enforced.
- [ ] Update/add tests when the invalid `subscriptionId` status code is corrected.
- [ ] Add coverage for DB-only fallback behavior if dependency fallback is implemented.
