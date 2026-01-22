# 📋 [REFACTOR: 2025-11-22] - azi-3-status-check-jobs Improvements

## References

- Source requirements: `devdocs/tinybots/azi-3-status-check-jobs/251122-refactor.md` (original user requirements)
- Repository OVERVIEW: `devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md`
- Global TinyBots standard: `devdocs/tinybots/OVERVIEW.md`
- Template reference: `devdocs/agent/TEMPLATE.md`

## User Requirements

1. `MonitoringScheduler` must be implemented as a CronJob to `scheduleAllRobots` at midnight daily. Current implementation only schedules on app start and for the next day, but subsequent days won't run.
2. `MonitoringSession` needs to track window history - when receiving events and creating new windows, previous windows should be preserved in history.
3. `SessionCleanupJob` needs proper implementation (currently a placeholder).

## 🎯 Objective

Refactor the `azi-3-status-check-jobs` service to ensure reliable daily monitoring, proper session lifecycle management, and historical window tracking for audit and debugging purposes.

### ⚠️ Key Considerations

- **Stateless Architecture**: Service runs in-memory only; sessions are lost on restart. Any persistence is future work.
- **Timezone Awareness**: Midnight scheduling must respect per-robot timezone configuration.
- **DI Framework**: Maintain Awilix container registration patterns and lifecycle management.
- **Test Coverage**: Preserve existing thresholds (statements/functions/lines 94%, branches 70%).
- **Backward Compatibility**: Ensure serialization/deserialization of sessions remains compatible with existing event flow.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Review current `MonitoringScheduler` implementation and scheduling logic
  - **Outcome**: Document current behavior - schedules once at startup + self-reschedules for next midnight only
  - **Files**: `src/services/MonitoringScheduler.ts` (to be moved to `src/jobs/`), `src/cmd/main.ts`
  
- [ ] Analyze `MonitoringSession` data model and window management
  - **Outcome**: Identify where windows are created/updated and define history structure
  - **Files**: `src/domain/monitoring/MonitoringSession.ts`, `src/domain/monitoring/MonitoringWindow.ts`
  
- [ ] Examine `SessionCleanupJob` placeholder and determine cleanup criteria
  - **Outcome**: Define cleanup rules (time-based retention, status-based, resource limits)
  - **Files**: `src/jobs/SessionCleanupJob.ts`, `src/services/RuleTracker.ts`

- [ ] Map dependencies and integration points
  - **Outcome**: Confirm Cron registration, IAsyncModule lifecycle, RuleTracker interaction
  - **Files**: `src/cmd/main.ts`, DI container setup

### Phase 2: Implementation (File/Code Structure)

**Current Structure:**
```
src/
├── services/
│   ├── RuleTracker.ts              # ✅ STABLE - May need minor updates for history
│   └── ActionOrchestrator.ts       # ✅ STABLE
├── domain/
│   └── monitoring/
│       ├── MonitoringSession.ts    # 🔄 TO REFACTOR - Add window history
│       ├── MonitoringWindow.ts     # ✅ STABLE - Review serialization
│       └── MonitoringRule.ts       # ✅ STABLE
├── jobs/
│   ├── MonitoringScheduler.ts      # 🔄 TO REFACTOR & MOVE - Convert to CronJob (moved from services/)
│   ├── SessionCleanupJob.ts        # 🚧 TO IMPLEMENT - Cleanup logic
│   ├── WindowExpirationChecker.ts  # ✅ STABLE - May interact with history
│   └── MonitorWorker.ts            # ✅ STABLE
└── cmd/
    └── main.ts                     # 🔄 TO UPDATE - Adjust scheduler registration & import path
```

**File Movement:**
- Move `src/services/MonitoringScheduler.ts` → `src/jobs/MonitoringScheduler.ts`
- Update import paths in `src/cmd/main.ts` and any other files that reference it

### Phase 3: Detailed Implementation Steps

#### Step 1: Move and Refactor MonitoringScheduler to CronJob Pattern

**Original Location**: `src/services/MonitoringScheduler.ts`  
**New Location**: `src/jobs/MonitoringScheduler.ts`

**Changes**:
- Move file from `services/` to `jobs/` directory to align with other cron-based jobs
- Implement `IAsyncModule` interface fully (if not already)
- **Change cron pattern from daily midnight (`0 0 * * *`) to every minute (`*/1 * * * *`)**
- Remove setTimeout-based scheduling logic (no longer needed with per-minute checks)
- Implement minute-by-minute check logic:
  - Check all robot+rule combinations
  - For each rule, check if current time matches the start time of daily window
  - If rule should start now, initialize session
  - If rule is within window time but has no active session, create recovery session (handles startup/failure cases)
- Add proper error handling and recovery for scheduling failures

**DI Container Update** (`src/cmd/main.ts`):
- Update import path: change `from '../services/MonitoringScheduler'` to `from '../jobs/MonitoringScheduler'`
- **Remove manual `scheduleAllRobots()` call at startup** - cron handles everything
- Verify cron context and logger are properly injected

**Expected Behavior**:
- Runs every minute via cron (`*/1 * * * *`)
- Each minute, checks all robot+rule combinations
- Creates session if:
  1. Current time (in robot's timezone) matches rule's start time (hour:minute)
  2. OR rule is within its daily window but has no active session (recovery mode)
- Handles all timezones correctly, including negative UTC offsets
- No need for app startup trigger - first cron run (within 1 minute) handles it

**Testing Considerations**:
- Mock Cron scheduler to verify daily invocation
- Test timezone edge cases (DST transitions, cross-timezone robots)
- Validate that sessions are correctly scheduled for "today" on cold start

#### Step 2: Add Window History to MonitoringSession

**Location**: `src/domain/monitoring/MonitoringSession.ts`

**Changes**:
- Add `windowHistory: MonitoringWindow[]` property
- When advancing window (in `WindowExpirationChecker` or `RuleTracker`):
  - Push current window to `windowHistory` before creating new window
  - Preserve window state (lastActivity, status, timestamps)
- Update `toJSON()` and `fromJSON()` to serialize/deserialize history array
- Add helper method `getWindowHistory(): MonitoringWindow[]` for read access

**Integration Points**:
- `WindowExpirationChecker`: After executing actions on expired window, archive it
- `RuleTracker.handleActivity`: When rolling window forward, preserve previous window
- Ensure history doesn't grow unbounded (consider max history size if needed)

**Data Structure**:
```typescript
{
  id: string
  robotId: number
  ruleId: string
  currentWindow: MonitoringWindow
  windowHistory: MonitoringWindow[]  // <-- NEW
  status: 'ACTIVE' | 'COMPLETED' | 'FAILED'
  // ... existing fields
}
```

**Testing Considerations**:
- Verify history accumulates correctly over multiple window cycles
- Test serialization/deserialization roundtrip with history
- Confirm history is read-only from external consumers

#### Step 3: Implement SessionCleanupJob

**Location**: `src/jobs/SessionCleanupJob.ts`

**Current State**: Placeholder with cron registration (hourly: `0 * * * *`)

**Cleanup Criteria** (define based on requirements):
1. **Completed Sessions**: Remove sessions with status `COMPLETED` older than X hours (e.g., 24h retention)
2. **Failed Sessions**: Archive or remove sessions with status `FAILED` after retention period
3. **Stale Active Sessions**: Detect sessions past their `endTime` but still marked `ACTIVE` (edge case/bug recovery)
4. **Resource Limits**: If in-memory sessions exceed threshold, remove oldest completed sessions

**Implementation Steps**:
- Inject `RuleTracker` and `Logger` via constructor
- Add configuration for retention periods (e.g., `cleanupRetentionHours` in `MonitoringConfig`)
- In `execute()` method:
  ```typescript
  async execute(ctx: IRequestContext): Promise<void> {
    const now = DateTime.now()
    const sessionsToCleanup = this.ruleTracker.findExpiredSessions(now, retentionHours)
    
    for (const session of sessionsToCleanup) {
      await this.ruleTracker.removeSession(session.id)
      this.logger.info('Cleaned up session', { sessionId: session.id, status: session.status })
    }
  }
  ```
- Add `findExpiredSessions()` and `removeSession()` methods to `RuleTracker`
  - `findExpiredSessions`: Filter sessions by status and age
  - `removeSession`: Delete from all internal maps (sessions, ruleKeys, subscriptionIndex)

**RuleTracker Updates** (`src/services/RuleTracker.ts`):
- Add `findExpiredSessions(now: DateTime, retentionHours: number): MonitoringSession[]`
- Add `removeSession(sessionId: string): void` - clean up all indices
- Ensure subscription cleanup (call `SubscriptionManager` to delete Megazord subscriptions)

**Configuration** (`src/config/types.ts`):
- Add optional `cleanupRetentionHours` to `MonitoringConfig` (default: 24)

**Testing Considerations**:
- Mock time to simulate aged sessions
- Verify subscriptions are properly deleted from Megazord
- Test edge case: cleanup doesn't remove active sessions prematurely
- Confirm all internal indices are cleaned (no memory leaks)

#### Step 4: Integration & Validation

**Actions**:
- Update `src/cmd/main.ts`:
  - Update import statement for `MonitoringScheduler` (new path: `../jobs/MonitoringScheduler`)
  - Ensure `MonitoringScheduler` is registered with correct cron pattern
  - Add explicit startup call to `scheduleAllRobots()`
  - Verify `SessionCleanupJob` cron is active (already registered hourly)
  
- Update Health/Debug Endpoints:
  - `GET /health`: Include window history count in stats
  - `GET /internal/v1/monitoring/sessions`: Show `windowHistory` in session dump

- Add Logging:
  - Log when scheduler runs via cron vs startup
  - Log window archival events
  - Log cleanup summary (sessions removed, subscriptions deleted)

**Manual Testing Scenarios**:
1. Start app, verify immediate scheduling + midnight cron registration
2. Simulate time progression, confirm windows move to history
3. Trigger cleanup job, verify old sessions are removed
4. Restart app, confirm no crash from missing persisted state (expected behavior)

### Phase 4: Testing & Quality Assurance

**Unit Tests** (add to `test/` directory):
- `MonitoringScheduler.test.ts`:
  - Test cron invocation pattern
  - Test timezone-aware scheduling
  - Mock time and verify daily re-scheduling
  
- `MonitoringSession.test.ts`:
  - Test window history append logic
  - Test JSON serialization/deserialization with history
  - Test history bounds (if implemented)
  
- `SessionCleanupJob.test.ts`:
  - Test cleanup criteria (completed, failed, stale)
  - Test retention period logic
  - Mock RuleTracker and verify `removeSession` calls
  
- `RuleTracker.test.ts`:
  - Test `findExpiredSessions` filters
  - Test `removeSession` cleans all indices
  - Test subscription cleanup integration

**Integration Tests**:
- End-to-end: Start app → schedule sessions → advance time → verify history → trigger cleanup
- SQS event flow: Ensure window history doesn't break event handling

**Coverage Goals**:
- Maintain existing thresholds: 94% statements/functions/lines, 70% branches
- Run `yarn test` to validate coverage after changes

### Phase 5: Documentation & Deployment

**Update Documentation**:
- `devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md`:
  - Document cron scheduling behavior (daily midnight + startup)
  - Describe window history feature and retention
  - Explain SessionCleanupJob retention policy
  
- Code Comments:
  - Add JSDoc to new methods (`findExpiredSessions`, `removeSession`, history handling)
  - Document configuration options (`cleanupRetentionHours`)

**Configuration Review**:
- Ensure `config/default.json` and `config/custom-environment-variables.json` include cleanup settings
- Validate environment variable mapping for `MONITORING_CLEANUP_RETENTION_HOURS`

**Deployment Checklist**:
- [ ] Run `yarn build` and verify no compilation errors
- [ ] Run `yarn lint` and fix any issues
- [ ] Run `yarn test` and confirm coverage thresholds met
- [ ] Test in dev environment with realistic config
- [ ] Review logs for proper scheduler and cleanup invocations
- [ ] Update any deployment docs or runbooks

## 📊 Summary of Results
> Will be completed after implementation

### ✅ Completed Achievements
- [ ] MonitoringScheduler runs reliably at midnight daily via cron
- [ ] MonitoringSession preserves window history for auditing
- [ ] SessionCleanupJob actively manages session lifecycle and memory
- [ ] Test coverage maintained at required thresholds
- [ ] Documentation updated with new behaviors

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Persistence Strategy**: Current solution remains in-memory only. Consider future work for database persistence of sessions and history.
- [ ] **History Retention**: Define max history size per session to prevent unbounded memory growth (or rely on cleanup job).
- [ ] **Timezone Configuration**: Verify cron scheduler's timezone handling across different deployment environments (UTC vs local).
- [ ] **Subscription Cleanup Errors**: Handle cases where Megazord subscription deletion fails during cleanup (retry? log-only?).
- [ ] **Metrics/Monitoring**: Add CloudWatch/Prometheus metrics for scheduler runs, cleanup operations, and history sizes.

### 🔮 Future Enhancements (Out of Scope)

- Database persistence for sessions (requires schema design + migration)
- Historical analytics/reporting endpoints
- Configurable cleanup strategies (per-robot retention policies)
- Manual session/window management APIs for operators
