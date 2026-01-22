# 📋 [PROD-XXX: 2026-01-21] - Soft Delete Recurring Schedule Series

## References

- **Target Repository**: `eve` (Java/Dropwizard)
- **Database Schema**: `typ-e`
- **Key Files**:
  - `eve/src/main/java/nl/tinybots/eve/resource/DeleteV4ScheduleResource.java`
  - `eve/src/main/java/nl/tinybots/eve/service/ScheduleService.java`
  - `eve/src/main/java/nl/tinybots/eve/repository/TaskRepository.java`
  - `eve/src/main/java/nl/tinybots/eve/util/ScheduleUtils.java`

## User Requirements

> **From Stakeholder:**
>
> **Relevance:**
> If on my.tinybots.academy you plan a recurring task. Then go let your robot execute a task. After it is executed you go to the future and delete the whole series, all executions are also deleted.
>
> We do not want that! So make sure the old executions stay and only tasks from that point are deleted.
>
> **Solution:**
> When the delete series command is used it should only delete items in the series in the future. If you delete from 2026-05-05 then only tasks after 2026-05-05 should be deleted. If you select a value in the past: 2020-10-10 it should be blocked.
>
> **Endpoint:**
> ```
> DELETE https://api.tinybots.academy/v4/schedules/{robotId}
> ```

## 🎯 Objective

Modify the DELETE schedule endpoint to **soft delete** recurring series by setting `end_at` on `task_schedule` instead of hard deleting `robot_schema`. This preserves historical execution records while preventing future task occurrences.

### ⚠️ Key Considerations

#### 1. Database Relationship Understanding

```
┌──────────────────┐        ┌─────────────────┐        ┌───────────────────┐
│   robot_schema   │        │  task_schedule  │        │ script_execution  │
│   (CHILD)        │───FK──►│   (PARENT)      │◄──FK───│   (CHILD)         │
│                  │        │                 │        │                   │
│ id               │        │ id              │        │ id                │
│ robot_id         │        │ start_at        │        │ schedule_id (FK)  │
│ schedule_id (FK) │        │ end_at ←────────┼────────│ planned           │
│ script_v2_task_id│        │ minute, hour... │        │ script_ref_id     │
└──────────────────┘        └─────────────────┘        └───────────────────┘
```

- **FK Constraints**: All are `RESTRICT` (no CASCADE)
- **Current behavior**: Deleting `robot_schema` leaves `task_schedule` orphaned but `script_execution` records intact
- **Problem**: Historical data relationship is broken, UI may not display correctly

#### 2. Evidence that `end_at` Controls Future Scheduling

The following code analysis proves that setting `end_at` will prevent future task occurrences:

##### 2.1 Query Filter - Schedules with `end_at <= now` are excluded

**File**: `eve/src/main/java/nl/tinybots/eve/repository/TaskRepository.java` (lines 57-61)
```java
// findRobotSchedule() - Robot's schedule query
+ "WHERE rs.`robot_id` = :robotId "
+ "AND (rs.script_v2_task_id IS NULL) "
+ "AND (ts.`end_at` > :now OR ts.`end_at` IS NULL)")  // ← KEY FILTER
```

**Impact**: Robot will NOT receive schedules where `end_at` has passed.

##### 2.2 `mightFire()` - Prevents scheduling after endTime

**File**: `eve/src/main/java/nl/tinybots/eve/util/ScheduleUtils.java` (lines 145-153)
```java
public static boolean mightFire(V6Schedule schedule, Robot robot, ZonedDateTime now) {
    ZonedDateTime start = schedule.getStartTime() == null ? now : schedule.getStartTime();
    if (schedule.getEndTime() == null) {
        return true;
    }
    ZonedDateTime end = schedule.getEndTime();
    ZonedDateTime nextFire = getNextOccurence(schedule, start);
    return nextFire != null && nextFire.isBefore(end);  // ← Returns false if nextFire >= end
}
```

**Impact**: New occurrences will NOT be created after `endTime`.

##### 2.3 `explode()` - Calendar view respects endTime

**File**: `eve/src/main/java/nl/tinybots/eve/util/ScheduleUtils.java` (lines 175-183)
```java
// Similarly we need to stop sooner when the until is after the Tasks endTime.
ZonedDateTime correctedUntil = schedule.getEndTime() != null && until.isAfter(schedule.getEndTime())
    ? schedule.getEndTime() : until;  // ← Caps at endTime

ZonedDateTime time = getNextOccurence(task.getSchedule(), correctedFrom);
while (time != null && time.isBefore(correctedUntil)) {  // ← Only creates occurrences before endTime
    result.add(withTime(task, time));
    time = getNextOccurence(task.getSchedule(), time);
}
```

**Impact**: Calendar will NOT display occurrences after `endTime`.

##### 2.4 Existing Pattern - `deleteScheduledScripts()` already uses this approach

**File**: `eve/src/main/java/nl/tinybots/eve/repository/TaskRepository.java` (lines 327-332)
```java
@SqlUpdate("UPDATE `task_schedule` AS ts "
    + "JOIN robot_schema AS rs ON (rs.schedule_id = ts.id)"
    + "JOIN script_task AS st ON (rs.script_task_id = st.`id`) "
    + "SET ts.`end_at` = NOW() "  // ← EXISTING PATTERN!
    + "WHERE rs.`robot_id` = :robotId AND st.`script_robot_id` = :scriptId "
    + "AND (ts.`end_at` > NOW() OR ts.`end_at` IS NULL)")
int deleteScheduledScripts(@Bind("scriptId") Long scriptId, @Bind("robotId") Long robotId);
```

**Impact**: This pattern is **already proven** in production for script unscheduling.

##### 2.5 Summary Table

| Checkpoint | Status | Evidence Location |
|------------|--------|-------------------|
| Robot không nhận schedule sau `end_at` | ✅ | `TaskRepository.findRobotSchedule()` SQL filter |
| Calendar không hiển thị sau `end_at` | ✅ | `ScheduleUtils.explode()` caps at endTime |
| Không tạo new executions sau `end_at` | ✅ | `ScheduleUtils.mightFire()` returns false |
| Pattern đã được sử dụng trong production | ✅ | `TaskRepository.deleteScheduledScripts()` |

#### 3. Business Rules

- `fromDate` parameter **MUST** be in the future (or today)
- If `fromDate` is in the past → return **400 Bad Request**
- Historical `script_execution` records must remain intact
- `robot_schema` record should **NOT** be deleted (to maintain relationships)

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze database schema and FK relationships
  - **Outcome**: `script_execution` → `task_schedule` (FK), no CASCADE delete
- [x] Analyze current DELETE logic flow
  - **Outcome**: Hard delete on `robot_schema`, `task_schedule` orphaned
- [x] Verify `end_at` behavior across codebase
  - **Outcome**: All queries filter by `end_at`, pattern already used in `deleteScheduledScripts()`
- [ ] Define edge cases:
  - `fromDate` in the past → Block with 400
  - `fromDate` = today → Allow (delete from today onwards)
  - Schedule already ended (`end_at` < `fromDate`) → No-op or 404
  - Non-recurring schedule → Delete single occurrence (existing behavior)
- [ ] Review existing integration tests
  - **Outcome**: `DeleteV4ScheduleResourceIT.java` has tests for single occurrence delete

### Phase 2: Implementation (File Structure)

```
eve/src/main/java/nl/tinybots/eve/
├── resource/
│   └── DeleteV4ScheduleResource.java       # 🔄 UPDATE - Add fromDate validation
├── service/
│   └── ScheduleService.java                # 🔄 UPDATE - Change delete logic
├── repository/
│   └── TaskRepository.java                 # 🔄 UPDATE - Add updateScheduleEndAt method
├── model/dto/
│   └── TaskIdentifierDto.java              # 🔄 UPDATE - Add fromDate field (optional)
├── validate/
│   └── InFuture.java                       # ✅ EXISTS - Reuse for fromDate validation

eve/src/test/java/nl/tinybots/eve/
├── resource/
│   └── DeleteV4ScheduleResourceIT.java     # 🔄 UPDATE - Add soft delete tests
├── service/
│   └── ScheduleServiceTest.java            # 🔄 UPDATE - Add unit tests
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Update `TaskIdentifierDto` - Add `fromDate` field

**File**: `eve/src/main/java/nl/tinybots/eve/model/dto/TaskIdentifierDto.java`

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class TaskIdentifierDto {
    
    @Min(1)
    @NotNull
    private Long id;
    
    @InFuture
    private ZonedDateTime time;  // Existing - for single occurrence delete
    
    @InFuture
    private ZonedDateTime fromDate;  // NEW - for series soft delete
}
```

**Behavior**:
- If `time` is provided → Delete single occurrence (existing behavior)
- If `fromDate` is provided (and `time` is null) → Soft delete series from `fromDate`
- If neither provided → Soft delete series from NOW (backward compatible)

#### Step 2: Add Repository Method

**File**: `eve/src/main/java/nl/tinybots/eve/repository/TaskRepository.java`

```java
/**
 * Soft delete a schedule by setting end_at to the specified date.
 * Only updates if current end_at is after the new end date or is null.
 */
@SqlUpdate("UPDATE `task_schedule` AS ts "
    + "SET ts.`end_at` = :endAt "
    + "WHERE ts.`id` = :scheduleId "
    + "AND (ts.`end_at` > :endAt OR ts.`end_at` IS NULL)")
int updateScheduleEndAt(@Bind("scheduleId") Long scheduleId, @Bind("endAt") ZonedDateTime endAt);
```

#### Step 3: Update `ScheduleService.delete()` Method

**File**: `eve/src/main/java/nl/tinybots/eve/service/ScheduleService.java`

```java
public List<Task> delete(@NonNull Task task, @NonNull Robot robot, 
                         @NonNull ZonedDateTime now, DateTimeZone robotTz) {
    Task toBeDeleted = taskRepository.findRobotTaskById(robot.getId(), task.getId());
    if (toBeDeleted == null) {
        throw new NotFoundException(...);
    }

    toBeDeleted.getSchedule().setTimeZone(ZoneId.of(robotTz.getID()));

    // Case 1: Delete single occurrence (existing behavior - unchanged)
    if (task.getTime() != null) {
        // ... existing split series logic ...
    }
    
    // Case 2: Soft delete series from a specific date
    ZonedDateTime fromDate = task.getFromDate() != null ? task.getFromDate() : now;
    
    // Validate fromDate is not in the past
    if (fromDate.isBefore(now)) {
        throw new BadRequestException("Cannot delete schedule series from a past date");
    }
    
    // Soft delete by setting end_at
    Long scheduleId = toBeDeleted.getSchedule().getId();
    int updated = taskRepository.updateScheduleEndAt(scheduleId, fromDate);
    
    if (updated == 0) {
        // Schedule was already ended before fromDate
        throw new NotFoundException("Schedule already ended or not found");
    }
    
    return Lists.newLinkedList();
}
```

#### Step 4: Update `DeleteV4ScheduleResource`

**File**: `eve/src/main/java/nl/tinybots/eve/resource/DeleteV4ScheduleResource.java`

```java
@DELETE
@Timed
public void deleteScheduledTask(
    @NotNull @HeaderParam(EveConstants.TZ_HEADER) String timeZoneId,
    @PathParam("robotId") Long robotId, 
    @Auth TinyPrincipal user, 
    TaskIdentifierDto taskDto) {
    
    Robot robot = ((User) user).getRobots().get(robotId);
    if (robot == null) {
        throw new ForbiddenException("Not allowed to manage robot with id " + robotId);
    }
    
    DateTimeZone robotTz = robotTimeZoneService.getRobotTimeZone(robot);
    ResourceUtils.checkTimezone(robotTz, timeZoneId);

    Task task = taskIdentifierDtoMapper.map(taskDto, robot, robotTz);
    task.setRobotId(robotId);
    
    scheduleService.delete(task, robot, robotTz);
    changeNotificationService.notifyChange(robot);
}
```

*(Mostly unchanged - validation happens in DTO and Service layer)*

#### Step 5: Update Mapper

**File**: `eve/src/main/java/nl/tinybots/eve/mapper/TaskIdentifierDtoMapper.java`

Update to map the new `fromDate` field from DTO to Task model.

#### Step 6: Integration Tests

**File**: `eve/src/test/java/nl/tinybots/eve/resource/DeleteV4ScheduleResourceIT.java`

```java
@Test
@DataSet("data/DeleteV4ScheduleResourceIT/recurringScheduleWithExecutions.sql")
public void softDeleteSeriesFromFutureDate_shouldPreserveHistoricalExecutions() {
    // Given: A recurring schedule with some past executions
    
    // When: Delete series from a future date
    TaskIdentifierDto deleteRequest = new TaskIdentifierDto();
    deleteRequest.setId(scheduleId);
    deleteRequest.setFromDate(ZonedDateTime.now(NL).plusDays(7));
    
    given()
        .contentType(ContentType.JSON)
        .header("X-Time-Zone", "Europe/Amsterdam")
        .body(deleteRequest)
        .delete("/v4/schedules/" + robotId)
    .then()
        .statusCode(204);
    
    // Then: Schedule end_at is updated
    // And: Historical executions still exist
    // And: Future occurrences are not returned
}

@Test
public void softDeleteSeriesFromPastDate_shouldReturn400() {
    TaskIdentifierDto deleteRequest = new TaskIdentifierDto();
    deleteRequest.setId(scheduleId);
    deleteRequest.setFromDate(ZonedDateTime.now(NL).minusDays(7));
    
    given()
        .contentType(ContentType.JSON)
        .header("X-Time-Zone", "Europe/Amsterdam")
        .body(deleteRequest)
        .delete("/v4/schedules/" + robotId)
    .then()
        .statusCode(400);
}

@Test
public void softDeleteSeriesWithoutFromDate_shouldDeleteFromNow() {
    // Given: A recurring schedule
    
    // When: Delete series without fromDate (backward compatible)
    TaskIdentifierDto deleteRequest = new TaskIdentifierDto();
    deleteRequest.setId(scheduleId);
    // No fromDate - defaults to NOW
    
    given()
        .contentType(ContentType.JSON)
        .header("X-Time-Zone", "Europe/Amsterdam")
        .body(deleteRequest)
        .delete("/v4/schedules/" + robotId)
    .then()
        .statusCode(204);
    
    // Then: Schedule end_at is set to approximately NOW
}
```

### Phase 4: Testing Checklist

- [ ] Unit tests for `ScheduleService.delete()` with `fromDate`
- [ ] Integration test: Soft delete from future date preserves executions
- [ ] Integration test: Soft delete from past date returns 400
- [ ] Integration test: Soft delete without `fromDate` defaults to NOW
- [ ] Integration test: Single occurrence delete still works (regression)
- [ ] Manual test on staging: Create recurring task → Execute → Delete series → Verify executions remain

## 📊 Summary of Results

> *To be completed after implementation*

### ✅ Completed Achievements

- [ ] Soft delete implemented using `end_at` pattern
- [ ] Historical executions preserved
- [ ] Backward compatible (no `fromDate` = delete from NOW)
- [ ] Past date validation working
- [ ] All integration tests passing

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Open Questions

1. **API Contract Change**: Adding `fromDate` to request body is backward compatible, but should we document this change in API specs (`docs/eve.yaml`)?

2. **Frontend Update**: Does `my.tinybots.academy` need to be updated to pass `fromDate` when user selects a specific date to delete from? Or should it always default to NOW?

3. **Notification**: Should we notify user/admin when a series is soft-deleted vs hard-deleted?
