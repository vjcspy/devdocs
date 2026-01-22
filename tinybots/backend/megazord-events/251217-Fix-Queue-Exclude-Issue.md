# 📋 251217 - Fix Queue Field Serialization Issue

## References

- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/domains/SubscriptionDomain.ts` - Domain model with problematic `@Exclude()` decorator
- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/services/EventSubscriptionService.ts` - Service using queue field for message routing
- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/controllers/EventSubscriptionsController.ts` - Controller exposing subscription endpoints
- `/Users/kai/work/tinybots/devdocs/tinybots/backend/megazord-events/251210-Queue-Per-Subscription.md` - Original refactor plan
- `/Users/kai/work/tinybots/devdocs/tinybots/backend/megazord-events/OVERVIEW.md` - Repository overview

## User Requirements

After refactoring according to `devdocs/tinybots/megazord-events/251210-Queue-Per-Subscription.md`, the `@Exclude()` decorator was added to the `queue` field in `SubscriptionDomain` to prevent it from being returned in API responses. However, this decorator also prevents the field from being populated when querying from the database, breaking the queue routing logic in `EventSubscriptionsService.broadcastToSubscriptionHandler()`.

**Current Problem:**
```typescript
// SubscriptionDomain.ts
@IsString()
@IsOptional()
@Exclude()  // ❌ This prevents queue from being populated from DB queries
queue?: string
```

**Expected Behavior:**
- Queue field should NOT be exposed in external API responses (GET endpoints)
- Queue field MUST be populated from database for internal business logic
- Queue routing in `broadcastToSubscriptionHandler()` must work correctly

## 🎯 Objective

Fix the `queue` field serialization strategy in `SubscriptionDomain` to:
1. Prevent exposure in external API responses (maintain API contract)
2. Allow internal business logic to access the queue value from database queries
3. Ensure proper queue routing in SQS message publishing

### ⚠️ Key Considerations

1. **Class-Transformer Context**: `@Exclude()` applies to both serialization (to JSON) and deserialization (from DB). We need the field excluded only during serialization to client, NOT during internal DB hydration.

2. **Transform Groups**: `class-transformer` supports transformation groups. We can use `@Exclude({ toPlainOnly: true })` to exclude only when transforming TO plain objects (API responses), but allow transformation FROM plain objects (DB results).

3. **API Contract**: The `queue` field was intentionally hidden from API responses to prevent clients from seeing queue implementation details. This must remain hidden.

4. **Internal Access**: Services like `EventSubscriptionsService.broadcastToSubscriptionHandler()` rely on `subscription.queue` to route messages to the correct SQS queue.

5. **Alternative Approaches**:
   - **Option A**: Use `@Exclude({ toPlainOnly: true })` - Excludes only when serializing to JSON (API response), allows DB hydration
   - **Option B**: Remove `@Exclude()` and handle exclusion at controller/route level using response DTOs
   - **Option C**: Keep `@Exclude()` and manually access queue from raw DB result before transformation

6. **Recommended Approach**: Option A (`@Exclude({ toPlainOnly: true })`) is cleanest - it preserves the domain model integrity while controlling serialization context.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze current decorator usage and transformation flow
  - **Outcome**: `@Exclude()` without options blocks both serialization and deserialization
  - **Outcome**: Repository queries transform DB rows to `SubscriptionDomain` using `class-transformer`, causing queue to be undefined
  
- [x] Review class-transformer documentation for contextual exclusion
  - **Outcome**: `@Exclude({ toPlainOnly: true })` excludes field only when calling `instanceToPlain()` or `classToPlain()`
  - **Outcome**: `plainToClass()` transformations (from DB) will still populate the field
  
- [x] Identify all code paths that access `subscription.queue`
  - **Outcome**: Primary usage in `EventSubscriptionsService.broadcastToSubscriptionHandler()` at line 190: `queue: subscription.queue || 'undefined'`
  - **Outcome**: Also used in `notify()` method for queue address construction
  
- [x] Verify API endpoints that return `SubscriptionDomain`
  - **Outcome**: Need to verify if controllers use `class-transformer` serialization groups
  - **Outcome**: Check if any GET endpoints currently return subscription objects

### Phase 2: Implementation (File/Code Structure)

```text
megazord-events/
├── src/
│   ├── models/
│   │   └── domains/
│   │       └── SubscriptionDomain.ts             # 🔄 IN PROGRESS - Fix @Exclude decorator
│   └── services/
│       └── EventSubscriptionService.ts           # ✅ NO CHANGES - Already logs queue value
└── test/
    ├── services/
    │   └── EventSubscriptionServiceTest.ts       # 🚧 TODO - Verify queue routing works
    └── controllers/
        └── EventSubscriptionControllerIT.ts      # 🚧 TODO - Verify queue not exposed in API
```

### Phase 3: Detailed Implementation Steps

#### Step 3.1: Fix Queue Field Decorator in SubscriptionDomain

**Task:** Update `@Exclude()` decorator to use `toPlainOnly` option, allowing DB hydration while preventing API exposure

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/domains/SubscriptionDomain.ts`

**Current Code (Lines 32-36):**
```typescript
@IsString()
@IsOptional()
@Exclude()
queue?: string
```

**Updated Code:**
```typescript
@IsString()
@IsOptional()
@Exclude({ toPlainOnly: true })  // Exclude only when serializing to JSON (API responses)
queue?: string
```

**Rationale:**
- `toPlainOnly: true` means exclude this field ONLY when transforming instance → plain object (JSON response)
- When transforming plain object → instance (DB query → domain), field will be populated
- This allows internal business logic to access `subscription.queue` while hiding it from API consumers

**Validation:**
- Verify `class-transformer` import includes `Exclude` with proper typing
- Confirm no TypeScript errors after change
- Ensure domain still extends `BaseDomain` correctly

#### Step 3.2: Verify Repository Query Transformation

**Task:** Ensure repository correctly transforms DB results to domain with queue field populated

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/repositories/EventSubscriptionsRepository.ts`

**Verification Points:**
- Confirm `GET_ALL` query includes `ES.QUEUE queue` in SELECT (line 69)
- Confirm `GET_BY_ID` query includes `ES.QUEUE queue` in SELECT (line 48)
- Verify repository uses `Repository.mappers` to transform results
- Check that base `Repository` class uses `plainToClass()` or similar for hydration

**Expected Behavior:**
- After fixing decorator, `subscription.queue` should be populated from DB results
- No code changes needed in repository if SELECT clauses already include QUEUE column

**Action:** Read repository to confirm, no changes expected

#### Step 3.3: Add Unit Test for Queue Field Hydration

**Task:** Create unit test verifying queue field is correctly populated from repository

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/test/services/EventSubscriptionServiceTest.ts`

**Test Case:** "should populate queue field from database when broadcasting events"

**Test Logic:**
```typescript
test('should populate queue field from database when broadcasting events', async () => {
  // Arrange: Create subscription with queue suffix
  const subscription = await eventSubscriptionsRepository.create({
    robotId: 123,
    eventTypeIds: [1],
    subscriptionType: SubscriptionType.SERVICE_SUBSCRIPTION,
    queue: 'test-queue-suffix'
  })
  
  // Act: Retrieve subscription
  const retrieved = await eventSubscriptionsRepository.getById(subscription)
  
  // Assert: Queue field should be populated
  expect(retrieved?.queue).toBe('test-queue-suffix')
})
```

**Expected Result:** Test passes, confirming queue is hydrated from DB

#### Step 3.4: Add Integration Test for API Response Exclusion

**Task:** Verify queue field is NOT exposed in API responses despite being populated internally

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/test/controllers/EventSubscriptionControllerIT.ts`

**Test Case:** "should not expose queue field in subscription API responses"

**Test Logic:**
```typescript
test('should not expose queue field in subscription GET response', async () => {
  // Arrange: Create subscription with queue
  const createResponse = await request(app)
    .post(`/internal/v1/events/robots/${robotId}/subscriptions`)
    .send({
      eventNames: ['TOILET_ACTIVITY'],
      queue: 'sensitive-queue-name',
      isTriggerSubscription: false
    })
  
  const subscriptionId = createResponse.body.id
  
  // Act: GET subscription (if such endpoint exists)
  const getResponse = await request(app)
    .get(`/internal/v1/events/robots/${robotId}/subscriptions/${subscriptionId}`)
  
  // Assert: Queue should NOT be in response
  expect(getResponse.body).not.toHaveProperty('queue')
  expect(getResponse.body).toHaveProperty('id')
})
```

**Note:** Verify if GET endpoint exists. If not, test may need to check CREATE response instead or be skipped.

**Expected Result:** Queue field is excluded from JSON response

#### Step 3.5: Verify Queue Routing in Event Broadcasting

**Task:** Confirm existing integration tests validate correct queue routing behavior

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/test/controllers/EventSubscriptionControllerIT.ts`

**Existing Test Reference:** According to `251210-Queue-Per-Subscription.md`, queue routing tests were already added in PR #49

**Verification:**
- Locate existing tests that verify messages sent to `${statusQueue.address}-{queue}`
- Run tests to confirm they pass with the decorator fix
- Check test logs for `queue: subscription.queue || 'undefined'` output

**Expected Behavior:**
- Tests should now pass if they were failing due to queue being undefined
- SQS mock/spy should receive messages at correct queue addresses
- Logs should show actual queue value instead of "undefined"

#### Step 3.6: Manual Testing Checklist

**Task:** Perform manual verification in development environment

**Steps:**
1. **Create subscription with queue suffix**
   ```bash
   curl -X POST http://localhost:3000/internal/v1/events/robots/123/subscriptions \
     -H "Content-Type: application/json" \
     -d '{
       "eventNames": ["TOILET_ACTIVITY"],
       "queue": "test-jobs",
       "isTriggerSubscription": false
     }'
   ```

2. **Trigger incoming event for the robot**
   ```bash
   curl -X POST http://localhost:3000/internal/v1/events/robots/123/incomings \
     -H "Content-Type: application/json" \
     -d '{
       "eventName": "TOILET_ACTIVITY",
       "providerId": 1,
       "rawEvent": {}
     }'
   ```

3. **Check application logs**
   - Should see: `queue: test-jobs` (NOT "undefined")
   - Should see SQS publish to `${statusQueueAddress}-test-jobs`

4. **Verify SQS queues** (if using LocalStack or AWS)
   ```bash
   aws sqs receive-message --queue-url <queue-url> --region us-east-1
   ```
   - Message should appear in suffixed queue

5. **Check API response** (if GET endpoint exists)
   - Response should NOT include `queue` field

**Expected Results:**
- ✅ Queue routing works correctly
- ✅ Messages sent to proper queue addresses
- ✅ Queue field hidden from API responses
- ✅ No errors in logs

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- [List major accomplishments]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Alternative Solutions Considered

**Option B: Response DTOs**
- Create separate response DTOs for API endpoints
- Map domain → response DTO in controllers
- More explicit but requires additional classes

**Option C: Controller-Level Transformation Groups**
- Use `class-transformer` groups in controller serialization
- Apply `@Exclude({ groups: ['external'] })` to queue field
- Pass `{ groups: ['external'] }` when serializing responses

**Recommendation:** Stick with Option A (`toPlainOnly: true`) unless API response control needs to be more granular across multiple fields.

### 📝 Future Considerations

1. **Response DTO Pattern**: Consider adopting response DTOs for all API endpoints to completely decouple internal domains from external contracts

2. **Serialization Strategy**: Document class-transformer usage patterns in `devdocs/tinybots/megazord-events/OVERVIEW.md` for future reference

3. **Validation**: Ensure `queue` field validation (alphanumeric + hyphens, ≤128 chars) still works correctly in CreateSubscriptionDto

4. **Migration Path**: Existing subscriptions with NULL queue should continue working (already tested in original refactor)
