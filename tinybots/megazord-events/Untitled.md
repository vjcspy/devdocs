- Chính SensaraEventSource chịu trách nhiệm mở SSE tới Sensara: nó nạp lastEventId, gọi REST registerStream/
    getStream, sau đó gắn listener cho các event Sensara như AdlEventResponse, NotificationResponse,
    StateExtramuralResponse và chuyển từng event cho callback đã truyền (src/eventsource/SensaraEventSource.ts:70,
    src/eventsource/SensaraEventSource.ts:96). Đây là điểm “Sensara gửi telemetry”.
  - Trong callback SensaraEventsJob.handleEvent, service parse payload, tra resident tương ứng, đóng gói
    thành SensaraEvent, rồi persist vào MySQL thông qua SensaraEventRepository.storeEvent. Repo này lo tạo/
    lấy sensara_event_schema và chèn bản ghi sensara_event nên đây là bước “sensara-adaptor persist” (src/jobs/
    SensaraEventsJob.ts:139, src/jobs/SensaraEventsJob.ts:154, src/repository/SensaraEventRepository.ts:42).
  - Sau khi lưu, job chuyển đổi event Sensara sang Tinybots event chuẩn (ADL, notification…) bằng convertEvent rồi
    dùng EventService.postEvent để gọi POST /internal/v1/events/robots/:robotId/incomings trên Event Service (tức
    megazord-events). Đây là bước “sensara-adaptor emit → megazord-events” (src/jobs/SensaraEventsJob.ts:161, src/
    jobs/SensaraEventsJob.ts:168, src/jobs/SensaraEventsJob.ts:179).

  Tóm lại, pipeline nằm hoàn toàn trong SensaraEventsJob + SensaraEventSource: SSE nhận event (Sensara),
  SensaraEventRepository lưu vào sensara_event, EventService phát Tinybots event sang megazord-events.