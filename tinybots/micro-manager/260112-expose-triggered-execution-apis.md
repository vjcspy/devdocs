# 📋 [260112] - Expose Triggered Script Execution APIs for Frontend

## Implementation Status

| API | Status | Date |
|-----|--------|------|
| API 1: List Triggered Executions | ✅ **DONE** | 2026-01-15 |
| API 2: Get Execution Detail | ⏳ PENDING | - |

## References

- **Global Standard**: `devdocs/tinybots/OVERVIEW.md`
- **Repo Standard**: `devdocs/tinybots/micro-manager/OVERVIEW.md`
- **Previous Plan**: `devdocs/tinybots/micro-manager/251206-store-trigger-script.md`
- **Related Services**:
  - `devdocs/tinybots/megazord-events/OVERVIEW.md` - Event ingestion & fan-out
  - `devdocs/tinybots/m-o-triggers/OVERVIEW.md` - Trigger scheduling
- **Key Files in micro-manager**:
  - `src/controllers/ScriptUserController.ts` - Existing user-facing execution endpoint
  - `src/controllers/ExecutionUserController.ts` - **NEW** User-facing triggered execution endpoint
  - `src/repository/ScriptExecutionRepository.ts` - Execution data access
  - `src/repository/ScriptRepository.ts` - Script metadata access
  - `src/services/ScriptExecutionService.ts` - Execution business logic
  - `docs/micro-manager.yaml` - OpenAPI specification

## User Requirements

**Current State:**

- ✅ Trigger-based script executions are being stored (implemented in `251206-store-trigger-script.md`)
- ✅ API 1 (List) to expose triggered executions to frontend - **IMPLEMENTED**
- ⏳ API 2 (Detail) for execution details - **PENDING**
- ⏳ Frontend integration - **PENDING**

**Business Problem:**

From stakeholder requirements (with attached images):
1. When a trigger occurs, show this in the schedule at the time it occurs
2. In the schedule, display the name of the script that is executed by the trigger
3. When clicking on the trigger in the schedule, show the steps of the script Tessa executed

**Goal:**

Create 2 APIs to expose script executions to frontend:
1. **List API**: Get triggered executions for schedule view (lightweight, returns script name + execution time)
2. **Detail API**: Get full execution details for **any execution type** (scheduled OR triggered) - Generic API using `script_execution.id`

---

## 🎯 Objective

Expose read APIs for frontend to display trigger-based script executions in the daily schedule view and detail view.

### Use Cases

**Schedule View:**
```
┌───────────────────────────────────────────────────────────────────┐
│  Dinsdag 6 januari 2026                                           │
│  ────────────────────────────────────────────────────────────────│
│  09:15  Geel alarm: iets langer in bed  🔒                       │ ← List API
│  14:30  Rood alarm: gevallen            🔒                        │
│  16:00  Groen alarm: inactief           🔒                        │
└───────────────────────────────────────────────────────────────────┘
```

**Detail View (TAAK OVERZICHT):**
```
┌───────────────────────────────────────────────────────────────────┐
│  💬 Geel alarm: iets langer in bed                                │
│                                                                   │
│  Wanneer is dit uitgevoerd?                                       │
│  📅 09:15 uur - Dinsdag, 6 januari 2026                          │ ← Detail API
│                                                                   │
│  Wat heeft Tessa gedaan?                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 🗣️ TESSA ZEI                                    09:15       │ │
│  │ Fijne ochtend op deze mooie ${DAY}...                        │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ ⏱️ 2 sec                                                     │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ ❓ TESSA VROEG                                   09:15       │ │
│  │ Heb je al een kopje thee...                                  │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │                                        ja        09:15       │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## 📚 Background: Database Schema

### Relevant Tables

```sql
-- script_execution (modified in V98)
-- Now supports both scheduled (schedule_id NOT NULL) and triggered (triggering_event_id NOT NULL)
┌──────────────────────────────────────────────────────────────────────────────┐
│ script_execution                                                              │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ script_reference_id│ BIGINT UNSIGNED NOT NULL → script_reference.id         │
│ script_version_id  │ BIGINT UNSIGNED NOT NULL → script_version.id           │
│ schedule_id        │ BIGINT UNSIGNED NULL → task_schedule.id (scheduled)    │
│ planned            │ DATETIME NULL (scheduled execution time)               │
│ triggering_event_id│ BIGINT UNSIGNED NULL → event_trigger.id (triggered)    │
│ created_at         │ DATETIME DEFAULT CURRENT_TIMESTAMP                     │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- script_version (contains script name)
┌──────────────────────────────────────────────────────────────────────────────┐
│ script_version                                                                │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ script_reference_id│ BIGINT UNSIGNED NOT NULL → script_reference.id         │
│ script_name        │ VARCHAR(255) NOT NULL  ← This is what we need!         │
│ script_category_id │ BIGINT UNSIGNED NOT NULL → script_category.id          │
│ duration           │ INT UNSIGNED                                           │
│ language_tag_id    │ BIGINT UNSIGNED NOT NULL                               │
│ capability_version │ INT UNSIGNED NOT NULL                                  │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- script_category (script categories)
┌──────────────────────────────────────────────────────────────────────────────┐
│ script_category                                                               │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ name               │ VARCHAR(255) NOT NULL  ← Category name for display     │
│ created_at         │ DATETIME DEFAULT CURRENT_TIMESTAMP                     │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- event_trigger (trigger record created by m-o-triggers)
┌──────────────────────────────────────────────────────────────────────────────┐
│ event_trigger                                                                 │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ setting_id         │ BIGINT UNSIGNED NOT NULL → event_trigger_setting.id    │
│ robot_id           │ INT UNSIGNED NOT NULL → robot_account.id               │
│ status             │ VARCHAR(64) NOT NULL                                   │
│ script_reference_id│ BIGINT UNSIGNED NOT NULL → script_reference.id         │
│ outgoing_event_id  │ BIGINT UNSIGNED NOT NULL → outgoing_event.id           │
│ level              │ TINYINT UNSIGNED NOT NULL (alarm level)                │
│ executed_at        │ TIMESTAMP (when actually executed)                     │
│ expected_executed_at│ TIMESTAMP NOT NULL                                     │
│ completed_at       │ TIMESTAMP                                              │
│ created_at         │ TIMESTAMP DEFAULT CURRENT_TIMESTAMP                    │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- event_trigger_setting (trigger configuration)
┌──────────────────────────────────────────────────────────────────────────────┐
│ event_trigger_setting                                                         │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ robot_id           │ INT UNSIGNED NOT NULL → robot_account.id               │
│ script_reference_id│ BIGINT UNSIGNED NOT NULL → script_reference.id         │
│ event_type_id      │ BIGINT UNSIGNED NOT NULL → event_schema.id             │
│ enabled            │ TINYINT(1) NOT NULL DEFAULT 1                          │
│ created_at         │ TIMESTAMP DEFAULT CURRENT_TIMESTAMP                    │
│ updated_at         │ TIMESTAMP                                              │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- event_schema (event type definitions with name)
┌──────────────────────────────────────────────────────────────────────────────┐
│ event_schema                                                                  │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ name               │ VARCHAR(255) NOT NULL  ← This is the trigger name!     │
│ event_source       │ VARCHAR(255) NOT NULL                                  │
│ created_at         │ TIMESTAMP DEFAULT CURRENT_TIMESTAMP                    │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- script_step_execution (step-level execution data)
┌──────────────────────────────────────────────────────────────────────────────┐
│ script_step_execution                                                         │
├────────────────────┬─────────────────────────────────────────────────────────┤
│ id                 │ BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY              │
│ script_execution_id│ BIGINT UNSIGNED NOT NULL → script_execution.id         │
│ script_step_id     │ BIGINT UNSIGNED NOT NULL → script_step.id              │
│ next_script_step_id│ BIGINT UNSIGNED NULL → script_step.id                  │
│ executed_at        │ DATETIME(3) NOT NULL (millisecond precision)           │
│ created_at         │ DATETIME DEFAULT CURRENT_TIMESTAMP                     │
└────────────────────┴─────────────────────────────────────────────────────────┘

-- Execution data tables (LEFT JOIN)
-- - closed_question_execution_data (answer, probability)
-- - multiple_choice_execution_data (answer, intention_type)
-- - report_execution_data (sent, message)
```

### Timestamp Sources for `executedAt`

| Execution Type | Source | Field | Reason |
|----------------|--------|-------|--------|
| **Scheduled** | `script_execution` | `planned` | Time the script was scheduled to run |
| **Triggered** | `event_trigger` | `executed_at` | Actual time the trigger executed the script |

**Note**: `script_execution.created_at` is when the record was **stored** in DB (after robot sends data), not when the script was actually **executed**. For triggered executions, we use `event_trigger.executed_at` which is the accurate execution timestamp.

---

## 🔧 API Design

### API 1: List Triggered Executions (Schedule View)

**Purpose:** Lightweight API to get triggered executions for schedule view display.

**Endpoint:**
```
GET /v6/scripts/user/robots/:robotId/executions/triggered
```

**Authentication:** Kong user header (`x-consumer-username`)

**Path Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| robotId | number | ✅ | Robot ID (validated via checkUserAccess) |

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| from | string | ✅ | - | ISO8601 datetime (start of range) |
| to | string | ✅ | - | ISO8601 datetime (end of range) |
| limit | number | ❌ | 100 | Max results to return |

**Request Example:**
```
GET /v6/scripts/user/robots/123/executions/triggered?from=2026-01-06T00:00:00Z&to=2026-01-06T23:59:59Z&limit=50
Headers:
  x-consumer-username: user-456
```

**Response (200 OK):**
```json
{
  "triggeredExecutions": [
    {
      "triggeredExecutionId": 100,
      "executedAt": "2026-01-06T09:15:00.000Z",
      "script": {
        "scriptReferenceId": 456,
        "scriptVersionId": 789,
        "scriptName": "Geel alarm: iets langer in bed",
        "scriptCategory": "alarm"
      },
      "trigger": {
        "triggeringEventId": 111,
        "triggerName": "Geel alarm trigger"
      }
    },
    {
      "triggeredExecutionId": 101,
      "executedAt": "2026-01-06T14:30:00.000Z",
      "script": {
        "scriptReferenceId": 457,
        "scriptVersionId": 790,
        "scriptName": "Rood alarm: gevallen",
        "scriptCategory": "alarm"
      },
      "trigger": {
        "triggeringEventId": 112,
        "triggerName": "Rood alarm trigger"
      }
    }
  ]
}
```

**Response Schema:**
```typescript
interface TriggeredExecutionListResponse {
  triggeredExecutions: TriggeredExecutionSummary[]
}

interface TriggeredExecutionSummary {
  triggeredExecutionId: number  // ID of script_execution record
  executedAt: string            // ISO8601 datetime of execution
  script: {
    scriptReferenceId: number   // Reference ID for script lookup
    scriptVersionId: number     // Version ID executed
    scriptName: string          // Display name for schedule view
    scriptCategory: string      // Category name (e.g. "alarm", "routine", "other")
  }
  trigger: {
    triggeringEventId: number   // ID of event_trigger record
    triggerName: string         // Display name of the trigger
  }
}
```

**Error Responses:**
- `400 Bad Request`: Invalid query params (from/to not valid dates)
- `403 Forbidden`: User doesn't have access to robot

---

### API 2: Get Execution Detail (Generic - works for both Scheduled and Triggered)

**Purpose:** Get full execution details including all steps and step data. Works for **both** scheduled and triggered executions using the unified `script_execution.id`.

**Endpoint:**
```
GET /v6/scripts/user/robots/:robotId/executions/:executionId
```

**Authentication:** Kong user header (`x-consumer-username`)

**Path Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| robotId | number | ✅ | Robot ID (validated via checkUserAccess) |
| executionId | number | ✅ | ID of script_execution record (works for both types) |

**Request Example (Triggered):**
```
GET /v6/scripts/user/robots/123/executions/100
Headers:
  x-consumer-username: user-456
```

**Response (200 OK) - Triggered Execution:**
```json
{
  "id": 100,
  "executionType": "triggered",
  "executedAt": "2026-01-06T09:15:00.000Z",
  "script": {
    "scriptReferenceId": 456,
    "scriptVersionId": 789,
    "scriptName": "Geel alarm: iets langer in bed",
    "scriptCategory": "alarm"
  },
  "trigger": {
    "triggeringEventId": 111,
    "triggerName": "Geel alarm trigger"
  },
  "scriptExecutionSteps": [
    {
      "id": 1,
      "scriptStepId": 20,
      "stepType": "say",
      "nextScriptStepId": 21,
      "executedAt": "2026-01-06T09:15:00.000Z",
      "data": null
    },
    {
      "id": 2,
      "scriptStepId": 21,
      "stepType": "wait",
      "nextScriptStepId": 22,
      "executedAt": "2026-01-06T09:15:02.000Z",
      "data": null
    },
    {
      "id": 3,
      "scriptStepId": 22,
      "stepType": "closedQuestion",
      "nextScriptStepId": 23,
      "executedAt": "2026-01-06T09:15:05.123Z",
      "data": {
        "answer": "ja",
        "probability": 0.95,
        "dataType": "closedQuestionData"
      }
    },
    {
      "id": 4,
      "scriptStepId": 23,
      "stepType": "say",
      "nextScriptStepId": null,
      "executedAt": "2026-01-06T09:15:08.456Z",
      "data": null
    }
  ]
}
```

**Response (200 OK) - Scheduled Execution:**
```json
{
  "id": 50,
  "executionType": "scheduled",
  "executedAt": "2026-01-06T10:00:00.000Z",
  "script": {
    "scriptReferenceId": 123,
    "scriptVersionId": 456,
    "scriptName": "Goedemorgen routine",
    "scriptCategory": "routine"
  },
  "schedule": {
    "scheduleId": 789,
    "planned": "2026-01-06T10:00:00.000Z"
  },
  "scriptExecutionSteps": [
    {
      "id": 10,
      "scriptStepId": 30,
      "stepType": "say",
      "nextScriptStepId": 31,
      "executedAt": "2026-01-06T10:00:00.000Z",
      "data": null
    }
  ]
}
```

**Response Schema:**
```typescript
interface ExecutionDetailResponse {
  id: number                              // ID of script_execution record
  executionType: 'scheduled' | 'triggered'  // Discriminator field
  executedAt: string                      // ISO8601 datetime of execution
  script: {
    scriptReferenceId: number             // Reference ID for script lookup
    scriptVersionId: number               // Version ID executed
    scriptName: string                    // Display name
    scriptCategory: string                // Category name (e.g. "alarm", "routine")
  }
  // Only present for scheduled executions
  schedule?: {
    scheduleId: number                    // ID of task_schedule record
    planned: string                       // ISO8601 datetime of planned time
  }
  // Only present for triggered executions
  trigger?: {
    triggeringEventId: number             // ID of event_trigger record
    triggerName: string                   // Display name of the trigger
  }
  scriptExecutionSteps: ExecutionStep[]
}

interface ExecutionStep {
  id: number
  scriptStepId: number
  stepType: 'say' | 'wait' | 'closedQuestion' | 'multipleChoice' | 'report' | 'statusCheck'
  nextScriptStepId: number | null
  executedAt: string            // ISO8601 datetime
  data: ClosedQuestionData | MultipleChoiceData | ReportData | null
}

interface ClosedQuestionData {
  answer: string                // "ja", "nee", "other"
  probability: number           // 0.0 - 1.0
  dataType: 'closedQuestionData'
}

interface MultipleChoiceData {
  answer: string
  intentionType: 'command' | 'offline' | 'other'
  dataType: 'multipleChoiceData'
}

interface ReportData {
  sent: boolean
  message: string
  dataType: 'reportData'
}
```

**Error Responses:**
- `403 Forbidden`: User doesn't have access to robot
- `404 Not Found`: Execution not found for given robot

---

## 🔄 Implementation Plan

### Phase 1: Repository Layer

**Priority**: High | **Effort**: 0.5 day | **Dependencies**: None | **Status**: 🟡 Partial

**Tasks:**

1. **Add New Repository Methods**
   - File: `micro-manager/src/repository/ScriptExecutionRepository.ts`
   - ✅ Add `getTriggeredExecutionsForSchedule()` method (list with script name) - **DONE**
   - ⏳ Add `getExecutionDetail()` method (execution + steps) - **PENDING**

**SQL Queries:**

```sql
-- getTriggeredExecutionsForSchedule (List API - triggered only)
-- ✅ IMPLEMENTED - Uses event_schema.name for trigger name
SELECT 
  se.id AS triggered_execution_id,
  et.executed_at AS executed_at,    -- Use event_trigger.executed_at (actual execution time)
  -- script info
  se.script_reference_id,
  se.script_version_id,
  sv.script_name,
  sc.name AS script_category,       -- Category from script_category table
  -- trigger info
  se.triggering_event_id,
  es.name AS trigger_name           -- Uses event_schema.name (via event_trigger_setting.event_type_id)
FROM script_execution se
JOIN script_version sv ON se.script_version_id = sv.id
JOIN script_category sc ON sv.script_category_id = sc.id  -- JOIN to get category
JOIN script_reference sr ON se.script_reference_id = sr.id
JOIN event_trigger et ON se.triggering_event_id = et.id
JOIN event_trigger_setting ets ON et.setting_id = ets.id
JOIN event_schema es ON ets.event_type_id = es.id         -- JOIN to get trigger name
WHERE sr.robot_id = ?
  AND se.triggering_event_id IS NOT NULL
  AND et.executed_at >= ?           -- Filter by actual execution time
  AND et.executed_at <= ?
ORDER BY et.executed_at DESC
LIMIT ?

-- getExecutionDetail (Detail API - GENERIC for both scheduled and triggered)
-- ⏳ NOT YET IMPLEMENTED
SELECT 
  se.id,
  -- execution type discriminator
  CASE 
    WHEN se.schedule_id IS NOT NULL THEN 'scheduled'
    WHEN se.triggering_event_id IS NOT NULL THEN 'triggered'
  END AS execution_type,
  -- executed_at: use planned for scheduled, event_trigger.executed_at for triggered
  CASE 
    WHEN se.schedule_id IS NOT NULL THEN se.planned
    WHEN se.triggering_event_id IS NOT NULL THEN et.executed_at
  END AS executed_at,
  -- script info
  se.script_reference_id,
  se.script_version_id,
  sv.script_name,
  sc.name AS script_category,       -- Category from script_category table
  -- schedule info (NULL for triggered)
  se.schedule_id,
  se.planned,
  -- trigger info (NULL for scheduled)
  se.triggering_event_id,
  es.name AS trigger_name           -- Uses event_schema.name (via event_trigger_setting.event_type_id)
FROM script_execution se
JOIN script_version sv ON se.script_version_id = sv.id
JOIN script_category sc ON sv.script_category_id = sc.id  -- JOIN to get category
JOIN script_reference sr ON se.script_reference_id = sr.id
LEFT JOIN event_trigger et ON se.triggering_event_id = et.id
LEFT JOIN event_trigger_setting ets ON et.setting_id = ets.id
LEFT JOIN event_schema es ON ets.event_type_id = es.id    -- JOIN to get trigger name
WHERE se.id = ?
  AND sr.robot_id = ?

-- getScriptStepExecutions (reuse existing GET_SCRIPT_STEP_EXECUTION)
-- Already implemented in ScriptExecutionRepository
```

**Acceptance Criteria:**
- ✅ `getTriggeredExecutionsForSchedule()` returns list with script names (triggered only)
- ✅ `getExecutionDetail()` returns execution with steps (both scheduled AND triggered)
- ✅ Generic query uses LEFT JOIN for trigger tables (nullable for scheduled executions)
- ✅ Response includes `executionType` discriminator field

---

### Phase 2: Service Layer

**Priority**: High | **Effort**: 0.5 day | **Dependencies**: Phase 1 | **Status**: 🟡 Partial

**Tasks:**

1. **Add Service Methods**
   - File: `micro-manager/src/services/ScriptExecutionService.ts`
   - ✅ Add `getTriggeredExecutionsForSchedule()` method (triggered list only) - **DONE**
   - ⏳ Add `getExecutionDetail()` method (generic - both types) - **PENDING**

```typescript
// ScriptExecutionService.ts

public async getTriggeredExecutionsForSchedule({
  robotId,
  from,
  to,
  limit = 100
}: {
  robotId: number
  from: Date | string
  to: Date | string
  limit?: number
}): Promise<TriggeredExecutionSummary[]> {
  return this.scriptExecutionRepository.getTriggeredExecutionsForSchedule({
    robotId, from, to, limit
  })
}

// GENERIC method - works for both scheduled and triggered executions
public async getExecutionDetail({
  robotId,
  executionId
}: {
  robotId: number
  executionId: number
}): Promise<ExecutionDetailResponse | null> {
  const execution = await this.scriptExecutionRepository
    .getExecutionDetail({ robotId, executionId })
  
  if (!execution) return null

  const steps = await this.scriptExecutionRepository
    .getScriptStepExecutions(executionId)
  
  // Build response based on execution type
  const response: ExecutionDetailResponse = {
    id: execution.id,
    executionType: execution.executionType,  // 'scheduled' | 'triggered'
    executedAt: execution.executedAt,
    script: {
      scriptReferenceId: execution.scriptReferenceId,
      scriptVersionId: execution.scriptVersionId,
      scriptName: execution.scriptName,
      scriptCategory: execution.scriptCategory  // Category from script_category table
    },
    scriptExecutionSteps: steps
  }

  // Add type-specific fields
  if (execution.executionType === 'scheduled') {
    response.schedule = {
      scheduleId: execution.scheduleId,
      planned: execution.planned
    }
  } else if (execution.executionType === 'triggered') {
    response.trigger = {
      triggeringEventId: execution.triggeringEventId,
      triggerName: execution.triggerName
    }
  }

  return response
}
```

**Acceptance Criteria:**
- ✅ `getTriggeredExecutionsForSchedule()` returns triggered executions list
- ✅ `getExecutionDetail()` returns execution for **both** scheduled AND triggered
- ✅ Detail method includes `executionType` discriminator
- ✅ `schedule` object present only for scheduled executions
- ✅ `trigger` object present only for triggered executions
- ✅ Method returns null if not found

---

### Phase 3: DTOs & Validation

**Priority**: High | **Effort**: 0.5 day | **Dependencies**: None | **Status**: 🟡 Partial

**Tasks:**

1. **Create New DTOs**
   - File: `micro-manager/src/schemas/body/TriggeredExecution.ts` (MODIFY existing)
   - ✅ Define `TriggeredExecutionSummary` interface (for list API) - **DONE**
   - ✅ Define `TriggeredExecutionListResponse` interface - **DONE**
   - ✅ Define `ScriptInfo`, `TriggerInfo` interfaces - **DONE**
   - ✅ Define `TriggeredExecutionRow` interface (DB mapping) - **DONE**
   - ⏳ Define `ExecutionDetailResponse` class (GENERIC for detail API) - **PENDING**

2. **Create Param Schema**
   - File: `micro-manager/src/schemas/params/robotIdExecutionIdSchema.ts` (NEW)
   - ⏳ Define Joi schema for robotId + executionId params (generic) - **PENDING** (for Detail API)

3. **Create Query Schema**
   - File: `micro-manager/src/schemas/query/triggeredExecutionQuerySchema.ts` (NEW)
   - ✅ Define Joi schema for from/to/limit query params - **DONE**

```typescript
// schemas/body/TriggeredExecution.ts

// Nested types for script info
export class ScriptInfo {
  @IsInt()
  scriptReferenceId: number

  @IsInt()
  scriptVersionId: number

  @IsString()
  scriptName: string

  @IsString()
  scriptCategory: string  // Category name from script_category table
}

// Nested types for schedule info (scheduled executions only)
export class ScheduleInfo {
  @IsInt()
  scheduleId: number

  @IsDate()
  planned: Date | string
}

// Nested types for trigger info (triggered executions only)
export class TriggerInfo {
  @IsInt()
  triggeringEventId: number

  @IsString()
  triggerName: string
}

// Summary for list API (triggered only)
export class TriggeredExecutionSummary {
  @IsInt()
  triggeredExecutionId: number

  @IsDate()
  executedAt: Date | string

  @ValidateNested()
  @Type(() => ScriptInfo)
  script: ScriptInfo

  @ValidateNested()
  @Type(() => TriggerInfo)
  trigger: TriggerInfo
}

// GENERIC Detail response - works for both scheduled and triggered
export class ExecutionDetailResponse {
  @IsInt()
  id: number

  @IsIn(['scheduled', 'triggered'])
  executionType: 'scheduled' | 'triggered'

  @IsDate()
  executedAt: Date | string

  @ValidateNested()
  @Type(() => ScriptInfo)
  script: ScriptInfo

  // Optional - only present for scheduled executions
  @IsOptional()
  @ValidateNested()
  @Type(() => ScheduleInfo)
  schedule?: ScheduleInfo

  // Optional - only present for triggered executions
  @IsOptional()
  @ValidateNested()
  @Type(() => TriggerInfo)
  trigger?: TriggerInfo

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ExecutionStep)
  scriptExecutionSteps: ExecutionStep[]
}

// List response wrapper
export class TriggeredExecutionListResponse {
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => TriggeredExecutionSummary)
  triggeredExecutions: TriggeredExecutionSummary[]
}
```

```typescript
// schemas/params/robotIdExecutionIdSchema.ts
import Joi from 'joi'

export const robotIdExecutionIdSchema = Joi.object({
  robotId: Joi.number().integer().min(1).required(),
  executionId: Joi.number().integer().min(1).required()
})
```

```typescript
// schemas/query/triggeredExecutionQuerySchema.ts
import Joi from 'joi'

export const triggeredExecutionQuerySchema = Joi.object({
  from: Joi.date().iso().required(),
  to: Joi.date().iso().required(),
  limit: Joi.number().integer().min(1).max(500).default(100)
})
```

**Acceptance Criteria:**
- ✅ `ExecutionDetailResponse` supports both execution types via `executionType` discriminator
- ✅ `schedule` field only present for scheduled executions
- ✅ `trigger` field only present for triggered executions
- ✅ Query params validated (from/to required dates, limit optional)

---

### Phase 4: Controller & Routes

**Priority**: High | **Effort**: 1 day | **Dependencies**: Phase 2, Phase 3 | **Status**: 🟡 Partial

**Tasks:**

1. **Create New Controller**
   - File: `micro-manager/src/controllers/ExecutionUserController.ts` (NEW)
   - ✅ Implement `getTriggeredExecutions()` handler (list - triggered only) - **DONE**
   - ⏳ Implement `getExecutionDetail()` handler (detail - GENERIC) - **PENDING**

```typescript
// controllers/ExecutionUserController.ts

import { Request, Response, NextFunction } from 'express'
import { ScriptExecutionService } from '../services/ScriptExecutionService'
import { 
  TriggeredExecutionListResponse,
  ExecutionDetailResponse 
} from '../schemas/body/TriggeredExecution'

export class ExecutionUserController {
  constructor(private scriptExecutionService: ScriptExecutionService) {}

  // API 1: List triggered executions (triggered only)
  public getTriggeredExecutions = async (
    req: Request,
    res: Response<TriggeredExecutionListResponse>,
    next: NextFunction
  ) => {
    try {
      const robotId = parseInt(req.params.robotId)
      const { from, to, limit } = req.query as {
        from: string
        to: string
        limit?: string
      }

      const triggeredExecutions = await this.scriptExecutionService
        .getTriggeredExecutionsForSchedule({
          robotId,
          from: new Date(from),
          to: new Date(to),
          limit: limit ? parseInt(limit) : 100
        })

      res.status(200).json({ triggeredExecutions })
    } catch (error) {
      next(error)
    }
  }

  // API 2: Get execution detail (GENERIC - both scheduled and triggered)
  public getExecutionDetail = async (
    req: Request,
    res: Response<ExecutionDetailResponse | { message: string }>,
    next: NextFunction
  ) => {
    try {
      const robotId = parseInt(req.params.robotId)
      const executionId = parseInt(req.params.executionId)

      const result = await this.scriptExecutionService
        .getExecutionDetail({ robotId, executionId })

      if (!result) {
        res.status(404).json({ 
          message: `Execution ${executionId} not found for robot ${robotId}` 
        })
        return
      }

      // Response includes executionType ('scheduled' | 'triggered')
      // with schedule or trigger fields accordingly
      res.status(200).json(result)
    } catch (error) {
      next(error)
    }
  }
}
```

2. **Register Routes**
   - File: `micro-manager/src/routes/routes.ts`
   - ✅ API 1 route registered - **DONE**
   - ⏳ API 2 route - **PENDING**

```typescript
// routes/routes.ts

import { ExecutionUserController } from '../controllers/ExecutionUserController'
import { triggeredExecutionQuerySchema } from '../schemas/query/triggeredExecutionQuerySchema'
import { robotIdExecutionIdSchema } from '../schemas/params/robotIdExecutionIdSchema'

// In route registration function:
const executionUserController: ExecutionUserController = 
  container.resolve('executionUserController')

// ✅ API 1: List triggered executions (triggered only - for schedule view) - DONE
app.get('/v6/scripts/user/robots/:robotId/executions/triggered',
  joiValidator.headers(kongHeaderSchema),
  joiValidator.params(robotIdSchema),
  joiValidator.query(triggeredExecutionQuerySchema),
  checkUserAccess,
  executionUserController.getTriggeredExecutions)

// ⏳ API 2: Get execution detail (GENERIC - works for both scheduled and triggered) - PENDING
app.get('/v6/scripts/user/robots/:robotId/executions/:executionId',
  joiValidator.headers(kongHeaderSchema),
  joiValidator.params(robotIdExecutionIdSchema),
  checkUserAccess,
  executionUserController.getExecutionDetail)
```

3. **Register in DI Container**
   - File: `micro-manager/src/buildContainer.ts`
   - ✅ Add `executionUserController` - **DONE**

**Acceptance Criteria:**
- ✅ List endpoint returns 200 with triggered executions only
- ✅ Detail endpoint returns 200 for **both** scheduled AND triggered executions
- ✅ Detail response includes `executionType` discriminator field
- ✅ Detail endpoint returns 404 if execution not found
- ✅ Both endpoints validate Kong headers
- ✅ Both endpoints check user access to robot

---

### Phase 5: API Documentation

**Priority**: Medium | **Effort**: 0.5 day | **Dependencies**: Phase 4

**Tasks:**

1. **Create OpenAPI Specs**
   - File: `tiny-specs/specs/local/paths/micro-manager/v6/triggered-executions.yaml` (NEW)
   - File: `tiny-specs/specs/local/components/micro-manager/v6/triggered-execution-schemas.yaml` (NEW)

2. **Update micro-manager docs**
   - File: `micro-manager/docs/micro-manager.yaml`

**Acceptance Criteria:**
- ✅ OpenAPI specs complete for both endpoints
- ✅ Examples match actual response format

---

### Phase 6: Testing

**Priority**: High | **Effort**: 1 day | **Dependencies**: Phase 4 | **Status**: 🟡 Partial

**Tasks:**

1. **Unit Tests**
   - File: `micro-manager/test/services/ScriptExecutionService.UT.spec.ts`
   - ✅ Test `getTriggeredExecutionsForSchedule()` - **DONE** (5 test cases)
   - ⏳ Test `getExecutionDetail()` - **PENDING**

2. **Integration Tests**
   - File: `micro-manager/test/IT/repositoryIT/ScriptExecutionRepository.triggered.IT.spec.ts`
   - ✅ Test `getTriggeredExecutionsForSchedule()` - **DONE**
   - Tests include: empty result, date filtering, robot filtering, limit, ordering

3. **Controller Tests**
   - File: `micro-manager/test/controllers/ExecutionUserController.IT.spec.ts` (NEW)
   - ✅ API 1 tests - **DONE** (10 test cases)
     - 200 with empty array
     - 400 for missing from/to params
     - 400 for invalid date format
     - 400 for limit out of range
     - 400 for missing Kong headers
     - 400 for invalid robotId
     - 401 for wrong consumer username
     - 403 for unauthorized access
   - ⏳ API 2 tests - **PENDING**

**Test Scenarios:**

**List API (Triggered Only):**

| Scenario | Expected |
|----------|----------|
| Valid request with data | 200 + list of triggered executions |
| Valid request, no data | 200 + empty array |
| Missing from param | 400 Bad Request |
| Invalid date format | 400 Bad Request |
| User can't access robot | 403 Forbidden |
| Limit=1 | Returns only 1 result |

**Detail API (Generic - Both Types):**

| Scenario | Expected |
|----------|----------|
| Valid request for **triggered** execution | 200 + `executionType: "triggered"` + `trigger` object |
| Valid request for **scheduled** execution | 200 + `executionType: "scheduled"` + `schedule` object |
| Execution not found | 404 Not Found |
| User can't access robot | 403 Forbidden |
| Execution belongs to different robot | 404 Not Found |
| Execution has steps | 200 + `scriptExecutionSteps` array populated |
| Execution has no steps | 200 + `scriptExecutionSteps: []` |

**Acceptance Criteria:**
- ✅ All unit tests pass
- ✅ All integration tests pass
- ✅ Coverage > 90% for new code

---

## 📊 Summary

### Files Created (API 1 - List)

| File | Purpose | Status |
|------|---------|--------|
| `src/controllers/ExecutionUserController.ts` | New controller | ✅ Created |
| `src/schemas/query/triggeredExecutionQuerySchema.ts` | Query param validation | ✅ Created |
| `src/schemas/body/TriggeredExecution.ts` | DTOs for list API | ✅ Created |
| `test/controllers/ExecutionUserController.IT.spec.ts` | Controller IT tests | ✅ Created |
| `typ-e/src/main/resources/db/migration/V101__grant_micro_manager_event_tables.sql` | DB permissions | ✅ Created |

### Files Modified (API 1 - List)

| File | Changes | Status |
|------|---------|--------|
| `src/repository/ScriptExecutionRepository.ts` | Added `getTriggeredExecutionsForSchedule()` | ✅ Done |
| `src/services/ScriptExecutionService.ts` | Added `getTriggeredExecutionsForSchedule()` | ✅ Done |
| `src/routes/routes.ts` | Registered List API route | ✅ Done |
| `src/buildContainer.ts` | Registered `executionUserController` | ✅ Done |
| `devtools/docker-compose.yaml` | Mounted V101 migration | ✅ Done |

### Files to Create (API 2 - Detail) - PENDING

| File | Purpose |
|------|---------|
| `src/schemas/params/robotIdExecutionIdSchema.ts` | Path param validation (generic) |
| `tiny-specs/specs/local/paths/micro-manager/v6/executions.yaml` | OpenAPI paths |
| `tiny-specs/specs/local/components/micro-manager/v6/execution-schemas.yaml` | OpenAPI schemas |

### Files to Modify (API 2 - Detail) - PENDING

| File | Changes |
|------|---------|
| `src/schemas/body/TriggeredExecution.ts` | Add `ExecutionDetailResponse`, `ScheduleInfo` DTOs |
| `src/repository/ScriptExecutionRepository.ts` | Add `getExecutionDetail()` (generic) |
| `src/services/ScriptExecutionService.ts` | Add `getExecutionDetail()` (generic) |
| `src/routes/routes.ts` | Register Detail API route |
| `src/controllers/ExecutionUserController.ts` | Add `getExecutionDetail()` handler |
| `micro-manager/docs/micro-manager.yaml` | Add API documentation |

### Database Migration Required

**File**: `typ-e/src/main/resources/db/migration/V101__grant_micro_manager_event_tables.sql`

The `micro-manager-rw` database user needs SELECT permission on additional tables for the List API query:

```sql
-- Grant SELECT permission to micro-manager-rw on event_trigger_setting
GRANT SELECT ON `tinybots`.`event_trigger_setting` TO 'micro-manager-rw'@'10.0.0.0/255.0.0.0';
GRANT SELECT ON `tinybots`.`event_trigger_setting` TO 'micro-manager-rw'@'172.16.0.0/255.240.0.0';
GRANT SELECT ON `tinybots`.`event_trigger_setting` TO 'micro-manager-rw'@'192.168.0.0/255.255.0.0';

-- Grant SELECT permission to micro-manager-rw on event_schema
GRANT SELECT ON `tinybots`.`event_schema` TO 'micro-manager-rw'@'10.0.0.0/255.0.0.0';
GRANT SELECT ON `tinybots`.`event_schema` TO 'micro-manager-rw'@'172.16.0.0/255.240.0.0';
GRANT SELECT ON `tinybots`.`event_schema` TO 'micro-manager-rw'@'192.168.0.0/255.255.0.0';
```

**Note**: For local Docker testing, the migration file is mounted in `devtools/docker-compose.yaml`.

### Effort Estimation

| Phase | Effort | Status |
|-------|--------|--------|
| Phase 1: Repository | 0.5 day | 🟡 Partial |
| Phase 2: Service | 0.5 day | 🟡 Partial |
| Phase 3: DTOs | 0.5 day | 🟡 Partial |
| Phase 4: Controller & Routes | 1 day | 🟡 Partial |
| Phase 5: API Documentation | 0.5 day | ⏳ Pending |
| Phase 6: Testing | 1 day | 🟡 Partial |
| **Total** | **4 days** | |

**Actual Progress (API 1 only)**: ~1.5 days

### Success Metrics

**Technical:**
- ✅ List API returns triggered executions correctly
- ✅ Detail API works for **both** scheduled AND triggered executions
- ✅ API latency < 100ms (p95)
- ✅ Test coverage > 90%
- ✅ No regression in existing APIs

**Business:**
- ✅ Frontend can display triggered executions in schedule view
- ✅ Frontend can show execution details for **any** execution type using one API
- ✅ Response includes `executionType` discriminator for frontend logic
- ✅ Data matches what Tessa actually executed

---

## 🔗 Related Links

- Previous plan: `devdocs/tinybots/micro-manager/251206-store-trigger-script.md`
- **Legacy** scheduled execution API: `GET /v2/scripts/user/robots/:robotId/scripts/:scriptReferenceId/executions/:scheduleId`
  - Uses `scheduleId` + `planned` query param
  - Still works but new generic API is recommended for new integrations

## 📝 Design Decision Notes

### Trigger Name Source (Implementation Note)

**Original assumption**: `event_trigger_setting.name` contains the trigger name.

**Actual implementation**: The `event_trigger_setting` table does NOT have a `name` column. Instead:
- `event_trigger_setting.event_type_id` references `event_schema.id`
- `event_schema.name` contains the trigger/event type name (e.g., "WAKEUP_EVENT", "FALL_ALARM")

**SQL Join Path**:
```
script_execution 
  → event_trigger (via triggering_event_id)
  → event_trigger_setting (via setting_id)
  → event_schema (via event_type_id) → name
```

### Why Generic Detail API?

The Detail API (`GET /v6/.../executions/:executionId`) is generic instead of triggered-specific because:

1. **Unified Data Model**: Both scheduled and triggered executions live in the same `script_execution` table with the same primary key
2. **Same Response Structure**: Execution info + steps structure is identical for both types
3. **Frontend Flexibility**: Schedule view displays both types together - one API simplifies frontend integration
4. **Future-proof**: If new execution types are added, no new endpoints needed

### Relationship with Legacy API

| API | Use Case |
|-----|----------|
| `GET /v2/.../executions/:scheduleId` | Legacy - requires `scheduleId` + `planned` |
| `GET /v6/.../executions/:executionId` | **NEW** - uses `script_execution.id` directly, works for both types |

The new API is more intuitive since it uses the actual database primary key.
