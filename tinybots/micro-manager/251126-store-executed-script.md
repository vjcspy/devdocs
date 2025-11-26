# 📋 [251126] - Store Executed Script (Trigger-Initiated Executions)

## References

- **Repository**: `micro-manager`
- **Global Standard**: `devdocs/tinybots/OVERVIEW.md`
- **Repo Standard**: `devdocs/tinybots/micro-manager/OVERVIEW.md`
- **Related Files**:
  - `src/repository/ScriptExecutionRepository.ts` - Current execution tracking (scheduled scripts)
  - `src/services/ScriptExecutionService.ts` - Execution business logic
  - `src/schemas/body/ScriptExecution.yaml` - Execution DTOs
  - `docs/micro-manager.yaml` - OpenAPI specification
  - `src/routes/routes.ts` - API routes

## User Requirements

Currently Tessa executes scripts and stores the executions of scripts that are executed based on a schedule.

**Problem**: Scripts that do not start from a schedule (trigger-initiated scripts) do not store the script executions.

**Goal**: Store script executions of scripts triggered by trigger events to gain insights into trigger-based automation.

**Required Information to Track**:

1. Node executed, with data (same as scheduled executions)
2. Time of execution of the node
3. Incoming `eventId` that started the execution

**Deliverables**:

1. Create new API in `tiny-specs`
2. Implement API in `micro-manager`

## 🎯 Objective

Extend micro-manager's script execution storage to support **trigger-initiated (unscheduled) scripts**, capturing:

- Script steps executed with execution data (same structure as scheduled scripts)
- Execution timestamps per step
- Triggering event ID for traceability back to the incoming event

This enables visibility into automation triggered by events (e.g., robot status changes, external triggers) rather than time-based schedules.

### ⚠️ Key Considerations

- **Existing Architecture**: Scheduled executions use `script_execution` table with `schedule_id` and `planned` timestamp as required fields
- **Schema Constraint**: Current schema requires `schedule_id` (BIGINT NOT NULL) - trigger-based executions have no schedule
- **Unscheduled Execution API**: There's already `PostUnscheduledScriptExecutionDTO` in the codebase but it doesn't persist to database
- **Event Traceability**: Need to link execution to `incoming_event.id` from megazord-events
- **API Versioning**: Create v6 API endpoint or extend existing internal endpoints
- **Backward Compatibility**: Scheduled execution flow must remain unchanged

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Understand current scheduled execution flow
  - **File**: `src/services/ScriptExecutionService.ts` → `saveExecution()` method
  - **Outcome**: Document how `schedule_id`, `planned`, `script_version_id`, `script_reference_id` are used
  - **Key Finding**: `saveExecution()` requires `scheduleId` and `planned` - both N/A for trigger-based scripts

- [ ] Analyze unscheduled execution handling
  - **File**: `src/services/ScriptExecutionService.ts` → `handleUnscheduledSteps()` method
  - **Current Behavior**: Only processes report steps, does NOT persist to database
  - **Outcome**: Confirm that unscheduled executions currently don't create `script_execution` records

- [ ] Review database schema constraints
  - **Table**: `script_execution`
  - **Columns**:
    - `schedule_id` BIGINT NOT NULL - ⚠️ Blocker for trigger-based
    - `planned` TIMESTAMP NOT NULL - ⚠️ Blocker for trigger-based
    - `script_version_id` BIGINT NOT NULL - ✅ OK
    - `script_reference_id` BIGINT NOT NULL - ✅ OK
  - **Outcome**: Define migration strategy (make nullable, use sentinel values, or add discriminator column)

- [ ] Identify incoming event ID source
  - **Context**: Trigger-based scripts start from `megazord-events` via `m-o-triggers`
  - **API Flow**: Robot receives trigger → executes script → reports back to micro-manager
  - **Question**: Does robot/API call include `eventId`? Or do we need to add it?
  - **Outcome**: Determine where `eventId` comes from in the request

### Phase 2: Database Schema Design

#### Option 1: Make Schedule Columns Nullable (Recommended)

**Pros**:

- Clean semantics: NULL = trigger-based, NOT NULL = scheduled
- Simple filtering: `WHERE schedule_id IS NOT NULL` vs `WHERE schedule_id IS NULL`
- No dummy data

**Cons**:

- Schema migration required
- Need to drop and recreate foreign key constraints

**Migration**:

```sql
-- Add new column for event traceability
ALTER TABLE script_execution
ADD COLUMN triggering_event_id BIGINT UNSIGNED NULL
COMMENT 'Event ID from megazord that triggered this execution (NULL for scheduled)';

-- Make schedule columns nullable
ALTER TABLE script_execution
DROP FOREIGN KEY IF EXISTS fk_script_execution_schedule_id;

ALTER TABLE script_execution
MODIFY COLUMN schedule_id BIGINT UNSIGNED NULL,
MODIFY COLUMN planned TIMESTAMP NULL;

-- Recreate FK with NULL support
ALTER TABLE script_execution
ADD CONSTRAINT fk_script_execution_schedule_id 
  FOREIGN KEY (schedule_id) 
  REFERENCES task_schedule(id);

-- Add index for trigger-based queries
CREATE INDEX idx_script_execution_event_id 
  ON script_execution(triggering_event_id);
```

#### Option 2: Use Sentinel Values (Quick Fix)

**Pros**:

- No schema migration
- Quick to implement for pilot

**Cons**:

- Confusing semantics (schedule_id=0 means "no schedule")
- Need dummy records in `task_schedule` table
- Harder to query

**Implementation**:

```sql
-- Create sentinel schedule record
INSERT INTO task_schedule (id, ...) VALUES (0, ...);

-- Trigger-based executions use:
schedule_id = 0
planned = execution start timestamp

-- Add event column
ALTER TABLE script_execution
ADD COLUMN triggering_event_id BIGINT UNSIGNED NULL;
```

**Recommendation**: **Option 1** for cleaner long-term solution

---

### Phase 3: API Design (tiny-specs)

#### New DTO Schema

**File**: `tiny-specs/specs/micro-manager/UnscheduledScriptExecution.yaml` (create new or extend existing)

```yaml
PostUnscheduledScriptExecutionDTO:
  type: object
  required:
    - scriptReferenceId
    - scriptVersionId
    - triggeringEventId
    - scriptExecutionSteps
  properties:
    scriptReferenceId:
      type: integer
      minimum: 1
      description: ID of the script reference
      example: 123
    scriptVersionId:
      type: integer
      minimum: 1
      description: ID of the script version executed
      example: 456
    triggeringEventId:
      type: integer
      minimum: 1
      description: ID of the incoming event from megazord that triggered this execution
      example: 789012
    scriptExecutionSteps:
      type: array
      description: Steps executed during the unscheduled script run
      items:
        $ref: '#/components/schemas/PutScriptStepExecutionDTO'

UnscheduledScriptExecutionResponse:
  type: object
  properties:
    scriptExecutionId:
      type: integer
      description: Created execution record ID
      example: 999
```

#### API Endpoint

**File**: `tiny-specs/specs/micro-manager/micro-manager.yaml`

```yaml
paths:
  /v6/robots/{robotId}/scripts/unscheduled-execution:
    post:
      tags:
        - Robot APIs
      summary: Record unscheduled script execution (trigger-based)
      operationId: postUnscheduledScriptExecution
      parameters:
        - name: robotId
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PostUnscheduledScriptExecutionDTO'
      responses:
        '201':
          description: Execution recorded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UnscheduledScriptExecutionResponse'
        '400':
          description: Invalid request
        '403':
          description: Forbidden (robot access check failed)
```

---

### Phase 4: Implementation (micro-manager)

#### Step 4.1: Update Schema Definitions (🚧 TODO)

**File**: `src/schemas/body/ScriptExecution.ts`

```typescript
export interface PostUnscheduledScriptExecutionDTO {
  scriptReferenceId: number
  scriptVersionId: number
  triggeringEventId: number  // ✅ NEW
  scriptExecutionSteps: ExecutionStepDTO[]
}

export interface UnscheduledScriptExecutionResponse {
  scriptExecutionId: number
}
```

Update validation schema:

**File**: `src/schemas/validation/UnscheduledScriptExecutionSchema.ts` (create new)

```typescript
import Joi from 'joi'

export const PostUnscheduledScriptExecutionSchema = {
  body: Joi.object({
    scriptReferenceId: Joi.number().integer().min(1).required(),
    scriptVersionId: Joi.number().integer().min(1).required(),
    triggeringEventId: Joi.number().integer().min(1).required(),
    scriptExecutionSteps: Joi.array().items(
      Joi.object({
        scriptStepId: Joi.number().integer().min(1).required(),
        nextScriptStepId: Joi.number().integer().min(1).optional(),
        executedAt: Joi.date().iso().required(),
        stepType: Joi.string().optional(),
        data: Joi.object().optional()
      })
    ).min(1).required()
  })
}
```

---

#### Step 4.2: Update Repository for Unscheduled Executions (🚧 TODO)

**File**: `src/repository/ScriptExecutionRepository.ts`

Add new method for trigger-based executions:

```typescript
export class ScriptExecutionRepository {
  // ... existing code ...

  private ADD_UNSCHEDULED_SCRIPT_EXECUTION = `
    INSERT INTO script_execution 
      (script_reference_id, script_version_id, schedule_id, planned, triggering_event_id) 
    VALUES (?, ?, NULL, NULL, ?)`

  /**
   * Create script execution record for trigger-initiated (unscheduled) script
   * @param robotId - Robot executing the script
   * @param scriptReferenceId - Script reference ID
   * @param scriptVersionId - Script version ID
   * @param triggeringEventId - Incoming event ID that triggered execution
   * @returns OkPacket with insertId
   */
  public addUnscheduledScriptExecution = async ({ 
    robotId, 
    scriptReferenceId, 
    scriptVersionId, 
    triggeringEventId 
  }: { 
    robotId: number
    scriptReferenceId: number
    scriptVersionId: number
    triggeringEventId: number 
  }): Promise<OkPacket> => {
    // Verify script reference belongs to robot
    const sRef = await this.scriptRepository.getScriptReference({ 
      id: scriptReferenceId, 
      robotId 
    })
    
    if (!sRef || sRef.length === 0) {
      throw new Error(`Script reference ${scriptReferenceId} not found for robot ${robotId}`)
    }

    const dbRes = await this.database.generalQuery<any>(
      this.ADD_UNSCHEDULED_SCRIPT_EXECUTION, 
      [sRef[0].id, scriptVersionId, triggeringEventId]
    )
    
    return dbRes as OkPacket
  }

  // Optional: Add getter for unscheduled executions
  private GET_UNSCHEDULED_EXECUTIONS = `
    SELECT se.id, se.script_reference_id, se.script_version_id, 
           se.triggering_event_id, se.created_at
    FROM script_execution se
    WHERE se.triggering_event_id IS NOT NULL
      AND se.triggering_event_id = ?`

  public getUnscheduledExecution = async (triggeringEventId: number) => {
    return this.database.generalQuery<any>(
      this.GET_UNSCHEDULED_EXECUTIONS, 
      [triggeringEventId]
    )
  }
}
```

---

#### Step 4.3: Update Service Layer (🚧 TODO)

**File**: `src/services/ScriptExecutionService.ts`

Add method to handle unscheduled execution persistence:

```typescript
export class ScriptExecutionService {
  // ... existing constructor and methods ...

  /**
   * Save trigger-initiated (unscheduled) script execution
   * @param robotId - Robot ID
   * @param scriptReferenceId - Script reference ID
   * @param scriptVersionId - Script version ID
   * @param triggeringEventId - Event ID that triggered execution
   * @param scriptExecutionSteps - Steps executed
   * @returns Created execution ID
   */
  public async saveUnscheduledExecution({ 
    robotId, 
    scriptReferenceId, 
    scriptVersionId,
    triggeringEventId,
    scriptExecutionSteps 
  }: {
    robotId: number
    scriptReferenceId: number
    scriptVersionId: number
    triggeringEventId: number
    scriptExecutionSteps: ExecutionStepDTO[]
  }): Promise<number> {
    // Validate execution steps
    await this.validateExecutionSteps(
      scriptExecutionSteps, 
      robotId, 
      scriptReferenceId, 
      scriptVersionId
    )

    // Create execution record
    const okPacket = await this.scriptExecutionRepository.addUnscheduledScriptExecution({ 
      robotId, 
      scriptReferenceId, 
      scriptVersionId, 
      triggeringEventId 
    })

    const scriptExecutionId = okPacket.insertId

    if (!scriptExecutionId) {
      throw new Error('Failed to create unscheduled script execution')
    }

    // Add execution steps
    const executionSteps = scriptExecutionSteps.map(step => ({ 
      ...step, 
      scriptExecutionId 
    }))

    // Handle report steps
    for (const step of executionSteps) {
      if (step.stepType === 'report') {
        // Use current timestamp since there's no planned time
        step.data = await this.reportingService.report(
          robotId, 
          scriptExecutionId, 
          scriptVersionId, 
          new Date(), 
          step
        )
      }
    }

    await this.scriptExecutionRepository.addScriptExecutionSteps(executionSteps)

    return scriptExecutionId
  }
}
```

---

#### Step 4.4: Create Controller (🚧 TODO)

**File**: `src/controllers/UnscheduledScriptExecutionController.ts` (create new)

```typescript
import { Request, Response } from 'express'
import { ScriptExecutionService } from '../services/ScriptExecutionService'
import { PostUnscheduledScriptExecutionDTO } from '../schemas/body/ScriptExecution'

export class UnscheduledScriptExecutionController {
  constructor(private scriptExecutionService: ScriptExecutionService) {}

  public postUnscheduledExecution = async (req: Request, res: Response) => {
    try {
      const robotId = parseInt(req.params.robotId)
      const body: PostUnscheduledScriptExecutionDTO = req.body

      const scriptExecutionId = await this.scriptExecutionService.saveUnscheduledExecution({
        robotId,
        scriptReferenceId: body.scriptReferenceId,
        scriptVersionId: body.scriptVersionId,
        triggeringEventId: body.triggeringEventId,
        scriptExecutionSteps: body.scriptExecutionSteps
      })

      res.status(201).json({ scriptExecutionId })
    } catch (error) {
      console.error('Failed to save unscheduled execution', error)
      
      if (error.message?.includes('not found')) {
        return res.status(404).json({ error: error.message })
      }
      
      res.status(500).json({ error: 'Internal server error' })
    }
  }
}
```

---

#### Step 4.5: Add Route (🚧 TODO)

**File**: `src/routes/routes.ts`

```typescript
import { UnscheduledScriptExecutionController } from '../controllers/UnscheduledScriptExecutionController'
import { PostUnscheduledScriptExecutionSchema } from '../schemas/validation/UnscheduledScriptExecutionSchema'
import { validate } from '../middleware/validation'
import { checkRobotAccess } from '../middleware/validation/checkRobotAccess'

// In route setup function:
const unscheduledExecutionController = container.resolve('unscheduledScriptExecutionController')

router.post(
  '/v6/robots/:robotId/scripts/unscheduled-execution',
  validate(PostUnscheduledScriptExecutionSchema),
  checkRobotAccess,  // Ensure robot identity matches :robotId
  unscheduledExecutionController.postUnscheduledExecution
)
```

---

#### Step 4.6: Register in DI Container (🚧 TODO)

**File**: `src/buildContainer.ts`

```typescript
import { UnscheduledScriptExecutionController } from './controllers/UnscheduledScriptExecutionController'

container.register({
  // ... existing registrations ...
  
  unscheduledScriptExecutionController: asClass(UnscheduledScriptExecutionController).singleton()
})
```

---

#### Step 4.7: Update OpenAPI Documentation (🚧 TODO)

**File**: `docs/micro-manager.yaml`

Add the new endpoint and schemas:

```yaml
paths:
  /v6/robots/{robotId}/scripts/unscheduled-execution:
    post:
      tags:
        - Robot v6
      summary: Record unscheduled script execution
      description: |
        Store execution data for scripts triggered by events (not schedules).
        Captures step-by-step execution with timestamp and links to triggering event.
      operationId: postUnscheduledScriptExecution
      parameters:
        - name: robotId
          in: path
          required: true
          schema:
            type: integer
            minimum: 1
        - $ref: '#/components/parameters/KongHeaders'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PostUnscheduledScriptExecutionDTO'
            example:
              scriptReferenceId: 123
              scriptVersionId: 456
              triggeringEventId: 789012
              scriptExecutionSteps:
                - scriptStepId: 10
                  executedAt: "2025-11-26T10:30:00Z"
                  stepType: "command"
                - scriptStepId: 11
                  nextScriptStepId: 12
                  executedAt: "2025-11-26T10:30:15Z"
                  stepType: "closedQuestion"
                  data:
                    answer: "yes"
                    probability: 0.95
                    dataType: "closedQuestionData"
      responses:
        '201':
          description: Execution recorded successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UnscheduledScriptExecutionResponse'
        '400':
          description: Invalid request body
        '403':
          description: Robot access forbidden
        '404':
          description: Script reference not found

components:
  schemas:
    PostUnscheduledScriptExecutionDTO:
      $ref: './schemas/UnscheduledScriptExecution.yaml#/PostUnscheduledScriptExecutionDTO'
    UnscheduledScriptExecutionResponse:
      $ref: './schemas/UnscheduledScriptExecution.yaml#/UnscheduledScriptExecutionResponse'
```

**File**: `docs/schemas/UnscheduledScriptExecution.yaml` (create new)

```yaml
PostUnscheduledScriptExecutionDTO:
  type: object
  required:
    - scriptReferenceId
    - scriptVersionId
    - triggeringEventId
    - scriptExecutionSteps
  properties:
    scriptReferenceId:
      type: integer
      minimum: 1
      description: ID of the script reference
      example: 123
    scriptVersionId:
      type: integer
      minimum: 1
      description: ID of the script version executed
      example: 456
    triggeringEventId:
      type: integer
      minimum: 1
      description: ID of the incoming event from megazord that triggered this execution
      example: 789012
    scriptExecutionSteps:
      type: array
      description: Steps executed during the unscheduled script run
      items:
        $ref: 'ScriptStepExecution.yaml/#/PutScriptStepExecutionDTO'

UnscheduledScriptExecutionResponse:
  type: object
  properties:
    scriptExecutionId:
      type: integer
      description: Created execution record ID
      example: 999
      minimum: 1
```

---

### Phase 5: Testing Strategy

**Unit Tests**:

**File**: `test/services/ScriptExecutionService.UT.spec.ts`

```typescript
describe('ScriptExecutionService - Unscheduled Execution', () => {
  it('should save unscheduled execution with triggering event ID', async () => {
    const mockRepository = mock(ScriptExecutionRepository)
    const mockStepsRepo = mock(StepsRepository)
    const mockReporting = mock(ReportingService)
    
    when(mockRepository.addUnscheduledScriptExecution(anything()))
      .thenResolve({ insertId: 999 } as OkPacket)
    
    when(mockStepsRepo.getScriptStepIds(anything()))
      .thenResolve([10, 11, 12])
    
    const service = new ScriptExecutionService(
      instance(mockRepository),
      instance(mockStepsRepo),
      instance(mockReporting)
    )
    
    const executionId = await service.saveUnscheduledExecution({
      robotId: 1,
      scriptReferenceId: 123,
      scriptVersionId: 456,
      triggeringEventId: 789012,
      scriptExecutionSteps: [
        { scriptStepId: 10, executedAt: new Date() }
      ]
    })
    
    expect(executionId).toBe(999)
    verify(mockRepository.addUnscheduledScriptExecution(anything())).once()
  })
})
```

**Integration Tests**:

**File**: `test/controllers/UnscheduledScriptExecutionController.IT.spec.ts`

```typescript
describe('POST /v6/robots/:robotId/scripts/unscheduled-execution', () => {
  it('should create execution record and return ID', async () => {
    const response = await request(app)
      .post('/v6/robots/1/scripts/unscheduled-execution')
      .send({
        scriptReferenceId: 123,
        scriptVersionId: 456,
        triggeringEventId: 789012,
        scriptExecutionSteps: [
          {
            scriptStepId: 10,
            executedAt: '2025-11-26T10:30:00Z'
          }
        ]
      })
      .expect(201)
    
    expect(response.body.scriptExecutionId).toBeGreaterThan(0)
  })
  
  it('should return 404 if script reference not found', async () => {
    await request(app)
      .post('/v6/robots/1/scripts/unscheduled-execution')
      .send({
        scriptReferenceId: 99999,  // Non-existent
        scriptVersionId: 456,
        triggeringEventId: 789012,
        scriptExecutionSteps: []
      })
      .expect(404)
  })
})
```

**Repository Integration Tests**:

**File**: `test/IT/repositoryIT/ScriptExecutionRepository.IT.spec.ts`

```typescript
describe('ScriptExecutionRepository - Unscheduled', () => {
  it('should insert unscheduled execution with NULL schedule_id', async () => {
    const repo = new ScriptExecutionRepository(database, scriptRepo)
    
    const result = await repo.addUnscheduledScriptExecution({
      robotId: 1,
      scriptReferenceId: 123,
      scriptVersionId: 456,
      triggeringEventId: 789012
    })
    
    expect(result.insertId).toBeGreaterThan(0)
    
    // Verify database record
    const [row] = await database.query(
      'SELECT * FROM script_execution WHERE id = ?', 
      [result.insertId]
    )
    
    expect(row.schedule_id).toBeNull()
    expect(row.planned).toBeNull()
    expect(row.triggering_event_id).toBe(789012)
  })
})
```

---

## 📋 Implementation Tasks Breakdown (Option 1: Nullable Columns)

### Task 1: Database Schema Migration & Repository Foundation

**Priority**: High | **Estimated Effort**: 2-3 days | **Dependencies**: None

**Scope**:

- Create and test database migration script
- Update `ScriptExecutionRepository` with new methods for unscheduled executions
- Add unit tests for repository layer

**Deliverables**:

1. Migration SQL file: `migrations/YYYYMMDD_add_unscheduled_execution_support.sql`
   - Make `schedule_id`, `planned` nullable
   - Add `triggering_event_id` column with index
   - Drop/recreate FK constraints
2. Repository method: `addUnscheduledScriptExecution()`
3. Repository method: `getUnscheduledExecution()` (optional, for queries)
4. Unit tests: `test/IT/repositoryIT/ScriptExecutionRepository.IT.spec.ts`
   - Test NULL schedule_id insertion
   - Test triggering_event_id persistence
   - Test backward compatibility with scheduled executions

**Acceptance Criteria**:

- ✅ Migration runs successfully on dev database
- ✅ Can insert execution with `schedule_id=NULL`, `planned=NULL`, `triggering_event_id=123`
- ✅ Existing scheduled executions remain queryable
- ✅ Repository tests pass with 100% coverage

**Rollback Plan**: Revert migration (restore NOT NULL constraints, drop triggering_event_id column)

---

### Task 2: API Contract Definition (tiny-specs)

**Priority**: High | **Estimated Effort**: 1-2 days | **Dependencies**: None (can run parallel with Task 1)

**Scope**:

- Define OpenAPI specification for unscheduled execution endpoint
- Create JSON schemas and examples
- Publish to tiny-specs repository

**Deliverables**:

1. OpenAPI endpoint: `POST /v6/robots/{robotId}/scripts/unscheduled-execution`
2. Schema file: `specs/micro-manager/UnscheduledScriptExecution.yaml`
   - `PostUnscheduledScriptExecutionDTO` with required fields
   - `UnscheduledScriptExecutionResponse`
3. Update main spec: `specs/micro-manager/micro-manager.yaml`
4. Example payloads with documentation

**Acceptance Criteria**:

- ✅ OpenAPI spec validates without errors
- ✅ All required fields documented (scriptReferenceId, scriptVersionId, triggeringEventId, scriptExecutionSteps)
- ✅ Response schema includes scriptExecutionId
- ✅ Kong authentication headers specified
- ✅ Error responses documented (400, 403, 404, 500)

**Review Checklist**:

- Schema matches existing `PutScriptStepExecutionDTO` structure for steps
- triggeringEventId is integer (matches incoming_event.id from megazord)
- Robot access control documented

---

### Task 3: Service & Controller Implementation

**Priority**: High | **Estimated Effort**: 3-4 days | **Dependencies**: Task 1 (repository), Task 2 (API contract)

**Scope**:

- Implement service layer business logic
- Create controller and route
- Add validation schemas
- Register in DI container
- Write unit tests

**Deliverables**:

1. Service method: `ScriptExecutionService.saveUnscheduledExecution()`
   - Validate execution steps
   - Create execution record with triggering event ID
   - Handle report steps
   - Add execution steps to database
2. Controller: `UnscheduledScriptExecutionController.postUnscheduledExecution()`
   - Request validation
   - Error handling
   - Response formatting
3. Validation: `src/schemas/validation/UnscheduledScriptExecutionSchema.ts`
4. Route: `/v6/robots/:robotId/scripts/unscheduled-execution`
5. DI registration in `buildContainer.ts`
6. Unit tests:
   - `test/services/ScriptExecutionService.UT.spec.ts`
   - `test/controllers/UnscheduledScriptExecutionController.IT.spec.ts`

**Acceptance Criteria**:

- ✅ Service validates step IDs belong to script version
- ✅ Service creates execution record with NULL schedule/planned
- ✅ Service stores triggering_event_id correctly
- ✅ Report steps processed (calls ReportingService)
- ✅ Controller enforces robot access control (checkRobotAccess)
- ✅ Returns 201 with scriptExecutionId on success
- ✅ Returns 404 if script reference not found
- ✅ Returns 403 if robot access denied
- ✅ All tests pass with >90% coverage

**Error Scenarios to Handle**:

- Invalid scriptReferenceId (not found or doesn't belong to robot)
- Invalid scriptStepId (not in script version)
- Missing required fields
- Database errors (transaction rollback)

---

### Task 4: Integration Testing & Documentation

**Priority**: Medium | **Estimated Effort**: 2 days | **Dependencies**: Task 3 (implementation complete)

**Scope**:

- End-to-end integration tests
- Update micro-manager OpenAPI docs
- Write deployment guide
- Verify backward compatibility

**Deliverables**:

1. Integration tests: Full API flow from request to database
   - Test with valid trigger execution
   - Test with multiple steps including report
   - Test error cases (404, 403, 400)
   - Test concurrent executions
2. OpenAPI documentation: `docs/micro-manager.yaml`
   - Add endpoint specification
   - Add schema references
   - Add examples
3. Update `docs/schemas/UnscheduledScriptExecution.yaml`
4. Deployment guide: Document migration steps
5. Backward compatibility verification:
   - Scheduled executions still work
   - Existing queries unaffected

**Acceptance Criteria**:

- ✅ Integration tests cover happy path and error cases
- ✅ Can retrieve stored executions via existing GET endpoints
- ✅ OpenAPI docs render correctly in Swagger UI
- ✅ Migration can be applied to staging environment
- ✅ Scheduled execution flow unchanged (regression test)
- ✅ Performance impact < 50ms per execution

**Pre-Production Checklist**:

- [ ] Database migration tested on staging
- [ ] API contract reviewed by robot team
- [ ] Error handling covers all edge cases
- [ ] Logging includes triggeringEventId for traceability
- [ ] Monitoring alerts configured for new endpoint
- [ ] Rollback procedure documented

---

## 📊 Summary of Results

> Implementation not started. Will update after completion.

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Critical Decisions Required

- [ ] **Decision 1: Schema Migration Strategy**
  - **Question**: Use nullable columns (Option 1) or sentinel values (Option 2)?
  - **Recommendation**: **Option 1** (nullable) for clean semantics
  - **Impact**: Requires migration script, FK constraint changes
  - **Timeline**: Migration can run during low-traffic window

- [ ] **Decision 2: Event ID Source**
  - **Question**: Where does `triggeringEventId` come from in robot's API call?
  - **Clarification Needed**: Does robot currently receive event ID when script is triggered?
  - **Alternative**: If robot doesn't have event ID, need to add it to trigger payload from `m-o-triggers`
  - **Impact**: May require changes in `m-o-triggers` service

- [ ] **Decision 3: API Version**
  - **Question**: Use v6 (new) or extend existing v5 endpoint?
  - **Recommendation**: **v6** for clear separation and easier rollback
  - **Impact**: Robot firmware may need update to call new endpoint

- [ ] **Decision 4: Backward Compatibility Period**
  - **Question**: How long to support old behavior (no persistence for trigger scripts)?
  - **Recommendation**: Make new endpoint opt-in initially, then migrate robots gradually
  - **Impact**: Deployment strategy, robot firmware rollout

### ⚠️ Clarifications Needed

- [ ] **Clarification 1: Robot Firmware Changes**
  - **Question**: Do robots need firmware update to call new endpoint? Or is this server-side only?
  - **Impact**: Deployment timeline, coordination with robot team

- [ ] **Clarification 2: Event ID Propagation**
  - **Question**: Full event flow: `megazord-events` → `m-o-triggers` → robot → `micro-manager`
  - **Current State**: Verify if event ID is already passed to robot when trigger fires
  - **If Missing**: Need to update trigger payload structure in `m-o-triggers`

- [ ] **Clarification 3: Query/Reporting Requirements**
  - **Question**: Will there be new reports/queries filtering by trigger-based executions?
  - **Example**: "Show all executions triggered by event type X in last 7 days"
  - **Impact**: May need additional indexes on `triggering_event_id`

- [ ] **Clarification 4: Execution Completeness**
  - **Question**: Do trigger-based scripts always run to completion? Or can they be partial?
  - **Impact**: How to mark execution as "complete" vs "in-progress" (currently inferred from steps)

- [ ] **Clarification 5: Duplicate Prevention**
  - **Question**: Can same trigger event cause multiple executions? Need deduplication?
  - **Impact**: May need unique constraint on `(script_reference_id, triggering_event_id)`
