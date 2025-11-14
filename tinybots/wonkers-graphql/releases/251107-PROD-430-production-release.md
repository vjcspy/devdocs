# Production release: Tessa Order Status Report / Retention KPI / Robot Profile

This release bundles updates that add Robot Profile data to rawData, expose Retention KPI metrics, and extend the Tessa Order Status Report.

## Scope & Jira coverage

| Jira key | Description | Notes |
| --- | --- | --- |
| [PROD-202](https://tinybots.atlassian.net/browse/PROD-202) | Add robot profile to raw data | Requires new read permission |
| [PROD-261](https://tinybots.atlassian.net/browse/PROD-261) | Retention KPI data query | Feeds dashboard KPI panels |
| [PROD-262](https://tinybots.atlassian.net/browse/PROD-262) | Tessa Order Status Report | Primary organisation report |
| [PROD-521](https://tinybots.atlassian.net/browse/PROD-521) | Handle orders without robots/subscriptions | Patches empty rows in Tessa report |

## Pre-deployment checklist

- [ ] Code reviews: [PR #88](https://bitbucket.org/tinybots/wonkers-graphql/pull-requests/88) and [PR #87](https://bitbucket.org/tinybots/wonkers-graphql/pull-requests/87) merged.
- [ ] Data migrations built and dry-run on staging.
- [ ] `ROBOT_PROFILE_READ_ALL` permission mapped to every required user/role.
- [ ] Environment variables `TINYBOTS_DB_HOST` and `DASHBOARD_DB_HOST` configured for the target runtime.

## Deployment steps

### 1. Run data migrations

- Typ-e @ `2188c82ac10b87e0c674dfe869365aff32d0bb00`
- Wonker-db @ `535f6b4508b07d671fdf7808aa0a0d29fdd93fdd`

### 2. Grant `ROBOT_PROFILE_READ_ALL`

- **Add Permission**: Add `ROBOT_PROFILE_READ_ALL` permission to necessary users/roles

### 3. Configure database hosts

System config expects two new env variables referenced in `config/databases`:

```json
"databases": {
  "tinybots": { "host": "TINYBOTS_DB_HOST" },
  "dashboard": { "host": "DASHBOARD_DB_HOST" }
}
```

4. ### Run Pipeline

   Run pipeline after merge 2 PRs [PR #88](https://bitbucket.org/tinybots/wonkers-graphql/pull-requests/88) and [PR #87](https://bitbucket.org/tinybots/wonkers-graphql/pull-requests/87).

## Build verification tests

### Robot Profile

**Filter by `robotIds`**

```graphql
query {
  rawData {
    robotProfiles(robotIds: [123, 456]) {
      robot_id
      endUserName
      created_at
      updated_at
    }
  }
}
```

**Filter by `serials`**

```graphql
query {
  rawData {
    robotProfiles(serials: ["ROBOT001", "ROBOT002"]) {
      robot_id
      endUserName
      created_at
      updated_at
    }
  }
}
```

### Tessa Order Status Report

```graphql
query {
  reports {
    organisationReports {
      tessaOrderStatusReport(relationIds: 159) {
        teamId
        clientId
        serial
        onlineStatus
        deviceLastSeen
      }
    }
  }
}
```

### Retention KPI data

```graphql
query {
  reports {
    kpi {
      operationKPIs {
        retentionKpi(P: 10, R: 10) {
          date
          retentionKPI
          population
        }
      }
    }
  }
}
```
