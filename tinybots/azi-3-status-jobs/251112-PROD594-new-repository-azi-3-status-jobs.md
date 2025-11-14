# 📋 [PROD594: 2025-11-12] - New Repository: azi-3-status-jobs

## User Requirements
> - Run a toilet-activity experiment for a single, pre-selected user by extending `azi-3` status checks (speed over generality).
> - Add `NO_TOILET_ACTIVITY_ALARM` to the Megazord events schema so downstream services can subscribe/trigger on it.
> - Between time A and time B each day: at time A subscribe to `TOILET_ACTIVITY` events for that user, monitor the next two hours, and reschedule checks every time a new visit is observed (bounded by time B).
> - If no `TOILET_ACTIVITY` arrives within a two-hour window, emit `NO_TOILET_ACTIVITY_ALARM`, log it for the script node, and immediately start monitoring the next two-hour slice (still capped by time B).
> - Implement the logic inside `azi-3-status-check` or the new `azi-3-status-jobs` repository (this plan covers the new repo path).

## 🎯 Objective
> Build and document a focused `azi-3-status-jobs` service plus ancillary schema changes that can watch toilet activity for one resident, post `NO_TOILET_ACTIVITY_ALARM` events when the resident skips two hours, and record every window in the shared `status_check*` tables so scripts can branch on the result.

### ⚠️ Key Considerations
- Keep scope to one resident/robot but structure config (per-robot windows, timezone) so we can add more without rewriting.
- `TOILET_ACTIVITY` originates from Sensara (`sensara-adaptor` ➜ `megazord-events`); alarms must flow back through Megazord so triggers/dashboards continue to work.
- All monitoring state must live in `status_check`, `status_check_poller`, and `status_check_record` so script nodes can read outcomes; no ephemeral memory.
- Time A/B are resident-timezone driven; clamp final windows to B and handle DST jumps (23/25-hour days).
- SQS delivery is at-least-once, so the worker must deduplicate, replay missed events via `EventService.getEvents`, and avoid emitting duplicate alarms for the same gap.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: `azi-3-status-jobs` must 1) add `NO_TOILET_ACTIVITY_ALARM` to Megazord + `tiny-internal-services`, 2) seed a toilet status-check template/description so Micro-Manager scripts can reference it, 3) create/destroy a Megazord subscription at the configured daily time A, 4) maintain rolling two-hour windows (capped by B) by combining SQS pushes with `incoming_event` history, and 5) emit/log `NO_TOILET_ACTIVITY_ALARM` both in Event Service and `status_check_record`.
- [ ] Define scope and edge cases
  - **Outcome**: Handle DST jumps, windows that cross midnight, robot offline periods, duplicate/out-of-order SQS deliveries, service restarts mid-window, updates to the A/B window without redeploys, ensuring alarms are only sent once per missed window, and guaranteeing subscription cleanup even if the worker crashes.

### Phase 2: Implementation (File/Code Structure)
> Describe the proposed file/directory structure, including the purpose of each key component. Remember use status markers like ✅ (Implemented), 🚧 (To-Do), 🔄 (In Progress).

```
azi-3-status-jobs/
├── package.json
├── tsconfig.json
├── src/
│   ├── cmd/
│   │   └── main.ts                          # 🚧 TODO - Bootstraps TinyDatabaseAppAuthenticated + background jobs
│   ├── config/
│   │   └── default.json                     # 🚧 TODO - DB/SQS/EventService addresses + per-robot monitoring windows
│   ├── domain/
│   │   └── StatusCheck.ts                   # 🚧 TODO - Domain models for status_check/_record/_poller rows
│   ├── infrastructure/
│   │   ├── MegazordEventClient.ts           # 🚧 TODO - Wraps tiny-internal EventService for subscribe/post/get
│   │   ├── StatusQueueConsumer.ts           # 🚧 TODO - ContextSQS consumer with subscription/event filters
│   │   └── Clock.ts                         # 🚧 TODO - Timezone-aware clock abstraction for deterministic tests
│   ├── repositories/
│   │   ├── StatusCheckRepository.ts         # 🚧 TODO - SQL for status_check, *_record, *_poller, subscriptions
│   │   └── StatusTemplateRepository.ts      # 🚧 TODO - CRUD + seeding helpers for template/description tables
│   ├── services/
│   │   ├── StatusCheckScheduler.ts          # 🚧 TODO - Creates daily checks, ensures subscription lifecycle
│   │   ├── ToiletActivityWindowService.ts   # 🚧 TODO - State machine that advances two-hour windows
│   │   ├── AlarmEmitter.ts                  # 🚧 TODO - Posts NO_TOILET alarms + records DB side-effects
│   │   └── MonitorCoordinator.ts            # 🚧 TODO - Glues repositories, consumer, and services together
│   ├── jobs/
│   │   ├── MonitorWorker.ts                 # 🚧 TODO - Back-off aware loop that polls queue + reconciles DB
│   │   └── SeedStatusTemplateJob.ts         # 🚧 TODO - Ensures template/description rows exist on boot
│   ├── types/
│   │   └── index.ts                         # 🚧 TODO - Shared DTOs/config typings
│   └── index.ts                             # 🚧 TODO - Export app for tests / CLI
├── scripts/
│   └── seed-status-check-template.ts        # 🚧 TODO - CLI to backfill template + status_check_description
└── test/
    ├── helpers/factories.ts                 # 🚧 TODO - Builders for db rows + fake SQS payloads
    └── services/ToiletActivityWindowService.test.ts  # 🚧 TODO - Unit tests for window math & transitions
```

**Supporting repositories**
- `megazord-events/schemas/gen.ts` & `megazord-events/schemas/events/no_toilet_activity_alarm.json` # 🚧 TODO - Add new event constant + generated schema.
- `tiny-internal-services/lib/model/events/TinybotsEvent.ts` # 🚧 TODO - Export `NO_TOILET_ACTIVITY_ALARM` for sharing across services.
- `devdocs/tinybots/megazord-events/OVERVIEW.md` & `devdocs/tinybots/sensara-adaptor/OVERVIEW.md` # 🚧 TODO - Mention the new alarm and its producer.
- `typ-e/src/main/resources/db/migration` (new SQL) # 🚧 TODO - Seed `status_check_description` + template rows for the toilet-alarm script node.

### Phase 3: Detailed Implementation Steps
1. **Extend Megazord + tiny-internal event vocabulary**
   - Add `NO_TOILET_ACTIVITY_ALARM` to `megazord-events/schemas/gen.ts`, regenerate JSON under `schemas/events/`, and bump fixtures/tests that read the schemas.
   - Mirror the constant in `tiny-internal-services/lib/model/events/TinybotsEvent.ts` so `azi-3-status-jobs` and scripts can use the same enum without string literals.
   - Document the new event (purpose, producer service, severity/level, trigger availability) inside `devdocs/tinybots/megazord-events/OVERVIEW.md` for discoverability.
2. **Seed toilet status-check template metadata**
   - Create a SQL migration (or dedicated seed script inside `typ-e`) that inserts:
     - `status_check_description` row (e.g., `TOILET_ACTIVITY_GAP`).
     - `status_check_template` referencing the description with `past_event_dependencies = '[]'`, `future_event_dependencies = '[{"event":"TOILET_ACTIVITY","maxGapMinutes":120}]'`, `poll_until_end = TRUE`.
     - `status_check_template_detail` entries representing phases: `SETUP` (subscription created), `MONITORING` (window active), `ALARM` (gap detected), `COMPLETE`.
   - Provide a seed CLI (`scripts/seed-status-check-template.ts`) that can be rerun safely (UPSERT by description name/version) so environments stay consistent.
3. **Scaffold the `azi-3-status-jobs` application**
   - Base it on `TinyDatabaseAppAuthenticated` for MySQL access + shared middlewares; wire Awilix for DI similar to `m-o-triggers`.
   - Configuration (`config/default*.json`) should capture DB creds, Megazord Event Service URL, SQS config (`statusQueue.address`, `pollIntervalMs`), job intervals, and the pilot `robotId`/timezone/time A/B.
   - Register modules: `ContextSQS` producer/consumer, `MegazordEventClient` (wrapping `tiny-internal` `EventService`), repositories, services, Cron scheduler, and health routes.
4. **Implement monitoring + subscription flow**
   - `StatusCheckScheduler`:
     - On boot and every midnight (based on configured timezone) create or reactivate the day's `status_check` + `status_check_poller` record for the pilot robot.
     - At time A, call `MegazordEventClient.postSubscription` with `{ eventNames: ['TOILET_ACTIVITY'], until: timeB }`, store `subscription_id` on the `status_check`, and write a `status_check_record` entry noting the subscription creation.
     - Ensure `status_check_poller` row tracks `since` (window start) and `until` (min(start+120m, timeB)) plus `lock_id` for workers.
   - `StatusQueueConsumer`:
     - Poll `statusQueue.address`, parse messages via `ContextSQS.parseMessage`, and filter by both `subscriptionId` and `robotId`.
     - Forward relevant messages (payload + metadata) into `ToiletActivityWindowService`.
     - Expose back-pressure friendly `async iterator` so we can gracefully stop on deploys.
   - Recovery path: when (re)starting a worker or regaining DB locks, call `MegazordEventClient.getEvents(robotId, { eventName: 'TOILET_ACTIVITY', createdSince: poller.since })` to replay missed visits before relying on SQS.
5. **Advance windows + emit alarms**
   - `ToiletActivityWindowService` logic:
     ```
     currentWindow = [poller.since, min(poller.since + 120 min, timeB)]
     if message.timestamp within window:
        insert status_check_record(eventName='TOILET_ACTIVITY', incoming_event_id=payload.sourceEventId, result='EVENT_OBSERVED')
        update poller.since = message.timestamp
        poller.until = min(message.timestamp + 120 min, timeB)
        if poller.since >= timeB -> finalize success (set status_check.result='COMPLETED', delete poller, unsubscribe)
     ```
   - Cron-driven gap detection:
     - Every minute, select `status_check_poller` rows where `lock_id is null and until <= now` and transition them.
     - Emit alarm by calling `MegazordEventClient.postEvent(robotId, { providerName: 'azi-3-status-jobs', eventName: TinybotsEvent.NO_TOILET_ACTIVITY_ALARM, level: config.alarmLevel, referenceId: \`\${statusCheckId}-\${poller.since.toISOString()}\` })`.
     - Log DB side-effects: insert `status_check_record` row referencing the new incoming event ID (returned by Megazord) with `result='ALARM_SENT'`, reset `poller.since = now`, `poller.until = min(now + 120 min, timeB)`, and, if `now >= timeB`, mark `result='ALARM_DAY_COMPLETE'` and unsubscribe.
     - Deduplicate by checking whether a record for the same window (same `status_check_id` + `phase` + `order`) already exists before posting.
6. **Subscription teardown + cleanup**
   - When `status_check.result` is set (success or day-complete), call `deleteSubscription` for the stored `subscription_id`, remove `status_check_poller`, and finalize `status_check` row (`result = 'SUCCESS' | 'ALARM_RAISED'`, `updated_at = NOW()`).
   - Add compensating job that scans for abandoned subscriptions (e.g., `result is null AND planned_at < NOW() - 1 day`) and cleans them up.
7. **Observability & verification**
   - Emit structured logs (`robotId`, `statusCheckId`, `windowSince`, `windowUntil`, `eventTimestamp`, `alarmReferenceId`), plus counters (`azi3.toilet.window.success`, `azi3.toilet.window.alarm`) via `tiny-backend-tools` metrics hooks if available.
   - Provide manual validation playbook: 1) call seed script, 2) start worker with fake SQS payloads, 3) confirm `status_check_record` rows and posted events via `GET /internal/v1/events/...`.
   - Even though tests are not required to run now, outline unit tests for `ToiletActivityWindowService` (window math, DST handling) and integration tests using `tiny-testing` DB fixtures + mocked SQS.

## 📊 Summary of Results

### ✅ Completed Achievements
- Consolidated the stakeholder request, constraints, and prior-art research into a single actionable design using the agent template.
- Mapped every required code/data touch-point (Megazord schema, tiny-internal enum, Typ-E seed, new repo skeleton) for introducing `NO_TOILET_ACTIVITY_ALARM`.
- Defined the monitoring algorithm (subscription lifecycle, window advancement, alarm emission, recovery, persistence) so implementation can start immediately.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Need the exact pilot `robotId`/resident info, timezone, and daily start/end times (A/B) to populate config + template rows.
- [ ] Confirm desired `level` and `hasTrigger` flags for `NO_TOILET_ACTIVITY_ALARM` so dashboards, triggers, and escalation rules behave correctly.
- [ ] Agree on the set of `status_check.result` values (`SUCCESS`, `ALARM_RAISED`, etc.) expected by Micro-Manager when it resumes the script.

### 🔮 Future Improvements (Optional)
- [ ] Generalize the monitoring config (e.g., store per-robot windows in DB) so additional residents can be onboarded without redeploying.
- [ ] Surface alarms to dashboards/Slack and add Datadog metrics for faster experimentation feedback loops.

---
**Last updated**: 2025-11-12
---
