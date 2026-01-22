# Trigger Configuration Enhancement - Schedule Constraints

**Date**: November 3, 2025
**Status**: In Progress - Phase 1 (allowedDaysOfWeek)

---

## Context

The trigger system is **event-driven**: when an external event arrives, the system creates a trigger and validates if it can execute based on time windows.

### Previous Behavior

Triggers were only constrained by **time-of-day** windows:
- `allowedStartTime` / `allowedEndTime` (e.g., 08:00-20:00)
- If current time is within window → Execute immediately
- If outside window + `isReschedulable=true` → Schedule for next day at startTime
- Otherwise → Reject

### Limitation

❌ Could not configure day-level constraints (e.g., "only Mon, Wed, Fri")

---

## Solution Approach

**Selected**: JSON Schedule Constraints (Solution B from proposal)

### Why JSON?

- ✅ Complements existing time window logic (not replaces)
- ✅ Extensible without schema changes
- ✅ Simple UI (checkboxes for days)
- ✅ Backward compatible (`NULL` = no constraints)

### Design

```typescript
// Data model
interface ScheduleConstraints {
  allowedDaysOfWeek?: ('mon'|'tue'|'wed'|'thu'|'fri'|'sat'|'sun')[];
  // Future: allowedDaysOfMonth, excludedDates, etc.
}
```

**Database**:
```sql
ALTER TABLE event_trigger_setting
ADD COLUMN schedule_constraints JSON DEFAULT NULL;
```

**Storage Examples**:
```json
{"allowedDaysOfWeek": ["mon", "wed", "fri"]}
{"allowedDaysOfWeek": ["mon", "tue", "wed", "thu", "fri"]}
null  // No constraints (backward compatible)
```

---

## Phase 1 Implementation (Current)

### Scope

**Implemented**: `allowedDaysOfWeek` only
**Deferred**: `allowedDaysOfMonth`, `excludedDates`, `allowedWeeksOfMonth`

### Files Modified

1. **Database Migration**: `ci/db/migrations/add_schedule_constraints.sql`
   - Added `schedule_constraints JSON` column

2. **Domain Model**: `src/models/domains/EventTriggerDomain.ts`
   - Added `ScheduleConstraints` interface
   - Added `scheduleConstraints?: ScheduleConstraints` to `EventTriggerSettingDomain`
   - JSON transform decorator for parsing

3. **DTO**: `src/models/dtos/UpsertTriggerSettingDto.ts`
   - Added `scheduleConstraints` field with validation
   - Validates `allowedDaysOfWeek` against valid day names

4. **Repository**: `src/repositories/EventTriggerRepository.ts`
   - Updated all SELECT queries to include `schedule_constraints`
   - Updated INSERT queries to handle JSON field
   - Added to `ICreateEventTriggerSetting` interface

5. **Business Logic**: `src/services/TriggerSchedulerParser.ts`
   - Modified `getExpectedExecutedTime()` to check day constraints
   - Added `findNextAllowedDate()` for day-of-week filtering
   - Added `advanceToNextDay()` for rescheduling logic

6. **Tests**: `test/services/TriggerSchedulerParserTest.ts`
   - Added unit tests for day-of-week filtering
   - Edge cases: disallowed days, rescheduling, weekend scenarios

---

## How It Works

### Flow

```typescript
Event arrives → TriggerSchedulerParser.pre()
  ↓
Get default setting (with scheduleConstraints)
  ↓
getExpectedExecutedTime(setting)
  ↓
1. Check time window (08:00-20:00) ← EXISTING
2. Check day constraint (mon, wed, fri) ← NEW
  ↓
If today allowed + in time window → Execute NOW
If today not allowed → Find next allowed day
If no valid day found → Reject
```

### Example

```javascript
// Configuration
{
  allowedStartTime: "08:00",
  allowedEndTime: "20:00",
  isReschedulable: true,
  scheduleConstraints: {
    allowedDaysOfWeek: ["mon", "wed", "fri"]
  }
}

// Event arrives Tuesday 10:00
→ Schedule for Wednesday 08:00 ✅

// Event arrives Friday 14:00
→ Execute NOW ✅

// Event arrives Friday 21:00 (after end time)
→ Schedule for Monday 08:00 (next week) ✅
```

---

## Key Implementation Details

### 1. Day Validation

```typescript
const dayName = candidate.format('ddd').toLowerCase(); // 'mon', 'tue', etc.
if (!allowedDaysOfWeek.includes(dayName)) {
  candidate = advanceToNextDay(candidate, setting);
}
```

### 2. Rescheduling Logic

- Advances to **next day at `allowedStartTime`**
- Searches up to **7 days** (one week) to find valid date
- Returns `null` if no valid date found within 7 days (trigger rejected)

### 3. Backward Compatibility

```typescript
if (!setting.scheduleConstraints) {
  return getExpectedExecutedTimeByWindow(now, setting); // Original logic
}
```

### 4. Robot-Specific Overrides

Works seamlessly with existing robot-specific triggers:
- Default trigger: `robotId = NULL`, `scheduleConstraints = {"allowedDaysOfWeek": ["mon", "wed", "fri"]}`
- Robot #42 override: `robotId = 42`, `scheduleConstraints = {"allowedDaysOfWeek": ["tue", "thu"]}`
- Repository already prioritizes robot-specific settings

---

## API Usage

### Create/Update Trigger Setting

```http
PUT /v1/triggers/settings
Content-Type: application/json

{
  "eventName": "event.restock",
  "robotId": 42,  // Optional - omit for default
  "allowedStartTime": "08:00",
  "allowedEndTime": "20:00",
  "isReschedulable": true,
  "scheduleConstraints": {
    "allowedDaysOfWeek": ["mon", "wed", "fri"]
  },
  "maxConcurrentTriggers": 3,
  "allowedConcurrencyTimeframe": "5m",
  "defaultScriptReferenceId": 123
}
```

### Response

```json
{
  "id": 1001,
  "robotId": 42,
  "eventTypeId": 7,
  "scheduleConstraints": {
    "allowedDaysOfWeek": ["mon", "wed", "fri"]
  },
  "allowedStartTime": "08:00",
  "allowedEndTime": "20:00",
  "isReschedulable": true,
  "isDefault": false,
  "defaultScriptReferenceId": 123,
  "createdAt": "2025-11-03T00:00:00.000Z",
  "updatedAt": "2025-11-03T00:00:00.000Z"
}
```

---

## Testing

### Unit Tests (`TriggerSchedulerParserTest.ts`)

- ✅ Execute immediately when day is allowed and in time window
- ✅ Reschedule to next allowed day when current day not allowed
- ✅ Reschedule to next week when needed (e.g., Sat → Mon)
- ✅ Return null when no allowed days configured
- ✅ Handle edge cases (reschedulable=false, max search exceeded)

### Integration Tests (TODO)

- Repository CRUD with `scheduleConstraints`
- End-to-end trigger creation with day filtering
- Cache invalidation on setting updates

---

## Future Extensions

### Phase 2: Day of Month

```json
{
  "allowedDaysOfMonth": [1, 15, 30]
}
```

### Phase 3: Advanced Patterns

```json
{
  "allowedWeeksOfMonth": [
    {"week": 1, "dayOfWeek": "mon"}  // First Monday
  ],
  "excludedDates": ["2025-12-25", "2026-01-01"]
}
```

---

## Migration Notes

### Running Migration

```bash
# Apply migration
mysql -u user -p database < ci/db/migrations/add_schedule_constraints.sql

# Verify
SHOW COLUMNS FROM event_trigger_setting LIKE 'schedule_constraints';
```

### Rollback

```sql
ALTER TABLE event_trigger_setting DROP COLUMN schedule_constraints;
```

### Data Migration

No data migration needed - existing triggers work with `NULL` constraints.

Optional: Add constraints to existing settings:
```sql
UPDATE event_trigger_setting
SET schedule_constraints = '{"allowedDaysOfWeek":["mon","tue","wed","thu","fri"]}'
WHERE event_type_id = 7 AND robot_id IS NULL;
```

---

## References

- **Full Proposal**: [schedule_constraints_proposal.md](./schedule_constraints_proposal.md)
- **Parser Logic**: `src/services/TriggerSchedulerParser.ts`
- **Domain Models**: `src/models/domains/EventTriggerDomain.ts`
- **Repository**: `src/repositories/EventTriggerRepository.ts`
