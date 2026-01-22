# EVE (Extraterrestrial Vegetation Evaluator)

## TL;DR

- EVE is a Java/Dropwizard service managing schedules for Tessa robots.
- Handles cron-based task scheduling for reminders, questions, music playlists, and scripts.
- Exposes REST APIs (v4, v5, v6) for users, robots, and internal services.
- Uses MySQL (typ-e database) via JDBI3 and SQS for change notifications.

## Service Purpose

EVE manages the schedules that determine when Tessa robots execute tasks. It allows:
- Users to create, update, and delete scheduled tasks for their robots
- Robots to fetch their own schedules
- Internal services to query and manage script/playlist schedules

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Java 11 |
| Framework | Dropwizard 2.1.4 |
| Build | Maven |
| Database | MySQL (typ-e) via JDBI3 |
| Auth | Kong (via kong-client) |
| Messaging | AWS SQS |
| DI | Google Guice |
| Scheduling | cron-utils + tiny-schedule |

## Project Structure

```
eve/
├── src/main/java/nl/tinybots/eve/
│   ├── EveApplication.java      # Main entry point
│   ├── EveConfiguration.java    # Dropwizard config
│   ├── EveModule.java           # Guice DI module
│   ├── model/                   # Domain models (Task, Schedule, Reminder, etc.)
│   │   ├── dto/                 # Data transfer objects
│   │   └── sqs/                 # SQS message models
│   ├── mapper/                  # Model mappers (entity <-> DTO)
│   ├── repository/              # Database access layer (JDBI)
│   ├── resource/                # REST API endpoints
│   ├── service/                 # Business logic
│   ├── validate/                # Input validation
│   └── util/                    # Utilities (cron, datetime, etc.)
├── src/test/                    # Unit & integration tests
├── docs/
│   └── eve.yaml                 # OpenAPI specification
├── config/                      # Checkstyle/Findbugs configs
├── ci/                          # CI/CD pipeline scripts
└── pom.xml                      # Maven config
```

## API Overview

### User Resources (Kong user auth)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v4/schedules/{robotId}` | Get robot schedule (excluding scriptV2) |
| GET | `/v5/schedules/{robotId}` | Get robot schedule (including scriptV2) |
| GET | `/v6/schedules/{robotId}` | Get robot schedule (v6 format with timezone support) |
| PUT | `/v4/schedules/{robotId}` | Add/update schedule item |
| PUT | `/v6/schedules/{robotId}` | Add/update schedule item (v6 format) |
| DELETE | `/v4/schedules/{robotId}` | Delete schedule item |

### Robot Resources (Kong robot auth)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v4/schedules/robot` | Robot fetches own schedule (excluding scriptV2) |
| GET | `/v5/schedules/robot` | Robot fetches own schedule (including scriptV2) |
| PUT | `/v4/schedules/robot/results` | Robot stores feedback/answers |

### Internal Resources (Service-to-service)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/internal/v4/schedules/music/{robotId}/{playlistId}` | Check playlist schedule |
| GET | `/internal/v4/schedules/scripts/{robotId}/{scriptId}` | Check script schedule |
| GET | `/internal/v5/schedules/scripts/{robotId}/{scriptId}` | Check scriptV2 schedule |
| DELETE | `/internal/v4/schedules/scripts/{robotId}/{scriptId}` | Unschedule script |
| DELETE | `/internal/v5/schedules/scripts/{robotId}/{scriptId}` | Unschedule scriptV2 |
| POST | `/internal/v6/schedules/count/search` | Count scheduled tasks per robot |

## Domain Model

### Task Types

| Type | Description |
|------|-------------|
| `reminder` | Text reminders with greeting, salutation, call-to-action |
| `question` | Questions for feedback collection |
| `music` | Music playlist scheduling |
| `script` | Legacy script execution |
| `scriptV2` | Script v2 execution |

### Schedule Format

Schedules use cron-style notation with fields:
- `minute`, `hour`, `month`, `dayOfMonth`, `dayOfWeek`
- `startTime`, `endTime` for bounded schedules
- `recurring` (server-computed)
- v6 adds: `recurrenceType`, `recurrenceInterval`, `dayOfWeekIndex`, `timeZone`

## Key Services

| Service | Responsibility |
|---------|---------------|
| `ScheduleService` | Core schedule CRUD operations |
| `ScriptService` | Legacy script management |
| `ScriptV2Service` | Script v2 management |
| `PermissionService` | User-robot authorization checks |
| `RobotTimeZoneService` | Timezone resolution for robots |
| `ChangeNotificationService` | SQS notifications on schedule changes |
| `AnswerService` | Feedback/answer storage |

## Data Storage

- **Database**: MySQL (typ-e) accessed via JDBI3
- **Tables**: task_schedules, task_categories, playlists, scripts, answers, etc.
- **Migrations**: Managed by Flyway in `typ-e` repository

## Integration Points

| Service | Direction | Purpose |
|---------|-----------|---------|
| Kong Gateway | Inbound | Authentication & request routing |
| typ-e MySQL | Read/Write | Persistent schedule storage |
| SQS | Outbound | Schedule change notifications |
| micro-manager | Consumer | Script execution on schedule triggers |
| wonkers-api/graphql | Consumer | User-facing schedule management |

## Development

### Prerequisites
- Java 11
- Maven
- Docker (for MySQL in dev/test)

### Build & Test

```bash
# Build
mvn clean package

# Run tests (unit only)
mvn test

# Run integration tests
mvn verify

# Run locally
java -jar target/EVE.jar server config.yml
```

### Configuration

Environment variables (see `application.yml`):
- `DB_URL`, `DB_USER`, `DB_PASSWORD` - MySQL connection
- `SQS_QUEUE_URL` - SQS notification queue
- `KONG_*` - Kong auth settings

## CI/CD

- Build: `ci/build-eve.yml`
- Test: `ci/test-eve.yml`
- Docker image built via `Dockerfile`
- Deployed as container to ECS/Kubernetes

## API Documentation

- OpenAPI spec: `docs/eve.yaml`
- API Blueprint: `docs/EVE.apib`

## Common Operations

### Add a schedule item
```bash
curl -X PUT "https://api.tinybots.academy/v4/schedules/{robotId}" \
  -H "Authorization: Bearer {token}" \
  -H "X-Time-Zone: Europe/Amsterdam" \
  -H "Content-Type: application/json" \
  -d '{"title": "Morning reminder", "type": "reminder", ...}'
```

### Query robot schedule
```bash
curl "https://api.tinybots.academy/v5/schedules/{robotId}?from=2024-01-01&until=2024-01-31" \
  -H "Authorization: Bearer {token}" \
  -H "X-Time-Zone: Europe/Amsterdam"
```

## Related Documentation

- Global Overview: `devdocs/tinybots/OVERVIEW.md`
- Database Schema: `devdocs/tinybots/typ-e/OVERVIEW.md`
- Script Execution: `devdocs/tinybots/micro-manager/OVERVIEW.md`
