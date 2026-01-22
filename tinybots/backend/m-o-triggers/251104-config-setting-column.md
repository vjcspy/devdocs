# Schedule Constraints Storage Options - Technical Decision Document

**Date**: 2024-11-04
**Status**: Decision Pending
**Context**: Implementation of `schedule_constraints` feature for trigger settings

---

## Executive Summary

This document presents 4 viable approaches for storing schedule constraint configuration data, with objective analysis of trade-offs. The decision criteria prioritize:

1. **Database Independence**: Database schema should be self-documenting without service knowledge
2. **Data Validation**: Database should enforce business rules independently
3. **Type Safety**: Database should prevent invalid data at storage level
4. **Maintainability**: Future schema evolution should be manageable
5. **Development Effort**: Implementation and maintenance cost

---

## Requirements

### Current (Phase 1)
- Store `allowedDaysOfWeek`: array of day names (e.g., `["mon", "wed", "fri"]`)
- Values must be one of: `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun`

### Future (Phase 2 - I'm not sure, but I'll assume so)
- `allowedDaysOfMonth`: array of day numbers (e.g., `[1, 15, 30]`)
- `excludedDates`: array of specific dates (holiday,...) (e.g., `["2025-12-25", "2025-01-01"]`)
- `allowedWeeksOfMonth`: array of week numbers (e.g., `[1, 3]` for 1st and 3rd week)

---

## Option 1: JSON Column with Schema Validation

### Implementation

#### Phase 1: allowedDaysOfWeek

```sql
ALTER TABLE event_trigger_settings
ADD COLUMN schedule_constraints JSON,
ADD CONSTRAINT check_schedule_constraints_schema
CHECK (
  schedule_constraints IS NULL OR
  (
    JSON_VALID(schedule_constraints) AND
    JSON_SCHEMA_VALID('{
      "type": "object",
      "required": [],
      "additionalProperties": false,
      "properties": {
        "allowedDaysOfWeek": {
          "type": "array",
          "items": {
            "type": "string",
            "enum": ["mon","tue","wed","thu","fri","sat","sun"]
          }
        }
      }
    }', schedule_constraints)
  )
);

-- Example data
INSERT INTO event_trigger_settings (schedule_constraints) VALUES
('{"allowedDaysOfWeek":["mon","wed","fri"]}');
```

#### Phase 2: Adding More Constraint Types

```sql
-- Update CHECK constraint (no data migration needed)
ALTER TABLE event_trigger_settings
DROP CONSTRAINT check_schedule_constraints_schema,
ADD CONSTRAINT check_schedule_constraints_schema
CHECK (
  schedule_constraints IS NULL OR
  JSON_SCHEMA_VALID('{
    "type": "object",
    "properties": {
      "allowedDaysOfWeek": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": ["mon","tue","wed","thu","fri","sat","sun"]
        }
      },
      "allowedDaysOfMonth": {
        "type": "array",
        "items": {"type": "integer", "minimum": 1, "maximum": 31}
      },
      "excludedDates": {
        "type": "array",
        "items": {"type": "string", "pattern": "^\\d{4}-\\d{2}-\\d{2}$"}
      }
    }
  }', schedule_constraints)
);

-- Old data still valid (optional fields)
-- New data can include new fields
```

### Key Points

**Database-Level Validation**:

- ✅ JSON syntax is valid
- ✅ Structure matches schema (object with specific properties)
- ✅ Field types are correct (array of strings)
- ✅ Values are in allowed enum (`mon`-`sun` only)
- ✅ No extra fields allowed (`additionalProperties: false`)

**What happens on invalid data**:
```sql
-- Invalid JSON syntax
INSERT INTO event_trigger_settings (schedule_constraints)
VALUES ('{invalid}');
-- Error: Invalid JSON text

-- Invalid value
INSERT INTO event_trigger_settings (schedule_constraints)
VALUES ('{"allowedDaysOfWeek":["monday"]}');
-- Error: JSON schema validation failed

-- Wrong type
INSERT INTO event_trigger_settings (schedule_constraints)
VALUES ('{"allowedDaysOfWeek":"mon,wed,fri"}');
-- Error: Expected array, got string

-- Extra fields
INSERT INTO event_trigger_settings (schedule_constraints)
VALUES ('{"allowedDaysOfWeek":["mon"],"extraField":"value"}');
-- Error: Additional properties not allowed
```

### Self-Documentation

**Reading directly from database**:

```sql
SELECT id, robot_id, schedule_constraints
FROM event_trigger_settings;

-- Result is human-readable:
-- {"allowedDaysOfWeek":["mon","wed","fri"]}
```

**Schema is in database**:
```sql
-- Anyone can see the schema without reading service code
SHOW CREATE TABLE event_trigger_settings;
-- Shows CHECK constraint with full JSON schema
```

**Query capabilities**:
```sql
-- Find all settings that do NOT run on Saturday
SELECT id, robot_id, schedule_constraints
FROM event_trigger_settings
WHERE schedule_constraints IS NOT NULL
  AND NOT JSON_CONTAINS(
    schedule_constraints->'$.allowedDaysOfWeek',
    '"sat"'
  );

-- Find all settings that run on Monday
SELECT id, robot_id, schedule_constraints
FROM event_trigger_settings
WHERE JSON_CONTAINS(
  schedule_constraints->'$.allowedDaysOfWeek',
  '"mon"'
);
```

### Pros

1. **Database Independence**
   - ✅ Schema is defined in database (CHECK constraint)
   - ✅ Validation happens at database level
   - ✅ Self-documenting (JSON is human-readable)
   - ✅ Works with any programming language

2. **Data Validation**
   - ✅ Database enforces allowed values
   - ✅ Database enforces data types
   - ✅ Database enforces structure
   - ✅ Invalid data rejected at INSERT time

3. **Type Safety**
   - ✅ Enum values enforced
   - ✅ Array type enforced
   - ✅ No free-form JSON (schema restricted)

4. **Maintainability**
   - ✅ Zero data migrations for new fields
   - ✅ Backward compatible (old data valid)
   - ✅ Single source of truth (CHECK constraint)

5. **Development Effort**
   - ✅ One-time setup
   - ✅ No manual serialization code
   - ✅ Native MySQL support

### Cons

1. **Database Independence Concerns**
   - ⚠️ Requires understanding of JSON schema syntax (though schema is in DB)
   - ⚠️ CHECK constraint syntax is MySQL-specific
2. **Complexity**
   - ⚠️ JSON schema validation requires MySQL 8.0.17+ [LINK](https://dev.mysql.com/doc/refman/8.0/en/json-validation-functions.html)
4. **Perception**
   - ⚠️ May be perceived as "too flexible" despite schema enforcement
   - ⚠️ Requires team buy-in on JSON approach


---

## Option 2: VARCHAR Column with Delimited String

### Implementation

#### Phase 1: allowedDaysOfWeek

```sql
ALTER TABLE event_trigger_settings
ADD COLUMN allowed_days_of_week VARCHAR(100);

-- Example data
INSERT INTO event_trigger_settings (allowed_days_of_week)
VALUES ('mon,wed,fri');
```

#### Phase 2: Adding More Constraint Types

```sql
-- Need new migration for each field
ALTER TABLE event_trigger_settings
ADD COLUMN allowed_days_of_month VARCHAR(200);  -- "1,15,30"

ALTER TABLE event_trigger_settings
ADD COLUMN excluded_dates TEXT;  -- "2025-12-25,2025-01-01"

-- Each needs its own CHECK constraint if validation desired
ADD CONSTRAINT check_days_of_month_format
CHECK (
  allowed_days_of_month IS NULL OR
  allowed_days_of_month REGEXP '^[1-9][0-9]?(,[1-9][0-9]?)*$'
);
```

### Key Points

**Database-Level Enforcement**:
- ✅ String length <= 100 characters

**What the database validates**:
- ✅ String length <= 100 characters

**What the database does NOT validate**:
- ❌ Valid day names (could store "abc,xyz,123" without CHECK)
- ❌ No duplicates (could store "mon,mon,mon")
- ❌ No empty strings vs NULL distinction


### Self-Documentation

**Reading directly from database**:
```sql
SELECT id, robot_id, allowed_days_of_week
FROM event_trigger_settings;

-- Result:
-- "mon,wed,fri"

-- Questions a DBA might have:
-- - Is this comma-separated or another delimiter?
-- - Are these full names or abbreviations?
-- - Is empty string = no days or all days?
-- - What happens if I see "Mon,Wed,Fri" vs "mon,wed,fri"?
```

- ⚠️ Must document format in external docs
- ⚠️ Not self-evident from database alone
- ⚠️ Future developers must read service code to understand

**Query capabilities**:
```sql
-- Find all settings that do NOT run on Saturday
-- Must use REGEXP or FIND_IN_SET (limited)
SELECT id, robot_id, allowed_days_of_week
FROM event_trigger_settings
WHERE allowed_days_of_week IS NOT NULL
  AND NOT FIND_IN_SET('sat', allowed_days_of_week);

-- FIND_IN_SET is slower and less flexible than JSON_CONTAINS
```

### Pros

1. **Simplicity (for Phase 1)**
   - ✅ Simple to understand initially
   - ✅ No JSON knowledge required
   - ✅ Direct string storage


### Cons

1. **Database Independence**
   - ❌ Format not defined in database
   - ❌ Requires external documentation
   - ❌ Parsing rules exist only in service code

2. **Data Validation**
   - ❌ Database doesn't enforce valid values (without complex REGEXP)
   - ❌ Can store invalid data silently
   - ❌ REGEXP validation is slow and incomplete

3. **Type Safety**
   - ❌ Everything is a string
   - ❌ No type distinction between "mon,wed" and "1,3"
   - ❌ Easy to corrupt data with wrong format

4. **Maintainability**
   - ❌ Each new field requires migration
   - ❌ 3 separate migrations for Phase 2
   - ❌ Parsing logic must be maintained in service

5. **Development Effort**
   - ❌ Must write serialization/deserialization code
   - ❌ Must update parsing for each new field
   - ❌ Must maintain REGEXP patterns

6. **Error Scenarios**
   - ❌ Junior developer uses "mon|wed|fri" instead of commas → silent bug
   - ❌ Junior uses "Monday" instead of "mon" → silent bug
   - ❌ Empty string vs NULL ambiguity
   - ❌ Errors only caught at runtime in service

---

## Option 3: TEXT Column Storing JSON String

### Implementation

#### Phase 1: allowedDaysOfWeek

```sql
ALTER TABLE event_trigger_settings
ADD COLUMN schedule_constraints TEXT;

-- Example data
INSERT INTO event_trigger_settings (schedule_constraints)
VALUES ('{"allowedDaysOfWeek":["mon","wed","fri"]}');
```

#### Phase 2: Adding More Constraint Types

```sql
-- Need new TEXT column for each constraint type
ALTER TABLE event_trigger_settings
ADD COLUMN allowed_days_of_month_constraints TEXT;  -- '{"allowedDaysOfMonth":[1,15,30]}'

ALTER TABLE event_trigger_settings
ADD COLUMN excluded_dates_constraints TEXT;  -- '{"excludedDates":["2025-12-25","2025-01-01"]}'

-- Each column needs manual JSON serialization in application code
-- No database validation possible
```

### Key Points

**Database-Level Enforcement**:
- ✅ Text storage (accepts any string)

**What the database does NOT validate**:
- ❌ JSON syntax
- ❌ JSON structure
- ❌ Field types
- ❌ Allowed values

**Cannot add validation**:
- ❌ MySQL doesn't have `JSON_VALID()` for TEXT columns
- ❌ Would need to convert to JSON type to validate

**Self-Documentation**:
```sql
SELECT id, robot_id, schedule_constraints
FROM event_trigger_settings;

-- Result:
-- {"allowedDaysOfWeek":["mon","wed","fri"]}
```

Human-readable like JSON, but:
- ⚠️ No guarantee it's actually valid JSON
- ⚠️ Could contain: `{invalid json`, `"just a string"`, `null`, etc.
- ⚠️ Schema not documented anywhere

**Query capabilities**:
```sql
-- Cannot query reliably
-- TEXT columns don't support JSON functions
SELECT id, robot_id, schedule_constraints
FROM event_trigger_settings
WHERE schedule_constraints LIKE '%"sat"%';  -- Unreliable and slow
```

### Pros

1. **Simplicity (Initial)**
   - ✅ No special MySQL features required
   - ✅ Works with older MySQL versions

### Cons

1. **Database Independence**
   - ❌ No schema defined anywhere
   - ❌ Database accepts any string (even invalid JSON)
   - ❌ No validation at database level

2. **Data Validation**
   - ❌ Can insert `{invalid` and database accepts it
   - ❌ Can insert `"this is not json"` and database accepts it
   - ❌ Errors only discovered at read time in service

3. **Type Safety**
   - ❌ Zero type safety
   - ❌ Worse than VARCHAR (at least VARCHAR has length limit)
   - ❌ Complete free-form text

4. **Maintainability**
   - ❌ Each new constraint type = new TEXT column (same as Option 2)
   - ❌ Phase 2 requires 2-3 new columns
   - ❌ Double effort: must handle JSON parsing AND multiple columns

5. **Worst of Both Worlds**
   - ❌ Lacks validation of JSON column (Option 1)
   - ❌ Lacks simplicity of VARCHAR (Option 2)
   - ❌ Combines worst aspects: no validation + multiple migrations
   - ❌ Requires JSON parsing like JSON column
   - ❌ No performance benefits (must parse on read)

6. **Error Scenarios**
   - ❌ Junior inserts invalid JSON → database accepts → runtime error
   - ❌ JSON syntax error → silent corruption
   - ❌ No protection against bad data

### Assessment

**This option is strictly worse than Option 1 (native JSON)**:
- Same code complexity
- Same data format
- But ZERO database validation
- No advantages over native JSON

**Recommendation**: If considering this, use Option 1 instead.

---

## Option 4: Relational Tables

### Implementation

#### Phase 1: allowedDaysOfWeek

```sql
-- Main settings table (no changes for schedule_constraints)
CREATE TABLE event_trigger_settings (
  id INT PRIMARY KEY AUTO_INCREMENT,
  robot_id INT,
  event_type_id INT,
  -- ... other fields
);

-- Separate table for allowed days
CREATE TABLE schedule_constraint_allowed_days (
  setting_id INT NOT NULL,
  day_of_week ENUM('mon','tue','wed','thu','fri','sat','sun') NOT NULL,
  PRIMARY KEY (setting_id, day_of_week),
  FOREIGN KEY (setting_id)
    REFERENCES event_trigger_settings(id)
    ON DELETE CASCADE
);

-- Example data
INSERT INTO event_trigger_settings (id, robot_id) VALUES (1, 199);
INSERT INTO schedule_constraint_allowed_days (setting_id, day_of_week) VALUES
(1, 'mon'),
(1, 'wed'),
(1, 'fri');
```

#### Phase 2: Adding More Constraint Types

```sql
-- New table for days of month
CREATE TABLE schedule_constraint_allowed_days_of_month (
  setting_id INT NOT NULL,
  day_of_month TINYINT NOT NULL,
  PRIMARY KEY (setting_id, day_of_month),
  FOREIGN KEY (setting_id) REFERENCES event_trigger_settings(id),
  CHECK (day_of_month BETWEEN 1 AND 31)
);

-- New table for excluded dates
CREATE TABLE schedule_constraint_excluded_dates (
  setting_id INT NOT NULL,
  excluded_date DATE NOT NULL,
  PRIMARY KEY (setting_id, excluded_date),
  FOREIGN KEY (setting_id) REFERENCES event_trigger_settings(id)
);

-- Complex query to get all constraints
SELECT
  s.id,
  GROUP_CONCAT(DISTINCT d.day_of_week) as allowed_days,
  GROUP_CONCAT(DISTINCT dm.day_of_month) as allowed_days_of_month,
  GROUP_CONCAT(DISTINCT ed.excluded_date) as excluded_dates
FROM event_trigger_settings s
LEFT JOIN schedule_constraint_allowed_days d ON s.id = d.setting_id
LEFT JOIN schedule_constraint_allowed_days_of_month dm ON s.id = dm.setting_id
LEFT JOIN schedule_constraint_excluded_dates ed ON s.id = ed.setting_id
GROUP BY s.id;
```

### Key Points

**Database-Level Enforcement**:
- ✅ ENUM enforces exact values (only `mon`-`sun`)
- ✅ PRIMARY KEY prevents duplicates
- ✅ FOREIGN KEY enforces referential integrity
- ✅ Type safety (ENUM is strictly typed)

### Self-Documentation

**Reading directly from database**:
```sql
-- Simple query
SELECT * FROM schedule_constraint_allowed_days WHERE setting_id = 1;

-- Result:
-- setting_id | day_of_week
-- 1          | mon
-- 1          | wed
-- 1          | fri

-- With setting details
SELECT
  s.id,
  s.robot_id,
  GROUP_CONCAT(d.day_of_week ORDER BY
    FIELD(d.day_of_week, 'mon','tue','wed','thu','fri','sat','sun')
  ) as allowed_days
FROM event_trigger_settings s
LEFT JOIN schedule_constraint_allowed_days d ON s.id = d.setting_id
GROUP BY s.id;

-- Result:
-- id | robot_id | allowed_days
-- 1  | 199      | mon,wed,fri
```

- ✅ ENUM values visible in schema: `SHOW CREATE TABLE schedule_constraint_allowed_days`
- ✅ Foreign key relationships clear
- ✅ Standard relational design

**Query capabilities**:
```sql
-- Find all settings that do NOT run on Saturday
SELECT DISTINCT s.id, s.robot_id
FROM event_trigger_settings s
WHERE s.id NOT IN (
  SELECT setting_id
  FROM schedule_constraint_allowed_days
  WHERE day_of_week = 'sat'
);

-- Or with LEFT JOIN
SELECT s.id, s.robot_id
FROM event_trigger_settings s
LEFT JOIN schedule_constraint_allowed_days d
  ON s.id = d.setting_id AND d.day_of_week = 'sat'
WHERE d.setting_id IS NULL;
```

### Pros

1. **Database Independence** ⭐
   - ✅ Pure relational design
   - ✅ No special features required
   - ✅ Schema is completely self-documenting
   - ✅ Works with any database

2. **Data Validation** ⭐
   - ✅ Maximum type safety (ENUM)
   - ✅ Foreign key integrity
   - ✅ No invalid values possible
   - ✅ Database enforces all rules

3. **Type Safety** ⭐
   - ✅ ENUM is strictest possible type
   - ✅ Impossible to store invalid day
   - ✅ Impossible to have duplicates

4. **Traditional Approach**
   - ✅ Familiar to all DBAs
   - ✅ Standard SQL patterns
   - ✅ Clear relationships

5. **Query Flexibility**
   - ✅ Can query individual days easily
   - ✅ Can join with other tables
   - ✅ Standard SQL operators

### Cons

1. **Complexity**
   - ❌ Multiple tables to manage
   - ❌ More complex queries (JOINs, GROUP_CONCAT)
   - ❌ More moving parts

2. **Maintainability**
   - ❌ Each new constraint type = new table (overkill for extensibility)
   - ❌ Phase 2 requires 3 new tables (3x database objects to manage)
   - ❌ Query complexity grows with each table (multiple JOINs)
   - ❌ Future constraint types (time windows, week numbers, etc.) = more tables

3. **Development Effort**
   - ❌ More code to manage relationships
   - ❌ More complex insert/update logic
   - ❌ Must handle cascading deletes

4. **Performance Considerations**
   - ⚠️ More JOINs for reads
   - ⚠️ More rows to manage (1 row per day vs 1 row per setting)
   - ⚠️ GROUP_CONCAT has length limits

5. **Practical Overhead**
   - ❌ May be "overkill" for simple array storage
   - ❌ 3x more tables for same functionality
   - ❌ More migration scripts to maintain

---

## Conclusion

| Criteria | Option 1: JSON + Schema | Option 2: VARCHAR | Option 3: TEXT (JSON string) | Option 4: Relational Tables |
|----------|------------------------|-------------------|-----------------------------|-----------------------------|
| **Database Independence** | | | | |
| Schema in database | ✅ Yes (CHECK constraint) | ⚠️ Partial (REGEXP) | ❌ No | ✅ Yes (ENUM + FK) |
| Self-documenting | ✅ Yes (readable JSON) | ⚠️ Format unclear | ⚠️ Format unclear | ✅ Yes (tables) |
| No service dependency | ✅ Schema in DB | ❌ Parse rules in service | ❌ Parse rules in service | ✅ Pure SQL |
| **Data Validation** | | | | |
| Database enforces values | ✅ Enum in schema | ⚠️ REGEXP (weak) | ❌ No | ✅ ENUM type |
| Database enforces types | ✅ Array, string, etc. | ❌ All strings | ❌ No | ✅ ENUM, INT, DATE |
| Database enforces structure | ✅ JSON schema | ❌ No | ❌ No | ✅ Foreign keys |
| Invalid data rejected | ✅ At INSERT | ⚠️ If REGEXP added | ❌ Accepted | ✅ At INSERT |
| **Type Safety** | | | | |
| Strict typing | ✅ JSON schema | ❌ String only | ❌ Text only | ✅ ENUM/types |
| Prevent invalid values | ✅ Yes | ⚠️ Weak | ❌ No | ✅ Yes |
| Prevent duplicates | ✅ Logic in schema | ❌ No | ❌ No | ✅ Primary key |
| **Maintainability** | | | | |
| Phase 2 migrations needed | ✅ 0 (update constraint) | ❌ 3 (new columns) | ❌ 2-3 (new TEXT columns) | ❌ 3 (new tables) |
| Code changes per new field | ✅ Minimal (type def) | ❌ Moderate (serialize) | ❌ High (serialize + parse) | ❌ High (relations) |
| Backward compatible | ✅ Yes (optional) | ⚠️ Manual handling | ⚠️ Manual handling | ⚠️ Must handle nulls |
| **Complexity** | | | | |
| DBA complexity | ⚠️ JSON schema syntax | ✅ Simple | ✅ Simple | ⚠️ Multiple tables |
| Developer complexity | ✅ Native support | ❌ Manual parsing | ❌ Manual parsing | ❌ Complex queries |
| Junior developer safety | ✅ DB validates | ❌ Silent bugs | ❌ Silent bugs | ✅ DB validates |
| **Querying** | | | | |
| Read single setting | ✅ Simple SELECT | ✅ Simple SELECT | ✅ Simple SELECT | ⚠️ JOIN required |
| Query by constraint | ✅ JSON_CONTAINS | ❌ REGEXP (slow) | ❌ Can't query | ✅ WHERE on joined table |
| Find settings NOT running on Saturday | ✅ JSON_CONTAINS | ⚠️ FIND_IN_SET | ❌ LIKE (unreliable) | ✅ LEFT JOIN/NOT IN |
| Analytical queries | ✅ JSON functions | ❌ Hard | ❌ Hard | ✅ Standard SQL |
| **Storage** | | | | |
| Size per row (typical) | 44 bytes | 12 bytes ✅ | 47 bytes | ~30 bytes (3 rows) |
| 1M records | 44 MB | 12 MB ✅ | 47 MB | 30 MB |
| Cost difference ($/year) | $0.06 | $0.016 | $0.064 | $0.04 |
| **Risk Assessment** | | | | |
| Invalid data risk | ✅ Low (DB rejects) | ❌ High (silent bugs) | ❌ Very high | ✅ Low (DB rejects) |
| Future extension risk | ✅ Low (no migration) | ❌ High (migrations) | ✅ Low | ⚠️ Medium (new tables) |
| Team learning curve | ⚠️ Medium (JSON schema) | ✅ Low | ✅ Low | ✅ Low |

---

## These are some questions taken from Vy/Arno

### Concern 1: "Database should be clear without service knowledge"

| Option | Addresses Concern? | Explanation |
|--------|-------------------|-------------|
| **Option 1** | ✅ Yes | CHECK constraint contains full JSON schema visible via `SHOW CREATE TABLE` |
| **Option 2** | ❌ No | Format rules exist only in service code; REGEXP doesn't document semantics |
| **Option 3** | ❌ No | No schema anywhere; complete free-form |
| **Option 4** | ✅ Yes | Schema is pure SQL tables with ENUM, self-documenting |

### Concern 2: "Database validates business rules independently"

| Option | Addresses Concern? | Explanation |
|--------|-------------------|-------------|
| **Option 1** | ✅ Yes | JSON schema enforces: allowed values, types, structure, no extra fields |
| **Option 2** | ⚠️ Partial | Can add REGEXP but incomplete (can't prevent duplicates, validate semantics) |
| **Option 3** | ❌ No | Database accepts any text, zero validation |
| **Option 4** | ✅ Yes | ENUM, CHECK constraints, foreign keys enforce all rules |

### Concern 3: "Future juniors might make mistakes - strict typing prevents this"

| Option | Junior Mistake Scenario | What Happens? |
|--------|------------------------|---------------|
| **Option 1** | Inserts `{"allowedDaysOfWeek":["monday"]}` | ❌ **Database rejects**: "JSON schema validation failed" |
| **Option 2** | Inserts `"monday,wednesday,friday"` | ✅ **Database accepts** → Silent bug in production |
| **Option 3** | Inserts `{invalid json syntax` | ✅ **Database accepts** → Runtime error when read |
| **Option 4** | Tries to insert `('monday')` into ENUM | ❌ **Database rejects**: "Invalid enum value" |


### Concern 4: "Speed of development hindered by mistakes without knowing cause"

| Option | Time to Discover Bug | Where Bug is Caught |
|--------|---------------------|---------------------|
| **Option 1** | Immediately at INSERT | ✅ Database error message with schema details |
| **Option 2** | At runtime in service | ❌ Silent data corruption, discovered later in logs |
| **Option 3** | At runtime when parsing | ❌ JSON parse error, unclear what's wrong |
| **Option 4** | Immediately at INSERT | ✅ Database ENUM/FK error |


### Concern 5: "JSON invalidates strict typing rules completely"

**Response**: This is true for **free-form JSON** (Option 3), but NOT for **JSON with schema validation** (Option 1).

**Option 1 with schema validation**:
- ✅ Strictly typed (schema defines exact structure)
- ✅ Database enforces types
- ✅ More strict than VARCHAR (which accepts any string)
- ✅ Comparable to ENUM table (Option 4) in strictness

**The key difference**:
- Free-form JSON = untyped
- JSON + schema validation = strictly typed
- Option 1 is the latter


---

## If We Must Choose Text/VARCHAR Approach

### Recommended: Enhanced VARCHAR with Validation

Combine the simplicity of Option 2 with stronger safeguards:


### Application-Level Safeguards

Since database validation is limited, add strong validation in application:

```typescript
// Domain validation
export class ScheduleConstraints {
  @Expose()
  @IsArray()
  @ArrayNotEmpty({ message: 'allowedDaysOfWeek cannot be empty' })
  @ArrayUnique({ message: 'Duplicate days not allowed' })
  @IsIn(['mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun'], {
    each: true,
    message: 'Invalid day. Must be: mon, tue, wed, thu, fri, sat, sun'
  })
  allowedDaysOfWeek!: DayOfWeek[]

  // Serialize for database
  toDbString(): string {
    return this.allowedDaysOfWeek.sort().join(',')
  }

  // Deserialize from database
  static fromDbString(value: string | null): DayOfWeek[] | undefined {
    if (!value) return undefined

    const days = value.split(',') as DayOfWeek[]

    // Runtime validation
    const validDays: DayOfWeek[] = ['mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun']
    for (const day of days) {
      if (!validDays.includes(day)) {
        throw new Error(`Invalid day in database: ${day}`)
      }
    }

    return days
  }
}
```

### Key Principles

1. **Always Sort Before Save**
   ```typescript
   // Prevents "mon,wed,fri" vs "fri,mon,wed" duplication
   toDbString(): string {
     return this.allowedDaysOfWeek.sort().join(',')
   }
   ```

2. **Unit Test the Serialization**
   ```typescript
   describe('ScheduleConstraints', () => {
     it('should serialize consistently', () => {
       const constraint = new ScheduleConstraints()
       constraint.allowedDaysOfWeek = ['fri', 'mon', 'wed']

       expect(constraint.toDbString()).to.equal('fri,mon,wed')
     })

     it('should reject invalid days from database', () => {
       expect(() => {
         ScheduleConstraints.fromDbString('monday,wednesday')
       }).to.throw('Invalid day in database')
     })
   })
   ```

3. **Document the Format**
   ```sql
   -- Add column comment
   ALTER TABLE event_trigger_settings
   MODIFY allowed_days_of_week VARCHAR(27)
   COMMENT 'Comma-separated day codes (mon,tue,wed,thu,fri,sat,sun), sorted alphabetically';
   ```

4. **Add Integration Test for Database Constraint**
   ```typescript
   it('should reject invalid format at database level', async () => {
     await expect(
       db.query(
         'INSERT INTO event_trigger_settings (allowed_days_of_week) VALUES (?)',
         ['monday,wednesday']  // Invalid
       )
     ).to.be.rejectedWith(/check constraint/)
   })
   ```

### Why This Approach is Better Than Plain VARCHAR

| Aspect | Plain VARCHAR (Option 2) | Enhanced VARCHAR | Improvement |
|--------|-------------------------|------------------|-------------|
| Database validation | ❌ None | ✅ REGEXP + length | Catches format errors |
| Application validation | ⚠️ Basic | ✅ Comprehensive | Prevents bugs early |
| Duplicates | ❌ Allowed | ✅ Prevented by sort | Data consistency |
| Documentation | ❌ External only | ✅ Column comments | Self-documenting |
| Error discovery | ❌ Runtime | ✅ Insert time (DB) + Save time (app) | Faster feedback |
| Testing | ⚠️ Manual | ✅ Automated tests | Regression protection |

### Trade-offs Accepted

1. ✅ **Accept**: Multiple columns for Phase 2 (3 migrations)
2. ✅ **Accept**: Application-level validation is primary defense
3. ✅ **Accept**: REGEXP is not perfect (can't prevent all duplicates)
4. ✅ **Accept**: Need to maintain serialization logic

### Trade-offs Mitigated

1. ✅ **Mitigated**: Invalid data via REGEXP + app validation
2. ✅ **Mitigated**: Silent bugs via comprehensive testing
3. ✅ **Mitigated**: Confusion via column comments + docs
4. ✅ **Mitigated**: Duplicates via automatic sorting

---

**Document Status**: Ready for decision
**Next Step**: Team review and selection
**Implementation**: Pending decision

---
