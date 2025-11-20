# Architecture Decision: Toilet Activity Monitoring Implementation

**Document Reference**: PROD594  
**Date**: November 19, 2025  
**Status**: Proposal for Stakeholder Review

---

## Executive Summary

This document analyzes two implementation approaches for the toilet activity monitoring feature and provides a recommendation based on architectural fit, risk assessment, and delivery timeline. The requirement is to monitor a specific resident's toilet activity during configured time windows and generate alerts when activity gaps exceed two hours.

**Recommendation**: Implement as a new dedicated service (`azi-3-status-jobs`) rather than extending the existing `azi-3-status-check` repository.

---

## Requirements Overview

### Functional Requirements

The system must:

1. **Initiate monitoring** at a configured start time (Time A) for a specific resident/robot
2. **Subscribe to TOILET_ACTIVITY events** through the Megazord Events service
3. **Monitor in 2-hour rolling windows**:
   - When activity is detected → reset the window to start from the event timestamp
   - When no activity for 2 hours → emit `NO_TOILET_ACTIVITY_ALARM` event and continue monitoring
4. **Terminate monitoring** at configured end time (Time B)
5. **Persist all monitoring state** in database for audit trail and script integration

### Non-Functional Requirements

- **Speed of delivery** prioritized over perfect generalization
- **Low risk** to existing production services
- **Experiment-friendly** architecture allowing rapid iteration
- **Scalable** to additional residents if successful

---

## Architecture Options

### Option 1: Create New Dedicated Service (azi-3-status-jobs)

#### Description

Build a standalone Node.js/TypeScript service focused exclusively on time-based monitoring jobs. This service would:

- Run scheduled cron jobs to initiate monitoring at Time A
- Manage Megazord event subscriptions independently
- Implement rolling window logic specific to activity gap detection
- Store monitoring state in shared `status_check*` database tables
- Emit alarm events back through Megazord for downstream consumption

#### Technical Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    azi-3-status-jobs                         │
│                                                              │
│  ┌──────────────┐      ┌─────────────────────────────┐     │
│  │ Daily Cron   │─────▶│ MonitorCoordinator          │     │
│  │ (at Time A)  │      │ - Create subscriptions      │     │
│  └──────────────┘      │ - Track 2-hour windows      │     │
│                        │ - Emit alarms               │     │
│  ┌──────────────┐      │ - Clean up at Time B        │     │
│  │ SQS Consumer │─────▶│                             │     │
│  │ (statusQueue)│      └─────────────────────────────┘     │
│  └──────────────┘               │           │               │
│                                 │           │               │
└─────────────────────────────────┼───────────┼───────────────┘
                                  │           │
                    ┌─────────────▼───────────▼──────────┐
                    │      Megazord Events Service       │
                    │  - Event subscriptions             │
                    │  - TOILET_ACTIVITY events          │
                    │  - NO_TOILET_ACTIVITY_ALARM events │
                    └────────────────────────────────────┘
```

#### Advantages

| Category | Benefit | Impact |
|----------|---------|--------|
| **Architectural Clarity** | Single responsibility: proactive monitoring jobs | High - Clear purpose and boundaries |
| **Development Speed** | No legacy code constraints; clean slate implementation | High - Faster to market |
| **Risk Management** | Zero impact on existing robot status-check workflows | Critical - Protects production |
| **Deployment Independence** | Can iterate, scale, and redeploy without coordination | High - Faster feedback loops |
| **Experimental Flexibility** | Easy to modify or deprecate if requirements change | High - Aligns with experiment nature |

#### Disadvantages

| Category | Concern | Mitigation |
|----------|---------|------------|
| **Code Duplication** | Reimplements Megazord subscription patterns | Acceptable - limited scope, can extract to shared library later |
| **Infrastructure Overhead** | Additional deployment, monitoring, CI/CD | Moderate - standard service setup process |
| **Repository Management** | One more service to maintain long-term | Low - if successful, value justifies maintenance |

#### Implementation Effort

- **Estimated Timeline**: 2-3 weeks
- **Complexity**: Medium
- **Risk Level**: Low

---

### Option 2: Extend Existing azi-3-status-check Service

#### Overview

Enhance the existing `azi-3-status-check` service to support both robot-initiated status checks and system-initiated monitoring jobs.

#### Current azi-3-status-check Architecture

The service currently handles:

- **Robot-initiated checks**: Robots POST to `/v1/status-check/check` when executing Micro-Managers scripts
- **Template-based evaluation**: YAML templates define past/future event dependencies
- **One-time validation**: Checks complete after evaluating fixed time windows
- **SQS integration**: Consumes events, notifies robots when checks complete
- **Scheduler**: Tears down expired checks via `StatusCheckTeardownSchedulerService`

#### Required Modifications

To support toilet monitoring, would need:

1. **New initiation path**: Background cron job (not robot API call)
2. **Template system changes**: Support rolling windows instead of fixed windows
3. **Additional scheduler**: Window reset logic (separate from teardown scheduler)
4. **Lifecycle modifications**: Continuous monitoring vs. one-time evaluation
5. **API segregation**: Separate endpoints for background jobs vs. robot requests

#### Benefits

| Category | Benefit | Impact |
|----------|---------|--------|
| **Infrastructure Reuse** | Leverages existing subscription/SQS/DB patterns | Moderate - saves some development time |
| **Data Centralization** | All monitoring in same database schema | Moderate - unified reporting potential |
| **Operational Consistency** | One service for all status/monitoring needs | Low - mixing concerns complicates operations |

#### Drawbacks

| Category | Concern | Impact |
|----------|---------|--------|
| **Architectural Mismatch** | Pull model (robot-initiated) vs. Push model (time-initiated) | **Critical** - Fundamental design conflict |
| **Template System Limitations** | Fixed windows vs. rolling windows | **High** - Significant refactoring required |
| **Regression Risk** | Changes could impact production robot workflows | **Critical** - Requires extensive testing |
| **Complexity Growth** | Service becomes multi-purpose and harder to reason about | High - Violates single responsibility |
| **Development Speed** | Must carefully navigate existing code and tests | High - Slower than greenfield |

#### Effort Estimate

- **Estimated Timeline**: 4-6 weeks
- **Complexity**: High
- **Risk Level**: High

---

## Comparative Analysis

### Architectural Fit

| Criterion | New Service | Extend Existing | Winner |
|-----------|-------------|-----------------|--------|
| **Separation of Concerns** | Clean separation: proactive monitoring | Mixed: status checks + monitoring | ✅ New Service |
| **Trigger Model Alignment** | Native time-based initiation | Retrofitted onto robot-initiated | ✅ New Service |
| **Window Management** | Purpose-built rolling windows | Adapted from fixed windows | ✅ New Service |
| **Script Integration** | Independent of Micro-Managers | Tightly coupled to scripts | ✅ New Service |

### Risk Assessment

| Risk Category | New Service | Extend Existing |
|---------------|-------------|-----------------|
| **Production Impact** | ✅ None - isolated | ⚠️ High - shared codebase |
| **Regression Testing** | ✅ Minimal - new endpoints only | ❌ Extensive - all status checks |
| **Rollback Complexity** | ✅ Simple - independent deploy | ⚠️ Complex - entangled features |
| **Experiment Failure** | ✅ Easy to deprecate | ❌ Difficult to extract |

### Delivery Timeline

| Phase | New Service | Extend Existing |
|-------|-------------|-----------------|
| **Design** | 2-3 days | 4-5 days (impact analysis) |
| **Implementation** | 8-10 days | 15-20 days |
| **Testing** | 3-4 days | 8-10 days (regression suite) |
| **Total** | **2-3 weeks** | **4-6 weeks** |

---

## Decision Rationale

### Primary Factors

1. **Trigger Model Incompatibility**

   The existing `azi-3-status-check` service is fundamentally designed for **robot-initiated, script-driven checks**:

   ```text
   Robot Script → API Call → Template Lookup → Event Evaluation → Robot Notification
   ```

   The toilet monitoring requirement needs **time-initiated, autonomous monitoring**:

   ```text
   Cron Schedule → Subscription Creation → Rolling Window Monitoring → Alarm Emission
   ```

   These are different interaction patterns that don't naturally compose.

2. **Window Semantics Mismatch**

   - **Status checks**: Fixed past/future windows defined at creation time
   - **Toilet monitoring**: Dynamic rolling windows that reset on each event

   Forcing rolling semantics into the template system would add complexity without providing reusable value for status checks.

3. **Risk vs. Speed Trade-off**

   The requirement explicitly prioritizes **speed** over perfect generalization. Given this is an **experiment for one user**:
   - Isolated implementation allows rapid iteration without production risk
   - If successful, can be generalized or merged later with full context
   - If unsuccessful, can be deprecated cleanly

4. **Separation of Concerns**

   From a domain modeling perspective:
   - **Status Checks**: *"Did the required events occur during the expected timeframe?"* (validation)
   - **Activity Monitoring**: *"Is the resident maintaining expected activity patterns?"* (surveillance)

   These are distinct problem domains that happen to share event infrastructure.

### Secondary Factors

- **Team Velocity**: Greenfield development is faster than careful refactoring
- **Deployment Flexibility**: Independent service can be scaled/updated without coordination
- **Future Optionality**: Keeps door open for additional monitoring job types
- **Operational Clarity**: Distinct services have clearer alert/logging boundaries

---

## Recommended Approach

### Decision: **Create New Service (azi-3-status-jobs)**

This recommendation is based on:

✅ **Lower risk** to production systems  
✅ **Faster delivery** timeline (2-3 weeks vs. 4-6 weeks)  
✅ **Better architectural fit** for time-based monitoring  
✅ **Easier to iterate** on experiment requirements  
✅ **Cleaner separation** of concerns

### Implementation Strategy

#### Phase 1: Shared Infrastructure Updates (Week 1)

- Add `NO_TOILET_ACTIVITY_ALARM` to Megazord event schemas
- Update `tiny-internal-services` with new event constant
- Seed `status_check_template` and description for toilet monitoring

#### Phase 2: Core Service Development (Week 1-2)

- Scaffold `azi-3-status-jobs` repository
- Implement Megazord subscription client (reuse patterns from azi-3-status-check)
- Build rolling window coordinator
- Integrate SQS consumer for TOILET_ACTIVITY events

#### Phase 3: Scheduling & Persistence (Week 2-3)

- Implement daily cron for monitoring initiation (Time A)
- Build window evaluation and alarm emission logic
- Persist state to `status_check*` tables
- Add subscription cleanup at Time B

#### Phase 4: Testing & Deployment (Week 3)

- Unit tests for window logic and edge cases
- Integration tests with mocked Megazord/SQS
- Configuration validation
- Production deployment

### Code Reuse Strategy

While creating a new service, we will **reuse proven patterns** from `azi-3-status-check`:

- Megazord event subscription lifecycle
- SQS consumer setup and error handling
- Database repository patterns for `status_check*` tables
- Timezone-aware scheduling logic
- Logging and observability patterns

This provides **80% of the benefit** of code reuse while maintaining **100% architectural clarity**.

### Future Consolidation Path

If the experiment succeeds and becomes permanent:

1. **Option A**: Extract shared monitoring patterns into `tiny-internal-services` library
2. **Option B**: Merge both services under unified "monitoring" service with clear module boundaries
3. **Option C**: Keep separate - if use cases remain distinct

Decision deferred until we have production usage data and clearer requirements for generalization.

---

## Risk Mitigation

| Risk | Mitigation Strategy |
|------|---------------------|
| **Duplicate Megazord integration code** | Create shared `MegazordClient` library in tiny-internal-services after validating patterns |
| **Additional deployment infrastructure** | Leverage existing CI/CD templates; automate from day 1 |
| **Database schema conflicts** | Coordinate with azi-3-status-check team on `status_check*` table changes |
| **Alert fatigue** | Start with single pilot user; tune thresholds before expanding |
| **Timezone handling complexity** | Reuse `Clock` abstraction pattern from azi-3-status-check |

---

## Success Criteria

### Delivery Metrics

- ✅ Service deployed to production within 3 weeks
- ✅ Zero impact on existing status-check workflows
- ✅ All monitoring state persisted for audit

### Functional Metrics

- ✅ Correctly detects 2-hour activity gaps
- ✅ Alarms visible in Megazord event logs
- ✅ Rolling windows reset properly after activity
- ✅ Clean subscription lifecycle (no leaked subscriptions)

### Operational Metrics

- ✅ Monitoring runs reliably during configured windows
- ✅ Graceful handling of service restarts
- ✅ Clear logs for debugging window transitions
- ✅ No false alarms or missed alarms

---

## Open Questions for Stakeholder Discussion

1. **Pilot Configuration**: Can we confirm the specific `robotId`, timezone, and Time A/B values?
2. **Alarm Severity**: What `level` should `NO_TOILET_ACTIVITY_ALARM` have for proper dashboard/trigger routing?
3. **Escalation**: Should repeated alarms (multiple 2-hour gaps) have different severity or notifications?
4. **Monitoring Scope**: Any other residents/robots planned for this monitoring in near term?
5. **Success Definition**: What metrics will determine if the experiment should become permanent?

---

## Appendices

### Appendix A: Technical Dependencies

- **Megazord Events**: Event subscription and `NO_TOILET_ACTIVITY_ALARM` posting
- **Tiny-internal-services**: Shared DTOs and event constants
- **Typ-e**: Database schema for `status_check*` tables
- **AWS SQS**: Status queue for event consumption
- **Kong**: Authentication for internal API endpoints (if needed)

### Appendix B: Database Schema Usage

Both approaches would use the existing `status_check` tables:

- `status_check`: Main monitoring record
- `status_check_poller`: Window tracking (since/until timestamps)
- `status_check_record`: Event history and alarm emissions
- `status_check_template`: Toilet monitoring template definition
- `status_check_description`: Human-readable monitoring description

This ensures unified reporting regardless of implementation approach.

---

**Document prepared by**: AI Engineering Assistant  
**Review requested from**: [Stakeholder Names]  
**Next steps**: Schedule walkthrough meeting to discuss recommendation and address open questions
