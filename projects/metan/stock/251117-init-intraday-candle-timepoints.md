# 📋 [251117: 2025-11-17] - Init Intraday Candle Timepoints

## User Requirements

Fully and carefully read overview at: `devdocs/projects/metan/stock/OVERVIEW.md`

Do

- trong ngày chỉ giao dịch vào một khung thời gian cụ thể
- cổ phiếu ở các sàn khác nhau sẽ có thời gian khớp lệnh liên tục khác nhau. Ví dụ đối với sàn HNX thì không có ATO nên sẽ khớp lệnh liên tục từ 9:00 AM còn HSX thì có ATO nên khớp lệnh liên tục từ 9:15
- số lượng candle trả về kể cả ở TCBS cũng không nhất quán

Từ đó quyết định sẽ viết 1 script để **tạo ra một danh sách “intraday candle timestamps” (hh:mm) cho một timeframe cố định**, không chứa ngày, dùng để **normalize nhiều ngày giao dịch khác nhau** 

Cách làm, sẽ lấy candle trong 1 ngày từ TCBS, chuẩn hóa, rồi lưu vào database

- Intraday (vì không có ngày)

- Candle timepoints

- Time interval / timeframe

- Mang tính “template” vì bạn dùng làm chuẩn cho việc normalize



Yêu cầu:

1. Đẩy ra dạng json, lưu ý tên key sẽ có dạng: INTRADAY_CANDLE_TIMEPOINTS_TÊN_SÀN_INTERVAL ví dụ: INTRADAY_CANDLE_TIMEPOINTS_HSX_300 (300 seconds là 5 mins). Lưu key như vậy là để đẩy vào database có dạng key, value(json)

2. Ta đã có sẵn 1 table như sau
   ```sql
   create table stock_common_configuration
   (
       id         bigint                   default nextval('stock_common_configuration_id_seq'::regclass) not null,
       key        text                                                                                    not null,
       value      jsonb,
       created_at timestamp with time zone default CURRENT_TIMESTAMP                                      not null,
       updated_at timestamp with time zone                                                                not null,
       primary key (),
       unique ()
   )
   ```

3. Sử dụng supabase để lưu, đã cấu hình supabase đầy đủ, chỉ cần làm tương tự như sau

```python
from metan.supabase.client import supabase
supabase.table("stock_trading_feature_candles").upsert(chunk, on_conflict="symbol,time").execute()
```

4. Lấy dữ liệu candle từ TCBS, refer: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` method `price_candles_by_date` Tôi cần bạn đọc full file này một cách cẩn thận để hiểu rõ cách lấy dữ liệu. Cho biết cổ phiếu CEO sẽ là sàn HNX, cổ phiếu FPT là sàn HSX. Lấy luôn ngày 2025-11-17 để là ngày fetch các candle từ TCBS

5. Price Candle `time` đã là ISO UTC dạng `YYYY-MM-DDTHH:mm:ss.SSSZ` (ví dụ: `2025-11-14T07:27:00.000Z`). Không convert về timezone khác; dùng trực tiếp UTC để lấy `HH:mm`.

6. Implement class generator trong `packages/stock/metan/stock/info/helper`, viết trong package stock, không tạo CLI/entry ở bên ngoài. Bạn sẽ tự gọi nó ở chỗ khác, generator không tự chạy.

## 🎯 Objective
- Tạo template “intraday candle timepoints” theo từng sàn và interval cố định dựa trên TCBS candles của ngày `2025-11-17`, giữ nguyên UTC khi trích xuất `HH:mm`.
- Persist JSON vào `stock_common_configuration` với key chuẩn `INTRADAY_CANDLE_TIMEPOINTS_{EXCHANGE}_{INTERVAL}` để phục vụ normalize nhiều ngày giao dịch.

### ⚠️ Key Considerations
- Khác biệt phiên giao dịch: HSX có ATO nên continuous bắt đầu 09:15; HNX bắt đầu 09:00; midday break/ATC được phản ánh tự nhiên qua số lượng slot TCBS.
- Time handling: parse ISO `Z` (UTC) và trích xuất `HH:mm` từ UTC; tuyệt đối không convert timezone.
- Interval: ưu tiên `300s` trước, nhưng generator chấp nhận các interval khác nếu TCBS hỗ trợ.
- Candle count variability: dùng chính danh sách slot TCBS trong ngày mục tiêu, không tự gap-fill.
- Persistence: ưu tiên `upsert on_conflict="key"`; nếu thiếu unique index, dùng delete-by-key rồi insert.
- Traceability: thêm metadata tối thiểu trong `value.json` (`symbol`, `date`, `interval`, `source`).

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: Template theo sàn+interval, dựa CEO/HNX và FPT/HSX ngày `2025-11-17`; giữ UTC; persist JSON với key định danh.
- [ ] Define scope and edge cases
  - **Outcome**: Xử lý trường hợp thiếu candles; đảm bảo thứ tự và uniqueness `HH:mm`; hỗ trợ `300s`; fallback khi `on_conflict` không khả dụng.

### Phase 2: Implementation (File/Code Structure)
> Đặt generator class trong package stock, không viết ra ngoài. Trạng thái: ✅ Implemented, 🚧 To-Do, 🔄 In Progress.

```
packages/stock/metan/stock/
├── info/
│   ├── helper/
│   │   ├── __init__.py                          # ✅ IMPLEMENTED - package init
│   │   └── intraday_timepoints_generator.py     # 🚧 TODO - Generator class (no CLI)
│   └── domain/
│       └── stock_data_collector/
│           └── stock_data_collector.py          # ✅ IMPLEMENTED - TCBS candle access
```

- Reuse `price_candles_by_date()` để lấy candles theo ngày: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py:355`.
- `IntradayTimepointsGenerator` chịu trách nhiệm: fetch → chuyển đổi `time` ISO UTC → `HH:mm` → tạo payload JSON → persist vào Supabase.

### Phase 3: Detailed Implementation Steps
- Fetch TCBS candles bằng `StockDataCollector`:
  - HSX baseline: `symbol="FPT"`, `interval=300`, `start_date=end_date="2025-11-17"`; lấy danh sách cho ngày `2025-11-17` từ `price_candles_by_date()`.
  - HNX baseline: `symbol="CEO"`, `interval=300`, `start_date=end_date="2025-11-17"`; tương tự.
  - Tham chiếu: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py:355`.
- Extract intraday timepoints:
  - Với mỗi candle `time` (ví dụ `2025-11-14T07:27:00.000Z`), parse ở UTC và format `HH:mm` trực tiếp từ UTC.
  - Deduplicate, sort ascending, không gap-fill.
- Build JSON payloads:
  - HSX key: `INTRADAY_CANDLE_TIMEPOINTS_HSX_300`
    - `value`: `{ "timepoints": ["HH:mm", ...], "interval": 300, "source": "TCBS", "symbol": "FPT", "date": "2025-11-17" }`
  - HNX key: `INTRADAY_CANDLE_TIMEPOINTS_HNX_300`
    - `value`: `{ "timepoints": ["HH:mm", ...], "interval": 300, "source": "TCBS", "symbol": "CEO", "date": "2025-11-17" }`
- Persist vào Supabase (`stock_common_configuration`):
  - Ưu tiên `upsert({ key, value }, on_conflict="key")`.
  - Nếu schema thiếu unique index cho `key`, thực hiện `delete where key = ...` rồi `insert` để đảm bảo idempotency.
- Validation checklist:
  - Số lượng `HH:mm` khớp số candles TCBS của ngày.
  - Timepoint đầu/cuối phản ánh slot theo phiên TCBS.
  - Chạy lại cho kết quả deterministic.

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and requested.

### ✅ Completed Achievements
- [Reserved until implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Xác nhận `stock_common_configuration` có unique index trên `key` để dùng upsert.
- [ ] Consumer phải coi `HH:mm` là UTC; không convert về local.

### 🔮 Future Improvements (Optional)
- [ ] Hỗ trợ thêm các interval (`60s`, `900s`) và sàn khác.
- [ ] Lưu metadata phiên (start/end, breaks) phục vụ chuẩn hóa sâu hơn.
- [ ] Thêm guardrails nếu quy tắc slot TCBS thay đổi.

   

