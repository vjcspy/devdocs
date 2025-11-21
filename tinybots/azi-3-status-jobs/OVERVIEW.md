# azi-3-status-jobs

## Overview

`azi-3-status-jobs` is a **stateless toilet activity monitoring service** that tracks resident activity patterns and emits alarms when concerning gaps are detected. This service is designed for pilot testing with minimal infrastructure requirements - no database connections, all configuration via environment variables.

## Purpose

Monitor toilet activity for specific residents during configurable daily time windows (e.g., 8 AM - 8 PM). If no `TOILET_ACTIVITY` events are detected within a 2-hour sliding window, the service emits a `NO_TOILET_ACTIVITY_ALARM` event to Megazord for dashboard/notification processing.

## Architecture

### Stateless Design

- **No Database**: All state maintained in-memory during process lifetime
- **Environment-based Config**: Single JSON environment variable supports multiple robots
- **Restart Semantics**: On restart, all state is lost and monitoring sessions are recreated from scratch
- **Future Persistence**: Code structured with `flush()`/`restore()` placeholders for easy database integration

### Key Components

#### 1. Configuration (`src/config/`)
- **`TOILET_MONITORING_CONFIG`**: Single JSON environment variable containing:
  - Megazord Event Service URL
  - SQS queue URL and AWS region
  - Window duration (default 120 minutes)
  - Array of robot configurations (robotId, timezone, daily window, enabled flag)

#### 2. Domain Models (`src/domain/`)
- **`MonitoringSession`**: Represents one daily monitoring session for a robot
  - One active session per robot at a time
  - Tracks start/end times, subscription ID, current window state
  - Includes `toJSON()`/`fromJSON()` for future persistence
  
- **`MonitoringWindow`**: Rolling 2-hour window within a session
  - Tracks window start/end times, events received, alarms emitted
  - Resets when activity detected

#### 3. Infrastructure (`src/infrastructure/`)
- **`Clock`**: Timezone-aware time abstraction (handles DST transitions)
- **`MegazordEventClient`**: Wraps `tiny-internal-services` EventService
  - Creates/deletes subscriptions
  - Posts alarm events
- **`SubscriptionManager`**: Manages Megazord subscription lifecycle

#### 4. Services (`src/services/`)
- **`WindowTracker`**: In-memory state machine for all monitoring sessions
  - Enforces single active session per robot
  - Handles activity events from SQS (deduplication, window resets)
  - Indexes sessions by robotId and subscriptionId
  
- **`AlarmEmitter`**: Posts `NO_TOILET_ACTIVITY_ALARM` to Megazord
  - Deduplicates alarms (in-memory Set)
  
- **`MonitoringScheduler`**: Creates daily sessions at configured start times
  - Per-robot timezone-aware scheduling
  - Handles midnight rollovers and mid-day restarts

#### 5. Background Jobs (`src/jobs/`)
All implement `IAsyncModule` for lifecycle management:

- **`MonitorWorker`**: SQS consumer polling status queue
  - Filters for `TOILET_ACTIVITY` events
  - Updates `WindowTracker` with activity timestamps
  
- **`WindowExpirationChecker`**: Cron job (every minute)
  - Finds expired windows
  - Emits alarms via `AlarmEmitter`
  - Advances windows for next monitoring period
  
- **`SessionCleanupJob`**: Cron job (hourly)
  - Placeholder for cleanup logic (completed sessions, orphaned subscriptions)

### Multi-Robot Support

From day one, the service supports monitoring **multiple robots concurrently**:
- Each robot has independent timezone and daily monitoring window
- Each robot gets one active `MonitoringSession` at a time
- SQS subscription filtering by subscriptionId ensures correct event routing
- All robots share the same 2-hour window duration (configurable globally)

## Event Flow

1. **Session Creation** (at Time A per robot timezone):
   - `MonitoringScheduler` creates `MonitoringSession`
   - Creates Megazord subscription for `TOILET_ACTIVITY` events
   - Initializes first 2-hour window

2. **Activity Monitoring**:
   - `MonitorWorker` polls SQS queue
   - On `TOILET_ACTIVITY` event: reset window from event time
   - Deduplicate by event ID

3. **Gap Detection**:
   - `WindowExpirationChecker` runs every minute
   - Finds expired windows (no activity for 2 hours)
   - Emits `NO_TOILET_ACTIVITY_ALARM`
   - Advances window for next 2-hour period

4. **Session Completion** (at Time B or when monitoring window extends beyond end time):
   - Mark session as `COMPLETED`
   - Clean up Megazord subscription
   - Remove from in-memory state after grace period

## Configuration Example

```json
{
  "megazordEventServiceUrl": "http://megazord-events:3000",
  "statusQueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789/status-queue",
  "awsRegion": "us-east-1",
  "windowDurationMinutes": 120,
  "robots": [
    {
      "robotId": 789,
      "timezone": "America/New_York",
      "dailyWindow": {
        "startTime": "08:00",
        "endTime": "20:00"
      },
      "enabled": true
    },
    {
      "robotId": 790,
      "timezone": "America/Los_Angeles",
      "dailyWindow": {
        "startTime": "07:00",
        "endTime": "19:00"
      },
      "enabled": true
    }
  ]
}
```

Set as environment variable:
```bash
export TOILET_MONITORING_CONFIG='{"megazordEventServiceUrl":"http://...","robots":[...]}'
```

## Restart Behavior

**Acceptable for pilot**: On service restart, all in-memory state is lost.

- Active monitoring sessions are recreated at next scheduled time (Time A)
- If restart happens mid-window, monitoring resumes from current time (not retroactive)
- Alarm deduplication cache is lost (could cause duplicate alarms immediately after restart if window already expired)

**Future enhancement**: Add persistence layer to restore state from database.

## Scaling Considerations

**Current design**: Single instance only (stateful in-memory).

**Future options**:
- Use Redis for shared state across multiple instances
- Add database persistence for session state
- Implement leader election for scheduling (only one instance creates sessions)

## Health & Monitoring

### Endpoints

- `GET /health`: Returns active session count and memory usage
- `GET /internal/v1/monitoring/sessions`: Debug endpoint listing all active sessions (remove in production)

### Logging

Structured JSON logging with:
- `robotId`, `sessionId`, `windowNumber` in all relevant logs
- Key events: session created, subscription created, window reset, alarm emitted, session completed

### Metrics (Future)

- `azi3.toilet.session.created`
- `azi3.toilet.activity.received`
- `azi3.toilet.window.reset`
- `azi3.toilet.alarm.sent`
- `azi3.toilet.session.completed`

## Event Produced

### NO_TOILET_ACTIVITY_ALARM

- **Producer**: azi-3-status-jobs
- **Trigger**: No TOILET_ACTIVITY events received in 2-hour window
- **Payload**:
  ```json
  {
    "sessionId": "uuid",
    "windowNumber": 3,
    "gapStartTime": "2025-11-20T14:00:00.000Z",
    "gapEndTime": "2025-11-20T16:00:00.000Z",
    "durationMinutes": 120
  }
  ```
- **Level**: 30 (WARNING)
- **hasTrigger**: true
- **Consumers**: Dashboards, notification triggers, care team alerts

## Testing Strategy

### Unit Tests
- `WindowTracker`: Window math, DST handling, single-session-per-robot enforcement
- `AlarmEmitter`: Deduplication logic
- `Clock`: Timezone conversions, DST transitions
- `MonitoringWindow`: Window reset logic, expiration checks

### Integration Tests
- Mock Megazord EventService using `tiny-internal-services-mocks`
- Mock SQS using in-memory queue
- Scenarios:
  - Create session → activity → reset → expire → alarm
  - Multiple windows in same day
  - Service restart (state loss)
  - Out-of-order events
  - Duplicate events

### Manual Validation
1. Set `TOILET_MONITORING_CONFIG` env var
2. Start service
3. Inject `TOILET_ACTIVITY` events via Megazord API
4. Verify window resets in logs
5. Wait for expiration, verify alarm event
6. Restart service, verify new session created

## Deployment

### Docker
Multi-stage build, Node.js base image.

### Environment Variables
- `TOILET_MONITORING_CONFIG` (required): JSON config string
- `NODE_ENV`: production | development
- `PORT`: HTTP server port (default 3000)

### Dependencies
- Megazord Event Service (HTTP)
- AWS SQS (status queue)
- No database required

## Limitations (Pilot)

1. **Single instance**: No horizontal scaling (state is in-memory)
2. **No persistence**: State lost on restart
3. **No audit trail**: Only logs, no database records
4. **Restart gaps**: Monitoring resumes from restart time, not retroactive
5. **Alarm deduplication reset**: On restart, could emit duplicate alarms for already-expired windows

## Future Enhancements

1. **Database persistence**:
   - Implement `WindowTracker.flush()` to save state to database
   - Restore sessions on startup
   - Enable horizontal scaling with shared state

2. **Redis for shared state**:
   - Replace in-memory Maps with Redis
   - Enable multi-instance deployment
   - Leader election for scheduling

3. **Configurable window duration per robot**:
   - Currently global setting
   - Could be per-robot for different care levels

4. **Historical reporting**:
   - Store alarm history in database
   - Analytics dashboard
   - Trend analysis

5. **Dynamic config reload**:
   - Currently requires restart to change config
   - Could watch for config file changes or API endpoint

## Related Documentation

- Implementation plan: `devdocs/tinybots/azi-3-status-jobs/2025-11-20-PROD594-toilet-activity-monitoring.md`
- Megazord event schema: `megazord-events/schemas/events/no_toilet_activity_alarm.json`
- Platform standards: `devdocs/tinybots/_general/tiny-backend-tools/OVERVIEW.md`
