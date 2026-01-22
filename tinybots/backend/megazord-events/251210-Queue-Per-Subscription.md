# 📋 251210 - Queue Per Subscription Support

## References

- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/services/EventSubscriptionService.ts` - Main subscription service handling event fan-out
- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/DTOs/CreateSubscriptionDto.ts` - Subscription creation DTO
- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/domains/SubscriptionDomain.ts` - Subscription domain model
- `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/SQSConfig.ts` - SQS configuration models
- `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/config/custom-environment-variables.json` - Jobs consumer configuration
- `/Users/kai/work/tinybots/tinybots/backend/tiny-internal-services/lib/services/EventService.ts` - Shared library event service
- `/Users/kai/work/tinybots/devdocs/tinybots/OVERVIEW.md` - Global TinyBots architecture
- `/Users/kai/work/tinybots/devdocs/tinybots/backend/megazord-events/OVERVIEW.md` - Megazord events service documentation

## User Requirements

Currently both `azi-3-status-check` and `azi-3-status-check-jobs` consume from the same SQS queue when `megazord-events` sends messages. This causes unnecessary processing and potential conflicts.

**Requirement:** Extend the subscription API to accept an optional queue suffix parameter, allowing services to specify a queue suffix for their subscription:

```json
{
  "eventNames": ["TOILET_ACTIVITY"],
  "queue": "azi-3-status-check-jobs"
}
```

**Queue Address Construction:** 
- If `subscription.queue` is provided: `address = "${statusQueueConfig.address}-${subscription.queue}"`
- If `subscription.queue` is NOT provided: `address = "${statusQueueConfig.address}"` (default behavior)

**Implementation Scope:** Only `azi-3-status-check-jobs` will be updated to use the queue suffix. `azi-3-status-check` remains unchanged and continues using the default queue without suffix.

## 🎯 Objective

Enable service-specific queue routing in megazord-events subscription system by accepting an optional queue suffix parameter. When provided, messages will be sent to `${statusQueueConfig.address}-${queue}` instead of the default `${statusQueueConfig.address}`, allowing multiple services to subscribe to the same events but receive notifications on different SQS queues.

### ⚠️ Key Considerations

1. **Queue Address Construction:** When `subscription.queue` is provided, construct the target queue as `${statusQueueConfig.address}-${subscription.queue}`
2. **Backward Compatibility:** Existing subscriptions and consumers (like `azi-3-status-check`) continue working without modification using default queue
3. **Limited Scope:** Only `azi-3-status-check-jobs` will be updated to use the queue suffix. `azi-3-status-check` remains unchanged
4. **Database Schema:** New column `queue` (nullable, varchar) in `event_subscription` table to store queue suffix
5. **Validation:** Queue suffix should be validated as alphanumeric with hyphens (used as suffix in queue name)
6. **Testing:** Must verify both default queue (no suffix) and custom queue (with suffix) behaviors

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze current subscription flow in `EventSubscriptionService`
  - **Outcome:** Identified `notify()` method uses `statusQueueConfig.address` for all SERVICE_SUBSCRIPTION types
  - **Outcome:** Need to modify this to construct `${statusQueueConfig.address}-${subscription.queue}` when queue is provided
  
- [x] Review database schema for `event_subscription` table
  - **Outcome:** Need to add `queue` column (nullable varchar) to store queue suffix
  - **Outcome:** Repository layer needs method to persist and retrieve queue suffix
  
- [x] Identify subscription consumers
  - **Outcome:** `azi-3-status-check` - NO CHANGES, continues using default queue
  - **Outcome:** `azi-3-status-check-jobs` - WILL BE UPDATED to pass queue suffix "azi-3-status-check-jobs"
  
- [x] Define scope and edge cases
  - **Scope:** 
    1. `megazord-events` - Add queue suffix support in DTO, domain, repository, service
    2. `azi-3-status-check-jobs` - Pass queue suffix when creating subscriptions  
    3. `tiny-internal-services` - Update EventService to support optional queue parameter
    4. `tiny-internal-services-mocks` - Update if mocks need queue parameter support
    5. `azi-3-status-check` - NO CHANGES (explicitly out of scope)
  - **Edge Cases to Handle:**
    1. Subscription created without queue → use default `statusQueueConfig.address`
    2. Subscription created with empty/whitespace queue → validation error
    3. Queue suffix validation (alphanumeric + hyphens only)
    4. SQS send failure to constructed queue → log error with full queue address
    5. Migration of existing subscriptions → all existing rows have NULL queue, treated as default
    6. Multiple subscriptions to same event with different queues → both receive messages at their respective addresses

### Phase 2: Implementation (File/Code Structure)

#### Repository: `megazord-events`

```text
megazord-events/
├── src/
│   ├── models/
│   │   ├── DTOs/
│   │   │   └── CreateSubscriptionDto.ts          # 🔄 IN PROGRESS - Add optional queue field
│   │   ├── domains/
│   │   │   └── SubscriptionDomain.ts             # 🔄 IN PROGRESS - Add queue property
│   │   └── SQSConfig.ts                          # ✅ IMPLEMENTED - Already has StatusQueueConfig
│   ├── repositories/
│   │   └── EventSubscriptionsRepository.ts       # 🚧 TODO - Add queue parameter to create/update
│   ├── services/
│   │   └── EventSubscriptionService.ts           # 🔄 IN PROGRESS - Update notify() to construct queue address
│   └── controllers/
│       └── EventSubscriptionsController.ts       # ✅ IMPLEMENTED - No changes needed (passes DTO through)
├── test/
│   ├── controllers/
│   │   └── EventSubscriptionControllerIT.ts      # 🚧 TODO - Add tests for queue parameter
│   └── services/
│       └── EventSubscriptionServiceTest.ts       # 🚧 TODO - Add unit tests for queue address construction
└── schemas/
    └── migrations/                                # 🚧 TODO - Add migration for queue column
```

#### Repository: `azi-3-status-check-jobs`

```text
azi-3-status-check-jobs/
├── src/
│   ├── infrastructure/
│   │   └── MegazordEventClient.ts                # 🚧 TODO - Add queue param to subscribe() method
│   └── services/
│       └── MonitoringScheduler.ts                # 🚧 TODO - Update initializeSessionForRule() to pass queue="azi-3-status-check-jobs"
└── config/
    ├── default.json                              # 🚧 TODO - Add queue name config (optional, can be hardcoded)
    └── custom-environment-variables.json         # 🚧 TODO - Map QUEUE_NAME env var (optional)
```

#### Repository: `tiny-internal-services`

```text
tiny-internal-services/
└── lib/
    └── services/
        └── EventService.ts                       # 🚧 TODO - Add optional queue parameter to createSubscription()
```

#### Repository: `tiny-internal-services-mocks` (if needed)

```text
tiny-internal-services-mocks/
└── lib/
    └── services/
        └── EventService.mock.ts                  # 🚧 TODO - Update mock to accept queue parameter
```

**Note:** `azi-3-status-check` is explicitly OUT OF SCOPE and requires no changes.

### Phase 3: Detailed Implementation Steps

#### Step 3.1: Database Schema Migration (typ-e repository)

**Task:** Create new migration script in `typ-e` repository to add `queue` column to `event_subscription` table

**Repository:** `typ-e` (database migration repository)

**File to Create:** Create new migration file following typ-e naming convention (e.g., `V123__add_queue_to_event_subscription.sql`)

**Migration Script:**

```sql
-- Migration: Add queue column to event_subscription
-- Purpose: Enable queue suffix routing for subscription-specific SQS queues
ALTER TABLE event_subscription 
ADD COLUMN queue VARCHAR(128) NULL 
COMMENT 'Optional queue suffix for SQS routing. When provided, messages are sent to ${statusQueueConfig.address}-${queue}. NULL uses default queue.';

-- Add index for potential query optimization
CREATE INDEX idx_event_subscription_queue ON event_subscription(queue);
```

**Rollback Script (if needed):**

```sql
-- Rollback: Remove queue column from event_subscription
DROP INDEX idx_event_subscription_queue ON event_subscription;
ALTER TABLE event_subscription DROP COLUMN queue;
```

**Validation:**

- Migration script follows typ-e repository conventions
- Migration runs successfully on dev environment
- Existing subscriptions have NULL queue (backward compatible)
- Column accepts both NULL and valid queue suffixes
- Index is created successfully

#### Step 3.2: Extend DTOs and Domain Models (megazord-events)

**Task:** Add queue suffix support to subscription data structures

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/DTOs/CreateSubscriptionDto.ts`

**Changes:**

```typescript
import { Expose, Type } from 'class-transformer'
import {
  ArrayUnique,
  IsArray,
  IsBoolean,
  IsDate,
  IsOptional,
  IsString,
  MaxLength,
  Matches
} from 'class-validator'

export class CreateSubscriptionDto {
  @Expose()
  @IsArray()
  @ArrayUnique()
  @IsString({ each: true })
  eventNames!: string[]

  @Expose()
  @IsBoolean()
  @IsOptional()
  isTriggerSubscription?: boolean

  @Expose()
  @IsDate()
  @IsOptional()
  @Type(() => Date)
  until?: Date

  @Expose()
  @IsString()
  @IsOptional()
  @MaxLength(128)
  @Matches(/^[a-zA-Z0-9-]+$/, {
    message: 'Queue must contain only alphanumeric characters and hyphens'
  })
  queue?: string  // Queue suffix, will be appended to statusQueueConfig.address
}
```

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/models/domains/SubscriptionDomain.ts`

**Changes:**

```typescript
export class SubscriptionDomain extends BaseDomain {
  @IsNumber()
  @Expose()
  id!: number

  @IsNumber()
  @Expose()
  robotId!: number

  @Expose()
  subscriptionDetails!: SubscriptionDetailDomain[]

  @IsBoolean()
  @Expose()
  @Transform(({ value }) => !!value)
  isActive!: boolean

  @IsNumber()
  @Expose()
  subscriptionType!: SubscriptionType

  @IsString()
  @IsOptional()
  @Expose()
  queue?: string  // NEW FIELD - queue suffix

  @IsDate()
  @Expose()
  createdAt!: Date

  @IsDate()
  @Expose()
  updatedAt!: Date

  private eventSchemas?: IReadonlyEventSchemasMap

  provide(eventSchemas: IReadonlyEventSchemasMap) {
    this.eventSchemas = eventSchemas
    this.subscriptionDetails.forEach(detail => {
      const schema = this.eventSchemas?.get(detail.eventTypeId)
      if (schema) {
        detail.provide(schema)
      }
    })
  }
}
```

**Validation:**

- DTOs compile without errors
- Validation decorators correctly enforce queue format (alphanumeric + hyphens)
- Domain model includes new field with proper serialization

#### Step 3.3: Update Repository Layer (megazord-events)

**Task:** Add queue suffix parameter support in repository operations

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/repositories/EventSubscriptionsRepository.ts`

**Changes:**

1. Update `create()` method signature to accept optional `queue`
2. Update INSERT SQL to include `queue` column
3. Update `getAll()` and `getById()` methods to SELECT `queue`
4. Update result mapping to hydrate `queue` property

**Pseudo-code:**

```typescript
interface ICreateSubscriptionRequest {
  eventTypeIds: number[]
  robotId: number
  subscriptionType: SubscriptionType
  queue?: string  // NEW - queue suffix
}

async create(req: ICreateSubscriptionRequest): Promise<number> {
  // INSERT with queue column
  const sql = `
    INSERT INTO event_subscription 
      (robot_id, subscription_type, is_active, queue, created_at, updated_at)
    VALUES (?, ?, 1, ?, NOW(), NOW())
  `
  // ... execute with req.queue
}

async getAll(filters: ...): Promise<SubscriptionDomain[]> {
  // SELECT queue in query
  const sql = `
    SELECT 
      es.id, es.robot_id, es.subscription_type, 
      es.is_active, es.queue, 
      es.created_at, es.updated_at
    FROM event_subscription es
    WHERE ...
  `
  // ... map results including queue
}
```

**Validation:**

- Repository methods compile and type-check correctly
- SQL queries include new column
- Existing tests pass with NULL queue values

#### Step 3.4: Update Service Layer (megazord-events)

**Task:** Implement queue address construction logic based on subscription queue suffix

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/src/services/EventSubscriptionService.ts`

**Changes:**

1. **Update `subscribe()` method** to pass queue suffix to repository:

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

  // Validate trigger subscription conflicts (existing logic)
  if (req.isTriggerSubscription) {
    const existingSubscriptions = await this.eventSubscriptionsRepository
      .getAll({
        robotId: robotReq.robotId,
        eventTypeId: eventTypeIds[0],
        subscriptionType: domains.SubscriptionType.TRIGGER_SUBSCRIPTION,
        isActive: true
      })

    if (existingSubscriptions.length > 0) {
      throw new ConflictError(
        `Active trigger subscription already exists for event '${
          req.eventNames[0]
        }' on robot ${robotReq.robotId}`
      )
    }
  }

  // NEW: Pass queue suffix to repository
  const subscriptionId = await this.eventSubscriptionsRepository.create({
    eventTypeIds,
    robotId: robotReq.robotId,
    subscriptionType,
    queue: req.queue  // NEW - pass queue suffix
  })

  // ... rest of method unchanged
}
```

2. **Update `broadcastToSubscriptionHandler()` method** to pass subscription to notify:

```typescript
private async broadcastToSubscriptionHandler(
  ctx: IRequestContext,
  incomingEvent: domains.IncomingEventDomain,
  subscription: domains.SubscriptionDomain
): Promise<void> {
  const logger = Logger.loggerFromCtx(ctx)

  const outgoingEvent = await this.outgoingEventsService.create(
    ctx,
    incomingEvent.id,
    subscription.id
  )

  outgoingEvent.provide(undefined, undefined, incomingEvent)

  logger.info('outgoing event created', {
    outgoingEvent,
    subscription,
    subscriptionId: subscription.id,
    incomingEvent: incomingEvent.id,
    queue: subscription.queue || 'default'  // NEW LOG
  })

  switch (subscription.subscriptionType) {
    case SubscriptionType.SERVICE_SUBSCRIPTION:
      // NEW: Pass subscription to notify for queue address construction
      await this.notifySubscription(
        ctx,
        incomingEvent.robotId,
        `/internal/v1/events/robots/${incomingEvent.robotId}/subscriptions/${subscription.id}/outgoings/${outgoingEvent.id}`,
        outgoingEvent,
        subscription  // NEW PARAMETER
      )
      break
    case SubscriptionType.TRIGGER_SUBSCRIPTION:
      await this.triggerService.sendTrigger(incomingEvent, outgoingEvent)
      break
    default:
      // Default case also needs update
      await this.notifySubscription(
        ctx,
        incomingEvent.robotId,
        `/internal/v1/events/robots/${incomingEvent.robotId}/subscriptions/${subscription.id}/outgoings/${outgoingEvent.id}`,
        outgoingEvent,
        subscription  // NEW PARAMETER
      )
  }
}
```

3. **Refactor `notify()` method** to construct queue address based on subscription queue suffix:

```typescript
private async notifySubscription<T extends BaseDomain>(
  ctx: IRequestContext,
  robotId: number,
  link: string,
  payload: T,
  subscription: domains.SubscriptionDomain  // NEW PARAMETER
) {
  const logger = Logger.loggerFromCtx(ctx)
  
  // NEW: Construct queue address based on subscription queue suffix
  let queueAddress = this.statusQueueConfig.address
  if (subscription.queue) {
    queueAddress = `${this.statusQueueConfig.address}-${subscription.queue}`
  }
  
  logger.debug('Sending notification to queue', {
    subscriptionId: subscription.id,
    baseAddress: this.statusQueueConfig.address,
    queueSuffix: subscription.queue,
    queueAddress,
    isCustomQueue: !!subscription.queue
  })

  const message: ISubscriptionMessage<Record<string, unknown>> = {
    from: this.fromAddress,
    to: {
      robotId: `${robotId}`
    },
    link: {
      link,
      rel: ''
    },
    payload: payload.toPlain()
  }

  try {
    await this.subscriptionSQS.send(
      ctx,
      queueAddress,  // NEW: Use constructed queue address
      message
    )
  } catch (error) {
    // NEW: Enhanced error handling for queue-specific failures
    logger.error('Failed to send message to queue', {
      error,
      baseAddress: this.statusQueueConfig.address,
      queueSuffix: subscription.queue,
      queueAddress,
      subscriptionId: subscription.id,
      robotId
    })
    // Re-throw to maintain existing error propagation
    throw error
  }
}
```

**Validation:**

- Service compiles and type-checks correctly
- Queue address correctly constructed: `${statusQueueConfig.address}-${subscription.queue}` when queue exists
- Default queue used when subscription.queue is null/undefined
- Logger includes queue construction details for debugging

#### Step 3.5: Update Consumer Jobs (azi-3-status-check-jobs)

**Task:** Configure jobs service to pass queue suffix "azi-3-status-check-jobs" when creating subscriptions

**Note:** `azi-3-status-check` service is explicitly OUT OF SCOPE and requires NO CHANGES. It will continue using the default queue (no suffix).

**File:** `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/src/infrastructure/MegazordEventClient.ts`

**Changes:**

```typescript
async subscribe(
  robotId: number,
  eventNames: string[],
  until: Date,
  queue?: string  // NEW PARAMETER
): Promise<string> {
  const subscription = await this.eventService.createSubscription(
    robotId,
    eventNames,
    until,
    queue  // NEW PARAMETER - pass through to tiny-internal-services EventService
  )
  return subscription.id.toString()
}
```

**File:** `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/src/services/MonitoringScheduler.ts`

**Changes:**

```typescript
private async initializeSessionForRule(
  robot: MonitoringRobot,
  rule: MonitoringRule,
  startTime: DateTime,
  endTime: DateTime
): Promise<void> {
  const logger = this.logger

  // NEW: Use app name from config as queue suffix
  const queueSuffix = this.config.app.name  // 'azi-3-status-check-jobs'

  logger.info('Initializing session for rule', {
    robotId: robot.robotId,
    ruleId: rule.ruleId,
    eventType: rule.eventType,
    startTime: startTime.toISO(),
    endTime: endTime.toISO(),
    queueSuffix  // NEW LOG
  })

  // Create subscription with queue suffix from app name
  const subscriptionId = await this.subscriptionManager.subscribe(
    robot.robotId,
    [rule.eventType],
    endTime.toJSDate(),
    queueSuffix  // NEW PARAMETER - will create queue address: ${baseAddress}-azi-3-status-check-jobs
  )

  // ... rest of method unchanged
}
```

**File:** `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/config/default.json`

**Changes (if not already present):**

```json
{
  "app": {
    "name": "azi-3-status-check-jobs"
  }
}
```

**Note:** The `app.name` configuration should already exist. We're just using it as the queue suffix.

**Validation:**

- Jobs service creates subscriptions with queue suffix "azi-3-status-check-jobs"
- Messages will be sent to `${statusQueueConfig.address}-azi-3-status-check-jobs`
- Existing monitoring sessions work correctly with new queue routing

#### Step 3.6: Update Shared Library (tiny-internal-services)

**Task:** Update EventService client to support queue parameter

**Note:** This requires changes to the shared library used by consumer services.

**File:** `tiny-internal-services/lib/services/EventService.ts` (approximate location)

**Changes:**
```typescript
async createSubscription(
  robotId: number,
  eventNames: string[],
  until?: Date,
  queue?: string  // NEW PARAMETER
): Promise<SubscriptionDto> {
  const body: any = {
    eventNames,
    until: until?.toISOString()
  }
  
  // NEW: Add queue if provided
  if (queue) {
    body.queue = queue
  }
  
  const response = await this.client.post(
    `/internal/v1/events/robots/${robotId}/subscriptions`,
    body
  )
  
  return response.data
}
```

**Validation:**
- Shared library change is backward compatible
- Both with and without queue parameter work correctly
- Update version and publish to npm/yarn registry

#### Step 3.8: Testing (megazord-events)

**Task:** Add comprehensive tests for queue routing functionality

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/test/controllers/EventSubscriptionControllerIT.ts`

**New Test Cases:**
```typescript
describe('POST /internal/v1/events/robots/:robotId/subscriptions with queue', () => {
  it('should create subscription with custom queue', async () => {
    const body = {
      eventNames: ['TOILET_ACTIVITY'],
      queue: 'http://localstack:4566/000000000000/custom-queue'
    }
    
    const response = await request(app)
      .post(`/internal/v1/events/robots/${robotId}/subscriptions`)
      .send(body)
      .expect(201)
    
    expect(response.body.queueAddress).to.equal(body.queue)
  })
  
  it('should use default queue when queue not specified', async () => {
    const body = {
      eventNames: ['TOILET_ACTIVITY']
    }
    
    const response = await request(app)
      .post(`/internal/v1/events/robots/${robotId}/subscriptions`)
      .send(body)
      .expect(201)
    
    expect(response.body.queueAddress).to.be.undefined
  })
  
  it('should reject invalid queue format', async () => {
    const body = {
      eventNames: ['TOILET_ACTIVITY'],
      queue: 'invalid-queue-format'
    }
    
    await request(app)
      .post(`/internal/v1/events/robots/${robotId}/subscriptions`)
      .send(body)
      .expect(400)
  })
})

describe('Event fan-out with custom queues', () => {
  it('should send message to custom queue when specified', async () => {
    // Setup: Create subscription with custom queue
    const customQueue = 'http://localstack:4566/000000000000/test-queue'
    const subscription = await createSubscription({
      eventNames: ['TEST_EVENT'],
      queue: customQueue
    })
    
    // Setup: Mock SQS client to capture sent messages
    const sentMessages: any[] = []
    sinon.stub(sqsProducer, 'send').callsFake((ctx, queueUrl, message) => {
      sentMessages.push({ queueUrl, message })
      return Promise.resolve()
    })
    
    // Act: Create incoming event that triggers subscription
    await createIncomingEvent({
      robotId: testRobotId,
      eventName: 'TEST_EVENT'
    })
    
    // Assert: Message sent to custom queue, not default
    await waitFor(() => sentMessages.length > 0)
    expect(sentMessages[0].queueUrl).to.equal(customQueue)
    expect(sentMessages[0].queueUrl).to.not.equal(defaultStatusQueue)
  })
  
  it('should send to default queue when subscription has no queue', async () => {
    // Setup: Create subscription without custom queue
    const subscription = await createSubscription({
      eventNames: ['TEST_EVENT']
    })
    
    const sentMessages: any[] = []
    sinon.stub(sqsProducer, 'send').callsFake((ctx, queueUrl, message) => {
      sentMessages.push({ queueUrl, message })
      return Promise.resolve()
    })
    
    // Act: Create incoming event
    await createIncomingEvent({
      robotId: testRobotId,
      eventName: 'TEST_EVENT'
    })
    
    // Assert: Message sent to default status queue
    await waitFor(() => sentMessages.length > 0)
    expect(sentMessages[0].queueUrl).to.equal(defaultStatusQueue)
  })
  
  it('should handle multiple subscriptions with different queues', async () => {
    // Setup: Create two subscriptions for same event, different queues
    const queue1 = 'http://localstack:4566/000000000000/queue-1'
    const queue2 = 'http://localstack:4566/000000000000/queue-2'
    
    await createSubscription({
      eventNames: ['TEST_EVENT'],
      queue: queue1
    })
    
    await createSubscription({
      eventNames: ['TEST_EVENT'],
      queue: queue2
    })
    
    const sentMessages: any[] = []
    sinon.stub(sqsProducer, 'send').callsFake((ctx, queueUrl, message) => {
      sentMessages.push({ queueUrl, message })
      return Promise.resolve()
    })
    
    // Act: Create incoming event
    await createIncomingEvent({
      robotId: testRobotId,
      eventName: 'TEST_EVENT'
    })
    
    // Assert: Two messages sent, one to each queue
    await waitFor(() => sentMessages.length === 2)
    const queueUrls = sentMessages.map(m => m.queueUrl).sort()
    expect(queueUrls).to.deep.equal([queue1, queue2].sort())
  })
})
```

**File:** `/Users/kai/work/tinybots/tinybots/backend/megazord-events/test/services/EventSubscriptionServiceTest.ts`

**New Unit Test Cases:**
```typescript
describe('EventSubscriptionsService - Queue Routing', () => {
  it('should pass queue to repository when creating subscription', async () => {
    const queueAddress = 'http://test-queue'
    const createStub = sinon.stub(repository, 'create').resolves(1)
    
    await service.subscribe(ctx, { robotId: 1 }, {
      eventNames: ['TEST_EVENT'],
      queue: queueAddress
    })
    
    expect(createStub).to.have.been.calledWith(
      sinon.match({ queueAddress })
    )
  })
  
  it('should not pass queue when not specified', async () => {
    const createStub = sinon.stub(repository, 'create').resolves(1)
    
    await service.subscribe(ctx, { robotId: 1 }, {
      eventNames: ['TEST_EVENT']
    })
    
    expect(createStub).to.have.been.calledWith(
      sinon.match({ queueAddress: undefined })
    )
  })
})
```

**Validation:**
- All new tests pass
- Existing tests continue to pass
- Test coverage remains above thresholds (95% statements/functions/lines)

#### Step 3.7: Environment Configuration

**Task:** Update deployment configurations for queue suffix support

**Note:** With the queue suffix approach, we only need ONE base queue address. The suffix is appended dynamically.

**Files to Update:**

1. **devtools/docker-compose.yaml** - Update queue environment variables:

```yaml
megazord-events:
  # ... existing config
  environment:
    # ... existing env vars
    STATUS_QUEUE_ADDRESS: http://localstack:4566/000000000000/status-queue

azi-3-status-check:
  # ... existing config (NO CHANGES)
  environment:
    # ... existing env vars
    STATUS_QUEUE_ADDRESS: http://localstack:4566/000000000000/status-queue
    # NO QUEUE_NAME/QUEUE_SUFFIX - uses default queue

azi-3-status-check-jobs:
  # ... existing config
  environment:
    # ... existing env vars
    STATUS_QUEUE_ADDRESS: http://localstack:4566/000000000000/status-queue
    # Queue suffix is hardcoded in the service code as "azi-3-status-check-jobs"
    # Messages will be sent to: http://localstack:4566/000000000000/status-queue-azi-3-status-check-jobs
```

2. **localstack initialization** - Create queues with suffix pattern:

**File:** `/Users/kai/work/tinybots/devtools/tinybots/local/localstack/init-queues.sh` (update or create)

```bash
#!/bin/bash
awslocal sqs create-queue --queue-name status-queue
awslocal sqs create-queue --queue-name status-queue-azi-3-status-check-jobs
echo "SQS queues created"
```

3. **Production/Staging Environment Variables** - Document in deployment guides:

```bash
# megazord-events
STATUS_QUEUE_ADDRESS=https://sqs.eu-central-1.amazonaws.com/ACCOUNT_ID/status-queue

# azi-3-status-check (NO CHANGES - uses default queue)
STATUS_QUEUE_ADDRESS=https://sqs.eu-central-1.amazonaws.com/ACCOUNT_ID/status-queue

# azi-3-status-check-jobs
STATUS_QUEUE_ADDRESS=https://sqs.eu-central-1.amazonaws.com/ACCOUNT_ID/status-queue
# Messages will be routed to: status-queue-azi-3-status-check-jobs (created automatically by megazord-events appending suffix)
```

**AWS Queue Creation:**

```bash
# Production
aws sqs create-queue --queue-name status-queue
aws sqs create-queue --queue-name status-queue-azi-3-status-check-jobs

# Staging
aws sqs create-queue --queue-name status-queue-staging
aws sqs create-queue --queue-name status-queue-staging-azi-3-status-check-jobs
```

**Validation:**

- Local development uses suffix-based queue routing
- azi-3-status-check consumes from base queue (no suffix)
- azi-3-status-check-jobs consumes from queue with suffix
- Configuration is simpler with only base address needed

#### Step 3.8: Documentation Updates

**Task:** Update service documentation to reflect queue suffix feature

**File:** `/Users/kai/work/tinybots/devdocs/tinybots/backend/megazord-events/OVERVIEW.md`

**Section to Add/Update:**

```markdown
## Subscription Queue Routing (Updated Dec 2025)

### Overview

Service subscriptions support optional queue suffix routing. Consumers can specify a queue suffix parameter, and megazord-events will construct the target queue address as `${statusQueueConfig.address}-${queue}`.

### API Changes

`POST /internal/v1/events/robots/:robotId/subscriptions` now accepts an optional `queue` parameter:

```json
{
  "eventNames": ["TOILET_ACTIVITY", "MOTION_DETECTED"],
  "queue": "azi-3-status-check-jobs",
  "until": "2025-12-31T23:59:59Z"
}
```

### Behavior

- **With queue suffix:** Messages are sent to `${statusQueueConfig.address}-${queue}`
  - Example: base address `http://sqs/status-queue`, suffix `azi-3-status-check-jobs` → `http://sqs/status-queue-azi-3-status-check-jobs`
- **Without queue suffix:** Messages use the default `statusQueueConfig.address`
- **Validation:** Queue suffix must contain only alphanumeric characters and hyphens
- **Backward Compatibility:** All existing subscriptions (NULL queue) continue using default queue

### Database Schema

- New column: `event_subscription.queue` (VARCHAR(128), nullable)
- Stores queue suffix (not full address)
- Existing subscriptions have NULL, treated as default queue
- Index: `idx_event_subscription_queue` for potential query optimization

### Consumer Implementation

Only `azi-3-status-check-jobs` currently uses queue suffix. `azi-3-status-check` continues using the default queue.

**Example (azi-3-status-check-jobs):**

```typescript
const subscription = await megazordEventClient.subscribe(
  robotId,
  ['TOILET_ACTIVITY'],
  pollingUntil,
  'azi-3-status-check-jobs'  // Queue suffix
)
// Messages will be sent to: ${statusQueueConfig.address}-azi-3-status-check-jobs
```

### Migration Notes

- Database migration adds nullable column, no data migration needed
- Existing subscriptions continue working with default queue
- New subscriptions can optionally specify queue suffix
- No breaking changes to API contracts
```

**File:** `/Users/kai/work/tinybots/devdocs/tinybots/backend/azi-3-status-check-jobs/OVERVIEW.md`

**Section to Add:**

```markdown
## Queue Configuration (Updated Dec 2025)

This service now specifies a queue suffix when creating Megazord event subscriptions:

- **Queue Suffix:** Hardcoded as `"azi-3-status-check-jobs"` in `MonitoringScheduler.initializeSessionForRule()`
- **Target Queue:** Messages are sent to `${baseQueueAddress}-azi-3-status-check-jobs`
- **Message Consumption:** Consumes from `STATUS_QUEUE_ADDRESS` environment variable (which should point to the suffixed queue)

This prevents message collision with `azi-3-status-check` service that subscribes to the same event types but uses the default queue without suffix.
```

**Validation:**

- Documentation accurately reflects queue suffix construction logic
- Examples show correct queue address pattern
- Migration notes cover all scenarios

**File:** `/Users/kai/work/tinybots/devdocs/tinybots/backend/azi-3-status-check-jobs/OVERVIEW.md`

**Section to Add:**

```markdown
## Queue Configuration (Updated Dec 2025)

This service now specifies its dedicated queue when creating Megazord event subscriptions:

- **Queue Name:** Configured via `QUEUE_NAME` environment variable (default: `azi-3-status-check-jobs`)
- **Subscription Creation:** `initializeSessionForRule()` passes queue name to Megazord Events
- **Message Consumption:** Continues consuming from `STATUS_QUEUE_ADDRESS` as configured

This prevents message collision with other services that subscribe to the same event types.
```

**Validation:**
- Documentation is clear and accurate
- Examples reflect actual implementation
- Migration notes cover all edge cases

### Phase 4: Integration & Validation

#### Step 4.1: Local Development Testing

**Tasks:**
1. Start localstack with separate queues
2. Start megazord-events service
3. Start azi-3-status-check service
4. Start azi-3-status-check-jobs service
5. Create robot event that triggers both subscriptions
6. Verify messages arrive in correct queues
7. Verify no cross-contamination

**Commands:**
```bash
# Start infrastructure
cd devtools
just start-db
docker-compose up -d localstack

# Create queues
docker-compose exec localstack sh -c '
  awslocal sqs create-queue --queue-name status-queue
  awslocal sqs create-queue --queue-name status-queue-jobs
'

# Run services
just dev-megazord-events
just dev-azi-3-status-check
just dev-azi-3-status-check-jobs

# Test: Create robot event and monitor queue consumption
```

**Validation Criteria:**
- ✅ Each service successfully starts with new configuration
- ✅ Services create subscriptions with queue parameter
- ✅ Messages route to correct queues
- ✅ No messages in wrong queues
- ✅ Logs show correct queue routing

#### Step 4.2: Integration Test Suite

**Tasks:**
1. Run all megazord-events tests
2. Run all azi-3-status-check tests
3. Run all azi-3-status-check-jobs tests
4. Verify coverage thresholds maintained

**Commands:**
```bash
cd megazord-events && yarn test
cd azi-3-status-check && yarn test
cd azi-3-status-check-jobs && yarn test
```

**Validation Criteria:**
- ✅ All existing tests pass
- ✅ All new tests pass
- ✅ Coverage ≥ 95% statements/functions/lines
- ✅ Coverage ≥ 70% branches
- ✅ No eslint violations

#### Step 4.3: Backward Compatibility Verification

**Tasks:**
1. Create subscription WITHOUT queue parameter
2. Verify it uses default queue
3. Verify message routing works
4. Verify existing consumers (if any without queue param) still work

**Test Scenarios:**
```bash
# Scenario 1: Legacy subscription (no queue)
curl -X POST http://megazord:8080/internal/v1/events/robots/1/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"eventNames": ["TEST_EVENT"]}'

# Expected: Subscription created, queueAddress is null, messages go to default queue

# Scenario 2: New subscription (with queue)
curl -X POST http://megazord:8080/internal/v1/events/robots/1/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"eventNames": ["TEST_EVENT"], "queue": "http://localstack:4566/000000000000/custom"}'

# Expected: Subscription created, queueAddress is custom, messages go to custom queue

# Scenario 3: Mixed subscriptions
# Create both types, trigger event, verify proper routing
```

**Validation Criteria:**
- ✅ Subscriptions without queue parameter work as before
- ✅ Default queue receives messages from legacy subscriptions
- ✅ Custom queues receive messages from new subscriptions
- ✅ No breaking changes to API contracts

#### Step 4.4: Error Handling & Edge Cases

**Tasks:**
1. Test invalid queue URL format
2. Test empty queue string
3. Test SQS send failure to custom queue
4. Test subscription with non-existent queue (if applicable)

**Test Scenarios:**
```bash
# Invalid queue format
curl -X POST ... -d '{"eventNames": ["TEST"], "queue": "not-a-url"}'
# Expected: 400 Bad Request with validation error

# Empty queue
curl -X POST ... -d '{"eventNames": ["TEST"], "queue": ""}'
# Expected: 400 Bad Request or treated as no queue

# Queue send failure (simulate by stopping localstack)
# Expected: Error logged, outgoing event created, SQS error propagated
```

**Validation Criteria:**
- ✅ Invalid queue formats rejected with clear error messages
- ✅ SQS failures logged with sufficient context
- ✅ Failures don't corrupt subscription state
- ✅ Error responses follow existing API patterns

### Phase 5: Deployment Preparation

#### Step 5.1: Database Migration

**Tasks:**
1. Test migration on local dev database
2. Prepare rollback script
3. Document migration steps for production

**Migration Script (Flyway/SQL):**
```sql
-- Up migration
ALTER TABLE event_subscription 
ADD COLUMN queue_address VARCHAR(512) NULL 
COMMENT 'Optional SQS queue URL for this subscription. NULL uses default status queue.';

CREATE INDEX idx_event_subscription_queue ON event_subscription(queue_address);

-- Rollback script (if needed)
-- DROP INDEX idx_event_subscription_queue ON event_subscription;
-- ALTER TABLE event_subscription DROP COLUMN queue_address;
```

**Validation:**
- ✅ Migration runs without errors
- ✅ Existing data preserved (queue_address is NULL)
- ✅ Rollback script tested
- ✅ Migration documented

#### Step 5.2: Infrastructure Setup

**Tasks:**

1. Create SQS queues in AWS with suffix pattern (per environment)
2. Configure queue policies/permissions
3. Update environment variables in deployment configs

**AWS Setup (per environment):**

```bash
# Production
aws sqs create-queue --queue-name status-queue
aws sqs create-queue --queue-name status-queue-azi-3-status-check-jobs

# Staging
aws sqs create-queue --queue-name status-queue-staging
aws sqs create-queue --queue-name status-queue-staging-azi-3-status-check-jobs

# Update IAM policies to grant services access to queues
```

**Validation:**

- ✅ Queues created in all environments with correct naming pattern
- ✅ IAM permissions configured correctly
- ✅ Queue URLs documented in deployment configs
- ✅ Service accounts can send/receive from queues

#### Step 5.3: Deployment Plan

**Sequence:**

1. Deploy database migration (typ-e/wonkers-db repository) - adds `queue` column
2. Deploy `tiny-internal-services` with updated EventService (optional queue parameter)
3. Deploy `megazord-events` with queue suffix support
4. Deploy `azi-3-status-check-jobs` with queue suffix parameter
5. Monitor queue separation and message routing
6. **DO NOT DEPLOY** `azi-3-status-check` (no changes needed)

**Rollback Plan:**

1. Revert service deployments to previous versions
2. Database column remains (nullable, not used) - safe to keep
3. Services fall back to default queue behavior
4. Optional: Rollback database migration if necessary

**Validation:**

- ✅ Deployment sequence documented
- ✅ Rollback plan tested
- ✅ Monitoring alerts configured for queue with suffix
- ✅ Deployment runbook created

## 📊 Summary of Results

> This section will be populated after implementation and testing are complete.

### ✅ Completed Achievements

- TBD after implementation

### 📈 Performance Metrics

- TBD after deployment

### 🐛 Known Issues & Future Work

- TBD during implementation

## 🚧 Outstanding Issues & Follow-up

### Key Changes in This Revision

1. **Queue Address Construction:** Changed from full queue URL to suffix-based approach
   - `address = "${statusQueueConfig.address}-${subscription.queue}"` when queue is provided
   - `address = "${statusQueueConfig.address}"` when queue is not provided

2. **Scope Reduction:** Only modifying these repositories:
   - `typ-e` - Create database migration script for `queue` column
   - `megazord-events` - Add queue suffix support
   - `azi-3-status-check-jobs` - Pass queue suffix from `config.app.name`
   - `tiny-internal-services` - Update EventService to accept optional queue parameter
   - `tiny-internal-services-mocks` - Update mocks if needed
   - **EXCLUDED:** `azi-3-status-check` - NO CHANGES (uses default queue)

3. **Database Schema:** Column name `queue` (stores suffix string, not full URL)
   - Created via separate migration script in `typ-e` repository

4. **Validation:** Queue suffix validated as alphanumeric + hyphens (not full URL validation)

5. **Queue Creation:** Queues must be pre-created during deployment
   - Localstack: via init script
   - AWS: manual creation before deployment

6. **Queue Suffix Source:** Use `config.app.name` as queue suffix in `azi-3-status-check-jobs`

7. **Testing:** Use real localstack queues for integration tests

### ✅ Resolved Clarifications

1. **Queue Suffix Configuration:** ✅ Use `app.name` from config as queue suffix (e.g., `config.app.name` = "azi-3-status-check-jobs")

2. **Queue Creation:** ✅ Queues must be pre-created during deployment
   - Localstack: Create via init script
   - AWS: Create before deploying services
   - Do NOT auto-create from megazord-events

3. **Testing Strategy:** ✅ Use real localstack queues for integration testing

4. **Migration Timing:** ✅ Create separate migration script in `typ-e` repository
   - Deploy migration first
   - Then deploy application services

5. **Additional Edge Cases:** ✅ No additional edge cases beyond what's in the plan

### 🔄 Next Steps

1. Begin Phase 3: Detailed Implementation Steps starting with database migration in `typ-e` repository
2. Implement DTO and domain model changes in `megazord-events`
3. Update repository and service layers with queue suffix logic
4. Update `tiny-internal-services` EventService
5. Update `azi-3-status-check-jobs` to pass `config.app.name` as queue suffix
6. Add comprehensive tests using real localstack queues
7. Update consumer services configuration
8. Perform integration testing with pre-created queues
9. Deploy to staging environment
10. Deploy to production

---

**Plan Status:** ✅ **READY FOR IMPLEMENTATION** - All clarifications resolved

**Notes:**

- All file paths are absolute and match workspace structure
- Implementation preserves backward compatibility
- Testing strategy covers integration, unit, and edge cases using real localstack queues
- Documentation updates ensure maintainability
- Deployment plan includes rollback procedures and pre-queue creation
- Simplified queue addressing using suffix pattern instead of full URLs
- Database migration in separate `typ-e` repository
- Implementation preserves backward compatibility
- Testing strategy covers integration, unit, and edge cases
- Documentation updates ensure maintainability
- Deployment plan includes rollback procedures
