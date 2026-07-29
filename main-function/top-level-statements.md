# Top-level statements

> **Đã gộp** vào [main-function.md](../main-function.md) §3–4, §7, §9.

Từ C# 10, executable code có thể nằm ở root file (không cần `Main` tường minh). Chi tiết quy tắc entry, `args` / `await` / `return`, tổng hợp `Program` / `<Main>$`, tương tác global usings, và pitfalls (nhiều file TLS, trộn `Main` + TLS) nằm ở trang chính.

```csharp
Console.WriteLine("Hello World!");
```

Xem thêm: [File-based apps (.NET 10)](../main-function.md#8-file-based-apps-net-10--c-14) · [C# 14 / .NET 10](../csharp14-dotnet10.md)
