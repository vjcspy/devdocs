# 🚀 Production Release Document: Day-of-Week Scheduling Constraints

**Release Date**: TBD
**Feature**: Add `allowed_days_of_week` column for day-of-week scheduling constraints
**JIRA Ticket**: [251107]
**Status**: ✅ Ready for Production Release

---

## 📋 Executive Summary

This release implements a **new feature** for day-of-week scheduling constraints for event triggers, allowing triggers to be scheduled only on specific days (e.g., Monday, Wednesday, Friday).

**Key Benefits**:
- ✅ **New capability**: Triggers can be restricted to specific days of the week
- ✅ **Smart rescheduling**: Automatic rescheduling to next allowed day when current day is not permitted
- ✅ **Performance**: Using VARCHAR column for efficient storage and indexing
- ✅ **Simple API**: Direct array format for easy integration

**Breaking Changes**:
- ✅ **None** - This is a new feature with backward compatibility
- ✅ Existing triggers without `allowedDaysOfWeek` continue to work as before (no day restrictions)---

## 🎯 What's Changed

### Database Changes
- **New column**: `allowed_days_of_week VARCHAR(100)` added to `event_trigger_setting` table
- **Storage format**: Comma-separated day codes (e.g., `"fri,mon,wed"`)
- **Valid values**: `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun` (sorted alphabetically)
- **NULL handling**: NULL means no day restrictions (any day allowed - backward compatible)
- **Default value**: NULL for all existing records (maintains current behavior)

### API Changes

#### New Optional Field: `allowedDaysOfWeek`
The existing API endpoint now accepts an **optional** new field for day-of-week constraints.

#### Request Format (New Feature)
```json
PUT /v1/triggers/settings
{
  "eventName": "event.test",
  "allowedStartTime": "08:00",
  "allowedEndTime": "20:00",
  "isReschedulable": true,
  "allowedDaysOfWeek": ["mon", "wed", "fri"],  // ✨ NEW optional field
  "maxConcurrentTriggers": 3,
  "allowedConcurrencyTimeframe": "5m",
  "defaultScriptReferenceId": 123
}
```

**Note**: `allowedDaysOfWeek` is **optional**. If omitted, triggers work as before (no day restrictions).

#### Response Format (New Feature)
```json
{
  "id": 1,
  "robotId": null,
  "eventTypeId": 5,
  "maxConcurrentTriggers": 3,
  "allowedConcurrencyTimeframe": 300000,
  "allowedStartTime": "08:00",
  "allowedEndTime": "20:00",
  "maxValidityDuration": 2000000,
  "isReschedulable": true,
  "isDefault": true,
  "defaultScriptReferenceId": 123,
  "allowedDaysOfWeek": ["mon", "wed", "fri"]  // ✨ NEW field in response (null if not set)
}
```

### Behavior

#### Scenario 1: Current Day is Allowed
- **Condition**: Trigger created on Monday with `allowedDaysOfWeek: ["mon", "wed", "fri"]`
- **Result**: Executes immediately (if within allowed time window)

#### Scenario 2: Current Day Not Allowed (Reschedulable)
- **Condition**: Trigger created on Tuesday with `allowedDaysOfWeek: ["mon", "wed", "fri"]`
- **Result**: Rescheduled to Wednesday at `allowedStartTime`

#### Scenario 3: Current Day Not Allowed (Non-Reschedulable)
- **Condition**: Trigger created on Tuesday with `allowedDaysOfWeek: ["mon", "wed"]` and `isReschedulable: false`
- **Result**: Trigger rejected with status `REJECTED`

#### Scenario 4: No Day Restrictions
- **Condition**: `allowedDaysOfWeek` is `null` or not provided
- **Result**: Works as before (any day allowed)

---

## 🚀 Release Steps

### 1: Database Migration (typ-e Pipeline)

- **Repository**: [`typ-e`](https://bitbucket.org/tinybots/typ-e/src/develop/) (database migration repository)
- **Migration File**: `V94__trigger_setting.sql.sql`
- **Commit ID**: `4afc6a466bacfd7d25e881dee5529cc5dae6ba08`

**Expected Output**:
```
COLUMN_NAME             | DATA_TYPE | CHARACTER_MAXIMUM_LENGTH | COLUMN_COMMENT
allowed_days_of_week    | varchar   | 100                     | Comma-separated list of allowed days...
```

**Verification**:
```sql
-- All existing records should have NULL (no day constraints)
SELECT id, allowed_days_of_week
FROM event_trigger_setting
LIMIT 10;

-- Expected: allowed_days_of_week = NULL for all existing record
```
### 2. Deploy m-o-trigger
- **Repository**: [`m-o-trigger`](https://bitbucket.org/tinybots/m-o-triggers/src/master/) 
- **Commit ID**: `46cfbbf834d77c43ccf8a6513d80a46329ab7412`

### 3. API Tests

#### Test 1: Create Default Trigger Setting with Day Constraints
```bash
curl -X PUT https://m-o-triggers.production.example.com/v1/triggers/settings \
  -H "Content-Type: application/json" \
  -H "x-consumer-id: test-consumer" \
  -H "x-consumer-username: tinybots-dashboard-users" \
  -H "x-authenticated-scope: primary" \
  -H "x-authenticated-userid: <ADMIN_USER_ID>" \
  -d '{
    "eventName": "event.test.schedule",
    "maxConcurrentTriggers": 10,
    "allowedConcurrencyTimeframe": "10m",
    "allowedStartTime": "08:00",
    "allowedEndTime": "18:00",
    "maxValidityDuration": 2000000,
    "isReschedulable": true,
    "defaultScriptReferenceId": <SCRIPT_ID>,
    "allowedDaysOfWeek": ["mon", "wed", "fri"]
  }'

# Expected Response (201 Created):
{
  "id": <SETTING_ID>,
  "robotId": null,
  "eventTypeId": <EVENT_TYPE_ID>,
  "maxConcurrentTriggers": 10,
  "allowedConcurrencyTimeframe": 600000,
  "allowedStartTime": "08:00",
  "allowedEndTime": "18:00",
  "maxValidityDuration": 2000000,
  "isReschedulable": true,
  "isDefault": true,
  "defaultScriptReferenceId": <SCRIPT_ID>,
  "allowedDaysOfWeek": ["mon", "wed", "fri"]
}
```

**Verification**:
```sql
-- Check database storage (should be sorted CSV)
SELECT id, allowed_days_of_week
FROM event_trigger_setting
WHERE id = <SETTING_ID>;

-- Expected: allowed_days_of_week = "fri,mon,wed"
```

---

#### Test 2: Create Robot-Specific Trigger Setting
```bash
curl -X PUT https://m-o-triggers.production.example.com/v1/triggers/settings \
  -H "Content-Type: application/json" \
  -H "x-consumer-id: test-consumer" \
  -H "x-consumer-username: tinybots-dashboard-users" \
  -H "x-authenticated-scope: primary" \
  -H "x-authenticated-userid: <ADMIN_USER_ID>" \
  -d '{
    "eventName": "event.test.schedule",
    "robotId": <ROBOT_ID>,
    "maxConcurrentTriggers": 3,
    "allowedConcurrencyTimeframe": "15m",
    "allowedStartTime": "09:00",
    "allowedEndTime": "17:00",
    "maxValidityDuration": 3000000,
    "isReschedulable": true,
    "defaultScriptReferenceId": <SCRIPT_ID>,
    "allowedDaysOfWeek": ["tue", "thu"]
  }'

# Expected Response (201 Created):
{
  "id": <SETTING_ID>,
  "robotId": <ROBOT_ID>,
  "eventTypeId": <EVENT_TYPE_ID>,
  "maxConcurrentTriggers": 3,
  "allowedConcurrencyTimeframe": 900000,
  "allowedStartTime": "09:00",
  "allowedEndTime": "17:00",
  "maxValidityDuration": 3000000,
  "isReschedulable": true,
  "isDefault": true,
  "defaultScriptReferenceId": <SCRIPT_ID>,
  "allowedDaysOfWeek": ["tue", "thu"]
}
```

---

#### Test 3: Create Trigger on Allowed Day (Should Execute Immediately)
**Precondition**: Run on Monday
**Setting**: `allowedDaysOfWeek: ["mon", "wed", "fri"]`

```bash
# First, ensure default setting exists (from Test 1)

# Create trigger on Monday
curl -X POST https://m-o-triggers.production.example.com/internal/v1/triggers \
  -H "Content-Type: application/json" \
  -d '{
    "robotId": <ROBOT_ID>,
    "eventTypeId": <EVENT_TYPE_ID>,
    "outgoingEventId": <EVENT_ID>,
    "level": 2,
    "scriptReferenceId": <SCRIPT_ID>
  }'

# Expected Response (201 Created):
{
  "id": <TRIGGER_ID>,
  "robotId": <ROBOT_ID>,
  "eventTypeId": <EVENT_TYPE_ID>,
  "status": "ACCEPTED",
  "expectedExecutedAt": "2025-11-11T10:30:45.000Z",  // Current time (Monday)
  ...
}
```

**Verification**:
- `status` should be `ACCEPTED`
- `expectedExecutedAt` should be close to current time (within seconds)

---

#### Test 4: Create Trigger on Non-Allowed Day (Should Reschedule)
**Precondition**: Run on Tuesday
**Setting**: `allowedDaysOfWeek: ["mon", "wed", "fri"]`, `isReschedulable: true`

```bash
# Create trigger on Tuesday
curl -X POST https://m-o-triggers.production.example.com/internal/v1/triggers \
  -H "Content-Type: application/json" \
  -d '{
    "robotId": <ROBOT_ID>,
    "eventTypeId": <EVENT_TYPE_ID>,
    "outgoingEventId": <EVENT_ID>,
    "level": 2,
    "scriptReferenceId": <SCRIPT_ID>
  }'

# Expected Response (201 Created):
{
  "id": <TRIGGER_ID>,
  "robotId": <ROBOT_ID>,
  "eventTypeId": <EVENT_TYPE_ID>,
  "status": "ACCEPTED",
  "expectedExecutedAt": "2025-11-12T08:00:00.000Z",  // Wednesday at allowedStartTime
  ...
}
```

**Verification**:
- `status` should be `ACCEPTED`
- `expectedExecutedAt` should be Wednesday at `08:00` (allowedStartTime)

---

#### Test 5: Create Trigger Without Day Constraints (Backward Compatibility)
**Purpose**: Verify existing API calls continue to work without the new field

```bash
curl -X PUT https://m-o-triggers.production.example.com/v1/triggers/settings \
  -H "Content-Type: application/json" \
  -H "x-consumer-id: test-consumer" \
  -H "x-consumer-username: tinybots-dashboard-users" \
  -H "x-authenticated-scope: primary" \
  -H "x-authenticated-userid: <ADMIN_USER_ID>" \
  -d '{
    "eventName": "event.test.no.constraints",
    "maxConcurrentTriggers": 5,
    "allowedConcurrencyTimeframe": "5m",
    "allowedStartTime": "00:00",
    "allowedEndTime": "23:59",
    "maxValidityDuration": 1000000,
    "isReschedulable": true,
    "defaultScriptReferenceId": <SCRIPT_ID>
  }'

# Note: No allowedDaysOfWeek provided (backward compatible)

# Expected Response (201 Created):
{
  "id": <SETTING_ID>,
  "robotId": null,
  "eventTypeId": <EVENT_TYPE_ID>,
  "maxConcurrentTriggers": 5,
  "allowedConcurrencyTimeframe": 300000,
  "allowedStartTime": "00:00",
  "allowedEndTime": "23:59",
  "maxValidityDuration": 1000000,
  "isReschedulable": true,
  "isDefault": true,
  "defaultScriptReferenceId": <SCRIPT_ID>,
  "allowedDaysOfWeek": null  // NULL when not provided
}
```

**Verification**:
- ✅ API call succeeds without new field (backward compatible)
- ✅ Response includes `allowedDaysOfWeek: null`
- ✅ Database has `allowed_days_of_week = NULL`
- ✅ Triggers created with this setting work on any day (as before)

---

#### Test 6: Retrieve Trigger with Setting (Check Deserialization)
```bash
curl -X GET https://m-o-triggers.production.example.com/v1/triggers/<TRIGGER_ID> \
  -H "x-consumer-id: <ROBOT_ID>" \
  -H "x-consumer-username: <ROBOT_USERNAME>" \
  -H "x-authenticated-scope: primary"

# Expected Response (200 OK):
{
  "id": <TRIGGER_ID>,
  "robotId": <ROBOT_ID>,
  "eventTypeId": <EVENT_TYPE_ID>,
  "status": "ACCEPTED",
  "expectedExecutedAt": "...",
  "setting": {
    "id": <SETTING_ID>,
    "allowedDaysOfWeek": ["mon", "wed", "fri"],  // Should be array, not string
    ...
  }
}
```

**Verification**:
- `setting.allowedDaysOfWeek` should be an array, not a CSV string
- Days should be lowercase and sorted

---

#### Test 7: Invalid Day Names (Should Reject)
```bash
curl -X PUT https://m-o-triggers.production.example.com/v1/triggers/settings \
  -H "Content-Type: application/json" \
  -H "x-consumer-id: test-consumer" \
  -H "x-consumer-username: tinybots-dashboard-users" \
  -H "x-authenticated-scope: primary" \
  -H "x-authenticated-userid: <ADMIN_USER_ID>" \
  -d '{
    "eventName": "event.test.invalid",
    "maxConcurrentTriggers": 5,
    "allowedConcurrencyTimeframe": "5m",
    "allowedStartTime": "08:00",
    "allowedEndTime": "18:00",
    "maxValidityDuration": 1000000,
    "isReschedulable": true,
    "defaultScriptReferenceId": <SCRIPT_ID>,
    "allowedDaysOfWeek": ["monday", "wednesday"]
  }'

# Expected Response (400 Bad Request):
{
  "error": "Validation Error",
  "message": "Invalid day. Must be: mon, tue, wed, thu, fri, sat, sun",
  ...
}
```

---

