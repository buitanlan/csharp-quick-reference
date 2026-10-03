# Tài liệu tham khảo ngôn ngữ lập trình C# / .NET

Bộ tài liệu tham chiếu **in-depth / advanced** cho ngôn ngữ C# trên nền **.NET 10 LTS** (C# **14**). Đây là **repo tra cứu theo chủ đề** (một file = một vùng ngôn ngữ/BCL: kiểu, LINQ, async, project/SDK, …) — **không** phải changelog “what's new”, không phải tutorial tuần tự, không thay [docs Microsoft](https://learn.microsoft.com/dotnet/csharp/).

**Cách đọc:** mở đúng topic file ở mục lục dưới (ví dụ `Main` → [main-function.md](main-function.md), NuGet/AOT → [projects-packages.md](projects-packages.md)). Trong file: mục lục → quy tắc/semantics → ví dụ → **pitfalls** / version gate. Bảng C# 14 vs 15 ở [projects-packages.md §14](projects-packages.md#14-best-practices--checklist) chỉ là *cổng* sang topic, không đủ để code feature mới.

Không phải giáo trình nhập môn: giả định đã biết class/method/`if`. Nếu chưa biết C#/.NET, bắt đầu bằng khóa học bên dưới, rồi quay lại đây khi cần semantics, IL/entry, `IQueryable`, AOT, v.v.

> **Baseline:** .NET **10** / C# **14** (GA 11/2025, hỗ trợ đến **14/11/2028**) — mặc định `net10.0`, `LangVersion` 14.  
> Mục ghi **C# 15 / .NET 11** theo **RC1** (08/09/2026, giấy phép **go-live**; GA dự kiến ~11/2026). Trên `net11.0`, C# 15 là ngôn ngữ **mặc định** — union, `closed`, extension indexer, `with(...)`, labeled `break`/`continue`, static non-virtual trên interface **không** cần `<LangVersion>preview</LangVersion>`. Baseline dài hạn của repo vẫn là **net10.0**. **Unsafe Evolution** (memory safety mới) vẫn là preview riêng, không đi theo C# 15 mặc định.

**Rà soát ngày 03/10/2026.** LTS được hỗ trợ 3 năm, STS 2 năm; .NET 8 và 9 hết hỗ trợ **10/11/2026**. .NET 11 RC1 vẫn là prerelease dù có go-live. [Chính sách hỗ trợ .NET](https://dotnet.microsoft.com/en-us/platform/support/policy). Baseline dài hạn: net10.0/C# 14. [Checklist nâng cấp](projects-packages.md), [entry/Main](main-function.md).

## Quy ước ví dụ và kiểm tra tài liệu

Ví dụ thường là **đoạn minh họa theo ngữ cảnh**, không phải mỗi code block là một chương trình độc lập. Các tên như db, logger, source, ProcessAsync cần implementation của ứng dụng; thêm namespace/package và đặt declaration trong type/method phù hợp. Với top-level statements, đặt statement trước type declaration hoặc tách file. Đoạn ghi “SAI”, “lỗi” hoặc diagnostics minh họa lỗi có chủ đích.

Chạy `node scripts/check-docs.mjs` từ repo để kiểm tra file/anchor nội bộ, code fence và ký tự lỗi encoding; script không biên dịch các snippet hoặc kiểm tra URL ngoài. Các quy tắc compiler/runtime được rà soát với SDK .NET 10; phần C# 15/.NET 11 đối chiếu release notes và đặc tả, cần SDK 11 tương ứng để chạy. [Release notes C# RC1](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/rc1/csharp.md).

---

Tham khảo: [Khóa học .NET nền tảng](https://github.com/daohainam/lets-learn-dotnet) · [C# 14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14) · [.NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) · [C# 15](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [.NET 11 RC1](https://devblogs.microsoft.com/dotnet/dotnet-11-rc-1/)

---

## Nội dung  

- [Hàm Main & entry](main-function.md)
- [Project, SDK & NuGet](projects-packages.md)
- [Hệ thống kiểu dữ liệu](typesystem.md)
- [Attributes & Reflection](attributes-reflection.md)
- [System.Text.Json](system-text-json.md)
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
