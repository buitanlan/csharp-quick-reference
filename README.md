# Tài liệu tham khảo ngôn ngữ lập trình C# / .NET

Bộ tài liệu tham chiếu **in-depth / advanced** cho ngôn ngữ C# trên nền **.NET 10 LTS** (C# **14**). Đây là **repo tra cứu theo chủ đề** (một file = một vùng ngôn ngữ/BCL: kiểu, LINQ, async, project/SDK, …) — **không** phải changelog “what's new”, không phải tutorial tuần tự, không thay [docs Microsoft](https://learn.microsoft.com/dotnet/csharp/).

**Cách đọc:** mở đúng topic file ở mục lục dưới (ví dụ `Main` → [main-function.md](main-function.md), NuGet/AOT → [projects-packages.md](projects-packages.md)). Trong file: mục lục → quy tắc/semantics → ví dụ → **pitfalls** / version gate. Bảng C# 14 vs 15 ở [projects-packages.md §14](projects-packages.md#14-best-practices--checklist) chỉ là *cổng* sang topic, không đủ để code feature mới.

Không phải giáo trình nhập môn: giả định đã biết class/method/`if`. Nếu chưa biết C#/.NET, bắt đầu bằng khóa học bên dưới, rồi quay lại đây khi cần semantics, IL/entry, `IQueryable`, AOT, v.v.

> **Baseline:** .NET **10** / C# **14** (GA 11/2025, hỗ trợ đến **14/11/2028**) — mặc định `net10.0`, `LangVersion` 14.  
> Mục ghi **C# 15 / .NET 11** theo **RC1** (08/09/2026, giấy phép **go-live**; GA dự kiến ~11/2026). Trên `net11.0`, C# 15 là ngôn ngữ **mặc định** — union, `closed`, extension indexer, `with(...)`, labeled `break`/`continue`, static non-virtual trên interface **không** cần `<LangVersion>preview</LangVersion>`. Baseline dài hạn của repo vẫn là **net10.0**. **Unsafe Evolution** (memory safety mới) vẫn là preview riêng, không đi theo C# 15 mặc định.

Chu kỳ hỗ trợ: LTS ~ 3 năm; xen kẽ STS. .NET 8/9 EOS ~ cùng cửa sổ GA .NET 11 (**~10/11/2026**). Production dài hạn: **net10.0** → C# 14 mặc định. Checklist nâng cấp: [projects-packages.md](projects-packages.md). Entry/`Main`/TLS/`args`: [main-function.md](main-function.md).

---

Tham khảo: [Khóa học .NET nền tảng](https://github.com/daohainam/lets-learn-dotnet) · [C# 14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14) · [.NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) · [C# 15](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [.NET 11 RC1](https://devblogs.microsoft.com/dotnet/dotnet-11-rc-1/)

---

## Nội dung  

- [Hàm Main & entry](main-function.md)
- [Project, SDK & NuGet](projects-packages.md)
- [Hệ thống kiểu dữ liệu](typesystem.md)
- [Memory, Span & unsafe](memory-spans.md)
- [Chỉ thị tiền biên dịch](preprocessor-directives.md)
- [Literal](literals.md)
- [Toán tử](operators.md)
- [Từ khóa](keywords.md)
- [Phát biểu](statements.md)
- [Phương thức](methods.md)
- [Delegate và Lambda](delegates-lambdas.md)
- [Exception](exceptions.md)
- [Lập trình hướng đối tượng trong C#](oop.md)
- [Tập hợp & Generics](collections-generics.md)
- [LINQ](linq.md)
- [Lập trình hàm](functional.md)
- [Thread](threading.md)
- [Lập trình bất đồng bộ](async.md)
