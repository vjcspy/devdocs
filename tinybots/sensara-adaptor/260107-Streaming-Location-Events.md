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

1. **API Version Migration**: Current implementation uses V3 API format. New Sensara API uses V4 format with different request structure:
   - V3: `{ dataTypes: [], filterProperty: "", filterValues: [] }`
   - V4: `{ dataTypeRequests: [{ dataType: "", filters: [{ filterProperty: "", filterValues: [] }] }] }`

2. **Backward Compatibility**: 
   - Activity check MUST remain unchanged (still uses polling)
   - Location polling endpoints should be deprecated but kept for rollback safety

3. **Event Mapping**: `LastLocationResponse` from Sensara needs to be mapped to TinyBots events:
   - `LOCATION_BATHROOM` → `IN_BATHROOM`
   - `LOCATION_KITCHEN` → `IN_KITCHEN`
   - Hearing range check needs to match location/label with stored hearable locations

4. **azi-3-status-check**: NO changes required - it only subscribes to events from megazord-events, does not call sensara-adaptor directly

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

### Phase 2: Implementation (File/Code Structure)

```
sensara-adaptor/src/
├── sensara/
│   └── SensaraApiService.ts           # 🔄 IN PROGRESS - Add V4 stream registration
├── model/sensara/
│   ├── LastLocationResponse.ts        # 🚧 TODO - New SSE event model
│   ├── RegisterNotification.ts        # 🔄 IN PROGRESS - Update to V4 format
│   └── SensaraEvent.ts                # 🔄 IN PROGRESS - Add LastLocationResponse type
├── eventsource/
│   └── SensaraEventSource.ts          # 🔄 IN PROGRESS - Add LastLocationResponse listener
├── jobs/
│   └── SensaraEventsJob.ts            # 🔄 IN PROGRESS - Handle & convert location events
├── service/
│   └── LocationEventMapper.ts         # ✅ EXISTS - Already maps LOCATION_* to IN_*
└── controller/
    └── LocationController.ts          # 📝 DEPRECATE - Mark polling endpoints as deprecated

megazord-events/src/
└── services/internal/
    └── SensaraEventsAdaptorService.ts # 🔄 IN PROGRESS - Remove location event registration
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Update Sensara API to V4 Format (sensara-adaptor)

**File**: `sensara-adaptor/src/model/sensara/RegisterNotification.ts`

```typescript
// NEW V4 format
export interface DataTypeFilter {
  filterProperty: string
  filterValues: string[]
}

export interface DataTypeRequest {
  dataType: string
  filters: DataTypeFilter[]
}

export class RegisterNotificationV4 {
  dataTypeRequests: DataTypeRequest[]
  maxFlowRateMillis?: number
}
```

**File**: `sensara-adaptor/src/sensara/SensaraApiService.ts`

- Update `STREAM_REGISTRATION_URL` from `/v3/streams/registrations` to `/v4/streams`
- Update `STREAM_URL` from `/v3/streams` to `/v4/streams`
- Update `registerStream()` method to use V4 format
- Add `LastLocationResponse` to dataTypes list

#### Step 2: Add LastLocationResponse Model (sensara-adaptor)

**File**: `sensara-adaptor/src/model/sensara/LastLocationResponse.ts` (NEW)

```typescript
export interface LastLocationResponse {
  organizationId: string
  residentId: string
  sectorId: string
  correlationId: string
  deviceId: string
  label: string
  timestamp: string // LocalDateTime
  location: {
    // SensorLocation object from Sensara
    name: string
    // ... other fields as needed
  }
}
```

#### Step 3: Update SensaraEvent to Handle LastLocationResponse (sensara-adaptor)

**File**: `sensara-adaptor/src/model/sensara/SensaraEvent.ts`

- Add `'LastLocationResponse'` to type union
- Add case handler in `fromEvent()` static method

#### Step 4: Add SSE Listener for LastLocationResponse (sensara-adaptor)

**File**: `sensara-adaptor/src/eventsource/SensaraEventSource.ts`

```typescript
// In _registerEvents() method, add:
this._pendingEventSource.addEventListener(
  'LastLocationResponse',
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

#### Step 5: Handle Location Event Conversion (sensara-adaptor)

**File**: `sensara-adaptor/src/jobs/SensaraEventsJob.ts`

Update `convertEvent()` to handle `LastLocationResponse`:

```typescript
case 'LastLocationResponse':
  // Use LocationEventMapper to convert label/location to TinybotsEvent
  const locationEvent = this._eventMapper.FromSensaraLocation({
    label: event.event,
    location: event.additionalData?.location ?? ''
  })
  
  if (locationEvent) {
    return {
      providerName: 'Sensara',
      eventName: locationEvent,
      referenceId: event.sensaraId
    }
  }
  
  // Check for CLIENT_IN_HEARING_RANGE
  // Need to check against stored hearable locations for resident
  return null
```

**Consideration**: `CLIENT_IN_HEARING_RANGE` requires checking if the location matches the resident's stored hearable locations. This requires:
- Fetching hearable locations from `ResidentRepository`
- Comparing with incoming location/label
- Emitting `CLIENT_IN_HEARING_RANGE` if matched

#### Step 6: Update SensaraEventsJob Constructor (sensara-adaptor)

- Inject `LocationEventMapper` 
- Inject `ResidentRepository` (for hearable locations lookup)
- Add logic to check hearable locations for `CLIENT_IN_HEARING_RANGE`

#### Step 7: Remove Location Event Registration from Megazord (megazord-events)

**File**: `megazord-events/src/services/internal/SensaraEventsAdaptorService.ts`

Remove/comment out location event mappings from `endpointsMapping`:

```typescript
private endpointsMapping: Map<string, [endpoint: string, mustForwardEventName: boolean]> = {
  // REMOVE these - now handled by SSE stream
  // 'CLIENT_IN_HEARING_RANGE': [CLIENT_IN_HEARING_RANGE_ENDPOINT, false],
  // [TinybotsEvent.IN_BATHROOM]: [LOCATION_V2_ENDPOINT, true],
  // ... other IN_* events
  
  // KEEP this - activity still uses polling
  'ACTIVITY': [ACTIVITY_ENDPOINT, false],
}
```

#### Step 8: Deprecation & Documentation (sensara-adaptor)

- Add `@deprecated` JSDoc comments to:
  - `LocationController.pollLocation()`
  - `LocationController.pollLocationV2()`
- Update OpenAPI docs to mark these endpoints as deprecated
- Keep endpoints functional for rollback safety

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

1. **V4 API Confirmation**: Need to confirm with Sensara team:
   - Exact V4 endpoint URLs (is it `/v4/streams` or different?)
   - `LastLocationResponse` exact schema (especially `location` field type - is it `SensorLocation` object or string?)
   
2. **CLIENT_IN_HEARING_RANGE Logic**: Current polling logic checks `hearableLocations` stored in DB per resident. With streaming approach:
   - Need to fetch hearable locations on each location event
   - OR cache hearable locations in memory (need refresh strategy)
   
3. **Transition Strategy**: 
   - How to handle existing active pollers during deployment?
   - Should we add feature flag to gradually rollout?

4. **Monitoring**: 
   - Add metrics for streaming events vs polled events
   - Alert when stream disconnects or no location events received for extended period
