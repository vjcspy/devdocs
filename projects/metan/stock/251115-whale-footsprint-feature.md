# General Requirement

Phân loại theo thuộc tính đối tượng và cách tính(point-in-time, **accu**mulative, moving-window)

<aside>
👉Quy định viết tắt tên field trong code để phân biệt, các chỉ số nếu:

- tại thời điểm trong `candle` ⇒ không cần viết gì thêm, ví dụ high, low, close
- accumulative ⇒ đây là cộng dồn trong 1 khoảng thời gian(không phải trong bar/candles, ví dụ khi mình xem 1 range) `accum_`
- moving window ⇒ giả sử trượt 15: `mov_15_`
</aside>

<aside>
💡Nên nhớ các features này là dành cho phái sinh [VN30F1M](https://finance.vietstock.vn/chung-khoan-phai-sinh/VN30F1M/hdtl-tong-quan.htm) nên sẽ cần cộng theo tỷ trọng của 30 cổ phiếu. Tuy nhiên, trước mắt cứ tính cho từng cổ phiếu đã, bước cộng theo tỷ trọng khá đơn giản nên có thể làm cuối cùng.
</aside>

## Features
- `shark_buy_value`,  `shark_sell_value` mua bán của shark trong **`*candle`***
    
    Để theo value như này có vẻ không ổn, có thể nomalize với giá trị giao dịch trung bình 5 ngày (đã làm)
    
- `avg_price_shark_buys`, `avg_price_shark_sells` Mức giá giao dịch BÌNH QUÂN của các lệnh MUA/BÁN lớn.
    
    Cho một cái nhìn để biết được hành vi của shark là đang mua/bán giá thấp hay là mua giá cao,
    
    *Insight:* Nếu `avg_price_shark_buys > avg_price_shark_sells`, có thể phe mua lớn chấp nhận mua giá cao hơn phe bán lớn.
    
    Hiện tại đang implementing là giá mua trung bình accumulate từ đầu ngày tới thời điểm của nến đang xem.
    
- `avg_shark450_buy_value_5d`,`avg_shark900_buy_value_5d`, `avg_shark450_sell_value_5d`,`avg_shark900_sell_value_5d`
- `ratio_shark_buy_shell` , `ratio_buy_shark_sheep`, `ratio_shell_shark_sheep` cho biết tỷ lệ giữa mua bán của shark, giữa mua của shark và sheep và bán của shark và sheep
    - Có nên tính accumulation trong n candle cho chỉ số này không?
- **`shark_urgency_spread = (avg_price_shark_buys - avg_price_shark_sells) / vwap`** Chuẩn hóa bằng VWAP giúp so sánh được giữa các cổ phiếu. Spread dương lớn cho thấy phe mua lớn đang rất "hung hăng", sẵn sàng mua đuổi giá cao.