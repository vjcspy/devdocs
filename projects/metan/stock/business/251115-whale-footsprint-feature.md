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
## Current Implementation

Hiện tại đã có các features sau

```json
{
  "date" : "2025-11-03",
  "candle_volume" : 172000,
  "candle_value" : 4311.0,
  "shark450_buy_value" : 480.0,
  "sheep450_buy_value" : 2517.0,
  "shark450_sell_value" : 0.0,
  "sheep450_sell_value" : 1282.0,
  "shark900_buy_value" : 0.0,
  "sheep900_buy_value" : 2997.0,
  "shark900_sell_value" : 0.0,
  "sheep900_sell_value" : 1282.0,
  "shark450_buy_avg_price" : 25000,
  "sheep450_buy_avg_price" : 25073,
  "shark450_sell_avg_price" : 26000,
  "sheep450_sell_avg_price" : 25085,
  "shark900_buy_avg_price" : 26000,
  "sheep900_buy_avg_price" : 25061,
  "shark900_sell_avg_price" : 26000,
  "sheep900_sell_avg_price" : 25085,
  "shark450_buy_ratio_5d_pc" : 0.0713,
  "sheep450_buy_ratio_5d_pc" : 0.3741,
  "shark450_sell_ratio_5d_pc" : 0.0,
  "sheep450_sell_ratio_5d_pc" : 0.1905,
  "shark900_buy_ratio_5d_pc" : 0.0,
  "sheep900_buy_ratio_5d_pc" : 0.4455,
  "shark900_sell_ratio_5d_pc" : 0.0,
  "sheep900_sell_ratio_5d_pc" : 0.1905,
  "shark450_urgency_spread" : -3.989,
  "shark900_urgency_spread" : 0.0
}
```

## Features

### Whale Footprint Features

Tạm thời sẽ dùng 2 threshold 450 và 900 để so sánh value xem có phải là shark hay không

#### Implemented

- `sharkXXX_buy_value`,  `sharkXXX_sell_value` mua bán của sharkXXX trong **`*candle`*** 
  
    Để theo value như này có vẻ không ổn, có thể nomalize với giá trị giao dịch trung bình 5 ngày.
    `sharkXXX_buy_ratio_5d_pc`: so sánh tỷ lệ với trung bình 5 ngày

- `sharkXXX_buy_avg_price`, `sharkXXX_sell_avg_price` Mức giá giao dịch BÌNH QUÂN của các lệnh MUA/BÁN lớn.
  
    Cho một cái nhìn để biết được hành vi của shark là đang mua/bán giá thấp hay là mua giá cao.

    *Insight:* Nếu `sharkXXX_buy_avg_price > sharkXXX_sell_avg_price`, có thể phe mua lớn chấp nhận mua giá cao hơn phe bán lớn.

    **NOTE**: Hiện tại đang implementing là giá mua trung bình accumulate từ đầu ngày tới thời điểm của nến đang xem.

- `percent_sharkXXX_buy_sell`, `percent_sheepXXX_buy_sell`, `percent_buy_sharkXXX_sheep`, `percent_sell_sharkXXX_sheep`

    Các chỉ số này được tính dưới dạng phần trăm (%) và có 2 phiên bản:
    1. Tính trong candle hiện tại (không có prefix).
    2. Tính lũy kế từ đầu phiên (prefix `accum_`).

    **Công thức cụ thể:**
  - `percent_sharkXXX_buy_sell` = `Shark Buy / (Shark Buy + Shark Sell)`: Tỷ trọng Mua trong tổng giao dịch của Shark.
  - `percent_sheepXXX_buy_sell` = `Sheep Buy / (Sheep Buy + Sheep Sell)`: Tỷ trọng Mua trong tổng giao dịch của Sheep.
  - `percent_buy_sharkXXX_sheep` = `Shark Buy / (Shark Buy + Sheep Buy)`: Tỷ trọng Shark đóng góp trong tổng lực Mua.
  - `percent_sell_sharkXXX_sheep` = `Shark Sell / (Shark Sell + Sheep Sell)`: Tỷ trọng Shark đóng góp trong tổng lực Bán.
  
- **`shark_urgency_spread = (avg_price_shark_buys - avg_price_shark_sells) / vwap`** 

  - Chuẩn hóa bằng VWAP giúp so sánh được giữa các cổ phiếu. Spread dương lớn cho thấy phe mua lớn đang rất "hung hăng", sẵn sàng mua đuổi giá cao.

#### TODO

- ~~`avg_sharkXXX_buy_value_5d`, `avg_sharkXXX_sell_value_5d`~~

  ~~It has not been implemented yet. Because I haven't found it's important right now~~

- so sánh với volume trung bình trượt 5 ngày tại timepoint đó, (tức là cộng dồn từ đầu phiên cho tới timepoint đó và chia trung bình trượt của 5 ngày) -> từ đó xem được khối lượng giao dịch đột biến hay không
