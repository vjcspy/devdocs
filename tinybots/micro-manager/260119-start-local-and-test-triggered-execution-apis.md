# [260119] - Start micro-manager local + test Triggered Execution APIs

## Mục tiêu

- Start được `micro-manager` ở local (Docker dependencies + chạy service).
- Có seed data tối thiểu để test 2 API:
  - Robot: store triggered execution
  - User: list triggered executions

## Prerequisites

- Docker + Docker Compose v2
- `just`

## 1) Start dependencies bằng DevTools

Chạy từ repo root `tinybots/`:

```bash
cd devtools/tinybots/local
just start-micro-manager
```

Lưu ý:
- `just start-micro-manager` sẽ chạy `docker compose up -d ...` rồi tail logs (`docker compose logs -f ...`) nên terminal sẽ “kẹt” ở chế độ xem logs.
- Có thể để nguyên (mở ở 1 terminal riêng), hoặc `Ctrl+C` sau khi thấy containers đã `Up` để quay lại prompt (containers vẫn chạy vì đã `up -d`).

Dependencies cần cho `micro-manager` (đã nằm trong `start-micro-manager`):
- `mysql-typ-e-db` (tinybots DB)
- `typ-e` (migrations)
- `checkpoint` (robot auth)
- `prowl` (user auth)

## 2) Seed data cho Triggered Execution APIs

Seed scope: `micro-manager-triggered`

Chạy từ repo root `tinybots/`:

```bash
cd devtools/tinybots/local && npm run generate && npm run seed:clean && npm run seed -- --scope=micro-manager-triggered
```

Seed này tạo các record deterministic để dùng cho curl:
- `robotId = 1`
- `userId = 1`
- `scriptReferenceId = 9001`
- `scriptVersionId = 9001`
- `scriptStepIds = [9101, 9102]`
- `triggeringEventId = 9701`
- `triggerName = "seed.micro-manager.trigger"`

## 3) Start micro-manager

### Option A (khuyến nghị): chạy trong Docker (DevTools)

```bash
cd devtools/tinybots/local
just dev-micro-manager
```

Nếu gặp lỗi `Bind for 0.0.0.0:18080 failed: port is already allocated` thì đang có service/container khác chiếm port `18080`.
- Tìm process đang listen: `lsof -nP -iTCP:18080 -sTCP:LISTEN`
- Nếu là Docker container micro-manager cũ: `docker ps | grep 18080` rồi `docker stop <container_name_or_id>`

Service sẽ listen tại:
- `http://localhost:18080`

### Option B: chạy trực tiếp trên host

Nếu muốn chạy ngoài Docker (vẫn dùng DB + checkpoint/prowl từ DevTools):

```bash
cd micro-manager
yarn install

DB_RW_HOST=127.0.0.1 \
DB_PORT=1123 \
CHECKPOINT_ADDRESS=http://127.0.0.1:3002 \
PROWL_ADDRESS=http://127.0.0.1:3001 \
PUBLIC_BOT_ID=999999 \
yarn dev
```

## 4) Curl test 2 API

### 4.1 Robot API - Store Triggered Script Execution

Endpoint:
- `PUT /v6/scripts/robot/scripts/:scriptReferenceId/executions/triggered/:triggeringEventId`

Example:

```bash
curl -i -X PUT \
  "http://localhost:18080/v6/scripts/robot/scripts/9001/executions/triggered/9701" \
  -H "content-type: application/json" \
  -H "x-consumer-id: f4823fd5-1f11-4b4a-9959-70701327bfe4" \
  -H "x-consumer-username: tinybots-robots" \
  -H "x-authenticated-scope: robot" \
  -H "x-authenticated-userid: 1" \
  -d '{
    "scriptVersionId": 9001,
    "scriptExecutionSteps": [
      {
        "scriptStepId": 9101,
        "stepType": "say",
        "nextScriptStepId": 9102,
        "executedAt": "2026-01-01T00:00:00.000Z",
        "data": null
      },
      {
        "scriptStepId": 9102,
        "stepType": "say",
        "nextScriptStepId": null,
        "executedAt": "2026-01-01T00:00:01.000Z",
        "data": null
      }
    ]
  }'
```

Expected:
- `204 No Content`

### 4.2 User API - List Triggered Executions (Schedule View)

Endpoint:
- `GET /v6/scripts/user/robots/:robotId/executions/triggered?from=...&to=...&limit=...`

Example (range rộng để luôn match `executed_at`):

```bash
curl -s \
  "http://localhost:18080/v6/scripts/user/robots/1/executions/triggered?from=2000-01-01T00:00:00.000Z&to=2100-01-01T00:00:00.000Z&limit=50" \
  -H "x-consumer-id: f4823fd5-1f11-4b4a-9959-70701327bfe4" \
  -H "x-consumer-username: tinybots-users" \
  -H "x-authenticated-scope: user" \
  -H "x-authenticated-userid: 1" | jq .
```

Note:
- `executedAt` trong response list là timestamp lúc hệ thống ghi nhận execution (không nhất thiết trùng với `executedAt` của từng step trong payload PUT ở 4.1).

Expected:
- `200 OK`
- `triggeredExecutions` chứa ít nhất 1 item (được tạo bởi curl PUT phía trên) và `trigger.triggerName = "seed.micro-manager.trigger"`.

### 4.3 User API - Get Execution Detail (Detail View)

Endpoint:
- `GET /v6/scripts/user/robots/:robotId/executions/:executionId`

Note:
- `executionId` chính là `script_execution.id` và với triggered execution thì trùng với `triggeredExecutionId` trả về từ API list (4.2).

Example:

```bash
curl -s \
  "http://localhost:18080/v6/scripts/user/robots/1/executions/<executionId>" \
  -H "x-consumer-id: f4823fd5-1f11-4b4a-9959-70701327bfe4" \
  -H "x-consumer-username: tinybots-users" \
  -H "x-authenticated-scope: user" \
  -H "x-authenticated-userid: 1" | jq .
```

Trong đó:
- `<executionId>` = `triggeredExecutions[0].triggeredExecutionId` từ response của API list ở 4.2.

Expected:
- `200 OK`
- `executionType = "triggered"`
- Có `trigger.triggerName = "seed.micro-manager.trigger"`
- `scriptExecutionSteps` chứa các step executions đã gửi lên ở curl PUT (4.1)
