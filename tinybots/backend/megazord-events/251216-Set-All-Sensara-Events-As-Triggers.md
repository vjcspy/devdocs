# 📋 251216 - Set All Sensara Events As Triggers

## References

- [Global Overview](/Users/kai/work/tinybots/devdocs/tinybots/OVERVIEW.md)
- [megazord-events OVERVIEW](/Users/kai/work/tinybots/devdocs/tinybots/backend/megazord-events/OVERVIEW.md)
- [sensara-adaptor OVERVIEW](/Users/kai/work/tinybots/devdocs/tinybots/backend/sensara-adaptor/OVERVIEW.md)
- [TinybotsEvent enum](/Users/kai/work/tinybots/tinybots/backend/tiny-internal-services/lib/model/events/TinybotsEvent.ts)
- [Event Schema Generator](/Users/kai/work/tinybots/tinybots/backend/megazord-events/schemas/gen.ts)
- [Event Schemas Directory](/Users/kai/work/tinybots/tinybots/backend/megazord-events/schemas/events/)

## User Requirements

> For Sensara, there are certain events that can be used as triggers. Set all events to be a possible trigger in the config

## 🎯 Objective

Enable all Sensara-related events (especially location-based `IN_*` events) to be used as triggers in the TinyBots automation system by updating the `hasTrigger` flag in event schema configuration.

### ⚠️ Key Considerations

1. **Sensara Location Events**: Sensara provides location telemetry that maps to TinyBots `IN_*` events (e.g., `IN_BATHROOM`, `IN_KITCHEN`, `IN_BEDROOM`, etc.) through the `sensara-adaptor` service
2. **Current State**: Most location events currently have `hasTrigger: false` in their JSON schema definitions
3. **Schema Loading**: Event schemas are seeded from `megazord-events/schemas/events/*.json` at startup via `EventSchemasLoader`
4. **Impact Scope**: 
   - Changes only affect which events can be used for `TRIGGER_SUBSCRIPTION` type subscriptions
   - Does not impact existing `SERVICE_SUBSCRIPTION` workflows
   - Sensara adaptor uses `LocationEventMapper` to map Sensara locations to TinyBots events dynamically
5. **Generation Logic**: The schema generator (`schemas/gen.ts`) has a `CustomConfigs` object that sets `hasTrigger: true` for specific events - this needs to be updated for all Sensara-related events

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] ✅ Analyze event schema structure and generator logic
  - **Outcome**: Confirmed that `schemas/gen.ts` contains `CustomConfigs` for setting `hasTrigger` flag, and JSON files in `schemas/events/` are loaded at startup
- [x] ✅ Identify all Sensara-related events from `TinybotsEvent` enum
  - **Outcome**: Found 39 events total in schemas directory, with majority being location-based `IN_*` events from Sensara:
    - Location events: `IN_BATHROOM`, `IN_KITCHEN`, `IN_BEDROOM`, `IN_LIVING_ROOM`, `IN_TOILET`, `IN_DINING_ROOM`, `IN_OFFICE`, `IN_HOBBY_ROOM`, `IN_GUEST_ROOM`, `IN_ENTRANCE`, `IN_STAIRS`, `IN_CORRIDOR`, `IN_OTHER`, `IN_LAUNDRY_ROOM`, `IN_STORAGE_ROOM`, `IN_GARAGE`, `IN_GARDEN`, `IN_TERRACE`, `IN_BALCONY`, `IN_COMMUNAL_AREA`, `IN_HALL`, `IN_PANTRY`
    - Activity events: `ACTIVITY`, `EATING_ACTIVITY`, `TOILET_ACTIVITY`, `BATHROOM_ACTIVITY`
    - Bed events: `IN_BED`
    - Home boundary: `INSIDE_HOME`, `OUTSIDE_HOME`
    - Hearing range: `CLIENT_IN_HEARING_RANGE`
- [x] ✅ Define scope: which events should have `hasTrigger: true`
  - **Outcome**: Based on user requirement "For Sensara, there are certain events that can be used as triggers", we should enable triggers for:
    1. All location-based `IN_*` events (22 events)
    2. Activity-related events that come from Sensara (`ACTIVITY`)
    3. Home boundary events (`INSIDE_HOME`, `OUTSIDE_HOME`)
    4. Keep existing trigger-enabled events unchanged (`SUSPICIOUS_INACTIVITY`, `SHORT_INACTIVITY`, `OUT_OF_BED`, `EARLY_OUT_OF_BED`, `LONGER_IN_BED_SHORT`, `LONGER_IN_BED_LONG`)

### Phase 2: Implementation (File/Code Structure)

**Files to Modify:**

```
megazord-events/
├── schemas/
│   ├── gen.ts                          # 🔄 UPDATE - Add Sensara events to CustomConfigs
│   └── events/                         # 🔄 REGENERATE - JSON files will be regenerated
│       ├── activity.json               # Update hasTrigger to true
│       ├── in_bathroom.json            # Update hasTrigger to true
│       ├── in_kitchen.json             # Update hasTrigger to true
│       ├── in_bedroom.json             # Update hasTrigger to true
│       ├── in_living_room.json         # Update hasTrigger to true
│       ├── in_hall.json                # Update hasTrigger to true
│       ├── in_toilet.json              # Update hasTrigger to true
│       ├── in_dining_room.json         # Update hasTrigger to true
│       ├── in_office.json              # Update hasTrigger to true
│       ├── in_hobby_room.json          # Update hasTrigger to true
│       ├── in_guest_room.json          # Update hasTrigger to true
│       ├── in_entrance.json            # Update hasTrigger to true
│       ├── in_stairs.json              # Update hasTrigger to true
│       ├── in_corridor.json            # Update hasTrigger to true
│       ├── in_other.json               # Update hasTrigger to true
│       ├── in_laundry_room.json        # Update hasTrigger to true
│       ├── in_storage_room.json        # Update hasTrigger to true
│       ├── in_garage.json              # Update hasTrigger to true
│       ├── in_garden.json              # Update hasTrigger to true
│       ├── in_terrace.json             # Update hasTrigger to true
│       ├── in_balcony.json             # Update hasTrigger to true
│       ├── in_communal_area.json       # Update hasTrigger to true
│       ├── in_pantry.json              # Update hasTrigger to true
│       ├── inside_home.json            # Update hasTrigger to true
│       ├── outside_home.json           # Update hasTrigger to true
│       └── in_bed.json                 # Update hasTrigger to true
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Update Schema Generator Configuration
**File:** `megazord-events/schemas/gen.ts`

**Action:** Add all Sensara-related events to `CustomConfigs` with `hasTrigger: true` and appropriate level

**Changes:**
1. Add entries for all 22 location-based `IN_*` events
2. Add entries for `ACTIVITY`, `IN_BED`, `INSIDE_HOME`, `OUTSIDE_HOME`
3. Keep existing custom configs for `SUSPICIOUS_INACTIVITY`, `SHORT_INACTIVITY`, `OUT_OF_BED`, `EARLY_OUT_OF_BED`, `LONGER_IN_BED_SHORT`, `LONGER_IN_BED_LONG`

**Implementation:**
```typescript
const CustomConfigs: {
  [K in keyof typeof TinybotsEvent]?: Config
} = {
  // Existing trigger-enabled events
  SUSPICIOUS_INACTIVITY: { level: 5, hasTrigger: true },
  SHORT_INACTIVITY: { level: 5, hasTrigger: true },
  OUT_OF_BED: { level: 10, hasTrigger: true },
  EARLY_OUT_OF_BED: { level: 10, hasTrigger: true },
  LONGER_IN_BED_SHORT: { level: 10, hasTrigger: true },
  LONGER_IN_BED_LONG: { level: 10, hasTrigger: true },
  
  // Sensara location events - all can be used as triggers
  IN_BATHROOM: { level: 10, hasTrigger: true },
  IN_KITCHEN: { level: 10, hasTrigger: true },
  IN_PANTRY: { level: 10, hasTrigger: true },
  IN_BEDROOM: { level: 10, hasTrigger: true },
  IN_LIVING_ROOM: { level: 10, hasTrigger: true },
  IN_HALL: { level: 10, hasTrigger: true },
  IN_TOILET: { level: 10, hasTrigger: true },
  IN_DINING_ROOM: { level: 10, hasTrigger: true },
  IN_OFFICE: { level: 10, hasTrigger: true },
  IN_HOBBY_ROOM: { level: 10, hasTrigger: true },
  IN_GUEST_ROOM: { level: 10, hasTrigger: true },
  IN_ENTRANCE: { level: 10, hasTrigger: true },
  IN_STAIRS: { level: 10, hasTrigger: true },
  IN_CORRIDOR: { level: 10, hasTrigger: true },
  IN_OTHER: { level: 10, hasTrigger: true },
  IN_LAUNDRY_ROOM: { level: 10, hasTrigger: true },
  IN_STORAGE_ROOM: { level: 10, hasTrigger: true },
  IN_GARAGE: { level: 10, hasTrigger: true },
  IN_GARDEN: { level: 10, hasTrigger: true },
  IN_TERRACE: { level: 10, hasTrigger: true },
  IN_BALCONY: { level: 10, hasTrigger: true },
  IN_COMMUNAL_AREA: { level: 10, hasTrigger: true },
  
  // Sensara activity and state events
  ACTIVITY: { level: 10, hasTrigger: true },
  IN_BED: { level: 10, hasTrigger: true },
  INSIDE_HOME: { level: 10, hasTrigger: true },
  OUTSIDE_HOME: { level: 10, hasTrigger: true }
}
```

#### Step 2: Regenerate Event Schema JSON Files
**Directory:** `megazord-events/schemas/events/`

**Action:** Use the schema generator to regenerate all event JSON files with updated `hasTrigger` values

**Commands:**
```bash
cd /Users/kai/work/tinybots/tinybots/backend/megazord-events
# Regenerate all schemas with FORCE_GENERATE flag to overwrite existing files
FORCE_GENERATE=true yarn generate:schemas
```

**Verification:** Confirm that all Sensara-related event JSON files now have `hasTrigger: true`

#### Step 3: Verify Changes
**Action:** Review the generated files to ensure correctness

**Checks:**
1. All `IN_*` location events have `hasTrigger: true`
2. `ACTIVITY`, `IN_BED`, `INSIDE_HOME`, `OUTSIDE_HOME` have `hasTrigger: true`
3. Existing trigger events retain their configuration
4. No unintended changes to other properties (level, isActive, description)

#### Step 4: Database Schema Update Consideration
**Note:** The event schemas are loaded into the database at application startup via `EventSchemasLoader`. The upsert logic in `EventSchemasLoader` will update existing `event_schema` rows when the service restarts.

**Action Required:**
- No manual database migration needed
- Schema changes take effect on next deployment/restart
- Existing subscriptions are unaffected; new trigger subscriptions can be created for these events going forward

#### Step 5: Testing Strategy
**Manual Testing Plan:**

1. **Schema Generation Test:**
   - Run generator and verify JSON files are updated
   - Check git diff to confirm only `hasTrigger` field changed for target events

2. **Application Startup Test:**
   - Deploy/restart `megazord-events` service
   - Verify logs show schemas loaded successfully
   - Query `event_schema` table to confirm `hasTrigger` updated

3. **Trigger Subscription Test:**
   - Create a trigger subscription for a Sensara location event (e.g., `IN_KITCHEN`)
   - Simulate or trigger an actual Sensara location event
   - Verify that `m-o-triggers` receives the trigger notification
   - Confirm no errors in megazord-events logs

4. **Sensara Adaptor Integration Test:**
   - Verify that `sensara-adaptor` continues to work correctly
   - Confirm location events are still posted to megazord-events
   - Check that trigger subscriptions fire when Sensara events arrive

**Expected Outcome:**
- All Sensara events can be selected when creating trigger subscriptions
- Trigger subscriptions work end-to-end for Sensara events
- No regression in existing functionality

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

**Questions to Consider:**
1. **Scope Confirmation**: Should ALL Sensara events have `hasTrigger: true`, or only specific ones?
   - Currently including: all `IN_*` location events (22), `ACTIVITY`, `IN_BED`, `INSIDE_HOME`, `OUTSIDE_HOME`
   - Excluding from triggers: `CLIENT_IN_HEARING_RANGE` (internal poller event), specific activity types like `EATING_ACTIVITY`, `TOILET_ACTIVITY`, `BATHROOM_ACTIVITY` (may be too granular)
   - **Recommendation**: Start with location and basic activity events; can expand later based on use cases

2. **Event Level Priority**: All Sensara events are currently set to `level: 10` (same as default)
   - Should some location events have higher priority (lower level number)?
   - Example: `IN_TOILET` might warrant higher priority than `IN_GARDEN`
   - **Recommendation**: Start with uniform level 10; adjust based on production usage patterns

3. **Backward Compatibility**: What happens to existing systems that may not expect these events to trigger?
   - Risk: Low - only affects new trigger subscriptions, existing subscriptions unchanged
   - Mitigation: Document the change and notify relevant teams

4. **Testing Coverage**: Should we add automated integration tests for these new trigger-enabled events?
   - Current test suite in `megazord-events/test/` has integration tests for subscriptions
   - **Recommendation**: Add test cases for at least 2-3 representative Sensara events to verify trigger flow

**Follow-up Tasks:**
- [ ] Monitor production logs after deployment for any unexpected behavior
- [ ] Update API documentation if it lists which events support triggers
- [ ] Consider adding dashboard UI to show all trigger-capable events
- [ ] Gather feedback from users on which Sensara triggers are most useful
