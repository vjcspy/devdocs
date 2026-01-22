# PROD-522: Add BETA_issue Field to Tessa Order Status Report

## Overview

Rosa V. from the operations team has requested that we add the `BETA_issue` data point to the new Tessa Order Status Report. This field already exists in the old "In Use Tessa Report" and provides information about what issue (if any) a robot is experiencing.

## Background

### Old Approach (Legacy)
The old approach uses `InUseTessaReportBuilder` which calculates `BETA_issue` by:
1. Determining if the robot is online or offline
2. Running through complex decision trees to derive an issue enum (`InUseTessaIssue`)
3. Looking up the corresponding issue description from `InUseTessaDerivedStatus` mapping

**Key Files (Old Approach):**
- Schema: `src/schema/InUseTessaReportModel.ts`
- Builder: `src/reports/InUseTessaReportBuilder.ts`
- Utilities: `src/reports/utils/TaasUtils.ts`, `src/reports/utils/RobotAccountUtils.ts`

### New Approach (Current Implementation)
The new approach uses `TessaOrderStatusReportService` which:
1. Queries data directly from Prisma (dashboard + tinybots databases)
2. Provides raw order status timestamps and derived status strings
3. Currently does NOT include `BETA_issue` or related derived fields

**Key Files (New Approach):**
- Schema: `src/graphql/schema/reports/tessaOrderStatusReport.ts`
- Service: `src/graphql/schema/reports/tessaOrderStatusReportService.ts`

## Requirement Analysis

### BETA_issue Values (from old approach)

The `BETA_issue` field can have the following values according to the old approach:

| Issue Description | Enum Value | inUse | Action |
|------------------|-----------|-------|--------|
| Tessa has not been paired with a user yet. | `NOT_PAIRED` | no | Pair with Tessa and plan tasks, so Tessa can be of use. |
| Tessa is online but has no planned tasks. | `NO_PLANNED_TASKS` | no | Instruct the paired user(s) to plan tasks, so Tessa can be of use. |
| New Tessa order: Tessa has just been requested and has not been delivered yet. | `NOT_DELIVERED` | no | null |
| New Tessa order: Tessa has been requested and has not been delivered yet. | `NOT_DELIVERED_THRESHOLD` | no | Contact Tinybots: Delivery is taking longer than expected. |
| New Tessa order: Tessa has been delivered but has not been installed yet. | `NOT_INSTALLED` | no | Install Tessa at the client so Tessa can be of use. |
| Return: Tessa has been set to return. Soon she should be returned, but currently she is still online. | `ONLINE_RETURNING` | yes | null |
| Return: Tessa should be returned, but she is still online. | `ONLINE_RETURNING_THRESHOLD` | yes | Check the status of the return process internally. Tessa should be returning to Tinybots, but the Tessa is still online. |
| Tessa should be online but she isn't. | `OFFLINE_UNKNOWN` | unknown | Check if Tessa is connected to the powergrid and internet. |
| Return: Tessa is returning to Tinybots. | `OFFLINE_RETURNING` | no | null |
| Return: Tessa is returning to Tinybots. | `OFFLINE_RETURNING_THRESHOLD` | no | Check the status of the return process internally: Tessa should be returned, but Tinybots did not receive the Tessa yet. |
| Tessa is already returned to Tinybots. | `RETURNED` | no | null |
| We can not get the right information on the Tessa to determine the status. | `INVALID` | unknown | null |
| null (no issue) | `NO_ISSUE` | yes | null |
| New Tessa order: Tessa has not been installed or delivered yet. | `NOT_DELIVERED_OR_INSTALLED` | no | null |
| New Tessa order: Tessa has not been installed or delivered yet. | `NOT_DELIVERED_OR_INSTALLED_THRESHOLD` | no | Please check if Tessa has arrived yet, and if she has, check the installation status. For questions contact Tinybots. |

## Coverage Analysis

### Data Available in New Approach

The new `TessaOrderStatusReport` currently provides:

**Core Fields:**
- `clientId`, `teamId`, `serial`
- `onlineStatus` (ONLINE/OFFLINE)
- `deviceLastSeen` (timestamp)

**Order Status Fields:**
- `salesOrderStatus` (derived string: ordered, processing, shipped, delivered, activated, etc.)
- `salesOrderReceivedAt` → `taas_order.created_at`
- `salesOrderDeclinedAt` → `taas_order_status.declined_at`
- `salesOrderAcceptedAt` → `taas_order_status.processing_at`
- `salesOrderShippedAt` → `taas_subscription.shipped_at`
- `salesOrderDeliveredAt` → `taas_order_status.delivered_at`
- `salesOrderActivatedAt` → `taas_order_status.activated_at`
- `salesOrderReturnReceivedAt` → `taas_order_status.return_process_started_at`
- `salesOrderReturnAcceptedAt` → `taas_order_status.return_accepted_at`
- `salesOrderReturnRejectedAt` → `taas_order_status.return_rejected_at`
- `salesOrderReturnedAt` → `taas_subscription.end_at`

**Subscription Fields:**
- `subscriptionStartAt` → `taas_subscription.start_at`
- `subscriptionEndAt` → `taas_subscription.end_at`

### Data NOT Available in New Approach

The following data points are used in the old approach but NOT available in the new approach:

1. **`numberOfPlannedTasks`** → Required to detect `NO_PLANNED_TASKS` issue
2. **`primaryUserId` / robotUsers** → Required to detect `NOT_PAIRED` issue
3. **Threshold constants** → Used to determine if issues are outside acceptable timeframes (14 days)

## BETA_issue Mapping Table

| Issue | Covered in New Approach? | Mapping Logic | Missing Data |
|-------|-------------------------|---------------|--------------|
| **Tessa has not been paired with a user yet.** | ❌ NO | Requires `primaryUserId` or robot user data | `primaryUserId`, `robotUsers` |
| **Tessa is online but has no planned tasks.** | ❌ NO | Requires `numberOfPlannedTasks` | `numberOfPlannedTasks` |
| **New Tessa order: Tessa has just been requested and has not been delivered yet.** | ✅ PARTIAL | `onlineStatus == 'OFFLINE' AND salesOrderStatus IN ['ordered', 'processing', 'shipping', 'shipped'] AND salesOrderDeliveredAt == null AND (salesOrderActivatedAt == null OR deviceLastSeen < subscriptionStartAt)` | Threshold logic for "just requested" |
| **New Tessa order: Tessa has been requested and has not been delivered yet.** | ✅ PARTIAL | Same as above + threshold check: `salesOrderShippedAt != null AND now() - salesOrderShippedAt > 14 days` | Threshold constant |
| **New Tessa order: Tessa has been delivered but has not been installed yet.** | ✅ YES | `onlineStatus == 'OFFLINE' AND salesOrderDeliveredAt != null AND salesOrderActivatedAt == null AND (deviceLastSeen == null OR deviceLastSeen < subscriptionStartAt)` | None |
| **Return: Tessa has been set to return. Soon she should be returned, but currently she is still online.** | ✅ PARTIAL | `onlineStatus == 'ONLINE' AND salesOrderReturnReceivedAt != null AND salesOrderReturnedAt == null` | Threshold logic |
| **Return: Tessa should be returned, but she is still online.** | ✅ PARTIAL | Same as above + threshold: `now() - salesOrderReturnAcceptedAt > 14 days` | Threshold constant |
| **Tessa should be online but she isn't.** | ✅ YES | `onlineStatus == 'OFFLINE' AND salesOrderActivatedAt != null AND salesOrderReturnReceivedAt == null AND deviceLastSeen >= subscriptionStartAt` | None |
| **Return: Tessa is returning to Tinybots.** | ✅ PARTIAL | `onlineStatus == 'OFFLINE' AND salesOrderReturnReceivedAt != null AND salesOrderReturnedAt == null` | Threshold logic |
| **Return: Tessa is returning to Tinybots.** (threshold) | ✅ PARTIAL | Same as above + threshold: `salesOrderReturnAcceptedAt != null AND now() - salesOrderReturnAcceptedAt > 14 days` | Threshold constant |
| **Tessa is already returned to Tinybots.** | ✅ YES | `salesOrderReturnedAt != null OR subscriptionEndAt != null` | None |
| **We can not get the right information on the Tessa to determine the status.** | ✅ YES | Catch-all when subscription data is invalid/missing | None |
| **null (no issue)** | ✅ YES | `onlineStatus == 'ONLINE' AND salesOrderActivatedAt != null AND salesOrderReturnReceivedAt == null` (and NOT other issues) | Requires `numberOfPlannedTasks` for complete validation |
| **New Tessa order: Tessa has not been installed or delivered yet.** | ✅ PARTIAL | `taasOrder == null AND subscriptionStartAt != null AND onlineStatus == 'OFFLINE'` | Threshold logic |
| **New Tessa order: Tessa has not been installed or delivered yet.** (threshold) | ✅ PARTIAL | Same as above + `now() - subscriptionStartAt > 14 days` | Threshold constant |

### Summary

- **Fully Covered (no missing data):** 4 issues
- **Partially Covered (threshold logic missing):** 8 issues
- **Not Covered (user/task data missing):** 2 issues

## Implementation Plan

### Option 1: Partial Implementation (Recommended)

Implement BETA_issue for the 12 issues that CAN be derived from available data in the new approach, excluding:
- NOT_PAIRED (requires user data)
- NO_PLANNED_TASKS (requires task data)

**Advantages:**
- No additional database queries needed
- Clean implementation using existing data
- Covers majority of use cases (86% coverage)

**Disadvantages:**
- Missing 2 edge cases related to user pairing and task planning
- May show "NO_ISSUE" for robots that are online but have no tasks/users

### Option 2: Full Implementation (Complex)

Add additional queries to fetch:
1. Robot users (tinybots.robot_user table) → detect pairing
2. Planned tasks count → detect task planning issues

**Advantages:**
- 100% feature parity with old approach
- Complete accuracy in issue detection

**Disadvantages:**
- Additional database queries (performance impact)
- More complex service logic
- Increases coupling with tinybots database schema

### Option 3: Hybrid Implementation

Implement Option 1 (partial) now, add a separate field like `BETA_issueWarning` that indicates when user/task data is unavailable.

**Advantages:**
- Transparent to users about data limitations
- Allows future enhancement without breaking changes
- Maintains performance

**Disadvantages:**
- Adds API complexity
- May confuse users

## Recommended Approach: Option 1 (Partial Implementation)

### Rationale

1. **Performance**: No additional database queries needed
2. **Simplicity**: Uses only existing data from the report
3. **Coverage**: Handles 12 out of 14 issues (86%)
4. **Use Case Fit**: The operations team's primary concerns (delivery, installation, returns) are fully covered
5. **Future-Proof**: Can be enhanced later if user/task data becomes critical

### Missing Issues Handling

For the 2 missing issues (NOT_PAIRED, NO_PLANNED_TASKS):
- Robot will show as `NO_ISSUE` when online and activated
- This is acceptable because:
  - These are edge cases (robot is technically "working" - online and not returning)
  - Operations team can manually verify pairing/tasks if needed
  - The primary report goal is order lifecycle tracking, not task management

## Implementation Steps

### Step 1: Add BETA_issue Field to Schema

**File:** `src/graphql/schema/reports/tessaOrderStatusReport.ts`

```typescript
export const TessaOrderStatusReportRow = objectType({
  name: 'TessaOrderStatusReportRow',
  definition(t) {
    // ... existing fields ...

    t.nullable.string('BETA_issue', {
      description: `What is the issue with this robot (if any issue)? (**In beta - may change at any time**)

Can be one of the following values:

- New Tessa order: Tessa has just been requested and has not been delivered yet.
- New Tessa order: Tessa has been requested and has not been delivered yet.
- New Tessa order: Tessa has been delivered but has not been installed yet.
- Return: Tessa has been set to return. Soon she should be returned, but currently she is still online.
- Return: Tessa should be returned, but she is still online.
- Tessa should be online but she isn't.
- Return: Tessa is returning to Tinybots.
- Tessa is already returned to Tinybots.
- We can not get the right information on the Tessa to determine the status.
- New Tessa order: Tessa has not been installed or delivered yet.

Note: Issues related to user pairing and task planning are not included in this report.`
    })
  }
})
```

### Step 2: Update Service Interface

**File:** `src/graphql/schema/reports/tessaOrderStatusReportService.ts`

```typescript
export interface TessaOrderStatusReportRow {
  // ... existing fields ...
  BETA_issue: string | null
}
```

### Step 3: Implement Issue Derivation Logic

**File:** `src/graphql/schema/reports/tessaOrderStatusReportService.ts`

Add a new private method `deriveBetaIssue()` that implements the decision tree based on available data.

```typescript
private deriveBetaIssue(reportRow: {
  onlineStatus: string
  deviceLastSeen: string | null
  salesOrderStatus: string | null
  salesOrderDeliveredAt: string | null
  salesOrderActivatedAt: string | null
  salesOrderShippedAt: string | null
  salesOrderReturnReceivedAt: string | null
  salesOrderReturnAcceptedAt: string | null
  salesOrderReturnedAt: string | null
  subscriptionStartAt: string | null
  subscriptionEndAt: string | null
}): string | null {
  // Implementation following the mapping table above
}
```

### Step 4: Integrate into Report Building

Update the `buildReport()` method to call `deriveBetaIssue()` for each report row.

### Step 5: Add Unit Tests

**File:** `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`

Add test cases for:
- Each issue scenario that can be derived
- Edge cases (missing data, null values)
- Threshold calculations (14-day logic)

### Step 6: Update Integration Tests

**File:** `test/graphqlIT/reports/tessaOrderStatusReportIT.ts`

Add assertions for `BETA_issue` field in integration tests.

### Step 7: Update Documentation

Update `devdocs/tinybots/wonkers-graphql/tessaOrderStatusReport.md` with:
- New field documentation
- Limitations (missing user/task-related issues)
- Mapping table reference

## Threshold Constants

The old approach uses a 14-day threshold for various time-based checks. The new implementation should:

1. Define a constant: `const THRESHOLD_DAYS = 14`
2. Use moment.js (already available) or native Date for date arithmetic
3. Apply threshold logic for:
   - Delivery delays (shipped > 14 days ago, not delivered)
   - Installation delays (delivered > 14 days ago, not activated)
   - Return delays (return accepted > 14 days ago, not returned)

## Issue Derivation Decision Tree

```
START
├─ Is subscription invalid/missing? → INVALID
├─ Is robot returned (salesOrderReturnedAt != null OR subscriptionEndAt != null)? → RETURNED
├─ Is robot ONLINE?
│  ├─ Is return process started (salesOrderReturnReceivedAt != null)?
│  │  ├─ Is return accepted AND threshold passed (returnAcceptedAt + 14 days < now)? → ONLINE_RETURNING_THRESHOLD
│  │  └─ Else → ONLINE_RETURNING
│  └─ Is robot activated (salesOrderActivatedAt != null)?
│     └─ No other issues → NO_ISSUE
│
└─ Is robot OFFLINE?
   ├─ Is return process started (salesOrderReturnReceivedAt != null)?
   │  ├─ Is return accepted AND threshold passed? → OFFLINE_RETURNING_THRESHOLD
   │  └─ Else → OFFLINE_RETURNING
   │
   ├─ Is robot activated (salesOrderActivatedAt != null)?
   │  └─ Robot should be online but isn't → OFFLINE_UNKNOWN
   │
   └─ Robot never activated - check order lifecycle:
      ├─ Is robot delivered (salesOrderDeliveredAt != null)?
      │  └─ Not installed yet → NOT_INSTALLED
      │
      ├─ Is order available (taasOrder != null)?
      │  ├─ Is shipped AND threshold passed (shippedAt + 14 days < now)? → NOT_DELIVERED_THRESHOLD
      │  └─ Not delivered yet → NOT_DELIVERED
      │
      └─ No order data available:
         ├─ Subscription started AND threshold passed? → NOT_DELIVERED_OR_INSTALLED_THRESHOLD
         └─ Else → NOT_DELIVERED_OR_INSTALLED
```

## Validation Checklist

Before marking this task as complete, verify:

- [ ] BETA_issue field is added to GraphQL schema
- [ ] Service method implements decision tree correctly
- [ ] Threshold logic (14 days) is applied correctly
- [ ] Date comparisons handle null values gracefully
- [ ] Edge cases are handled (missing data, invalid states)
- [ ] Unit tests cover all derivable issue types
- [ ] Integration tests validate end-to-end functionality
- [ ] Documentation is updated with field description and limitations
- [ ] GraphQL schema generates correctly (`npm run codegen`)
- [ ] All existing tests still pass
- [ ] Performance impact is minimal (no additional queries)

## Future Enhancements

If Rosa or the operations team later requires the missing issues (NOT_PAIRED, NO_PLANNED_TASKS):

1. Add queries to fetch:
   - Robot users: `tinybots.robot_user` table
   - Planned tasks: Existing data source or new query

2. Extend `TessaOrderStatusReportService.buildReport()` to include this data

3. Update decision tree to include:
   - Check for primary user (if online and activated)
   - Check for planned tasks (if paired with user)

4. Update documentation and tests

## Notes

- The old approach has complex dependencies on data sources and utilities that are part of the legacy REST-based system
- The new approach aims to be self-contained using only Prisma queries
- Maintaining 100% parity may not be necessary if the primary use cases are covered
- The operations team should be consulted to confirm which issues are most critical

## References

- Old approach schema: `src/schema/InUseTessaReportModel.ts`
- Old approach builder: `src/reports/InUseTessaReportBuilder.ts`
- Old approach utilities: `src/reports/utils/TaasUtils.ts`
- New approach schema: `src/graphql/schema/reports/tessaOrderStatusReport.ts`
- New approach service: `src/graphql/schema/reports/tessaOrderStatusReportService.ts`
- Documentation: `devdocs/tinybots/wonkers-graphql/tessaOrderStatusReport.md`
