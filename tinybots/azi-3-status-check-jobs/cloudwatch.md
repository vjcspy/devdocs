# Cloudwatch azi-3-status-check-jobs

## Query

1. Check xem đã init được sessiong hay chưa

   ```
   fields @timestamp, session, rule
   | filter message = "Monitoring session initialized"
   | sort @timestamp asc
   | limit 10000
   ```

Từ đây quan trọng nhất lấy được **session.robotId**, **session.subscriptionId** và **session.id**

2. Lấy các log xử lý event từ queue

   ```
   fields @timestamp, message, subscriptionId, _callRef
   | filter message like "Processing activity event"
   | filter eventName = "TOILET_ACTIVITY"
   | filter robotId = 61343
   | sort @timestamp asc
   | limit 10000
   ```

   

   <u>Chỗ này có thể lọc thêm subscriptionId nữa thì lấy chính xác được những event sẽ được xử lý</u> (luồng reset)

   ```
   fields @timestamp, message, subscriptionId, _callRef
   | filter message like "Processing activity event"
   | filter eventName = "TOILET_ACTIVITY"
   | filter robotId = 61343
   | filter subscriptionId = 4344
   | sort @timestamp asc
   | limit 10000
   ```

3. Hiện tại có issue là là job được init duy nhất 1 lần nên callRef đều giống nhau, hơi chuối. Query này để lấy các job xử lý

```
fields @timestamp, message, _callRef, subscriptionId
| filter _callRef = "azi-3-status-check-jobs_2779acb4-1d4b-4bd8-bb44-d7ad78ff7d7e"
| sort @timestamp asc
| limit 10000
```

4. <u>Check xem những thời điểm window bị expired</u> (luồng expired)
```
fields @timestamp, message, sessionId, robotId
| filter message like "Window advanced after action execution"
| sort @timestamp asc
| limit 10000
```

Xem toàn bộ cron job log

```
fields @timestamp, message, sessionId, robotId, _serviceRef
#| filter message = "Found expired windows"
| filter _serviceRef like "azi-3-status-check-jobs_e169cb38-787d-4544-885b-731d6f5e7e03_WindowExpirationChecker"
| sort @timestamp asc
| limit 10000
```

