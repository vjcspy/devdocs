# 📋 [TYP-E-ATLAS-PERM: 2026-01-11] - Grant Event Trigger Permissions to Atlas User

## References

- [devdocs/tinybots/typ-e/OVERVIEW.md](./OVERVIEW.md)
- [devdocs/tinybots/atlas/260104-Atlas-FK-Constraint-Fix.md](../atlas/260104-Atlas-FK-Constraint-Fix.md) - Atlas FK constraint fix requiring these permissions
- [typ-e/src/main/resources/db/migration/V83__trigger_service.sql](../../../typ-e/src/main/resources/db/migration/V83__trigger_service.sql) - event_trigger table definitions
- [typ-e/src/main/resources/db/migration/V59__script_anonymisation.sql](../../../typ-e/src/main/resources/db/migration/V59__script_anonymisation.sql) - atlas-rw user creation pattern
- [typ-e/pom.xml](../../../typ-e/pom.xml) - Project version (must match migration version)

## User Requirements

From Atlas FK Constraint Fix plan (`devdocs/tinybots/atlas/260104-Atlas-FK-Constraint-Fix.md`):

> **Pre-existing Test Failures (NOT related to this fix)**
> - **Batch jobs IT**: `SELECT command denied to user 'atlas-rw' for table 'event_trigger'` - **Database permission issue**
> - **Action Required**: Grant SELECT/DELETE on `event_trigger` and `event_trigger_setting` tables to `atlas-rw` user

## 🎯 Objective

Grant `SELECT` and `DELETE` permissions on `event_trigger` and `event_trigger_setting` tables to the `atlas-rw` MySQL user to enable Atlas batch jobs to delete orphaned event triggers when deleting archived `script_reference` records.

### ⚠️ Key Considerations

1. **FK Constraint Chain**: `event_trigger` → `event_trigger_setting` → `script_reference`
   - Atlas must delete `event_trigger` records before `event_trigger_setting` before `script_reference`
   - Without SELECT/DELETE permissions, the FK constraint cleanup fails

2. **Network Ranges**: `atlas-rw` user is created for 3 private network ranges:
   - `10.0.0.0/255.0.0.0` (Class A private)
   - `172.16.0.0/255.240.0.0` (Class B private)
   - `192.168.0.0/255.255.0.0` (Class C private)

3. **Permissions Pattern**: Consistent with existing Atlas permissions (e.g., `V59__script_anonymisation.sql`, `V59_2__script_task_permissions.sql`)
   - `SELECT` - Required to query records for deletion
   - `DELETE` - Required to remove records

4. **Versioning Convention**: `pom.xml` version must match migration version
   - Pattern: `{MIGRATION_VERSION}.0.0-SNAPSHOT`
   - Current: V99 migration → `99.0.0-SNAPSHOT`
   - Next: V100 migration → `100.0.0-SNAPSHOT`

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation ✅ COMPLETED

- [x] Identify required tables from Atlas implementation
  - **Result**: `event_trigger`, `event_trigger_setting`
  
- [x] Confirm tables exist in typ-e schema
  - **Result**: Both tables created in `V83__trigger_service.sql`
  
- [x] Identify permission pattern for atlas-rw user
  - **Result**: GRANT SELECT, DELETE for 3 network ranges (see `V59__script_anonymisation.sql`)
  
- [x] Identify next migration version number
  - **Result**: V100 (latest is V99)

- [x] Identify current pom.xml version and required update
  - **Current**: `99.0.0-SNAPSHOT`
  - **Required**: `100.0.0-SNAPSHOT` (must match migration V100)

### Phase 2: Implementation

#### 2.1 Files to Modify/Create

```
typ-e/
├── pom.xml                                            # 🚧 TO MODIFY - bump version
└── src/main/resources/db/migration/
    └── V100__grant_atlas_event_trigger_permissions.sql    # 🚧 TO CREATE
```

#### 2.2 Update pom.xml Version

**File**: `pom.xml` (line 6)

```diff
- <version>99.0.0-SNAPSHOT</version>
+ <version>100.0.0-SNAPSHOT</version>
```

#### 2.3 Migration SQL Content

**File**: `V100__grant_atlas_event_trigger_permissions.sql`

```sql
-- Grant permissions on event_trigger and event_trigger_setting tables to atlas-rw user
-- Required for Atlas batch job to clean up FK dependencies before deleting script_reference
-- See: devdocs/tinybots/atlas/260104-Atlas-FK-Constraint-Fix.md

-- event_trigger table permissions
GRANT SELECT, DELETE ON `tinybots`.`event_trigger` TO 'atlas-rw'@'10.0.0.0/255.0.0.0';
GRANT SELECT, DELETE ON `tinybots`.`event_trigger` TO 'atlas-rw'@'172.16.0.0/255.240.0.0';
GRANT SELECT, DELETE ON `tinybots`.`event_trigger` TO 'atlas-rw'@'192.168.0.0/255.255.0.0';

-- event_trigger_setting table permissions
GRANT SELECT, DELETE ON `tinybots`.`event_trigger_setting` TO 'atlas-rw'@'10.0.0.0/255.0.0.0';
GRANT SELECT, DELETE ON `tinybots`.`event_trigger_setting` TO 'atlas-rw'@'172.16.0.0/255.240.0.0';
GRANT SELECT, DELETE ON `tinybots`.`event_trigger_setting` TO 'atlas-rw'@'192.168.0.0/255.255.0.0';
```

### Phase 3: Verification

1. **Local Verification** (if Docker databases are running):
   ```bash
   # Apply migration to local typ-e-db
   cd typ-e
   mvn flyway:migrate -Dflyway.url=jdbc:mysql://localhost:1123/tinybots -Dflyway.user=root -Dflyway.password=ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN
   
   # Verify grants applied
   docker exec mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN -e "SHOW GRANTS FOR 'atlas-rw'@'10.0.0.0/255.0.0.0';" | grep event_trigger
   ```

2. **CI/CD Verification**:
   - Migration will be applied automatically by Flyway during deployment
   - Atlas batch job tests should pass after this migration is deployed

### Phase 4: Deployment Order

**CRITICAL**: Deploy typ-e migration BEFORE running Atlas batch jobs in production.

1. ✅ Deploy `typ-e` with new migration `V100__grant_atlas_event_trigger_permissions.sql`
2. ✅ Verify migration applied successfully in each environment
3. ✅ Deploy/run Atlas batch jobs - FK constraint cleanup should work

## 📊 Summary of Results

> *Do not fill until implementation is complete*

### ✅ Completed Achievements
- [ ] pom.xml version bumped to `100.0.0-SNAPSHOT`
- [ ] Migration file `V100__grant_atlas_event_trigger_permissions.sql` created
- [ ] Local verification passed
- [ ] Deployed to staging
- [ ] Deployed to production
- [ ] Atlas batch jobs verified working

## 🚧 Outstanding Issues & Follow-up

### Dependencies
- [ ] Coordinate with Atlas deployment - typ-e migration must be deployed first
- [ ] Update Atlas plan to mark Phase 4 (DB permissions) as completed after this is deployed
