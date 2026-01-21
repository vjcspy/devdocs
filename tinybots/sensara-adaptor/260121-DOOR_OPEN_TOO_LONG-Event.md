# 📋 [260121] - Add ST_DOOR_OPEN Sensara Alarm as Trigger

## References

- [Global Overview](/Users/kai/work/tinybots/devdocs/tinybots/OVERVIEW.md)
- [megazord-events OVERVIEW](/Users/kai/work/tinybots/devdocs/tinybots/megazord-events/OVERVIEW.md)
- [sensara-adaptor OVERVIEW](/Users/kai/work/tinybots/devdocs/tinybots/sensara-adaptor/OVERVIEW.md)
- [Similar Implementation Plan: PROD591 Add New Event Type](/Users/kai/work/tinybots/devdocs/tinybots/megazord-events/251112-PROD591-add-new-event-type.md)
- [TinybotsEvent enum](/Users/kai/work/tinybots/tiny-internal-services/lib/model/events/TinybotsEvent.ts)
- [SensaraNotificationType enum (shared)](/Users/kai/work/tinybots/tiny-internal-services/lib/model/sensara/SensaraNotificationDto.ts)
- [SensaraNotificationType enum (local)](/Users/kai/work/tinybots/sensara-adaptor/src/model/sensara/NotificationResponse.ts)
- [Event Schema Generator](/Users/kai/work/tinybots/megazord-events/schemas/gen.ts)
- [SensaraEventsJob](/Users/kai/work/tinybots/sensara-adaptor/src/jobs/SensaraEventsJob.ts)

## User Requirements

> Context:
> A client lives in a dangerous neighbourhood and forgets to close the door. Sensara introduced a new event for this ST_DOOR_OPEN, we want to be able to receive this event and mark it as trigger.
>
> Please add this alarm from Sensara as trigger to our sensara adaptor system: **ST_DOOR_OPEN**
>
> It should translate to the following event: **DOOR_OPEN_TOO_LONG** in megazord events
>
> Priority: Same as other triggers (they're all the same)
>
> Services involved:
> - megazord-events
> - sensara-adaptor

## 🎯 Objective

Add support for the new Sensara alarm `ST_DOOR_OPEN` by:
1. Creating a new TinyBots event `DOOR_OPEN_TOO_LONG` 
2. Mapping the Sensara notification `ST_DOOR_OPEN` to `DOOR_OPEN_TOO_LONG` in the sensara-adaptor pipeline
3. Enabling trigger functionality for downstream automation workflows

---

## 📊 Code Analysis: Current SSE Streaming Implementation

### ✅ SSE Streaming ĐÃ IMPLEMENT cho:

| Event Type | Description | Status |
|------------|-------------|--------|
| `NotificationResponse` | Sensara alarms/notifications (ST_*, LT_*, TA_*, etc.) | ✅ Working |
| `AdlEventResponse` | Activities of Daily Living events (TOILETING, EATING, SLEEPING, etc.) | ✅ Working |
| `StateExtramuralResponse` | State changes (BedState, etc.) | ✅ Working |

**Evidence từ code:**

```typescript
// sensara-adaptor/src/sensara/SensaraApiService.ts (lines 96-99)
const registerConfig: RegisterNotification = {
  dataTypes: [
    'NotificationResponse',      // ← ST_DOOR_OPEN sẽ đến qua đây
    'AdlEventResponse',
    'StateExtramuralResponse'
  ],
  filterProperty: 'residentId',
  filterValues: residents.map(resident => resident.residentId)
}
```

```typescript
// sensara-adaptor/src/eventsource/SensaraEventSource.ts (lines 123-155)
this._pendingEventSource.addEventListener('AdlEventResponse', ...)
this._pendingEventSource.addEventListener('NotificationResponse', ...)  // ← Listener đã có
this._pendingEventSource.addEventListener('StateExtramuralResponse', ...)
```

### 🔴 SSE Streaming CHƯA IMPLEMENT cho:

| Event Type | Description | Status | Plan |
|------------|-------------|--------|------|
| `LastLocationResponse` | Real-time location updates from sensors | ❌ Not implemented | [260107-Streaming-Location-Events.md](/Users/kai/work/tinybots/devdocs/tinybots/sensara-adaptor/260107-Streaming-Location-Events.md) |

**Hiện tại Location Events sử dụng POLLING** (qua `LocationPoller`, `ActivityPoller`), không phải streaming.

### Event Flow cho ST_DOOR_OPEN

```
┌─────────────────┐
│   Sensara API   │
│   (V3 SSE)      │
└────────┬────────┘
         │ SSE Stream: NotificationResponse
         │ { notificationType: "ST_DOOR_OPEN", residentId: "...", ... }
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ sensara-adaptor                                                     │
│                                                                     │
│  SensaraEventSource.addEventListener('NotificationResponse', ...)  │
│         │                                                           │
│         ▼                                                           │
│  SensaraEventsJob.handleEvent(event)                               │
│         │                                                           │
│         ▼                                                           │
│  SensaraEvent.fromEvent() → extracts notificationType              │
│         │                                                           │
│         ▼                                                           │
│  SensaraEventsJob.convertEvent()                                   │
│    case SensaraNotificationType.ST_DOOR_OPEN:        ← NEED TO ADD │
│      return _createEvent(event, TinybotsEvent.DOOR_OPEN_TOO_LONG)  │
│         │                                                           │
│         ▼                                                           │
│  EventService.postEvent(robotId, { eventName: 'DOOR_OPEN_TOO_LONG' })│
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ megazord-events │
│                 │
│ - Store event   │
│ - Fan out to    │
│   subscriptions │
│ - Trigger if    │
│   hasTrigger=true│
└─────────────────┘
```

---

## ⚠️ Key Considerations

1. **SSE Streaming đã hoạt động**: `ST_DOOR_OPEN` sẽ đến qua SSE stream hiện có dưới dạng `NotificationResponse` event (giống như `ST_SLEEPING_AWAKE_DELAYED`, `ST_ACTIVITY_SHORT_INACTIVITY`, etc.)

2. **Có 2 nơi cần thêm `SensaraNotificationType`**:
   - `tiny-internal-services/lib/model/sensara/SensaraNotificationDto.ts` (shared library)
   - `sensara-adaptor/src/model/sensara/NotificationResponse.ts` (local copy)

3. **Schema Configuration**: Based on stakeholder confirmation ("same priority as others"):
   - `level: 10` (standard event level)
   - `hasTrigger: true` (enables trigger subscriptions)

4. **SensaraEventsAdaptorService**: NO changes needed to `endpointsMapping` - that mapping is only for events we **register** with Sensara (polling), not for incoming notifications.

5. **Testing**: All tests must run in Docker via `just -f devtools/Justfile test-sensara-adaptor`

---

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Verify SSE streaming architecture and event flow
  - **Outcome**: SSE streaming IS implemented for `NotificationResponse` events. `ST_DOOR_OPEN` will arrive via this existing mechanism.

- [x] Verify `ST_DOOR_OPEN` is NOT already present in enums
  - **File 1**: `tiny-internal-services/lib/model/sensara/SensaraNotificationDto.ts` → NOT present
  - **File 2**: `sensara-adaptor/src/model/sensara/NotificationResponse.ts` → NOT present (has `ST_DOOR_PASSAGE` but not `ST_DOOR_OPEN`)

- [x] Verify `DOOR_OPEN_TOO_LONG` is NOT already present in `TinybotsEvent` enum
  - **File**: `tiny-internal-services/lib/model/events/TinybotsEvent.ts` → NOT present

### Phase 2: Implementation (File/Code Structure)

```
tiny-internal-services/
├── lib/model/events/TinybotsEvent.ts              # 🚧 TODO - Add DOOR_OPEN_TOO_LONG
├── lib/model/sensara/SensaraNotificationDto.ts    # 🚧 TODO - Add ST_DOOR_OPEN
├── package.json                                   # 🚧 TODO - Bump version (e.g., v1.24.0)

sensara-adaptor/
├── src/model/sensara/NotificationResponse.ts      # 🚧 TODO - Add ST_DOOR_OPEN to local enum
├── src/jobs/SensaraEventsJob.ts                   # 🚧 TODO - Add ST_DOOR_OPEN → DOOR_OPEN_TOO_LONG mapping
├── test/jobs/SensaraEventsJobTest.ts              # 🚧 TODO - Add test for new mapping
├── package.json                                   # 🚧 TODO - Bump tiny-internal-services dependency

megazord-events/
├── schemas/gen.ts                                 # 🚧 TODO - Add DOOR_OPEN_TOO_LONG config
├── schemas/events/door_open_too_long.json         # 🚧 TODO - Generated schema (level:10, hasTrigger:true)
├── package.json                                   # 🚧 TODO - Bump tiny-internal-services dependency
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Extend Shared Enums (`tiny-internal-services`)

**1.1 Add `ST_DOOR_OPEN` to `SensaraNotificationType`**

**File:** `tiny-internal-services/lib/model/sensara/SensaraNotificationDto.ts`

```typescript
export enum SensaraNotificationType {
  // ... existing entries ...
  ST_DOOR_PASSAGE = 'ST_DOOR_PASSAGE',
  ST_DOOR_OPEN = 'ST_DOOR_OPEN',  // ← ADD THIS LINE (after ST_DOOR_PASSAGE for logical grouping)
  ST_FIRE_DETECTION = 'ST_FIRE_DETECTION',
  // ... rest of entries ...
}
```

**1.2 Add `DOOR_OPEN_TOO_LONG` to `TinybotsEvent`**

**File:** `tiny-internal-services/lib/model/events/TinybotsEvent.ts`

```typescript
export enum TinybotsEvent {
  // ... existing entries ...
  LONGER_IN_BED_LONG = 'LONGER_IN_BED_LONG',
  NO_TOILET_ACTIVITY_ALARM = 'NO_TOILET_ACTIVITY_ALARM',
  DOOR_OPEN_TOO_LONG = 'DOOR_OPEN_TOO_LONG'  // ← ADD THIS LINE
}
```

**1.3 Publish New Version**

```bash
cd /Users/kai/work/tinybots/tiny-internal-services
# Update version in package.json (e.g., 1.23.0 → 1.24.0)
yarn build
yarn publish  # or follow your team's release process
```

---

#### Step 2: Update Local Enum in sensara-adaptor

**File:** `sensara-adaptor/src/model/sensara/NotificationResponse.ts`

```typescript
export enum SensaraNotificationType {
  // ... existing entries ...
  ST_DOOR_PASSAGE = 'ST_DOOR_PASSAGE',
  ST_DOOR_OPEN = 'ST_DOOR_OPEN',  // ← ADD THIS LINE
  ST_FIRE_DETECTION = 'ST_FIRE_DETECTION',
  // ... rest of entries ...
}
```

> **Note:** sensara-adaptor có local copy của enum này để handle trường hợp Sensara gửi notification types mới mà chưa được thêm vào shared library.

---

#### Step 3: Add Translation in sensara-adaptor

**File:** `sensara-adaptor/src/jobs/SensaraEventsJob.ts`

Add new case in `convertEvent` switch statement (around line 239):

```typescript
public static convertEvent(event: SensaraEvent): IncomingEventBodyDto | null {
  switch (event.event) {
    // ... existing cases ...
    case SensaraNotificationType.ST_SLEEPING_AWAKE_LARGE_DELAY:
      return this._createEvent(event, TinybotsEvent.LONGER_IN_BED_LONG)
    // ← ADD THIS CASE
    case SensaraNotificationType.ST_DOOR_OPEN:
      return this._createEvent(event, TinybotsEvent.DOOR_OPEN_TOO_LONG)
    default:
      return null
  }
}
```

**Update Dependency:**

```bash
cd /Users/kai/work/tinybots/sensara-adaptor
# Update package.json: "tiny-internal-services": "^1.24.0"
yarn install
```

---

#### Step 4: Update megazord-events Schemas

**4.1 Update Schema Generator Config**

**File:** `megazord-events/schemas/gen.ts`

Add `DOOR_OPEN_TOO_LONG` to both the local `TinybotsEvent` object and `CustomConfigs`:

```typescript
const TinybotsEvent = {
  // ... existing entries ...
  LONGER_IN_BED_LONG: 'LONGER_IN_BED_LONG',
  NO_TOILET_ACTIVITY_ALARM: 'NO_TOILET_ACTIVITY_ALARM',
  DOOR_OPEN_TOO_LONG: 'DOOR_OPEN_TOO_LONG'  // ← ADD THIS LINE
} as const

const CustomConfigs: {
  [K in keyof typeof TinybotsEvent]?: Config
} = {
  // ... existing entries ...
  OUTSIDE_HOME: {
    level: 10,
    hasTrigger: true
  },
  // ← ADD THIS BLOCK
  DOOR_OPEN_TOO_LONG: {
    level: 10,
    hasTrigger: true
  }
}
```

**4.2 Generate Event Schema**

```bash
cd /Users/kai/work/tinybots/megazord-events
yarn generate:schemas
```

**Expected output:** Creates `schemas/events/door_open_too_long.json`:

```json
{
    "eventName": "DOOR_OPEN_TOO_LONG",
    "level": 10,
    "hasTrigger": true,
    "isActive": true,
    "description": "Auto generated schema definition by megazord-events"
}
```

**4.3 Update Dependency:**

```bash
cd /Users/kai/work/tinybots/megazord-events
# Update package.json: "tiny-internal-services": "^1.24.0"
yarn install
```

---

#### Step 5: Implement Tests

**File:** `sensara-adaptor/test/jobs/SensaraEventsJobTest.ts`

Add new test case following existing pattern (around line 343, after `LONGER_IN_BED_LONG` test):

```typescript
it('should handle a DOOR_OPEN_TOO_LONG notification event', async () => {
  const residentRobot: ResidentRobot = {
    id: 1,
    residentId: 'abcdefg',
    robotId: 23
  }
  const response: NotificationResponse = {
    id: 'door-open-too-long-id',
    residentId: residentRobot.residentId,
    notificationType: SensaraNotificationType.ST_DOOR_OPEN,
    intervalStartTime: '2026-01-21T12:01:01.000Z',
    correlationId: 'door-open-too-long-correlation',
    sensorLocation: 'entrance',
    parameters: {
      additionalProp1: '',
      additionalProp2: '',
      additionalProp3: ''
    }
  }

  const raw: MessageEvent<NotificationResponse> = new MessageEvent<
    NotificationResponse
  >(
    'NotificationResponse',
    {
      data: response,
      lastEventId: 'doorOpenTooLongId'
    }
  )

  const event = SensaraEvent.fromEvent(residentRobot, raw, raw.data)
  event.id = 99

  when(residentRepository.getResidentByResidentId('abcdefg')).thenResolve(
    residentRobot
  )
  when(sensaraEventRepository.storeEvent(anything())).thenResolve(event)

  const expectedBody: IncomingEventBodyDto = {
    providerName: 'Sensara',
    eventName: TinybotsEvent.DOOR_OPEN_TOO_LONG,
    level: undefined,
    referenceId: 'doorOpenTooLongId'
  }

  eventServiceMock.postEvent(residentRobot.robotId, expectedBody)

  await job.handleEvent(mockCtx, raw)

  verify(
    eventServiceMock.getMock().postEvent(
      mockCtx,
      residentRobot.robotId,
      deepEqual(expectedBody)
    )
  )
    .once()
})
```

---

#### Step 6: Testing & Validation

**Run Tests (CRITICAL: Must use Docker)**

```bash
# From project root
cd /Users/kai/work/tinybots

# Run sensara-adaptor tests
just -f devtools/Justfile test-sensara-adaptor

# Run megazord-events tests (optional, to verify schema loading)
just -f devtools/Justfile test-megazord-events
```

**Manual Verification Checklist:**

1. [ ] `tiny-internal-services` builds successfully with new enums
2. [ ] `sensara-adaptor` local enum updated
3. [ ] `megazord-events` schema file `door_open_too_long.json` is generated correctly
4. [ ] `sensara-adaptor` compiles without errors after adding new case
5. [ ] All existing tests pass
6. [ ] New test for `DOOR_OPEN_TOO_LONG` passes

---

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements

- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Pre-Implementation Questions (Optional)

1. **Sensara Documentation**: Confirm the exact event name from Sensara is `ST_DOOR_OPEN` (not a variant like `ST_DOOR_OPEN_ALARM` or similar)

2. **Event Description**: Should we update the auto-generated description to something more specific like "Door has been left open for too long"?

### Deployment Order

Recommended deployment sequence:

1. **Deploy `tiny-internal-services`** with new version (adds enums)
2. **Deploy `megazord-events`** (loads new schema on startup)
3. **Deploy `sensara-adaptor`** (starts translating new events)

### Follow-up Tasks

- [ ] Monitor production logs after deployment for `ST_DOOR_OPEN` events
- [ ] Verify trigger subscriptions can be created for `DOOR_OPEN_TOO_LONG`
- [ ] Update any API documentation listing available event types
- [ ] Notify stakeholders when feature is live
