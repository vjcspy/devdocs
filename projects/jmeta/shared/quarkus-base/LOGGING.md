# Quarkus Base Logging Guide

This document explains how structured logging works in `shared/quarkus-base`, how to use the `kv(...)` helper for rich context, and how to customize serialization for specific classes (e.g., `ReactiveEventAction`).

## TL;DR
- Use `kv("key", value)` (or `kvs(...)`) inside SLF4J/Quarkus logger placeholders `{}` to attach structured fields.
- Message shows concise "key=value" pairs; JSON output includes proper nested fields.
- For certain types we register reducers so only important fields are logged (privacy + readability).

## Helpers

- `KvArg` implements `StructuredArgument` from quarkiverse-logging-json so it enriches the JSON output.
- `SA` provides:
  - `kv(String k, Object v)` and `kvs(Object... kvs)`
  - A lightweight registry to customize:
    - JSON serialization via `registerJsonWriter(Class<?>, JsonFieldWriter)`
    - Message formatting via `registerFormatter(Class<?>, MsgFormatter)`

### Example
Ensure you import the custom helpers you wrote:

Static import (recommended):
```java
import static com.vjcspy.quarkus.base.util.SA.kv;
import static com.vjcspy.quarkus.base.util.SA.kvs; // when needed
```

Or use fully qualified method calls:
```java
log.info("Process action {}|{}",
  com.vjcspy.quarkus.base.util.SA.kv("type", evt.getType()),
  com.vjcspy.quarkus.base.util.SA.kv("action", evt));
```

```java
log.info("Process action {}|{}", kv("type", evt.getType()), kv("action", evt));
```
Output:
- Message: `Process action type=ACTION_1|action={type=ACTION_1, correlationId=d07a...}`
- JSON (excerpt):
```json
{
  "message": "Process action {}|{}",
  "type": "ACTION_1",
  "action": { "type": "ACTION_1", "correlationId": "d07a..." }
}
```

## Custom reducers (Serialization + Message formatting)
To keep logs concise, we can define how particular classes are rendered.

Startup config: `com.vjcspy.quarkus.base.logging.LoggingConfig` registers reducers for `ReactiveEventAction`:
- JSON reducer keeps only `type` and `correlationId`.
- Message formatter renders `action={type=..., correlationId=...}`.

### Register your own
```java
@ApplicationScoped
class MyLoggingConfig {
  void onStart(@Observes StartupEvent ev) {
    SA.registerJsonWriter(MyType.class, (gen, key, value) -> {
      MyType v = (MyType) value;
      Map<String, Object> reduced = Map.of(
        "id", v.id(),
        "name", v.name()
      );
      gen.writeObjectField(key, reduced);
    });

    SA.registerFormatter(MyType.class, (k, value) -> {
      MyType v = (MyType) value;
      return k + "={id=" + v.id() + ", name=" + v.name() + "}";
    });
  }
}
```
Tips:
- Prefer immutable maps or LinkedHashMap for field order.
- Omit sensitive fields.
- Keep message formatting short; details live in JSON.

## Safety and fallbacks
- If no custom writer/formatter is registered, `KvArg` falls back to default behavior:
  - JSON: writes the whole object as-is using the underlying JSON generator.
  - Message: `key=String.valueOf(value)`.
- Null values are handled safely.

## Do/Don’t
- Do pass `kv(...)` instead of string concatenation to leverage structured logs.
- Don’t log entire domain objects; add a reducer or extract only necessary fields.
- Do ensure correlation IDs are set; they’re automatically propagated via filters and event manager.

## Related components
- Request correlation: `RequestLoggingFilter`, `CorrelationIdFilter`, `CorrelationIdResponseFilter`.
- Reactive events: `ReactiveEventManager`, `ReactiveEventAction`.
- Example usage: `TedBedEffect`.
