# Feature Engineering

## 1. Đặc trưng về "Dấu chân Người khổng lồ" (Whale Footprints)

- `large_buy_volume`/`large_sell_volume` Tổng khối lượng từ các lệnh MUA/Bán lớn trong ngày.

- `large_trades_ratio` Tỷ lệ `(large_buy_volume + large_sell_volume) / total_volume`. Cho biết mức độ chi phối của "cá mập" trong ngày.

- `num_large_buys` / `num_large_sells`  Số lượng lệnh mua/bán lớn. 1 lệnh 100k cổ phiếu khác với 10 lệnh 10k cổ phiếu.

- `avg_price_large_buys`/`avg_price_large_sells` Mức giá giao dịch BÌNH QUÂN của các lệnh MUA/BÁN lớn.

  *Insight:* Nếu `avg_price_large_buys > avg_price_large_sells`, có thể phe mua lớn chấp nhận mua giá cao hơn phe bán lớn.

- `diff_shark / diff_sheep / accum_diff_shark_7 / accum_diff_shark_15 / accum_diff_sheep_7 / accum_diff_sheep_15` tính dữ liệu trong ngày và 7,15 ngày cộng dồn

- **`shark_urgency_spread = (avg_price_large_buys - avg_price_large_sells) / vwap`**. Chuẩn hóa bằng VWAP giúp so sánh được giữa các cổ phiếu. Spread dương lớn cho thấy phe mua lớn đang rất "hung hăng", sẵn sàng mua đuổi giá cao.

## 2. Đặc trưng về "Sức mạnh Hấp thụ & Áp lực" (Absorption & Pressure)

### Hấp thụ Lực bán (Sell-side Absorption)

> Đây chính là nhóm để trả lời câu hỏi : "bán nhiều nhưng giá không giảm? Mua nhiều sao giá không tăng?".

**Ý tưởng:** Đo lường những lần có lệnh BÁN lớn hoặc một chuỗi lệnh bán liên tục xuất hiện, nhưng giá không giảm hoặc thậm chí tăng ngay sau đó.

**Cách tính (ví dụ):**

1. Chia phiên giao dịch thành các cửa sổ thời gian nhỏ (ví dụ: 5 phút).
2. Trong mỗi cửa sổ, nếu `sell_volume > buy_volume` (áp lực bán chiếm ưu thế).
3. Hãy xem giá ở cuối cửa sổ này so với giá ở cuối cửa sổ tiếp theo.
4. Nếu giá không giảm (hoặc tăng), đó là một "sự kiện hấp thụ".
5. **Feature `absorption_score`:** Đếm số lần "sự kiện hấp thụ" này xảy ra trong ngày, hoặc tổng khối lượng bị bán trong những lần đó mà giá vẫn đứng vững.

### Đặc trưng Áp lực Bán Gây sụp đổ (Effective Sell Pressure):

**Ý tưởng:** Ngược lại với ở trên. Tìm những lúc chỉ cần một lượng bán nhỏ cũng đủ làm giá giảm mạnh.

**Cách tính (ví dụ):**

1. Tìm những cửa sổ 5 phút có `sell_volume > buy_volume`.
2. Tính "hiệu quả" của lực bán: `price_change / (sell_volume - buy_volume)`.
3. Nếu chỉ số này rất âm (giảm mạnh dù delta volume không lớn), nó cho thấy bên mua rất yếu.
4. **Feature `brittle_market_score`:** Trung bình hoặc tổng của các chỉ số "hiệu quả" âm nhất trong ngày.

## 3. Đặc trưng về "Động lực Phiên" (Session Dynamics)

### **Sức mạnh Tương quan Sáng/Chiều:**

- `morning_buy_sell_ratio`: Tỷ lệ mua/bán chỉ tính trong phiên sáng.
- `afternoon_buy_sell_ratio`: Tỷ lệ mua/bán chỉ tính trong phiên chiều.
- `afternoon_reversal_flag`: Nếu `morning_ratio > 1` (sáng mua mạnh) nhưng `afternoon_ratio < 1` (chiều bán mạnh), có thể là tín hiệu phân phối. Gắn cờ = 1.

### **Sức mạnh lúc Đóng cửa (Closing Power):**

**Ý tưởng:** Hành vi trong 30 phút cuối phiên (đặc biệt là phiên ATC) cực kỳ quan trọng, thể hiện ý chí của nhà tạo lập.

- Cách tính:
  1. Tính `buy_sell_ratio` chỉ trong 30 phút cuối cùng của phiên.
  2. **Feature `closing_power_ratio`:** Giá trị ratio này. Một ratio > 1 cho thấy phe mua đang nỗ lực đẩy giá lên vào cuối ngày.
  3. **Feature `close_vs_vwap`:** Tỷ lệ giữa giá đóng cửa (`Close`) và giá trung bình theo khối lượng (`VWAP - Volume Weighted Average Price`) của cả ngày. Nếu `Close > VWAP`, phe mua đã chiến thắng vào cuối ngày.

## 4. Đặc trưng về Biến động trong Phiên (Intraday Volatility)

- ```
  intraday_volatility
  ```

  : Độ lệch chuẩn của giá trong suốt cả ngày. Một ngày đi ngang trong biên độ hẹp có 

  ```
  volatility
  ```

   thấp, một ngày giật mạnh lên xuống có 

  ```
  volatility
  ```

   cao.

  - **Định nghĩa của bạn:** "Độ lệch chuẩn của giá trong suốt cả ngày." - Chính xác. Đây là cách đo lường phổ biến nhất.

  - Góc nhìn sâu hơn:

     Feature này không chỉ cho biết ngày đó "đi ngang" hay "giật mạnh". Một giá trị volatility cao có thể mang nhiều ý nghĩa:

    - **Sự bất đồng quan điểm:** Phe mua và phe bán đang giao tranh quyết liệt.
    - **Phản ứng với tin tức:** Có một thông tin bất ngờ nào đó được tung ra trong phiên.
    - **Hành động của nhà tạo lập (Market Maker):** Họ đang "vẽ chart", tạo thanh khoản hoặc rũ bỏ nhà đầu tư nhỏ lẻ.

  - Gợi ý tối ưu:

    - **Input để tính toán:** Thay vì dùng giá (price), bạn nên cân nhắc tính độ lệch chuẩn trên **tỷ suất sinh lợi theo logarit (log returns)** ở các khung thời gian nhỏ (1 phút, 5 phút). Log returns phản ánh tốt hơn sự thay đổi tương đối của giá và có các đặc tính thống kê tốt hơn.
    - **Chuẩn hóa (Normalization):** Để so sánh volatility giữa các cổ phiếu có thị giá khác nhau, hoặc cùng một cổ phiếu ở các thời điểm lịch sử khác nhau, bạn nên chuẩn hóa nó. Một cách phổ biến là dùng **ATR (Average True Range)**. Bạn có thể tính một phiên bản intraday của ATR, ví dụ: `intraday_volatility / ATR(14)`. Feature này sẽ cho biết mức độ biến động hôm nay so với mức độ biến động "bình thường" của cổ phiếu đó là cao hay thấp.

- ```
  volatility_concentration
  ```

  : Biến động có tập trung vào một thời điểm nào không (ví dụ: chỉ biến động mạnh 15 phút đầu rồi đi ngang)? Bạn có thể tính 

  ```
  volatility
  ```

   của buổi sáng và buổi chiều để so sánh.

  - **Định nghĩa của bạn:** "Biến động có tập trung vào một thời điểm nào không?" - Ý tưởng xuất sắc. Nó giúp phân biệt một ngày có xu hướng rõ ràng với một ngày "sáng nắng chiều mưa".

  - **Góp ý về tên gọi:** Tên `volatility_concentration` khá ổn về mặt kỹ thuật. Tuy nhiên, để dễ hình dung hơn trong bối cảnh Việt Nam, bạn có thể gọi nó là **"Phân Bố Biến Động Trong Phiên"** hoặc cụ thể hơn là **"Độ Lệch Biến Động Sáng-Chiều"**.

  - Gợi ý tối ưu & Mở rộng:

    - Không chỉ Sáng/Chiều:

       Thay vì chỉ chia 2, hãy chia phiên làm 4 phần: Đầu phiên sáng (9:00-9:45), cuối phiên sáng (9:45-11:30), đầu phiên chiều (13:00-13:45), và cuối phiên chiều + ATC (13:45-14:45). Biến động ở mỗi giai đoạn này mang một ý nghĩa riêng:

      - **Đầu phiên sáng:** Phản ứng với tin tức qua đêm, tâm lý từ phiên trước.
      - **Cuối phiên chiều + ATC:** Thường là lúc các quỹ, tổ chức lớn cơ cấu danh mục. Biến động mạnh ở đây có ý nghĩa quan trọng hơn nhiều.

    - **Đo lường sự tập trung:** Thay vì chỉ so sánh, bạn có thể tính một chỉ số cụ thể. Ví dụ: Tính volatility của từng 15 phút, sau đó tính **chỉ số Gini** hoặc **độ lệch chuẩn của các giá trị volatility** này. Một chỉ số cao cho thấy biến động của cả ngày chỉ tập trung vào một vài khoảnh khắc ngắn ngủi.

    - **Feature mới - "Đà biến động" (Volatility Momentum):** Tính xem volatility nửa sau của phiên cao hơn hay thấp hơn nửa đầu. Volatilitychie^ˋu/Volatilitysaˊng. Một giá trị > 1 cho thấy sự sôi động đang tăng dần về cuối phiên, thường là tín hiệu cho một ngày có xu hướng mạnh mẽ.

## 5. Đặc trưng về Đột biến & So sánh Tương quan (Anomaly & Comparative Analysis)

- `shark_buy_volume_to_average_5_days` / `shark_buy_value_to_average_5_days` / … so sánh tương quan ngày hiện tại với trung bình 5 ngày trước đó để xem đột biến

- `shark_net_flow_value` / `shark_net_flow_volume`

  **Dòng tiền ròng:** Đừng chỉ nhìn vào Mua hoặc Bán. Hãy tạo feature quan trọng nhất:

  ```
  shark_net_flow_volume = shark_buy_volume - shark_sell_volume
  ```

  So sánh các giá trị ròng này với trung bình 5 ngày sẽ hiệu quả hơn nhiều.

- `shark_volume_participation` Tổng lệnh của shark (cả mua và bán) / tổng volume

  Cho thấy mức độ chi phối

- `price_change_per_shark_flow` (close_price - open_price) / shark_net_flow_value

  Feature này đo lường: Với một lượng tiền ròng mà "cá mập" bỏ ra, họ đẩy/dìm giá được bao nhiêu. Nếu `shark_net_flow_value` rất lớn nhưng giá không đổi, có thể họ đang bị một lực cản đối ứng mạnh mẽ.

## 6. Đặc trưng Kỹ thuật Cổ điển

- Chỉ báo Xu hướng (Trend Indicators):
  - **Moving Averages (SMA, EMA):** Làm mượt giá để xác định xu hướng chính. Ví dụ: `EMA(12)`, `EMA(26)`.
- Chỉ báo Động lượng (Momentum Indicators):
  - **RSI (Relative Strength Index):** Đo lường tốc độ và sự thay đổi của biến động giá, thường dùng để xác định các vùng quá mua/quá bán.
  - **MACD (Moving Average Convergence Divergence):** Thể hiện mối quan hệ giữa hai đường EMA, cung cấp tín hiệu về sự thay đổi động lượng.
- Chỉ báo Biến động (Volatility Indicators):
  - **Bollinger Bands:** Tạo ra một dải bao quanh đường SMA, giúp xác định mức độ biến động và các mức giá tương đối cao/thấp.
  - **ATR (Average True Range):** Một thước đo về mức độ biến động của thị trường, hữu ích trong việc đặt các ngưỡng dừng lỗ.
- RSI & Dấu chân Người khổng lồ:
  - `shark_buy_volume` tăng đột biến khi RSI < 30 (quá bán) là một tín hiệu bắt đáy tiềm năng mạnh hơn nhiều so với khi RSI = 50.
  - `large_sell_volume` tăng vọt khi RSI > 70 (quá mua) có thể là tín hiệu xác nhận một "blow-off top" (đỉnh cao trào).
- Bollinger Bands & Sức mạnh Hấp thụ:
  - Khi giá chạm vào dải Bollinger dưới và đồng thời `absorption_score` tăng cao, đó là một tín hiệu xác nhận hỗ trợ rất mạnh.
- MACD & Động lực Phiên:
  - Khi MACD vừa có giao cắt vàng (bullish crossover) và ngày hôm đó có `closing_power_ratio` > 1.5, xác suất tăng tiếp vào ngày hôm sau sẽ cao hơn.

