> **Branch:** feature/PROD-594 (user-approved deviation from develop)  
> **Last Commit:** 4310f0c (Updated from N/A – previous hash not recorded)  
> **Last Updated:** 2025-11-22

# azi-3-status-check-jobs

## TL;DR
- Stateless Node.js worker that schedules per-robot, per-rule monitoring windows and runs actions (currently alarms) when gaps in events are detected.  
- Uses cron + SQS consumer + Megazord Event Service; all configuration is provided via environment (config files / `TOILET_MONITORING_CONFIG`).  
- Supports multiple monitoring rules per robot, each with its own event type, window duration, and action list; state is in-memory only.

## Recent Changes Log
- Overview rebuilt with metadata; prior version lacked commit anchor, so this serves as the new baseline.  
- Code now models **monitoring rules** with dynamic `eventType`, rule-scoped windows, and pluggable actions (AlarmAction via ActionFactory).  
- Scheduling runs per rule per robot; MonitorWorker routes SQS events by `eventType`, enabling flexible event monitoring beyond toilet activity.

## Repo Purpose & Bounded Context
- Background job service for AZI-3 status checks: creates daily monitoring sessions for configured robots and raises actions (alarms) when expected events are absent within configured rolling windows.  
- Integrates with Megazord Event Service for subscriptions and alarm events; consumes robot telemetry/events from an SQS status queue.  
- Intended to run as a single stateless process (in-memory sessions); persistence/HA is future work.

## Project Structure
- `src/cmd/main.ts`: TinyAppUnauthenticated bootstrap, DI container wiring, middleware, health/debug endpoints, lifecycle start/stop.  
- `src/config/types.ts`: class-validator DTOs for app/log/SQS/megazord/statusQueue and `monitoring` (robots → monitoringRules with daily windows and action configs).  
- `src/constants/index.ts`: Awilix token names for DI.  
- `src/domain/monitoring/*`: `MonitoringRule`, `MonitoringSession`, `MonitoringWindow` domain models with JSON (de)serializers.  
- `src/domain/action/*`: Action abstraction (`ActionExecutor`), concrete `AlarmAction`, and `ActionFactory` registry.  
- `src/infrastructure/*`: `Clock` (tz-aware), `MegazordEventClient` (wraps tiny-internal-services EventService), `SubscriptionManager`.  
- `src/services/*`: `RuleTracker`, `MonitoringScheduler`, `ActionOrchestrator`, legacy `AlarmEmitter` (unused currently).  
- `src/jobs/*`: `MonitorWorker` (SQS consumer), `WindowExpirationChecker` (cron every minute), `SessionCleanupJob` (hourly placeholder).  
- `config/*.json`: default/dev configs; `custom-environment-variables.json` maps to env keys (`TOILET_MONITORING_CONFIG`, AWS, Megazord).  
- `test/`: empty scaffold (no automated tests yet).  
- `dist/`: build output; keep generated JS/typings out of source edits.

## Controllers & Public Surface
- `GET /health`: returns `status: ok` plus RuleTracker stats (total sessions, active sessions, rule count, subscription index).  
- `GET /internal/v1/monitoring/sessions`: debug endpoint dumping active sessions (from in-memory state).  
- No authenticated routes; service is intended to run in trusted network; error handling via tiny-backend-tools `errorMiddleware`.

## Core Services & Logic
### Scheduling & Session Lifecycle
- `MonitoringScheduler.scheduleAllRobots()` iterates enabled robots and their monitoringRules, scheduling per-timezone daily sessions. Handles late starts (immediate start if within window) and re-schedules itself at local midnight.  
- `initializeSessionForRule` creates a `MonitoringSession`, provisions Megazord subscription via `SubscriptionManager`, seeds first `MonitoringWindow`, and indexes subscription → session in `RuleTracker`.  
- `RuleTracker` keeps session maps by sessionId, robotId+ruleId, and subscriptionId; deduplicates events; completes sessions when end time reached or on failure; future `flush/restore` stubs for persistence.

### Event Ingest & Window Management
- `MonitorWorker` (IAsyncModule) polls SQS using tiny-backend-tools `ContextSQS`; parses SNS-wrapped messages, derives `eventType` (defaults to `TOILET_ACTIVITY`), matches to robot rule, and forwards to `RuleTracker.handleActivity`.  
- `RuleTracker.handleActivity` validates event type, window bounds, deduplicates by eventId, resets rolling window duration per rule, and completes session when window passes endTime.

### Gap Detection & Actions
- `WindowExpirationChecker` cron (*/1 * * * *) finds expired windows, executes actions via `ActionOrchestrator`, advances window, and completes session when beyond endTime.  
- `ActionOrchestrator` builds `ActionContext` and runs all rule-configured actions with validation; fails-fast if any action rejects.  
- `ActionFactory` registers available executors; currently only `AlarmAction`, which posts `NO_TOILET_ACTIVITY_ALARM` (or configured event) to Megazord with per-session/window deduping.  
- `AlarmEmitter` remains as an older direct-alarm helper; not wired into the current flow.

### Configuration Model
- `monitoring.robots[]` each define `robotId`, `timezone`, `enabled`, and `monitoringRules[]` with `ruleId`, `eventType`, `windowDurationMinutes`, `windowCheckIntervalMinutes`, `dailyWindow{startTime,endTime}`, `actions[]` (type + config).  
- Env mapping: `TOILET_MONITORING_CONFIG` can supply the entire `monitoring` object; AWS/Megazord endpoints map via `custom-environment-variables.json`.

## External Dependencies & Cross-Service Contracts
- **Megazord Event Service** (`tiny-internal-services` `EventService`): create/delete subscriptions and post alarm events.  
- **SQS Status Queue**: polled via tiny-backend-tools `ContextSQS`; expects SNS-wrapped messages with `subscriptionId`, `sourceEvent.id/timestamp/robot_id/eventType`.  
- **Tiny Backend Tools**: HTTP app shell, cron scheduler, DI utilities, context middleware, and SQS abstraction.  
- **Timezones/Clock**: `luxon` for zone-aware scheduling and window math.  
- Environment-only state: no DB; subscriptions and session state are in-memory and lost on restart.

## Testing & Quality Gates
- `yarn test` runs `nyc mocha` (tsx) then enforces coverage thresholds: statements/functions/lines 94%, branches 70%, followed by `yarn lint`.  
- Dev dependencies include `tiny-testing`, `supertest`, `nock`, `ts-mockito`, but `test/` currently empty—add coverage for scheduling, SQS handling, and action execution.  
- `yarn build` runs lint then `tsc --project tsconfig.prod.json`; ensure configs match deployed env (SQS queueUrl injected from `statusQueue.address` during bootstrap).
