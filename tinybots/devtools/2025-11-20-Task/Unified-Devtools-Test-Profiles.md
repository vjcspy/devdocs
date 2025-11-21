# 📋 [UNIFIED-DEVTOOLS-TEST-PROFILES: 2025-11-20] - Centralize docker-compose test flows via profiles/just


## References

> Any source or file you reference must adhere to the following rule: If it is a file, the path must be cited exactly as provided by the user to ensure seamless accessibility for other AI Agents.

## User Requirements

> This section contains the original requirements from the user. If the user defined them, leave them as they are; otherwise, you can omit this section.

- Muốn hợp nhất việc chạy test cho các service (ban đầu mỗi service có `ci/docker-compose.yml` riêng) vào một compose chung `devtools/docker-compose.yaml` dùng profile. Cần demo trước cho 2 service `megazord-events` và `wonkers-graphql` dựa trên các file compose cũ (`megazord-events/ci/docker-compose.yml`, `wonkers-graphql/ci/docker-compose.yml`) để tạo profile chính xác.
- Muốn dùng `just` (hoặc script tương đương) để chạy lệnh test; test của từng service phải khởi động đúng nhóm container cần thiết trong compose chung.
- Yêu cầu vòng đời container khi test:
  - Khi chạy test chỉ cần xem log của node container test; không cần hiển thị log của các container phụ trợ.
  - Quan tâm tới kết quả test: sau khi test xong, lệnh test phải kết thúc; node container test auto-stop, các container phụ trợ khác phải tiếp tục chạy.
  - Cách cũ attach log qua `docker attach $(docker ps -q --filter=label=wonkers-graphql)`; cần thiết kế cách mới đáp ứng cùng mục tiêu (xem log đúng container, nhận exit code test).
- Yêu cầu reset DB sạch trước mỗi lần test: xóa 2 container MySQL (`mysql-db`, `wonkers-mysql-db`) rồi khởi động lại toàn bộ container theo profile để flyway/typ-e/wonkers-db init lại DB.
- Kết quả mong muốn: có cách chạy test megazord-events và wonkers-graphql dùng compose chung với profile; script test giữ nguyên dịch vụ phụ trợ sau khi node test dừng.
- Lệnh tách riêng:
  - `start-<profile>`: bật toàn bộ dependencies của profile (trừ node), xem log của tất cả container khi được up.
  - `test-<profile>`: chạy reset-db cho profile đó, sau đó chỉ run node container test (log node, lấy exit code).
  - `log-<profile>`: xem log phụ trợ để debug khi cần.

## 🎯 Objective
> Briefly describe the key objective to be achieved with this ticket.

Thiết kế và đặt kế hoạch áp dụng profile trong `devtools/docker-compose.yaml` cùng bộ lệnh `just`/script để chạy test cho `megazord-events` và `wonkers-graphql`: khởi tạo đúng phụ trợ, xem log khi khởi động, reset DB sạch mỗi run, node test container tự dừng còn phụ trợ giữ nguyên.

### ⚠️ Key Considerations
> This describes the extremely important points or reasons that need attention

- Chuẩn hóa tên DB về `mysql-dashboard-db` và `mysql-type-db`; cập nhật host/link tương ứng (`wonkers-mysql-db` → `mysql-dashboard-db`, `mysql-db` → `mysql-type-db`). Giữ nguyên naming theo mong đợi của `wonkers-graphql`.
- Quản lý log: khi test chỉ stream log node container; khi start profile cần log toàn bộ container; `log-<profile>` phục vụ debug phụ trợ.
- Reset DB an toàn: chỉ `docker compose rm -sf mysql-dashboard-db mysql-type-db`, không ảnh hưởng container phụ trợ khác.
- Thứ tự khởi động: node test phải chạy sau khi `typ-e` và `wonkers-db` hoàn tất migrate; dùng `depends_on` với `condition: service_completed_successfully`. Không cần healthcheck.
- Profiles tách biệt để không boot thừa service cho từng test.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: Bảng ánh xạ dependency cho `megazord-events` và `wonkers-graphql` (DB, localstack, checkpoint/prowl/wonkers/wonkers-account), hostname cần alias.
- [ ] Define scope and edge cases
  - **Outcome**: Danh sách edge cases: mismatch alias host, container chưa healthcheck nhưng node chạy sớm, log treo terminal, cleanup DB làm mất state mong muốn khác.

### Phase 2: Implementation (File/Code Structure)
> Describe the proposed file/directory structure, including the purpose of each key component. Remember use status markers like ✅ (Implemented), 🚧 (To-Do), 🔄 (In Progress).

```
devtools/
├── docker-compose.yaml            # 🚧 TODO - Thêm profiles megazord-events & wonkers-graphql, chuẩn hóa tên DB, depends_on service_completed_successfully cho typ-e/wonkers-db
├── Justfile                       # 🚧 TODO - Lệnh start-<profile> (up deps + log tất), test-<profile> (reset DB + run node, log node), log-<profile> (tail log phụ trợ)
│                                   #          Ví dụ: just start megazord-events / test megazord-events / log megazord-events
└── scripts/                       # 🚧 TODO - Helper shell (nếu cần) để attach log và run node test (docker compose run --rm)
```

### Phase 3: Detailed Implementation Steps

1) Chuẩn hóa compose
- [ ] Trích dependency từ `megazord-events/ci/docker-compose.yml` và `wonkers-graphql/ci/docker-compose.yml`; xác định service bắt buộc, optional.
- [ ] Thêm `profiles` vào `devtools/docker-compose.yaml` cho từng nhóm; đặt `depends_on` với `condition: service_completed_successfully` cho `typ-e` và `wonkers-db` (flyway job).
- [ ] Đổi tên dịch vụ DB trong compose chung thành chuẩn: `mysql-dashboard-db` (thay cho `wonkers-mysql-db`) và `mysql-type-db` (thay cho `mysql-db`); cập nhật toàn bộ `environment`/`links`/host tham chiếu của phụ trợ và node test.
- [ ] Không thêm healthcheck; dựa vào completion của flyway để đảm bảo DB sẵn sàng.

2) Chiến lược log & chạy test
- [ ] `start-<profile>`: `docker compose up -d --profile <profile> <deps>` rồi `docker compose logs -f <deps>` để xem log tất cả phụ trợ khi khởi động (không chạy node).
- [ ] `test-<profile>`: `docker compose rm -sf <mysqls>` để reset DB, `docker compose up -d --profile <profile> <deps>` nếu chưa chạy, sau đó `docker compose run --rm --no-deps --use-aliases node-<svc>` và chỉ stream log node, lấy exit code test.
- [ ] `log-<profile>`: `docker compose logs -f <deps>` phục vụ debug phụ trợ khi cần, tách biệt với lệnh test.

3) Script/Just recipes
- [ ] Viết recipe `reset-db` (rm -sf mysql-type-db mysql-dashboard-db) trước mỗi test.
- [ ] Viết recipe `start-<profile>` (up -d phụ trợ + stream log tất cả khi khởi động, không launch node).
- [ ] Viết recipe `test-<profile>` (reset DB + up deps nếu cần + run node với log node).
- [ ] Viết recipe `log-<profile>` để xem log phụ trợ khi debug, không chạy test.

4) Tài liệu nhanh
- [ ] Ghi chú cách chạy `just test-megazord-events` / `just test-wonkers-graphql`, cảnh báo reset DB, cách thêm profile mới.

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- Chuẩn hóa mount volume cho `localstack` trong `devtools/docker-compose.yaml`
- [List major accomplishments]

## 🚧 Outstanding Issues & Follow-up
> If you have any outstanding issues or any question needs to clarify, list them here. Otherwise, you can omit this section.
### ⚠️ Issues/Clarifications (Optional)
- [ ] [Issue 1 – Describe and note impact]
- [ ] [Issue 2 – Describe and note impact]
