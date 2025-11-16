# ReactiveEvent Actions/Effects (Quarkus + Mutiny)

Lightweight, in-memory action pipelines on Vert.x Event Bus. Optimized for clear, testable flows.

## Overview

Architecture
```
Action (dispatch)
    → Event Bus ("reactive-event-bus")
    → ReactiveEventManager (@ConsumeEvent)
    → Effect handler(s)
    → New Action(s) (publish)
    → ...
```

Core pieces
- Action: type + payload + correlationId
- Manager: dispatch + routing; registers handlers
- Effect: bean returning `ReactiveEventHandler` (Mutiny `Multi` → `Multi`)
- Initializer: auto-registers effects

Processing modes
- Concurrent: `transformToUniAndMerge`
- Serialized (one-by-one): `transformToUniAndConcatenate`

When to use
- Merge for throughput; Concatenate for ordered pagination, idempotent upserts, single-threaded workflows.

## API Reference

### ReactiveEventAction<T>
```java
// Constructor
new ReactiveEventAction<>(String type, T payload)

// Methods
String getType()
T getPayload()
UUID getCorrelationId()
void setCorrelationId(UUID correlationId) // Set-once only
<R> R getPayloadAs(Class<R> clazz) // Type-safe payload extraction
// Convenience: a no-op action type that ends the chain when no handlers are registered
ReactiveEventActionFactory<Void> EMPTY
```

### ReactiveEventManager
```java
// Dispatch action
static void dispatch(ReactiveEventAction<Object> action)

// Register handler (done automatically by @Effect)
static void registerEvent(String[] eventTypes, ReactiveEventHandler handler)
```

### @Effect Annotation
```java
@Effect(types = {"ACTION_TYPE_1", "ACTION_TYPE_2"}) // Required: explicit action types
```

### ReactiveEventHandler Interface
```java
@FunctionalInterface
public interface ReactiveEventHandler {
    Multi<ReactiveEventAction<Object>> apply(Multi<ReactiveEventAction<Object>> upstream);
}
```

Operator choices
- Merge for concurrent fan-out
- Concatenate for strict sequencing

## Quickstart: Price sync (serialized)

Flow: `PRICE_SYNC_START → PRICE_LOAD → PRICE_SAVE → (loop) → PRICE_FINISHED | PRICE_ERROR`

Key ideas
- Use Concatenate for strict ordering
- Keep pagination in payload: `code`, `startDate`, `endDate`, `offset`, `limit`
- `hasMore = count >= limit` → next `PRICE_LOAD` or `PRICE_FINISHED`
- Resume state with `sync_price_{code}`; optional `fromBeginning=true` to backfill
- Throttle between pages; settle correlation on terminal steps; end with `EMPTY`

Sketch
```java
@ApplicationScoped @Unremovable
class PriceSyncEffect {
    static final int DEFAULT_LIMIT = 300;

    @Effect(types = { PRICE_SYNC_START })
    ReactiveEventHandler onStart() {
        return up -> up.onItem().transformToUniAndConcatenate(evt ->
            syncStatus.getByKey(jobKey(evt)).onItem().transform(state -> PRICE_LOAD.create(Map.of(
                "code", code(evt),
                "startDate", resolveStartDate(evt, state),
                "endDate", LocalDate.now(ZoneOffset.UTC),
                "offset", 0,
                "limit", DEFAULT_LIMIT))));
    }

    @Effect(types = { PRICE_LOAD })
    ReactiveEventHandler onLoad() {
        return up -> up.onItem().transformToUniAndConcatenate(evt ->
            fetch.fetchPage(code(evt), start(evt), end(evt), offset(evt), limit(evt))
                .onItem().call(x -> Uni.createFrom().voidItem().onItem().delayIt().by(Duration.ofMillis(250)))
                .onItem().transform(list -> list == null || list.isEmpty()
                    ? PRICE_FINISHED.create(Map.of("code", code(evt)))
                    : PRICE_SAVE.create(Map.of(
                            "code", code(evt), "data", list,
                            "offset", offset(evt), "limit", limit(evt), "count", list.size(),
                            "startDate", start(evt), "endDate", end(evt))))
                .onFailure().recoverWithItem(t -> PRICE_ERROR.create(Map.of("code", code(evt), "error", t))));
    }

    @Effect(types = { PRICE_SAVE })
    ReactiveEventHandler onSave() {
        return up -> up.onItem().transformToUniAndConcatenate(evt -> {
            boolean more = count(evt) >= limit(evt);
            int next = offset(evt) + limit(evt);
            return price.upsertAll(data(evt))
                .replaceWith(more ? PRICE_LOAD.create(nextPayload(evt, next))
                                                    : PRICE_FINISHED.create(Map.of("code", code(evt))))
                .onFailure().recoverWithItem(t -> PRICE_ERROR.create(Map.of("code", code(evt), "error", t)));
        });
    }

    @Effect(types = { PRICE_FINISHED })
    ReactiveEventHandler onFinished() {
        return up -> up.onItem().transformToUniAndConcatenate(evt ->
            status.saveSuccess(jobKey(evt), now(), null, null)
                .onFailure().recoverWithNull()
                .chain(() -> settleByCorrelationId(evt.getCorrelationId(), null))
                .replaceWith(ReactiveEventAction.EMPTY.create(null)));
    }

    @Effect(types = { PRICE_ERROR })
    ReactiveEventHandler onError() {
        return up -> up.onItem().transformToUniAndConcatenate(evt ->
            status.saveError(jobKey(evt), now(), error(evt).getMessage(), null)
                .onFailure().recoverWithNull()
                .invoke(() -> slack.postMessage("sync stock price error " + code(evt)))
                .chain(() -> settleByCorrelationId(evt.getCorrelationId(), error(evt)))
                .replaceWith(ReactiveEventAction.EMPTY.create(null)));
    }
}
```

## Correlation & dispatch

Correlation ID is automatically:
- Assigned a new UUID if action doesn't have one
- Copied from original action to new action in effects
- Validated to ensure no changes in pipeline

```java
// Manual correlation (if needed)
var action = MyActions.START.create(payload);
action.setCorrelationId(customUUID);
ReactiveEventManager.dispatch(action);
```

Correlation-aware ack/nack (optional)
- If bridging with a queue/HTTP gateway, track message refs by correlationId and settle on FINISHED/ERROR.

```java
private Uni<Void> settleByCorrelationId(UUID correlationId, Throwable error) {
    if (correlationId == null) return Uni.createFrom().voidItem();
    return messageStore.remove(correlationId)
        .map(opt -> opt.orElse(null))
        .onItem().transformToUni(ref -> {
            if (ref == null) return Uni.createFrom().voidItem();
            return Uni.createFrom().completionStage(error == null ? ref.ack() : ref.nack(error))
                .replaceWithVoid();
        });
}
```

## Error handling (concise)

```java
@Effect(types = {"RISKY_ACTION"})
public ReactiveEventHandler errorHandling() {
    return upstream -> upstream
    .onItem().transformToUniAndMerge(action ->
            riskyService.process(action.getPayload())
                .onItem().transform(result -> SuccessActions.DONE.create(result))
                .onFailure().recoverWithItem(ex -> {
                    logger.error("Failed to process action", ex);
                    return ErrorActions.FAILED.create(ex.getMessage());
                }));
}
```

### 5. Multiple Input Types

```java
@Effect(types = {"TYPE_A", "TYPE_B", "TYPE_C"})
public ReactiveEventHandler multiTypeEffect() {
    return upstream -> upstream
        .onItem().transform(action -> {
            return switch (action.getType()) {
                case "TYPE_A" -> handleTypeA(action);
                case "TYPE_B" -> handleTypeB(action);
                case "TYPE_C" -> handleTypeC(action);
                default -> action; // Pass through
            };
        });
}
```

## Troubleshooting

**Problem**: Can't see "Registered ReactiveEvent handler" log
- **Solution**: Add `@Unremovable` to effect bean

**Problem**: NoClassDefFoundError on startup
- **Solution**: Initializer already filters `com.vjcspy.*` packages

**Problem**: Effect not running
- **Solution**: Check action type matching (explicit types are required) and payload type casting

**Problem**: Infinite loop
- **Solution**: Effect must emit different action type than input type

**Problem**: Event Bus codec error ("No message codec for type ...")
- **Solution**: Register a local codec for `ReactiveEventAction` or serialize actions to a supported format when using Event Bus

**Problem**: No-op end-of-chain still publishes
- **Solution**: Emit an action type with no handlers (e.g., `EMPTY`).

## Router behavior (under the hood)

- A single `@ConsumeEvent("reactive-event-bus")` method in `ReactiveEventManager` receives actions
- For the incoming action type, all registered handlers are looked up and invoked
- Each handler receives a single-item `Multi` of the origin action and returns a `Multi` of new actions
- Outputs are merged reactively with bounded concurrency (configurable)
- Correlation ID is validated and propagated to child actions
- Child actions are re-published to the Event Bus via `dispatch` (fire-and-forget)

## Configuration

- Merge concurrency (default 64):
    - `com.vjcspy.reactive-event.merge-concurrency=64`
## Payloads, throttling, testing

Payloads
- Typed payloads for domain safety; Map for orchestration state (`offset`, `limit`, `fromBeginning`).
- Helper for safe cast:
```java
@SuppressWarnings("unchecked")
private Map<String, Object> asMap(Object p) {
    return p instanceof Map<?, ?> m ? (Map<String, Object>) m : Map.of();
}
```

Throttling & retries
- Throttle: `delayIt().by(Duration.ofMillis(250))`
- Backoff: `onFailure().retry().withBackOff(min, max).atMost(n)`

Testing
- Collect outputs from `handler.apply(Multi.createFrom().items(action))` and assert types/payloads.
