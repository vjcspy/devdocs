# Fix WhaleFootprintFeatureCalculator

## User Requirements

1. Read overview file in `devdocs/projects/metan/stock/OVERVIEW.md` to understand the project.
2. Read carefully `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`, `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` and all relevant files (if needed) to understand the current implementation 
3. You can see in `compute_per_candle_baselines` we need to have 5 previous days to calculate `pc_value_5d` value. But now in `tick_candles_by_date` we just return 1 day candles. Read `prices` method to know how we can retrieve 5 previous days.
4. I want to validate we have same total days in prices and ticks (compare every row in prices and ticks to ensure they are same)
5. Chỗ này tôi giải thích bằng Vietnamese cho rõ. Chúng ta chỉ cần tính toán feature từ start_date tới end_date, nhưng để tính toán các giá trị base line bắt buộc phải lấy thêm 5 ngày trước đó. Tuy nhiên, một số hàm validate hiện tại ví dụ như ở trong `compute_per_candle_baselines` đang check như sau:
   ```
   bad = daily_df[(daily_df["candle_count"].notna()) & (daily_df["pc_value_today"] <= 0)].index.tolist()
   ```

   Như vậy là đang bị sai do những ngày chúng ta lấy thêm sẽ không cần validate, nó chỉ nhằm mục đích tính toán các base values.

   Chúng ta cần sửa chỉ validate các ngày trong khoảng thời gian từ start_date đến end_date mà thôi

   Cũng do point này nên khi build rows ở method `iter_candle_dates` cũng chỉ lấy các ngày từ start_date đến end_date để tính toán cho các feature row. Nhưng cần kiểm tra nếu cần tính toán base value vẫn phải tính đủ. => chỗ này cần kiểm tra kỹ nhé 