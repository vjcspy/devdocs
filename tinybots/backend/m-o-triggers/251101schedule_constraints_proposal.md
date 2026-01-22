# Schedule Constraints Proposal

## Executive Summary

This proposal addresses the need to extend trigger scheduling capabilities beyond simple time-of-day windows. Currently, triggers can only be constrained by `allowedStartTime` and `allowedEndTime` (e.g., 08:00-20:00). This proposal introduces day-level scheduling constraints to support business requirements such as "only run on weekdays" or "only run on the first day of the month."

---

## Current System Behavior

### Event-Driven Trigger Flow

1. **External event arrives** → System creates a trigger
2. **Time window validation**:
   - If `NOW` is between `allowedStartTime` and `allowedEndTime` → **Execute immediately**
   - If `NOW < allowedStartTime` AND `isReschedulable=true` → **Schedule for startTime today**
   - If `NOW > allowedEndTime` AND `isReschedulable=true` → **Schedule for startTime tomorrow**
   - Otherwise → **Reject**

### Example
```javascript
Event: "Robot needs restock" arrives at 14:30
Setting: allowedStartTime=08:00, allowedEndTime=20:00
Result: Execute NOW (14:30) because it's within the time window

Event arrives at 22:00
Setting: allowedStartTime=08:00, allowedEndTime=20:00, isReschedulable=true
Result: Schedule for 08:00 tomorrow
```

### Current Limitations

❌ Cannot configure: "Only run on Monday, Wednesday, Friday"
❌ Cannot configure: "Only run on the 1st and 15th of each month"
❌ Cannot configure: "Only run on the first Monday of the month"

---

## Business Requirements

### User Stories

**Story 1: Day-of-Week Filtering**
```
AS A warehouse manager
I WANT triggers to only execute on specific days of the week
SO THAT I can align robot tasks with our shipping schedule

Acceptance Criteria:
- Admin can select which days of the week are allowed (Mon-Sun)
- Events arriving on disallowed days are rescheduled to the next allowed day
- Time window (HH:mm) still applies on allowed days
```

**Story 2: Day-of-Month Patterns**
```
AS AN operations manager
I WANT triggers to execute on specific dates of the month
SO THAT I can coordinate with monthly inventory cycles

Acceptance Criteria:
- Admin can specify specific days of the month (1-31)
- Events arriving on other days are rescheduled to the next allowed date
- Handles months with fewer than 31 days gracefully
```

**Story 3: Advanced Patterns (Future)**
```
AS A scheduler admin
I WANT to define complex scheduling rules
SO THAT I can handle seasonal or special scheduling needs

Examples:
- First Monday of each month
- Last business day of the month
- Exclude specific dates (holidays)
```

---

## Proposed Solutions

### Solution A: Full CRON Expression

#### Design

Add a `schedule_cron` column that uses standard CRON syntax to define when triggers are allowed to execute.

**Database Schema:**
```sql
ALTER TABLE event_trigger_setting
ADD COLUMN schedule_cron VARCHAR(255) DEFAULT NULL
COMMENT 'Cron expression for allowed execution times (Quartz format)';
```

**Examples:**
```bash
'0 9 * * MON-FRI'     # 9am on weekdays
'0 0 * * MON#1'       # First Monday of month at midnight
'0 */15 9-17 * *'     # Every 15 min, 9am-5pm, any day
NULL                  # Always allowed (backward compatible)
```

#### Implementation

```typescript
import parser from 'cron-parser';

export class EventTriggerSettingDomain extends BaseDomain {
  @Expose()
  @IsString()
  @IsOptional()
  scheduleCron?: string; // Quartz cron format

  getNextAllowedExecution(from: Date): Date | null {
    if (!this.scheduleCron) return from; // No constraints

    const interval = parser.parseExpression(this.scheduleCron, {
      currentDate: from,
      tz: 'UTC'
    });

    return interval.next().toDate();
  }
}
```

#### Pros & Cons

**Pros:**
- ✅ Extremely flexible - handles any scheduling pattern
- ✅ Industry standard (Quartz, Spring, Kubernetes)
- ✅ Battle-tested libraries available (`cron-parser`, `croner`)
- ✅ Can calculate exact "next execution time"
- ✅ Future-proof for complex requirements

**Cons:**
- ⚠️ **Overkill for current needs** - minute/hour precision unnecessary
- ⚠️ **Potential conflicts** - CRON includes time-of-day, but we already have `allowedStartTime`/`allowedEndTime`
- ⚠️ **Complex UX** - Non-technical users struggle with CRON syntax
- ⚠️ **Learning curve** - Team needs to understand CRON format
- ⚠️ **Validation complexity** - Must ensure CRON doesn't conflict with existing time logic

#### Use Cases

Best for:
- Systems that need minute/hour-level precision
- Scheduled background jobs (not event-driven triggers)
- Teams already familiar with CRON

Not ideal for:
- Event-driven systems with existing time-window logic
- Non-technical administrators
- Simple day-level filtering needs

---

### Solution B: JSON Schedule Constraints (Recommended)

#### Design

Add a `schedule_constraints` JSON column that stores structured day-level constraints, working alongside existing time windows.

**Database Schema:**
```sql
ALTER TABLE event_trigger_setting
ADD COLUMN schedule_constraints JSON DEFAULT NULL
COMMENT 'Additional scheduling constraints beyond time window';
```

**Examples:**
```json
{"allowedDaysOfWeek": ["mon", "wed", "fri"]}
{"allowedDaysOfMonth": [1, 15]}
{"allowedDaysOfWeek": ["mon"], "excludedDates": ["2025-12-25"]}
NULL  // No constraints (backward compatible)
```

#### Data Model

```typescript
export interface ScheduleConstraints {
  allowedDaysOfWeek?: ('mon'|'tue'|'wed'|'thu'|'fri'|'sat'|'sun')[];
  allowedDaysOfMonth?: number[]; // [1, 15, 30]

  // Future extensions:
  allowedWeeksOfMonth?: {
    week: 1 | 2 | 3 | 4 | 5 | 'first' | 'last';
    dayOfWeek: 'mon'|'tue'|'wed'|'thu'|'fri'|'sat'|'sun';
  }[];c
  excludedDates?: string[]; // ['2025-12-25', '2026-01-01']
  timezone?: string; // 'Asia/Ho_Chi_Minh'
}

export class EventTriggerSettingDomain extends BaseDomain {
  @Expose()
  @IsOptional()
  @ValidateNested()
  scheduleConstraints?: ScheduleConstraints;
}
```

#### Implementation

```typescript
private getExpectedExecutedTime(
  setting: EventTriggerSettingDomain
): dayjs.Dayjs | null {
  const now = dayjs();

  // Step 1: Check time window (existing logic)
  const { isInTimeWindow, nextWindowStart } = this.checkTimeWindow(now, setting);

  // Step 2: Apply schedule constraints
  if (setting.scheduleConstraints) {
    const startDate = isInTimeWindow ? now : nextWindowStart;
    const allowedDate = this.findNextAllowedDate(startDate, setting);

    if (!allowedDate) return null;
    return allowedDate;
  }

  // Step 3: No constraints - use existing behavior
  return isInTimeWindow ? now : nextWindowStart;
}

private findNextAllowedDate(
  startFrom: dayjs.Dayjs,
  setting: EventTriggerSettingDomain
): dayjs.Dayjs | null {
  let candidate = startFrom;
  let attempts = 0;
  const MAX_SEARCH_DAYS = 60;

  while (attempts < MAX_SEARCH_DAYS) {
    // Check day-of-week constraint
    if (setting.scheduleConstraints?.allowedDaysOfWeek) {
      const dayName = candidate.format('ddd').toLowerCase();
      if (!setting.scheduleConstraints.allowedDaysOfWeek.includes(dayName)) {
        candidate = this.advanceToNextDay(candidate, setting);
        attempts++;
        continue;
      }
    }

    // Check day-of-month constraint
    if (setting.scheduleConstraints?.allowedDaysOfMonth) {
      const dayOfMonth = candidate.date();
      if (!setting.scheduleConstraints.allowedDaysOfMonth.includes(dayOfMonth)) {
        candidate = this.advanceToNextDay(candidate, setting);
        attempts++;
        continue;
      }
    }

    return candidate; // Found valid date
  }

  return null; // No valid date found within search window
}
```

#### API Example

```json
PUT /v1/triggers/settings
{
  "eventName": "event.restock",
  "robotId": 42,
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

**Translation**: "Only run from 8am-8pm on Monday, Wednesday, Friday"

#### Pros & Cons

**Pros:**
- ✅ **Separation of concerns** - Time (HH:mm) vs. Days (constraints) are orthogonal
- ✅ **User-friendly** - Simple checkbox UI for day selection
- ✅ **No conflicts** - Complements existing time window logic
- ✅ **Incremental adoption** - Can add features phase by phase
- ✅ **Extensible** - JSON can grow without schema changes
- ✅ **Clear business logic** - Easy to explain to stakeholders
- ✅ **Backward compatible** - NULL = no constraints

**Cons:**
- ⚠️ **JSON query performance** - Filtering in SQL is less efficient than columns
- ⚠️ **Custom parsing** - Need to implement constraint checking logic
- ⚠️ **Limited by design** - Cannot express minute-level patterns (intentional)

#### Use Cases

Best for:
- Event-driven systems with time-window filtering
- Day-level scheduling constraints
- Non-technical administrators
- Incremental feature rollout

Ideal when:
- You already have time-of-day logic
- You need simple, clear business rules
- You want room for future extensions

---

### Solution C: Separate Columns

#### Design

Add explicit columns for each type of constraint.

**Database Schema:**
```sql
ALTER TABLE event_trigger_setting
ADD COLUMN allowed_days_of_week VARCHAR(50) DEFAULT NULL
  COMMENT 'Comma-separated: mon,wed,fri',
ADD COLUMN allowed_days_of_month VARCHAR(100) DEFAULT NULL
  COMMENT 'Comma-separated: 1,15,30';
```

**Examples:**
```sql
allowed_days_of_week: 'mon,wed,fri'
allowed_days_of_month: '1,15'
allowed_days_of_week: NULL  -- No constraint
```

#### Data Model

```typescript
export class EventTriggerSettingDomain extends BaseDomain {
  @Expose()
  @IsString()
  @IsOptional()
  allowedDaysOfWeek?: string; // 'mon,wed,fri'

  @Expose()
  @IsString()
  @IsOptional()
  allowedDaysOfMonth?: string; // '1,15,30'

  // Helper methods
  getDaysOfWeek(): string[] {
    return this.allowedDaysOfWeek?.split(',') || [];
  }

  getDaysOfMonth(): number[] {
    return this.allowedDaysOfMonth?.split(',').map(Number) || [];
  }
}
```

#### Implementation

```typescript
private findNextAllowedDate(
  startFrom: dayjs.Dayjs,
  setting: EventTriggerSettingDomain
): dayjs.Dayjs | null {
  const allowedWeekDays = setting.getDaysOfWeek();
  const allowedMonthDays = setting.getDaysOfMonth();

  let candidate = startFrom;
  let attempts = 0;

  while (attempts < 60) {
    if (allowedWeekDays.length > 0) {
      const dayName = candidate.format('ddd').toLowerCase();
      if (!allowedWeekDays.includes(dayName)) {
        candidate = this.advanceToNextDay(candidate, setting);
        attempts++;
        continue;
      }
    }

    if (allowedMonthDays.length > 0) {
      const dayOfMonth = candidate.date();
      if (!allowedMonthDays.includes(dayOfMonth)) {
        candidate = this.advanceToNextDay(candidate, setting);
        attempts++;
        continue;
      }
    }

    return candidate;
  }

  return null;
}
```

#### Pros & Cons

**Pros:**
- ✅ **Explicit schema** - Clear what constraints exist
- ✅ **Easy SQL queries** - Direct column access: `WHERE allowed_days_of_week LIKE '%mon%'`
- ✅ **Better indexing** - Can create indexes on specific columns
- ✅ **Simple parsing** - Just split by comma
- ✅ **Clear validation** - One column = one concern

**Cons:**
- ⚠️ **Less extensible** - Adding new constraint types requires ALTER TABLE
- ⚠️ **More columns** - Schema grows with each new constraint type
- ⚠️ **Duplication** - Similar parsing logic for each column type
- ⚠️ **Migration overhead** - Each new feature needs schema change

#### Use Cases

Best for:
- Fixed set of constraint types
- Heavy SQL querying/filtering needs
- Performance-critical scenarios
- Teams that prefer explicit schemas

Not ideal for:
- Rapidly evolving requirements
- Need for complex nested constraints
- Frequent schema changes

---

## Comparison Matrix

| Feature | Full CRON | JSON Constraints | Separate Columns |
|---------|-----------|------------------|------------------|
| **Handles day-of-week** | ✅ | ✅ | ✅ |
| **Handles day-of-month** | ✅ | ✅ | ✅ |
| **Handles hour/minute precision** | ✅ (unnecessary) | ❌ (by design) | ❌ (by design) |
| **Works with existing time window** | ⚠️ Conflict risk | ✅ Complementary | ✅ Complementary |
| **Admin UX complexity** | ⚠️ High (CRON syntax) | ✅ Low (checkboxes) | ✅ Low (checkboxes) |
| **Extensibility** | ✅ Very high | ✅ High (JSON) | ⚠️ Medium (ALTER TABLE) |
| **Developer experience** | ⚠️ Learn CRON | ✅ Clear logic | ✅ Clear logic |
| **SQL query performance** | N/A | ⚠️ JSON parsing | ✅ Direct column access |
| **Schema complexity** | Low (1 column) | Low (1 column) | Medium (N columns) |
| **Validation complexity** | High | Medium | Low |
| **Future-proofing** | ✅ Handles anything | ✅ Easy to extend | ⚠️ Requires migrations |
| **Backward compatibility** | ✅ NULL = no constraint | ✅ NULL = no constraint | ✅ NULL = no constraint |
| **Conflicts with existing logic** | ⚠️ Yes (time overlap) | ✅ No (orthogonal) | ✅ No (orthogonal) |
| **Learning curve** | High | Low | Low |

---

## Recommendation

### **Solution B: JSON Schedule Constraints**

#### Rationale

1. **Fits the Current Architecture**
   - System is event-driven, not scheduled
   - Already has time-window logic (`allowedStartTime`/`allowedEndTime`)
   - Constraints are supplementary, not replacement

2. **Separation of Concerns**
   - **Time precision** (HH:mm): Handled by existing fields
   - **Day filtering**: Handled by `scheduleConstraints`
   - Clear, orthogonal responsibilities

3. **User Experience**
   - Non-technical admins can configure easily
   - Simple UI with checkboxes for days
   - Clear mental model: "Time window + Day filter"

4. **Extensibility**
   ```javascript
   // Phase 1: Basic day filtering
   {"allowedDaysOfWeek": ["mon", "wed", "fri"]}

   // Phase 2: Monthly patterns
   {"allowedDaysOfMonth": [1, 15]}

   // Phase 3: Advanced (future)
   {
     "allowedWeeksOfMonth": [{"week": 1, "dayOfWeek": "mon"}],
     "excludedDates": ["2025-12-25"],
     "timezone": "Asia/Ho_Chi_Minh"
   }
   ```

5. **No Breaking Changes**
   - `NULL` = no constraints (current behavior)
   - Existing triggers continue working
   - Gradual adoption possible

6. **Avoids Over-Engineering**
   - CRON is overkill for day-level filtering
   - Separate columns limit future flexibility
   - JSON strikes the right balance

#### When to Reconsider

Switch to **Full CRON** if:
- Requirements emerge for minute/hour precision scheduling
- System shifts from event-driven to time-scheduled jobs
- Need to match external systems using CRON

Switch to **Separate Columns** if:
- SQL query performance becomes critical
- Constraint types are fixed forever
- Need advanced database-level filtering

---

## Implementation Roadmap

### Phase 1: MVP - Day of Week (Sprint 1-2)

**Scope:**
- Add `schedule_constraints` JSON column
- Implement `allowedDaysOfWeek` support only
- Update `TriggerSchedulerParser.getExpectedExecutedTime()`
- Basic validation and tests
- Simple UI: 7 checkboxes for days

**Deliverables:**
- Database migration script
- Updated domain models and DTOs
- Business logic in `TriggerSchedulerParser`
- Unit + integration tests
- API documentation

### Phase 2: Day of Month (Sprint 3-4)

**Scope:**
- Add `allowedDaysOfMonth` support
- Handle edge cases (Feb 30, etc.)
- Validation for combining constraints
- Enhanced UI with text input

**Deliverables:**
- Extended constraint logic
- Additional test coverage
- Updated documentation

### Phase 3: Advanced Patterns (Future)

**Scope:**
- First/Last day-of-week of month
- Exclude specific dates (holidays)
- Timezone-aware scheduling
- Admin UI improvements (visual calendar, presets)

---

## Example Scenarios

### Scenario 1: Weekday Shipping Schedule

**Configuration:**
```json
{
  "allowedStartTime": "08:00",
  "allowedEndTime": "18:00",
  "scheduleConstraints": {
    "allowedDaysOfWeek": ["mon", "wed", "fri"]
  },
  "isReschedulable": true
}
```

**Behavior:**
| Event Arrival | Current System | With Constraints |
|---------------|----------------|------------------|
| Mon 10:00 | Execute NOW | Execute NOW ✅ |
| Tue 10:00 | Execute NOW | Schedule Wed 08:00 ✅ |
| Fri 19:00 | Schedule Sat 08:00 | Schedule Mon 08:00 ✅ |
| Sun 10:00 | Execute NOW | Schedule Mon 08:00 ✅ |

### Scenario 2: Monthly Inventory Cycle

**Configuration:**
```json
{
  "allowedStartTime": "00:00",
  "allowedEndTime": "23:59",
  "scheduleConstraints": {
    "allowedDaysOfMonth": [1, 15]
  },
  "isReschedulable": true
}
```

**Behavior:**
| Event Arrival | Result |
|---------------|--------|
| Oct 1, 10:00 | Execute NOW ✅ |
| Oct 10, 10:00 | Schedule Oct 15, 00:00 ✅ |
| Oct 15, 14:00 | Execute NOW ✅ |
| Oct 20, 10:00 | Schedule Nov 1, 00:00 ✅ |

---

## Technical Considerations

### Database Migration

```sql
-- Migration: Add schedule_constraints column
ALTER TABLE event_trigger_setting
ADD COLUMN schedule_constraints JSON DEFAULT NULL
COMMENT 'Day-level scheduling constraints (allowedDaysOfWeek, allowedDaysOfMonth)';

-- Rollback plan
ALTER TABLE event_trigger_setting
DROP COLUMN schedule_constraints;
```

### Validation Rules

```typescript
class ScheduleConstraintsDto {
  @IsArray()
  @IsOptional()
  @IsIn(['mon','tue','wed','thu','fri','sat','sun'], { each: true })
  allowedDaysOfWeek?: string[];

  @IsArray()
  @IsOptional()
  @IsInt({ each: true })
  @Min(1, { each: true })
  @Max(31, { each: true })
  allowedDaysOfMonth?: number[];

  // Business rule: Start simple, combine later
  @Validate(NoConflictConstraints)
  validate() {
    if (this.allowedDaysOfWeek && this.allowedDaysOfMonth) {
      throw new Error('Cannot combine constraints in v1');
    }
  }
}
```

### Performance Impact

**Read Path**: Minimal impact
- Constraint checking happens in-memory (application layer)
- No additional database queries

**Write Path**: No impact
- Single JSON column update

**Query Path**: Moderate impact
- Cannot filter by constraints in SQL efficiently
- Acceptable for current scale (triggers are not heavily queried)

---

## Open Questions

1. **Timezone Handling**: Should day calculations use UTC or configurable timezone?
2. **Combining Constraints**: Allow `allowedDaysOfWeek` + `allowedDaysOfMonth` in Phase 1 or later?
3. **Validation**: Block empty arrays (`[]`) or treat as "no days allowed"?
4. **UI Location**: Add to existing settings form or create new "Advanced Scheduling" tab?

---

## Appendix

### Admin UI Mockup

```
┌─────────────────────────────────────────────────┐
│ Trigger Setting Configuration                   │
├─────────────────────────────────────────────────┤
│ Event Type: [Restock Event ▼]                  │
│ Robot: [Robot #42 ▼] (Optional - for override) │
│                                                 │
│ ─── Time Window ───────────────────────────────│
│ Start Time: [08:00]    End Time: [20:00]       │
│ ☑ Allow Rescheduling                           │
│                                                 │
│ ─── Day Constraints (Optional) ────────────────│
│ Run On Days:                                    │
│ ☑ Mon  ☐ Tue  ☑ Wed  ☐ Thu  ☑ Fri  ☐ Sat  ☐ Sun│
│                                                 │
│ Or Specific Dates:                              │
│ [1, 15] (comma-separated, 1-31)                │
│                                                 │
│ ─── Concurrency ───────────────────────────────│
│ Max Concurrent: [3]  Window: [5m]              │
│                                                 │
│ ─── Script ────────────────────────────────────│
│ Default Script ID: [123]                        │
│                                                 │
│               [Cancel]  [Save Setting]          │
└─────────────────────────────────────────────────┘
```

### References

- [Current Implementation: TriggerSchedulerParser.ts](../src/services/TriggerSchedulerParser.ts)
- [Domain Models: EventTriggerDomain.ts](../src/models/domains/EventTriggerDomain.ts)
- [Repository: EventTriggerRepository.ts](../src/repositories/EventTriggerRepository.ts)
- [Existing Documentation: robot_specific-triggers.md](./robot_specific-triggers.md)

---

**Document Version**: 1.0
**Date**: October 27, 2025
**Author**: Development Team
**Status**: Proposal - Pending Review
