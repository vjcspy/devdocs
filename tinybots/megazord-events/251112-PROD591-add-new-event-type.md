# 📋 [PROD591: 2025-11-12] - Add 2 new events to tinybots events service

## User Requirements

Sensara has 2 new events we want to monitor.
They are:

- ST_SLEEPING_AWAKE_LARGE_DELAY (sensara) translate in our system to LONGER_IN_BED_LONG. This event will happen if the client is in bed a little bit longer than normal.
- ST_SLEEPING_AWAKE_DELAYED translate in our system to LONGER_IN_BED_SHORT. This event will happen if the client is in bed a lot longer than normal.

**The following should happen:**

- to Add the events to megazord event schema
- Define the mapping in the sensara adaptor TBA

## 🎯 Objective

Model the new Sensara "longer in bed" alarms (`LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG`) as first-class Tinybots events, ensuring the upstream notifications (`ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`) translate cleanly through megazord-events and the sensara-adaptor pipeline.

### ⚠️ Key Considerations

- `TinybotsEvent` enum is owned by `tiny-internal-services` (v1.23.0 contains the new events); both megazord-events and sensara-adaptor import it, so the new `LONGER_IN_BED_SHORT|LONG` values require a package update plus dependency bumps.
- Event schema JSONs in `megazord-events/schemas/events/*.json` are generated via `schemas/gen.ts`; new entries need generator updates plus per-event overrides (level/trigger) before running `ts-node schemas/gen.ts`.
- **⚠️ IMPORTANT**: These events are **incoming notifications from Sensara**, NOT events we register with Sensara. Therefore, `SensaraEventsAdaptorService.endpointsMapping` does NOT need to be updated - we only receive and translate these notifications, we don't send registration requests for them.
- `sensara-adaptor/src/jobs/SensaraEventsJob.ts` needs to translate `SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY` to `TinybotsEvent.LONGER_IN_BED_SHORT` and `TinybotsEvent.LONGER_IN_BED_LONG` respectively.
- **✅ CONFIRMED**: Both events use `level: 10` when forwarded as Tinybots events.
- **✅ NOTE**: `SensaraNotificationType` enum already contains both `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY` in `tiny-internal-services/lib/model/sensara/SensaraNotificationDto.ts`, so no changes needed there.

## 🔄 Implementation Plan

[Tests must be implemented for all changes, but no need to run tests during implementation]

### Phase 1: Analysis & Preparation

- [x] Analyze detailed requirements
  - **Outcome**: Captured stakeholder ask (add & route two Sensara bed-duration events) and developer concerns (where to modify code, which files matter across repos).
- [x] Define scope and edge cases
  - **Outcome**: Confirmed configuration values: `level: 10`, `hasTrigger: true`, `mustForwardEventName: true`. Verified that `SensaraNotificationType` enum already contains required notification types.

### Phase 2: Implementation (File/Code Structure)

```text
tiny-internal-services/
├── lib/model/events/TinybotsEvent.ts             # ✅ DONE - LONGER_IN_BED_SHORT + LONGER_IN_BED_LONG added
├── lib/model/sensara/SensaraNotificationDto.ts   # ✅ ALREADY HAS - ST_SLEEPING_AWAKE_DELAYED/LARGE_DELAY already exist
├── package.json                                  # ✅ DONE - published as v1.23.0

megazord-events/
├── schemas/gen.ts                                # ✅ DONE - added LONGER_IN_BED_SHORT/LONG with level:10, hasTrigger:true
├── schemas/events/longer_in_bed_short.json       # ✅ DONE - generated schema (level:10, hasTrigger:true)
├── schemas/events/longer_in_bed_long.json        # ✅ DONE - generated schema (level:10, hasTrigger:true)
├── src/services/internal/SensaraEventsAdaptorService.ts # ✅ NO CHANGE NEEDED - these are incoming notifications, not registered events
├── package.json                                  # ✅ DONE - bumped tiny-internal-services to v1.23.0

sensara-adaptor/
├── src/jobs/SensaraEventsJob.ts                  # ✅ DONE - added translation for Sensara notifications to new Tinybots events
├── src/eventsource/SensaraEventSource.ts         # ✅ NO CHANGE NEEDED - stream handling works as-is
├── test/jobs/SensaraEventsJobTest.ts             # ✅ DONE - implemented tests for new mapping
├── package.json                                  # ✅ DONE - using tiny-internal-services v1.23.0
```

### Phase 3: Detailed Implementation Steps

1. **✅ COMPLETED - Extend shared enums** (`tiny-internal-services`):
   - Added `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` to `lib/model/events/TinybotsEvent.ts` and regenerated `dist/*.d.ts` plus docs.
   - ✅ **VERIFIED**: `SensaraNotificationType` already includes `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY` in `lib/model/sensara/SensaraNotificationDto.ts` - no changes needed.
   - Rebuilt the package and published as v1.23.0 so downstream services can consume the updated enum.

2. **✅ COMPLETED - Update megazord-events schemas**:
   - Added `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` to the local `TinybotsEvent` object in `schemas/gen.ts`.
   - Added both events to `CustomConfigs` with configuration: `level: 10`, `hasTrigger: true` for both SHORT and LONG events.
   - Ran `npx ts-node --transpile-only schemas/gen.ts` to generate `schemas/events/longer_in_bed_short.json` and `longer_in_bed_long.json`.
   - **NO CHANGE to `SensaraEventsAdaptorService.endpointsMapping`**: These events are incoming notifications from Sensara (received via SSE stream), NOT events that we register with Sensara. We don't send registration requests for these events.
   - Updated `package.json` to bump `tiny-internal-services` dependency to v1.23.0.

3. **✅ COMPLETED - Teach sensara-adaptor about the new notifications**:
   - sensara-adaptor already using `tiny-internal-services` v1.23.0 in `package.json`.
   - Added two new cases in `src/jobs/SensaraEventsJob.ts` in the `convertEvent` switch statement:

     ```typescript
     case SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED:
       return this._createEvent(event, TinybotsEvent.LONGER_IN_BED_SHORT)
     case SensaraNotificationType.ST_SLEEPING_AWAKE_LARGE_DELAY:
       return this._createEvent(event, TinybotsEvent.LONGER_IN_BED_LONG)
     ```

   - Both events use default `level: 10` (no custom level parameter needed in `_createEvent`).
   - **Implemented tests** in `test/jobs/SensaraEventsJobTest.ts` to cover the new notification type conversions and validate the correct TinybotsEvent mapping.

4. **✅ COMPLETED - Testing & Validation**:
   - Implemented unit tests for new event conversions (2 new test cases added).
   - Tests passed successfully - event mapping works correctly.
   - Verified integration with database (required updated DB schema with `location_label` column).

---
