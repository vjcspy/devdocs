# TinyBots Release – Sensara Events & Event-Check Jobs (2025-12-01)

## Scope

- Ship Sensara adaptor updates for new Sensara notification types and refreshed event mappings using `tiny-internal-services` v1.25.0.
- Register new Megazord event schemas needed by downstream monitoring and alarm flows.
- Deploy AZI-3 Status Check Jobs (event-check jobs) worker that monitors robot activity gaps and emits alarm events.

## Repositories & PRs

| Repo | PR | Key Changes |
| --- | --- | --- |
| sensara-adaptor | bitbucket.org/tinybots/sensara-adaptor/pull-requests/59 | Handle Sensara notification types `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`; adjust mapping for `ST_ACTIVITY_SHORT_INACTIVITY`; bump `tiny-internal-services` + mocks to v1.25.0 with tests. |
| megazord-events | bitbucket.org/tinybots/megazord-events/pull-requests/48 | Add event schemas `LONGER_IN_BED_LONG`, `LONGER_IN_BED_SHORT`, `SHORT_INACTIVITY`, `NO_TOILET_ACTIVITY_ALARM`; update `schemas/gen.ts`, add `yarn test:only`, refresh `.gitignore`. |
| azi-3-status-check-jobs | bitbucket.org/tinybots/azi-3-status-check-jobs/pull-requests/9 | Introduce monitoring worker (Scheduler + SQS MonitorWorker + WindowExpirationChecker) with RuleTracker/ActionOrchestrator; `AlarmAction` posts `NO_TOILET_ACTIVITY_ALARM` (configurable) via Event Service; extensive unit tests added. |

## Validation Checklist

- Megazord: hit `GET /internal/v1/events/robots/{id}/incomings` after posting sample `LONGER_IN_BED_*`, `SHORT_INACTIVITY`, and `NO_TOILET_ACTIVITY_ALARM` payloads; confirm rows in `incoming_event` and `event_schema` lookup succeeds.
- Sensara adaptor: send synthetic Sensara notifications for `ST_SLEEPING_AWAKE_DELAYED` and `ST_SLEEPING_AWAKE_LARGE_DELAY`; confirm outgoing TinyBots events are produced and no mapping errors in logs.
- Event-check jobs: feed SNS-wrapped SQS messages for a monitored robot, allow window expiry, and confirm `NO_TOILET_ACTIVITY_ALARM` (or configured event) is emitted to Megazord and visible in the outgoing/status queue.
- Smoke health endpoints: sensara-adaptor `/health`, megazord-events `/internal/v1/status` (if exposed), azi-3-status-check-jobs `/health`.
