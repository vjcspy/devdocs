# 📋 [PROD591: 2025-11-12] - Add 2 new events to tinybots events service

## User Requirements
> Stakeholder: Sensara confirmed the upstream notification IDs are `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`. We want to expose them internally as `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG`, so please add the events to megazord-events and update the sensara adaptor mappings accordingly.

## 🎯 Objective
Model the new Sensara “longer in bed” alarms (`LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG`) as first-class Tinybots events, ensuring the upstream notifications (`ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`) translate cleanly through megazord-events and the sensara-adaptor pipeline.

### ⚠️ Key Considerations
- `TinybotsEvent` enum is owned by `tiny-internal-services` (v1.20.0); both megazord-events and sensara-adaptor import it, so the new `LONGER_IN_BED_SHORT|LONG` values require a package update plus dependency bumps.
- Event schema JSONs in `megazord-events/schemas/events/*.json` are generated via `schemas/gen.ts`; new entries need generator updates plus per-event overrides (level/trigger) before running `ts-node schemas/gen.ts`.
- `SensaraEventsAdaptorService` only forwards whitelisted events to Sensara; without extending its `endpointsMapping`, registration requests for the new events will be ignored.
- `sensara-adaptor/src/jobs/SensaraEventsJob.ts` currently doesn’t translate `SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED` or `ST_SLEEPING_AWAKE_LARGE_DELAY`; these must now become `TinybotsEvent.LONGER_IN_BED_SHORT` and `TinybotsEvent.LONGER_IN_BED_LONG` respectively. Confirm severity and whether both notifications are live in production streams.
- Need feedback on alert `level` differences (e.g., LONG might warrant a higher level/trigger) and on whether `mustForwardEventName` should be `true` for Sensara ingestion.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements
  - **Outcome**: Captured stakeholder ask (add & route two Sensara bed-duration events) and developer concerns (where to modify code, which files matter across repos).
- [x] Define scope and edge cases
  - **Outcome**: Need to decide severity levels, confirm upstream notification names, ensure both megazord-events and sensara-adaptor reference the same enum version, and verify whether short/long alarms require throttling or duplicate suppression.

### Phase 2: Implementation (File/Code Structure)
```
tiny-internal-services/
├── lib/model/events/TinybotsEvent.ts             # 🚧 TODO - append LONGER_IN_BED_* + rebuild/dist/docs
├── lib/model/sensara/SensaraNotificationDto.ts   # 🚧 TODO - expose ST_SLEEPING_AWAKE_DELAYED/LARGE_DELAY if absent
├── package.json                                  # ✅ publish new tag (e.g. 1.21.0) once enums land

megazord-events/
├── schemas/gen.ts                                # 🚧 TODO - include new constants & custom level/trigger if needed
├── schemas/events/longer_in_bed_short.json       # 🚧 TODO - generated schema for SHORT event
├── schemas/events/longer_in_bed_long.json        # 🚧 TODO - generated schema for LONG event
├── src/services/internal/SensaraEventsAdaptorService.ts # 🚧 TODO - map new events to ACTIVITY endpoint (forward name=true)
├── package.json                                  # ✅ bump tiny-internal-services dependency version after publish

sensara-adaptor/
├── src/jobs/SensaraEventsJob.ts                  # 🚧 TODO - translate Sensara notifications into new Tinybots events
├── src/eventsource/SensaraEventSource.ts         # ✅ no change, ensures stream handling
├── test/jobs/SensaraEventsJob.test.ts            # 🚧 TODO - cover new mapping once implemented
```

### Phase 3: Detailed Implementation Steps
1. **Extend shared enums** (`tiny-internal-services`):
   - Add `LONGER_IN_BED_SHORT` and `LONGER_IN_BED_LONG` to `lib/model/events/TinybotsEvent.ts` and regenerate `dist/*.d.ts` plus docs.
   - Ensure `SensaraNotificationType` includes `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`; export if missing (they are now the confirmed upstream identifiers).
   - Publish a new tag (e.g., `1.21.0`) so downstream services can consume the updated enum.
2. **Update megazord-events schemas**:
   - Mirror the new enum names (`LONGER_IN_BED_SHORT|LONG`) inside `schemas/gen.ts` (so JSON files exist) and override `CustomConfigs` if `SHORT` vs `LONG` should have different `level` or `hasTrigger` values (e.g., LONG => `level: 5`, `hasTrigger: true`).
   - Run `npx ts-node --transpile-only schemas/gen.ts` to regenerate `schemas/events/*.json`, validating that `longer_in_bed_*.json` contain the desired metadata.
   - Review `SensaraEventsAdaptorService.endpointsMapping`: map each new event to the `/internal/v1/sensara/residents/activity/register` endpoint (or whichever endpoint Ops confirms) with `mustForwardEventName: true` so Sensara knows which alarm to toggle.
3. **Teach sensara-adaptor about the new notifications**:
   - In `src/jobs/SensaraEventsJob.ts`, map `SensaraNotificationType.ST_SLEEPING_AWAKE_DELAYED` → `TinybotsEvent.LONGER_IN_BED_SHORT` and `ST_SLEEPING_AWAKE_LARGE_DELAY` → `TinybotsEvent.LONGER_IN_BED_LONG` inside `convertEvent`.
   - Decide alert `level` payloads: pass a higher priority (e.g., `level = 5`) for LONG while reusing default for SHORT; document rationale.
   - Extend job/unit tests (e.g., `test/jobs/SensaraEventsJob.test.ts`) to assert the new conversions and guard against regressions.
   - Deploy sensara-adaptor after megazord-events is using the updated enum version to avoid runtime `undefined` event names.
4. **Operational follow-up**:
   - Coordinate rollout so resident configurations requesting these alarms call the updated `register` API; otherwise registration attempts will log “no sensara adaptor endpoint configured”.
   - Communicate with CareOps/Sensara to confirm when these alarms go live and whether additional notification IDs need mapping.

## 📊 Summary of Results

### ✅ Completed Achievements
- Captured stakeholder + developer requirements, including the confirmed upstream notification IDs and desired Tinybots translations, and translated them into actionable steps.
- Identified all relevant repositories (`megazord-events`, `sensara-adaptor`, `tiny-internal-services`) and the specific files to touch.
- Produced an implementation roadmap covering enum updates, schema generation, adaptor mappings, and notification conversions.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Need confirmation on desired `level`/trigger values for SHORT vs LONG alarms; defaulting both to 10 may under-prioritize the LONG case.
- [x] Upstream Sensara payload names are confirmed as `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`; translations must stay aligned with `LONGER_IN_BED_SHORT`/`LONG`.

### 🔮 Future Improvements (Optional)
- [ ] Automate event schema generation from `tiny-internal-services` enums to avoid duplicate sources of truth.
- [ ] Add integration tests that replay Sensara notification samples to validate end-to-end forwarding.

---
**Last updated**: 2025-11-12
---
