# Production Release: Store Trigger-Based Script Executions

This release adds the ability to store script executions that are triggered by events (e.g., robot enters a room, sensor detects motion) alongside existing scheduled executions in micro-manager. It also includes backward compatibility fixes to ensure existing APIs continue to return only scheduled executions.

## Business Context

### Problem Statement

Previously, TinyBots robots could only store execution history for **scheduled scripts** (scripts executed at specific times). Scripts triggered by events (triggers) were executed but **NOT persisted**, resulting in:

- ❌ No visibility into trigger-based automation
- ❌ Cannot trace which event triggered which script
- ❌ Missing insights for debugging and analytics

### Solution

**Part 1 - New API for Triggered Executions:**

A new API endpoint allows robots to store triggered script executions with:

- ✅ Full execution steps data (same structure as scheduled executions)
- ✅ Timestamp per step (`executedAt`)
- ✅ Traceability back to the triggering event (`triggeringEventId` → `event_trigger.id`)
- ✅ Idempotent design (safe retries on network failures)

**Part 2 - Backward Compatibility Fix:**

Since the `script_execution` table now contains both scheduled AND triggered executions, existing APIs have been updated to:

- ✅ Explicitly filter `WHERE schedule_id IS NOT NULL` to return ONLY scheduled executions
- ✅ Ensure triggered executions are invisible to existing clients/dashboards
- ✅ Maintain 100% backward compatibility for all existing consumers

---

## Scope & Jira Coverage

| Jira Key | Description | Notes |
| --- | --- | --- |
| [PROD-724](https://tinybots.atlassian.net/browse/PROD-724) | Store trigger-based script executions | New API + database schema |
| 260105 | Filter scheduled executions only in existing APIs | Backward compatibility fix |

---

## Pre-deployment Checklist

- [ ] Code review: micro-manager PR merged (includes both new API + backward compatibility fix)
- [ ] Code review: tiny-specs PR merged (API documentation)
- [ ] Database migration `V98__add_triggered_script_execution_support.sql` executed on staging
- [ ] Verify existing scheduled execution APIs return ONLY scheduled executions
- [ ] Verify `POST /internal/v4/scripts/executions/search` excludes triggered executions
- [ ] Verify micro-manager can SELECT from `event_trigger` and `outgoing_event` tables
- [ ] All existing micro-manager tests pass (no regression)

---

## Deployment Steps

### 1. Run Database Migration (typ-e)

**Repository**: `typ-e`  
**Migration File**: `src/main/resources/db/migration/V98__add_triggered_script_execution_support.sql`

**Changes Applied:**

| Change | Description |
| --- | --- |
| Add column | `triggering_event_id BIGINT UNSIGNED NULL` with index |
| Modify column | `schedule_id` → nullable |
| Modify column | `planned` → nullable |
| Add FK | `fk_script_execution_triggering_event_id` → `event_trigger(id)` |
| Add CHECK | `chk_execution_source` - ensures either scheduled OR triggered |
| Grant SELECT | `micro-manager-rw` on `event_trigger` and `outgoing_event` |

**Verification Query:**

```sql
-- Verify migration applied
DESCRIBE script_execution;

-- Expected: triggering_event_id column exists, schedule_id and planned are nullable
-- +----------------------+---------------------+------+-----+-------------------+-------------------+
-- | Field                | Type                | Null | Key | Default           | Extra             |
-- +----------------------+---------------------+------+-----+-------------------+-------------------+
-- | id                   | bigint unsigned     | NO   | PRI | NULL              | auto_increment    |
-- | script_reference_id  | bigint unsigned     | NO   | MUL | NULL              |                   |
-- | script_version_id    | bigint unsigned     | NO   | MUL | NULL              |                   |
-- | schedule_id          | bigint unsigned     | YES  | MUL | NULL              |                   |
-- | planned              | datetime            | YES  |     | NULL              |                   |
-- | created_at           | datetime            | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
-- | triggering_event_id  | bigint unsigned     | YES  | MUL | NULL              |                   |
-- +----------------------+---------------------+------+-----+-------------------+-------------------+

-- Verify check constraint
SELECT CONSTRAINT_NAME, CHECK_CLAUSE 
FROM INFORMATION_SCHEMA.CHECK_CONSTRAINTS 
WHERE TABLE_NAME = 'script_execution';
```

### 2. Deploy micro-manager

**Repository**: `micro-manager`  

**Files Changed (New Triggered Execution API):**

| File | Change |
| --- | --- |
| `src/controllers/ScriptTriggeredExecutionController.ts` | NEW - Controller for triggered executions |
| `src/services/ScriptExecutionService.ts` | Added `saveTriggeredExecution()` method |
| `src/repository/ScriptExecutionRepository.ts` | Added 4 new methods for triggered executions |
| `src/schemas/body/ScriptExecution.ts` | Added `PostTriggeredScriptExecutionDTO` |
| `src/schemas/params/scriptReferenceIdTriggeringEventIdSchema.ts` | NEW - Param validation |
| `src/middleware/validation/body.ts` | Added validation middleware |
| `src/routes/routes.ts` | Registered new endpoint |
| `src/buildContainer.ts` | Registered controller in DI |

**Files Changed (Backward Compatibility - Filter Scheduled Only):**

| File | Change |
| --- | --- |
| `src/repository/ScriptExecutionRepository.ts` | Added `schedule_id IS NOT NULL` filter to 3 existing queries |

**SQL Queries Modified for Backward Compatibility:**

| Query | Used By | Change |
| --- | --- | --- |
| `GET_SCRIPT_EXECUTION` | `GET /v2/.../executions/:scheduleId` | Added explicit `AND se.schedule_id IS NOT NULL` |
| `GET_SCRIPT_EXECUTION_ID` | `PUT /v2/.../executions/:scheduleId` (idempotency) | Added explicit `AND se.schedule_id IS NOT NULL` |
| `GET_REPORT_EXECUTIONS` | `POST /internal/v4/scripts/executions/search` | **CRITICAL FIX**: Added `AND se.schedule_id IS NOT NULL` |

### 3. Deploy tiny-specs (API Documentation)

**Repository**: `tiny-specs`  

**Files Added:**

| File | Description |
| --- | --- |
| `specs/local/paths/micro-manager/v6/paths.yaml` | Endpoint definition |
| `specs/local/components/micro-manager/v6/schemas.yaml` | Request/response schemas |

---

## API Documentation

### Endpoint

```
PUT /v6/scripts/robot/scripts/{scriptReferenceId}/executions/triggered/{triggeringEventId}
```

### Authentication

- **Security**: Kong Robot Auth (via `x-consumer-username` header)
- **Access Control**: Robot must own the script reference

### Path Parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `scriptReferenceId` | integer | ID of the script reference being executed |
| `triggeringEventId` | integer | ID from `event_trigger` table that triggered this execution |

### Request Body

```json
{
  "scriptVersionId": 42,
  "scriptExecutionSteps": [
    {
      "scriptStepId": 101,
      "stepType": "say",
      "nextScriptStepId": 102,
      "executedAt": "2025-12-23T10:30:00.000Z",
      "data": null
    },
    {
      "scriptStepId": 102,
      "stepType": "closedQuestion",
      "nextScriptStepId": 103,
      "executedAt": "2025-12-23T10:30:15.000Z",
      "data": {
        "dataType": "closedQuestionData",
        "answer": "yes",
        "probability": 0.95
      }
    },
    {
      "scriptStepId": 103,
      "stepType": "report",
      "nextScriptStepId": null,
      "executedAt": "2025-12-23T10:30:30.000Z",
      "data": {
        "dataType": "reportData",
        "sent": true,
        "message": "Script execution completed"
      }
    }
  ]
}
```

### Response Codes

| Code | Description |
| --- | --- |
| `204 No Content` | Success - execution stored (idempotent) |
| `400 Bad Request` | Validation error (invalid body, empty steps) |
| `403 Forbidden` | Robot doesn't own script or invalid step IDs |
| `404 Not Found` | Script reference or triggering event not found |

### Idempotency

The endpoint is **idempotent** - calling it multiple times with the same `triggeringEventId` for the same robot/script will not create duplicate records. This allows robots to safely retry on network failures.

---

## Build Verification Tests

### Prerequisites

- Access to micro-manager service (internal or via Kong)
- Valid robot credentials (Kong consumer)
- Test data: existing `script_reference`, `script_version`, `script_step`, and `event_trigger` records

### 1. Basic Success - Store Triggered Execution

```bash
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/456' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{
    "scriptVersionId": 42,
    "scriptExecutionSteps": [
      {
        "scriptStepId": 101,
        "stepType": "say",
        "nextScriptStepId": null,
        "executedAt": "2025-01-14T10:30:00.000Z",
        "data": null
      }
    ]
  }'
```

**Expected Response:** `204 No Content`

### 2. Verify Idempotency - Same Request Twice

```bash
# First call
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/456' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{"scriptVersionId": 42, "scriptExecutionSteps": [{"scriptStepId": 101, "stepType": "say", "nextScriptStepId": null, "executedAt": "2025-01-14T10:30:00.000Z", "data": null}]}'

# Second call (same data)
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/456' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{"scriptVersionId": 42, "scriptExecutionSteps": [{"scriptStepId": 101, "stepType": "say", "nextScriptStepId": null, "executedAt": "2025-01-14T10:30:00.000Z", "data": null}]}'
```

**Expected:** Both return `204 No Content`, only ONE record in database

### 3. Verify With ClosedQuestion Data

```bash
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/457' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{
    "scriptVersionId": 42,
    "scriptExecutionSteps": [
      {
        "scriptStepId": 102,
        "stepType": "closedQuestion",
        "nextScriptStepId": null,
        "executedAt": "2025-01-14T10:31:00.000Z",
        "data": {
          "dataType": "closedQuestionData",
          "answer": "yes",
          "probability": 0.92
        }
      }
    ]
  }'
```

**Expected Response:** `204 No Content`

### 4. Verify Validation - Missing scriptVersionId

```bash
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/458' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{
    "scriptExecutionSteps": [
      {
        "scriptStepId": 101,
        "stepType": "say",
        "nextScriptStepId": null,
        "executedAt": "2025-01-14T10:30:00.000Z",
        "data": null
      }
    ]
  }'
```

**Expected Response:** `400 Bad Request`

### 5. Verify Validation - Empty Steps Array

```bash
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/123/executions/triggered/459' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-789' \
  -d '{
    "scriptVersionId": 42,
    "scriptExecutionSteps": []
  }'
```

**Expected Response:** `400 Bad Request`

### 6. Verify Database Record

```sql
-- Check execution was stored
SELECT se.*, et.outgoing_event_id
FROM script_execution se
LEFT JOIN event_trigger et ON se.triggering_event_id = et.id
WHERE se.triggering_event_id IS NOT NULL
ORDER BY se.created_at DESC
LIMIT 10;

-- Check execution steps were stored
SELECT sse.*, se.triggering_event_id
FROM script_step_execution sse
JOIN script_execution se ON sse.script_execution_id = se.id
WHERE se.triggering_event_id IS NOT NULL
ORDER BY sse.executed_at DESC
LIMIT 10;
```

---

## Understanding the System (For Testers)

### 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACES                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │
│  │   Tessa App     │    │   Family App    │    │   Admin Dashboard       │ │
│  │   (Mobile)      │    │   (Mobile)      │    │   (Web)                 │ │
│  └────────┬────────┘    └────────┬────────┘    └────────────┬────────────┘ │
│           │                      │                          │               │
│           ▼                      ▼                          ▼               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API Gateway (Kong)                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│           ┌────────────────────────┼────────────────────────┐              │
│           ▼                        ▼                        ▼              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐│
│  │      EVE        │    │  micro-manager  │    │   wonkers-graphql       ││
│  │ (Schedules)     │    │ (Scripts &      │    │   (Reports &            ││
│  │                 │    │  Executions)    │    │    Analytics)           ││
│  └────────┬────────┘    └────────┬────────┘    └────────────┬────────────┘│
│           │                      │                          │              │
│           └──────────────────────┼──────────────────────────┘              │
│                                  ▼                                         │
│                    ┌─────────────────────────────┐                         │
│                    │    MySQL Database (typ-e)   │                         │
│                    │  ┌───────────────────────┐  │                         │
│                    │  │   script_execution    │  │ ← MODIFIED TABLE        │
│                    │  │  (scheduled + triggered)│ │                         │
│                    │  └───────────────────────┘  │                         │
│                    └─────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 📊 Data Flow - What Changed

**Before this release:**
- `script_execution` table chỉ chứa **scheduled executions**
- Tất cả UI/APIs query table này → chỉ thấy scheduled data

**After this release:**
- `script_execution` table chứa **CẢ scheduled VÀ triggered executions**
- Existing APIs được cập nhật để **filter chỉ scheduled executions**
- Triggered executions được lưu nhưng **CHƯA hiển thị trên UI**

### 🔍 How to Identify Execution Types

| Field | Scheduled Execution | Triggered Execution |
|-------|---------------------|---------------------|
| `schedule_id` | ✅ Has value (NOT NULL) | ❌ NULL |
| `planned` | ✅ Has timestamp | ❌ NULL |
| `triggering_event_id` | ❌ NULL | ✅ Has value (NOT NULL) |

**SQL to check:**
```sql
-- Scheduled executions (should appear on UI)
SELECT * FROM script_execution WHERE schedule_id IS NOT NULL;

-- Triggered executions (should NOT appear on UI yet)
SELECT * FROM script_execution WHERE triggering_event_id IS NOT NULL;
```

---

## UI Regression Tests (For Testers)

⚠️ **CRITICAL**: Những tests này đảm bảo UI hiển thị đúng sau khi thay đổi schema `script_execution`. Triggered executions **KHÔNG ĐƯỢC** xuất hiện trên UI hiện tại.

### 🧪 Test Data Setup (Pre-requisite)

**Option 1: Use existing staging/dev data**

1. Chọn robot đã có scheduled executions trước đó
2. Verify robot có data:
```sql
-- Check robot has scheduled executions
SELECT COUNT(*) 
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = [ROBOT_ID] AND se.schedule_id IS NOT NULL;
```

**Option 2: Create test data via Robot**

1. Schedule 1 script cho robot test
2. Đợi robot thực hiện script (hoặc trigger manually)
3. Verify execution được tạo trong DB

**Option 3: Create triggered execution via API (for mixed data testing)**

```bash
# Create triggered execution (robot auth required)
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/{scriptReferenceId}/executions/triggered/{triggeringEventId}' \
  -H 'x-consumer-username: robot-{robotId}' \
  -H 'Content-Type: application/json' \
  -d '{
    "scriptVersionId": {versionId},
    "scriptExecutionSteps": [
      {"scriptStepId": 101, "stepType": "say", "nextScriptStepId": null, "executedAt": "2025-01-14T10:00:00Z", "data": null}
    ]
  }'
```

---

### 📱 Test Case UI-1: Schedule View (Tessa App / Dashboard)

**Mục đích:** Verify lịch scheduled tasks vẫn hiển thị đúng trên app Tessa

**Luồng test:**

1. Login vào Tessa App (hoặc Dashboard web)
2. Chọn robot có scheduled tasks
3. Xem lịch ngày hôm nay / tuần này

**Expected:**
- [ ] Scheduled tasks hiển thị đúng theo thời gian đã lên lịch
- [ ] Các task types hiển thị đúng: Reminder, Question, Script, Music
- [ ] Không có task "lạ" xuất hiện (triggered executions không được hiển thị)

**Screenshot location:** (Attach screenshot here)

---

### 📱 Test Case UI-2: Execution Detail View (Click vào Scheduled Task)

**Mục đích:** Verify chi tiết execution của scheduled task hiển thị đúng

**Prerequisites:** 
- Robot đã thực hiện ít nhất 1 scheduled script
- Script đó có các step types: say, closedQuestion, multipleChoice, report

**Luồng test:**

1. Mở Schedule View
2. Click vào một **scheduled task đã hoàn thành** (có checkmark ✅)
3. Xem màn hình "TAAK OVERZICHT" / Task Detail

**Expected:**
```
┌───────────────────────────────────────────────────────────────────┐
│  💬 [Tên Script]                                                  │
│                                                                   │
│  Wanneer is dit uitgevoerd?                                       │
│  📅 [HH:MM] uur - [Dag], [Datum]                                  │
│                                                                   │
│  Wat heeft Tessa gedaan?                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 🗣️ TESSA ZEI                                    [Time]      │ │
│  │ [Script step content...]                                    │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ ❓ TESSA VROEG                                   [Time]      │ │
│  │ [Question content...]                                       │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │                                        [Answer]  [Time]      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

**Verification checklist:**
- [ ] Tên script hiển thị đúng
- [ ] Thời gian thực hiện hiển thị đúng
- [ ] Các step hiển thị theo thứ tự thực hiện
- [ ] Các câu trả lời (nếu có) hiển thị đúng
- [ ] Không có error khi load detail

**Screenshot location:** (Attach screenshot here)

---

### 📱 Test Case UI-3: Empty Execution (Script chưa có execution)

**Mục đích:** Verify script scheduled nhưng chưa thực hiện hiển thị đúng

**Luồng test:**

1. Mở Schedule View
2. Click vào một **scheduled task trong tương lai** (chưa đến giờ thực hiện)

**Expected:**
- [ ] Hiển thị thông tin task (tên, thời gian scheduled)
- [ ] Không hiển thị execution detail (hoặc hiển thị "Chưa thực hiện")
- [ ] Không có error

---

### 📊 Test Case UI-4: Execution Report (Admin Dashboard)

**Mục đích:** Verify Execution Report chỉ chứa scheduled executions

**Prerequisites:**
- Tài khoản admin có quyền xem reports
- Có data: robot với BOTH scheduled executions VÀ triggered executions (trong DB)

**Luồng test:**

1. Login Admin Dashboard (wonkers-graphql)
2. Navigate: Reports → Execution Report
3. Filter: Chọn organization, date range có cả 2 loại executions

**Expected:**
- [ ] Report chỉ hiển thị scheduled executions
- [ ] Triggered executions **KHÔNG** xuất hiện trong report
- [ ] Các cột hiển thị đúng: Script Name, Category, Executed At, Robot, Organization

**Verification:**

So sánh count với database:

```sql
-- Count scheduled executions (should match report)
SELECT COUNT(*) FROM script_execution 
WHERE schedule_id IS NOT NULL 
  AND created_at BETWEEN '[from_date]' AND '[to_date]';

-- Count triggered executions (should NOT appear in report)
SELECT COUNT(*) FROM script_execution 
WHERE triggering_event_id IS NOT NULL 
  AND created_at BETWEEN '[from_date]' AND '[to_date]';
```

**Screenshot location:** (Attach screenshot here)

---

### 📱 Test Case UI-5: Multiple Robots - Mixed Data

**Mục đích:** Verify với nhiều robots có different execution types

**Prerequisites:**
- User có quyền truy cập 2+ robots
- Robot A: chỉ có scheduled executions
- Robot B: có cả scheduled VÀ triggered executions

**Luồng test:**

1. Xem Schedule của Robot A → verify scheduled tasks hiển thị đúng
2. Xem Schedule của Robot B → verify CHỈ scheduled tasks hiển thị
3. Click vào scheduled task của Robot B → verify execution detail đúng

**Expected:**
- [ ] Robot A: Scheduled tasks hiển thị đúng
- [ ] Robot B: CHỈ scheduled tasks hiển thị (triggered KHÔNG xuất hiện)
- [ ] Execution details cho cả 2 robots đều load đúng

---

### 🔄 Test Case UI-6: Refresh / Reload Behavior

**Mục đích:** Verify data consistency khi refresh

**Luồng test:**

1. Mở Schedule View
2. Note lại danh sách tasks
3. Pull-to-refresh (mobile) hoặc F5 (web)
4. So sánh danh sách tasks

**Expected:**
- [ ] Danh sách tasks không thay đổi sau refresh
- [ ] Không có triggered executions "nhảy vào" sau refresh
- [ ] Performance không bị ảnh hưởng đáng kể

---

## API Regression Tests (For Developers/QA)

⚠️ **CRITICAL**: These tests ensure existing APIs continue to work correctly after the `script_execution` table schema change. Triggered executions must NOT appear in these APIs.

### Prerequisites for Regression Testing

**Test Data Setup:**

You need a robot with BOTH types of executions in the database:

```sql
-- Verify test robot has mixed executions
SELECT 
  CASE 
    WHEN schedule_id IS NOT NULL THEN 'scheduled'
    WHEN triggering_event_id IS NOT NULL THEN 'triggered'
  END AS execution_type,
  COUNT(*) as count
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = :testRobotId
GROUP BY execution_type;

-- Expected: Both 'scheduled' and 'triggered' rows exist
```

---

### Regression Test 1: GET User Execution Detail

**Endpoint:** `GET /v2/scripts/user/robots/:robotId/scripts/:scriptReferenceId/executions/:scheduleId`

**Purpose:** Verify user-facing execution detail API returns ONLY scheduled executions

```bash
# Get a scheduled execution
curl -X GET 'https://api.tinybots.io/v2/scripts/user/robots/123/scripts/456/executions/100?planned=2025-01-14T10:00:00Z' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <user-token>'
```

**Expected Response:** `200 OK` with scheduled execution data

```json
{
  "scriptExecution": {
    "id": 1,
    "scheduleId": 100,
    "scriptVersionId": 42,
    "scriptReferenceId": 456,
    "planned": "2025-01-14T10:00:00.000Z"
  },
  "scriptExecutionSteps": [...]
}
```

**Verification:**
- [ ] Response contains `scheduleId` (NOT `triggeringEventId`)
- [ ] Response does NOT include any triggered executions
- [ ] `planned` timestamp is present and valid

---

### Regression Test 2: Internal Execution Search

**Endpoint:** `POST /internal/v4/scripts/executions/search`

**Purpose:** Verify internal search API returns ONLY scheduled executions (used by reporting/analytics)

```bash
curl -X POST 'https://api.tinybots.io/internal/v4/scripts/executions/search' \
  -H 'Content-Type: application/json' \
  -d '{
    "robotIds": [123],
    "from": "2025-01-01T00:00:00Z",
    "to": "2025-01-31T23:59:59Z",
    "lastExecutionId": 0,
    "limit": 100
  }'
```

**Expected Response:** `200 OK` with array of scheduled executions only

```json
{
  "executions": [
    {
      "id": 1,
      "robotId": 123,
      "scheduleId": 100,
      "planned": "2025-01-14T10:00:00.000Z",
      "scriptReferenceId": 456,
      "scriptVersionId": 42,
      "createdAt": "2025-01-14T10:00:05.000Z"
    }
  ]
}
```

**Verification:**
- [ ] All returned executions have `scheduleId` (not null)
- [ ] All returned executions have `planned` timestamp (not null)
- [ ] NO triggered executions are returned (triggering_event_id records excluded)
- [ ] Count matches expected scheduled executions in date range

**Database Verification Query:**

```sql
-- Count what the API should return (scheduled only)
SELECT COUNT(*) as expected_count
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = 123
  AND se.planned >= '2025-01-01 00:00:00'
  AND se.planned <= '2025-01-31 23:59:59'
  AND se.schedule_id IS NOT NULL;  -- ← Key filter

-- Count triggered executions (should NOT be in API response)
SELECT COUNT(*) as triggered_count
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = 123
  AND se.created_at >= '2025-01-01 00:00:00'
  AND se.created_at <= '2025-01-31 23:59:59'
  AND se.triggering_event_id IS NOT NULL;
```

---

### Regression Test 3: Scheduled Execution PUT (Idempotency)

**Endpoint:** `PUT /v2/scripts/robot/scripts/:scriptReferenceId/executions/:scheduleId`

**Purpose:** Verify scheduled execution save/idempotency works correctly (does NOT collide with triggered executions)

```bash
# First call - create scheduled execution
curl -X PUT 'https://api.tinybots.io/v2/scripts/robot/scripts/456/executions/100' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-123' \
  -d '{
    "scriptVersionId": 42,
    "planned": "2025-01-14T12:00:00.000Z",
    "scriptExecutionSteps": [
      {
        "scriptStepId": 101,
        "stepType": "say",
        "nextScriptStepId": null,
        "executedAt": "2025-01-14T12:00:05.000Z",
        "data": null
      }
    ]
  }'
```

**Expected Response:** `204 No Content`

```bash
# Second call - retry (idempotency test)
curl -X PUT 'https://api.tinybots.io/v2/scripts/robot/scripts/456/executions/100' \
  -H 'Content-Type: application/json' \
  -H 'x-consumer-username: robot-123' \
  -d '{
    "scriptVersionId": 42,
    "planned": "2025-01-14T12:00:00.000Z",
    "scriptExecutionSteps": [
      {
        "scriptStepId": 101,
        "stepType": "say",
        "nextScriptStepId": null,
        "executedAt": "2025-01-14T12:00:05.000Z",
        "data": null
      }
    ]
  }'
```

**Expected Response:** `204 No Content` (idempotent - no duplicate created)

**Verification:**
- [ ] First call succeeds with `204`
- [ ] Second call (retry) succeeds with `204`
- [ ] Only ONE execution record exists in database
- [ ] Execution has `schedule_id = 100` (not triggered)

**Database Verification Query:**

```sql
-- Verify only one scheduled execution created
SELECT COUNT(*) 
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = 123
  AND se.script_reference_id = 456
  AND se.schedule_id = 100
  AND se.planned = '2025-01-14 12:00:00';
-- Expected: 1
```

---

### Regression Test 4: Mixed Execution Types Segregation

**Purpose:** Verify scheduled and triggered executions are properly segregated

**Setup:** Create both types of executions for the same robot/script:

```bash
# 1. Create scheduled execution
curl -X PUT 'https://api.tinybots.io/v2/scripts/robot/scripts/456/executions/101' \
  -H 'x-consumer-username: robot-123' \
  -H 'Content-Type: application/json' \
  -d '{"scriptVersionId": 42, "planned": "2025-01-14T14:00:00.000Z", "scriptExecutionSteps": [{"scriptStepId": 101, "stepType": "say", "nextScriptStepId": null, "executedAt": "2025-01-14T14:00:05.000Z", "data": null}]}'

# 2. Create triggered execution (same robot/script, different trigger)
curl -X PUT 'https://api.tinybots.io/v6/scripts/robot/scripts/456/executions/triggered/500' \
  -H 'x-consumer-username: robot-123' \
  -H 'Content-Type: application/json' \
  -d '{"scriptVersionId": 42, "scriptExecutionSteps": [{"scriptStepId": 101, "stepType": "say", "nextScriptStepId": null, "executedAt": "2025-01-14T14:01:00.000Z", "data": null}]}'

# 3. Search via internal API - should return ONLY scheduled
curl -X POST 'https://api.tinybots.io/internal/v4/scripts/executions/search' \
  -H 'Content-Type: application/json' \
  -d '{"robotIds": [123], "from": "2025-01-14T00:00:00Z", "to": "2025-01-14T23:59:59Z", "lastExecutionId": 0, "limit": 100}'
```

**Verification:**
- [ ] Internal search returns scheduled execution (schedule_id = 101)
- [ ] Internal search does NOT return triggered execution (triggering_event_id = 500)
- [ ] Both executions exist in database but are properly segregated

**Database Verification:**

```sql
-- Should show 2 executions (1 scheduled, 1 triggered)
SELECT 
  se.id,
  se.schedule_id,
  se.triggering_event_id,
  se.planned,
  se.created_at,
  CASE 
    WHEN se.schedule_id IS NOT NULL THEN 'scheduled'
    WHEN se.triggering_event_id IS NOT NULL THEN 'triggered'
  END AS execution_type
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = 123
  AND se.script_reference_id = 456
ORDER BY se.created_at DESC;
```

---

### Regression Test 5: Data Integrity Check

**Purpose:** Verify CHECK constraint prevents invalid data combinations

```sql
-- This should FAIL due to CHECK constraint
-- (cannot have both schedule_id AND triggering_event_id)
INSERT INTO script_execution 
  (script_reference_id, script_version_id, schedule_id, planned, triggering_event_id)
VALUES 
  (456, 42, 100, '2025-01-14 15:00:00', 500);
-- Expected: ERROR - Check constraint 'chk_execution_source' is violated

-- This should also FAIL
-- (cannot have NEITHER schedule_id NOR triggering_event_id)
INSERT INTO script_execution 
  (script_reference_id, script_version_id, schedule_id, planned, triggering_event_id)
VALUES 
  (456, 42, NULL, NULL, NULL);
-- Expected: ERROR - Check constraint 'chk_execution_source' is violated

-- Verify no invalid data exists
SELECT COUNT(*) as invalid_count
FROM script_execution
WHERE (schedule_id IS NOT NULL AND triggering_event_id IS NOT NULL)
   OR (schedule_id IS NULL AND triggering_event_id IS NULL);
-- Expected: 0
```

---

## Verification Checklist

### Database Migration

- [ ] `triggering_event_id` column exists in `script_execution`
- [ ] `schedule_id` and `planned` columns are nullable
- [ ] CHECK constraint `chk_execution_source` is active
- [ ] FK `fk_script_execution_triggering_event_id` exists
- [ ] `micro-manager-rw` can SELECT from `event_trigger` and `outgoing_event`

### New API Functionality (Triggered Executions)

- [ ] PUT returns `204` for valid triggered execution
- [ ] Idempotency works (duplicate requests don't create duplicates)
- [ ] All step types work: `say`, `wait`, `closedQuestion`, `multipleChoice`, `report`, `statusCheck`
- [ ] Step data is stored correctly for each type
- [ ] Validation rejects invalid requests (`400`)
- [ ] Authorization works (`403` for wrong robot)
- [ ] Not found handled (`404` for invalid IDs)

### UI Regression (For Testers - CRITICAL)

- [ ] **UI-1**: Schedule View hiển thị scheduled tasks đúng
- [ ] **UI-2**: Execution Detail View hiển thị chi tiết đúng khi click scheduled task
- [ ] **UI-3**: Script chưa thực hiện không gây error
- [ ] **UI-4**: Execution Report (Admin) chỉ chứa scheduled executions
- [ ] **UI-5**: Multiple robots với mixed data hiển thị đúng
- [ ] **UI-6**: Refresh không làm thay đổi data hoặc hiển thị triggered executions
- [ ] Triggered executions **KHÔNG** xuất hiện ở bất kỳ UI nào

### API Regression (For Developers - CRITICAL)

- [ ] `GET /v2/.../executions/:scheduleId` returns ONLY scheduled executions
- [ ] `POST /internal/v4/scripts/executions/search` returns ONLY scheduled executions
- [ ] `PUT /v2/.../executions/:scheduleId` idempotency works correctly
- [ ] Triggered executions are NOT visible in any scheduled execution API
- [ ] Mixed execution types (scheduled + triggered) properly segregated
- [ ] CHECK constraint prevents invalid data combinations
- [ ] No duplicate executions created across scheduled/triggered types
- [ ] All existing micro-manager tests pass
- [ ] No performance degradation

### Data Integrity

- [ ] No executions have BOTH `schedule_id` AND `triggering_event_id`
- [ ] No executions have NEITHER `schedule_id` NOR `triggering_event_id`
- [ ] Scheduled executions have `planned` timestamp (not null)
- [ ] Triggered executions have `triggering_event_id` (not null)

---

## Test Results

**Unit Tests:** ✅ 8 test cases passing
- `saveTriggeredExecution()` flow validation
- Step validation reuse
- Report processing
- Error handling
- Idempotency scenarios

**Integration Tests (Repository):** ✅ 15 test cases passing
- `addTriggeredScriptExecution()` insert
- `getTriggeredExecutionId()` idempotency check
- `getExecutionsByTrigger()` queries
- CHECK constraint enforcement
- Error handling

**Integration Tests (Controller):** ✅ 13 test cases passing
- 204 response for valid request
- Validation errors (400)
- Authorization (403)
- All step data types
- Idempotency

**Total Test Coverage:** 36 comprehensive test cases

---

## Rollback Plan

If issues are discovered post-deployment:

### 1. Disable Endpoint (Quick)

Comment out route in `micro-manager/src/routes/routes.ts`:

```typescript
// Temporarily disable triggered execution endpoint
// app.put('/v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered/:triggeringEventId', ...)
```

### 2. Full Rollback (If Required)

**Step 1:** Delete triggered execution records:

```sql
-- Delete step execution data first (due to FK)
DELETE sse FROM script_step_execution sse
JOIN script_execution se ON sse.script_execution_id = se.id
WHERE se.triggering_event_id IS NOT NULL;

-- Delete triggered executions
DELETE FROM script_execution WHERE triggering_event_id IS NOT NULL;
```

**Step 2:** Revert schema changes (manual migration):

```sql
-- Remove FK and CHECK constraints
ALTER TABLE script_execution
  DROP FOREIGN KEY fk_script_execution_triggering_event_id,
  DROP CONSTRAINT chk_execution_source;

-- Remove column
ALTER TABLE script_execution
  DROP INDEX idx_triggering_event_id,
  DROP COLUMN triggering_event_id;

-- Restore NOT NULL (only if no NULL values exist)
ALTER TABLE script_execution
  DROP FOREIGN KEY script_execution_ibfk_1;

ALTER TABLE script_execution
  MODIFY COLUMN schedule_id BIGINT UNSIGNED NOT NULL,
  MODIFY COLUMN planned DATETIME NOT NULL;

ALTER TABLE script_execution
  ADD CONSTRAINT script_execution_ibfk_1
    FOREIGN KEY (schedule_id) REFERENCES task_schedule(id);

-- Revoke permissions
REVOKE SELECT ON tinybots.event_trigger FROM 'micro-manager-rw'@'10.0.0.0/255.0.0.0';
REVOKE SELECT ON tinybots.event_trigger FROM 'micro-manager-rw'@'172.16.0.0/255.240.0.0';
REVOKE SELECT ON tinybots.event_trigger FROM 'micro-manager-rw'@'192.168.0.0/255.255.0.0';
REVOKE SELECT ON tinybots.outgoing_event FROM 'micro-manager-rw'@'10.0.0.0/255.0.0.0';
REVOKE SELECT ON tinybots.outgoing_event FROM 'micro-manager-rw'@'172.16.0.0/255.240.0.0';
REVOKE SELECT ON tinybots.outgoing_event FROM 'micro-manager-rw'@'192.168.0.0/255.255.0.0';
```

---

## Related Documentation

- [Implementation Plan - Store Triggered Executions](../251206-store-trigger-script.md)
- [Implementation Plan - Filter Scheduled Only](../260105-filter-scheduled-executions-only.md)
- [Micro-Manager OVERVIEW](../OVERVIEW.md)
- [Megazord Events OVERVIEW](../../megazord-events/OVERVIEW.md)
- [M-O-Triggers OVERVIEW](../../m-o-triggers/OVERVIEW.md)
