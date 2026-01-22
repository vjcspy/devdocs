# 📋 [Streaming-Location-Events: 2026-01-07] - Implement Streaming API with Location Data

## References

- Sensara Stream V4 Documentation: `devdocs/tinybots/sensara-adaptor/shareable-documentation.md`
- sensara-adaptor OVERVIEW: `devdocs/tinybots/sensara-adaptor/OVERVIEW.md`
- megazord-events OVERVIEW: `devdocs/tinybots/megazord-events/OVERVIEW.md`

## User Requirements

> Implement the new streaming api with location data. Location events should now be forwarded from the streaming api to megazord events
>
> If working, the following current flows should now rely on the events from the event stream:
> - location check
> - client in hearing range check
>
> The following should remain unchanged:
> - activity check
>
> Repos: Sensara adaptor, Megazord event, (maybe) azi-3 status check

## 🎯 Objective

Migrate from **polling-based** to **streaming-based** (SSE) approach for Location Events from Sensara, providing:
- Reduced latency: receive events in real-time instead of polling every 3 seconds
- Reduced API calls: single SSE connection instead of multiple REST calls
- Increased reliability: no missed events between poll intervals

---

## 📊 Existing SSE Infrastructure (IMPORTANT CONTEXT)

### ✅ SSE Streaming ĐÃ ĐƯỢC IMPLEMENT cho các event types sau:

| Event Type | Description | Status |
|------------|-------------|--------|
| `NotificationResponse` | Sensara alarms (ST_*, LT_*, TA_*) | ✅ Working |
| `AdlEventResponse` | ADL events (TOILETING, EATING, SLEEPING, etc.) | ✅ Working |
| `StateExtramuralResponse` | State changes (BedState, etc.) | ✅ Working |
| `LastLocationResponse` | Real-time location updates | ❌ **This is what we need to add** |

### Existing SSE Architecture

```
sensara-adaptor/src/
├── sensara/SensaraApiService.ts
│   └── registerStream() → POST /v3/streams/registrations
│       └── dataTypes: ['NotificationResponse', 'AdlEventResponse', 'StateExtramuralResponse']
│
├── eventsource/SensaraEventSource.ts
│   └── _registerEvents() → addEventListener for each event type
│
├── jobs/SensaraEventsJob.ts
│   ├── handleEvent() → receives SSE events
│   └── convertEvent() → maps Sensara events to TinybotsEvent
│
└── model/sensara/SensaraEvent.ts
    └── fromEvent() → parses different event types
```

### 🔑 Key Insight: RE-USE Existing Infrastructure

**KHÔNG CẦN tạo mới SSE infrastructure.** Chỉ cần:
1. **ADD** `LastLocationResponse` vào `dataTypes` array trong `registerStream()`
2. **ADD** event listener trong `SensaraEventSource._registerEvents()`
3. **ADD** case handler trong `SensaraEvent.fromEvent()`
4. **ADD** conversion logic trong `SensaraEventsJob.convertEvent()`

**V4 API Migration** (nếu cần) là concern riêng biệt, không liên quan trực tiếp đến việc thêm `LastLocationResponse`.

---

### 📊 Architecture Flow Comparison

#### Current Flow (Polling-based)

```
┌─────────────────┐    1. Subscribe to IN_BATHROOM    ┌──────────────────┐
│  azi-3-status   │ ────────────────────────────────► │  megazord-events │
│     check       │                                   │                  │
└─────────────────┘                                   └────────┬─────────┘
                                                               │
                            2. Call /internal/v1/sensara/      │
                               residents/location/register     │
                                                               ▼
                                                      ┌──────────────────┐
                                                      │ sensara-adaptor  │
                                                      │                  │
                                                      │  LocationPoller  │
                                                      │  (polls every 3s)│
                                                      └────────┬─────────┘
                                                               │
                            3. REST API call every 3s          │
                               GET /v3/hardware/last-location  │
                                                               ▼
                                                      ┌──────────────────┐
                                                      │   Sensara API    │
                                                      └──────────────────┘
```

**Problem**: The mapping in megazord-events triggers sensara-adaptor to START a poller for each subscription.

#### New Flow (Streaming-based)

```
┌──────────────────┐     SSE Stream (always on)      ┌──────────────────┐
│   Sensara API    │ ──────────────────────────────► │ sensara-adaptor  │
│                  │     LastLocationResponse        │                  │
└──────────────────┘                                 └────────┬─────────┘
                                                              │
                              POST event automatically        │
                              when location changes           │
                                                              ▼
                                                     ┌──────────────────┐
                                                     │  megazord-events │
                                                     │                  │
                                                     └────────┬─────────┘
                                                              │
                              Fan out to subscribers          │
                                                              ▼
                                                     ┌─────────────────┐
                                                     │  azi-3-status   │
                                                     │     check       │
                                                     └─────────────────┘
```

**Solution**: Events flow automatically via SSE - no need to register pollers!

#### Why Remove Location Event Mappings from megazord-events?

If we **keep** the mapping after implementing streaming:
- megazord-events will **still call sensara-adaptor** to start a poller
- But the poller is **unnecessary** because events already come from the SSE stream
- Result: **duplicate events** and **wasted resources**

By **removing** the mapping:
- megazord-events just stores the subscription
- Events arrive via: SSE → sensara-adaptor → megazord-events → fan out to subscribers
- Clean, single source of truth

**Note:** We keep `ACTIVITY` mapping because activity check still uses polling (as per requirements).

### ⚠️ Key Considerations

1. **RE-USE, Don't Reinvent**: 
   - SSE infrastructure đã được implement và đang hoạt động tốt cho 3 event types
   - Chỉ cần ADD `LastLocationResponse` theo cùng pattern
   - KHÔNG cần tạo mới EventSource, job handlers, etc.

2. **V4 API Migration (SEPARATE CONCERN)**:
   - Current V3: `{ dataTypes: [], filterProperty: "", filterValues: [] }`
   - New V4: `{ dataTypeRequests: [{ dataType: "", filters: [{ filterProperty: "", filterValues: [] }] }] }`
   - **Cần confirm với Sensara**: V3 có support `LastLocationResponse` không?
   - Nếu cần V4, đó là migration riêng biệt, không liên quan trực tiếp đến việc thêm location streaming

3. **Backward Compatibility**: 
   - Activity check MUST remain unchanged (still uses polling)
   - Location polling endpoints should be deprecated but kept for rollback safety

4. **Event Mapping**: `LastLocationResponse` from Sensara needs to be mapped to TinyBots events:
   - `LOCATION_BATHROOM` → `IN_BATHROOM`
   - `LOCATION_KITCHEN` → `IN_KITCHEN`
   - Có thể re-use `LocationEventMapper` đã có sẵn
   - Hearing range check needs to match location/label with stored hearable locations

5. **azi-3-status-check**: NO changes required - it only subscribes to events from megazord-events, does not call sensara-adaptor directly

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Analyze current SSE implementation in `sensara-adaptor`
  - **Outcome**: Current `SensaraEventSource` listens to `AdlEventResponse`, `NotificationResponse`, `StateExtramuralResponse`
  
- [ ] Analyze `LastLocationResponse` schema from Sensara V4 API
  - **Outcome**: Contains `organizationId`, `residentId`, `sectorId`, `correlationId`, `deviceId`, `label`, `timestamp`, `location`
  
- [ ] Define scope and edge cases
  - **Outcome**: 
    - Edge case 1: Multiple residents with same location label
    - Edge case 2: Stream disconnection handling (existing backoff logic applies)
    - Edge case 3: Transition period - old pollers may still be running

### Phase 2: Implementation (File/Code Structure) - RE-USE EXISTING

```
sensara-adaptor/src/
├── sensara/
│   └── SensaraApiService.ts           # 🔄 MODIFY - Add 'LastLocationResponse' to dataTypes array
├── model/sensara/
│   ├── LastLocationResponse.ts        # 🚧 NEW - Model for SSE event
│   └── SensaraEvent.ts                # 🔄 MODIFY - Add case in fromEvent()
├── eventsource/
│   └── SensaraEventSource.ts          # 🔄 MODIFY - Add addEventListener('LastLocationResponse')
├── jobs/
│   └── SensaraEventsJob.ts            # 🔄 MODIFY - Add conversion in convertEvent()
├── service/
│   └── LocationEventMapper.ts         # ✅ EXISTS - Already maps LOCATION_* to IN_*
└── controller/
    └── LocationController.ts          # 📝 DEPRECATE - Mark polling endpoints as deprecated

megazord-events/src/
└── services/internal/
    └── SensaraEventsAdaptorService.ts # 🔄 MODIFY - Remove location event registration
```

### Phase 3: Detailed Implementation Steps (RE-USE Pattern)

> **Pattern**: Follow the same approach used for NotificationResponse, AdlEventResponse, StateExtramuralResponse

#### Step 1: Add LastLocationResponse to Stream Registration

**File**: `sensara-adaptor/src/sensara/SensaraApiService.ts`

```typescript
// BEFORE (line 96-99)
const registerConfig: RegisterNotification = {
  dataTypes: [
    'NotificationResponse',
    'AdlEventResponse',
    'StateExtramuralResponse'
  ],
  // ...
}

// AFTER - simply add to existing array
const registerConfig: RegisterNotification = {
  dataTypes: [
    'NotificationResponse',
    'AdlEventResponse',
    'StateExtramuralResponse',
    'LastLocationResponse'  // ← ADD THIS LINE
  ],
  // ...
}
```

> **Note về V4 API**: Nếu Sensara yêu cầu V4 format cho `LastLocationResponse`, đó là concern riêng biệt. Có thể:
> - Option A: Migrate toàn bộ sang V4 (breaking change)
> - Option B: Chỉ dùng V4 cho LastLocationResponse (parallel registration)
> - Option C: Confirm với Sensara nếu V3 vẫn support LastLocationResponse

#### Step 2: Add LastLocationResponse Model

**File**: `sensara-adaptor/src/model/sensara/LastLocationResponse.ts` (NEW)

```typescript
export interface LastLocationResponse {
  organizationId: string
  residentId: string
  sectorId: string
  correlationId: string
  deviceId: string
  label: string
  timestamp: string
  location: {
    name: string
    // ... other fields as needed
  }
}
```

#### Step 3: Update SensaraEvent.fromEvent() - Follow Existing Pattern

**File**: `sensara-adaptor/src/model/sensara/SensaraEvent.ts`

```typescript
// BEFORE - existing pattern
export class SensaraEvent {
  // ...
  type: 'AdlEventResponse' | 'NotificationResponse' | 'StateExtramuralResponse'
  
  static fromEvent(residentRobot, event, eventData) {
    switch (event.type) {
      case 'AdlEventResponse': // ...
      case 'NotificationResponse': // ...
      case 'StateExtramuralResponse': // ...
    }
  }
}

// AFTER - add new case following same pattern
export class SensaraEvent {
  // ...
  type: 'AdlEventResponse' | 'NotificationResponse' | 'StateExtramuralResponse' | 'LastLocationResponse'
  
  static fromEvent(residentRobot, event, eventData) {
    switch (event.type) {
      case 'AdlEventResponse': // ...
      case 'NotificationResponse': // ...
      case 'StateExtramuralResponse': // ...
      case 'LastLocationResponse':  // ← ADD THIS CASE
        return new SensaraEvent({
          residentRobot,
          event: (eventData as LastLocationResponse).label,
          sensaraId: event.lastEventId,
          type: event.type
        })
    }
  }
}
```

#### Step 4: Add SSE Listener - Follow Existing Pattern

**File**: `sensara-adaptor/src/eventsource/SensaraEventSource.ts`

```typescript
// In _registerEvents() method - add same pattern as other listeners:
this._pendingEventSource.addEventListener(
  'LastLocationResponse',  // ← Same pattern as 'NotificationResponse'
  async (event: MessageEvent<any>) => {
    try {
      logger.info(`received LastLocationResponse for ${this.name}`)
      await this._handleEvent(ctx, event)
    } catch (error) {
      logger.error(error)
    }
  }
)
```

#### Step 5: Add Conversion in SensaraEventsJob.convertEvent()

**File**: `sensara-adaptor/src/jobs/SensaraEventsJob.ts`

```typescript
public static convertEvent(event: SensaraEvent): IncomingEventBodyDto | null {
  switch (event.event) {
    // ... existing cases for AdlEventType, SensaraNotificationType ...
    
    // ADD: Handle location labels from LastLocationResponse
    case 'LOCATION_BATHROOM':
      return this._createEvent(event, TinybotsEvent.IN_BATHROOM)
    case 'LOCATION_KITCHEN':
      return this._createEvent(event, TinybotsEvent.IN_KITCHEN)
    // ... other LOCATION_* → IN_* mappings
    // OR: Use LocationEventMapper for dynamic mapping
    
    default:
      return null
  }
}
```

**Alternative - Use LocationEventMapper** (đã có sẵn):

```typescript
// Inject LocationEventMapper vào SensaraEventsJob
// Sử dụng nó để map location labels → TinybotsEvent dynamically
```

#### Step 6: Handle CLIENT_IN_HEARING_RANGE (Special Case)

**Consideration**: `CLIENT_IN_HEARING_RANGE` cần logic đặc biệt:
- Cần check location có match với `hearableLocations` của resident không
- Cần inject `ResidentRepository` để fetch hearable locations
- Có thể cache để tránh query DB mỗi event

#### Step 7: Remove Location Event Registration from megazord-events

**File**: `megazord-events/src/services/internal/SensaraEventsAdaptorService.ts`

```typescript
// REMOVE location events - now handled by SSE stream
// KEEP 'ACTIVITY' - still uses polling
```

#### Step 8: Deprecate Polling Endpoints

- Add `@deprecated` JSDoc to `LocationController.pollLocation()` và `pollLocationV2()`
- Keep endpoints functional for rollback

### Phase 4: Testing Strategy

- [ ] Unit tests for new `LastLocationResponse` handling in `SensaraEventsJob`
- [ ] Unit tests for V4 stream registration format
- [ ] Integration tests for SSE → Event forwarding flow
- [ ] Verify `ACTIVITY` polling still works unchanged
- [ ] Manual E2E testing with Sensara test environment

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

1. **V3 vs V4 API**: Need to confirm with Sensara team:
   - Does V3 API support `LastLocationResponse` data type? (preferred - no migration needed)
   - If V4 required, what's the exact endpoint and format?
   - `LastLocationResponse` exact schema (especially `location` field type)
   
2. **CLIENT_IN_HEARING_RANGE Logic**: Current polling logic checks `hearableLocations` stored in DB per resident. With streaming approach:
   - Need to fetch hearable locations on each location event
   - OR cache hearable locations in memory (need refresh strategy)
   
3. **Transition Strategy**: 
   - How to handle existing active pollers during deployment?
   - Should we add feature flag to gradually rollout?

4. **Monitoring**: 
   - Add metrics for streaming events vs polled events
   - Alert when stream disconnects or no location events received for extended period

### 📝 Implementation Checklist (RE-USE Approach)

- [ ] Confirm V3 support for `LastLocationResponse` with Sensara
- [ ] Add `LastLocationResponse` to `dataTypes` in `SensaraApiService.registerStream()`
- [ ] Create `LastLocationResponse` model
- [ ] Add case in `SensaraEvent.fromEvent()`
- [ ] Add `addEventListener('LastLocationResponse')` in `SensaraEventSource`
- [ ] Add location → TinybotsEvent mapping in `SensaraEventsJob.convertEvent()`
- [ ] Handle `CLIENT_IN_HEARING_RANGE` special case
- [ ] Remove location mappings from `megazord-events/SensaraEventsAdaptorService`
- [ ] Add unit tests following existing patterns
- [ ] Deprecate polling endpoints
