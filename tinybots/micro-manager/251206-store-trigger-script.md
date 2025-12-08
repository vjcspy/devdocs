# 📋 [251206] - Store Trigger-Based Script Executions

## References

- **Global Standard**: `devdocs/tinybots/OVERVIEW.md`
- **Repo Standard**: `devdocs/tinybots/micro-manager/OVERVIEW.md`
- **Related Services**:
  - `devdocs/tinybots/megazord-events/OVERVIEW.md` - Event ingestion & fan-out
  - `devdocs/tinybots/sensara-adaptor/OVERVIEW.md` - Sensara telemetry bridge
  - `devdocs/tinybots/m-o-triggers/OVERVIEW.md` - Trigger scheduling
- **Key Files in micro-manager**:
  - `src/controllers/ScriptRobotController.ts` - Scheduled execution endpoint
  - `src/controllers/ScriptRobotControllerV3.ts` - Unscheduled execution endpoint (current)
  - `src/services/ScriptExecutionService.ts` - Execution business logic
  - `src/repository/ScriptExecutionRepository.ts` - Database access layer
  - `src/schemas/body/ScriptExecution.ts` - DTOs for executions
  - `src/routes/routes.ts` - API routing
  - `docs/micro-manager.yaml` - OpenAPI specification

## User Requirements

**Current State:**

- Tessa (robots) execute scripts and store executions for **scheduled scripts** ✅
- Scripts triggered by events (not scheduled) do **NOT** store executions ❌

**Business Problem:**

- No visibility into trigger-based automation (e.g., robot enters room → execute script)
- Cannot trace which event triggered which script execution
- Missing insights for debugging and analytics

**Goal:**
Store script executions for trigger-initiated scripts with:

1. **Node executed with data** (same structure as scheduled executions)
2. **Time of execution** of each node
3. **Incoming eventId** that started the execution (traceability back to megazord-events)

**Deliverables:**

1. Create new API specification in `tiny-specs`
2. Implement API in `micro-manager`

---

### 📝 Understanding Requirement Details

#### Requirement 1: "Node executed, with data (same as current)"

**Question**: Scheduled execution đã có chưa? Implement như thế nào?

**Answer**: ✅ **Đã có sẵn hoàn toàn!**

**Current Implementation in Scheduled Executions:**

```typescript
// Từ ScriptExecutionRepository.addScriptExecutionSteps()
public addScriptExecutionSteps = async (executionSteps: {
  scriptExecutionId: number,
  stepType?: string,
  scriptStepId: number,          // ← Node ID
  nextScriptStepId?: number,
  executedAt: string | Date,     // ← Timestamp per node
  data?: ExecutionData           // ← Node execution data
}[]) => {
  // Transaction...
  for (const step of executionSteps) {
    // 1. Insert step execution
    INSERT INTO script_step_execution 
      (script_execution_id, script_step_id, next_script_step_id, executed_at)
    
    // 2. Insert step-specific data based on type
    if (step.data.dataType === 'closedQuestionData') {
      INSERT INTO closed_question_execution_data 
        (script_step_execution_id, answer, probability)
    }
    else if (step.data.dataType === 'multipleChoiceData') {
      INSERT INTO multiple_choice_execution_data 
        (script_step_execution_id, answer, intention_type)
    }
    else if (step.data.dataType === 'reportData') {
      INSERT INTO report_execution_data 
        (script_step_execution_id, sent, message)
    }
  }
}
```

**Database Tables (Already Exist):**

```sql
-- Main step execution record
CREATE TABLE script_step_execution (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  script_execution_id BIGINT UNSIGNED NOT NULL,  -- Links to parent execution
  script_step_id BIGINT UNSIGNED NOT NULL,       -- Which node/step was executed
  next_script_step_id BIGINT UNSIGNED NULL,      -- Next node (if any)
  executed_at DATETIME(3) NOT NULL,              -- ✅ Requirement 2: Timestamp
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (script_execution_id) REFERENCES script_execution(id),
  FOREIGN KEY (script_step_id) REFERENCES script_step(id)
);

-- Step type: closedQuestion (Yes/No questions)
CREATE TABLE closed_question_execution_data (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  script_step_execution_id BIGINT UNSIGNED NOT NULL,
  answer VARCHAR(255) NOT NULL,           -- ✅ "yes", "no", "other"
  probability DECIMAL(5,4),               -- ✅ Confidence score
  FOREIGN KEY (script_step_execution_id) REFERENCES script_step_execution(id)
);

-- Step type: multipleChoice (Multiple choice questions)
CREATE TABLE multiple_choice_execution_data (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  script_step_execution_id BIGINT UNSIGNED NOT NULL,
  answer VARCHAR(255),                    -- ✅ User's answer
  intention_type ENUM('command', 'offline', 'other'),  -- ✅ Intent classification
  FOREIGN KEY (script_step_execution_id) REFERENCES script_step_execution(id)
);

-- Step type: report (Data submission)
CREATE TABLE report_execution_data (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  script_step_execution_id BIGINT UNSIGNED NOT NULL,
  sent TINYINT(1) NOT NULL,               -- ✅ Was report sent successfully?
  message TEXT,                           -- ✅ Response message
  FOREIGN KEY (script_step_execution_id) REFERENCES script_step_execution(id)
);
```

**Example Data (Scheduled Execution):**

```sql
-- script_execution record
INSERT INTO script_execution VALUES (
  1,      -- id
  123,    -- script_reference_id
  456,    -- script_version_id
  789,    -- schedule_id
  '2025-12-06 10:00:00',  -- planned
  NOW()   -- created_at
);

-- script_step_execution records (one per node)
INSERT INTO script_step_execution VALUES
  (10, 1, 20, NULL, '2025-12-06 10:00:05.123', NOW()),  -- Step 1: Say "Hello"
  (11, 1, 21, 22,   '2025-12-06 10:00:10.456', NOW()),  -- Step 2: Ask question
  (12, 1, 22, NULL, '2025-12-06 10:00:15.789', NOW());  -- Step 3: Final step

-- closed_question_execution_data (for step 11)
INSERT INTO closed_question_execution_data VALUES
  (1, 11, 'yes', 0.95);  -- User answered "yes" with 95% confidence
```

**What We Get from Queries:**

```typescript
// GET /v2/scripts/user/robots/:robotId/scripts/:scriptReferenceId/executions/:scheduleId
{
  "scriptExecution": {
    "id": 1,
    "scheduleId": 789,
    "scriptVersionId": 456,
    "scriptReferenceId": 123,
    "planned": "2025-12-06T10:00:00Z"
  },
  "scriptExecutionSteps": [
    {
      "id": 10,
      "scriptStepId": 20,
      "stepType": "say",
      "nextScriptStepId": null,
      "executedAt": "2025-12-06T10:00:05.123Z",  // ✅ Timestamp per node
      "data": null
    },
    {
      "id": 11,
      "scriptStepId": 21,
      "stepType": "closedQuestion",
      "nextScriptStepId": 22,
      "executedAt": "2025-12-06T10:00:10.456Z",  // ✅ Timestamp per node
      "data": {                                    // ✅ Execution data
        "answer": "yes",
        "probability": 0.95,
        "dataType": "closedQuestionData"
      }
    },
    {
      "id": 12,
      "scriptStepId": 22,
      "stepType": "report",
      "nextScriptStepId": null,
      "executedAt": "2025-12-06T10:00:15.789Z",  // ✅ Timestamp per node
      "data": {                                    // ✅ Execution data
        "sent": true,
        "message": "Report submitted successfully",
        "dataType": "reportData"
      }
    }
  ]
}
```

**✅ Conclusion:**

- ✅ **Requirement 1** (node executed with data): **Fully implemented** in scheduled executions
- ✅ **Requirement 2** (time of execution): **Fully implemented** via `script_step_execution.executed_at`
- ✅ **Requirement 3** (eventId): **Need to add** - this is the NEW part

**What We Need to Do:**

For triggered executions, we will:

1. ✅ **Reuse existing tables** (`script_step_execution`, `*_execution_data`)
2. ✅ **Reuse existing logic** (`addScriptExecutionSteps()` method)
3. ✅ **Only add parent record** in `script_execution` with event IDs instead of schedule_id

**No changes needed to step execution logic!** Just need to:

- Add `outgoing_event_id`, `incoming_event_id` columns to `script_execution`
- Allow `schedule_id`, `planned` to be NULL
- Create new endpoint/controller that calls existing step storage methods

---

## 🎯 Objective

Extend micro-manager's script execution storage to support **trigger-initiated (unscheduled) scripts**, enabling:

- Full execution history for event-driven automation
- Traceability from trigger event → script execution → step-by-step results
- Same data fidelity as scheduled executions (steps, timestamps, execution data)
- Foundation for trigger-based analytics and debugging

### ⚠️ Key Considerations

1. **Architecture Constraint**: Current `script_execution` table requires `schedule_id` (BIGINT NOT NULL) and `planned` (TIMESTAMP NOT NULL) - incompatible with trigger-based scripts
2. **Existing Endpoint**: `POST /v3/scripts/robot/scripts/:scriptReferenceId/executions/unscheduled` exists but only processes report steps, doesn't persist to database
3. **Event Flow**: Events flow from `megazord-events` → `m-o-triggers` → robot → `micro-manager`
4. **Event ID Source**: Need to verify if `eventId` is already propagated to robot when trigger fires
5. **Backward Compatibility**: Scheduled execution flow must remain unchanged
6. **API Versioning**: Consider v6 endpoint or extend existing v5/internal endpoints

---

## 📚 Background: Understanding Current Execution Flows

### Flow 1: Scheduled Script Execution (Current - ✅ Working)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Schedule Service (EVE)                                           │
│    - User creates scheduled task via wonkers-api                    │
│    - Schedule stored with cron expression, script_reference_id      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Robot (Tessa)                                                    │
│    - Fetches schedule at boot                                       │
│    - Executes script at scheduled time                              │
│    - Collects execution data (steps, timestamps, answers)           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Micro-Manager API                                                │
│    PUT /v2/scripts/robot/scripts/:scriptReferenceId/executions/    │
│        :scheduleId                                                  │
│                                                                     │
│    Headers:                                                         │
│      x-consumer-username: robot-123                                 │
│                                                                     │
│    Body: PutScriptExecutionDTO                                      │
│      - scriptVersionId: 456                                         │
│      - planned: "2025-12-06T10:00:00Z"  ← Required!                │
│      - scriptExecutionSteps: [...]                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. ScriptRobotController.scriptExecutionPut()                      │
│    - Extract scheduleId from URL params                             │
│    - Extract robotId from Kong headers (res.locals.robotId)        │
│    - Extract scriptExecution from validated body                    │
│    - Call ScriptExecutionService.saveExecution()                    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. ScriptExecutionService.saveExecution()                          │
│    a. Check if execution already exists (idempotency)               │
│       - getScriptExecutionId(robotId, scheduleId, planned, ...)     │
│                                                                     │
│    b. Validate execution steps                                      │
│       - validateExecutionSteps() verifies:                          │
│         * scriptStepId exists in script_version                     │
│         * nextScriptStepId (if present) exists in script_version    │
│                                                                     │
│    c. Create execution record if doesn't exist                      │
│       - addScriptExecution(robotId, scriptVersionId,                │
│                            scriptReferenceId, scheduleId, planned)  │
│       → INSERT INTO script_execution                                │
│         (script_reference_id, script_version_id,                    │
│          schedule_id, planned)  ← Both required!                    │
│       → Returns insertId (scriptExecutionId)                        │
│                                                                     │
│    d. Process execution steps                                       │
│       - Map steps with scriptExecutionId                            │
│       - For each step with stepType='report':                       │
│         * Call ReportingService.report()                            │
│         * Store response in step.data                               │
│                                                                     │
│    e. Save execution steps                                          │
│       - addScriptExecutionSteps(executionSteps)                     │
│       → Transaction:                                                │
│         - INSERT INTO script_step_execution (script_execution_id,   │
│             script_step_id, next_script_step_id, executed_at)       │
│         - For closedQuestion: INSERT INTO                           │
│             closed_question_execution_data (answer, probability)    │
│         - For multipleChoice: INSERT INTO                           │
│             multiple_choice_execution_data (answer, intentionType)  │
│         - For report: INSERT INTO report_execution_data             │
│             (sent, message)                                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. Database (MySQL tinybots.script_execution)                      │
│    ┌────┬─────────────┬──────────────┬────────────┬─────────────┐  │
│    │ id │ script_ref  │ script_ver   │ schedule_id│ planned     │  │
│    ├────┼─────────────┼──────────────┼────────────┼─────────────┤  │
│    │ 1  │ 123         │ 456          │ 789        │ 2025-12-06  │  │
│    │    │             │              │            │ 10:00:00    │  │
│    └────┴─────────────┴──────────────┴────────────┴─────────────┘  │
│                                                                     │
│    script_step_execution:                                           │
│    ┌────┬─────────────┬──────────────┬─────────────┬─────────────┐ │
│    │ id │ script_ex_id│ script_step  │ next_step   │ executed_at │ │
│    ├────┼─────────────┼──────────────┼─────────────┼─────────────┤ │
│    │ 10 │ 1           │ 20           │ 21          │ 10:00:05    │ │
│    │ 11 │ 1           │ 21           │ 22          │ 10:00:10    │ │
│    │ 12 │ 1           │ 22           │ NULL        │ 10:00:15    │ │
│    └────┴─────────────┴──────────────┴─────────────┴─────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Takeaways:**
- ✅ Requires `scheduleId` in URL
- ✅ Requires `planned` timestamp in body
- ✅ Full validation of steps against script version
- ✅ Transactional insert with rollback on error
- ✅ Report steps processed via ReportingService

---

### Flow 2: Unscheduled Script Execution (Current - ❌ Not Persisted)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Robot executes unscheduled script                                │
│    - Ad-hoc execution (e.g., via voice command, manual trigger)     │
│    - NOT from a trigger event (those don't exist yet)               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Micro-Manager API                                                │
│    POST /v3/scripts/robot/scripts/:scriptReferenceId/executions/   │
│         unscheduled                                                 │
│                                                                     │
│    Body: PostUnscheduledScriptExecutionDTO                          │
│      - scriptVersionId: 456                                         │
│      - scriptExecutionSteps: [...]                                  │
│      ⚠️ NO scheduleId, NO planned, NO eventId                       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. ScriptRobotControllerV3.scriptUnscheduledExecutionPost()        │
│    - Extract robotId from Kong headers                              │
│    - Extract scriptExecution from validated body                    │
│    - Call ScriptExecutionService.handleUnscheduledSteps()           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. ScriptExecutionService.handleUnscheduledSteps()                 │
│    ⚠️ CURRENT IMPLEMENTATION - MINIMAL:                             │
│                                                                     │
│    for (const step of execution.scriptExecutionSteps) {             │
│      if (step.stepType === 'report') {                              │
│        step.data = await this.reportingService.report(              │
│          robotId, null, execution.scriptVersionId,                  │
│          new Date(), step                                           │
│        )                                                            │
│      }                                                              │
│    }                                                                │
│                                                                     │
│    ❌ No validation of steps                                        │
│    ❌ No database insert                                            │
│    ❌ No execution record created                                   │
│    ❌ Only processes report steps                                   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Response: 204 No Content                                         │
│    ⚠️ Execution data is LOST - not stored anywhere                  │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Takeaways:**
- ❌ Execution steps are NOT persisted to database
- ❌ No validation of steps against script version
- ❌ Only report steps are processed (for side effects)
- ❌ Cannot query or analyze unscheduled executions
- ⚠️ This endpoint was designed for edge cases, not trigger-based automation

---

### Flow 3: Trigger-Based Script Execution (NEW - 🚧 To Be Implemented)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Event Source (Sensara, Robot Telemetry, External API)           │
│    Example: Robot enters "Living Room"                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Sensara Adaptor                                                  │
│    - Receives location event from Sensara SSE stream                │
│    - Maps to TinyBots event: LOCATION_LIVING_ROOM                   │
│    - POST to Megazord Events:                                       │
│      /internal/v1/events/robots/123/incomings                       │
│      Body: { eventName: "LOCATION_LIVING_ROOM", provider: ... }    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Megazord Events                                                  │
│    a. IncomingEventsService.createIncomingEvent()                   │
│       - Validates event schema & provider                           │
│       - INSERT INTO incoming_event (robot_id, event_schema_id,      │
│           event_provider_id, created_at)                            │
│       → Returns incomingEventId: 789012                             │
│                                                                     │
│    b. EventSubscriptionsService (listens to event emitter)          │
│       - Finds active subscriptions for robot 123 +                  │
│         event "LOCATION_LIVING_ROOM"                                │
│       - For TRIGGER_SUBSCRIPTION:                                   │
│         * Create outgoing_event record                              │
│         * POST to Trigger Service (m-o-triggers):                   │
│           /internal/v1/triggers/triggers                            │
│           Body: { robotId: 123, eventId: 789012,                    │
│                   eventName: "LOCATION_LIVING_ROOM" }               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. M-O-Triggers Service                                             │
│    - Receives trigger event                                         │
│    - Finds event trigger settings for robot 123 +                   │
│      event "LOCATION_LIVING_ROOM"                                   │
│    - Resolves trigger action: EXECUTE_SCRIPT → script_ref_id: 456   │
│    - Queues script execution command to robot via SQS/MQTT          │
│      ⚠️ Question: Does trigger payload include eventId?             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Robot (Tessa)                                                    │
│    - Receives execution command from queue                          │
│    - Fetches script from micro-manager:                             │
│      GET /v5/scripts/robot/scripts/:scriptReferenceId               │
│    - Executes script steps                                          │
│    - Collects execution data                                        │
│    ⚠️ Question: Does robot receive eventId in trigger command?      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. NEW Micro-Manager API (TO BE IMPLEMENTED)                        │
│    POST /v6/scripts/robot/scripts/:scriptReferenceId/executions/   │
│         triggered                                                   │
│                                                                     │
│    Headers:                                                         │
│      x-consumer-username: robot-123                                 │
│                                                                     │
│    Body: PostTriggeredScriptExecutionDTO (NEW!)                     │
│      - scriptVersionId: 456                                         │
│      - triggeringEventId: 789012  ← NEW: Link to incoming_event    │
│      - scriptExecutionSteps: [                                      │
│          { scriptStepId: 20, executedAt: "2025-12-06T10:30:05Z",   │
│            stepType: "say", data: null },                           │
│          { scriptStepId: 21, executedAt: "2025-12-06T10:30:10Z",   │
│            stepType: "closedQuestion",                              │
│            data: { answer: "yes", probability: 0.95,                │
│                    dataType: "closedQuestionData" } }               │
│        ]                                                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. NEW ScriptRobotControllerV6.scriptTriggeredExecutionPost()      │
│    (TO BE IMPLEMENTED)                                              │
│    - Extract robotId from Kong headers                              │
│    - Extract scriptReferenceId from URL params                      │
│    - Extract body: PostTriggeredScriptExecutionDTO                  │
│    - Call ScriptExecutionService.saveTriggeredExecution()           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 8. NEW ScriptExecutionService.saveTriggeredExecution()             │
│    (TO BE IMPLEMENTED)                                              │
│                                                                     │
│    a. Validate execution steps (reuse existing logic)               │
│       - validateExecutionSteps(scriptExecutionSteps, robotId,       │
│           scriptReferenceId, scriptVersionId)                       │
│       → Ensures scriptStepIds exist in script version               │
│                                                                     │
│    b. Create execution record with NULL schedule                    │
│       - addTriggeredScriptExecution(robotId, scriptVersionId,       │
│           scriptReferenceId, triggeringEventId)                     │
│       → INSERT INTO script_execution                                │
│         (script_reference_id, script_version_id,                    │
│          schedule_id, planned, triggering_event_id)                 │
│         VALUES (123, 456, NULL, NULL, 789012)  ← NEW!               │
│       → Returns insertId (scriptExecutionId)                        │
│                                                                     │
│    c. Process execution steps (same as scheduled)                   │
│       - Map steps with scriptExecutionId                            │
│       - For report steps: call ReportingService.report()            │
│                                                                     │
│    d. Save execution steps (same as scheduled)                      │
│       - addScriptExecutionSteps(executionSteps)                     │
│       → Transaction inserts into script_step_execution +            │
│         data tables (closed_question_execution_data, etc.)          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 9. Database (MySQL tinybots.script_execution) - NEW SCHEMA         │
│    ┌────┬─────────┬──────────┬────────────┬─────────┬────────────┐ │
│    │ id │ scr_ref │ scr_ver  │ schedule_id│ planned │ trigger_evt│ │
│    ├────┼─────────┼──────────┼────────────┼─────────┼────────────┤ │
│    │ 1  │ 123     │ 456      │ 789        │ 2025... │ NULL       │ │ ← Scheduled
│    │ 2  │ 456     │ 789      │ NULL       │ NULL    │ 789012     │ │ ← Triggered! 
│    └────┴─────────┴──────────┴────────────┴─────────┴────────────┘ │
│                                                                     │
│    ⚠️ Schema Change Required:                                       │
│    - ALTER TABLE script_execution                                   │
│      MODIFY COLUMN schedule_id BIGINT UNSIGNED NULL                 │
│    - ALTER TABLE script_execution                                   │
│      MODIFY COLUMN planned TIMESTAMP NULL                           │
│    - ALTER TABLE script_execution                                   │
│      ADD COLUMN triggering_event_id BIGINT UNSIGNED NULL            │
│    - CREATE INDEX idx_triggering_event_id                           │
│        ON script_execution(triggering_event_id)                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 10. Response: 201 Created                                           │
│     Body: { scriptExecutionId: 2 }                                  │
│                                                                     │
│     ✅ Execution fully persisted                                    │
│     ✅ Can query by triggeringEventId                               │
│     ✅ Traceability: Event 789012 → Execution 2                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Takeaways:**
- ✅ Full execution history like scheduled scripts
- ✅ Traceability from incoming event to execution
- ✅ Same validation & data structure as scheduled
- ⚠️ Requires schema migration (nullable columns)
- ⚠️ Need to verify eventId propagation through trigger flow

---

## 🔍 Key Components to Focus On

### 1. Database Schema (High Priority)

**Current Schema (from typ-e migration V42):**

```sql
-- script_execution table (current)
CREATE TABLE `script_execution` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `script_reference_id` BIGINT UNSIGNED NOT NULL,
  `script_version_id` BIGINT UNSIGNED NOT NULL,
  `schedule_id` BIGINT UNSIGNED NOT NULL,  -- ⚠️ BLOCKER: Cannot be NULL
  `planned` DATETIME NOT NULL,             -- ⚠️ BLOCKER: Cannot be NULL
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  FOREIGN KEY `fk_scheduled_script_schedule_id` (`schedule_id`) 
    REFERENCES `task_schedule` (`id`),
  CONSTRAINT FOREIGN KEY `script_execution_script_reference_id` (`script_reference_id`) 
    REFERENCES `script_reference` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT FOREIGN KEY `script_step_script_version_id` (`script_version_id`) 
    REFERENCES `script_version` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- script_step_execution table (stores individual step execution)
CREATE TABLE `script_step_execution` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `script_execution_id` BIGINT UNSIGNED NOT NULL,
  `script_step_id` BIGINT UNSIGNED NOT NULL,
  `executed_at` DATETIME(3) NOT NULL,  -- ✅ Already has timestamp per step!
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  CONSTRAINT FOREIGN KEY `script_step_execution_script_execution_id` (`script_execution_id`) 
    REFERENCES `script_execution` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT FOREIGN KEY `script_step_execution_script_step_id` (`script_step_id`) 
    REFERENCES `script_step` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Execution data tables (LEFT JOIN)
-- - closed_question_execution_data (answer, probability)
-- - multiple_choice_execution_data (answer, intention_type)
-- - report_execution_data (sent, message)
```

**⚠️ Note về `script_step_execution.executed_at`:**
- ✅ Đã có sẵn timestamp cho từng node/step execution
- ✅ Precision DATETIME(3) = milliseconds
- ✅ Requirement "time of execution of the node" đã được fulfill

---

**Migration Options:**

#### **Option 1: Make Columns Nullable (RECOMMENDED)**

**Pros:**
- ✅ Clean semantics: NULL = trigger-based, NOT NULL = scheduled
- ✅ Easy filtering: `WHERE schedule_id IS NULL` vs `WHERE schedule_id IS NOT NULL`
- ✅ No dummy/sentinel data
- ✅ Natural database design

**Cons:**
- ⚠️ Schema migration required (ALTER TABLE locks)
- ⚠️ Need to drop and recreate FK constraint
- ⚠️ Existing queries may need review (though most already filter by robot_id)

**Migration Script (Flyway V[XX]__add_triggered_execution_support.sql):**

```sql
-- Add columns for triggered executions
ALTER TABLE script_execution
  ADD COLUMN outgoing_event_id BIGINT UNSIGNED NULL
    COMMENT 'outgoing_event.id from megazord-events (NULL for scheduled)',
  ADD COLUMN incoming_event_id BIGINT UNSIGNED NULL
    COMMENT 'incoming_event.id from megazord-events (NULL for scheduled)';

-- Make scheduled columns nullable
ALTER TABLE script_execution
  MODIFY COLUMN schedule_id BIGINT UNSIGNED NULL
    COMMENT 'NULL for trigger-based executions, NOT NULL for scheduled',
  MODIFY COLUMN planned DATETIME NULL
    COMMENT 'NULL for trigger-based executions, scheduled time for scheduled';

-- Drop old FK constraint
ALTER TABLE script_execution 
  DROP FOREIGN KEY fk_scheduled_script_schedule_id;

-- Recreate FK allowing NULL
ALTER TABLE script_execution
  ADD CONSTRAINT fk_scheduled_script_schedule_id 
    FOREIGN KEY (schedule_id) 
    REFERENCES task_schedule(id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT;

-- Add indexes for trigger queries
CREATE INDEX idx_outgoing_event_id 
  ON script_execution(outgoing_event_id);

CREATE INDEX idx_incoming_event_id 
  ON script_execution(incoming_event_id);

-- Add check constraint (MySQL 8.0.16+)
ALTER TABLE script_execution
  ADD CONSTRAINT chk_execution_source
    CHECK (
      (schedule_id IS NOT NULL AND outgoing_event_id IS NULL AND incoming_event_id IS NULL) OR
      (schedule_id IS NULL AND (outgoing_event_id IS NOT NULL OR incoming_event_id IS NOT NULL))
    )
    COMMENT 'Execution must be either scheduled OR triggered (with event IDs)';

-- Add generated column for easy filtering (optional)
ALTER TABLE script_execution
  ADD COLUMN execution_type VARCHAR(20) 
    GENERATED ALWAYS AS (
      CASE 
        WHEN schedule_id IS NOT NULL THEN 'scheduled'
        WHEN outgoing_event_id IS NOT NULL THEN 'triggered'
        ELSE NULL
      END
    ) STORED
    COMMENT 'Type: scheduled or triggered';

CREATE INDEX idx_execution_type ON script_execution(execution_type);
```

**Rollback Script:**

```sql
-- Remove new columns
ALTER TABLE script_execution
  DROP COLUMN execution_type,
  DROP COLUMN incoming_event_id,
  DROP COLUMN outgoing_event_id;

-- Remove check constraint
ALTER TABLE script_execution
  DROP CONSTRAINT chk_execution_source;

-- Restore NOT NULL constraints
ALTER TABLE script_execution
  MODIFY COLUMN schedule_id BIGINT UNSIGNED NOT NULL,
  MODIFY COLUMN planned DATETIME NOT NULL;

-- Note: This rollback will FAIL if triggered executions exist in the table
-- Need to DELETE triggered executions first
```

---

#### **Option 2: Separate Table for Triggered Executions**

**Pros:**
- ✅ Zero risk to existing scheduled execution flow
- ✅ No migration of existing table
- ✅ Can add different columns specific to triggered executions
- ✅ Easy to query each type separately
- ✅ Gradual rollout possible (keep both systems running)

**Cons:**
- ⚠️ Need to query two tables for "all executions"
- ⚠️ More complex queries with UNION
- ⚠️ Duplicate some columns (script_reference_id, script_version_id)
- ⚠️ Need to update `script_step_execution` FK to reference both tables

**Schema Design:**

```sql
-- New table for triggered executions
CREATE TABLE `script_execution_triggered` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `script_reference_id` BIGINT UNSIGNED NOT NULL,
  `script_version_id` BIGINT UNSIGNED NOT NULL,
  `outgoing_event_id` BIGINT UNSIGNED NOT NULL,
  `incoming_event_id` BIGINT UNSIGNED NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  CONSTRAINT FOREIGN KEY `script_execution_triggered_script_reference_id` 
    (`script_reference_id`) REFERENCES `script_reference` (`id`) 
    ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT FOREIGN KEY `script_execution_triggered_script_version_id` 
    (`script_version_id`) REFERENCES `script_version` (`id`) 
    ON UPDATE RESTRICT ON DELETE RESTRICT,
  INDEX idx_outgoing_event_id (outgoing_event_id),
  INDEX idx_incoming_event_id (incoming_event_id),
  INDEX idx_script_reference_created (script_reference_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Option A: Keep script_step_execution as is, add discriminator
ALTER TABLE script_step_execution
  ADD COLUMN execution_type ENUM('scheduled', 'triggered') NOT NULL DEFAULT 'scheduled'
    COMMENT 'Type of parent execution';

-- Option B: Separate step execution tables (more complex)
CREATE TABLE `script_step_execution_triggered` (
  -- Same structure as script_step_execution
  -- FK to script_execution_triggered instead
);
```

**Query Pattern (Union for "all executions"):**

```sql
-- Get all executions for a robot (scheduled + triggered)
SELECT 
  'scheduled' as type,
  se.id, 
  se.script_reference_id, 
  se.script_version_id,
  se.created_at,
  se.schedule_id,
  NULL as outgoing_event_id,
  NULL as incoming_event_id
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = ?

UNION ALL

SELECT 
  'triggered' as type,
  set.id,
  set.script_reference_id,
  set.script_version_id,
  set.created_at,
  NULL as schedule_id,
  set.outgoing_event_id,
  set.incoming_event_id
FROM script_execution_triggered set
JOIN script_reference sr ON set.script_reference_id = sr.id
WHERE sr.robot_id = ?

ORDER BY created_at DESC;
```

---

#### **Option 3: Sentinel Value (Quick Fix - NOT RECOMMENDED)**

**Pros:**
- ✅ No schema change
- ✅ Fastest to implement

**Cons:**
- ❌ Confusing semantics (schedule_id=0 means "no schedule")
- ❌ Need dummy record in task_schedule table
- ❌ Queries become complex (`WHERE schedule_id != 0`)
- ❌ planned timestamp is meaningless (use created_at or dummy date)
- ❌ Hard to distinguish from actual schedule_id=0 if it exists

**Implementation:**

```sql
-- Insert sentinel schedule record
INSERT INTO task_schedule (id, ...) VALUES (0, 'SENTINEL_TRIGGER', ...);

-- For triggered executions:
INSERT INTO script_execution 
  (script_reference_id, script_version_id, schedule_id, planned)
VALUES (123, 456, 0, NOW());  -- schedule_id=0 = triggered

-- Add event columns
ALTER TABLE script_execution
  ADD COLUMN outgoing_event_id BIGINT UNSIGNED NULL,
  ADD COLUMN incoming_event_id BIGINT UNSIGNED NULL;
```

**❌ Not recommended** - This is a hack that will cause maintenance issues later.

---

#### **Recommendation: Option 1 (Nullable Columns)**

**Rationale:**
- ✅ Most natural database design
- ✅ Same table = simpler queries, no UNION needed
- ✅ Clear semantics (NULL vs NOT NULL)
- ✅ Similar pattern to other tables in TinyBots
- ⚠️ Migration risk is manageable with proper planning
- ⚠️ Can use pt-online-schema-change for zero downtime if needed

**Queries After Migration:**

```sql
-- Get all triggered executions
SELECT * FROM script_execution 
WHERE triggering_event_id IS NOT NULL;

-- Get executions for specific event
SELECT se.*, incoming_event.event_name
FROM script_execution se
JOIN incoming_event ON se.triggering_event_id = incoming_event.id
WHERE se.triggering_event_id = 789012;

-- Get all executions for robot (scheduled + triggered)
SELECT se.*, 
  CASE 
    WHEN se.schedule_id IS NOT NULL THEN 'scheduled'
    WHEN se.triggering_event_id IS NOT NULL THEN 'triggered'
  END AS execution_type
FROM script_execution se
JOIN script_reference sr ON se.script_reference_id = sr.id
WHERE sr.robot_id = 123
ORDER BY se.created_at DESC;
```

---

### 2. Repository Layer (`ScriptExecutionRepository.ts`)

**Current Method (Scheduled):**

```typescript
public addScriptExecution = async ({ 
  robotId, 
  scriptVersionId, 
  scriptReferenceId, 
  scheduleId,   // ← Required
  planned       // ← Required
}: { 
  robotId: number
  scriptVersionId: number
  scriptReferenceId: number
  scheduleId: number     // ← Required
  planned: string | Date // ← Required
}) => {
  const sRef = await this.scriptRepository.getScriptReference({ 
    id: scriptReferenceId, 
    robotId 
  })
  
  const dbRes = await this.database.generalQuery<any>(
    this.ADD_SCRIPT_EXECUTION, 
    [sRef[0].id, scriptVersionId, scheduleId, planned]
  )
  
  return dbRes as OkPacket
}

private ADD_SCRIPT_EXECUTION = `
  INSERT INTO script_execution 
    (script_reference_id, script_version_id, schedule_id, planned) 
  VALUES(?, ?, ?, ?)`
```

**NEW Method (Triggered) - To Implement:**

```typescript
public addTriggeredScriptExecution = async ({ 
  robotId, 
  scriptVersionId, 
  scriptReferenceId, 
  outgoingEventId,  // ← NEW: From trigger command
  incomingEventId   // ← OPTIONAL: Can be fetched or passed
}: { 
  robotId: number
  scriptVersionId: number
  scriptReferenceId: number
  outgoingEventId: number  // ← NEW
  incomingEventId?: number // ← OPTIONAL
}): Promise<OkPacket> => {
  // Verify script reference belongs to robot
  const sRef = await this.scriptRepository.getScriptReference({ 
    id: scriptReferenceId, 
    robotId 
  })
  
  if (!sRef || sRef.length === 0) {
    throw new NotFoundError(
      `Script reference ${scriptReferenceId} not found for robot ${robotId}`
    )
  }

  const dbRes = await this.database.generalQuery<any>(
    this.ADD_TRIGGERED_SCRIPT_EXECUTION, 
    [sRef[0].id, scriptVersionId, outgoingEventId, incomingEventId || null]
  )
  
  return dbRes as OkPacket
}

private ADD_TRIGGERED_SCRIPT_EXECUTION = `
  INSERT INTO script_execution 
    (script_reference_id, script_version_id, schedule_id, planned, 
     outgoing_event_id, incoming_event_id) 
  VALUES (?, ?, NULL, NULL, ?, ?)`
  // ⚠️ NULL for schedule_id and planned since this is triggered
```

**Query Methods - To Implement:**

```typescript
// Get executions by outgoing event ID
private GET_EXECUTIONS_BY_OUTGOING_EVENT = `
  SELECT se.*, sr.robot_id
  FROM script_execution se
  JOIN script_reference sr ON se.script_reference_id = sr.id
  WHERE se.outgoing_event_id = ?`

public getExecutionsByOutgoingEvent = async (
  outgoingEventId: number
): Promise<any[]> => {
  return this.database.generalQuery<any>(
    this.GET_EXECUTIONS_BY_OUTGOING_EVENT, 
    [outgoingEventId]
  )
}

// Get executions by incoming event ID
private GET_EXECUTIONS_BY_INCOMING_EVENT = `
  SELECT se.*, sr.robot_id
  FROM script_execution se
  JOIN script_reference sr ON se.script_reference_id = sr.id
  WHERE se.incoming_event_id = ?`

public getExecutionsByIncomingEvent = async (
  incomingEventId: number
): Promise<any[]> => {
  return this.database.generalQuery<any>(
    this.GET_EXECUTIONS_BY_INCOMING_EVENT, 
    [incomingEventId]
  )
}

// Get triggered executions for robot within date range
private GET_TRIGGERED_EXECUTIONS = `
  SELECT se.*, sr.robot_id
  FROM script_execution se
  JOIN script_reference sr ON se.script_reference_id = sr.id
  WHERE sr.robot_id = ?
    AND se.outgoing_event_id IS NOT NULL
    AND se.created_at >= ?
    AND se.created_at <= ?
  ORDER BY se.created_at DESC
  LIMIT ?`

public getTriggeredExecutions = async ({
  robotId,
  from,
  to,
  limit = 100
}: {
  robotId: number
  from: Date | string
  to: Date | string
  limit?: number
}): Promise<any[]> => {
  return this.database.generalQuery<any>(
    this.GET_TRIGGERED_EXECUTIONS,
    [robotId, from, to, limit]
  )
}
```

---

### 3. Service Layer (`ScriptExecutionService.ts`)

**Current Method (Scheduled):**

```typescript
public async saveExecution ({ 
  robotId, 
  scriptReferenceId, 
  scriptExecution, 
  scheduleId,        // ← Required
  planned,           // ← Required
  scriptVersionId 
}: {
  robotId: number
  scriptReferenceId: number
  scriptExecution: PutScriptExecutionDTO
  scheduleId: number       // ← Required
  planned: Date            // ← Required
  scriptVersionId: number
}) {
  // 1. Check if execution already exists (idempotency)
  const executionId = await this.scriptExecutionRepository.getScriptExecutionId({
    robotId,
    scriptReferenceId,
    scriptVersionId: scriptExecution.scriptVersionId,
    scheduleId,
    planned
  })

  let okPacket
  
  // 2. Validate steps
  await this.validateExecutionSteps(
    scriptExecution.scriptExecutionSteps, 
    robotId, 
    scriptReferenceId, 
    scriptVersionId
  )
  
  // 3. Create execution if doesn't exist
  if (!executionId) {
    okPacket = await this.scriptExecutionRepository.addScriptExecution({ 
      robotId, 
      scriptReferenceId, 
      scriptVersionId, 
      scheduleId, 
      planned 
    })
  }
  
  const scriptExecutionId = executionId || okPacket.insertId
  
  if (scriptExecutionId) {
    // 4. Process steps
    const executionSteps = scriptExecution.scriptExecutionSteps
      .map(step => ({ ...step, scriptExecutionId }))
    
    // 5. Handle report steps
    for (const step of executionSteps) {
      if (step.stepType === 'report') {
        step.data = await this.reportingService.report(
          robotId, 
          scriptExecutionId, 
          scriptExecution.scriptVersionId, 
          planned,  // ← Uses planned time
          step
        )
      }
    }
    
    // 6. Save steps
    await this.scriptExecutionRepository.addScriptExecutionSteps(executionSteps)
  } else {
    throw Error('Failed to create script execution')
  }
}
```

**NEW Method (Triggered) - To Implement:**

```typescript
public async saveTriggeredExecution ({ 
  robotId, 
  scriptReferenceId, 
  scriptVersionId,
  outgoingEventId,  // ← NEW: Event ID from trigger command
  incomingEventId,  // ← OPTIONAL: Can be fetched or passed
  scriptExecutionSteps 
}: {
  robotId: number
  scriptReferenceId: number
  scriptVersionId: number
  outgoingEventId: number  // ← NEW
  incomingEventId?: number // ← OPTIONAL
  scriptExecutionSteps: ExecutionStepDTO[]
}): Promise<number> {
  // 1. Validate steps (reuse existing logic)
  await this.validateExecutionSteps(
    scriptExecutionSteps, 
    robotId, 
    scriptReferenceId, 
    scriptVersionId
  )

  // 2. Create execution record with outgoing event
  const okPacket = await this.scriptExecutionRepository
    .addTriggeredScriptExecution({ 
      robotId, 
      scriptReferenceId, 
      scriptVersionId, 
      outgoingEventId,
      incomingEventId
    })

  const scriptExecutionId = okPacket.insertId

  if (!scriptExecutionId) {
    throw new InternalServerError('Failed to create triggered script execution')
  }

  // 3. Process steps (same as scheduled)
  const executionSteps = scriptExecutionSteps
    .map(step => ({ ...step, scriptExecutionId }))

  // 4. Handle report steps
  for (const step of executionSteps) {
    if (step.stepType === 'report') {
      step.data = await this.reportingService.report(
        robotId, 
        scriptExecutionId, 
        scriptVersionId, 
        new Date(),  // ← Use current time since no planned time
        step
      )
    }
  }

  // 5. Save steps (same as scheduled)
  await this.scriptExecutionRepository.addScriptExecutionSteps(executionSteps)

  return scriptExecutionId
}

// Reused private method (no changes)
private validateExecutionSteps = async (
  executionSteps: ExecutionStepDTO[],
  robotId: number, 
  scriptReferenceId: number, 
  scriptVersionId: number
) => {
  const stepIds = executionSteps.map(s => s.scriptStepId)
  const nextStepIds = executionSteps.map(s => s.nextScriptStepId)
  
  const dbIds = await this.stepsRepository.getScriptStepIds({ 
    robotId, 
    scriptReferenceId, 
    scriptVersionId 
  })
  
  stepIds.forEach(stepId => {
    if (!dbIds.includes(stepId)) {
      throw new CustomError('Invalid execution steps', 403)
    }
  })
  
  nextStepIds.forEach(stepId => {
    if (stepId && !dbIds.includes(stepId)) {
      throw new CustomError('Invalid execution steps', 403)
    }
  })
}
```

**Key Design Decision:**
- ✅ Reuse `validateExecutionSteps()` (same logic for scheduled & triggered)
- ✅ Reuse `addScriptExecutionSteps()` (same step structure)
- ✅ Different creation methods (different SQL, different params)
- ✅ Report processing uses `new Date()` instead of `planned` time

---

### 4. DTOs & Schemas (`schemas/body/ScriptExecution.ts`)

**Current DTO (Scheduled):**

```typescript
export class PutScriptExecutionDTO {
  @IsOptional()
  @IsInt()
  id?: number

  @IsInt()
  scriptVersionId: number

  @Transform(sqlDateParser)
  @IsDate()
  planned: Date  // ← Required for scheduled

  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ExecutionStepDTO)
  scriptExecutionSteps?: ExecutionStepDTO[]
}
```

**Current DTO (Unscheduled - Incomplete):**

```typescript
export class PostUnscheduledScriptExecutionDTO {
  @IsInt()
  scriptVersionId: number

  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ExecutionStepDTO)
  scriptExecutionSteps?: ExecutionStepDTO[]
  
  // ❌ Missing: triggeringEventId
  // ❌ Not used for persistence currently
}
```

**NEW DTO (Triggered) - To Implement:**

```typescript
export class PostTriggeredScriptExecutionDTO {
  @IsInt()
  @Min(1)
  scriptVersionId: number

  @IsInt()
  @Min(1)
  triggeringEventId: number  // ← NEW: Link to incoming_event.id

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ExecutionStepDTO)
  scriptExecutionSteps: ExecutionStepDTO[]  // Required (not optional)
}

export class TriggeredScriptExecutionResponse {
  @IsInt()
  scriptExecutionId: number
}
```

**Execution Step DTO (Shared - No Changes):**

```typescript
export class ExecutionStepDTO {
  @IsInt()
  scriptStepId: number

  @IsIn(['say', 'wait', 'closedQuestion', 'multipleChoice', 'report', 'statusCheck'])
  public stepType: 'say' | 'wait' | 'closedQuestion' | 
                   'multipleChoice' | 'report' | 'statusCheck'

  @IsInt()
  @IsOptional()
  nextScriptStepId?: number | null

  @Transform(sqlDateParser)
  @IsDate()
  executedAt: Date | string

  @IsOptional()
  data?: ExecutionData | null  // ClosedQuestionData | MultipleChoiceData | ReportData
}
```

---

### 5. Controller Layer (To Implement)

**NEW Controller:**

```typescript
// src/controllers/ScriptTriggeredExecutionController.ts

import { Request, Response, NextFunction } from 'express'
import { ScriptExecutionService } from '../services/ScriptExecutionService'
import { 
  PostTriggeredScriptExecutionDTO, 
  TriggeredScriptExecutionResponse 
} from '../schemas/body/ScriptExecution'

export class ScriptTriggeredExecutionController {
  constructor(
    private scriptExecutionService: ScriptExecutionService
  ) {}

  public postTriggeredExecution = async (
    req: Request, 
    res: Response<TriggeredScriptExecutionResponse>, 
    next: NextFunction
  ) => {
    try {
      // Extract from Kong auth middleware
      const robotId = parseInt(res.locals.robotId)
      
      // Extract from URL params
      const scriptReferenceId = parseInt(req.params.scriptReferenceId)
      
      // Extract from validated request body
      const body: PostTriggeredScriptExecutionDTO = res.locals.scriptExecution

      // Save execution
      const scriptExecutionId = await this.scriptExecutionService
        .saveTriggeredExecution({
          robotId,
          scriptReferenceId,
          scriptVersionId: body.scriptVersionId,
          triggeringEventId: body.triggeringEventId,
          scriptExecutionSteps: body.scriptExecutionSteps
        })

      // Return 201 Created with execution ID
      res.status(201).json({ scriptExecutionId })
    } catch (error) {
      next(error)
    }
  }
}
```

**Route Registration:**

```typescript
// src/routes/routes.ts

import { ScriptTriggeredExecutionController } from '../controllers/ScriptTriggeredExecutionController'
import { postTriggeredScriptExecution } from '../middleware/validation/body'
import { checkRobotAccess } from '../middleware/validation/access'
import validation from 'express-joi-validation'
import scriptReferenceIdSchema from '../schemas/params/scriptReferenceIdSchema'
import kongHeaderSchema from '../schemas/headers/kongHeaderSchema'

const joiValidator = validation.createValidator({ passError: true })

export = (app: Express.Application, container: AwilixContainer) => {
  const scriptTriggeredExecutionController: ScriptTriggeredExecutionController = 
    container.resolve('scriptTriggeredExecutionController')

  // ... existing routes ...

  // NEW: Triggered execution endpoint
  app.post(
    '/v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered',
    joiValidator.headers(kongHeaderSchema),
    joiValidator.params(scriptReferenceIdSchema),
    checkRobotAccess,  // Verifies robot identity matches :robotId from Kong
    postTriggeredScriptExecution,  // Validates body, sets res.locals.scriptExecution
    scriptTriggeredExecutionController.postTriggeredExecution
  )
}
```

**Validation Middleware:**

```typescript
// src/middleware/validation/body.ts

export const postTriggeredScriptExecution = async (
  req: Request, 
  res: Response, 
  next: NextFunction
) => {
  try {
    const execution = plainToInstance(
      PostTriggeredScriptExecutionDTO, 
      req.body
    )
    
    const errors = await validate(execution)
    
    if (errors.length > 0) {
      const messages = errors.map(err => 
        Object.values(err.constraints || {}).join(', ')
      )
      throw new BadRequestError(
        `Validation failed: ${messages.join('; ')}`
      )
    }
    
    res.locals.scriptExecution = execution
    next()
  } catch (error) {
    next(error)
  }
}
```

---

### 6. Dependency Injection (`buildContainer.ts`)

```typescript
// src/buildContainer.ts

import { ScriptTriggeredExecutionController } from './controllers/ScriptTriggeredExecutionController'

export default () => {
  const container = createContainer()

  container.register({
    // ... existing registrations ...

    // Repositories (already registered)
    scriptExecutionRepository: asClass(ScriptExecutionRepository).singleton(),
    stepsRepository: asClass(StepsRepository).singleton(),
    
    // Services (already registered)
    scriptExecutionService: asClass(ScriptExecutionService).singleton(),
    reportingService: asClass(ReportingService).singleton(),
    
    // NEW Controller
    scriptTriggeredExecutionController: asClass(
      ScriptTriggeredExecutionController
    ).singleton()
  })

  return container
}
```

---

## 🔄 Implementation Plan

### Phase 1: Database Schema Migration

**Priority**: Critical | **Effort**: 1-2 days | **Dependencies**: None

**Tasks:**

1. [ ] **Create Migration Script**
   - File: `migrations/YYYYMMDD_add_triggered_execution_support.sql`
   - Make `schedule_id` and `planned` nullable
   - Add `triggering_event_id` column
   - Add `trigger_type` computed column (optional)
   - Add check constraint
   - Add index on `triggering_event_id`
   - Drop/recreate FK constraint

2. [ ] **Test Migration on Dev Database**
   - Run migration on local dev database
   - Verify existing scheduled executions are queryable
   - Test INSERT with NULL schedule_id
   - Test INSERT with triggering_event_id
   - Verify indexes created

3. [ ] **Rollback Script**
   - Create reverse migration
   - Test rollback procedure

**Acceptance Criteria:**
- ✅ Migration runs without errors
- ✅ Can insert execution with `schedule_id=NULL, planned=NULL, triggering_event_id=123`
- ✅ Can insert execution with `schedule_id=789, planned='2025-12-06', triggering_event_id=NULL`
- ✅ Existing queries return same results
- ✅ FK constraint allows NULL

**Risks:**
- ⚠️ Downtime during migration (lock table)
- ⚠️ FK constraint recreation may fail if orphaned records exist

---

### Phase 2: Repository Layer Implementation

**Priority**: High | **Effort**: 2-3 days | **Dependencies**: Phase 1 (schema migration)

**Tasks:**

1. [ ] **Update `ScriptExecutionRepository.ts`**
   - Add `ADD_TRIGGERED_SCRIPT_EXECUTION` SQL statement
   - Implement `addTriggeredScriptExecution()` method
   - Implement `getExecutionsByTriggeringEvent()` method
   - Implement `getTriggeredExecutions()` method (date range query)

2. [ ] **Unit Tests**
   - File: `test/IT/repositoryIT/ScriptExecutionRepository.IT.spec.ts`
   - Test `addTriggeredScriptExecution()` inserts with NULL schedule
   - Test `getExecutionsByTriggeringEvent()` returns correct rows
   - Test `getTriggeredExecutions()` date filtering
   - Test error handling (invalid robot/script)

**Acceptance Criteria:**
- ✅ `addTriggeredScriptExecution()` returns insertId
- ✅ Query methods return correct data structure
- ✅ Error handling throws NotFoundError for invalid script
- ✅ Tests pass with 100% coverage of new methods

---

### Phase 3: Service Layer Implementation

**Priority**: High | **Effort**: 2-3 days | **Dependencies**: Phase 2 (repository layer)

**Tasks:**

1. [ ] **Update `ScriptExecutionService.ts`**
   - Implement `saveTriggeredExecution()` method
   - Reuse `validateExecutionSteps()` (no changes)
   - Handle report steps with `new Date()` instead of `planned`

2. [ ] **Unit Tests**
   - File: `test/services/ScriptExecutionService.UT.spec.ts`
   - Test `saveTriggeredExecution()` calls repository correctly
   - Test validation of steps
   - Test report processing
   - Test error propagation

**Acceptance Criteria:**
- ✅ Method validates steps before insert
- ✅ Method creates execution record with triggeringEventId
- ✅ Method saves execution steps
- ✅ Report steps are processed
- ✅ Tests pass with >90% coverage

---

### Phase 4: DTO & Schema Definition

**Priority**: High | **Effort**: 1 day | **Dependencies**: None (can run parallel)

**Tasks:**

1. [ ] **Update `schemas/body/ScriptExecution.ts`**
   - Create `PostTriggeredScriptExecutionDTO` class
   - Create `TriggeredScriptExecutionResponse` class
   - Add validation decorators

2. [ ] **Create Joi Schema (if needed)**
   - File: `schemas/validation/TriggeredScriptExecutionSchema.ts`
   - Define Joi validation schema for body

3. [ ] **Unit Tests**
   - Test DTO validation (valid/invalid cases)

**Acceptance Criteria:**
- ✅ DTO validates required fields
- ✅ triggeringEventId must be integer > 0
- ✅ scriptExecutionSteps must be non-empty array

---

### Phase 5: Controller & Route Implementation

**Priority**: High | **Effort**: 2 days | **Dependencies**: Phase 3 (service), Phase 4 (DTOs)

**Tasks:**

1. [ ] **Create `ScriptTriggeredExecutionController.ts`**
   - Implement `postTriggeredExecution()` handler
   - Add error handling
   - Add request logging

2. [ ] **Create Validation Middleware**
   - File: `middleware/validation/body.ts`
   - Implement `postTriggeredScriptExecution` validator

3. [ ] **Update `routes/routes.ts`**
   - Add POST route for triggered executions
   - Wire Kong auth, checkRobotAccess, validation

4. [ ] **Update `buildContainer.ts`**
   - Register new controller

5. [ ] **Integration Tests**
   - File: `test/controllers/ScriptTriggeredExecutionController.IT.spec.ts`
   - Test 201 Created response
   - Test 400 Bad Request (invalid body)
   - Test 403 Forbidden (wrong robot)
   - Test 404 Not Found (invalid script)

**Acceptance Criteria:**
- ✅ Endpoint returns 201 with scriptExecutionId
- ✅ Kong auth enforced
- ✅ Robot access check enforced
- ✅ Body validation enforced
- ✅ Error responses have correct status codes
- ✅ Integration tests pass

---

### Phase 6: API Documentation (tiny-specs)

**Priority**: Medium | **Effort**: 1-2 days | **Dependencies**: Phase 5 (implementation complete)

**Tasks:**

1. [ ] **Create OpenAPI Schema**
   - File: `tiny-specs/specs/micro-manager/TriggeredScriptExecution.yaml`
   - Define `PostTriggeredScriptExecutionDTO` schema
   - Define `TriggeredScriptExecutionResponse` schema
   - Add examples

2. [ ] **Update Main Spec**
   - File: `tiny-specs/specs/micro-manager/micro-manager.yaml`
   - Add endpoint: `POST /v6/scripts/robot/scripts/{scriptReferenceId}/executions/triggered`
   - Reference schemas
   - Document responses (201, 400, 403, 404, 500)

3. [ ] **Update micro-manager Local Docs**
   - File: `micro-manager/docs/micro-manager.yaml`
   - Add endpoint specification
   - Add examples

**Acceptance Criteria:**
- ✅ OpenAPI spec validates without errors
- ✅ Swagger UI renders endpoint correctly
- ✅ All fields documented
- ✅ Examples are realistic

---

### Phase 7: Testing & Documentation

**Priority**: Medium | **Effort**: 2-3 days | **Dependencies**: Phase 6 (docs)

**Tasks:**

1. [ ] **End-to-End Integration Tests**
   - Test full flow: request → database → response
   - Test with real MySQL database
   - Test concurrent executions
   - Test idempotency (duplicate requests)

2. [ ] **Update Repository Overview**
   - File: `devdocs/tinybots/micro-manager/OVERVIEW.md`
   - Document new triggered execution flow
   - Update controller/service descriptions

3. [ ] **Deployment Guide**
   - Document migration steps
   - Document rollback procedure
   - Document monitoring/alerts to add

4. [ ] **Backward Compatibility Testing**
   - Verify scheduled executions still work
   - Run full regression test suite

**Acceptance Criteria:**
- ✅ E2E tests pass
- ✅ Regression tests pass
- ✅ Documentation updated
- ✅ Performance within acceptable range (<100ms overhead)

---

### Phase 8: Event ID Propagation (Cross-Service Coordination)

**Priority**: Critical | **Effort**: 3-5 days | **Dependencies**: Phase 1-7 (micro-manager ready)

⚠️ **This phase requires coordination with other services**

**Investigation Tasks:**

1. [ ] **Verify Current Event Flow**
   - Trace: megazord-events → m-o-triggers → robot
   - Question: Does robot receive eventId when trigger fires?
   - Check m-o-triggers trigger payload structure
   - Check robot firmware/SDK

2. [ ] **Update m-o-triggers (if needed)**
   - Add `eventId` to trigger payload sent to robot
   - Update trigger queue message structure
   - Test with robot simulator

3. [ ] **Update Robot SDK (if needed)**
   - Parse `eventId` from trigger command
   - Include in execution report to micro-manager
   - Update firmware

**Acceptance Criteria:**
- ✅ eventId flows from megazord to robot
- ✅ Robot includes eventId in execution report
- ✅ E2E test: event → trigger → execution → storage

**Risks:**
- ⚠️ Robot firmware update required (deployment coordination)
- ⚠️ m-o-triggers changes may affect other consumers

---

## 🚧 Outstanding Issues & Follow-up

### ✅ RESOLVED Issues

#### ✅ Question 1: Database Schema Options

**Original Question**: Chỉ có Option 1 (nullable columns)?

**Answer**: Đã thêm **3 options** với phân tích đầy đủ:

- **Option 1: Make Columns Nullable** (RECOMMENDED)
  - ✅ Clean semantics, easy filtering
  - ⚠️ Requires migration with table lock
  
- **Option 2: Separate Table (`script_execution_triggered`)**
  - ✅ Zero risk to existing flow
  - ⚠️ Complex queries with UNION
  
- **Option 3: Sentinel Value** (NOT RECOMMENDED)
  - ✅ Quick fix, no migration
  - ❌ Confusing semantics, maintenance nightmare

**Recommendation**: **Option 1** for long-term maintainability

---

#### ✅ Question 2: Event ID Propagation

**Original Question**: Robot có receive eventId không?

**Answer**: ✅ **TriggerId is available!**

**Implementation**:

- Robot receives `triggerId` from m-o-triggers in trigger command
- Robot reports execution with `triggerId` to micro-manager
- Micro-manager queries local `tinybots` database:
  - Get `outgoing_event_id` from `event_trigger` table
  - Get `incoming_event_id` from `outgoing_event.source_event_id`

**Database Columns**:

```sql
outgoing_event_id BIGINT UNSIGNED NOT NULL  -- Resolved from triggerId
incoming_event_id BIGINT UNSIGNED NOT NULL  -- Resolved from outgoing_event
```

**✅ All queries within same database - No cross-service API calls!**

---

#### ✅ Question 3: "Node executed, with data" - Đã có chưa?

**Original Question**: Requirement này schedule execution đã có chưa?

**Answer**: ✅ **Fully implemented!**

**What Exists (Scheduled Executions)**:

1. ✅ **Node executed**: `script_step_execution` table
   - `script_step_id`: Which node/step was executed
   - `next_script_step_id`: Next node in flow

2. ✅ **With data**: Separate tables per step type
   - `closed_question_execution_data`: answer, probability
   - `multiple_choice_execution_data`: answer, intention_type
   - `report_execution_data`: sent, message

3. ✅ **Time of execution**: `executed_at DATETIME(3)` per step
   - Millisecond precision
   - Individual timestamp for each node

**What We Need to Add (Triggered Executions)**:

- ❌ Only need to add parent record in `script_execution` with event IDs
- ✅ **Reuse 100% of step execution logic** (no changes!)
- ✅ Same tables, same methods, same data structure

**Key Insight**: Requirements 1 & 2 are **already fulfilled**. Only need Requirement 3 (eventId tracking).

---

### ⚠️ Critical Decisions Required

#### Decision 1: Event ID Propagation Strategy ✅ RESOLVED

**Question**: Does robot currently receive `eventId` when trigger fires?

**Answer from Investigation**:
✅ **TriggerId is available** from m-o-triggers

**Implementation Strategy**:

1. Robot receives `triggerId` from m-o-triggers in trigger command
2. Robot sends `triggerId` to micro-manager when reporting execution
3. Micro-manager queries **local tinybots database** to resolve event IDs:
   - Query `event_trigger` table to get `outgoing_event_id` from `triggerId`
   - Query `outgoing_event` table to get `source_event_id` (incoming_event_id)
4. Micro-manager stores both `outgoing_event_id` and `incoming_event_id`

**Database Schema (All tables in tinybots DB)**:

```sql
-- M-O-Triggers table (in tinybots DB)
CREATE TABLE event_trigger (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  outgoing_event_id BIGINT UNSIGNED NOT NULL,  -- Links to outgoing_event
  -- ... other trigger fields
);

-- Megazord-Events tables (in tinybots DB)
CREATE TABLE outgoing_event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  source_event_id BIGINT UNSIGNED NOT NULL,  -- FK to incoming_event
  subscription_id BIGINT UNSIGNED NOT NULL,
  status VARCHAR(64) NOT NULL,
  FOREIGN KEY (source_event_id) REFERENCES incoming_event(id) ON DELETE CASCADE
);

CREATE TABLE incoming_event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  robot_id BIGINT UNSIGNED NOT NULL,
  event_schema_id BIGINT UNSIGNED NOT NULL,
  -- ... other event fields
);

-- Micro-Manager table (in tinybots DB)
CREATE TABLE script_execution (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  -- ... existing fields
  outgoing_event_id BIGINT UNSIGNED NULL,  -- Resolved from triggerId
  incoming_event_id BIGINT UNSIGNED NULL   -- Resolved from outgoing_event.source_event_id
);
```

**Event Traceability Chain**:

```text
1. Sensara → Megazord: Creates incoming_event (id: 789012) in tinybots DB
2. Megazord: Creates outgoing_event (id: 999, source_event_id: 789012) in tinybots DB
3. M-O-Triggers: Creates event_trigger (id: 555, outgoing_event_id: 999) in tinybots DB
4. M-O-Triggers → Robot: Sends triggerId: 555
5. Robot → Micro-Manager: Reports execution with triggerId: 555
6. Micro-Manager:
   a. Query tinybots.event_trigger WHERE id = 555
      → Get outgoing_event_id: 999
   b. Query tinybots.outgoing_event WHERE id = 999
      → Get source_event_id (incoming_event_id): 789012
   c. Store in script_execution:
      - outgoing_event_id: 999
      - incoming_event_id: 789012
```

**DTO Changes**:

```typescript
export class PostTriggeredScriptExecutionDTO {
  @IsInt()
  @Min(1)
  scriptVersionId: number

  @IsInt()
  @Min(1)
  triggerId: number  // ← From m-o-triggers command

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ExecutionStepDTO)
  scriptExecutionSteps: ExecutionStepDTO[]
}
```

**Query Logic in Service**:

```typescript
public async saveTriggeredExecution({
  triggerId,
  scriptVersionId,
  scriptExecutionSteps
}: {
  triggerId: number
  scriptVersionId: number
  scriptExecutionSteps: ExecutionStepDTO[]
}): Promise<number> {
  // 1. Resolve outgoing_event_id from triggerId
  const trigger = await this.database.query(
    'SELECT outgoing_event_id FROM event_trigger WHERE id = ?',
    [triggerId]
  )
  const outgoingEventId = trigger[0].outgoing_event_id

  // 2. Resolve incoming_event_id from outgoing_event
  const outgoingEvent = await this.database.query(
    'SELECT source_event_id FROM outgoing_event WHERE id = ?',
    [outgoingEventId]
  )
  const incomingEventId = outgoingEvent[0].source_event_id

  // 3. Create execution with both event IDs
  const okPacket = await this.scriptExecutionRepository
    .addTriggeredScriptExecution({
      robotId,
      scriptReferenceId,
      scriptVersionId,
      outgoingEventId,
      incomingEventId
    })

  // ... rest of execution logic
}
```

**✅ All data in same database - No cross-service API calls needed!**

---

#### Decision 2: API Versioning Strategy

**Question**: Should we use v6 endpoint or extend existing v5?

**Options**:

**Option A: New v6 Endpoint (Recommended)**
```
POST /v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered
```
**Pros**:
- ✅ Clear separation of concerns
- ✅ Easier to version independently
- ✅ Simple rollback (just disable v6)
- ✅ Explicit intent in URL

**Cons**:
- ⚠️ Robot firmware needs update to call new endpoint
- ⚠️ Two execution endpoints to maintain

**Option B: Extend Existing Unscheduled Endpoint**
```
POST /v3/scripts/robot/scripts/:scriptReferenceId/executions/unscheduled
  + Add optional triggeringEventId field
```
**Pros**:
- ✅ No robot firmware change
- ✅ Backward compatible

**Cons**:
- ⚠️ Ambiguous: "unscheduled" != "triggered"
- ⚠️ Optional field may be forgotten
- ⚠️ Harder to enforce eventId requirement later

**Recommendation**: **Option A (v6)** for clarity and future-proofing

---

#### Decision 3: Migration Timing

**Question**: When to run database migration?

**Considerations**:
- Migration locks `script_execution` table during FK constraint modification
- Risk of downtime if table is large

**Options**:

**Option A: Maintenance Window**
- Schedule during low-traffic period (e.g., 2-4 AM)
- Announce downtime to users
- Timeline: 5-10 minutes expected

**Option B: Online Migration (Percona pt-online-schema-change)**
- Use pt-online-schema-change for zero-downtime
- Creates shadow table, copies data, swaps atomically
- Timeline: 30-60 minutes (no downtime)

**Option C: Blue-Green Deployment**
- Run migration on replica
- Promote replica to primary
- Timeline: Requires DB infrastructure setup

**Recommendation**: **Option B** if table is large (>1M rows), else **Option A**

---

### ⚠️ Clarifications Needed

#### Clarification 1: Duplicate Execution Prevention

**Question**: Can the same trigger event cause multiple script executions?

**Scenarios**:
- Event fires → Robot executes script → Robot crashes mid-execution
- Robot retries execution → Sends duplicate report to micro-manager
- Should we deduplicate? Or allow multiple executions per event?

**Proposed Solution**:
- Add unique constraint: `UNIQUE(script_reference_id, triggering_event_id)`
- Or add `execution_attempt` column to allow retries

**Impact**: Need to clarify retry/idempotency requirements

---

#### Clarification 2: Query/Reporting Requirements

**Question**: What queries/reports will be built on triggered executions?

**Possible Use Cases**:
- "Show all executions triggered by event X"
- "Show all executions for robot Y in last 7 days"
- "Compare scheduled vs triggered execution counts"
- "Find failed trigger executions"

**Impact**: May need additional indexes:
```sql
CREATE INDEX idx_robot_created 
  ON script_execution(script_reference_id, created_at);

CREATE INDEX idx_trigger_type 
  ON script_execution(trigger_type, created_at);
```

---

#### Clarification 3: Execution Completeness Tracking

**Question**: How to know if triggered execution completed successfully?

**Current State**:
- Scheduled executions: Inferred from presence of steps
- No explicit "status" field (e.g., RUNNING, COMPLETED, FAILED)

**Proposed Enhancement**:
```sql
ALTER TABLE script_execution
  ADD COLUMN status ENUM('in_progress', 'completed', 'failed') 
    DEFAULT 'in_progress';

-- Update to 'completed' when all steps received
-- Update to 'failed' if robot reports error
```

**Impact**: May require additional API for status updates

---

#### Clarification 4: Cross-Service Error Handling

**Question**: What happens if micro-manager API call fails?

**Scenarios**:
- Robot executes script successfully
- Reports to micro-manager → 500 Internal Server Error
- Execution data lost

**Proposed Solutions**:
- Robot retries with exponential backoff
- Robot persists execution locally (SQLite) until ack
- Micro-manager returns 202 Accepted + async processing

**Impact**: Reliability requirements, need SLA definition

---

## 📊 Summary of Results

> **Status**: Planning Phase Complete ✅

### ✅ Key Findings

1. **Requirements Analysis**:
   - ✅ "Node executed with data" - **Already implemented** in scheduled executions
   - ✅ "Time of execution" - **Already implemented** via `script_step_execution.executed_at`
   - ❌ "Event ID tracking" - **Need to add** (primary objective)

2. **Event ID Propagation**:
   - ✅ **Resolved**: Robot sends `triggerId` from m-o-triggers
   - ✅ Micro-manager queries local `tinybots` DB to resolve event IDs
   - ✅ No cross-service API calls needed (all tables in same DB)

3. **Database Schema**:
   - ✅ Analyzed 3 migration options
   - ✅ **Recommended**: Option 1 (Nullable columns)
   - ✅ Migration script ready for review

4. **Implementation Scope**:
   - ✅ Can **reuse 100%** of step execution logic
   - ✅ Only need new parent record creation method
   - ✅ Estimated effort: 10-15 days across 8 phases

### 📋 Deliverables Ready

- [x] Full flow diagrams (scheduled vs triggered vs new)
- [x] Database schema analysis with 3 options
- [x] Migration scripts (Option 1)
- [x] Code examples for all layers (Repository, Service, Controller, DTO)
- [x] Implementation plan (8 phases with tasks)
- [x] Test strategy and acceptance criteria
- [x] Risk analysis and rollback procedures

### 🎯 Next Steps

**Phase 1: Database Migration** (Priority: Critical)

1. Review migration script with DBA
2. Choose migration strategy (maintenance window vs pt-online-schema-change)
3. Test migration on dev/staging
4. Schedule production migration

**Phase 2-7: Implementation** (Priority: High)

1. Implement repository layer
2. Implement service layer
3. Create DTOs and validation
4. Create controller and routes
5. Write API documentation
6. Write tests (unit + integration)

**Phase 8: Cross-Team Coordination** (Priority: Medium)

1. Verify `outgoingEventId` in m-o-triggers payload
2. Coordinate robot firmware update (if needed)
3. Plan gradual rollout

### 📈 Success Metrics

**Technical Metrics**:

- [ ] Migration completes with <10min downtime (or zero with pt-online)
- [ ] API latency <100ms (p95)
- [ ] Test coverage >90%
- [ ] Zero regression in scheduled execution flow

**Business Metrics**:

- [ ] 100% of triggered executions are stored
- [ ] Full traceability: event → execution → steps
- [ ] Analytics dashboard shows trigger vs scheduled ratio
- [ ] Debug time reduced by 50% for trigger issues

---

## 📚 References & Related Work

### Related Tickets
- [251126] Store Executed Script - Similar work, provides pattern to follow

### External Documentation
- [Megazord Events OVERVIEW](devdocs/tinybots/megazord-events/OVERVIEW.md)
- [Sensara Adaptor OVERVIEW](devdocs/tinybots/sensara-adaptor/OVERVIEW.md)
- [M-O-Triggers OVERVIEW](devdocs/tinybots/m-o-triggers/OVERVIEW.md)
- [TinyBots Platform OVERVIEW](devdocs/tinybots/OVERVIEW.md)

### Technical References
- MySQL Nullable Columns: https://dev.mysql.com/doc/refman/8.0/en/data-type-defaults.html
- MySQL Check Constraints: https://dev.mysql.com/doc/refman/8.0/en/create-table-check-constraints.html
- Percona pt-online-schema-change: https://docs.percona.com/percona-toolkit/pt-online-schema-change.html
