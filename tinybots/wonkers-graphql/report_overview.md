# Robot Usage Report - Implementation Overview

## Task Context

### Project Architecture

- **Framework**: Node.js + TypeScript with Apollo GraphQL Server
- **Schema Approach**: Using Nexus for type-safe GraphQL schema generation
- **Database Access**: Prisma ORM with two separate database connections
- **Authentication**: Kong-based authentication with permission checking
- **Legacy vs New**: Migrating from REST-based datasources to direct Prisma queries

### Databases

1. **Dashboard DB** (`wonkers-db`): Contains TaaS orders, subscriptions, robots, and organization data
2. **Tinybots DB** (`typ-e`): Contains robot accounts, users, scripts, schedules, and online status

### Report Objective

Create a comprehensive GraphQL report that aggregates robot usage data across both databases, filtered by time period,
team, user, or organization.

### Key Requirements

- **Required filter**: Time period (ISO 8601 format: `2023-01-01T00:00:00.000Z`)
- **Optional filters**: appUserId, teamId, relationId
- **Multiple permissions required**: 11 different permission checks
- **Cross-database joins**: Data must be aggregated from both databases

---

## Database Schema Analysis

### Dashboard Database (wonkers-db) Tables

| Database      | Table                | Column             | Type            | Relation/Purpose          | Notes                 |
|---------------|----------------------|--------------------|-----------------|---------------------------|-----------------------|
| **dashboard** | `taas_order`         | `id`               | Int (PK)        | Unique order identifier   | Primary key           |
|               |                      | `relation_id`      | Int (FK)        | → `dashboard_relation.id` | Organization/relation |
|               |                      | `client_id`        | String          | Client identifier         | Filter field          |
|               |                      | `team_id`          | String          | Team identifier           | Filter field          |
|               |                      | `taas_id`          | Int (FK)        | → `taas_subscription.id`  | Links to subscription |
|               |                      | `created_at`       | DateTime        | Order creation time       | For time filtering    |
|               |                      | `updated_at`       | DateTime        | Last update time          |                       |
|               |                      | `integration_id`   | Int (FK)        | → `integration.id`        |                       |
|               |                      | `hardware_type_id` | Int (FK)        | → `hardware_type.id`      |                       |
| **dashboard** | `taas_order_status`  | `taas_order_id`    | Int (PK, FK)    | → `taas_order.id`         | One-to-one relation   |
|               |                      | `ordered_at`       | DateTime        | Status: ordered           | Status tracking       |
|               |                      | `processing_at`    | DateTime        | Status: processing        |                       |
|               |                      | `delivered_at`     | DateTime        | Status: delivered         |                       |
|               |                      | `activated_at`     | DateTime        | Status: activated         |                       |
|               |                      | `declined_at`      | DateTime        | Status: declined          |                       |
|               |                      | `created_at`       | DateTime        | Record creation           |                       |
|               |                      | `updated_at`       | DateTime        | Last update               |                       |
| **dashboard** | `taas_subscription`  | `id`               | Int (PK)        | Unique subscription ID    | Primary key           |
|               |                      | `serial_id`        | BigInt (FK)     | → `dashboard_robot.id`    | Links to robot        |
|               |                      | `relation_id`      | Int (FK)        | → `dashboard_relation.id` | Organization          |
|               |                      | `client_id`        | String          | Client identifier         |                       |
|               |                      | `team_id`          | String          | Team identifier           | Filter field          |
|               |                      | `start_at`         | DateTime        | Subscription start        | Key field for report  |
|               |                      | `end_at`           | DateTime        | Subscription end          |                       |
|               |                      | `shipped_at`       | DateTime        | Robot shipped date        |                       |
|               |                      | `created_at`       | DateTime        | Record creation           |                       |
|               |                      | `updated_at`       | DateTime        | Last update               |                       |
| **dashboard** | `dashboard_robot`    | `id`               | BigInt (PK)     | Unique robot ID           | Primary key           |
|               |                      | `serial`           | String (unique) | Robot serial number       | Join key to tinybots  |
|               |                      | `relation_id`      | Int (FK)        | → `dashboard_relation.id` | Organization          |
|               |                      | `taas`             | Boolean         | Is TaaS robot?            | Filter field          |
|               |                      | `created_at`       | DateTime        | Record creation           |                       |
|               |                      | `updated_at`       | DateTime        | Last update               |                       |
|               |                      | `deactivated_at`   | DateTime        | Deactivation date         | Default: 9999-12-31   |
| **dashboard** | `dashboard_relation` | `id`               | Int (PK)        | Unique organization ID    | Primary key           |
|               |                      | `name`             | String          | Organization name         | Display field         |
|               |                      | `relation_number`  | Int             | Organization number       |                       |
|               |                      | `teamleader_id`    | String (unique) | External ID               |                       |
|               |                      | `created_at`       | DateTime        | Record creation           |                       |
|               |                      | `updated_at`       | DateTime        | Last update               |                       |
|               |                      | `deleted_at`       | DateTime        | Soft delete               |                       |

### Tinybots Database (typ-e) Tables

| Database     | Table                         | Column                | Type            | Relation/Purpose            | Notes                            |
|--------------|-------------------------------|-----------------------|-----------------|-----------------------------|----------------------------------|
| **tinybots** | `robot_account`               | `id`                  | Int (PK)        | Robot account ID            | Primary key, robotId             |
|              |                               | `username`            | String (unique) | Robot serial                | Join to `dashboard_robot.serial` |
|              |                               | `account_status_id`   | Int (FK)        | → `robot_account_status.id` | Status                           |
|              |                               | `last_active_at`      | DateTime        | Last activity               |                                  |
|              |                               | `created_at`          | DateTime        | Account creation            |                                  |
|              |                               | `updated_at`          | DateTime        | Last update                 |                                  |
| **tinybots** | `user_robot`                  | `id`                  | Int (PK)        | User-robot link ID          | Primary key                      |
|              |                               | `robot_id`            | Int (FK)        | → `robot_account.id`        | Robot reference                  |
|              |                               | `user_id`             | Int (FK)        | → `user_account.id`         | User reference                   |
|              |                               | `role_id`             | Int (FK)        | → `user_robot_role.id`      | 1 = Primary user                 |
|              |                               | `created_at`          | DateTime        | Link creation               |                                  |
|              |                               | `deleted_at`          | DateTime        | Link deletion               | Soft delete                      |
| **tinybots** | `user_account`                | `id`                  | Int (PK)        | User account ID             | Primary key, userId              |
|              |                               | `email`               | String (unique) | User email                  | Primary user email               |
|              |                               | `account_status_id`   | Int (FK)        | → `user_account_status.id`  | Status                           |
|              |                               | `last_active_at`      | DateTime        | Last activity               | For filtering                    |
|              |                               | `created_at`          | DateTime        | Account creation            | For filtering                    |
|              |                               | `updated_at`          | DateTime        | Last update                 |                                  |
| **tinybots** | `user_profile`                | `user_id`             | Int (PK, FK)    | → `user_account.id`         | One-to-one                       |
|              |                               | `first_name`          | String          | User first name             | Display field                    |
|              |                               | `last_name`           | String          | User last name              | Display field                    |
|              |                               | `phone_number`        | BigInt          | User phone                  | Optional                         |
|              |                               | `created_at`          | DateTime        | Profile creation            |                                  |
|              |                               | `updated_at`          | DateTime        | Last update                 |                                  |
| **tinybots** | `robot_online_status`         | `robot_id`            | Int (PK, FK)    | → `robot_account.id`        | One-to-one, current status       |
|              |                               | `status_id`           | Int             | Online status               | 1 = Online                       |
|              |                               | `updated_at`          | DateTime        | Last status update          |                                  |
| **tinybots** | `robot_online_status_history` | `robot_id`            | Int             | → `robot_account.id`        | Historical records               |
|              |                               | `status_id`           | Int             | Status at timestamp         | 1 = Online                       |
|              |                               | `timestamp`           | DateTime        | Status change time          | For time-based queries           |
| **tinybots** | `script_execution`            | `id`                  | BigInt (PK)     | Execution ID                | Primary key                      |
|              |                               | `script_reference_id` | BigInt (FK)     | → `script_reference.id`     | Script reference                 |
|              |                               | `script_version_id`   | BigInt (FK)     | → `script_version.id`       | Script version                   |
|              |                               | `schedule_id`         | BigInt (FK)     | → `task_schedule.id`        | Schedule                         |
|              |                               | `planned`             | DateTime        | Planned execution           |                                  |
|              |                               | `created_at`          | DateTime        | Actual execution            | **Filter by this**               |
| **tinybots** | `script_reference`            | `id`                  | BigInt (PK)     | Script reference ID         | Primary key                      |
|              |                               | `robot_id`            | Int (FK)        | → `robot_account.id`        | Robot owner                      |
|              |                               | `created_at`          | DateTime        | Creation time               |                                  |
|              |                               | `archived_at`         | DateTime        | Archive time                | Soft delete                      |
| **tinybots** | `script_version`              | `id`                  | BigInt (PK)     | Script version ID           | Primary key                      |
|              |                               | `script_reference_id` | BigInt (FK)     | → `script_reference.id`     | Reference                        |
|              |                               | `script_category_id`  | Int (FK)        | → `script_category.id`      | Category                         |
|              |                               | `script_name`         | String          | Script name                 | Display field                    |
|              |                               | `created_at`          | DateTime        | Version creation            | **Latest = last edit**           |
| **tinybots** | `script_category`             | `id`                  | Int (PK)        | Category ID                 | Primary key                      |
|              |                               | `name`                | String (unique) | Category name               | Group by this                    |
|              |                               | `created_at`          | DateTime        | Category creation           |                                  |
| **tinybots** | `task_schedule`               | `id`                  | BigInt (PK)     | Schedule ID                 | Primary key                      |
|              |                               | `start_at`            | DateTime        | Schedule start              | Filter field                     |
|              |                               | `end_at`              | DateTime        | Schedule end                | Can be NULL                      |
|              |                               | `created_at`          | DateTime        | Record creation             |                                  |
| **tinybots** | `robot_schema`                | `id`                  | Int (PK)        | Schema ID                   | Primary key                      |
|              |                               | `robot_id`            | Int (FK)        | → `robot_account.id`        | Robot owner                      |
|              |                               | `schedule_id`         | BigInt (FK)     | → `task_schedule.id`        | Schedule link                    |
|              |                               | `title`               | String          | Task title                  | Display field                    |
|              |                               | `created_at`          | DateTime        | Creation time               |                                  |
|              |                               | `updated_at`          | DateTime        | Last update                 |                                  |
| **tinybots** | `robot_profile`               | `robot_id`            | Int (PK, FK)    | → `robot_account.id`        | One-to-one                       |
|              |                               | `endUserName`         | String          | Robot alias                 | Display field                    |
|              |                               | `created_at`          | DateTime        | Profile creation            |                                  |
|              |                               | `updated_at`          | DateTime        | Last update                 |                                  |

---

## Data Flow & Join Strategy

### Primary Join Path

```
dashboard_relation (relationId filter)
    ↓
taas_order (teamId, clientId filter)
    ↓
taas_order_status (get status)
    ↓
taas_subscription (get start_at, taasId)
    ↓
dashboard_robot (get serial)
    ↓ (JOIN via serial = username)
robot_account (robotId)
    ↓
[Multiple branches from robot_account]
```

### Branch 1: User Information

```
robot_account
    ↓
user_robot (role_id = 1 for primary)
    ↓
user_account (userId, email)
    ↓
user_profile (first_name, last_name)
```

### Branch 2: Online Status

```
robot_account
    ↓
robot_online_status (current status)
    ↓
robot_online_status_history (within time period)
```

### Branch 3: Script Executions

```
robot_account
    ↓
script_reference
    ↓
script_execution (filter by created_at within period)
    ↓
script_version (get category_id)
    ↓
script_category (group by name)
```

### Branch 4: Last Script Edit

```
robot_account
    ↓
script_reference
    ↓
script_version (MAX(created_at))
```

### Branch 5: Planned Tasks

```
robot_account
    ↓
robot_schema
    ↓
task_schedule (COUNT where start_at/end_at overlaps period)
```

### Branch 6: Robot Alias

```
robot_account
    ↓
robot_profile (endUserName)
```

---

## Key Business Logic

### Online Status Definition

**Clarification needed**:

- Option A: Robot was online at ANY point during the time period → Query `robot_online_status_history`
- Option B: Robot is CURRENTLY online → Query `robot_online_status`
- **Recommendation**: Use Option A (historical) for report accuracy

### Primary User

- Filter: `user_robot.role_id = 1`
- Should be exactly ONE primary user per robot
- Contact email = `user_account.email` where `role_id = 1`

### Script Categories

- Group `script_execution` by `script_category.name`
- Count executions per category within time period
- Only count scripts that were actually executed (created_at within period)

### Last Script Edit Date

- Find `MAX(script_version.created_at)` for all scripts belonging to robot
- Via `script_reference.robot_id`

### Planned Tasks Count

- Count `task_schedule` where:
    - `start_at >= timePeriodStart` OR `start_at IS NULL`
    - `end_at <= timePeriodEnd` OR `end_at IS NULL`
    - Connected via `robot_schema.robot_id`

---

## Filter Logic

### Time Period (Required)

- Format: ISO 8601 (`2023-01-01T00:00:00.000Z`)
- Apply to:
    - `script_execution.created_at`
    - `robot_online_status_history.timestamp`
    - `task_schedule.start_at` and `end_at`
    - Optional: `user_account.last_active_at`, `user_account.created_at`

### Team ID (Optional)

- Filter: `taas_order.team_id = ?`
- Alternative to `appUserId`
- **Clarification needed**: Can both filters be used together?

### App User ID (Optional)

- Filter: `taas_order.client_id = ?` (assuming this is app user ID)
- **Clarification needed**: Exact mapping of appUserId field

### Relation ID (Optional)

- Filter: `dashboard_relation.id = ?`
- Primary organization filter

---

## Permissions Required

| Permission                     | Purpose                | Database  | Tables Accessed                                         |
|--------------------------------|------------------------|-----------|---------------------------------------------------------|
| `TAAS_ORDER_READ_ALL`          | Read TaaS orders       | dashboard | `taas_order`, `taas_order_status`                       |
| `TAAS_READ_ALL`                | Read subscriptions     | dashboard | `taas_subscription`                                     |
| `DASHBOARD_ROBOT_READ_ALL`     | Read robot data        | dashboard | `dashboard_robot`                                       |
| `ROBOT_ACCOUNT_READ_ALL`       | Read robot accounts    | tinybots  | `robot_account`                                         |
| `ROBOT_USER_READ_ALL`          | Read robot-user links  | tinybots  | `user_robot`                                            |
| `USER_ACCOUNTS_READ_ALL`       | Read user accounts     | tinybots  | `user_account`, `user_profile`                          |
| `ROBOT_ONLINE_STATUS_READ_ALL` | Read online status     | tinybots  | `robot_online_status`, `robot_online_status_history`    |
| `SCRIPT_EXECUTION_READ_ALL`    | Read script executions | tinybots  | `script_execution`                                      |
| `SCRIPT_READ_ALL`              | Read script versions   | tinybots  | `script_version`, `script_category`, `script_reference` |
| `SCHEDULE_READ_ALL`            | Read task schedules    | tinybots  | `task_schedule`, `robot_schema`                         |
| `ROBOT_PROFILE_READ_ALL`       | Read robot profiles    | tinybots  | `robot_profile`                                         |

**Permission Strategy**: Use OR logic - user must have ANY of these permissions to access the report.

---

## Output Schema Structure

### RobotUsageReportRow

```typescript
{
    // From taas_order + taas_order_status
    orderId: Int
    orderStatus: String  // derived from taas_order_status timestamps

    // From taas_order
    clientId: String
    teamId: String
    teamName: String  // from dashboard_relation.name

    // From taas_subscription
    taasId: Int
    subscriptionStartAt: DateTime

    // From dashboard_robot + dashboard_relation
    serialId: BigInt
    serial: String
    relationId: Int
    organisationName: String

    // From robot_account → user_robot → user_account → user_profile
    primaryUser: {
        userId: Int
        email: String
        firstName: String
        lastName: String
    }

    // From robot_online_status / robot_online_status_history
    onlineStatus: {
        isCurrentlyOnline: Boolean
        wasOnlineInPeriod: Boolean
        lastSeenAt: DateTime
    }

    // From script_execution → script_version → script_category
    executedScriptsByCategory: [{
        categoryName: String
        executionCount: Int
    }]

    // From script_version (MAX created_at)
    lastScriptEditedAt: DateTime

    // From robot_schema → task_schedule
    numberOfPlannedTasks: Int

    // From robot_profile
    robotAlias: String(endUserName)
}
```

---

## Implementation Phases

### Phase 1: Setup & Validation ✅

- [x] Read and understand Prisma schemas
- [x] Document table relationships
- [x] Create implementation plan
- [ ] Clarify business logic questions

### Phase 2: Type Definitions (Nexus)

- [ ] Create `RobotUsageReportFilterInput` input type
- [ ] Create `RobotUsageReportRow` object type
- [ ] Create nested types: `PrimaryUserInfo`, `OnlineStatusInfo`, `ExecutedScriptCategory`

### Phase 3: Service Layer

- [ ] Create `RobotUsageReportService` class
- [ ] Implement filter builders
- [ ] Implement data fetching methods per branch
- [ ] Implement aggregation logic

### Phase 4: Resolver

- [ ] Create `robotUsageReport` query resolver
- [ ] Implement input validation (class-validator)
- [ ] Implement permission checking
- [ ] Call service and return results

### Phase 5: Testing

- [ ] Unit tests for service methods
- [ ] Integration tests for GraphQL query
- [ ] Test all filter combinations
- [ ] Test permission scenarios

### Phase 6: Documentation

- [ ] Update README
- [ ] Add JSDoc comments
- [ ] Generate Nexus types

---

## Open Questions

1. **Online Status**: Should we check current status or historical status within the time period?
    - **Recommendation**: Include both for comprehensive reporting

2. **Contact Email**: Is this always the primary user's email (role_id = 1)?
    - **Assumption**: Yes, based on requirements

3. **Filter Combination**: Can `teamId` and `appUserId` be used together?
    - **Recommendation**: Allow both, apply as AND condition

4. **App User ID Mapping**: Which field exactly is `appUserId`?
    - **Best guess**: `taas_order.client_id` or possibly links to `user_account.id`

5. **Performance**: Expected data volume? Should we implement pagination?
    - **Recommendation**: Implement pagination from the start (cursor-based)

6. **Script Categories**: Count total executions or distinct scripts executed?
    - **Recommendation**: Total executions per category

7. **Serial Matching**: Is `dashboard_robot.serial` always equal to `robot_account.username`?
    - **Assumption**: Yes, this is the primary join key

---

## Technical Notes

- Use `ctx.prisma.dashboard` for dashboard DB queries
- Use `ctx.prisma.tinybots` for tinybots DB queries
- Aggregate in memory (can't do cross-database joins in Prisma)
- Use `flattenDate` utility for ISO 8601 date formatting
- Use `BaseResolver.Wrap` pattern for permission checking
- Follow existing patterns from `retentionKpi` and `robotProfiles` implementations
