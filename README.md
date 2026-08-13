# Tài liệu tham khảo ngôn ngữ lập trình C# / .NET

Bộ tài liệu tham chiếu **in-depth / advanced** cho ngôn ngữ C# trên nền **.NET 10 LTS** (C# **14**). Không phải giáo trình nhập môn: các khái niệm được trình bày dạng tham khảo nhanh kèm chi tiết nâng cao (semantics, version gates, pitfalls). Nếu chưa biết C#/.NET, bắt đầu bằng khóa học bên dưới, rồi dùng bộ này khi cần tra cứu sâu hơn.

> **Baseline:** .NET **10** / C# **14** (GA 11/2025, hỗ trợ đến **14/11/2028**).  
> Mục ghi **C# 15 / .NET 11 Preview** (Preview 7 · 08/2026; GA dự kiến ~11/2026) là *preview* — chưa dùng cho production.

Chu kỳ hỗ trợ: LTS ~ 3 năm; xen kẽ STS. .NET 8/9 EOS ~ cùng cửa sổ GA .NET 11 (**~10/11/2026**). Production dài hạn: **net10.0** → C# 14 mặc định. Checklist nâng cấp: [projects-packages.md](projects-packages.md).

---

Tham khảo: [Khóa học .NET nền tảng](https://github.com/daohainam/lets-learn-dotnet) · [C# 14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14) · [.NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) · [C# 15 preview](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15)

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
- [Thread](threading.md)
- [Lập trình bất đồng bộ](async.md)
