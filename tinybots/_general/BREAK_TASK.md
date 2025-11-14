Bây giờ tôi cần bạn break các ticket nhỏ hơn để làm feature này. Yêu cầu như sau:

1. Bạn là expert BA, khi break task thì cần phải sử dụng English.

2. task có description và file structure sẽ change. Với template

   ```tex
   Description here
   
   ### File structure impact
   src/
   ├── ScreenshotCapture/          # IMPLEMENTED - CLI và Service mode
   │   ├── Program.cs              # Entry point với System.CommandLine
   ├── WinServicesRAG.Core/        # IMPLEMENTED - Screenshot providers
   ├── WorkerService/              # TODO - Business logic service
   ├── WatchdogService/            # TODO - Process monitoring
   ```

   

3. Task implement logic chính của feature thì sẽ phải bao gồm cả test (unit test/integration test)