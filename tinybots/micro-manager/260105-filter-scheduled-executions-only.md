# 📋 [260105] - Filter Scheduled Executions Only in Existing APIs

## References

- **Global Standard**: `devdocs/tinybots/OVERVIEW.md`
- **Repo Standard**: `devdocs/tinybots/micro-manager/OVERVIEW.md`
- **Original Implementation Plan**: `devdocs/tinybots/micro-manager/251206-store-trigger-script.md`
- **Key Files in micro-manager**:
  - `src/repository/ScriptExecutionRepository.ts` - Database queries that need filtering
  - `src/controllers/ScriptUserController.ts` - User-facing execution retrieval endpoint
  - `src/controllers/InternalExecutionController.ts` - Internal execution search endpoint
  - `src/services/ScriptExecutionService.ts` - Business logic layer
  - `src/routes/routes.ts` - API routing

## User Requirements

**Background:**

In ticket `251206-store-trigger-script`, we implemented storing trigger-based script executions in the `script_execution` table alongside scheduled executions. The table now contains TWO types of executions:

1. **Scheduled executions**: Have `schedule_id` and `planned` timestamp (existing behavior)
2. **Triggered executions**: Have `triggering_event_id`, but `schedule_id` and `planned` are NULL (new)

**Stakeholder Requirement:**

- **Backward Compatibility**: All existing APIs that query `script_execution` must continue to return **ONLY scheduled executions**
- **No Breaking Changes**: Trigger executions should be invisible to existing clients/users
- **New APIs**: Only the new v6 triggered execution endpoint should return triggered executions

**Problem:**

Currently, after implementation of `251206-store-trigger-script`, some existing APIs might accidentally return both scheduled AND triggered executions because the SQL queries don't filter by `schedule_id IS NOT NULL`.

---

## 🎯 Objective

**Add filtering conditions to all existing APIs that query `script_execution` table to ensure they return ONLY scheduled executions, excluding trigger-based executions.**

### ⚠️ Key Considerations

1. **Backward Compatibility is Critical**: Existing API consumers (robots, dashboards, users) must not see any behavior change
2. **SQL Query Safety**: All SELECT queries against `script_execution` must explicitly filter `WHERE schedule_id IS NOT NULL` (except for triggered execution endpoints)
3. **Don't Touch Triggered APIs**: The new v6 triggered execution endpoint (`PUT /v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered/:triggeringEventId`) should remain unchanged
4. **Database Schema**: The schema already supports this (schedule_id can be NULL for triggered executions)
5. **Performance**: Adding `schedule_id IS NOT NULL` to indexed queries should have minimal performance impact

---

## 🔍 Analysis: APIs & Queries Affected

### Current APIs Querying `script_execution` Table

Based on code analysis, here are ALL endpoints that query `script_execution`:

| # | Endpoint | Controller | Repository Method | SQL Query | Status |
|---|----------|------------|-------------------|-----------|--------|
| 1 | `GET /v2/scripts/user/robots/:robotId/scripts/:scriptReferenceId/executions/:scheduleId` | `ScriptUserController.scriptExecutionGet` | `getScriptExecution()` | `GET_SCRIPT_EXECUTION` | 🔄 **NEEDS FILTER** |
| 2 | `POST /internal/v4/scripts/executions/search` | `InternalExecutionController.getExecutionsV4` | `getScriptExecutions()` | `GET_REPORT_EXECUTIONS` | 🔄 **NEEDS FILTER** |
| 3 | `PUT /v2/scripts/robot/scripts/:scriptReferenceId/executions/:scheduleId` | `ScriptRobotController.scriptExecutionPut` | `getScriptExecutionId()` | `GET_SCRIPT_EXECUTION_ID` | 🔄 **NEEDS FILTER** |
| 4 | `PUT /v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered/:triggeringEventId` | `ScriptTriggeredExecutionController.putTriggeredExecution` | `getTriggeredExecutionId()` | `GET_TRIGGERED_EXECUTION_ID` | ✅ **ALREADY CORRECT** (filters by `triggering_event_id IS NOT NULL`) |

---

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation ✅ COMPLETED

**Analysis Results:**

I have identified **3 SQL queries** that need modification:

1. ✅ **`GET_SCRIPT_EXECUTION`** - Used by user-facing execution detail API
2. ✅ **`GET_REPORT_EXECUTIONS`** - Used by internal execution search (reporting/analytics)
3. ✅ **`GET_SCRIPT_EXECUTION_ID`** - Used for idempotency check when saving scheduled executions

**Edge Cases:**

- **Case 1: Empty Results**: If all executions for a robot are triggered (no scheduled), existing APIs should return empty array/404 (expected behavior)
- **Case 2: Mixed Executions**: If a robot has both scheduled and triggered executions in the same time range, only scheduled should be returned
- **Case 3: Idempotency Check**: When robots retry saving scheduled executions, the query must only find scheduled executions (not triggered ones with same timestamp)

---

### Phase 2: Implementation (File/Code Structure)

**Files to Modify:**

```
micro-manager/
└── src/
    └── repository/
        └── ScriptExecutionRepository.ts    # 🔄 MODIFY - Add schedule_id filters to 3 queries
```

**No other files need changes** because:
- Controllers and Services call repository methods (abstraction preserved)
- API routes unchanged
- DTOs unchanged
- Business logic unchanged

---

### Phase 3: Detailed Implementation Steps

#### Step 1: Modify `GET_SCRIPT_EXECUTION` Query

**File:** `src/repository/ScriptExecutionRepository.ts`

**Current Query (Lines 14-18):**

```typescript
private GET_SCRIPT_EXECUTION = `
  SELECT se.id, se.schedule_id, se.script_version_id, se.script_reference_id, se.planned
  FROM script_reference sr
  JOIN script_execution se ON (se.script_reference_id = sr.id)
  WHERE se.schedule_id = ? AND sr.robot_id = ? AND se.planned = ?`
```

**Analysis:**
- ✅ **Already implicitly filters scheduled executions** because it requires `se.schedule_id = ?` (cannot be NULL)
- ⚠️ **However**, it's not explicit that we're excluding triggered executions
- **Recommendation**: Add explicit check for clarity and maintainability

**Proposed Change:**

```typescript
private GET_SCRIPT_EXECUTION = `
  SELECT se.id, se.schedule_id, se.script_version_id, se.script_reference_id, se.planned
  FROM script_reference sr
  JOIN script_execution se ON (se.script_reference_id = sr.id)
  WHERE se.schedule_id = ? 
    AND sr.robot_id = ? 
    AND se.planned = ?
    AND se.schedule_id IS NOT NULL`  // ← Explicit filter for scheduled executions
```

**Justification:**
- Makes intent explicit: "This query is for scheduled executions only"
- Prevents future bugs if query is modified
- Minimal performance impact (schedule_id already indexed for `= ?` check)

---

#### Step 2: Modify `GET_SCRIPT_EXECUTION_ID` Query

**File:** `src/repository/ScriptExecutionRepository.ts`

**Current Query (Lines 20-25):**

```typescript
private GET_SCRIPT_EXECUTION_ID = `
  SELECT se.id, se.schedule_id, se.script_version_id, se.script_reference_id, se.planned
  FROM script_execution se
  JOIN script_reference sr ON (se.script_reference_id = sr.id)
  WHERE se.schedule_id = ? AND sr.robot_id = ? AND se.script_reference_id=? AND se.script_version_id=? AND se.planned = ?`
```

**Analysis:**
- ✅ **Already implicitly filters scheduled executions** because it requires `se.schedule_id = ?`
- **Used for**: Idempotency check when robots retry saving scheduled executions
- **Critical**: Must NOT accidentally match triggered executions

**Proposed Change:**

```typescript
private GET_SCRIPT_EXECUTION_ID = `
  SELECT se.id, se.schedule_id, se.script_version_id, se.script_reference_id, se.planned
  FROM script_execution se
  JOIN script_reference sr ON (se.script_reference_id = sr.id)
  WHERE se.schedule_id = ? 
    AND sr.robot_id = ? 
    AND se.script_reference_id = ? 
    AND se.script_version_id = ? 
    AND se.planned = ?
    AND se.schedule_id IS NOT NULL`  // ← Explicit filter for scheduled executions
```

**Justification:** Same as Step 1 - explicit intent, future-proofing

---

#### Step 3: Modify `GET_REPORT_EXECUTIONS` Query (CRITICAL)

**File:** `src/repository/ScriptExecutionRepository.ts`

**Current Query (Lines 59-66):**

```typescript
private GET_REPORT_EXECUTIONS = `
  SELECT se.id, sr.robot_id, se.planned, se.schedule_id, se.script_reference_id, se.script_version_id, se.created_at
  FROM script_reference AS sr
  RIGHT JOIN script_execution AS se ON (se.script_reference_id = sr.id)
  WHERE sr.robot_id IN (?)
  AND se.planned >= ? AND se.planned <= ?
  AND se.id > ?
  ORDER BY se.id ASC
  LIMIT ?`
```

**Analysis:**
- ❌ **CRITICAL BUG**: This query does NOT filter by `schedule_id`
- **Impact**: Will return BOTH scheduled AND triggered executions
- **Used by**: `POST /internal/v4/scripts/executions/search` - internal API for reporting/analytics
- **Breaking Change**: After trigger implementation, this API started returning mixed results

**Proposed Change:**

```typescript
private GET_REPORT_EXECUTIONS = `
  SELECT se.id, sr.robot_id, se.planned, se.schedule_id, se.script_reference_id, se.script_version_id, se.created_at
  FROM script_reference AS sr
  RIGHT JOIN script_execution AS se ON (se.script_reference_id = sr.id)
  WHERE sr.robot_id IN (?)
    AND se.planned >= ? 
    AND se.planned <= ?
    AND se.id > ?
    AND se.schedule_id IS NOT NULL  -- ← NEW: Exclude triggered executions
  ORDER BY se.id ASC
  LIMIT ?`
```

**Justification:**
- **Restores Original Behavior**: Before trigger implementation, all executions had schedule_id, so this query only returned scheduled executions implicitly
- **Fixes Breaking Change**: Stakeholders expect this API to return ONLY scheduled executions
- **Performance**: `schedule_id IS NOT NULL` is a lightweight check

---

#### Step 4: Verify Triggered Execution Queries Are Untouched

**Queries that should ONLY return triggered executions:**

1. **`GET_TRIGGERED_EXECUTION_ID`** (Line 28-31) - Already correct:
   ```typescript
   WHERE se.triggering_event_id = ? AND sr.robot_id = ? AND se.script_reference_id = ? AND se.script_version_id = ?
   ```
   ✅ Filters by `triggering_event_id IS NOT NULL` implicitly

2. **`GET_EXECUTIONS_BY_TRIGGER`** (Line 68-72) - Already correct:
   ```typescript
   WHERE se.triggering_event_id = ?
   ```
   ✅ Only returns triggered executions

3. **`GET_TRIGGERED_EXECUTIONS_WITH_EVENTS`** (Line 74-86) - Already correct:
   ```typescript
   WHERE se.triggering_event_id IS NOT NULL
   ```
   ✅ Explicitly filters for triggered executions

**Action:** ✅ No changes needed for these queries

---

### Phase 4: Testing Strategy

#### Manual Testing Checklist

**Test Case 1: Scheduled Execution Retrieval**

```bash
# Setup: Robot 123 has:
# - 2 scheduled executions (schedule_id = 100, 101)
# - 1 triggered execution (triggering_event_id = 500)

# Test: GET user execution detail
GET /v2/scripts/user/robots/123/scripts/456/executions/100?planned=2025-12-06T10:00:00Z

# Expected: Returns scheduled execution 100 (not triggered one)
# Actual: [To be verified during implementation]
```

**Test Case 2: Internal Execution Search**

```bash
# Setup: Robot 123 has mixed executions in Dec 2025:
# - 5 scheduled executions
# - 3 triggered executions

# Test: Search all executions
POST /internal/v4/scripts/executions/search
Body: {
  "robotIds": [123],
  "from": "2025-12-01T00:00:00Z",
  "to": "2025-12-31T23:59:59Z",
  "lastExecutionId": 0,
  "limit": 100
}

# Expected: Returns only 5 scheduled executions (not the 3 triggered ones)
# Actual: [To be verified during implementation]
```

**Test Case 3: Idempotency Check for Scheduled Executions**

```bash
# Setup: Robot retries saving same scheduled execution twice

# First attempt:
PUT /v2/scripts/robot/scripts/456/executions/100
Body: { "scriptVersionId": 789, "planned": "2025-12-06T10:00:00Z", ... }
# Expected: 200 OK, execution created

# Second attempt (retry):
PUT /v2/scripts/robot/scripts/456/executions/100
Body: { "scriptVersionId": 789, "planned": "2025-12-06T10:00:00Z", ... }
# Expected: 200 OK, no duplicate created (idempotency works)

# Verify: Only 1 execution exists in DB
```

**Test Case 4: Triggered Executions Still Work**

```bash
# Test: Save triggered execution
PUT /v6/scripts/robot/scripts/456/executions/triggered/500
Body: { "scriptVersionId": 789, "scriptExecutionSteps": [...] }

# Expected: 200 OK, triggered execution saved with triggering_event_id = 500

# Test: Retry (idempotency)
PUT /v6/scripts/robot/scripts/456/executions/triggered/500
Body: { "scriptVersionId": 789, "scriptExecutionSteps": [...] }

# Expected: 200 OK, no duplicate created

# Verify: Triggered execution is NOT returned by scheduled execution APIs
GET /v2/scripts/user/robots/123/scripts/456/executions/100
# Expected: Does NOT include the triggered execution
```

#### Database Verification Queries

```sql
-- Verify scheduled executions have schedule_id
SELECT id, schedule_id, triggering_event_id, planned, created_at
FROM script_execution
WHERE script_reference_id = 456
  AND schedule_id IS NOT NULL;

-- Verify triggered executions have triggering_event_id
SELECT id, schedule_id, triggering_event_id, created_at
FROM script_execution
WHERE script_reference_id = 456
  AND triggering_event_id IS NOT NULL;

-- Verify no overlap (exclusive)
SELECT id, schedule_id, triggering_event_id
FROM script_execution
WHERE script_reference_id = 456
  AND schedule_id IS NOT NULL
  AND triggering_event_id IS NOT NULL;
-- Expected: 0 rows (mutually exclusive)
```

---

## 📊 Summary of Changes

### Modified Files

| File | Lines Changed | Change Type | Impact |
|------|--------------|-------------|--------|
| `src/repository/ScriptExecutionRepository.ts` | 3 queries (lines ~14-66) | Add `schedule_id IS NOT NULL` filter | ✅ Restores backward compatibility |

### SQL Query Changes Summary

| Query Name | Change | Reason |
|-----------|--------|--------|
| `GET_SCRIPT_EXECUTION` | Add explicit `schedule_id IS NOT NULL` | Clarity and future-proofing (already implicitly filtered) |
| `GET_SCRIPT_EXECUTION_ID` | Add explicit `schedule_id IS NOT NULL` | Clarity and future-proofing (already implicitly filtered) |
| `GET_REPORT_EXECUTIONS` | **Add `schedule_id IS NOT NULL`** | **CRITICAL FIX: Currently returns mixed results** |

### API Behavior Changes

| Endpoint | Before Fix | After Fix |
|----------|-----------|-----------|
| `GET /v2/scripts/user/.../executions/:scheduleId` | Returns scheduled only (by accident) | Returns scheduled only (explicit) |
| `POST /internal/v4/scripts/executions/search` | ❌ Returns scheduled + triggered | ✅ Returns scheduled only |
| `PUT /v2/scripts/robot/.../executions/:scheduleId` | Idempotency check works (by accident) | Idempotency check works (explicit) |
| `PUT /v6/scripts/robot/.../executions/triggered/:triggeringEventId` | ✅ Works correctly | ✅ No change |

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Questions for Stakeholders

1. **Question 1: Reporting & Analytics Impact**
   - **Issue**: Internal execution search API (`POST /internal/v4/scripts/executions/search`) currently returns mixed results after trigger implementation
   - **Impact**: Any reporting dashboards or analytics using this API might see data discrepancies after fix
   - **Action Required**: Verify which services/dashboards consume this API and inform them of the fix
   - **Consumers to Check**: 
     - `wonkers-graphql` (if it queries execution data)
     - `wonkers-api` (if it has execution analytics endpoints)
     - Any BI tools or dashboards

2. **Question 2: Future API for Triggered Executions**
   - **Issue**: Currently, there's NO public API to retrieve/search triggered executions (only create)
   - **Impact**: No visibility into trigger execution history for users/admins
   - **Follow-up Ticket**: Should we create `GET /v6/scripts/user/robots/:robotId/scripts/:scriptReferenceId/executions/triggered?from=...&to=...`?

3. **Question 3: Migration of Existing Data**
   - **Issue**: If triggered executions were accidentally created with schedule_id before this fix, they might be visible in scheduled execution APIs
   - **Action Required**: Run database audit to check if any executions have BOTH `schedule_id` and `triggering_event_id`
   - **SQL Audit Query**:
     ```sql
     SELECT COUNT(*) 
     FROM script_execution 
     WHERE schedule_id IS NOT NULL 
       AND triggering_event_id IS NOT NULL;
     ```
   - **Expected**: 0 rows (mutually exclusive by design)
   - **If > 0**: Data cleanup needed before deploying this fix

### 📝 Recommendations

1. **Add Database Constraint** (Optional but Recommended):
   ```sql
   ALTER TABLE script_execution
   ADD CONSTRAINT check_execution_type
   CHECK (
     (schedule_id IS NOT NULL AND triggering_event_id IS NULL) OR
     (schedule_id IS NULL AND triggering_event_id IS NOT NULL)
   );
   ```
   **Benefit**: Enforces data integrity at DB level (prevents mixed execution types)

2. **Add Execution Type Enum** (Future Enhancement):
   - Consider adding `execution_type ENUM('scheduled', 'triggered')` column
   - Makes queries more readable: `WHERE execution_type = 'scheduled'`
   - Simplifies logic if we add more execution types in future (e.g., 'manual', 'api-initiated')

3. **Add Unit Tests** (Strongly Recommended):
   - Test `getScriptExecution()` returns only scheduled
   - Test `getScriptExecutions()` excludes triggered
   - Test `getScriptExecutionId()` doesn't match triggered executions
   - Test triggered execution idempotency is separate from scheduled

---

## ✅ Acceptance Criteria

- [ ] All 3 SQL queries in `ScriptExecutionRepository.ts` are modified with explicit `schedule_id IS NOT NULL` filter
- [ ] `GET /v2/scripts/user/.../executions/:scheduleId` returns ONLY scheduled executions
- [ ] `POST /internal/v4/scripts/executions/search` returns ONLY scheduled executions (CRITICAL FIX)
- [ ] `PUT /v2/scripts/robot/.../executions/:scheduleId` idempotency check works correctly (no collision with triggered)
- [ ] `PUT /v6/scripts/robot/.../executions/triggered/:triggeringEventId` continues to work unchanged
- [ ] Manual testing confirms scheduled and triggered executions are properly segregated
- [ ] Database audit confirms no executions have both schedule_id and triggering_event_id
- [ ] Code review confirms no other queries need modification
- [ ] Documentation updated to clarify execution type filtering

---

## 📅 Implementation Timeline Estimate

- **Analysis & Planning**: ✅ Completed (this document)
- **Code Changes**: ~30 minutes (3 simple SQL query modifications)
- **Manual Testing**: ~1 hour (4 test cases + database verification)
- **Code Review**: ~30 minutes
- **Deployment**: Standard CI/CD pipeline

**Total Estimated Time**: ~2-3 hours

**Risk Level**: 🟢 **LOW** (simple WHERE clause additions, backward-compatible)
