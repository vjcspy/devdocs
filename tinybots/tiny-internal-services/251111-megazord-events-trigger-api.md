# 📋 [PROD-438: 2025-11-11] - Update tiny-internal-services + mocks for trigger API

## User Requirements
> We just implemented the new trigger subscription ticket (`megazord-events/devdocs/tinybots/megazord-events/251107-PROD438-internal-trigger-endpoint.md`). After understanding it you must read all relevant source code files.
>
> Because we created three new internal endpoints we have to update `tiny-internal-services` and `tiny-internal-services-mocks`, so read both repos carefully to understand how they currently work.
>
> This is the first time we do this kind of task, so define a document that helps another AI agent or a new developer repeat it later.

## 🎯 Objective
Document the exact steps needed to extend `tiny-internal-services` and `tiny-internal-services-mocks` with the three internal trigger subscription endpoints (POST/GET/DELETE) that now exist in Megazord Events, ensuring the SDK clients, mocks, and tests stay aligned with the new single-event (`eventName`) contract and duplicate-trigger validation rules.

### ⚠️ Key Considerations
- Internal trigger endpoints mirror the existing external ones but **must not** use admin/permission middleware; clients hit `/internal/v1/events/robots/:robotId/subscriptions/triggers`.
- Trigger subscriptions now accept exactly **one** `eventName` per request; DTOs must enforce this and the service layer already rejects duplicate active subscriptions.
- `tiny-internal-services` exports are used widely (Sensara adaptor, dashboards, CLIs); changes must be backward compatible for non-trigger subscriptions.
- `tiny-internal-services-mocks` relies on `nock` + `ts-mockito`; its helpers should cover success and failure paths (400 invalid payload, 404 unknown event, 409 duplicate).
- Remember to regenerate docs / types (`yarn build && yarn docs`) before publishing new package versions, otherwise downstream tooling will miss the new API surface.

## 🔄 Implementation Plan
[Doesn't require running tests inside this doc; rely on repo-specific instructions when touching the code.]

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements  
  - **Outcome**: Confirmed payload (`{ eventName: string }`), endpoints, duplicate validation, and shared logic from `251107-PROD438-internal-trigger-endpoint.md`.
- [x] Define scope and edge cases  
  - **Outcome**: Documented POST (201), GET (single `SubscriptionDomain`), DELETE (204) flows plus error codes (400 invalid payload, 404 unknown event, 409 duplicate) to mirror in mocks.

### Phase 2: Implementation (File/Code Structure)
> Describe the proposed file/directory structure, including the purpose of each key component. Remember use status markers like ✅ (Implemented), 🚧 (To-Do), 🔄 (In Progress).

```
tiny-internal-services/
├── lib/model/events/CreateSubscriptionDto.ts          # ✅ EXISTING - array payload for generic subscriptions
├── lib/model/events/CreateTriggerSubscriptionDto.ts   # ✅ IMPLEMENTED - single-event DTO for triggers
├── lib/services/EventService.ts                       # ✅ IMPLEMENTED - POST/GET/DELETE helpers for internal triggers
└── lib/index.ts                                       # ✅ IMPLEMENTED - exports new DTO + EventService helpers

tiny-internal-services-mocks/
├── lib/mocks/EventServiceMocks.ts                     # ✅ IMPLEMENTED - trigger-specific mock + nock helpers
├── test/EventServiceMocksTest.ts                      # ✅ IMPLEMENTED - covers trigger endpoints
└── lib/index.ts                                       # ✅ IMPLEMENTED - re-export through existing barrel
```

### Phase 3: Detailed Implementation Steps

#### Step 3.1: tiny-internal-services – Models & Contracts
- ✅ Added `lib/model/events/CreateTriggerSubscriptionDto.ts` (single `eventName: string` payload) and exported it via `lib/index.ts`.
- ✅ Left `CreateSubscriptionDto` untouched for non-trigger flows; doc comment in the new DTO clarifies its scope.
- ✅ Included inline usage notes (single-event contract, shared Megazord endpoint) for quick reference.

```ts
/**
 * POST /internal/v1/events/robots/{robotId}/subscriptions/triggers
 * Request: { eventName: 'ACTIVITY' }
 * Response: SubscriptionDomain (single event in subscriptionDetails)
 */
```

#### Step 3.2: tiny-internal-services – EventService client additions
- ✅ Added `postTriggerSubscription`, `getTriggerSubscription`, `deleteTriggerSubscription` methods.
- ✅ Targeted `/internal/v1/events/robots/${robotId}/subscriptions/triggers` (POST returns 201 & `SubscriptionDomain`, GET returns a single `SubscriptionDomain`, DELETE returns 204).
- ✅ Reused `request<T>` for telemetry/error handling; error messages mention “trigger subscription” to distinguish them from generic ones.

#### Step 3.3: tiny-internal-services – Exports & docs
- ✅ `lib/index.ts` now exports `CreateTriggerSubscriptionDto` alongside the existing DTOs/services.
- 🔄 Remember to run `yarn build && yarn docs` before publishing so downstream consumers get updated typings/docs (not yet executed in this iteration).

#### Step 3.4: tiny-internal-services-mocks – EventServiceMocks updates
- ✅ Imported the new DTO and added `createTriggerSubscriptionFromDto` helper to build deterministic `SubscriptionDomain` responses.
- ✅ Added `postTriggerSubscription`, `nockPostTriggerSubscription` (201), `getTriggerSubscription`, `nockGetTriggerSubscription` (200 with single subscription), plus delete counterparts hitting the `/triggers` routes.
- 🔮 Optionally extend later with explicit 400/404/409 mock helpers once consumer needs arise.

#### Step 3.5: tiny-internal-services-mocks – Tests & publishing checklist
- ✅ Added trigger-path coverage to `test/EventServiceMocksTest.ts` (POST, GET, DELETE happy paths).
- ✅ Barrel exports already cover the new helpers via `export *` from `EventServiceMocks`.
- 🔄 Still need to bump package versions and regenerate docs once Yarn test runs cleanly.

## 📊 Summary of Results

### ✅ Completed Achievements
- Captured trigger endpoint requirements (single-event DTO, duplicate prevention, HTTP status expectations).
- Implemented `CreateTriggerSubscriptionDto`, EventService helpers, mocks, and tests reflecting POST 201 / GET single `SubscriptionDomain` behavior.
- Documented the concrete implementation steps so future contributors can replicate/extend the pattern.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Yarn 3.8.7 currently fails locally with “fastqueue concurrency must be greater than 1”, preventing `yarn test` from running inside `tiny-internal-services-mocks`. Resolve before publishing to ensure mocks compile + tests pass.

### 🔮 Future Improvements (Optional)
- [ ] Automate doc regeneration + package publish via CI to avoid manual “yarn docs” steps.
- [ ] Provide a higher-level helper in `tiny-internal-services` that converts friendly event names into IDs to reduce boilerplate for new consumers.

---
**Last updated**: 2025-11-11
---
