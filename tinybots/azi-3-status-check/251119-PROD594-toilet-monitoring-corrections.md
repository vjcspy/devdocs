# 📋 Corrections & Clarifications for Toilet Activity Monitoring

## 🔍 Critical Issues Identified

### Issue 1: Event Schema Properties ❌

**Problem**: Step 1 incorrectly included `properties` in event schema definition.

**Correction**: Event schemas in `megazord-events/schemas/events/` chỉ chứa metadata, KHÔNG chứa payload structure.

**Correct schema format**:

```json
{
  "eventName": "NO_TOILET_ACTIVITY_ALARM",
  "level": 30,
  "hasTrigger": true,
  "isActive": true,
  "description": "Alarm when no toilet activity detected for 2 hours"
}
```

**Payload** sẽ được truyền khi post event:

```typescript
await megazordEventsService.postEvent(ctx, robotId, {
  providerName: 'azi-3-status-check-job',
  eventName: TinybotsEvent.NO_TOILET_ACTIVITY_ALARM,
  level: 'ERROR',
  payload: {
    windowStart: '2025-11-19T08:00:00Z',
    windowEnd: '2025-11-19T10:00:00Z',
    gapMinutes: 120
  }
})
```

---

### Issue 2: Configuration Scope ❌

**Problem**: Step 3 tạo global config, nhưng toilet monitoring phải theo từng robot.

**Correction**: Configuration phải hỗ trợ **multiple robots**, mỗi robot có:

- `robotId`
- `timezone`
- `dailyStartTime` (Time A)
- `dailyEndTime` (Time B)
- `windowDurationMinutes` (120)

**Proposed config structure**:

```json
{
  "toiletMonitoring": {
    "enabled": true,
    "robots": [
      {
        "robotId": 12345,
        "timezone": "America/Los_Angeles",
        "dailyStartTime": "08:00",
        "dailyEndTime": "20:00",
        "windowDurationMinutes": 120,
        "alarmLevel": "ERROR"
      },
      {
        "robotId": 67890,
        "timezone": "America/New_York",
        "dailyStartTime": "06:00",
        "dailyEndTime": "22:00",
        "windowDurationMinutes": 120,
        "alarmLevel": "WARNING"
      }
    ],
    "templateName": "toilet_activity_monitoring"
  }
}
```

**Alternative approach**: Store config in database table `robot_monitoring_config`:

```sql
CREATE TABLE `robot_monitoring_config` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `robot_id` INT UNSIGNED NOT NULL,
  `monitoring_type` VARCHAR(64) NOT NULL, -- 'TOILET_ACTIVITY'
  `timezone` VARCHAR(64) NOT NULL,
  `daily_start_time` TIME NOT NULL,
  `daily_end_time` TIME NOT NULL,
  `window_duration_minutes` INT NOT NULL DEFAULT 120,
  `alarm_level` VARCHAR(16) NOT NULL DEFAULT 'WARNING',
  `is_active` BOOLEAN NOT NULL DEFAULT TRUE,
  `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_robot_monitoring_type` (`robot_id`, `monitoring_type`),
  CONSTRAINT `fk_robot_monitoring_config_robot` FOREIGN KEY (`robot_id`) REFERENCES `robot_account` (`id`) ON DELETE CASCADE
) ENGINE = InnoDB;
```

**Benefits of DB approach**:

- Dynamic configuration (no redeploy)
- Can add/remove robots via admin API
- Easier to manage multiple robots
- Configuration history/audit trail

---

### Issue 3: Database Schema Understanding ❌

**Problem**: Steps 4-5 sử dụng methods không tồn tại trong `StatusCheckActiveModel` (`setPollerWindow`).

## 📊 Database Tables Analysis

### Table 1: `status_check` (Main tracking table)

| Column | Type | Purpose | Notes for Toilet Monitoring |
|--------|------|---------|----------------------------|
| `id` | BIGINT UNSIGNED | Primary key | Unique ID cho mỗi monitoring session |
| `robot_id` | INT UNSIGNED | FK to robot_account | Robot được monitor |
| `status_check_template_id` | INT UNSIGNED | FK to template | Link đến template 'toilet_activity_monitoring' |
| `phase` | VARCHAR(64) | Current phase | Use `FUTURE` (monitoring active) hoặc `COMPLETED` |
| `subscription_id` | BIGINT UNSIGNED | FK to event_subscription | Link đến Megazord subscription cho TOILET_ACTIVITY |
| `result` | VARCHAR(64) | Final result | `PENDING` (đang chạy), `COMPLETED` (xong), `FAILED` |
| `step_id` | BIGINT UNSIGNED | FK to script_step | **SET NULL** - không dùng script |
| `next_step_id` | BIGINT UNSIGNED | FK to script_step | **SET NULL** - không dùng script |
| `script_version_id` | BIGINT UNSIGNED | FK to script_version | **SET NULL** - không dùng script |
| `scheduled_id` | BIGINT UNSIGNED | Schedule reference | **SET NULL** hoặc tạo dummy value |
| `planned_at` | TIMESTAMP | When monitoring starts | Set = Time A của ngày đó |
| `created_at` | TIMESTAMP | Record creation | Auto |
| `updated_at` | TIMESTAMP | Last modification | Auto on update |

**Key insight**: Columns `step_id`, `script_version_id`, `scheduled_id` có NOT NULL constraint trong schema hiện tại. Phải xử lý:

- **Option A**: Set dummy values (0 hoặc tạo dummy records)
- **Option B**: Migration to make these columns nullable cho job mode
- **Option C**: Create separate table `status_check_job` without script dependencies

**Recommended**: Option A (least invasive) - Use dummy values for pilot, consider Option C if scaling.

---

### Table 2: `status_check_poller` (Window tracking)

| Column | Type | Purpose | Notes for Toilet Monitoring |
|--------|------|---------|----------------------------|
| `status_check_id` | BIGINT UNSIGNED | FK to status_check | Link to main record |
| `since` | TIMESTAMP | Window start time | **Current 2-hour window start** |
| `until` | TIMESTAMP | Window end time | **Current 2-hour window end** (capped by Time B) |
| `lock_id` | VARCHAR(256) | Lock for processing | Used by window expiration monitor |
| `created_at` | TIMESTAMP | Record creation | Auto |
| `updated_at` | TIMESTAMP | Last modification | **CRITICAL**: Updated mỗi lần reset window |

**Key operations**:

1. **Initial creation** (at Time A):
   ```sql
   INSERT INTO status_check_poller (status_check_id, since, until)
   VALUES (?, '2025-11-19 08:00:00', '2025-11-19 10:00:00'); -- Time A to Time A+2h
   ```

2. **Window reset** (when TOILET_ACTIVITY received):
   ```sql
   UPDATE status_check_poller 
   SET since = '2025-11-19 09:30:00',  -- event time
       until = '2025-11-19 11:30:00'   -- event time + 2h (or Time B)
   WHERE status_check_id = ?;
   ```

3. **Query expired windows**:
   ```sql
   SELECT sp.status_check_id
   FROM status_check_poller sp
   JOIN status_check sc ON sc.id = sp.status_check_id
   WHERE sp.until <= NOW()
     AND sp.lock_id IS NULL
     AND sc.result = 'PENDING'
     AND sc.step_id IS NULL;  -- Identify job mode checks
   ```

**Important**: Không có method `setPollerWindow()` trong ActiveModel. Phải **UPDATE trực tiếp** qua repository hoặc raw SQL.

---

### Table 3: `status_check_record` (Event history)

| Column | Type | Purpose | Notes for Toilet Monitoring |
|--------|------|---------|----------------------------|
| `id` | BIGINT UNSIGNED | Primary key | Auto |
| `status_check_id` | BIGINT UNSIGNED | FK to status_check | Link to monitoring session |
| `incoming_event_id` | BIGINT UNSIGNED | FK to incoming_event | Link to TOILET_ACTIVITY event (if activity) |
| `outgoing_event_id` | BIGINT UNSIGNED | FK to outgoing_event | Link to NO_TOILET_ACTIVITY_ALARM (if alarm) |
| `event_name` | VARCHAR(256) | Event type | 'TOILET_ACTIVITY' or 'NO_TOILET_ACTIVITY_ALARM' |
| `order` | INT UNSIGNED | Sequence number | Auto-increment per status_check |
| `phase` | VARCHAR(64) | Phase when recorded | 'FUTURE' (monitoring active) |
| `result` | VARCHAR(64) | Record result | 'EVENT_OBSERVED', 'ALARM_SENT', etc. |
| `created_at` | TIMESTAMP | When recorded | Event timestamp |

**Key operations**:

1. **Record activity event**:
   ```typescript
   activeModel.addRecord(
     TinybotsEvent.TOILET_ACTIVITY,
     incomingEventId,
     domains.StatusCheckPhase.FUTURE,
     'EVENT_OBSERVED',
     eventTimestamp
   )
   ```

2. **Record alarm emission**:
   ```typescript
   activeModel.addRecord(
     TinybotsEvent.NO_TOILET_ACTIVITY_ALARM,
     null, // no incoming event
     outgoingEventId,
     domains.StatusCheckPhase.FUTURE,
     'ALARM_SENT',
     new Date()
   )
   ```

**Usage**: Provides complete audit trail of all events and alarms during monitoring window.

---

### Table 4: `status_check_template` (Template metadata)

**Purpose**: Store template definition for toilet monitoring.

**Seed data needed**:

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
  '[]',  -- No past dependencies
  '[{"eventName":"TOILET_ACTIVITY"}]',  -- Subscribe to TOILET_ACTIVITY
  1,
  TRUE,
  TRUE  -- Continue polling until Time B
);
```

**Note**: Job mode sẽ bypass template evaluation logic (không dùng EJS expressions), chỉ dùng để store metadata.

---

### Table 5: `script_next_multiple_choice` (NOT USED by job mode)

**Problem**: `StatusCheckActiveModel.loadNextStepResources()` expects this table to have data for `step_id`.

**Solution**: Since job mode doesn't use scripts:

1. Set `step_id = 0` (dummy value)
2. Skip loading next step resources in job flow
3. OR create dummy record:
   ```sql
   INSERT INTO script_step (id, ...) VALUES (0, ...);
   INSERT INTO script_next_multiple_choice (script_step_id, intention_type, primary_command, next_step_id)
   VALUES (0, 'other', 'unknown', 0);
   ```

---

## 📝 Corrected Implementation Steps

### Correction for Step 4: ToiletMonitoringService

**Problem**: `activeModel.setPollerWindow()` không tồn tại.

**Solution**: Update poller directly via SQL hoặc tạo helper method mới.

**Corrected method signatures**:

```typescript
export interface IStatusChecksRepository {
  // ... existing methods
  
  // NEW: Update poller window for job mode
  updatePollerWindow(
    statusCheckId: number,
    since: Date,
    until: Date
  ): Promise<void>
}

export class StatusChecksRepository extends Repository {
  private readonly UPDATE_POLLER_WINDOW = `
    UPDATE status_check_poller
    SET since = ?, until = ?, updated_at = NOW()
    WHERE status_check_id = ?
  `

  async updatePollerWindow(
    statusCheckId: number,
    since: Date,
    until: Date
  ): Promise<void> {
    await this.database.query(this.UPDATE_POLLER_WINDOW, [
      since,
      until,
      statusCheckId
    ])
  }
}
```

**Updated ToiletMonitoringService.handleToiletActivity()**:

```typescript
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
    logger.debug('Event after monitoring period, ignoring')
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

  // Update poller window directly via repository
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
```

---

### Correction for Step 5: Understanding SQS Flow

**Problem**: Không rõ `statusChecksSQSClient` poll từ đâu.

**Clarification**: Flow hoạt động như sau:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MEGAZORD EVENTS                              │
│                                                                      │
│  1. Receives TOILET_ACTIVITY event from Sensara/Robot               │
│  2. Stores in incoming_event table                                  │
│  3. Finds matching subscriptions (by robot_id + event_name)         │
│  4. For SERVICE_SUBSCRIPTION type:                                  │
│     - Creates outgoing_event record                                 │
│     - Publishes to STATUS_QUEUE (AWS SQS) via ContextSQS           │
│       Payload: OutgoingEventSQSMessage {                            │
│         subscriptionId,                                             │
│         sourceEvent: IncomingEventDomain                            │
│       }                                                             │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ SQS Message
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AZI-3-STATUS-CHECK (API Mode)                     │
│                                                                      │
│  StatusChecksService.init():                                        │
│    - Polls statusQueue via statusChecksSQSClient                    │
│    - Calls handleStatusCheckSQSMessage()                            │
│    - Loads status_check by subscriptionId                           │
│    - IF status_check found:                                         │
│        → Execute template validation                                │
│        → Update status_check.result                                 │
│        → Notify robot via robotQueue                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ ISSUE: Conflict!
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AZI-3-STATUS-CHECK (Job Mode)                     │
│                                                                      │
│  ToiletMonitoringWorker.init():                                     │
│    - ALSO polls statusQueue via statusChecksSQSClient               │
│    - Calls handleMessage()                                          │
│    - Loads status_check by subscriptionId                           │
│    - IF status_check found:                                         │
│        → Reset poller window                                        │
│        → Update status_check_record                                 │
│                                                                      │
│  ⚠️  PROBLEM: Both API mode and Job mode consume same queue!       │
│      → Race condition                                               │
│      → Messages might go to wrong handler                           │
│      → API mode will pick up job mode messages                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Root cause**: `StatusChecksService.handleStatusCheckSQSMessage()` (API mode) sẽ process **TẤT CẢ** messages từ `statusQueue`, including toilet monitoring messages.

**Current filter in API mode**:

```typescript
// StatusChecksService.handleStatusCheckSQSMessage()
const statusCheck = await this.statusChecksRepository
  .getByRobotIdAndSubscriptionId(
    outgoingEventMessage.payload.sourceEvent.robotId,
    outgoingEventMessage.payload.subscriptionId
  )

if (!statusCheck) {
  // Skip if no status_check found
  await message.ack()
  return
}

// Continue processing...
```

**The conflict**: API mode sẽ tìm thấy toilet monitoring status_check records (vì cùng table) và sẽ cố process chúng!

---

### Solution: Distinguish Job Mode vs API Mode Records

**Option A: Use `step_id` as discriminator**

```typescript
// In StatusChecksService.handleStatusCheckSQSMessage()
const statusCheck = await this.statusChecksRepository
  .getByRobotIdAndSubscriptionId(...)

if (!statusCheck) {
  await message.ack()
  return
}

// SKIP job mode records (identified by null/0 step_id)
if (!statusCheck.model.stepId || statusCheck.model.stepId === 0) {
  logger.debug('Skipping job mode status check in API mode')
  await message.ack()
  return
}

// Continue with API mode processing...
```

**Option B: Use separate SQS queue for job mode**

```json
{
  "statusQueue": {
    "address": "http://STATUS_QUEUE_ADDRESS"  // API mode
  },
  "jobStatusQueue": {
    "address": "http://JOB_STATUS_QUEUE_ADDRESS"  // Job mode only
  }
}
```

Then in Megazord, create subscriptions with different queue targets based on subscription metadata.

**Option C: Add `monitoring_type` column to status_check**

```sql
ALTER TABLE status_check
ADD COLUMN monitoring_type VARCHAR(64) DEFAULT 'SCRIPT_BASED';

-- Job mode uses 'AUTONOMOUS_MONITORING'
-- API mode uses 'SCRIPT_BASED'
```

Filter in both handlers:

```typescript
// API mode
if (statusCheck.model.monitoringType !== 'SCRIPT_BASED') {
  await message.ack()
  return
}

// Job mode
if (statusCheck.model.monitoringType !== 'AUTONOMOUS_MONITORING') {
  await message.ack()
  return
}
```

**Recommended**: Option A (least changes) for pilot, Option C for production scaling.

---

### Updated Step 5: Worker with Conflict Handling

**Corrected ToiletMonitoringWorker**:

```typescript
private async handleMessage(
  ctx: IRequestContext,
  message: SQS.IContextMessage,
  payload: Record<string, any>
): Promise<void> {
  const logger = Logger.loggerFromCtx(ctx)
  
  try {
    const outgoingEvent = domains.OutgoingEventSQSMessage.FromPlain(payload)
    
    // Only process TOILET_ACTIVITY events
    if (outgoingEvent.payload.sourceEvent.eventName !== TinybotsEvent.TOILET_ACTIVITY) {
      await message.ack()
      return
    }

    // Find active monitoring check by subscription ID
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

    // CRITICAL: Skip if this is API mode check (has step_id)
    if (statusCheck.model.stepId && statusCheck.model.stepId > 0) {
      logger.debug('Skipping API mode status check in job worker')
      await message.ack()
      return
    }

    // CRITICAL: Only process if this robot is in our config
    const robotConfig = this.config.robots.find(
      r => r.robotId === statusCheck.model.robotId
    )
    if (!robotConfig) {
      logger.debug('Robot not in job mode config, skipping')
      await message.ack()
      return
    }

    // Handle the toilet activity event
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
```

**Corresponding filter in API mode** (add to existing `StatusChecksService.handleStatusCheckSQSMessage()`):

```typescript
// After loading statusCheck
if (!statusCheck) {
  await message.ack()
  return
}

// NEW: Skip job mode checks
if (!statusCheck.model.stepId || statusCheck.model.stepId === 0) {
  logger.debug('Skipping job mode status check, not for API mode')
  await message.ack()
  return
}

// Continue existing logic...
```

---

## 🎯 Summary of Corrections

| Issue | Original | Corrected |
|-------|----------|-----------|
| **Event Schema** | Included `properties` | Only metadata (level, hasTrigger, isActive) |
| **Config Scope** | Global singleton | Per-robot configuration (array or DB table) |
| **Database Operations** | `setPollerWindow()` method | Direct SQL UPDATE via repository helper |
| **SQS Understanding** | Unclear source | Megazord publishes to statusQueue; both modes consume |
| **Conflict Handling** | None | Filter by `step_id` to distinguish API vs Job mode |
| **Window Management** | Via ActiveModel | Via direct repository UPDATE |
| **Multi-robot Support** | Single robot | Support multiple robots with individual configs |

---

## ✅ Next Steps

1. **Update plan document** with corrected implementation details
2. **Add repository method** `updatePollerWindow()` to StatusChecksRepository
3. **Add conflict filters** to both API mode and Job mode SQS handlers
4. **Design config structure** - decide between JSON config array vs DB table
5. **Plan migration** for making `step_id`, `script_version_id`, `scheduled_id` nullable OR create dummy records
6. **Test isolation** - ensure API mode and Job mode don't interfere

---

**Document Date**: 2025-11-19  
**Status**: Corrections for review
