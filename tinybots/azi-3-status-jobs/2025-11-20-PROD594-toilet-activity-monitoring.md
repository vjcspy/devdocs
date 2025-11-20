# 📋 [PROD594: 2025-11-20] - Toilet Activity Monitoring Service (azi-3-status-jobs)

## User Requirements

This is an experiment for a specific user (we could think about a more general approach, but speed is more important. A very specific approach for this user is fine)

What you need:

- Add NO_TOILET_ACTIVITY_ALARM to megazord event schema
- Between time A and time B do the following:
  - At time A subscribe to TOILET_ACTIVITY event in megazord events and check from now to 2 hours in the future for a specific user
  - If a TOILET_ACTIVITY event is received in these 2 hours, plan a new check from now to 2 hours in the future (or until time B, whichever comes first)
  - If no TOILET_ACTIVITY event is received after 2 hours, send a new event to megazord event called NO_TOILET_ACTIVITY_ALARM then start a new check from now to 2 hours in the future (or until time B, whichever comes first)

**Additional Constraints**:

1. **No Database**: Do NOT create or use any database tables. The service will NOT connect to database at all (for now - design should allow easy database integration later)
2. **Environment-Based Configuration**: Single environment variable `TOILET_MONITORING_CONFIG` containing JSON array to support **multiple robots**
3. **Multi-Robot Support**: Architecture must handle monitoring multiple robots simultaneously from day 1

## 🎯 Objective

Build a new standalone, **stateless** service `azi-3-status-jobs` that monitors toilet activity for specific residents in configurable time windows, emitting alarms when 2-hour activity gaps are detected. This service operates completely in-memory without any database dependencies.

### ⚠️ Key Considerations

**Architectural Decision**: Create new dedicated, stateless service

- **Zero risk** to production systems (no database, no shared state)
- **Ultra-fast delivery** (1 week for MVP - no DB schema, repositories, or migrations)
- **Stateless design** for time-based monitoring using only in-memory tracking
- **Easier experimentation** and iteration without data persistence concerns
- **Minimal infrastructure** (just SQS + Megazord Event Service)

**Critical Requirements**:

- **NO DATABASE**: Service runs entirely stateless with in-memory state tracking
- **Environment-based config**: All settings (robot, timezone, time windows) via single JSON env var
- SQS delivery is at-least-once; must deduplicate and handle out-of-order events in memory
- Time A/B are resident-timezone driven; must handle DST transitions
- Subscription lifecycle managed in memory (create at A, cleanup at B or on failure)
- On restart, recreate subscriptions from scratch (acceptable for pilot)
- Alarms flow through Megazord for consistent trigger/dashboard integration

**Speed over Perfection**:

- Single pilot user/robot; all config in environment variable
- Stateless in-memory state machine - restart resets everything
- No persistence, no audit trail (logs only)
- Focus on proving the monitoring logic works
- If successful, can add persistence layer later

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze detailed requirements
  - **Outcome**: New stateless service `azi-3-status-jobs` will:
    1. Add `NO_TOILET_ACTIVITY_ALARM` to Megazord schemas (simple format like bathroom_activity.json)
    2. Run entirely in-memory without any database connections (but designed for easy persistence later)
    3. Parse monitoring config from single JSON environment variable supporting **multiple robots**
    4. Create daily Megazord subscriptions at configured time A for **each configured robot**
    5. Maintain rolling 2-hour windows in-memory by consuming SQS messages for **all monitored robots**
    6. Emit alarms through Megazord when gaps detected for any monitored robot
    7. Clean up subscriptions at time B or on completion for each robot
    8. Design state management with future `flush()` method for database persistence

- [x] Define scope and edge cases
  - **Outcome**: Must handle:
    - DST transitions (23/25-hour days)
    - Windows crossing midnight
    - Duplicate/out-of-order SQS deliveries (in-memory deduplication)
    - Service restarts (acceptable: lose state, recreate subscriptions)
    - Dynamic config updates require restart (acceptable for pilot)
    - Alarm deduplication within current process lifetime
    - Graceful shutdown with subscription cleanup
    - Multiple residents/robots via JSON config (future expansion)

### Phase 2: Implementation (File/Code Structure)

```text
azi-3-status-jobs/
├── package.json                             # ✅ Dependencies: tiny-internal-services, express, luxon
├── tsconfig.json                            # ✅ Extends base TS config
├── tsconfig.base.json                       # ✅ Shared compiler options
├── tsconfig.prod.json                       # ✅ Production build config
├── Dockerfile                               # 🚧 TODO - Multi-stage Node.js build
├── dprint.json                              # 🚧 TODO - Code formatting config
├── eslint.config.js                         # 🚧 TODO - Linting rules
├── devdocs                                  # 🚧 TODO - Service-specific documentation
│   └── OVERVIEW.md                          # 🚧 TODO - Architecture & stateless design
├── config/
│   ├── default.json                         # 🚧 TODO - Minimal config (SQS, EventService URLs)
│   └── production.json                      # 🚧 TODO - Production overrides
├── src/
│   ├── cmd/
│   │   └── main.ts                          # 🚧 TODO - Bootstrap Express app + background jobs
│   ├── config/
│   │   ├── index.ts                         # 🚧 TODO - Config loader + JSON env var parser
│   │   └── types.ts                         # 🚧 TODO - TypeScript config interfaces
│   ├── domain/
│   │   ├── MonitoringSession.ts             # 🚧 TODO - In-memory session model
│   │   └── MonitoringWindow.ts              # 🚧 TODO - Rolling window value object
│   ├── infrastructure/
│   │   ├── MegazordEventClient.ts           # 🚧 TODO - Wraps EventService from tiny-internal
│   │   ├── StatusQueueConsumer.ts           # 🚧 TODO - SQS consumer with filtering
│   │   ├── Clock.ts                         # 🚧 TODO - Timezone-aware clock for testing
│   │   └── SubscriptionManager.ts           # 🚧 TODO - Lifecycle management for subscriptions
│   ├── services/
│   │   ├── MonitoringScheduler.ts           # 🚧 TODO - Creates sessions at time A daily
│   │   ├── WindowTracker.ts                 # 🚧 TODO - In-memory rolling window state machine
│   │   ├── AlarmEmitter.ts                  # 🚧 TODO - Posts alarms to Megazord
│   │   └── SessionCoordinator.ts            # 🚧 TODO - Orchestrates session lifecycle
│   ├── jobs/
│   │   ├── MonitorWorker.ts                 # 🚧 TODO - Main SQS polling loop
│   │   ├── WindowExpirationChecker.ts       # 🚧 TODO - Periodic check for expired windows
│   │   └── SessionCleanupJob.ts             # 🚧 TODO - Cleanup completed sessions
│   ├── types/
│   │   └── index.ts                         # 🚧 TODO - Shared DTOs/enums
│   └── index.ts                             # 🚧 TODO - Export app for tests
├── scripts/
│   └── validate-config.ts                   # 🚧 TODO - CLI to test JSON env var parsing
├── ci/
│   ├── localtest.sh                         # 🚧 TODO - Lint + unit tests (no integration)
│   └── Jenkinsfile                          # 🚧 TODO - CI/CD pipeline
└── test/
    ├── helpers/
    │   ├── factories.ts                     # 🚧 TODO - Test data builders
    │   └── mocks.ts                         # 🚧 TODO - Mocked Megazord/SQS
    └── services/
        ├── WindowTracker.test.ts            # 🚧 TODO - Unit tests for window logic
        ├── AlarmEmitter.test.ts             # 🚧 TODO - Deduplication tests
        └── SessionCoordinator.test.ts       # 🚧 TODO - Lifecycle scenarios
```

**Supporting Repositories** (External changes):

```text
megazord-events/
├── schemas/
│   ├── gen.ts                               # 🚧 TODO - Add NO_TOILET_ACTIVITY_ALARM constant
│   └── events/
│       └── no_toilet_activity_alarm.json    # 🚧 TODO - New event schema definition

tiny-internal-services/
├── lib/
│   └── model/
│       └── events/
│           └── TinybotsEvent.ts             # 🚧 TODO - Export NO_TOILET_ACTIVITY_ALARM enum

devdocs/tinybots/
├── megazord-events/OVERVIEW.md              # 🚧 TODO - Document new alarm event
└── azi-3-status-jobs/OVERVIEW.md            # 🚧 TODO - Document stateless architecture
```

**Note**: No database repositories, migrations, or typ-e changes needed since this service is stateless.

### Phase 3: Detailed Implementation Steps

#### Step 1: Extend Event Vocabulary (Megazord + tiny-internal-services)

**1.1 Add event to Megazord schemas**

- **File**: `megazord-events/schemas/gen.ts`
- **Action**: Add `NO_TOILET_ACTIVITY_ALARM` to event constants array
- **Details**: Follow existing pattern (e.g., similar to `LOCATION_CHANGE`, `FALL_DETECTED`)

**1.2 Create event schema**

- **File**: `megazord-events/schemas/events/no_toilet_activity_alarm.json`
- **Schema** (simple format matching bathroom_activity.json):

  ```json
  {
    "eventName": "NO_TOILET_ACTIVITY_ALARM",
    "level": 30,
    "hasTrigger": true,
    "isActive": true,
    "description": "No toilet activity detected in 2-hour monitoring window"
  }
  ```

- **Regenerate**: Run `yarn generate-schemas` to update TypeScript types
- **Note**: All monitoring metadata (gap times, window numbers, etc.) will be in event payload, not schema

**1.3 Update tiny-internal-services**

- ✅ **ALREADY DONE**: `NO_TOILET_ACTIVITY_ALARM` already added to `TinybotsEvent` enum
- No action needed in this step

**1.4 Document the event**

- **File**: `devdocs/tinybots/megazord-events/OVERVIEW.md`
- **Section**: Add to event catalog:

  ```markdown
  ### NO_TOILET_ACTIVITY_ALARM
  - **Producer**: azi-3-status-jobs
  - **Trigger**: No TOILET_ACTIVITY events received in 2-hour window
  - **Payload**: Gap timestamps, session ID, window number
  - **Level**: WARNING (configurable)
  - **Consumers**: Dashboards, notification triggers
  - **Note**: Stateless monitoring - no database persistence
  ```

#### Step 2: Scaffold azi-3-status-jobs Stateless Service

**2.1 Initialize repository structure**

- **Base architecture**: Lightweight Express app (no TinyDatabaseApp - no DB needed)
- **Dependencies**:

  ```json
  {
    "dependencies": {
      "tiny-internal-services": "workspace:*",
      "express": "^4.18.2",
      "luxon": "^3.4.0",
      "zod": "^3.22.0",
      "@aws-sdk/client-sqs": "^3.400.0",
      "node-cron": "^3.0.2"
    }
  }
  ```

**2.2 Environment Variable Configuration**

- **Environment Variable**: `TOILET_MONITORING_CONFIG` (single JSON string)
- **Schema** (supports **multiple robots**):

  ```json
  {
    "megazordEventServiceUrl": "http://megazord-events:3000",
    "statusQueueUrl": "https://sqs.us-east-1.amazonaws.com/...",
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

- **Validation**: Use Zod schema to parse and validate array of robots on startup
- **Error Handling**: Fail fast if config is invalid, log which robot configs are problematic
- **Note**: Each robot can have different timezone and monitoring windows

**2.3 Bootstrap application**

- **File**: `src/cmd/main.ts`
- **Pattern**:

  ```typescript
  import express from 'express';
  import { parseConfig } from '../config';
  import { initializeServices } from '../services';
  import { startJobs } from '../jobs';

  const app = express();
  const config = parseConfig(process.env.TOILET_MONITORING_CONFIG);

  const services = initializeServices(config);
  startJobs(services);

  app.get('/health', (req, res) => res.json({ status: 'ok' }));
  app.listen(config.port || 3000);
  ```

#### Step 3: Implement Core Infrastructure

**3.1 MegazordEventClient**

- **File**: `src/infrastructure/MegazordEventClient.ts`
- **Purpose**: Wraps `tiny-internal-services` EventService with error handling
- **Methods**:
  - `postSubscription(robotId, eventNames, until)`: Create subscription
  - `deleteSubscription(subscriptionId)`: Cleanup
  - `postEvent(robotId, event)`: Emit alarm
- **No getEvents()**: We don't replay missed events (stateless design)
- **Error Handling**: Retry on 5xx, log failures

**3.2 StatusQueueConsumer**

- **File**: `src/infrastructure/StatusQueueConsumer.ts`
- **Base**: AWS SDK SQS client
- **Logic**:

  ```typescript
  async *consumeMessages() {
    while (this.running) {
      const response = await this.sqsClient.receiveMessage({
        QueueUrl: this.config.statusQueueUrl,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20,
      });

      for (const msg of response.Messages || []) {
        const parsed = JSON.parse(msg.Body);
        if (this.shouldProcess(parsed)) {
          yield { 
            message: parsed, 
            ack: () => this.sqsClient.deleteMessage({
              QueueUrl: this.config.statusQueueUrl,
              ReceiptHandle: msg.ReceiptHandle,
            })
          };
        } else {
          await this.sqsClient.deleteMessage({
            QueueUrl: this.config.statusQueueUrl,
            ReceiptHandle: msg.ReceiptHandle,
          });
        }
      }
    }
  }

  private shouldProcess(msg): boolean {
    // Check if this event is for any of our monitored robots
    const monitoredRobotIds = new Set(this.config.robots.map(r => r.robotId));
    return msg.sourceEvent?.eventName === 'TOILET_ACTIVITY' &&
           monitoredRobotIds.has(msg.sourceEvent?.robotId);
  }
  ```

**3.3 Clock (Timezone Abstraction)**

- **File**: `src/infrastructure/Clock.ts`
- **Purpose**: Deterministic time for tests, DST-aware
- **Interface**:

  ```typescript
  interface Clock {
    now(): DateTime; // Luxon DateTime in configured timezone
    todayAt(time: string): DateTime; // "08:00" -> today at 8am
    isBetween(start: DateTime, end: DateTime): boolean;
  }
  ```

#### Step 4: Implement In-Memory State Management

**4.1 MonitoringSession (Domain Model)**

- **File**: `src/domain/MonitoringSession.ts`
- **Purpose**: Represents one day's monitoring session (in-memory, designed for persistence)
- **Design**: Prepare for future database persistence with `toJSON()`/`fromJSON()` and `flush()` methods
- **Properties**:

  ```typescript
  class MonitoringSession {
    id: string; // UUID
    robotId: number;
    subscriptionId: number | null;
    startTime: DateTime; // Time A
    endTime: DateTime; // Time B
    currentWindow: MonitoringWindow;
    status: 'PENDING' | 'ACTIVE' | 'COMPLETED';
    createdAt: DateTime;
    updatedAt: DateTime;
    
    // Future persistence support
    toJSON(): object {
      // Serialize for database/logging
      return {
        id: this.id,
        robotId: this.robotId,
        subscriptionId: this.subscriptionId,
        startTime: this.startTime.toISO(),
        endTime: this.endTime.toISO(),
        currentWindow: this.currentWindow.toJSON(),
        status: this.status,
        createdAt: this.createdAt.toISO(),
        updatedAt: this.updatedAt.toISO(),
      };
    }
    
    static fromJSON(data: object): MonitoringSession {
      // Deserialize from database
      // Implementation for future database loading
    }
  }
  ```

**4.2 MonitoringWindow (Value Object)**

- **File**: `src/domain/MonitoringWindow.ts`
- **Purpose**: Represents current 2-hour window
- **Design**: Immutable value object with serialization support
- **Properties**:

  ```typescript
  class MonitoringWindow {
    since: DateTime;
    until: DateTime;
    windowNumber: number;
    eventsReceived: Set<string>; // Dedupe by event ID
    alarmsEmitted: number; // Count of alarms in this session
    
    isExpired(now: DateTime): boolean {
      return now >= this.until;
    }
    
    contains(eventTime: DateTime): boolean {
      return eventTime >= this.since && eventTime < this.until;
    }
    
    resetFrom(eventTime: DateTime, maxUntil: DateTime, duration: number): MonitoringWindow {
      return new MonitoringWindow(
        eventTime,
        DateTime.min(eventTime.plus({ minutes: duration }), maxUntil),
        this.windowNumber + 1,
        new Set(),
        this.alarmsEmitted
      );
    }
    
    // Persistence support
    toJSON(): object {
      return {
        since: this.since.toISO(),
        until: this.until.toISO(),
        windowNumber: this.windowNumber,
        eventsReceived: Array.from(this.eventsReceived),
        alarmsEmitted: this.alarmsEmitted,
      };
    }
    
    static fromJSON(data: any, timezone: string): MonitoringWindow {
      return new MonitoringWindow(
        DateTime.fromISO(data.since, { zone: timezone }),
        DateTime.fromISO(data.until, { zone: timezone }),
        data.windowNumber,
        new Set(data.eventsReceived),
        data.alarmsEmitted || 0
      );
    }
  }
  ```

**4.3 WindowTracker (In-Memory State Machine)**

- **File**: `src/services/WindowTracker.ts`
- **Purpose**: Manages all active monitoring sessions in memory for **multiple robots**
- **Design**: Repository pattern with future persistence via `flush()` method
- **State**:

  ```typescript
  class WindowTracker {
    private sessions: Map<string, MonitoringSession> = new Map(); // sessionId -> session
    private robotSessions: Map<number, Set<string>> = new Map(); // robotId -> sessionIds
    private subscriptionIndex: Map<number, string> = new Map(); // subscriptionId -> sessionId
    
    // Multi-robot session creation
    createSession(robotId: number, start: DateTime, end: DateTime): MonitoringSession {
      const session = new MonitoringSession(uuid(), robotId, start, end);
      this.sessions.set(session.id, session);
      
      // Index by robot
      if (!this.robotSessions.has(robotId)) {
        this.robotSessions.set(robotId, new Set());
      }
      this.robotSessions.get(robotId)!.add(session.id);
      
      this.logger.info('Session created', { sessionId: session.id, robotId });
      return session;
    }
    
    indexSubscription(sessionId: string, subscriptionId: number) {
      this.subscriptionIndex.set(subscriptionId, sessionId);
    }
    
    handleActivity(subscriptionId: number, eventTime: DateTime, eventId: string, duration: number) {
      const sessionId = this.subscriptionIndex.get(subscriptionId);
      if (!sessionId) {
        this.logger.warn('No session found for subscription', { subscriptionId });
        return;
      }
      
      const session = this.sessions.get(sessionId);
      if (!session) return;
      
      // Deduplicate
      if (session.currentWindow.eventsReceived.has(eventId)) {
        this.logger.debug('Duplicate event ignored', { eventId });
        return;
      }
      
      // Check if event is in current window
      if (!session.currentWindow.contains(eventTime)) {
        this.logger.warn('Event outside window', { 
          eventTime: eventTime.toISO(), 
          windowSince: session.currentWindow.since.toISO(),
          windowUntil: session.currentWindow.until.toISO()
        });
        return;
      }
      
      // Record event
      session.currentWindow.eventsReceived.add(eventId);
      session.updatedAt = DateTime.now();
      
      // Reset window
      session.currentWindow = session.currentWindow.resetFrom(
        eventTime,
        session.endTime,
        duration
      );
      
      this.logger.info('Window reset', { 
        sessionId: session.id, 
        robotId: session.robotId,
        newWindowUntil: session.currentWindow.until.toISO() 
      });
      
      // Check if monitoring complete
      if (session.currentWindow.until >= session.endTime) {
        this.completeSession(session.id, 'SUCCESS');
      }
    }
    
    getExpiredWindows(now: DateTime): MonitoringSession[] {
      return Array.from(this.sessions.values()).filter(session =>
        session.status === 'ACTIVE' && session.currentWindow.isExpired(now)
      );
    }
    
    getActiveSessionsForRobot(robotId: number): MonitoringSession[] {
      const sessionIds = this.robotSessions.get(robotId) || new Set();
      return Array.from(sessionIds)
        .map(id => this.sessions.get(id))
        .filter(s => s && s.status === 'ACTIVE') as MonitoringSession[];
    }
    
    completeSession(sessionId: string, reason: string) {
      const session = this.sessions.get(sessionId);
      if (session) {
        session.status = 'COMPLETED';
        session.updatedAt = DateTime.now();
        
        this.logger.info('Session completed', { sessionId, robotId: session.robotId, reason });
        
        // Keep in memory for grace period, then remove
        setTimeout(() => {
          this.sessions.delete(sessionId);
          this.robotSessions.get(session.robotId)?.delete(sessionId);
          if (session.subscriptionId) {
            this.subscriptionIndex.delete(session.subscriptionId);
          }
        }, 60000);
      }
    }
    
    // Future persistence support
    async flush(): Promise<void> {
      // TODO: Implement database persistence
      // For each session in this.sessions:
      //   - Serialize with toJSON()
      //   - UPSERT to database
      //   - Save window state
      //   - Save event history
      this.logger.info('Flush called', { sessionCount: this.sessions.size });
      
      // Placeholder for future implementation:
      // const serialized = Array.from(this.sessions.values()).map(s => s.toJSON());
      // await this.repository.bulkUpsert(serialized);
    }
    
    async restore(): Promise<void> {
      // TODO: Implement database restoration on startup
      // Load active sessions from database
      // Recreate in-memory state
      // Rebuild indexes (robotSessions, subscriptionIndex)
      this.logger.info('Restore called - not yet implemented');
    }
    
    // Debugging/monitoring
    getStats() {
      return {
        totalSessions: this.sessions.size,
        activeSessions: Array.from(this.sessions.values()).filter(s => s.status === 'ACTIVE').length,
        robotsMonitored: this.robotSessions.size,
        subscriptionsIndexed: this.subscriptionIndex.size,
      };
    }
  }
  ```

#### Step 5: Implement Monitoring Services

**5.1 MonitoringScheduler**

- **File**: `src/services/MonitoringScheduler.ts`
- **Trigger**: Cron at midnight (per-timezone) + at Time A for **each robot**
- **Multi-Robot Logic**:

  ```typescript
  class MonitoringScheduler {
    private scheduledJobs: Map<string, NodeJS.Timeout> = new Map();
    
    // Schedule daily sessions for ALL configured robots
    async scheduleAllRobots() {
      for (const robotConfig of this.config.robots) {
        if (!robotConfig.enabled) {
          this.logger.info('Robot monitoring disabled', { robotId: robotConfig.robotId });
          continue;
        }
        
        await this.scheduleDailySessionForRobot(robotConfig);
      }
    }
    
    private async scheduleDailySessionForRobot(robotConfig: RobotConfig) {
      const clock = new Clock(robotConfig.timezone);
      const today = clock.now().startOf('day');
      const [startHour, startMin] = robotConfig.dailyWindow.startTime.split(':').map(Number);
      const [endHour, endMin] = robotConfig.dailyWindow.endTime.split(':').map(Number);
      
      const startTime = today.set({ hour: startHour, minute: startMin, second: 0 });
      const endTime = today.set({ hour: endHour, minute: endMin, second: 0 });
      
      // Schedule subscription creation at startTime
      const delay = startTime.diff(clock.now()).milliseconds;
      
      if (delay > 0) {
        const timeoutId = setTimeout(
          () => this.initializeSession(robotConfig, startTime, endTime, clock),
          delay
        );
        this.scheduledJobs.set(`${robotConfig.robotId}-${today.toISODate()}`, timeoutId);
      } else if (clock.now() < endTime) {
        // Already past start time but before end time - start now
        await this.initializeSession(robotConfig, clock.now(), endTime, clock);
      } else {
        this.logger.info('Monitoring window already passed for today', { 
          robotId: robotConfig.robotId,
          window: `${robotConfig.dailyWindow.startTime} - ${robotConfig.dailyWindow.endTime}`
        });
      }
      
      // Schedule next day's session at midnight (timezone-aware)
      const midnight = today.plus({ days: 1 }).startOf('day');
      const midnightDelay = midnight.diff(clock.now()).milliseconds;
      setTimeout(() => this.scheduleDailySessionForRobot(robotConfig), midnightDelay);
    }
    
    private async initializeSession(
      robotConfig: RobotConfig,
      start: DateTime,
      end: DateTime,
      clock: Clock
    ) {
      const session = this.windowTracker.createSession(
        robotConfig.robotId,
        start,
        end
      );
      
      try {
        // Create Megazord subscription
        const subscription = await this.megazordClient.postSubscription(
          session.robotId,
          ['TOILET_ACTIVITY'],
          end.toJSDate()
        );
        
        session.subscriptionId = subscription.id;
        session.status = 'ACTIVE';
        session.currentWindow = new MonitoringWindow(
          start,
          DateTime.min(start.plus({ minutes: this.config.windowDurationMinutes }), end),
          1,
          new Set(),
          0
        );
        
        // Index subscription for fast lookup
        this.windowTracker.indexSubscription(session.id, subscription.id);
        
        this.logger.info('Monitoring session initialized', { 
          sessionId: session.id,
          robotId: session.robotId,
          timezone: robotConfig.timezone,
          window: `${start.toISO()} - ${end.toISO()}`
        });
      } catch (error) {
        this.logger.error('Failed to initialize session', { 
          robotId: robotConfig.robotId, 
          error 
        });
        this.windowTracker.completeSession(session.id, 'INITIALIZATION_FAILED');
      }
    }
    
    shutdown() {
      // Clear all scheduled timeouts
      for (const [key, timeoutId] of this.scheduledJobs.entries()) {
        clearTimeout(timeoutId);
        this.logger.debug('Cancelled scheduled job', { key });
      }
      this.scheduledJobs.clear();
    }
  }
  ```

**5.2 AlarmEmitter**

- **File**: `src/services/AlarmEmitter.ts`
- **Purpose**: Posts alarm to Megazord (no DB side effects)
- **Deduplication**: Track emitted alarms in memory (Set of alarm IDs)
- **Logic**:

  ```typescript
  class AlarmEmitter {
    private emittedAlarms: Set<string> = new Set(); // sessionId-windowNumber
    
    async emitAlarm(session: MonitoringSession) {
      const alarmKey = `${session.id}-${session.currentWindow.windowNumber}`;
      
      // Deduplicate
      if (this.emittedAlarms.has(alarmKey)) {
        this.logger.info('Alarm already emitted, skipping', { alarmKey });
        return;
      }
      
      const event = await this.megazordClient.postEvent(session.robotId, {
        providerName: 'azi-3-status-jobs',
        eventName: TinybotsEvent.NO_TOILET_ACTIVITY_ALARM,
        level: this.config.monitoring.alarmLevel,
        referenceId: alarmKey,
        payload: {
          gapStartTime: session.currentWindow.since.toISO(),
          gapEndTime: session.currentWindow.until.toISO(),
          monitoringSessionId: session.id,
          windowNumber: session.currentWindow.windowNumber,
        },
      });
      
      this.emittedAlarms.add(alarmKey);
      this.logger.info('Alarm emitted', { sessionId: session.id, eventId: event.id });
      
      // Clean up old alarm keys periodically to prevent memory leak
      if (this.emittedAlarms.size > 1000) {
        this.emittedAlarms.clear();
      }
    }
  }
  ```

#### Step 6: Implement Background Jobs

**6.1 MonitorWorker (SQS Consumer)**

- **File**: `src/jobs/MonitorWorker.ts`
- **Purpose**: Main loop consuming SQS messages
- **Pattern**:

  ```typescript
  class MonitorWorker {
    async start() {
      this.logger.info('Starting monitor worker');
      
      for await (const { message, ack } of this.consumer.consumeMessages()) {
        try {
          await this.processMessage(message);
          await ack();
        } catch (error) {
          this.logger.error('Failed to process message', { error, message });
          // Leave message in queue for retry
        }
      }
    }
    
    private async processMessage(msg: StatusQueueMessage) {
      const eventTime = DateTime.fromISO(msg.sourceEvent.createdAt);
      const robotId = msg.sourceEvent.robotId;
      
      // Get robot-specific config for window duration
      const robotConfig = this.config.robots.find(r => r.robotId === robotId);
      const duration = this.config.windowDurationMinutes;
      
      this.windowTracker.handleActivity(
        msg.subscriptionId,
        eventTime,
        msg.sourceEvent.id,
        duration
      );
    }
  }
  ```

**6.2 WindowExpirationChecker (Cron)**

- **File**: `src/jobs/WindowExpirationChecker.ts`
- **Trigger**: Every minute
- **Logic**:

  ```typescript
  class WindowExpirationChecker {
    async checkExpiredWindows() {
      const now = this.clock.now();
      const expiredSessions = this.windowTracker.getExpiredWindows(now);
      
      for (const session of expiredSessions) {
        // Emit alarm
        await this.alarmEmitter.emitAlarm(session);
        
        // Reset window from now
        session.currentWindow = new MonitoringWindow(
          now,
          DateTime.min(now.plus({ minutes: 120 }), session.endTime),
          session.currentWindow.windowNumber + 1,
          new Set()
        );
        
        // Check if monitoring complete (reached time B)
        if (now >= session.endTime) {
          this.windowTracker.completeSession(session.id, 'ALARM_DAY_COMPLETE');
          
          // Clean up subscription
          if (session.subscriptionId) {
            await this.subscriptionManager.deleteSubscription(session.subscriptionId);
          }
        }
      }
    }
  }
  ```

**6.3 SessionCleanupJob (Cron)**

- **File**: `src/jobs/SessionCleanupJob.ts`
- **Trigger**: Every hour
- **Purpose**: Clean up completed sessions and orphaned subscriptions
- **Logic**:

  ```typescript
  class SessionCleanupJob {
    async cleanup() {
      const now = this.clock.now();
      const oldSessions = this.windowTracker.getCompletedSessions(
        now.minus({ hours: 24 })
      );
      
      for (const session of oldSessions) {
        if (session.subscriptionId) {
          await this.subscriptionManager.deleteSubscription(session.subscriptionId);
        }
        this.windowTracker.removeSession(session.id);
      }
    }
  }
  ```

#### Step 7: Testing Strategy

**7.1 Unit Tests**

- `WindowTracker.test.ts`: Window math, DST handling, edge cases, in-memory state
- `AlarmEmitter.test.ts`: Deduplication logic, payload construction
- `Clock.test.ts`: Timezone conversions, DST transitions
- `MonitoringWindow.test.ts`: Window reset logic, expiration checks

**7.2 Integration Tests**

- Mock Megazord EventService using `tiny-internal-services-mocks`
- Mock SQS using in-memory queue
- Test scenarios:
  - Create session → activity → reset → expire → alarm
  - Multiple windows in same day
  - Service restart (lose state, acceptable)
  - Out-of-order events
  - Duplicate events

**7.3 Manual Validation Playbook**

1. Set `TOILET_MONITORING_CONFIG` env var
2. Start service
3. Inject TOILET_ACTIVITY events via Megazord API
4. Verify window resets via logs
5. Wait for expiration, verify alarm event
6. Check logs for session lifecycle
7. Restart service, verify new session created

#### Step 8: Observability & Operations

**8.1 Structured Logging**

- All log entries include: `robotId`, `sessionId`, `windowSince`, `windowUntil`
- Key events: session created, subscription created, window reset, alarm emitted, session completed
- Use structured JSON logging

**8.2 Metrics (Optional - if metrics library available)**

- `azi3.toilet.session.created`: Daily session creation
- `azi3.toilet.activity.received`: TOILET_ACTIVITY events processed
- `azi3.toilet.window.reset`: Window advancement count
- `azi3.toilet.alarm.sent`: Alarms emitted
- `azi3.toilet.session.completed`: Successful monitoring days

**8.3 Health Checks**

- `GET /health`: Basic liveness + active session count
- `GET /health/sessions`: List active sessions (for debugging)

**8.4 Deployment**

- Dockerfile: Multi-stage build
- CI/CD: Lint → Test → Build → Deploy
- Scaling: Single instance (stateful in-memory) or use Redis for shared state
- Config: Single `TOILET_MONITORING_CONFIG` environment variable

#### Step 9: Documentation

**9.1 Service Overview**

- **File**: `devdocs/tinybots/azi-3-status-jobs/OVERVIEW.md`
- **Sections**: 
  - Purpose (stateless toilet activity monitoring)
  - Architecture (in-memory state, no database)
  - Configuration (JSON env var format)
  - Restart behavior (lose state, recreate sessions)
  - Scaling limitations (single instance or Redis)

**9.2 Update Related Docs**

- `devdocs/tinybots/megazord-events/OVERVIEW.md`: Mention new alarm
- `devdocs/tinybots/OVERVIEW.md`: Add azi-3-status-jobs to service catalog

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements

- [Phase 1] Analyzed requirements and defined comprehensive scope for stateless architecture
- [Phase 1] Identified all edge cases (DST, restarts, duplicates, etc.) with acceptable trade-offs
- [Phase 2] Designed complete repository structure with in-memory state management
- [Phase 3] Created detailed implementation roadmap with 9 sequential steps (no database dependencies)

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Multi-Robot Configuration**: Get initial list of robots to monitor with their timezone and time windows for `TOILET_MONITORING_CONFIG` JSON
- [ ] **Alarm Level**: Determine appropriate `level` field value (10=INFO, 20=NOTICE, 30=WARNING, 40=ERROR?) for `NO_TOILET_ACTIVITY_ALARM`
- [ ] **Event Trigger**: Should `hasTrigger` be `true` or `false` for the new event?
- [ ] **Dashboard Integration**: Verify alarm displays correctly in existing dashboards (may need UI updates)
- [ ] **Escalation Rules**: Define behavior for repeated alarms (multiple gaps in same day, same robot)
- [ ] **Restart Behavior**: Confirm acceptable to lose in-memory state on restart (recreate subscriptions from scratch)
- [ ] **Scaling Strategy**: Single instance only or implement Redis for shared state across multiple instances?
- [ ] **Persistence Timeline**: When should we implement the `flush()` method to add database persistence?
- [ ] **Success Metrics**: Define KPIs to evaluate experiment (alarm accuracy, false positive rate per robot, user feedback)

### 🔮 Future Enhancements (Post-MVP)

- [ ] **Persistence Layer**: Implement `flush()` method to persist sessions to database
  - Add repository layer for MonitoringSession, MonitoringWindow
  - Implement `restore()` to rebuild in-memory state on restart
  - Add audit trail for all window transitions and alarms
- [ ] **Script Integration**: Once persisted, add Micro-Manager nodes for branching on alarm state
- [ ] **Dynamic Configuration**: Move robot config from env var to database for runtime updates
- [ ] **Admin API**: REST endpoints to add/remove/update monitored robots without restart
- [ ] **Dynamic Windows**: Allow different window durations per resident
- [ ] **Shared State with Redis**: Enable horizontal scaling with Redis-backed session storage
- [ ] **Alert Channels**: Direct Slack/email notifications (not just Megazord events)
- [ ] **Dashboard Widget**: Real-time monitoring status visualization for all robots
- [ ] **Historical Reports**: Analytics on activity patterns and alarm frequency (requires persistence)
- [ ] **Per-Robot Metrics**: Track accuracy, false positives, window resets per robot

---

**Last Updated**: 2025-11-20  
**Status**: Ready for Implementation  
**Estimated Timeline**: 1 week to stateless MVP (no database)
