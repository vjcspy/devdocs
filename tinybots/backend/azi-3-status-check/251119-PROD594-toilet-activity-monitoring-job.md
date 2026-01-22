# 📋 [PROD594: 2025-11-19] - Toilet Activity Monitoring Job (Reuse azi-3-status-check)

[TOC]

## User Requirements

This is an experiment for a specific user (we could think about a more general approach, but speed is more important. A very specific approach for this user is fine)

What you need:

- add NO_TOILET_ACTIVITY_ALARM to megazord event schema

It should do the following:

- Between time A and time B do the following
- At time A subscribe to TOILET_ACTIVITY event in megazord events and check from now to 2 hours in the future for a specific user
- If a TOILET_ACTIVITY event is received in these 2 hours plan a new check from now to 2 hours in the future (or until time B, which ever comes first)
- If no TOILET_ACTIVITY event is received after 2 hours, send a new event to megazord event called NO_TOILET_ACTIVITY_ALARM
  then start a new check from now to 2 hours in the future (or until time B, which ever comes first)

## 🎯 Objective

Implement toilet activity monitoring as a **separate job mode** within the existing `azi-3-status-check` repository, completely isolated from the current robot-initiated status check functionality. The job will autonomously monitor TOILET_ACTIVITY events in 2-hour rolling windows and emit alarms when gaps are detected.

### ⚠️ Key Considerations

- **Zero impact on existing functionality**: The new job mode (`src/cmd/job`) must be completely separate from the existing API mode (`src/cmd/app`)
- **Code reuse without modification**: Leverage existing services (MegazordEventsService, repositories, SQS consumers) without changing their implementation
- **Shared infrastructure**: Use existing database tables (`status_check*`), SQS setup, and dependency injection patterns
- **Independent deployment**: Job mode can be deployed separately with different config, scaling independently from API mode
- **Template-free approach**: Create status checks programmatically without requiring Micro-Managers script integration
- **Rolling window logic**: Different from existing fixed-window evaluation - implement as new service without touching existing StatusChecksService
- **Conflict resolution**: Both modes consume same SQS queue - must implement discriminator to prevent cross-processing

### 📊 System Architecture & Data Flow

#### End-to-End Flow Diagram

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                            SENSARA / ROBOT APP                              │
│                                                                             │
│  • Resident uses toilet                                                    │
│  • Sensor detects activity                                                 │
│  • Posts event to Megazord                                                 │
│    POST /internal/v1/events/robots/{robotId}/incomings                      │
│    Body: { providerName: "sensara", eventName: "TOILET_ACTIVITY", ... }     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTP POST
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MEGAZORD-EVENTS SERVICE                             │
│                                                                             │
│  1. EventsController.createIncomingEvent()                                  │
│     • Validates event schema                                                │
│     • Stores in incoming_event table                                        │
│     • Returns incoming_event_id                                             │
│                                                                             │
│  2. EventsService.fanOutEvent()                                             │
│     • Queries event_subscription table                                      │
│     • Finds subscriptions WHERE:                                            │
│         robot_id = {robotId}                                                │
│         AND event_names CONTAINS 'TOILET_ACTIVITY'                          │
│         AND until > NOW()                                                   │
│     • Creates outgoing_event records                                        │
│                                                                             │
│  3. For SERVICE_SUBSCRIPTION type:                                          │
│     • Publishes to STATUS_QUEUE (AWS SQS)                                  │
│     • Message payload:                                                     │
│       {                                                                    │
│         subscriptionId: 123,                                               │
│         sourceEvent: {                                                     │
│           id: 456,                                                         │
│           robotId: 789,                                                    │
│           eventName: "TOILET_ACTIVITY",                                    │
│           createdAt: "2025-11-20T09:30:00Z",                               │
│           payload: { ... }                                                 │
│         }                                                                  │
│       }                                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ AWS SQS Message
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│  AZI-3-STATUS-CHECK (API Mode)  │   │  AZI-3-STATUS-CHECK (Job Mode)  │
│                                 │   │                                 │
│  StatusChecksService.init()     │   │  ToiletMonitoringWorker.init()  │
│  • Polls statusQueue            │   │  • Polls statusQueue            │
│  • handleStatusCheckSQSMessage()│   │  • handleMessage()              │
│                                 │   │                                 │
│  Filter Logic:                  │   │  Filter Logic:                  │
│  1. Load status_check by        │   │  1. Check eventName ==          │
│     subscriptionId              │   │     "TOILET_ACTIVITY"           │
│  2. IF step_id = 0 OR NULL:     │   │  2. Load status_check by        │
│     → SKIP (job mode record)    │   │     subscriptionId              │
│     → ack() and return          │   │  3. IF step_id > 0:             │
│  3. IF step_id > 0:             │   │     → SKIP (API mode record)    │
│     → Process with template     │   │     → ack() and return          │
│       validation                │   │  4. IF step_id = 0:             │
│     → Update result             │   │     → Process toilet monitoring │
│     → Notify robot via          │   │     → Reset window              │
│       robotQueue                │   │     → Update poller             │
│                                 │   │                                 │
│  ⚠️  CONFLICT PREVENTION:      │   │  ⚠️  CONFLICT PREVENTION:       │
│  Must add step_id filter to     │   │  Must check step_id to avoid    │
│  skip job mode records          │   │  processing API mode records    │
└─────────────────────────────────┘   └─────────────────────────────────┘
                                                      │
                                                      │ 2-hour gap detected
                                                      ▼
                                    ┌─────────────────────────────────────┐
                                    │  ToiletMonitoringService            │
                                    │  .emitAlarmAndContinue()            │
                                    │                                     │
                                    │  • POST to Megazord Events:         │
                                    │    /internal/v1/events/robots/      │
                                    │    {robotId}/incomings              │
                                    │  • Event:                           │
                                    │    NO_TOILET_ACTIVITY_ALARM         │
                                    │  • Record in status_check_record    │
                                    │  • Reset window or finalize         │
                                    └─────────────────────────────────────┘
                                                      │
                                                      │ Alarm event posted
                                                      ▼
                                    ┌─────────────────────────────────────┐
                                    │  MEGAZORD-EVENTS SERVICE            │
                                    │                                     │
                                    │  • Stores alarm as incoming_event   │
                                    │  • Fans out to subscribers          │
                                    │    (dashboards, triggers, etc.)     │
                                    │  • Available for reporting          │
                                    └─────────────────────────────────────┘
```

**Key Insights from Flow**:

1. **Single SQS Queue**: Both API mode and Job mode consume same `statusQueue`
2. **Discriminator Required**: Must filter by `step_id` (or other discriminator) to route correctly
3. **Megazord is Central Hub**: All events (input + alarms) flow through Megazord
4. **Subscription-Based Routing**: Megazord uses `event_subscription` table to determine who gets what
5. **Job Mode is Publisher**: Posts alarms back to Megazord, creating feedback loop

### 🗄️ Database Schema Analysis

This section details all database tables used by toilet monitoring job mode, including purpose, columns, and usage patterns.

#### Table 1: `status_check` (Main Monitoring Session)

**Purpose**: Primary table tracking each monitoring session. One row = one day's monitoring for one robot.

**Schema**:

```sql
CREATE TABLE `status_check` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `robot_id` INT UNSIGNED NOT NULL,
  `status_check_template_id` INT UNSIGNED NOT NULL,
  `phase` VARCHAR(64) NOT NULL,
  `subscription_id` BIGINT UNSIGNED DEFAULT NULL,
  `result` VARCHAR(64) DEFAULT NULL,
  `step_id` BIGINT UNSIGNED NOT NULL,              -- ⚠️ DISCRIMINATOR
  `next_step_id` BIGINT UNSIGNED DEFAULT NULL,
  `script_version_id` BIGINT UNSIGNED NOT NULL,    -- ⚠️ NOT NULL constraint
  `scheduled_id` BIGINT UNSIGNED NOT NULL,         -- ⚠️ NOT NULL constraint
  `planned_at` TIMESTAMP NOT NULL,
  `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `idx_status_check_result_template_robot_id` (`robot_id`, `status_check_template_id`, `result`),
  CONSTRAINT `fk_status_check_robot_id` FOREIGN KEY (`robot_id`) REFERENCES `robot_account` (`id`),
  CONSTRAINT `fk_status_check_template_id` FOREIGN KEY (`status_check_template_id`) 
    REFERENCES `status_check_template` (`id`),
  CONSTRAINT `fk_status_check_step_id` FOREIGN KEY (`step_id`) 
    REFERENCES `script_step` (`id`),
  CONSTRAINT `fk_status_check_subscription_id` FOREIGN KEY (`subscription_id`) 
    REFERENCES `event_subscription` (`id`)
) ENGINE = InnoDB;
```

**Column Details**:

| Column                     | Type        | Job Mode Value           | API Mode Value                  | Notes                            |
|----------------------------|-------------|--------------------------|---------------------------------|----------------------------------|
| `id`                       | BIGINT      | Auto-increment           | Auto-increment                  | Primary key                      |
| `robot_id`                 | INT         | Target robot ID          | Robot from script               | Which robot is monitored         |
| `status_check_template_id` | INT         | Toilet template ID       | Template from script            | Links to template metadata       |
| `phase`                    | VARCHAR(64) | `FUTURE` → `COMPLETED`   | `PAST` → `FUTURE` → `COMPLETED` | Current lifecycle phase          |
| `subscription_id`          | BIGINT      | Megazord subscription ID | Megazord subscription ID        | Links to event_subscription      |
| `result`                   | VARCHAR(64) | `PENDING` → `COMPLETED`  | `PENDING` → `PASS`/`FAIL`       | Final outcome                    |
| `step_id`                  | BIGINT      | **0 (discriminator)**    | **> 0 (has script)**            | ⚠️ KEY: Distinguishes job vs API |
| `next_step_id`             | BIGINT      | NULL                     | Script next step                | Not used by job mode             |
| `script_version_id`        | BIGINT      | **0 (dummy)**            | Script version                  | ⚠️ NOT NULL constraint           |
| `scheduled_id`             | BIGINT      | **0 (dummy)**            | Schedule ID                     | ⚠️ NOT NULL constraint           |
| `planned_at`               | TIMESTAMP   | Time A of monitoring day | Script execution time           | When monitoring starts           |

**Usage in Job Mode**:

- **Create**: At Time A daily, insert row with `step_id=0`, `phase=FUTURE`, `result=PENDING`
- **Update**: Set `subscription_id` after Megazord subscription, update `result` when complete
- **Query**: Load by `id` or by `robot_id + subscription_id`
- **Filter**: Both modes must filter by `step_id` to avoid conflicts

**Critical Constraint Issues**:

- `step_id`, `script_version_id`, `scheduled_id` have NOT NULL + FK constraints
- Job mode needs dummy values (0) OR schema migration to make nullable
- See "Decision 3" in Outstanding Issues for resolution options

---

#### Table 2: `status_check_poller` (Rolling Window Tracking)

**Purpose**: Tracks current 2-hour monitoring window. Updated every time activity is detected or window expires.

**Schema**:

```sql
CREATE TABLE `status_check_poller` (
  `status_check_id` BIGINT UNSIGNED NOT NULL,
  `since` TIMESTAMP NOT NULL,
  `until` TIMESTAMP NOT NULL,
  `lock_id` VARCHAR(256) DEFAULT NULL,
  `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`status_check_id`),
  INDEX `idx_status_check_poller_lock_id` (`lock_id`),
  INDEX `idx_status_check_poller_until` (`until`),
  CONSTRAINT `fk_status_check_poller_check_id` FOREIGN KEY (`status_check_id`) 
    REFERENCES `status_check` (`id`) ON DELETE CASCADE
) ENGINE = InnoDB;
```

**Column Details**:

| Column            | Job Mode Behavior                               | Purpose                            |
|-------------------|-------------------------------------------------|------------------------------------|
| `status_check_id` | FK to status_check                              | Links to parent monitoring session |
| `since`           | **Window start time** (resets on activity)      | Current monitoring window start    |
| `until`           | **Window end time** (since + 2 hours or Time B) | Current monitoring window end      |
| `lock_id`         | Used by window expiration monitor               | Prevents concurrent processing     |
| `updated_at`      | **Critical**: Updated on every window reset     | Tracks last activity/reset time    |

**Lifecycle Example**:

```sql
-- Initial creation at Time A (08:00)
INSERT INTO status_check_poller (status_check_id, since, until)
VALUES (123, '2025-11-20 08:00:00', '2025-11-20 10:00:00');

-- Activity detected at 09:30 → Reset window
UPDATE status_check_poller 
SET since = '2025-11-20 09:30:00',
    until = '2025-11-20 11:30:00',
    updated_at = NOW()
WHERE status_check_id = 123;

-- No activity by 11:30 → Window expires
-- Window monitor locks row
UPDATE status_check_poller
SET lock_id = 'window-monitor-uuid'
WHERE status_check_id = 123 AND until <= NOW() AND lock_id IS NULL;

-- Emit alarm, then reset for next window
UPDATE status_check_poller
SET since = '2025-11-20 11:30:00',
    until = '2025-11-20 13:30:00',
    lock_id = NULL,
    updated_at = NOW()
WHERE status_check_id = 123;

-- Continue until Time B (20:00)
-- If now >= Time B, delete poller and mark status_check complete
```

**Query Patterns**:

- **Find expired windows**: `WHERE until <= NOW() AND lock_id IS NULL AND until < {TimeB}`
- **Update on activity**: Direct UPDATE via `StatusChecksRepository.updatePollerWindow()`
- **Check current window**: `WHERE status_check_id = ?`

**Important Notes**:

- No `setPollerWindow()` method in ActiveModel - must use direct SQL UPDATE
- `updated_at` is automatic, tracks last modification
- One-to-one relationship with `status_check` (cascade delete)

---

#### Table 3: `status_check_record` (Event & Alarm Audit Trail)

**Purpose**: Chronological log of all TOILET_ACTIVITY events and NO_TOILET_ACTIVITY_ALARM emissions during monitoring session.

**Schema**:

```sql
CREATE TABLE `status_check_record` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `status_check_id` BIGINT UNSIGNED NOT NULL,
  `incoming_event_id` BIGINT UNSIGNED DEFAULT NULL,
  `outgoing_event_id` BIGINT UNSIGNED DEFAULT NULL,
  `event_name` VARCHAR(256) NOT NULL,
  `order` INT UNSIGNED NOT NULL,
  `phase` VARCHAR(64) NOT NULL,
  `result` VARCHAR(64) NOT NULL,
  `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`, `phase`),
  CONSTRAINT `fk_status_check_record_check_id` FOREIGN KEY (`status_check_id`) 
    REFERENCES `status_check` (`id`) ON DELETE CASCADE,
  CONSTRAINT `fk_status_check_record_incoming_event_id` FOREIGN KEY (`incoming_event_id`) 
    REFERENCES `incoming_event` (`id`),
  CONSTRAINT `fk_status_check_record_outgoing_event_id` FOREIGN KEY (`outgoing_event_id`) 
    REFERENCES `outgoing_event` (`id`)
) ENGINE = InnoDB;
```

**Column Details**:

| Column              | Purpose                       | Job Mode Usage                                  |
|---------------------|-------------------------------|-------------------------------------------------|
| `status_check_id`   | Link to monitoring session    | FK to parent status_check                       |
| `incoming_event_id` | Link to TOILET_ACTIVITY event | Set when activity detected                      |
| `outgoing_event_id` | Link to alarm event           | Set when alarm emitted                          |
| `event_name`        | Event type                    | 'TOILET_ACTIVITY' or 'NO_TOILET_ACTIVITY_ALARM' |
| `order`             | Sequence number               | Auto-increment per status_check                 |
| `phase`             | Phase when recorded           | Always 'FUTURE' for job mode                    |
| `result`            | Record outcome                | 'EVENT_OBSERVED', 'ALARM_SENT', etc.            |
| `created_at`        | Timestamp                     | Event time or alarm emission time               |

**Example Records for One Monitoring Session**:

```sql
-- 1. Activity at 09:30
INSERT INTO status_check_record 
(status_check_id, incoming_event_id, event_name, `order`, phase, result, created_at)
VALUES (123, 456, 'TOILET_ACTIVITY', 1, 'FUTURE', 'EVENT_OBSERVED', '2025-11-20 09:30:00');

-- 2. No activity for 2 hours → Alarm at 11:30
INSERT INTO status_check_record
(status_check_id, outgoing_event_id, event_name, `order`, phase, result, created_at)
VALUES (123, 789, 'NO_TOILET_ACTIVITY_ALARM', 2, 'FUTURE', 'ALARM_SENT', '2025-11-20 11:30:00');

-- 3. Activity at 12:00
INSERT INTO status_check_record
(status_check_id, incoming_event_id, event_name, `order`, phase, result, created_at)
VALUES (123, 460, 'TOILET_ACTIVITY', 3, 'FUTURE', 'EVENT_OBSERVED', '2025-11-20 12:00:00');

-- ... continue throughout day
```

**Usage in Job Mode**:

- **Insert on activity**: Via `StatusCheckActiveModel.addRecord()` when TOILET_ACTIVITY received
- **Insert on alarm**: Via `addRecord()` after posting NO_TOILET_ACTIVITY_ALARM to Megazord
- **Query history**: Load all records for debugging/reporting: `WHERE status_check_id = ? ORDER BY \`order\``

**Benefits**:

- Complete audit trail of monitoring session
- Can reconstruct timeline of events
- Debugging tool for missed/duplicate alarms
- Links to actual Megazord event IDs for cross-reference

---

#### Table 4: `status_check_template` (Monitoring Template Metadata)

**Purpose**: Defines toilet monitoring template metadata. Job mode uses this for consistency but bypasses template validation logic.

**Schema**:

```sql
CREATE TABLE `status_check_template` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `template_description_id` INT UNSIGNED NOT NULL,
  `past_event_dependencies` VARCHAR(512) NOT NULL,
  `future_event_dependencies` VARCHAR(512) NOT NULL,
  `version` SMALLINT UNSIGNED NOT NULL DEFAULT 1,
  `is_active` BOOLEAN NOT NULL DEFAULT TRUE,
  `poll_until_end` BOOLEAN NOT NULL DEFAULT FALSE,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  CONSTRAINT UNIQUE KEY `status_check_template_name_version` 
    (`template_description_id`, `version`),
  CONSTRAINT `fk_status_check_template_to_description` FOREIGN KEY (`template_description_id`) 
    REFERENCES `status_check_description` (`id`)
) ENGINE = InnoDB;
```

**Toilet Monitoring Template Row**:

```sql
INSERT INTO status_check_template (
  template_description_id,
  past_event_dependencies,
  future_event_dependencies,
  version,
  is_active,
  poll_until_end
) VALUES (
  (SELECT id FROM status_check_description WHERE name = 'TOILET_ACTIVITY_MONITORING'),
  '[]',                                      -- No past dependencies
  '[{"eventName":"TOILET_ACTIVITY"}]',       -- Subscribe to TOILET_ACTIVITY
  1,                                         -- Version 1
  TRUE,                                      -- Active
  TRUE                                       -- Poll until Time B
);
```

**Job Mode Usage**:

- **Read-only**: Load template ID to store in `status_check.status_check_template_id`
- **No validation**: Job mode doesn't use EJS expressions or template evaluation
- **Metadata only**: Used for consistent naming and reporting

**Not Used by Job Mode**:

- `status_check_template_detail` table (EJS expressions)
- Template validation/evaluation logic
- Past/future dependency parsing

---

#### Table 5: `status_check_description` (Template Naming)

**Purpose**: Human-readable names and descriptions for status check types.

**Schema**:

```sql
CREATE TABLE `status_check_description` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(128) NOT NULL,
  `description` TEXT,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_status_check_description_name` (`name`)
) ENGINE = InnoDB;
```

**Toilet Monitoring Row**:

```sql
INSERT INTO status_check_description (name, description)
VALUES (
  'TOILET_ACTIVITY_MONITORING',
  'Autonomous monitoring of toilet activity in 2-hour rolling windows. Emits alarm when no activity detected.'
);
```

**Usage**: Referenced by template, provides display name for UI/reports.

---

#### Table 6: `script_next_multiple_choice` (Not Used - Potential Issue)

**Purpose**: Defines next script steps for robot status checks (API mode only).

**⚠️ Problem for Job Mode**:

- `StatusCheckActiveModel.load()` calls `loadNextStepResources()`
- This method queries `script_next_multiple_choice` using `step_id`
- If `step_id=0`, query returns empty → throws error

**Solutions**:

1. **Create dummy records** for `step_id=0`
2. **Modify `loadNextStepResources()`** to skip if `step_id=0` (recommended)
3. **Don't use ActiveModel** in job mode (use direct repository queries)

**Recommended Implementation** (Option 2):

```typescript
// In StatusCheckActiveModel.ts
private async loadNextStepResources(): Promise<void> {
  // Skip for job mode records
  if (!this.model.stepId || this.model.stepId === 0) {
    this.model.nextStepsResource = new domains.StatusCheckNextStepsResource()
    return
  }
  
  // Existing logic for API mode...
  const res = await this.database.query<Record<string, any>[]>(
    StatusCheckActiveModel.GET_STATUS_CHECK_NEXT_STEPS,
    [this.model.stepId]
  )
  // ... rest of code
}
```

---

#### Table 7: `event_subscription` (Megazord - External)

**Purpose**: Megazord-owned table tracking active event subscriptions. Job mode creates subscriptions here via MegazordEventsService API.

**Relevant Columns**:

- `id`: Subscription ID (stored in `status_check.subscription_id`)
- `robot_id`: Target robot
- `event_names`: JSON array of subscribed events (e.g., `["TOILET_ACTIVITY"]`)
- `subscription_type`: `SERVICE_SUBSCRIPTION` (routes to statusQueue)
- `until`: Expiration timestamp (Time B)

**Job Mode Interaction**:

```typescript
// Subscribe at Time A
const subscription = await megazordEventsService.subscribe(
  ctx,
  robotId,
  ['TOILET_ACTIVITY'],
  timeB.toISOString()
)
// Store subscription.id in status_check

// Unsubscribe at Time B or when finalizing
await megazordEventsService.unsubscribe(ctx, robotId, subscription.id)
```

---

#### Table 8: `incoming_event` (Megazord - External)

**Purpose**: Megazord-owned table storing all incoming events. TOILET_ACTIVITY events stored here first.

**Job Mode Usage**:

- **Read-only**: Get `incoming_event.id` from SQS message to store in `status_check_record.incoming_event_id`
- **No direct writes**: Job mode doesn't write to this table (only Sensara/robots do)
- **Alarm events**: When job mode posts NO_TOILET_ACTIVITY_ALARM, Megazord creates `incoming_event` row

---

#### Table 9: `outgoing_event` (Megazord - External)

**Purpose**: Megazord-owned table for outgoing event notifications. Links SQS messages to source events.

**Job Mode Usage**:

- **Receive via SQS**: Worker receives `outgoing_event` wrapped in SQS message
- **Alarm creation**: When posting alarm, Megazord creates `outgoing_event` and returns ID
- **Record linking**: Store `outgoing_event.id` in `status_check_record.outgoing_event_id` for alarms

---

### 🗂️ Database Tables Summary

| Table                         | Owned By           | Job Mode Access       | API Mode Access       | Conflict Risk                          |
|-------------------------------|--------------------|-----------------------|-----------------------|----------------------------------------|
| `status_check`                | azi-3-status-check | Read/Write            | Read/Write            | 🔥 **HIGH** - Need discriminator       |
| `status_check_poller`         | azi-3-status-check | Read/Write            | Read/Write            | ⚠️ **MEDIUM** - Filter by step_id      |
| `status_check_record`         | azi-3-status-check | Write                 | Read                  | ✅ **LOW** - Append-only                |
| `status_check_template`       | azi-3-status-check | Read                  | Read                  | ✅ **NONE** - Static data               |
| `status_check_description`    | azi-3-status-check | Read                  | Read                  | ✅ **NONE** - Static data               |
| `script_next_multiple_choice` | Micro-Managers     | NOT USED              | Read                  | ⚠️ **ISSUE** - Load error if step_id=0 |
| `event_subscription`          | Megazord           | Create/Delete via API | Create/Delete via API | ✅ **NONE** - Separate subscriptions    |
| `incoming_event`              | Megazord           | Read (from SQS)       | Read (from SQS)       | ✅ **NONE** - Read-only                 |
| `outgoing_event`              | Megazord           | Read (from SQS)       | Read (from SQS)       | ✅ **NONE** - Read-only                 |

## 🔄 Implementation Plan

[Don't require running any test]

### Phase 1: Analysis & Preparation

- [x] Analyze detailed requirements
  - **Outcome**: Job mode will create its own bootstrap (cmd/job) completely separate from cmd/app. It will schedule daily monitoring jobs at configured times (Time A), subscribe to Megazord events, track 2-hour rolling windows in status_check_poller, and emit alarms when gaps detected. Uses same database/services but different execution path - zero overlap with robot API flows.

- [x] Define scope and edge cases
  - **Outcome**: Must handle:
    - Multiple monitoring windows per day (if B-A > 2 hours)
    - DST transitions (23/25-hour days)
    - Service restarts mid-window (recover from DB state)
    - Duplicate SQS messages (idempotent alarm emission)
    - Subscription cleanup on service shutdown
    - Robot offline periods (continue monitoring, alarm if needed)
    - Time B occurring mid-window (truncate to B, finalize)
    - Configuration updates without restart (reload from config/DB)
    - **Conflict resolution**: Both API mode and Job mode consume same `statusQueue` - must distinguish records

- [x] Database Tables Analysis
  - **Outcome**: See detailed table breakdown in Phase 1.5 below

### Phase 1.5: Database Tables Analysis & Conflict Resolution Strategy

#### 📊 Database Tables Usage

| Table                           | Purpose                 | Columns for Job Mode                                                                                                                                                                                                                                          | Key Considerations                                                                                    |
|---------------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| **status_check**                | Main monitoring session | • `robot_id`: Target robot<br>• `status_check_template_id`: toilet template<br>• `phase`: FUTURE → COMPLETED<br>• `subscription_id`: Megazord subscription<br>• `result`: PENDING → COMPLETED<br>• `step_id`: **TBD discriminator**<br>• `planned_at`: Time A | **⚠️ CONFLICT**: API mode also queries this table. Need discriminator to separate job vs API records  |
| **status_check_poller**         | 2-hour window tracking  | • `since`: Current window start<br>• `until`: Current window end<br>• `updated_at`: Updates on every reset                                                                                                                                                    | • Must UPDATE directly (no setPollerWindow method)<br>• Existing teardown scheduler also queries this |
| **status_check_record**         | Event/alarm audit trail | • `event_name`: TOILET_ACTIVITY / NO_TOILET_ACTIVITY_ALARM<br>• `incoming_event_id`: Link to events<br>• `result`: EVENT_OBSERVED / ALARM_SENT                                                                                                                | • Safe - read-only from API mode<br>• Use ActiveModel.addRecord()                                     |
| **status_check_template**       | Template metadata       | • Seed once with toilet monitoring config                                                                                                                                                                                                                     | • Job mode bypasses evaluation logic                                                                  |
| **script_next_multiple_choice** | Next step mappings      | NOT USED                                                                                                                                                                                                                                                      | **⚠️ PROBLEM**: ActiveModel.load() tries to load this                                                 |

#### 🔀 Critical Decision: How to Distinguish Job Mode vs API Mode Records?

**Problem Statement**: Both modes consume same `statusQueue` SQS queue and query same `status_check` table. Without discrimination:

- API mode will process job mode records → template validation fails
- Job mode will process API mode records → wrong window logic
- Race conditions on SQS message consumption

**Options for Discriminator**:

##### Option 1: Use `step_id` Column (Existing Schema)

**Approach**:

- API mode: `step_id > 0` (always has script step)
- Job mode: `step_id = 0` or `step_id = NULL`

**Pros**:

- ✅ No schema changes required
- ✅ Quick to implement (pilot ready)
- ✅ Clear semantic difference

**Cons**:

- ❌ `step_id` has NOT NULL constraint + FK constraint in schema
- ❌ Need dummy `script_step` record with `id=0`
- ❌ Confusing semantics (0 usually means unset, not "job mode")
- ❌ `script_version_id` and `scheduled_id` also have NOT NULL constraints

**Implementation**:

```sql
-- Create dummy records
INSERT INTO script_version (id, ...) VALUES (0, ...);
INSERT INTO script_step (id, script_version_id, ...) VALUES (0, 0, ...);
INSERT INTO scheduled (id, ...) VALUES (0, ...);

-- Job mode creates status_check with:
step_id = 0
script_version_id = 0  
scheduled_id = 0
```

**Filter logic**:

```typescript
// API mode
if (statusCheck.model.stepId === 0) {
  await message.ack(); return; // Skip job records
}

// Job mode  
if (statusCheck.model.stepId > 0) {
  await message.ack(); return; // Skip API records
}
```

**DB Migration Needed**: None (uses existing columns)

---

##### Option 2: Add `monitoring_type` Column (New Schema)

**Approach**:

- API mode: `monitoring_type = 'SCRIPT_BASED'`
- Job mode: `monitoring_type = 'AUTONOMOUS_MONITORING'`

**Pros**:

- ✅ Explicit and self-documenting
- ✅ No dummy records needed
- ✅ Easy to extend (future monitoring types)
- ✅ Clean filtering logic

**Cons**:

- ❌ Requires schema migration
- ❌ Need to update existing records (backfill)
- ❌ Longer deployment timeline

**Implementation**:

```sql
-- Migration
ALTER TABLE status_check
ADD COLUMN monitoring_type VARCHAR(64) NOT NULL DEFAULT 'SCRIPT_BASED'
AFTER status_check_template_id;

-- Index for filtering
CREATE INDEX idx_status_check_monitoring_type ON status_check(monitoring_type, result);
```

**Filter logic**:

```typescript
// API mode
if (statusCheck.model.monitoringType !== 'SCRIPT_BASED') {
  await message.ack(); return;
}

// Job mode
if (statusCheck.model.monitoringType !== 'AUTONOMOUS_MONITORING') {
  await message.ack(); return;
}
```

**DB Migration Needed**:

1. Add column with default value
2. Existing records auto-set to 'SCRIPT_BASED'
3. Update domain models and DTOs

---

##### Option 3: Make Script Columns Nullable (Schema Change)

**Approach**:

- API mode: `step_id IS NOT NULL`
- Job mode: `step_id IS NULL`

**Pros**:

- ✅ Clean semantics (NULL = no script)
- ✅ No dummy records
- ✅ Simple filter: `WHERE step_id IS NOT NULL`

**Cons**:

- ❌ Requires schema migration
- ❌ Must drop/recreate FK constraints
- ❌ Potentially breaks existing queries assuming NOT NULL

**Implementation**:

```sql
-- Migration (complex due to FKs)
ALTER TABLE status_check
DROP FOREIGN KEY fk_status_check_step_id,
DROP FOREIGN KEY fk_status_check_script_version_id,
DROP FOREIGN KEY fk_status_check_scheduled_id;

ALTER TABLE status_check
MODIFY COLUMN step_id BIGINT UNSIGNED NULL,
MODIFY COLUMN script_version_id BIGINT UNSIGNED NULL,
MODIFY COLUMN scheduled_id BIGINT UNSIGNED NULL;

ALTER TABLE status_check
ADD CONSTRAINT fk_status_check_step_id 
  FOREIGN KEY (step_id) REFERENCES script_step(id),
ADD CONSTRAINT fk_status_check_script_version_id
  FOREIGN KEY (script_version_id) REFERENCES script_version(id),
ADD CONSTRAINT fk_status_check_scheduled_id
  FOREIGN KEY (scheduled_id) REFERENCES scheduled(id);
```

**Filter logic**:

```typescript
// API mode
if (statusCheck.model.stepId === null) {
  await message.ack(); return;
}

// Job mode
if (statusCheck.model.stepId !== null) {
  await message.ack(); return;
}
```

**DB Migration Needed**: Complex FK constraint changes

---

##### Option 4: Separate SQS Queues (Infrastructure Change)

**Approach**:

- API mode: Consumes `statusQueue`
- Job mode: Consumes `jobStatusQueue` (separate queue)
- Megazord: Routes subscription events based on metadata

**Pros**:

- ✅ Complete isolation (no code conflicts)
- ✅ Independent scaling
- ✅ No DB schema changes
- ✅ Clear separation of concerns

**Cons**:

- ❌ Requires Megazord changes (subscription routing)
- ❌ More infrastructure to manage
- ❌ Harder to implement for pilot

**Implementation**:

```json
// Config
{
  "statusQueue": { "address": "https://sqs.../status-queue" },
  "jobStatusQueue": { "address": "https://sqs.../job-status-queue" }
}
```

**Megazord subscription**:

```typescript
// Job mode subscribes with metadata
await megazordEventsService.subscribe(ctx, robotId, events, until, {
  queueType: 'JOB_QUEUE'  // Routes to jobStatusQueue
})
```

**Filter logic**: Not needed (separate queues)

**Changes Needed**:

1. New SQS queue in AWS
2. Megazord routing logic
3. Job mode config

---

##### Option 5: Create Separate `status_check_job` Table (New Table)

**Approach**:

- API mode: Uses `status_check` table
- Job mode: Uses `status_check_job` table (same structure, no script columns)

**Pros**:

- ✅ Complete isolation at DB level
- ✅ No FK constraints issues
- ✅ Can optimize schema for job mode
- ✅ Zero impact on existing API mode

**Cons**:

- ❌ Code duplication (separate repositories)
- ❌ Can't share ActiveModel
- ❌ More DB tables to maintain
- ❌ Higher implementation effort

**Implementation**:

```sql
CREATE TABLE status_check_job (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  robot_id INT UNSIGNED NOT NULL,
  status_check_template_id INT UNSIGNED NOT NULL,
  phase VARCHAR(64) NOT NULL,
  subscription_id BIGINT UNSIGNED,
  result VARCHAR(64),
  monitoring_type VARCHAR(64) NOT NULL,  -- 'TOILET_ACTIVITY', etc
  planned_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_robot_subscription (robot_id, subscription_id)
) ENGINE = InnoDB;

CREATE TABLE status_check_job_poller (
  status_check_job_id BIGINT UNSIGNED NOT NULL,
  since TIMESTAMP NOT NULL,
  until TIMESTAMP NOT NULL,
  lock_id VARCHAR(256),
  PRIMARY KEY (status_check_job_id),
  INDEX idx_until (until)
) ENGINE = InnoDB;

CREATE TABLE status_check_job_record (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  status_check_job_id BIGINT UNSIGNED NOT NULL,
  incoming_event_id BIGINT UNSIGNED,
  outgoing_event_id BIGINT UNSIGNED,
  event_name VARCHAR(256) NOT NULL,
  `order` INT UNSIGNED NOT NULL,
  phase VARCHAR(64) NOT NULL,
  result VARCHAR(64) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE = InnoDB;
```

**Filter logic**: Not needed (separate tables)

**Changes Needed**:

1. New tables + repositories
2. Can't reuse ActiveModel (or create JobActiveModel)
3. Separate queries and logic

---

#### 📋 Comparison Matrix

| Criteria                 | Option 1: step_id=0 | Option 2: monitoring_type | Option 3: Nullable | Option 4: Separate Queue | Option 5: Separate Table |
|--------------------------|---------------------|---------------------------|--------------------|--------------------------|--------------------------|
| **Implementation Speed** | ⚡⚡⚡ Fastest         | ⚡⚡ Medium                 | ⚡⚡ Medium          | ⚡ Slow                   | ⚡ Slow                   |
| **Schema Changes**       | ✅ None              | ⚠️ Add column             | ⚠️ Modify columns  | ✅ None                   | ❌ New tables             |
| **Code Clarity**         | ⚠️ Confusing        | ✅ Explicit                | ✅ Clear            | ✅ Very clear             | ✅ Very clear             |
| **Maintenance**          | ⚠️ Dummy records    | ✅ Clean                   | ✅ Clean            | ⚠️ More infra            | ⚠️ More code             |
| **Risk to API Mode**     | ⚠️ Low-Medium       | ✅ Low                     | ⚠️ Medium          | ✅ None                   | ✅ None                   |
| **Scalability**          | ❌ Doesn't scale     | ✅ Good                    | ✅ Good             | ✅ Excellent              | ✅ Excellent              |
| **Pilot Friendly**       | ✅ Yes               | ✅ Yes                     | ⚠️ Medium          | ❌ No                     | ❌ No                     |

#### 🎯 Recommendation

**For Pilot (Quick Validation)**: **Option 1** (step_id=0)

- Fastest to implement
- No schema changes
- Can validate approach quickly
- Accept technical debt for short term

**For Production (Long Term)**: **Option 2** (monitoring_type column)

- Clean and explicit
- Easy to extend for future monitoring types
- Single migration, then stable
- Good balance of effort vs maintainability

**If Expecting Heavy Scale**: **Option 4** (Separate queues) or **Option 5** (Separate table)

- Complete isolation
- Independent scaling
- Worth the extra effort if this becomes a major feature

#### ⚠️ Additional Issues to Resolve

**Issue 1: ActiveModel.loadNextStepResources() Error**

When `step_id=0` or `NULL`, `loadNextStepResources()` will fail.

**Solutions**:

- **A**: Create dummy `script_next_multiple_choice` records for step_id=0
- **B**: Modify `loadNextStepResources()` to skip if step_id=0/NULL
- **C**: Don't use ActiveModel in job mode, use direct repository queries

**Recommended**: Option B (modify ActiveModel)

```typescript
private async loadNextStepResources(): Promise<void> {
  // Skip for job mode records
  if (!this.model.stepId || this.model.stepId === 0) {
    this.model.nextStepsResource = new domains.StatusCheckNextStepsResource()
    return
  }
  
  // Existing logic for API mode...
}
```

**Issue 2: Repository Method for Window Updates**

`StatusCheckActiveModel` không có method để update poller window.

**Solution**: Add repository method

```typescript
// In StatusChecksRepository
async updatePollerWindow(
  statusCheckId: number,
  since: Date,
  until: Date
): Promise<void> {
  await this.database.query(
    `UPDATE status_check_poller 
     SET since = ?, until = ?, updated_at = NOW() 
     WHERE status_check_id = ?`,
    [since, until, statusCheckId]
  )
}
```

### Phase 2: Implementation (File/Code Structure)

> Describe the proposed file/directory structure, including the purpose of each key component. Remember use status markers like ✅ (Implemented), 🚧 (To-Do), 🔄 (In Progress).

```
azi-3-status-check/
├── src/
│   ├── cmd/
│   │   ├── app/                               # ✅ EXISTING - HTTP API mode (untouched)
│   │   │   ├── main.ts                        # ✅ EXISTING - Bootstraps Express app
│   │   │   └── index.ts                       # ✅ EXISTING - Entry point
│   │   └── job/                               # 🚧 NEW - Background job mode (independent)
│   │       ├── main.ts                        # 🚧 NEW - Bootstraps job runner without HTTP
│   │       └── index.ts                       # 🚧 NEW - Job entry point
│   │
│   ├── services/
│   │   ├── MegazordEventsService.ts           # ✅ EXISTING - Reused as-is
│   │   ├── StatusCheckService.ts              # ✅ EXISTING - API mode only (untouched)
│   │   ├── StatusCheckTeardownSchedulerService.ts  # ✅ EXISTING - Reused by job mode
│   │   ├── StatusChecksSQSProducerService.ts  # ✅ EXISTING - Reused as-is
│   │   ├── StatusTemplatesLoaderService.ts    # ✅ EXISTING - Not used by job mode
│   │   └── ToiletMonitoringService.ts         # 🚧 NEW - Job-specific rolling window logic
│   │
│   ├── jobs/
│   │   ├── ToiletMonitoringScheduler.ts       # 🚧 NEW - Cron job for daily monitoring setup
│   │   └── ToiletMonitoringWorker.ts          # 🚧 NEW - SQS consumer for TOILET_ACTIVITY events
│   │
│   ├── models/
│   │   ├── ToiletMonitoringConfig.ts          # 🚧 NEW - Job-specific config (robotId, time A/B, timezone)
│   │   └── DTOs/                              # ✅ EXISTING - Reused as-is
│   │
│   ├── repositories/
│   │   ├── StatusChecksRepository.ts          # ✅ EXISTING - Reused as-is
│   │   └── StatusTemplateRepository.ts        # ✅ EXISTING - Reused for seeding only
│   │
│   └── constants/
│       └── StatusCheckTypes.ts                # 🚧 NEW - Add TOILET_MONITORING constant
│
├── config/
│   ├── default.json                           # 🔄 EXTEND - Add toilet monitoring config section
│   └── production.json                        # 🔄 EXTEND - Production-specific settings
│
├── scripts/
│   └── seed-toilet-template.ts                # 🚧 NEW - One-time template seeding
│
└── package.json                               # ✅ EXISTING - Already has start:job script
```

**Supporting repositories (external changes):**

```
megazord-events/
├── schemas/
│   ├── gen.ts                                 # 🔄 EXTEND - Add NO_TOILET_ACTIVITY_ALARM
│   └── events/
│       └── no_toilet_activity_alarm.json      # 🚧 NEW - Event schema

tiny-internal-services/
└── lib/model/events/
    └── TinybotsEvent.ts                       # 🔄 EXTEND - Export NO_TOILET_ACTIVITY_ALARM

typ-e/
└── src/main/resources/db/migration/
    └── V{next}_seed_toilet_monitoring_template.sql  # 🚧 NEW - Template/description seed
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Add NO_TOILET_ACTIVITY_ALARM to Megazord Event Schema

**Files to modify:**

- `megazord-events/schemas/gen.ts` - Add new event constant
- `megazord-events/schemas/events/no_toilet_activity_alarm.json` - Define event schema (simple, no properties/payload)
- `tiny-internal-services/lib/model/events/TinybotsEvent.ts` - Export constant

**Implementation:**

```typescript
// megazord-events/schemas/gen.ts
export const Events = {
  // ... existing events
  NO_TOILET_ACTIVITY_ALARM: 'NO_TOILET_ACTIVITY_ALARM'
} as const

// no_toilet_activity_alarm.json
{
  "eventName": "NO_TOILET_ACTIVITY_ALARM",
  "level": 30,
  "hasTrigger": true,
  "isActive": true,
  "description": "Alarm when no toilet activity detected for 2 hours"
}

// tiny-internal-services TinybotsEvent.ts
export enum TinybotsEvent {
  // ... existing
  NO_TOILET_ACTIVITY_ALARM = 'NO_TOILET_ACTIVITY_ALARM'
}
```

**Rationale:** Event schema chỉ định nghĩa metadata của event (level, hasTrigger, isActive), không chứa payload/properties. Payload sẽ được truyền khi post event qua MegazordEventsService.

---

#### Step 2: Seed Toilet Monitoring Template in Database

**Files to create:**

- `typ-e/src/main/resources/db/migration/V{next}_seed_toilet_monitoring_template.sql`
- `azi-3-status-check/scripts/seed-toilet-template.ts`

**Implementation:**

```sql
-- Insert status_check_description
INSERT INTO status_check_description (name, description, created_at, updated_at)
VALUES ('TOILET_ACTIVITY_MONITORING', 'Monitor toilet activity in 2-hour rolling windows', NOW(), NOW())
ON DUPLICATE KEY UPDATE updated_at = NOW();

-- Insert status_check_template
INSERT INTO status_check_template (
  status_check_description_id,
  name,
  version,
  past_event_dependencies,
  future_event_dependencies,
  poll_until_end,
  is_active,
  created_at,
  updated_at
)
SELECT 
  scd.id,
  'toilet_activity_monitoring',
  '1.0.0',
  '[]',
  '[{"event":"TOILET_ACTIVITY","maxGapMinutes":120}]',
  TRUE,
  TRUE,
  NOW(),
  NOW()
FROM status_check_description scd
WHERE scd.name = 'TOILET_ACTIVITY_MONITORING'
ON DUPLICATE KEY UPDATE updated_at = NOW();

-- Insert template details (phases)
INSERT INTO status_check_template_detail (
  status_check_template_id,
  phase,
  result,
  pass_expression,
  fail_expression,
  unknown_expression,
  created_at,
  updated_at
)
SELECT 
  sct.id,
  'MONITORING',
  'PENDING',
  NULL,
  NULL,
  NULL,
  NOW(),
  NOW()
FROM status_check_template sct
WHERE sct.name = 'toilet_activity_monitoring'
ON DUPLICATE KEY UPDATE updated_at = NOW();
```

**Rationale:** Reuse existing template infrastructure for metadata, even though job mode creates checks programmatically.

---

#### Step 3: Design Configuration Strategy for Multiple Robots

**Decision Required**: Choose between config file array vs database table.

**Files to create:**

- `src/models/ToiletMonitoringConfig.ts`

**Implementation:**

```typescript
export interface IToiletMonitoringConfig {
  enabled: boolean
  robotId: number
  timezone: string // e.g., 'America/Los_Angeles'
  dailyStartTime: string // HH:mm format, e.g., '08:00'
  dailyEndTime: string // HH:mm format, e.g., '20:00'
  windowDurationMinutes: number // default 120
  alarmLevel: string // e.g., 'WARNING' | 'ERROR'
  templateName: string // 'toilet_activity_monitoring'
}

export class ToiletMonitoringConfig implements IToiletMonitoringConfig {
  enabled: boolean
  robotId: number
  timezone: string
  dailyStartTime: string
  dailyEndTime: string
  windowDurationMinutes: number
  alarmLevel: string
  templateName: string

  constructor(config: any) {
    this.enabled = config.enabled ?? false
    this.robotId = config.robotId
    this.timezone = config.timezone
    this.dailyStartTime = config.dailyStartTime
    this.dailyEndTime = config.dailyEndTime
    this.windowDurationMinutes = config.windowDurationMinutes ?? 120
    this.alarmLevel = config.alarmLevel ?? 'WARNING'
    this.templateName = config.templateName ?? 'toilet_activity_monitoring'
  }

  validate(): void {
    if (!this.robotId) throw new Error('robotId is required')
    if (!this.timezone) throw new Error('timezone is required')
    if (!this.dailyStartTime || !this.dailyEndTime) {
      throw new Error('dailyStartTime and dailyEndTime are required')
    }
  }
}
```

**Add to config/default.json:**

```json
{
  "toiletMonitoring": {
    "enabled": false,
    "robots": [
      {
        "robotId": null,
        "timezone": "America/Los_Angeles",
        "dailyStartTime": "08:00",
        "dailyEndTime": "20:00",
        "windowDurationMinutes": 120,
        "alarmLevel": "WARNING"
      }
    ],
    "templateName": "toilet_activity_monitoring"
  }
}
```

**Note**: Configuration supports multiple robots, mỗi robot có timezone và time window riêng.

**Alternative**: Store trong database table `robot_monitoring_config` để dynamic config (xem Phase 1.5 để biết chi tiết).

---

#### Step 4: Implement StatusChecksRepository Extensions

**Files to modify:**

- `src/repositories/StatusCheckRepository.ts`
- `src/repositories/internal/StatusCheckActiveModel.ts`

**New methods needed:**

```typescript
import dayjs from 'dayjs'
import timezone from 'dayjs/plugin/timezone'
import utc from 'dayjs/plugin/utc'
import { IRequestContext, Logger } from 'tiny-backend-tools'
import { TinybotsEvent } from 'tiny-internal-services'
import { domains } from '../models'
import { IToiletMonitoringConfig } from '../models/ToiletMonitoringConfig'
import { repositories } from '../repositories'
import { StatusCheckActiveModel } from '../repositories/internal'
import { IMegazordEventsService } from './MegazordEventsService'

dayjs.extend(utc)
dayjs.extend(timezone)

export interface IToiletMonitoringService {
  /**
   * Create a new monitoring check at Time A
   * Subscribes to TOILET_ACTIVITY events until Time B
   */
  createDailyMonitoring(ctx: IRequestContext): Promise<StatusCheckActiveModel>

  /**
   * Handle incoming TOILET_ACTIVITY event
   * Reset window if within current monitoring period
   */
  handleToiletActivity(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel,
    event: domains.IncomingEventDomain
  ): Promise<void>

  /**
   * Emit NO_TOILET_ACTIVITY_ALARM when 2-hour gap detected
   * Reset window and continue monitoring until Time B
   */
  emitAlarmAndContinue(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel
  ): Promise<void>

  /**
   * Finalize monitoring at Time B or when service stops
   */
  finalizeMonitoring(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel
  ): Promise<void>
}

export class ToiletMonitoringService implements IToiletMonitoringService {
  constructor(
    private readonly config: IToiletMonitoringConfig,
    private readonly statusChecksRepository: repositories.IStatusChecksRepository,
    private readonly statusTemplatesRepository: repositories.IStatusTemplateRepository,
    private readonly megazordEventsService: IMegazordEventsService
  ) {}

  async createDailyMonitoring(ctx: IRequestContext): Promise<StatusCheckActiveModel> {
    const logger = Logger.loggerFromCtx(ctx)
    
    // Calculate today's monitoring window in robot's timezone
    const now = dayjs().tz(this.config.timezone)
    const timeA = now.startOf('day')
      .add(parseInt(this.config.dailyStartTime.split(':')[0]), 'hour')
      .add(parseInt(this.config.dailyStartTime.split(':')[1]), 'minute')
    
    const timeB = now.startOf('day')
      .add(parseInt(this.config.dailyEndTime.split(':')[0]), 'hour')
      .add(parseInt(this.config.dailyEndTime.split(':')[1]), 'minute')

    // Load template for metadata
    const template = await this.statusTemplatesRepository.getActiveByName(
      this.config.templateName
    )
    if (!template) {
      throw new Error(`Template ${this.config.templateName} not found`)
    }

    // Create status check programmatically (no script/step reference)
    const statusCheck = await this.statusChecksRepository.create(
      this.config.robotId,
      null, // scheduledId - not applicable
      null, // stepId - not applicable  
      null, // scriptVersionId - not applicable
      template.id,
      timeA.toDate(),
      domains.StatusCheckPhase.FUTURE, // Start in FUTURE phase (monitoring)
      timeA.toDate(), // pollingSince = Time A
      timeB.toDate()  // pollingUntil = Time B
    )

    logger.info('Created toilet monitoring check', {
      statusCheckId: statusCheck.id,
      robotId: this.config.robotId,
      timeA: timeA.toISOString(),
      timeB: timeB.toISOString()
    })

    // Subscribe to TOILET_ACTIVITY events
    const subscription = await this.megazordEventsService.subscribe(
      ctx,
      this.config.robotId,
      [TinybotsEvent.TOILET_ACTIVITY],
      timeB.toISOString()
    )

    // Update status check with subscription ID
    const activeModel = await StatusCheckActiveModel.load(
      statusCheck.id,
      this.statusChecksRepository
    )
    activeModel.setSubscriptionId(subscription.id)
    
    // Set initial poller window (Time A to Time A + 2 hours OR Time B)
    const firstWindowEnd = dayjs.min(
      timeA.add(this.config.windowDurationMinutes, 'minute'),
      timeB
    )
    activeModel.setPollerWindow(timeA.toDate(), firstWindowEnd.toDate())
    
    await activeModel.flush()

    logger.info('Subscribed to TOILET_ACTIVITY events', {
      statusCheckId: statusCheck.id,
      subscriptionId: subscription.id,
      firstWindowEnd: firstWindowEnd.toISOString()
    })

    return activeModel
  }

  async handleToiletActivity(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel,
    event: domains.IncomingEventDomain
  ): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    const eventTime = dayjs(event.createdAt)
    const pollingUntil = dayjs(statusCheck.pollingUntil)

    // Ignore events outside monitoring period
    if (eventTime.isAfter(pollingUntil)) {
      logger.debug('Event after monitoring period, ignoring', {
        eventTime: eventTime.toISOString(),
        pollingUntil: pollingUntil.toISOString()
      })
      return
    }

    // Record the activity event
    statusCheck.addRecord(
      TinybotsEvent.TOILET_ACTIVITY,
      event.id,
      domains.StatusCheckPhase.FUTURE,
      'EVENT_OBSERVED',
      eventTime.toDate()
    )
    await statusCheck.flush()

    // Reset window: new start = event time, new end = event time + 2 hours OR Time B
    const newWindowEnd = dayjs.min(
      eventTime.add(this.config.windowDurationMinutes, 'minute'),
      pollingUntil
    )

    // Update poller window directly via repository (không có setPollerWindow method)
    await this.statusChecksRepository.updatePollerWindow(
      statusCheck.id,
      eventTime.toDate(),
      newWindowEnd.toDate()
    )

    logger.info('Toilet activity detected, window reset', {
      statusCheckId: statusCheck.id,
      eventTime: eventTime.toISOString(),
      newWindowEnd: newWindowEnd.toISOString()
    })
  }

  async emitAlarmAndContinue(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel
  ): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    const now = dayjs()
    const pollingUntil = dayjs(statusCheck.pollingUntil)
    const pollingSince = dayjs(statusCheck.pollingSince)

    // Emit NO_TOILET_ACTIVITY_ALARM event
    const alarmEvent = await this.megazordEventsService.postEvent(
      ctx,
      this.config.robotId,
      {
        providerName: 'azi-3-status-check',
        eventName: TinybotsEvent.NO_TOILET_ACTIVITY_ALARM,
        level: this.config.alarmLevel,
        payload: {
          windowStart: pollingSince.toISOString(),
          windowEnd: now.toISOString(),
          lastActivity: null, // Could track from previous event
          statusCheckId: statusCheck.id
        },
        referenceId: `toilet-alarm-${statusCheck.id}-${now.unix()}`
      }
    )

    // Record alarm emission
    statusCheck.addRecord(
      TinybotsEvent.NO_TOILET_ACTIVITY_ALARM,
      alarmEvent.id,
      domains.StatusCheckPhase.FUTURE,
      'ALARM_SENT',
      `No activity for ${this.config.windowDurationMinutes} minutes`
    )

    logger.warn('Toilet activity alarm emitted', {
      statusCheckId: statusCheck.id,
      alarmEventId: alarmEvent.id,
      gapMinutes: this.config.windowDurationMinutes
    })

    // Check if we should continue monitoring
    if (now.isBefore(pollingUntil)) {
      // Reset window for next monitoring period
      const newWindowEnd = dayjs.min(
        now.add(this.config.windowDurationMinutes, 'minute'),
        pollingUntil
      )
      
      statusCheck.setPollerWindow(now.toDate(), newWindowEnd.toDate())
      await statusCheck.flush()

      logger.info('Continuing monitoring after alarm', {
        statusCheckId: statusCheck.id,
        newWindowEnd: newWindowEnd.toISOString()
      })
    } else {
      // Time B reached, finalize
      await this.finalizeMonitoring(ctx, statusCheck)
    }
  }

  async finalizeMonitoring(
    ctx: IRequestContext,
    statusCheck: StatusCheckActiveModel
  ): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)

    // Unsubscribe from events
    if (statusCheck.subscriptionId) {
      await this.megazordEventsService.unsubscribe(
        ctx,
        this.config.robotId,
        statusCheck.subscriptionId
      )
    }

    // Mark as completed
    statusCheck.setResult('COMPLETED')
    statusCheck.setNewPhase(domains.StatusCheckPhase.COMPLETED)
    statusCheck.unlockPoller()
    
    await statusCheck.flush()

    logger.info('Toilet monitoring finalized', {
      statusCheckId: statusCheck.id,
      totalRecords: statusCheck.history.length
    })
  }
}
```

**Rationale:** Encapsulates all rolling window logic separately from existing StatusChecksService. Reuses repository/Megazord patterns without modification.

---

#### Step 6: Implement Scheduler and Worker Jobs with Conflict Handling

**Files to create:**

- `src/jobs/ToiletMonitoringScheduler.ts`
- `src/jobs/ToiletMonitoringWorker.ts`

**Critical**: Both API mode and Job mode consume same `statusQueue`. Must add filters to avoid conflicts.

**ToiletMonitoringScheduler.ts:**

```typescript
import { CronJob } from 'cron'
import dayjs from 'dayjs'
import { IRequestContext, Logger, Modules } from 'tiny-backend-tools'
import { IToiletMonitoringConfig } from '../models/ToiletMonitoringConfig'
import { IToiletMonitoringService } from '../services/ToiletMonitoringService'

export interface IToiletMonitoringScheduler extends Modules.IAsyncModule {
  scheduleDaily(ctx: IRequestContext): void
}

export class ToiletMonitoringScheduler implements IToiletMonitoringScheduler {
  private cronJob?: CronJob

  constructor(
    private readonly config: IToiletMonitoringConfig,
    private readonly toiletMonitoringService: IToiletMonitoringService
  ) {}

  async init(ctx: IRequestContext): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    
    if (!this.config.enabled) {
      logger.info('Toilet monitoring disabled, skipping scheduler')
      return
    }

    logger.info('Initializing toilet monitoring scheduler', {
      robotId: this.config.robotId,
      timezone: this.config.timezone,
      startTime: this.config.dailyStartTime
    })

    this.scheduleDaily(ctx)
  }

  async stop(ctx: IRequestContext): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    
    if (this.cronJob) {
      this.cronJob.stop()
      logger.info('Toilet monitoring scheduler stopped')
    }
  }

  scheduleDaily(ctx: IRequestContext): void {
    const logger = Logger.loggerFromCtx(ctx)
    
    // Parse start time
    const [hour, minute] = this.config.dailyStartTime.split(':').map(Number)
    
    // Cron pattern: run at configured time daily
    const cronPattern = `${minute} ${hour} * * *`
    
    this.cronJob = new CronJob(
      cronPattern,
      async () => {
        try {
          logger.info('Starting daily toilet monitoring')
          await this.toiletMonitoringService.createDailyMonitoring(ctx)
        } catch (error) {
          logger.error('Failed to create daily monitoring', { error })
        }
      },
      null,
      true,
      this.config.timezone
    )

    logger.info('Toilet monitoring scheduled', {
      cronPattern,
      timezone: this.config.timezone,
      nextRun: this.cronJob.nextDate().toISOString()
    })
  }
}
```

**ToiletMonitoringWorker.ts (with conflict filter):**

```typescript
import dayjs from 'dayjs'
import { IRequestContext, Logger, Modules, SQS } from 'tiny-backend-tools'
import { TinybotsEvent } from 'tiny-internal-services'
import { domains } from '../models'
import { IStatusQueue } from '../models/SQSConfig'
import { IToiletMonitoringConfig } from '../models/ToiletMonitoringConfig'
import { repositories } from '../repositories'
import { IToiletMonitoringService } from '../services/ToiletMonitoringService'

export interface IToiletMonitoringWorker extends Modules.IAsyncModule {
  poll(ctx: IRequestContext): Promise<void>
}

export class ToiletMonitoringWorker implements IToiletMonitoringWorker {
  constructor(
    private readonly config: IToiletMonitoringConfig,
    private readonly statusChecksRepository: repositories.IStatusChecksRepository,
    private readonly toiletMonitoringService: IToiletMonitoringService,
    private readonly statusChecksSQSClient: SQS.ISQSConsumer,
    private readonly statusQueueConfig: IStatusQueue
  ) {}

  async init(ctx: IRequestContext): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    
    if (!this.config.enabled) {
      logger.info('Toilet monitoring disabled, skipping worker')
      return
    }

    logger.info('Starting toilet monitoring SQS worker')
    this.poll(ctx)
  }

  async stop(ctx: IRequestContext): Promise<void> {
    Logger.loggerFromCtx(ctx).info('Toilet monitoring worker stopped')
  }

  async poll(ctx: IRequestContext): Promise<void> {
    for await (
      const message of this.statusChecksSQSClient.poll(
        ctx,
        this.statusQueueConfig.address
      )
    ) {
      setImmediate(() => {
        this.handleMessage(message.ctx, message, message.message)
      })
    }
  }

  private async handleMessage(
    ctx: IRequestContext,
    message: SQS.IContextMessage,
    payload: Record<string, any>
  ): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    
    try {
      const outgoingEvent = domains.OutgoingEventSQSMessage.FromPlain(payload)
      
      // Filter 1: Only process TOILET_ACTIVITY events
      if (outgoingEvent.payload.sourceEvent.eventName !== TinybotsEvent.TOILET_ACTIVITY) {
        await message.ack()
        return
      }

      // Find status check by subscription ID
      const statusCheck = await this.statusChecksRepository
        .getByRobotIdAndSubscriptionId(
          outgoingEvent.payload.sourceEvent.robotId,
          outgoingEvent.payload.subscriptionId
        )

      if (!statusCheck) {
        logger.debug('No active monitoring for this subscription')
        await message.ack()
        return
      }

      // Filter 2: CRITICAL - Skip API mode records (has step_id > 0)
      // This prevents job mode from processing script-based status checks
      if (statusCheck.model.stepId && statusCheck.model.stepId > 0) {
        logger.debug('Skipping API mode status check in job worker', {
          statusCheckId: statusCheck.id,
          stepId: statusCheck.model.stepId
        })
        await message.ack()
        return
      }

      // Filter 3: Only process robots in our job config
      const robotInConfig = this.config.robots.some(
        r => r.robotId === statusCheck.model.robotId
      )
      if (!robotInConfig) {
        logger.debug('Robot not in job mode config, skipping', {
          robotId: statusCheck.model.robotId
        })
        await message.ack()
        return
      }

      // Process the toilet activity event
      logger.info('Processing toilet activity event', {
        statusCheckId: statusCheck.id,
        robotId: statusCheck.model.robotId,
        eventTime: outgoingEvent.payload.sourceEvent.createdAt
      })

      await this.toiletMonitoringService.handleToiletActivity(
        ctx,
        statusCheck,
        outgoingEvent.payload.sourceEvent
      )

      await message.ack()
      
    } catch (error) {
      logger.error('Error handling toilet activity message', { error })
      await message.fail()
    }
  }
}
```

**Corresponding filter in API mode (add to existing StatusChecksService.handleStatusCheckSQSMessage)**:

```typescript
// In StatusChecksService.handleStatusCheckSQSMessage()
// After loading statusCheck, add this filter BEFORE processing:

const statusCheck = await this.statusChecksRepository
  .getByRobotIdAndSubscriptionId(
    outgoingEventMessage.payload.sourceEvent.robotId,
    outgoingEventMessage.payload.subscriptionId
  )

if (!statusCheck) {
  await message.ack()
  return
}

// CRITICAL NEW FILTER: Skip job mode checks (step_id = 0 or NULL)
// This prevents API mode from processing autonomous monitoring checks
if (!statusCheck.model.stepId || statusCheck.model.stepId === 0) {
  logger.debug('Skipping job mode status check in API mode', {
    statusCheckId: statusCheck.id,
    subscriptionId: outgoingEventMessage.payload.subscriptionId
  })
  await message.ack()
  return
}

// Continue with existing validation logic...
if (dayjs(outgoingEventMessage.payload.createdAt).isAfter(statusCheck.pollingUntil)) {
  // ... existing code
}
```

**Rationale**:

- **Filter by step_id**: Discriminator giữa job mode (step_id=0) và API mode (step_id>0)
- **Three-layer filtering**: event name → status check exists → record type
- **Explicit logging**: Debug logs giúp trace routing decisions
- **Must update API mode**: Thêm filter để skip job records, tránh conflict

**Note**: Phụ thuộc vào decision ở Phase 1.5 về discriminator strategy, có thể thay step_id bằng monitoring_type column hoặc approaches khác.

---

#### Step 7: Implement Window Expiration Monitor

**Files to create:**

- `src/jobs/ToiletWindowMonitor.ts`

**Implementation:**

```typescript
import { CronJob } from 'cron'
import dayjs from 'dayjs'
import { IRequestContext, Logger, Modules } from 'tiny-backend-tools'
import { IToiletMonitoringConfig } from '../models/ToiletMonitoringConfig'
import { repositories } from '../repositories'
import { IToiletMonitoringService } from '../services/ToiletMonitoringService'

export interface IToiletWindowMonitor extends Modules.IAsyncModule {
  startMonitoring(ctx: IRequestContext): void
}

export class ToiletWindowMonitor implements IToiletWindowMonitor {
  private cronJob?: CronJob

  constructor(
    private readonly config: IToiletMonitoringConfig,
    private readonly statusChecksRepository: repositories.IStatusChecksRepository,
    private readonly toiletMonitoringService: IToiletMonitoringService
  ) {}

  async init(ctx: IRequestContext): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    
    if (!this.config.enabled) {
      logger.info('Toilet monitoring disabled, skipping window monitor')
      return
    }

    logger.info('Starting toilet window expiration monitor')
    this.startMonitoring(ctx)
  }

  async stop(ctx: IRequestContext): Promise<void> {
    if (this.cronJob) {
      this.cronJob.stop()
      Logger.loggerFromCtx(ctx).info('Toilet window monitor stopped')
    }
  }

  startMonitoring(ctx: IRequestContext): void {
    const logger = Logger.loggerFromCtx(ctx)
    
    // Run every minute to check for expired windows
    this.cronJob = new CronJob(
      '* * * * *', // Every minute
      async () => {
        try {
          await this.checkExpiredWindows(ctx)
        } catch (error) {
          logger.error('Error checking expired windows', { error })
        }
      },
      null,
      true,
      this.config.timezone
    )

    logger.info('Window expiration monitor started')
  }

  private async checkExpiredWindows(ctx: IRequestContext): Promise<void> {
    const logger = Logger.loggerFromCtx(ctx)
    const now = dayjs()

    // Find all active toilet monitoring checks with expired windows
    // Uses existing StatusCheckTeardownSchedulerService pattern
    const expiredPollers = await this.statusChecksRepository
      .acquireByPollerUntil(
        now.subtract(1, 'minute').toDate(),
        now.add(1, 'minute').toDate(),
        'toilet-window-monitor'
      )

    if (expiredPollers.length === 0) {
      return
    }

    logger.info('Found expired toilet monitoring windows', {
      count: expiredPollers.length
    })

    for (const poller of expiredPollers) {
      try {
        // Load full status check
        const statusCheck = await this.statusChecksRepository.getById(poller.statusCheckId)
        
        // Only process toilet monitoring checks (identify by template name or flag)
        // Skip robot-initiated checks handled by StatusCheckTeardownSchedulerService
        if (!this.isToiletMonitoringCheck(statusCheck)) {
          continue
        }

        // Emit alarm and continue or finalize
        await this.toiletMonitoringService.emitAlarmAndContinue(ctx, statusCheck)
        
      } catch (error) {
        logger.error('Failed to process expired window', {
          statusCheckId: poller.statusCheckId,
          error
        })
      }
    }
  }

  private isToiletMonitoringCheck(statusCheck: any): boolean {
    // Check if this is a toilet monitoring check
    // Could use template name, custom flag, or absence of stepId/scriptVersionId
    return !statusCheck.stepId && !statusCheck.scriptVersionId
  }
}
```

**Rationale:** Polls database every minute for expired windows, similar to StatusCheckTeardownSchedulerService but for different purpose. Only acts on toilet monitoring checks, not robot-initiated ones.

---

#### Step 7: Create Job Mode Bootstrap

**Files to create:**

- `src/cmd/job/main.ts`
- `src/cmd/job/index.ts`

**main.ts:**

```typescript
import 'reflect-metadata'
import { asClass, asFunction, asValue } from 'awilix'
import axios from 'axios'
import { randomUUID } from 'node:crypto'
import {
  Cron,
  IRequestContext,
  loadConfigValue,
  LogConfig,
  Logger,
  Modules,
  MySQLConfig,
  SQS,
  TinyDatabaseApp
} from 'tiny-backend-tools'
import winston from 'winston'
import { ContainerNames } from '../../constants'
import {
  AddressConfig,
  AppConfig,
  ISchedulerConfig,
  SchedulerConfig
} from '../../models'
import { IStatusQueue, QueueConfig, SQSConfig } from '../../models/SQSConfig'
import { ToiletMonitoringConfig } from '../../models/ToiletMonitoringConfig'
import { repositories } from '../../repositories'
import { services } from '../../services'
import { ToiletMonitoringService } from '../../services/ToiletMonitoringService'
import {
  ToiletMonitoringScheduler,
  ToiletMonitoringWorker,
  ToiletWindowMonitor
} from '../../jobs'

export class JobApp extends TinyDatabaseApp {
  private logger!: Cron.ExtendableLogger
  private ctx: IRequestContext
  private asyncContainer: Modules.AwilixWrapper<any>
  private isStopping: boolean = false

  constructor(
    protected mysqlConfig: MySQLConfig,
    protected readonly logConfig: LogConfig,
    protected readonly appConfig: AppConfig,
    protected readonly sqsConfig: SQS.ISQSConfig,
    protected readonly megazordEventsConfig: AddressConfig,
    private readonly statusQueueConfig: IStatusQueue,
    private readonly statusChecksSchedulerConfig: ISchedulerConfig,
    private readonly toiletMonitoringConfig: ToiletMonitoringConfig
  ) {
    super(mysqlConfig)

    this.setDefaultLogger()
    this.ctx = Cron.newCronContext(this.logger, `${appConfig.appName}-job`)
    this.asyncContainer = new Modules.AwilixWrapper(this.container)

    this.extendContainer()
  }

  private setDefaultLogger(): void {
    const format = this.appConfig.isLocal
      ? winston.format.combine(
          winston.format.timestamp(),
          winston.format.splat(),
          winston.format.json(),
          winston.format.prettyPrint({ colorize: true })
        )
      : winston.format.combine(
          winston.format.timestamp(),
          winston.format.splat(),
          winston.format.json()
        )

    const winstonLogger = winston.createLogger({
      level: this.logConfig.level,
      format,
      transports: [new winston.transports.Console()],
      defaultMeta: {
        _appName: `${this.appConfig.appName}-job`,
        _mode: 'job'
      }
    })

    this.logger = {
      log: winstonLogger.log.bind(winstonLogger),
      error: winstonLogger.error.bind(winstonLogger),
      warn: winstonLogger.warn.bind(winstonLogger),
      info: winstonLogger.info.bind(winstonLogger),
      debug: winstonLogger.debug.bind(winstonLogger)
    }
  }

  private extendContainer(): void {
    // Configs
    this.container.register({
      [ContainerNames.ToiletMonitoringConfig]: asValue(this.toiletMonitoringConfig),
      [ContainerNames.StatusQueueConfig]: asValue(this.statusQueueConfig),
      [ContainerNames.StatusChecksSchedulerConfig]: asValue(this.statusChecksSchedulerConfig)
    })

    // Shared services (reuse existing)
    this.container.register({
      [ContainerNames.MegazordEventsService]: asClass(services.MegazordEventsService).singleton(),
      [ContainerNames.StatusChecksRepository]: asClass(repositories.StatusChecksRepository).singleton(),
      [ContainerNames.StatusTemplateRepository]: asClass(repositories.StatusTemplateRepository).singleton()
    })

    // SQS client for consuming status queue
    this.container.register({
      [ContainerNames.StatusChecksSQSClient]: asFunction(() => {
        return SQS.createSQSConsumerClient(
          this.logger,
          this.sqsConfig.address,
          this.sqsConfig.region,
          this.sqsConfig.useLocalstack,
          this.sqsConfig.localstackPort
        )
      }).singleton()
    })

    // Job-specific services
    this.container.register({
      [ContainerNames.ToiletMonitoringService]: asClass(ToiletMonitoringService).singleton()
    })

    // Job workers
    this.container.register({
      [ContainerNames.ToiletMonitoringScheduler]: asClass(ToiletMonitoringScheduler).singleton(),
      [ContainerNames.ToiletMonitoringWorker]: asClass(ToiletMonitoringWorker).singleton(),
      [ContainerNames.ToiletWindowMonitor]: asClass(ToiletWindowMonitor).singleton()
    })
  }

  async start(): Promise<void> {
    const logger = Logger.loggerFromCtx(this.ctx)

    logger.info('Starting Toilet Monitoring Job Mode')

    // Validate config
    this.toiletMonitoringConfig.validate()

    // Initialize all async modules
    await this.asyncContainer.init(this.ctx, [
      ContainerNames.ToiletMonitoringScheduler,
      ContainerNames.ToiletMonitoringWorker,
      ContainerNames.ToiletWindowMonitor
    ])

    logger.info('Toilet Monitoring Job Mode started successfully', {
      robotId: this.toiletMonitoringConfig.robotId,
      timezone: this.toiletMonitoringConfig.timezone
    })

    // Handle graceful shutdown
    process.on('SIGTERM', () => this.stop())
    process.on('SIGINT', () => this.stop())
  }

  async stop(): Promise<void> {
    if (this.isStopping) return
    this.isStopping = true

    const logger = Logger.loggerFromCtx(this.ctx)
    logger.info('Stopping Toilet Monitoring Job Mode')

    await this.asyncContainer.stop(this.ctx, [
      ContainerNames.ToiletWindowMonitor,
      ContainerNames.ToiletMonitoringWorker,
      ContainerNames.ToiletMonitoringScheduler
    ])

    logger.info('Toilet Monitoring Job Mode stopped')
    process.exit(0)
  }
}

export async function bootstrap(): Promise<JobApp> {
  const mysqlConfig = loadConfigValue('mysql', MySQLConfig)
  const logConfig = loadConfigValue('logs', LogConfig)
  const appConfig = loadConfigValue('app', AppConfig)
  const sqsConfig = loadConfigValue('sqs', SQS.SQSConfig)
  const megazordEventsConfig = loadConfigValue('megazordEvents', AddressConfig)
  const statusQueueConfig = loadConfigValue('statusQueue', QueueConfig)
  const statusChecksSchedulerConfig = loadConfigValue('statusChecksScheduler', SchedulerConfig)
  const toiletMonitoringConfig = loadConfigValue('toiletMonitoring', ToiletMonitoringConfig)

  const app = new JobApp(
    mysqlConfig,
    logConfig,
    appConfig,
    sqsConfig,
    megazordEventsConfig,
    statusQueueConfig,
    statusChecksSchedulerConfig,
    toiletMonitoringConfig
  )

  await app.start()
  return app
}
```

**index.ts:**

```typescript
import { bootstrap } from './main'

bootstrap().catch((error) => {
  console.error('Failed to start job mode:', error)
  process.exit(1)
})
```

**Rationale:** Completely separate bootstrap from API mode. No HTTP server, no Kong auth, no robot validators. Only database, SQS, and job schedulers.

---

#### Step 8: Update Container Names Constants

**Files to modify:**

- `src/constants/ContainerNames.ts` (or create if doesn't exist)

**Implementation:**

```typescript
export const ContainerNames = {
  // Existing (used by API mode)
  StatusChecksService: 'StatusChecksService',
  StatusChecksSQSProducerService: 'StatusChecksSQSProducerService',
  StatusChecksTeardownSchedulerService: 'StatusChecksTeardownSchedulerService',
  MegazordEventsService: 'MegazordEventsService',
  MicroManagersService: 'MicroManagersService',
  StatusTemplatesLoaderService: 'StatusTemplatesLoaderService',
  StatusChecksRepository: 'StatusChecksRepository',
  StatusTemplateRepository: 'StatusTemplateRepository',
  StatusChecksSQSClient: 'StatusChecksSQSClient',
  StatusQueueConfig: 'StatusQueueConfig',
  RobotQueueConfig: 'RobotQueueConfig',
  StatusChecksSchedulerConfig: 'StatusChecksSchedulerConfig',
  
  // New (job mode only)
  ToiletMonitoringConfig: 'ToiletMonitoringConfig',
  ToiletMonitoringService: 'ToiletMonitoringService',
  ToiletMonitoringScheduler: 'ToiletMonitoringScheduler',
  ToiletMonitoringWorker: 'ToiletMonitoringWorker',
  ToiletWindowMonitor: 'ToiletWindowMonitor'
} as const
```

---

#### Step 9: Update Configuration Files

**Files to modify:**

- `config/default.json`

**Add new section:**

```json
{
  "app": {
    "appName": "azi-3-status-check",
    "isLocal": true
  },
  
  "toiletMonitoring": {
    "enabled": false,
    "robotId": null,
    "timezone": "America/Los_Angeles",
    "dailyStartTime": "08:00",
    "dailyEndTime": "20:00",
    "windowDurationMinutes": 120,
    "alarmLevel": "WARNING",
    "templateName": "toilet_activity_monitoring"
  }
}
```

**config/production.json:**

```json
{
  "toiletMonitoring": {
    "enabled": true,
    "robotId": 12345,
    "timezone": "America/Los_Angeles",
    "dailyStartTime": "06:00",
    "dailyEndTime": "22:00",
    "windowDurationMinutes": 120,
    "alarmLevel": "ERROR",
    "templateName": "toilet_activity_monitoring"
  }
}
```

**Rationale:** Environment variables or config files control which robot to monitor, making it easy to adjust without code changes.

---

#### Step 10: Update Deployment Configuration

**Files to modify:**

- `Dockerfile` (if needs job mode support)
- `ci/docker-compose.yml` (add job service)

**docker-compose.yml addition:**

```yaml
services:
  # Existing API service
  azi-3-status-check-api:
    build: .
    command: yarn start
    environment:
      - NODE_ENV=production
    # ... existing config

  # New job service
  azi-3-status-check-job:
    build: .
    command: yarn start:job
    environment:
      - NODE_ENV=production
      - TOILET_MONITORING_ENABLED=true
      - TOILET_MONITORING_ROBOT_ID=${ROBOT_ID}
    depends_on:
      - mysql
      - localstack
    # ... similar config to API service
```

**Rationale:** Deploy as separate container/process. Can scale independently. API can run multiple instances, job only needs one.

---

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements

- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Critical Decisions Required (For Stakeholder Discussion)

#### Decision 1: Discriminator Strategy - How to Distinguish Job Mode vs API Mode?

**Context**: Both modes share `status_check` table and consume same `statusQueue`. Must prevent cross-processing.

**Options Summary** (see Phase 1.5 for details):

| Option                        | Implementation Speed              | Risk Level                | Long-term Maintainability        |
|-------------------------------|-----------------------------------|---------------------------|----------------------------------|
| 1. Use step_id=0              | ⚡⚡⚡ Fastest (no migration)        | ⚠️ Medium (dummy records) | ❌ Low (confusing semantics)      |
| 2. Add monitoring_type column | ⚡⚡ Medium (1 migration)           | ✅ Low                     | ✅ High (explicit, extensible)    |
| 3. Make columns nullable      | ⚡⚡ Medium (complex migration)     | ⚠️ Medium (FK changes)    | ✅ High (clean semantics)         |
| 4. Separate SQS queues        | ⚡ Slow (infra + Megazord changes) | ✅ None (isolated)         | ✅ Excellent (complete isolation) |
| 5. Separate tables            | ⚡ Slow (new tables + code)        | ✅ None (isolated)         | ✅ Excellent (zero conflicts)     |

**Questions for Stakeholder**:

1. **Timeline priority**: Need launch in 2 weeks (Option 1) or can wait 3-4 weeks for cleaner solution (Option 2)?
2. **Future scale**: Expect to add more autonomous monitoring types beyond toilet? (favors Option 2, 4, or 5)
3. **Risk tolerance**: OK with dummy records and step_id=0 hack for pilot? Or prefer clean schema from start?
4. **Infrastructure flexibility**: Can provision separate SQS queue easily? (makes Option 4 viable)

**Recommendation**:

- **Pilot**: Option 1 (fastest validation)
- **Production**: Option 2 (best balance)
- **Long-term/Scale**: Option 4 or 5 (if this becomes major feature)

---

#### Decision 2: Configuration Storage - JSON Config vs Database?

**Context**: Need to configure monitoring per robot (robotId, timezone, time A/B).

**Option A: JSON Config File (Current Plan)**

```json
{
  "toiletMonitoring": {
    "robots": [
      { "robotId": 123, "timezone": "America/Los_Angeles", ... }
    ]
  }
}
```

**Pros**: Simple, no DB changes, fast to implement
**Cons**: Requires redeploy to change config, no runtime updates

**Option B: Database Table**

```sql
CREATE TABLE robot_monitoring_config (
  robot_id INT UNSIGNED NOT NULL,
  monitoring_type VARCHAR(64) NOT NULL,
  timezone VARCHAR(64) NOT NULL,
  daily_start_time TIME NOT NULL,
  daily_end_time TIME NOT NULL,
  ...
)
```

**Pros**: Dynamic config, can update via admin API, audit trail
**Cons**: More implementation work, need admin interface

**Questions**:

1. How often will config change? (daily → DB, rarely → config file)
2. Need to add/remove robots without deploy? (yes → DB)
3. Multiple environments with different robots? (DB easier to manage)

**Recommendation**: Start with config file for pilot, migrate to DB if config changes frequently.

---

#### Decision 3: Handling Script Column Constraints

**Context**: `step_id`, `script_version_id`, `scheduled_id` have NOT NULL + FK constraints.

**Options**:

1. **Create dummy records** (step_id=0, script_version_id=0, etc.)
2. **Migration to make nullable** (ALTER TABLE)
3. **Use separate table** (status_check_job)

**Impact on ActiveModel.loadNextStepResources()**:

- Option 1: Must create dummy `script_next_multiple_choice` records OR modify loadNextStepResources
- Option 2: Modify loadNextStepResources to skip if NULL
- Option 3: Create separate JobActiveModel

**Recommended**: Modify `loadNextStepResources()` to gracefully skip when step_id=0/NULL (least invasive).
