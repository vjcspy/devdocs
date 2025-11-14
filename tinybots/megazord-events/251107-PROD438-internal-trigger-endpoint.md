# 📋 [PROD-438: 2025-11-07] - Create Internal Trigger Subscription Endpoints

## 🎯 Objective
> Create internal endpoints for trigger subscriptions that can be called by the Sensara adaptor, allowing trigger subscriptions to be managed without requiring admin authentication. Implement validation to ensure trigger subscriptions contain only one event per subscription and prevent duplicate active trigger subscriptions for the same event.

**Context**:
- External endpoints already exist: POST/GET/DELETE `/v1/events/robots/{robotId}/subscriptions/triggers`
- These require admin authentication and M_O_TRIGGERS_SETTING_WRITE_ALL permission
- Sensara adaptor needs to create trigger subscriptions without admin overhead
- Need to enforce business rule: **1 event per trigger subscription** (enforced at DTO level via string instead of array)
- Need to prevent duplicate active trigger subscriptions for the same event on a robot
- Validation applies to **BOTH internal and external endpoints** (shared service logic)

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze existing external trigger endpoints implementation
  - **Outcome**: Located in `EventSubscriptionController`:
    - `createTriggerSubscription`: Uses `CreateTriggerSubscriptionDto`, converts to array
    - `getTriggerSubscriptions`: Filters by `SubscriptionType.TRIGGER_SUBSCRIPTION`
    - `deleteSubscription`: Shared with internal endpoint (generic delete)
  - **Key finding**: Currently allows multiple events (array structure), no duplicate validation

- [x] Analyze service layer implementation
  - **Outcome**: `EventSubscriptionsService.subscribe()` handles both types:
    - Sets `subscriptionType` based on `isTriggerSubscription` flag
    - Creates subscription with multiple `eventTypeIds`
    - No current validation for duplicate active trigger subscriptions

- [x] Define scope and edge cases
  - **Outcome**: Edge cases to handle:
    1. ✅ Single event enforcement → Handled by DTO change (string instead of array)
    2. ✅ Active trigger subscription exists for event → Reject with 409 Conflict
    3. ✅ Inactive trigger subscription exists for event → Allow new subscription
    4. ✅ Robot has no trigger subscriptions → Allow creation
    5. ✅ Event name doesn't exist → Return 404 (existing behavior)
    6. ✅ Empty/invalid eventName → Validation error 400 (DTO validation)

### Phase 2: Implementation (File/Code Structure)

```
src/
├── controllers/
│   └── EventSubscriptionsController.ts          # 🔄 UPDATE - Modify existing methods
├── services/
│   └── EventSubscriptionService.ts              # 🚧 TODO - Add duplicate validation
├── models/
│   └── DTOs/
│       ├── CreateTriggerSubscriptionDto.ts      # 🔄 UPDATE - Change array to string
│       └── index.ts                              # ✅ EXISTING - Already exports DTO
├── cmd/
│   └── app/
│       └── main.ts                               # 🚧 TODO - Add 3 internal routes
test/
├── controllers/
│   └── EventSubscriptionControllerIT.ts         # 🚧 TODO - Update & add tests
└── services/
    └── EventSubscriptionServiceTest.ts          # 🚧 TODO - Add validation tests
```

### Phase 3: Implementation Steps

#### Step 3.1: Update CreateTriggerSubscriptionDto 🚧 TODO
**File**: `src/models/DTOs/CreateTriggerSubscriptionDto.ts`

**Change eventNames from array to string**:
```typescript
// BEFORE (current)
export class CreateTriggerSubscriptionDto {
  @Expose()
  @IsArray()
  @ArrayUnique()
  @IsString({ each: true })
  eventNames!: string[]
}

// AFTER (new)
export class CreateTriggerSubscriptionDto {
  @Expose()
  @IsString()
  @IsNotEmpty()
  eventName!: string  // Changed from eventNames array to eventName string
}
```

**Rationale**:
- Enforces single-event rule at DTO level (compile-time + runtime validation)
- No need for service-layer validation of array length
- Simpler API contract for consumers
- Type-safe: impossible to send multiple events

#### Step 3.2: Update Existing Controller Method � UPDATE
**File**: `src/controllers/EventSubscriptionsController.ts`

**Modify `createTriggerSubscription` method** to handle new DTO structure:

```typescript
public createTriggerSubscription = async (
  req: Request,
  res: Response
): Promise<void> => {
  const ctx = getContext(req)
  const logger = Logger.loggerFromCtx(ctx)

  const body = req.body as DTOs.CreateTriggerSubscriptionDto
  const robotReq = req.params as unknown as DTOs.RobotIdParamDto

  logger.info('create a new trigger subscription', {
    body,
    robotReq
  })

  // Convert single eventName to array for service layer
  const createSubscriptionDto: DTOs.CreateSubscriptionDto = {
    eventNames: [body.eventName],  // Changed: wrap string in array
    isTriggerSubscription: true
  }

  const subscription = await this.eventSubscriptionsService.subscribe(
    ctx,
    robotReq,
    createSubscriptionDto
  )

  res.status(201).json(subscription)
}
```

**Note**: `getTriggerSubscriptions` and `deleteSubscription` remain unchanged.

#### Step 3.3: Add Duplicate Validation to Service Layer 🚧 TODO
**File**: `src/services/EventSubscriptionService.ts`

**Changes needed**:
1. Import error class: `import { ConflictError } from 'ts-http-errors'`
2. Add validation in `subscribe()` method when `isTriggerSubscription === true`

**Validation logic** (add BEFORE creating subscription):
```typescript
async subscribe(
  ctx: IRequestContext,
  robotReq: DTOs.RobotIdParamDto,
  req: DTOs.CreateSubscriptionDto
): Promise<domains.SubscriptionDomain> {
  const eventTypeIds: number[] = req.eventNames.map(eventName =>
    this.eventSchemasService.getByName(ctx, eventName).id
  )

  const subscriptionType = req.isTriggerSubscription
    ? domains.SubscriptionType.TRIGGER_SUBSCRIPTION
    : domains.SubscriptionType.SERVICE_SUBSCRIPTION

  // NEW: Validate no duplicate active trigger subscriptions
  if (req.isTriggerSubscription) {
    const existingSubscriptions = await this.eventSubscriptionsRepository.getAll({
      robotId: robotReq.robotId,
      eventTypeId: eventTypeIds[0],
      subscriptionType: domains.SubscriptionType.TRIGGER_SUBSCRIPTION,
      isActive: true
    })

    if (existingSubscriptions.length > 0) {
      throw new ConflictError(
        `Active trigger subscription already exists for event '${req.eventNames[0]}' on robot ${robotReq.robotId}`
      )
    }
  }

  // Existing code continues...
  const subscriptionId = await this.eventSubscriptionsRepository.create({
    eventTypeIds,
    robotId: robotReq.robotId,
    subscriptionType
  })
  // ...
}
```

**Note**: Single-event rule is now enforced by DTO, no validation needed here.

#### Step 3.4: Register Internal Routes 🚧 TODO
**File**: `src/cmd/app/main.ts`

**Add routes in `setEndpoints()` method** (after existing internal endpoints, before external trigger endpoints):

```typescript
#### Step 3.4: Register Internal Routes 🚧 TODO
**File**: `src/cmd/app/main.ts`

**Add routes in `setEndpoints()` method** (after existing internal endpoints, before external trigger endpoints):

```typescript
// Internal trigger subscription endpoints (for Sensara adaptor)
this.app.post(
  '/internal/v1/events/robots/:robotId/subscriptions/triggers',
  ValidationMiddleware.pathValidator(DTOs.RobotIdParamDto),
  ValidationMiddleware.bodyValidator(DTOs.CreateTriggerSubscriptionDto),
  asyncHandler(subscriptionController.createTriggerSubscription)  // REUSE existing method!
)

this.app.get(
  '/internal/v1/events/robots/:robotId/subscriptions/triggers',
  ValidationMiddleware.pathValidator(DTOs.RobotIdParamDto),
  asyncHandler(subscriptionController.getTriggerSubscriptions)  // REUSE existing method!
)

this.app.delete(
  '/internal/v1/events/robots/:robotId/subscriptions/triggers/:subscriptionId',
  ValidationMiddleware.pathValidator(DTOs.RobotIdAndSubscriptionIdParamDto),
  asyncHandler(subscriptionController.deleteSubscription)  // REUSE existing method!
)
```

**Key differences from external endpoints**:
- ❌ No `useAdminValidator` middleware
- ❌ No `usePermissionValidator` middleware
- ✅ Same controller methods (100% reuse - no separate internal methods needed!)
- ✅ Same validation for path params and body
- ✅ Same business logic through service layer
- ✅ Same validation (duplicate detection applies to both)

#### Step 3.5: Update Integration Tests 🚧 TODO
```

**Key differences from external endpoints**:
- ❌ No `useAdminValidator`
- ❌ No `usePermissionValidator`
- ✅ Same validation for path params and body
- ✅ Same business logic through service layer

#### Step 3.5: Update Integration Tests 🚧 TODO
**File**: `test/controllers/EventSubscriptionControllerIT.ts`

**Update existing external endpoint tests** (they now validate duplicate detection):
```typescript
describe('POST /v1/events/robots/:robotId/subscriptions/triggers', () => {
  // EXISTING TESTS - Update expectations
  it('should create a trigger subscription')  // Update to use eventName (string)
  it('should throw BadRequest if eventName is not string')  // Update from array validation
  it('should throw NotFound if eventName does not exist')

  // NEW TESTS - Add duplicate validation
  it('should throw Conflict when active trigger subscription exists for same event')
  it('should allow creation when inactive trigger subscription exists for same event')
  it('should allow creation of different trigger subscriptions on same robot')
})
```

**Add new internal endpoint tests**:
```typescript
describe('POST /internal/v1/events/robots/:robotId/subscriptions/triggers', () => {
  // Setup: Mock sensara adaptor, create event schemas

  it('should create internal trigger subscription with single event')
  it('should throw BadRequest when eventName is empty')
  it('should throw BadRequest when eventName is not string')
  it('should throw Conflict when active trigger subscription exists for event')
  it('should allow creation when inactive trigger subscription exists')
  it('should throw NotFound when event name does not exist')
  it('should NOT require authentication headers')  // Key difference from external
})

describe('GET /internal/v1/events/robots/:robotId/subscriptions/triggers', () => {
  it('should return only trigger subscriptions for robot')
  it('should return empty array when no trigger subscriptions')
  it('should NOT require authentication headers')
})

describe('DELETE /internal/v1/events/robots/:robotId/subscriptions/triggers/:subscriptionId', () => {
  it('should deactivate internal trigger subscription')
  it('should be idempotent (allow repeated deletes)')
  it('should throw NotFound when subscription does not exist')
  it('should throw NotFound when subscription belongs to different robot')
  it('should NOT require authentication headers')
})
```

#### Step 3.6: Add Service Unit Tests 🚧 TODO
**File**: `test/services/EventSubscriptionServiceTest.ts`

**Add tests**:
```typescript
describe('subscribe() - trigger duplicate validation', () => {
  it('should reject trigger subscription when active subscription exists for same event')
  it('should allow trigger subscription when inactive subscription exists for same event')
  it('should allow trigger subscription when no existing subscription')
  it('should allow multiple trigger subscriptions for different events on same robot')
  it('should not validate duplicates for regular subscriptions (allow multiple)')
})
```

**Note**: No need to test single-event validation in service - that's enforced by DTO type system.

### Phase 4: Integration with External Services 🔮 FUTURE

**Tiny Internal Services** (Separate repository - not in scope):
- Add internal trigger endpoints to service interface
- Document in API contract

**Tiny Internal Service Mocks** (Separate repository - not in scope):
- Add mock responses for 3 internal endpoints
- Include validation errors (400, 409) in mock scenarios

## 📊 Summary of Results

### ✅ Completed Achievements
- [x] Analysis of existing trigger endpoints implementation
- [x] Analysis of subscription service architecture
- [x] Identified validation requirements and edge cases
- [ ] DTO update: Change eventNames array to eventName string
- [ ] Controller update: Modify createTriggerSubscription to use new DTO
- [ ] Service layer validation for duplicate prevention
- [ ] Three internal route registrations (reusing existing controller methods)
- [ ] Update existing external endpoint tests for new DTO structure
- [ ] Add integration tests for internal endpoints
- [ ] Add unit tests for duplicate validation logic

### 🔧 Technical Decisions

1. **Single Event Enforcement**:
   - ✅ **DTO Level**: Changed `eventNames: string[]` to `eventName: string`
   - ✅ Type-safe: Impossible to pass multiple events (compile-time + runtime)
   - ✅ Simpler API: More intuitive for consumers
   - ❌ Not service layer validation (redundant with type system)

2. **Controller Method Reuse**:
   - ✅ **100% Reuse**: All three methods (POST/GET/DELETE) shared between internal & external
   - ✅ Only difference: Middleware stack (auth vs no-auth)
   - ✅ Maintains DRY principle
   - ✅ Single source of truth for business logic

3. **Validation Location**:
   - ✅ **Single event**: DTO level (type system enforcement)
   - ✅ **Duplicate detection**: Service layer (applies to both internal/external)
   - ✅ Shared validation ensures consistency across endpoints

4. **Error Handling**:
   - `400 BadRequest`: Invalid eventName (empty, wrong type) - DTO validation
   - `409 Conflict`: Active trigger subscription exists for same event - Service validation
   - `404 NotFound`: Event name doesn't exist - Service validation (existing)

5. **Route Pattern**:
   - Matches external pattern with `/internal/` prefix
   - Maintains RESTful structure
   - `/internal/v1/events/robots/:robotId/subscriptions/triggers`
   - Same endpoints, different middleware

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues
- [ ] Need to verify error class imports
  - ConflictError: Check if available in `ts-http-errors` or `tiny-backend-tools`
  - Import pattern: `import { ConflictError } from 'ts-http-errors'`
  - BadRequestError not needed (DTO validation handles it)

- [ ] Breaking change: External API contract changes
  - Old: `{ "eventNames": ["ACTIVITY"] }`
  - New: `{ "eventName": "ACTIVITY" }`
  - **Impact**: Existing consumers must update their requests
  - **Mitigation**: Coordinate with teams using external endpoint (if any exist)

### 🔮 Future Improvements
- [ ] Add metrics/logging for duplicate subscription attempts
- [ ] Consider soft validation mode for migration scenarios
- [ ] Add endpoint to list all subscriptions (including inactive) for debugging
- [ ] Database index on (robot_id, event_type_id, subscription_type, is_active) for performance
- [ ] API documentation in Swagger/OpenAPI format

### 📝 External Repository Updates (Separate Tickets)
- [ ] **tiny-internal-services**: Add 3 internal trigger endpoints to service interface
- [ ] **tiny-internal-service-mocks**: Add mock implementations with validation
- [ ] **sensara-adaptor**: Integrate with new internal endpoints

---
**Last updated**: 2025-11-07 (Revised)
**Status**: 🚧 Implementation Plan Ready - Aligned with 3 key changes
**Key Changes**:
1. ✅ DTO enforces single event (string instead of array)
2. ✅ Controller methods 100% reused (no separate internal methods)
3. ✅ Duplicate validation applies to both internal AND external endpoints
---