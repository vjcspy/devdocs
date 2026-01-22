# 📋 [251107-2025-11-07] - Replace schedule_constraints with allowed_days_of_week Column

## 🎯 Objective
> Replace the JSON `schedule_constraints` column with a simple VARCHAR `allowed_days_of_week` column to store comma-separated day values for better database compatibility and simplicity.

**Background**: After review, stakeholders want to simplify the design by:
1. **Removing** the complex JSON `schedule_constraints` column (not yet deployed)
2. **Adding** a simple `allowed_days_of_week VARCHAR` column
3. **Renaming** `ScheduleConstraints` → simpler domain model focused only on days of week
4. **No migration needed** - feature hasn't been deployed to production yet

---

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze current implementation
  - **Outcome**: Current implementation uses JSON type with `{"allowedDaysOfWeek":["mon","wed","fri"]}` format
  - **Decision**: Since NOT deployed yet, we can do breaking changes:
    - ❌ **Remove**: `schedule_constraints JSON` column
    - ✅ **Add**: `allowed_days_of_week VARCHAR(100)` column
    - ✅ **Simplify**: Remove complex `ScheduleConstraints` class
  - **Affected Files**:
    - `src/models/domains/EventTriggerDomain.ts` - Simplify domain model
    - `src/repositories/EventTriggerRepository.ts` - Update column name
    - `src/models/dtos/UpsertTriggerSettingDto.ts` - Simplify DTO
    - `src/services/TriggerSchedulerParser.ts` - Update to use new field
    - `test/services/TriggerSchedulerParserTest.ts` - Update test data
- [x] Define scope and edge cases
  - **Outcome**:
    - Edge cases to handle:
      - `NULL` values (no constraints = any day allowed)
      - Empty string (should be treated as NULL)
      - Invalid day names (should reject at application level)
      - Duplicate days (should deduplicate via Set)
      - Unsorted days (should normalize to sorted order)
      - Case sensitivity (normalize to lowercase)
    - **No data migration needed** - feature not deployed yet
    - **VARCHAR vs TEXT decision**:
      - VARCHAR(100) ✅ Recommended: Better for indexed columns, faster for short strings
      - TEXT ❌ Overkill: For very long content (>255 chars)
      - Max we need: "fri,mon,sat,sun,thu,tue,wed" = 31 chars → VARCHAR(100) is perfect---

### Phase 2: Implementation (File/Code Structure)

#### 📁 File Structure
```
src/
├── models/
│   ├── domains/
│   │   └── EventTriggerDomain.ts        # 🔄 IN PROGRESS - Remove ScheduleConstraints, add allowedDaysOfWeek
│   └── dtos/
│       └── UpsertTriggerSettingDto.ts   # 🔄 IN PROGRESS - Change to allowedDaysOfWeek array
├── repositories/
│   └── EventTriggerRepository.ts         # 🔄 IN PROGRESS - Update column name + serialization
├── services/
│   └── TriggerSchedulerParser.ts         # 🔄 IN PROGRESS - Use allowedDaysOfWeek directly
test/
├── services/
│   └── TriggerSchedulerParserTest.ts     # 🔄 IN PROGRESS - Update test data
├── repositories/
│   └── EventTriggersRepositoryTest.ts    # 🔄 IN PROGRESS - Add tests for new column
└── models/
    └── domains/
        └── EventTriggerDomainTest.ts     # 🚧 TODO - Add serialization tests
ci/
└── db/
    └── migrations/
        └── V94__replace_schedule_constraints_with_allowed_days.sql  # 🚧 TODO - Create migration
```

---

### Phase 3: Detailed Implementation Steps

#### Step 1: Create Database Migration (🚧 TODO)
**File**: `ci/db/migrations/V94__replace_schedule_constraints_with_allowed_days.sql`

**Decision: VARCHAR(100) vs TEXT**
- **VARCHAR(100)** ✅ **RECOMMENDED**
  - Fixed maximum length (100 bytes)
  - Can be indexed efficiently
  - Stored inline with row data (faster access)
  - Perfect for short, known-length strings
  - Our max: "fri,mon,sat,sun,thu,tue,wed" = 31 chars
- **TEXT**
  - Variable length, no max (up to 64KB)
  - Cannot be fully indexed (only prefix)
  - Stored separately from row (slower access)
  - Use only for large content (>255 chars)

**✅ Choose VARCHAR(100)** - More than enough for our use case and better performance.

**Migration SQL**:
```sql
-- Step 1: Drop old JSON column (not deployed yet, safe to drop)
ALTER TABLE event_trigger_setting
DROP COLUMN schedule_constraints;

-- Step 2: Add new VARCHAR column
ALTER TABLE event_trigger_setting
ADD COLUMN allowed_days_of_week VARCHAR(100) DEFAULT NULL
COMMENT 'Comma-separated list of allowed days: fri,mon,sat,sun,thu,tue,wed (sorted alphabetically)';

-- Optional: Add check constraint for MySQL 8.0.16+ (validate format at DB level)
-- ALTER TABLE event_trigger_setting
-- ADD CONSTRAINT check_allowed_days_format
-- CHECK (
--   allowed_days_of_week IS NULL OR
--   allowed_days_of_week REGEXP '^(fri|mon|sat|sun|thu|tue|wed)(,(fri|mon|sat|sun|thu|tue|wed))*$'
-- );
```

**Rollback SQL**:
```sql
-- Remove new column
ALTER TABLE event_trigger_setting
DROP COLUMN allowed_days_of_week;

-- Restore old column (if needed for rollback)
ALTER TABLE event_trigger_setting
ADD COLUMN schedule_constraints JSON DEFAULT NULL;
```

---

#### Step 2: Update Domain Model (🔄 IN PROGRESS)
**File**: `src/models/domains/EventTriggerDomain.ts`

**Changes**:
1. **❌ REMOVE** the entire `ScheduleConstraints` class (not needed anymore)

2. **✅ ADD** direct field with helper methods:
   ```typescript
   // Helper functions for day serialization (move to EventTriggerSettingDomain or utility)
   export const serializeDaysOfWeek = (days: DayOfWeek[] | undefined): string | null => {
     if (!days || days.length === 0) return null

     // Remove duplicates, sort alphabetically
     const uniqueSorted = [...new Set(days)].sort()
     return uniqueSorted.join(',')
   }

   export const deserializeDaysOfWeek = (value: string | null | undefined): DayOfWeek[] | undefined => {
     if (!value || value.trim() === '') return undefined

     const days = value.split(',').map(d => d.trim().toLowerCase()) as DayOfWeek[]

     // Runtime validation
     const validDays: DayOfWeek[] = ['mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun']
     for (const day of days) {
       if (!validDays.includes(day)) {
         throw new Error(`Invalid day in database: ${day}. Must be one of: ${validDays.join(', ')}`)
       }
     }

     return days
   }
   ```

3. **✅ UPDATE** EventTriggerSettingDomain:
   ```typescript
   export class EventTriggerSettingDomain extends BaseDomain {
     // ... existing fields ...

     // REMOVE this:
     // @Expose()
     // @IsOptional()
     // scheduleConstraints?: ScheduleConstraints

     // ADD this instead:
     @Expose()
     @IsArray()
     @IsOptional()
     @IsIn(['mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun'], { each: true })
     @Transform(({ value }) => {
       if (!value) return undefined

       // From database: "mon,wed,fri" → ['mon', 'wed', 'fri']
       if (typeof value === 'string') {
         return deserializeDaysOfWeek(value)
       }

       // From API/DTO: already array
       return value
     })
     allowedDaysOfWeek?: DayOfWeek[]
   }
   ```

**Key Changes**:
- ❌ Remove `ScheduleConstraints` class entirely
- ❌ Remove `scheduleConstraints?: ScheduleConstraints` field
- ✅ Add `allowedDaysOfWeek?: DayOfWeek[]` field directly
- ✅ Simpler API: `setting.allowedDaysOfWeek` instead of `setting.scheduleConstraints?.allowedDaysOfWeek`

---

#### Step 3: Update Repository Layer (🔄 IN PROGRESS)
**File**: `src/repositories/EventTriggerRepository.ts`

**Changes**:

1. **Update INSERT query string** (change column name):
   ```typescript
   private INSERT_EVENT_TRIGGER_SETTING = `
     INSERT INTO
       event_trigger_setting (
         ROBOT_ID,
         EVENT_TYPE_ID,
         MAX_CONCURRENT_TRIGGERS,
         ALLOWED_CONCURRENCY_TIMEFRAME,
         ALLOWED_START_TIME,
         ALLOWED_END_TIME,
         MAX_VALIDITY_DURATION,
         IS_RESCHEDULABLE,
         IS_DEFAULT,
         DEFAULT_SCRIPT_REFERENCE_ID,
         ALLOWED_DAYS_OF_WEEK  -- CHANGED from SCHEDULE_CONSTRAINTS
       )
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
   `
   ```

2. **Update SELECT queries** (change column name in all places):
   ```typescript
   private GET_EVENT_TRIGGER_SETTING = `
     SELECT
         S.ID id,
         S.ROBOT_ID robotId,
         S.EVENT_TYPE_ID eventTypeId,
         S.MAX_CONCURRENT_TRIGGERS maxConcurrentTriggers,
         S.ALLOWED_CONCURRENCY_TIMEFRAME allowedConcurrencyTimeframe,
         S.ALLOWED_START_TIME allowedStartTime,
         S.ALLOWED_END_TIME allowedEndTime,
         S.MAX_VALIDITY_DURATION maxValidityDuration,
         S.IS_RESCHEDULABLE isReschedulable,
         S.IS_DEFAULT isDefault,
         S.DEFAULT_SCRIPT_REFERENCE_ID defaultScriptReferenceId,
         S.ALLOWED_DAYS_OF_WEEK allowedDaysOfWeek,  -- CHANGED from SCHEDULE_CONSTRAINTS
         S.CREATED_AT createdAt,
         S.UPDATED_AT updatedAt
     FROM
       event_trigger_setting S
   `
   ```

3. **Update INSERT - no serialization needed** (DTO already did it):
   ```typescript
   // In createSetting() and createWithSetting()
   const res = await transaction.query<ResultSetHeader>(
     this.INSERT_EVENT_TRIGGER_SETTING,
     [
       setting.robotId ?? null,
       setting.eventTypeId,
       setting.maxConcurrentTriggers ?? null,
       setting.allowedConcurrencyTimeframe ?? null,
       setting.allowedStartTime ?? null,
       setting.allowedEndTime ?? null,
       setting.maxValidityDuration ?? null,
       setting.isReschedulable,
       setting.isDefault,
       setting.defaultScriptReferenceId,
       // OLD: JSON.stringify(setting.scheduleConstraints)
       // NEW: Already serialized by DTO mapper - just pass through
       setting.allowedDaysOfWeek ?? null
     ]
   )
   ```

4. **Update interface** (change to string since DTO serializes it):
   ```typescript
   export interface ICreateEventTriggerSetting {
     robotId?: number
     eventTypeId: number
     maxConcurrentTriggers?: number
     allowedConcurrencyTimeframe?: number
     allowedStartTime?: Date
     allowedEndTime?: Date
     maxValidityDuration?: number
     isReschedulable: boolean
     isDefault: boolean
     defaultScriptReferenceId: number
     // OLD: scheduleConstraints?: domains.ScheduleConstraints
     // NEW: Already serialized CSV string from DTO mapper
     allowedDaysOfWeek?: string | null
   }
   ```

**Summary of Changes**:
- Column name: `SCHEDULE_CONSTRAINTS` → `ALLOWED_DAYS_OF_WEEK`
- Update ALL SELECT queries (3-4 queries) - returns CSV string
- Update INSERT query (1 query) - receives CSV string
- Interface change: array → string (serialization done in DTO layer)
- **No serialization in repository** - just pass through the string from DTO

**Data Flow**:
```
API Request (array) → DTO (array) → toRepositoryRequest() serializes → Repository (string) → Database (VARCHAR)
Database (VARCHAR) → Repository (string) → Domain Transform deserializes → Domain (array) → API Response (array)
```

---

#### Step 4: Update DTO Layer (🔄 IN PROGRESS)
**File**: `src/models/dtos/UpsertTriggerSettingDto.ts`

**Changes**:
1. **Simplify DTO field**:
   ```typescript
   export class UpsertTriggerSettingDto {
     // ... existing fields ...

     // OLD: Remove this
     // @Expose()
     // @IsOptional()
     // scheduleConstraints?: domains.ScheduleConstraints

     // NEW: Add this
     @Expose()
     @IsArray()
     @IsOptional()
     @IsIn(['mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun'], {
       each: true,
       message: 'Invalid day. Must be: mon, tue, wed, thu, fri, sat, sun'
     })
     allowedDaysOfWeek?: domains.DayOfWeek[]
   }
   ```

2. **Update toRepositoryRequest method** (this is the mapper - serialize here):
   ```typescript
   toRepositoryRequest(
     eventSchema: domains.EventSchemaDomain
   ): repositories.ICreateEventTriggerSetting {
     return {
       eventTypeId: eventSchema.id,
       isDefault: true,
       isReschedulable: this.isReschedulable,
       allowedConcurrencyTimeframe: parseTimeframe(this.allowedConcurrencyTimeframe),
       allowedEndTime: parseTimeOnly(this.allowedEndTime),
       allowedStartTime: parseTimeOnly(this.allowedStartTime),
       maxConcurrentTriggers: this.maxConcurrentTriggers,
       maxValidityDuration: this.maxValidityDuration,
       robotId: this.robotId,
       defaultScriptReferenceId: this.defaultScriptReferenceId,
       // OLD: scheduleConstraints: this.scheduleConstraints
       // NEW: Serialize array to CSV string (follows existing pattern)
       allowedDaysOfWeek: domains.serializeDaysOfWeek(this.allowedDaysOfWeek)
     }
   }
   ```

**Note**: `toRepositoryRequest()` acts as a **mapper/transformer** between DTO and Repository layers, similar to how `parseTimeframe()` and `parseTimeOnly()` transform other fields. Serialization belongs here to match the existing pattern.

**API Request Changes**:
```diff
  // OLD API format (nested object)
- {
-   "scheduleConstraints": {
-     "allowedDaysOfWeek": ["mon", "wed", "fri"]
-   }
- }

  // NEW API format (direct array) - BREAKING CHANGE
+ {
+   "allowedDaysOfWeek": ["mon", "wed", "fri"]
+ }
```

**✅ This is acceptable** since the feature hasn't been deployed yet.

---

#### Step 5: Update Service Layer (🔄 IN PROGRESS)
**File**: `src/services/TriggerSchedulerParser.ts`

**Changes**:
```typescript
// In getExpectedExecutedTime()
private getExpectedExecutedTime(
  setting: domains.EventTriggerSettingDomain
): dayjs.Dayjs | null {
  // ... existing time window logic ...

  // Apply day-of-week filtering
  // OLD: if (setting.scheduleConstraints?.allowedDaysOfWeek) {
  // NEW:
  if (setting.allowedDaysOfWeek) {
    return this.findNextAllowedDate(candidateTime, setting)
  }

  return candidateTime
}

private findNextAllowedDate(
  startFrom: dayjs.Dayjs,
  setting: domains.EventTriggerSettingDomain
): dayjs.Dayjs | null {
  // OLD: if (!setting.scheduleConstraints?.allowedDaysOfWeek) {
  // NEW:
  if (!setting.allowedDaysOfWeek) {
    return startFrom
  }

  // OLD: const allowedDays = setting.scheduleConstraints.allowedDaysOfWeek
  // NEW:
  const allowedDays = setting.allowedDaysOfWeek

  // ... rest of logic unchanged ...
}
```

---

#### Step 6: Update Tests (🔄 IN PROGRESS)

##### A. Unit Tests for Serialization (🚧 TODO)
**New File**: `test/models/domains/EventTriggerDomainTest.ts`

```typescript
import { expect } from 'chai'
import { serializeDaysOfWeek, deserializeDaysOfWeek, DayOfWeek } from '../../../src/models/domains/EventTriggerDomain'

describe('Day of Week Serialization', () => {
  describe('serializeDaysOfWeek', () => {
    it('should serialize and sort consistently', () => {
      expect(serializeDaysOfWeek(['fri', 'mon', 'wed']))
        .to.equal('fri,mon,wed')
    })

    it('should handle duplicates', () => {
      expect(serializeDaysOfWeek(['mon', 'mon', 'wed']))
        .to.equal('mon,wed')
    })

    it('should return null for empty array', () => {
      expect(serializeDaysOfWeek([])).to.be.null
    })

    it('should return null for undefined', () => {
      expect(serializeDaysOfWeek(undefined)).to.be.null
    })
  })

  describe('deserializeDaysOfWeek', () => {
    it('should deserialize comma-separated string', () => {
      expect(deserializeDaysOfWeek('mon,wed,fri'))
        .to.deep.equal(['mon', 'wed', 'fri'])
    })

    it('should return undefined for null', () => {
      expect(deserializeDaysOfWeek(null)).to.be.undefined
    })

    it('should return undefined for empty string', () => {
      expect(deserializeDaysOfWeek('')).to.be.undefined
    })

    it('should reject invalid day names', () => {
      expect(() => {
        deserializeDaysOfWeek('monday,wednesday')
      }).to.throw('Invalid day in database: monday')
    })

    it('should handle whitespace', () => {
      expect(deserializeDaysOfWeek(' mon , wed , fri '))
        .to.deep.equal(['mon', 'wed', 'fri'])
    })
  })
})
```

##### B. Update Existing Tests
**File**: `test/services/TriggerSchedulerParserTest.ts`

**Actions**:
- ✅ Update test data to use new field structure:
  ```typescript
  // OLD:
  // setting.scheduleConstraints = {
  //   allowedDaysOfWeek: ['mon', 'wed', 'fri']
  // }

  // NEW:
  setting.allowedDaysOfWeek = ['mon', 'wed', 'fri']
  ```

**File**: `test/repositories/EventTriggersRepositoryTest.ts`

**Actions**:
- Add integration tests for VARCHAR column handling
- Test NULL, empty string, valid CSV, invalid CSV
- Update existing tests that reference `scheduleConstraints`

---

### Phase 4: Testing & Validation

#### Integration Test Checklist (🚧 TODO)
- [ ] Test creating setting with `allowedDaysOfWeek: ['mon', 'wed', 'fri']`
  - Verify database stores: `"fri,mon,wed"` (sorted alphabetically, no quotes)
- [ ] Test retrieving setting and verify correct deserialization
- [ ] Test NULL `allowed_days_of_week` (no constraints = any day allowed)
- [ ] Test empty array serialization (should store NULL)
- [ ] Test duplicate days handling (e.g., ['mon', 'mon', 'wed'] → 'mon,wed')
- [ ] Test invalid day names (should fail validation at DTO level)
- [ ] Test end-to-end trigger creation with day constraints
- [ ] Verify all SELECT queries return correct field

#### Manual Testing Steps (🚧 TODO)
1. **Verify old column removed**:
   ```sql
   -- Should return 0 rows
   SELECT COLUMN_NAME
   FROM INFORMATION_SCHEMA.COLUMNS
   WHERE TABLE_NAME = 'event_trigger_setting'
     AND COLUMN_NAME = 'schedule_constraints';
   ```

2. **Run migration**:
   ```bash
   # No data migration needed - feature not deployed yet
   mysql -u root -p tinybots < ci/db/migrations/V94__replace_schedule_constraints_with_allowed_days.sql
   ```

3. **Verify new column**:
   ```sql
   -- Should show VARCHAR(100)
   SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, COLUMN_COMMENT
   FROM INFORMATION_SCHEMA.COLUMNS
   WHERE TABLE_NAME = 'event_trigger_setting'
     AND COLUMN_NAME = 'allowed_days_of_week';
   ```

4. **API Test** (Updated format):
   ```bash
   curl -X PUT http://localhost:8080/v1/triggers/settings \
     -H "Content-Type: application/json" \
     -d '{
       "eventName": "event.test",
       "allowedStartTime": "08:00",
       "allowedEndTime": "20:00",
       "isReschedulable": true,
       "allowedDaysOfWeek": ["mon", "wed", "fri"],
       "maxConcurrentTriggers": 3,
       "allowedConcurrencyTimeframe": "5m",
       "defaultScriptReferenceId": 123
     }'
   ```

5. **Verify database value**:
   ```sql
   -- Should show: "fri,mon,wed" (sorted alphabetically)
   SELECT id, allowed_days_of_week
   FROM event_trigger_setting
   WHERE allowed_days_of_week IS NOT NULL;
   ```

---

## 📊 Summary of Results

### ✅ Completed Achievements
- [ ] Database migration created and tested
- [ ] Domain model simplified (removed ScheduleConstraints class)
- [ ] Repository layer updated (column name + VARCHAR serialization)
- [ ] Service layer updated (direct field access)
- [ ] DTO layer simplified (direct array field)
- [ ] Unit tests added for day serialization helpers
- [ ] Integration tests updated
- [ ] All existing tests updated to new structure
- [ ] Documentation updated

### Key Benefits
- ✅ **Simpler Code**: No nested `scheduleConstraints.allowedDaysOfWeek`, just `allowedDaysOfWeek`
- ✅ **Better Database Schema**: VARCHAR(100) is more efficient than JSON for short strings
- ✅ **Better Performance**: VARCHAR can be indexed, stored inline with row
- ✅ **Cleaner Storage**: `"fri,mon,wed"` (31 chars) vs `{"allowedDaysOfWeek":["fri","mon","wed"]}` (45 chars)
- ✅ **Consistent Ordering**: Automatic alphabetical sorting prevents duplicates
- ✅ **Cleaner API**: Direct array instead of nested object
- ✅ **No Migration Complexity**: Feature not deployed yet, clean slate

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues
- [ ] **BREAKING CHANGE**: API format changed (acceptable - not deployed yet)
  - Old: `{"scheduleConstraints": {"allowedDaysOfWeek": [...]}}`
  - New: `{"allowedDaysOfWeek": [...]}`
- [ ] Need to update all test fixtures to new format
- [ ] Consider adding CHECK constraint for MySQL 8.0.16+ (optional)

### 🔮 Future Improvements
- [ ] Add CHECK constraint for valid day names (MySQL 8.0.16+):
  ```sql
  ALTER TABLE event_trigger_setting
  ADD CONSTRAINT check_allowed_days_format
  CHECK (
    allowed_days_of_week IS NULL OR
    allowed_days_of_week REGEXP '^(fri|mon|sat|sun|thu|tue|wed)(,(fri|mon|sat|sun|thu|tue|wed))*$'
  );
  ```
- [ ] Add database index if filtering by specific days becomes common:
  ```sql
  CREATE INDEX idx_allowed_days ON event_trigger_setting(allowed_days_of_week);
  ```
- [ ] Consider future constraint types (keep as **separate columns** for simplicity):
  - `allowed_days_of_month VARCHAR(100)` for "1,15,30"
  - `excluded_dates TEXT` for "2025-12-25,2026-01-01"
  - Each field is independent, simpler to query/index

### Migration Risk Assessment
**Risk Level**: � **Low** (feature not deployed yet)
- **Data Loss Risk**: None (no existing data to migrate)
- **Downtime Required**: No (simple column add/drop)
- **Rollback Complexity**: Low (just revert code + restore old column)

### Rollback Plan (if needed)
1. Revert code changes (git revert)
2. Run rollback SQL:
   ```sql
   ALTER TABLE event_trigger_setting DROP COLUMN allowed_days_of_week;
   ALTER TABLE event_trigger_setting ADD COLUMN schedule_constraints JSON DEFAULT NULL;
   ```

---

## 🔗 References
- **Original Implementation**: `devdocs/tinybots/m-o-triggers/251103-trigger-configuration.md`
- **Domain Model**: `src/models/domains/EventTriggerDomain.ts`
- **Repository**: `src/repositories/EventTriggerRepository.ts`
- **API DTO**: `src/models/dtos/UpsertTriggerSettingDto.ts`
- **Tests**: `test/services/TriggerSchedulerParserTest.ts`

---

**Last updated**: 2025-11-07
**Status**: 🔄 Implementation in Progress
**Next Steps**: Create migration script and unit tests for ScheduleConstraints serialization
