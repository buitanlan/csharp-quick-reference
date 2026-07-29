# Tham số của hàm Main

> **Đã gộp** vào [main-function.md](../main-function.md) §5.

`Main` / top-level statements nhận `string[] args` — **không bao giờ `null`**. Khi không khai báo tham số, vẫn lấy dòng lệnh qua `Environment.GetCommandLineArgs()` / `Environment.CommandLine` (lưu ý: `GetCommandLineArgs()[0]` là đường dẫn executable, khác `args[0]`).

Chi tiết so sánh API, `dotnet run -- …`, và gợi ý parse (`System.CommandLine`, …): xem trang chính §5.
