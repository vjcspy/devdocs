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

- `TinybotsEvent` enum is owned by `tiny-internal-services` (v1.20.0); both megazord-events and sensara-adaptor import it, so the new `LONGER_IN_BED_SHORT|LONG` values require a package update plus dependency bumps.
- Event schema JSONs in `megazord-events/schemas/events/*.json` are generated via `schemas/gen.ts`; new entries need generator updates plus per-event overrides (level/trigger) before running `ts-node schemas/gen.ts`.
- `SensaraEventsAdaptorService` only forwards whitelisted events to Sensara; without extending its `endpointsMapping`, registration requests for the new events will be ignored.
- `sensara-adaptor/src/jobs/SensaraEventsJob.ts` currently doesn't translate `SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED` or `ST_SLEEPING_AWAKE_LARGE_DELAY`; these must now become `TinybotsEvent.LONGER_IN_BED_SHORT` and `TinybotsEvent.LONGER_IN_BED_LONG` respectively.
- **✅ CONFIRMED**: Both events use `level: 10`, `hasTrigger: true`, and `mustForwardEventName: true` for Sensara registration.
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
├── lib/model/events/TinybotsEvent.ts             # 🚧 TODO - append LONGER_IN_BED_SHORT + LONGER_IN_BED_LONG
├── lib/model/sensara/SensaraNotificationDto.ts   # ✅ ALREADY HAS - ST_SLEEPING_AWAKE_DELAYED/LARGE_DELAY already exist
├── package.json                                  # ✅ publish new tag (e.g. 1.21.0) once enums land

megazord-events/
├── schemas/gen.ts                                # 🚧 TODO - add LONGER_IN_BED_SHORT/LONG with level:10, hasTrigger:true
├── schemas/events/longer_in_bed_short.json       # 🚧 TODO - generated schema (level:10, hasTrigger:true)
├── schemas/events/longer_in_bed_long.json        # 🚧 TODO - generated schema (level:10, hasTrigger:true)
├── src/services/internal/SensaraEventsAdaptorService.ts # 🚧 TODO - map both events to ACTIVITY_ENDPOINT with mustForwardEventName:true
├── package.json                                  # ✅ bump tiny-internal-services dependency version after publish

sensara-adaptor/
├── src/jobs/SensaraEventsJob.ts                  # 🚧 TODO - translate Sensara notifications into new Tinybots events
├── src/eventsource/SensaraEventSource.ts         # ✅ no change, ensures stream handling
├── test/jobs/SensaraEventsJob.test.ts            # 🚧 TODO - implement tests for new mapping
```

### Phase 3: Detailed Implementation Steps

1. **Extend shared enums** (`tiny-internal-services`):
   - Add `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` to `lib/model/events/TinybotsEvent.ts` and regenerate `dist/*.d.ts` plus docs.
   - ✅ **VERIFIED**: `SensaraNotificationType` already includes `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY` in `lib/model/sensara/SensaraNotificationDto.ts` - no changes needed.
   - Rebuild the package and publish a new tag (e.g., `1.21.0`) so downstream services can consume the updated enum.

2. **Update megazord-events schemas**:
   - Add `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` to the local `TinybotsEvent` object in `schemas/gen.ts`.
   - Add both events to `CustomConfigs` with configuration: `level: 10`, `hasTrigger: true` for both SHORT and LONG events.
   - Run `npx ts-node --transpile-only schemas/gen.ts` to generate `schemas/events/longer_in_bed_short.json` and `longer_in_bed_long.json`.
   - Update `SensaraEventsAdaptorService.endpointsMapping`: map both new events to `ACTIVITY_ENDPOINT` (`/internal/v1/sensara/residents/activity/register`) with `mustForwardEventName: true`:
     ```typescript
     [TinybotsEvent.LONGER_IN_BED_SHORT]: [ACTIVITY_ENDPOINT, true],
     [TinybotsEvent.LONGER_IN_BED_LONG]: [ACTIVITY_ENDPOINT, true]
     ```
   - Update `package.json` to bump `tiny-internal-services` dependency to v1.21.0.

3. **Teach sensara-adaptor about the new notifications**:
   - Update `package.json` to bump `tiny-internal-services` dependency to v1.21.0.
   - In `src/jobs/SensaraEventsJob.ts`, add two new cases in the `convertEvent` switch statement:
     ```typescript
     case SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED:
       return this._createEvent(event, TinybotsEvent.LONGER_IN_BED_SHORT)
     case SensaraNotificationType.ST_SLEEPING_AWAKE_LARGE_DELAY:
       return this._createEvent(event, TinybotsEvent.LONGER_IN_BED_LONG)
     ```
   - Both events will use default `level: 10` (no custom level parameter needed in `_createEvent`).
   - **Implement tests** in `test/jobs/SensaraEventsJob.test.ts` to cover the new notification type conversions and validate the correct TinybotsEvent mapping.

4. **Testing & Validation**:
   - Implement unit tests for all changes (schema generation validation, endpoint mapping, event conversion).
   - Tests should be created but do not need to be run during implementation phase.
   - Verify that generated JSON schemas have correct `level: 10` and `hasTrigger: true` values.

## 📊 Summary of Results

### ✅ Completed Achievements

- Captured stakeholder + developer requirements, including the confirmed upstream notification IDs and desired Tinybots translations, and translated them into actionable steps.
- Identified all relevant repositories (`megazord-events`, `sensara-adaptor`, `tiny-internal-services`) and the specific files to touch.
- Produced an implementation roadmap covering enum updates, schema generation, adaptor mappings, and notification conversions.
- Confirmed all configuration values: `level: 10`, `hasTrigger: true`, `mustForwardEventName: true`.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues

- [x] ✅ **RESOLVED**: Both events confirmed to use `level: 10`, `hasTrigger: true`.
- [x] ✅ **CONFIRMED**: Upstream Sensara payload names are `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`; they map to `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` respectively.
- [x] ✅ **CONFIRMED**: Both events use `mustForwardEventName: true` with `ACTIVITY_ENDPOINT`.
- [x] ✅ **VERIFIED**: `SensaraNotificationType` enum already contains required notification types.

### 🔮 Future Improvements

- [ ] Automate event schema generation from `tiny-internal-services` enums to avoid duplicate sources of truth.
- [ ] Add integration tests that replay Sensara notification samples to validate end-to-end forwarding.

---

**Last updated**: 2025-11-16 (Plan reviewed and confirmed with stakeholder)

---
