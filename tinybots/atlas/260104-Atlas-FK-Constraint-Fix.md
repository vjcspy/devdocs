# 📋 [ATLAS-FK-FIX: 2026-01-04] - Fix Foreign Key Constraint Issues and Flair Model Loading

## References

- [devdocs/tinybots/atlas/OVERVIEW.md](../OVERVIEW.md)
- [devtools/prisma/tinybots/schema.prisma](../../../devtools/prisma/tinybots/schema.prisma#L471) - `event_trigger` FK constraint
- [devtools/prisma/tinybots/schema.prisma](../../../devtools/prisma/tinybots/schema.prisma#L504) - `event_trigger_setting` FK constraint
- [atlas/src/repository/delete/DeleteTablesV2Repository.ts](../../../atlas/src/repository/delete/DeleteTablesV2Repository.ts)
- [atlas/test/setup/DbSetupTypE.ts](../../../atlas/test/setup/DbSetupTypE.ts)
- [atlas/test/setup/DbSetupQueries.ts](../../../atlas/test/setup/DbSetupQueries.ts)

## User Requirements

Production failures in Atlas batch job execution:

1. **Academy Environment**: `OSError: [Errno 39] Directory not empty: '/root/.flair/models/ner-dutch-large'` during model loading
2. **Production Environment**: `sqlMessage: 'Cannot delete or update a parent row: a foreign key constraint fails'` during script deletion

Suspected foreign key constraints:
- `fk_event_trigger_script_ref_id` (event_trigger.script_reference_id → script_reference.id)
- `fk_event_trigger_setting_default_script_ref_id` (event_trigger_setting.default_script_reference_id → script_reference.id)

## 🎯 Objective

**Fix two critical Atlas batch job issues:**
1. Resolve foreign key constraint violations preventing `script_reference` deletion in production
2. Fix Flair model loading race condition causing directory conflicts in Academy environment

### ⚠️ Key Considerations

1. **Data Integrity**: Must maintain referential integrity while deleting archived scripts
2. **Deletion Order**: Child records in `event_trigger` and `event_trigger_setting` must be deleted before `script_reference`
3. **Test Coverage**: Tests must reflect actual database schema including new FK constraints
4. **GDPR Compliance**: Script deletion is required for data retention compliance - cannot be blocked
5. **Python Environment**: Flair model caching may have concurrency issues in containerized environments

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Investigate foreign key constraint failure in detail
  - **Outcome**: Confirm which FK constraints are failing and identify affected tables
  - **Action**: Query production logs for full error messages with constraint names
  - **Action**: Review schema.prisma for all FK relationships to `script_reference`

- [ ] Analyze Flair model loading error
  - **Outcome**: Understand Flair model cache behavior and identify race condition
  - **Action**: Review [atlas/Dockerfile](../../../atlas/Dockerfile) model loading process
  - **Action**: Check if multiple processes attempt to download models concurrently
  - **Action**: Review [atlas/ci/loadFlair.py](../../../atlas/ci/loadFlair.py) for atomic operations

- [ ] Map deletion dependency chain
  - **Outcome**: Complete ordered list of tables that must be deleted before `script_reference`
  - **Current known dependencies**:
    - `script_version` (already handled)
    - `script_v2_task` (already handled)
    - `robot_schema` (already handled)
    - `event_trigger` ❌ **MISSING**
    - `event_trigger_setting` ❌ **MISSING**

### Phase 2: Implementation (File/Code Structure)

#### 2.1 Foreign Key Constraint Fix

**Files to Modify:**

```
atlas/src/repository/delete/
├── DeleteTablesV2Repository.ts    # 🔄 Add event_trigger deletion methods
└── DeleteTables.ts                # ✅ Base class (no changes needed)

atlas/src/batchJobs/
└── BatchJobs.ts                   # 🔄 Add event_trigger deletion to DeleteScriptsV2()

atlas/test/setup/
├── DbSetupTypE.ts                 # 🔄 Add event_trigger cleanup & setup methods
├── DbSetupQueries.ts              # 🔄 Add event_trigger query helpers
└── DbSetupIntelligence.ts         # 🔄 Add event_trigger cleanup (if needed)

atlas/test/repository/delete/
└── DeleteTablesV2RepositoryIT.ts  # 🔄 Add integration tests for event_trigger deletion
```

#### 2.2 Flair Model Loading Fix

**Files to Modify:**

```
atlas/
├── Dockerfile                     # 🔄 Ensure atomic model download
└── ci/
    └── loadFlair.py               # 🔄 Add file locking or idempotency checks
```

### Phase 3: Detailed Implementation Steps

#### Step 3.1: Add `event_trigger` Deletion Logic

**File**: [atlas/src/repository/delete/DeleteTablesV2Repository.ts](../../../atlas/src/repository/delete/DeleteTablesV2Repository.ts)

**Changes Required:**

1. Add SQL DELETE query for `event_trigger` table:
   ```typescript
   private DELETE_EVENT_TRIGGER = 'DELETE `et` FROM `event_trigger` as `et` ' +
     'INNER JOIN `script_reference` as `sr` on `et`.`script_reference_id` = `sr`.`id` ' +
     'WHERE ((`sr`.`archived_at` IS NOT NULL) AND (`sr`.`robot_id` IS NULL))'
   ```

2. Add SQL DELETE query for `event_trigger_setting` table:
   ```typescript
   private DELETE_EVENT_TRIGGER_SETTING = 'DELETE `ets` FROM `event_trigger_setting` as `ets` ' +
     'INNER JOIN `script_reference` as `sr` on `ets`.`default_script_reference_id` = `sr`.`id` ' +
     'WHERE ((`sr`.`archived_at` IS NOT NULL) AND (`sr`.`robot_id` IS NULL))'
   ```

3. Add public deletion methods:
   ```typescript
   public async deleteEventTrigger(): Promise<any> {
     console.info(`Starting to delete rows with sql: \n\t${this.DELETE_EVENT_TRIGGER}`)
     return this.dbTypE.query(this.DELETE_EVENT_TRIGGER, [])
   }

   public async deleteEventTriggerSetting(): Promise<any> {
     console.info(`Starting to delete rows with sql: \n\t${this.DELETE_EVENT_TRIGGER_SETTING}`)
     return this.dbTypE.query(this.DELETE_EVENT_TRIGGER_SETTING, [])
   }
   ```

**Rationale**: These tables have foreign keys to `script_reference` and must be deleted first to avoid constraint violations.

#### Step 3.2: Update DeleteScriptsV2 Batch Job

**File**: [atlas/src/batchJobs/BatchJobs.ts](../../../atlas/src/batchJobs/BatchJobs.ts)

**Changes Required:**

Add deletion calls **before** `deleteScriptReference()`:

```typescript
export async function DeleteScriptsV2 () {
  console.info('Starting batch job: DeleteScriptsV2')
  const dbTypE = new TypEDatabase()
  const dbIntelligence = new IntelligenceDatabase()
  const deleteTablesRepository = new DeleteTablesV2Repository(dbTypE, dbIntelligence)

  // ... existing deletions ...
  await deleteTablesRepository.deleteScriptStep()
  await deleteTablesRepository.deleteScriptExecution()
  await deleteTablesRepository.deleteScriptVersion()
  await deleteTablesRepository.deleteRobotSchema()
  await deleteTablesRepository.deleteScriptTask()
  
  // ⚠️ NEW: Delete event_trigger dependencies BEFORE script_reference
  await deleteTablesRepository.deleteEventTrigger()
  await deleteTablesRepository.deleteEventTriggerSetting()
  
  await deleteTablesRepository.deleteScriptReference()

  dbTypE.close()
  dbIntelligence.close()
}
```

**Critical Order**:
1. Delete `event_trigger` (child of `script_reference`)
2. Delete `event_trigger_setting` (child of `script_reference`)
3. Delete `script_reference` (parent)

#### Step 3.3: Update Test Database Setup

**File**: [atlas/test/setup/DbSetupTypE.ts](../../../atlas/test/setup/DbSetupTypE.ts)

**Changes Required:**

1. Add cleanup queries in `clearDb()` method (insert **before** `REMOVE_SCRIPT_REFERENCES`):
   ```typescript
   const REMOVE_EVENT_TRIGGER = 'DELETE FROM event_trigger'
   await this.database.query(REMOVE_EVENT_TRIGGER, [])
   const REMOVE_EVENT_TRIGGER_SETTING = 'DELETE FROM event_trigger_setting'
   await this.database.query(REMOVE_EVENT_TRIGGER_SETTING, [])
   ```

2. Add helper methods for test data creation:
   ```typescript
   public async addEventTriggerSetting(
     id: number,
     eventTypeId: number,
     defaultScriptReferenceId: number,
     robotId?: number
   ) {
     const INSERT_EVENT_TRIGGER_SETTING = 'INSERT INTO `event_trigger_setting` ' +
       '(`id`, `event_type_id`, `default_script_reference_id`, `robot_id`) VALUES (?, ?, ?, ?)'
     return await this.database.query(INSERT_EVENT_TRIGGER_SETTING, 
       [id, eventTypeId, defaultScriptReferenceId, robotId || null])
   }

   public async addEventTrigger(
     id: number,
     settingId: number,
     robotId: number,
     scriptReferenceId: number,
     outgoingEventId: number,
     status: string = 'pending'
   ) {
     const INSERT_EVENT_TRIGGER = 'INSERT INTO `event_trigger` ' +
       '(`id`, `setting_id`, `robot_id`, `script_reference_id`, `outgoing_event_id`, ' +
       '`status`, `level`, `expected_executed_at`) VALUES (?, ?, ?, ?, ?, ?, ?, NOW())'
     return await this.database.query(INSERT_EVENT_TRIGGER, 
       [id, settingId, robotId, scriptReferenceId, outgoingEventId, status, 1])
   }
   ```

**File**: [atlas/test/setup/DbSetupQueries.ts](../../../atlas/test/setup/DbSetupQueries.ts)

**Changes Required:**

Add cleanup in `clearDb()` (same as DbSetupTypE):
```typescript
const REMOVE_EVENT_TRIGGER = 'DELETE FROM event_trigger'
await this.database.query(REMOVE_EVENT_TRIGGER, [])
const REMOVE_EVENT_TRIGGER_SETTING = 'DELETE FROM event_trigger_setting'
await this.database.query(REMOVE_EVENT_TRIGGER_SETTING, [])
```

#### Step 3.4: Add Integration Tests

**File**: [atlas/test/repository/delete/DeleteTablesV2RepositoryIT.ts](../../../atlas/test/repository/delete/DeleteTablesV2RepositoryIT.ts)

**Changes Required:**

Add test case to verify `event_trigger` and `event_trigger_setting` deletion:

```typescript
describe('Delete event_trigger and event_trigger_setting with script_reference FK', () => {
  it('should delete event_trigger linked to archived script_reference', async () => {
    // Setup: Create archived script_reference with event_trigger
    const scriptRefId = 100
    const eventTriggerId = 200
    const settingId = 300
    const eventSchemaId = 400
    const outgoingEventId = 500
    
    await dbSetupTypE.addEventSchema(eventSchemaId, 'test.event', 1)
    await dbSetupTypE.addOutgoingEvent(outgoingEventId, eventSchemaId, robotId1)
    await dbSetupTypE.addScriptReference(scriptRefId, robotId1, null, archivedDate)
    await dbSetupTypE.addEventTriggerSetting(settingId, eventSchemaId, scriptRefId, robotId1)
    await dbSetupTypE.addEventTrigger(eventTriggerId, settingId, robotId1, scriptRefId, outgoingEventId)
    
    // Execute: Delete event_trigger
    await deleteTablesRepository.deleteEventTrigger()
    
    // Verify: event_trigger is deleted
    const eventTriggers = await dbSetupTypE.query('SELECT * FROM event_trigger WHERE id = ?', [eventTriggerId])
    expect(eventTriggers).to.be.empty
  })

  it('should delete event_trigger_setting linked to archived script_reference', async () => {
    // Similar test for event_trigger_setting
    // ...
  })
  
  it('should allow script_reference deletion after event_trigger cleanup', async () => {
    // Setup: Create full dependency chain
    // Execute: Delete in correct order
    // Verify: script_reference is successfully deleted
    // ...
  })
})
```

#### Step 3.5: Fix Flair Model Loading Race Condition

**File**: [atlas/ci/loadFlair.py](../../../atlas/ci/loadFlair.py)

**Investigation Required First:**
1. Check current implementation of loadFlair.py
2. Determine if model loading uses file locking
3. Identify if multiple processes call this concurrently

**Potential Solutions:**

**Option A: File-based Locking**
```python
import os
import fcntl
import flair

LOCK_FILE = '/tmp/flair_model_download.lock'

def load_flair_with_lock():
    with open(LOCK_FILE, 'w') as lock_file:
        try:
            fcntl.flock(lock_file, fcntl.LOCK_EX | fcntl.LOCK_NB)
            # Load model only if lock acquired
            flair.models.SequenceTagger.load('nl-ner-large')
        except IOError:
            # Another process is loading, wait for it
            fcntl.flock(lock_file, fcntl.LOCK_EX)
            # Model should be loaded by now
        finally:
            fcntl.flock(lock_file, fcntl.LOCK_UN)
```

**Option B: Pre-download in Dockerfile (Preferred)**
```dockerfile
# In Dockerfile, ensure model is fully downloaded during image build
RUN python3 /loadFlair.py && \
    # Verify model exists before proceeding
    python3 -c "import flair; flair.models.SequenceTagger.load('nl-ner-large')"
```

**Option C: Check if Model Exists**
```python
import os
from pathlib import Path

MODEL_PATH = Path('/root/.flair/models/ner-dutch-large')

def load_flair_idempotent():
    if MODEL_PATH.exists() and len(list(MODEL_PATH.glob('*'))) > 0:
        print('Model already downloaded')
        return
    
    # Download model
    flair.models.SequenceTagger.load('nl-ner-large')
```

**File**: [atlas/Dockerfile](../../../atlas/Dockerfile)

**Changes Required:**

Ensure model loading is fully completed during build:
```dockerfile
# Change from:
RUN python3 /loadFlair.py
RUN python3 -m spacy download nl_core_news_lg

# To:
RUN python3 /loadFlair.py && \
    python3 -c "import flair; print('Flair loaded:', flair.models.SequenceTagger.load('nl-ner-large'))" && \
    python3 -m spacy download nl_core_news_lg && \
    python3 -c "import spacy; print('spaCy loaded:', spacy.load('nl_core_news_lg'))"
```

### Phase 4: Testing & Validation

#### Step 4.1: Unit Test Execution

```bash
cd atlas
yarn unit-test
```

**Expected**: All existing tests pass + new event_trigger deletion tests pass

#### Step 4.2: Integration Test Execution

```bash
cd atlas
yarn test
```

**Expected**: DeleteTablesV2RepositoryIT tests pass with event_trigger setup

#### Step 4.3: Manual Verification in Dev Environment

1. Set up test database with `event_trigger` and `event_trigger_setting` referencing archived `script_reference`
2. Run `yarn delete-scripts-v2`
3. Verify no FK constraint errors
4. Verify correct deletion order in logs

#### Step 4.4: Flair Model Loading Test

1. Build Docker image with updated Dockerfile
2. Run container multiple times to ensure idempotency
3. Verify no "Directory not empty" errors
4. Check model files are correctly cached

### Phase 5: Deployment Strategy

#### Step 5.1: Pre-deployment Verification

- [ ] Verify schema.prisma matches production database schema
- [ ] Confirm no other tables have FK constraints to `script_reference`
- [ ] Review deletion order in DeleteScriptsV2 one final time

#### Step 5.2: Production Deployment

1. **Deploy to Academy environment first** (currently affected by Flair error)
2. Monitor logs for successful batch job execution
3. If successful, deploy to Production environment
4. Monitor FK constraint errors resolution

#### Step 5.3: Rollback Plan

If FK constraint errors persist:
1. Temporarily disable `DeleteScriptsV2` batch job
2. Investigate additional FK constraints not covered
3. Apply hotfix with additional table deletions

## 📊 Summary of Results

> Do not summarize results until implementation is done

### ✅ Completed Achievements
- TBD after implementation

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Verify event_schema dependency**: Does `event_trigger_setting` require `event_schema` to exist? May need to preserve or handle differently
- [ ] **Check outgoing_event dependency**: `event_trigger` references `outgoing_event` - verify if cascade delete is configured correctly
- [ ] **Confirm robot_account cascade**: Verify if `event_trigger` with `robot_id` FK has proper cascade on robot deletion
- [ ] **Review intelligence DB**: Confirm if `event_trigger` tables exist in intelligence DB or only typ-e DB
- [ ] **Load test impact**: Assess if adding two more DELETE queries impacts batch job performance significantly
- [ ] **Get actual production error logs**: Need full SQL error message to confirm exact constraint name failing

### 🔍 Additional Investigation Needed

1. **Check for additional FK constraints**: Run query on production schema to find ALL tables referencing `script_reference`:
   ```sql
   SELECT 
     TABLE_NAME, 
     CONSTRAINT_NAME, 
     COLUMN_NAME, 
     REFERENCED_TABLE_NAME, 
     REFERENCED_COLUMN_NAME
   FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
   WHERE REFERENCED_TABLE_NAME = 'script_reference'
     AND CONSTRAINT_SCHEMA = 'tinybots';
   ```

2. **Review loadFlair.py**: Need to read actual implementation to determine best fix approach

3. **Confirm batch job execution order**: Verify if `DeleteScriptsV2` runs after `CopyScriptsV2` (should not delete before copying)

---

## Implementation Checklist

- [ ] Phase 1: Analysis & Preparation completed
- [ ] Phase 2: Code structure designed
- [ ] Phase 3.1: event_trigger deletion logic added
- [ ] Phase 3.2: DeleteScriptsV2 batch job updated
- [ ] Phase 3.3: Test database setup updated
- [ ] Phase 3.4: Integration tests added
- [ ] Phase 3.5: Flair model loading fixed
- [ ] Phase 4: All tests passing
- [ ] Phase 5: Deployed and verified in production
