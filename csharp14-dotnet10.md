# C# 14 / .NET 10 — điểm nổi bật

*(LTS — GA tháng 11/2025; hỗ trợ đến **14/11/2028**)*

**.NET 10** là bản **Long-Term Support (LTS)** tiếp theo sau .NET 8; ngôn ngữ đi kèm là **C# 14**. Trang này là **hub** tóm tắt điểm nổi bật cho developer ngôn ngữ / BCL — không phải changelog toàn bộ runtime/ops — và liên kết sang các chương chuyên đề trong bộ tham chiếu.

> **Baseline repo:** .NET **10** / C# **14**. Mục **C# 15 / .NET 11** bên dưới là *preview* — chưa GA.

Trang chính thức:

- [What's new in C# 14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14)
- [What's new in .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview)
- [Support policy](https://dotnet.microsoft.com/platform/support/policy/dotnet-core)

> Cross-link (hub → topical): [main-function.md](main-function.md) (TLS, file-based apps) · [methods.md](methods.md) / [oop.md](oop.md) (extension members, `field`, partial) · [operators.md](operators.md) (null-conditional assignment, compound assignment) · [delegates-lambdas.md](delegates-lambdas.md) (lambda modifiers) · [memory-spans.md](memory-spans.md) (Span conversions) · [preprocessor-directives.md](preprocessor-directives.md) (`#:`)

---

## Mục lục

- [C# 14 / .NET 10 — điểm nổi bật](#c-14--net-10--điểm-nổi-bật)
  - [Mục lục](#mục-lục)
  - [1. Bối cảnh LTS](#1-bối-cảnh-lts)
  - [2. C# 14 final — ngôn ngữ](#2-c-14-final--ngôn-ngữ)
    - [2.1 Extension members](#21-extension-members)
    - [2.2 `field` keyword](#22-field-keyword)
    - [2.3 Null-conditional assignment](#23-null-conditional-assignment)
    - [2.4 `nameof` với unbound generics](#24-nameof-với-unbound-generics)
    - [2.5 Implicit Span conversions](#25-implicit-span-conversions)
    - [2.6 Lambda parameter modifiers](#26-lambda-parameter-modifiers)
    - [2.7 Partial constructors \& events](#27-partial-constructors--events)
    - [2.8 User-defined compound assignment / `++` `--`](#28-user-defined-compound-assignment----)
    - [2.9 File-based apps \& `#:` directives](#29-file-based-apps---directives)
  - [3. Runtime / BCL (.NET 10) — tóm tắt](#3-runtime--bcl-net-10--tóm-tắt)
  - [4. C# 15 / .NET 11 — PREVIEW](#4-c-15--net-11--preview)
  - [5. Nhắc lại nền tảng đã final trước 14](#5-nhắc-lại-nền-tảng-đã-final-trước-14)
  - [6. Tài nguyên \& checklist nâng cấp](#6-tài-nguyên--checklist-nâng-cấp)

---

## 1. Bối cảnh LTS

| Mốc | Ý nghĩa |
|-----|---------|
| .NET 8 LTS | Baseline production trước đó (hỗ trợ đến ~11/2026) |
| .NET 9 STS | Cầu nối hàng năm; C# 13 |
| **.NET 10 LTS** | GA **11/11/2025** · hỗ trợ đến **14/11/2028** · **C# 14** |
| .NET 11 + C# 15 | **Preview** (chu kỳ tiếp theo; GA dự kiến ~11/2026) |

Chu kỳ: LTS ~ 3 năm hỗ trợ; xen kẽ STS. Nâng lên 10 khi cần cửa sổ hỗ trợ dài và ngôn ngữ 14.

Target framework mặc định cho doc này:

```xml
<TargetFramework>net10.0</TargetFramework>
<!-- LangVersion mặc định theo TFM = C# 14; không cần ghi trừ khi hạ / preview -->
```

---

## 2. C# 14 final — ngôn ngữ

### 2.1 Extension members

Headline C# 14: khối `extension(...)` trong `static class` — mở rộng **properties**, **methods**, **operators**, kể cả member **static** trên type (không chỉ instance như extension method cũ).

```csharp
public static class EnumerableExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();

        public IEnumerable<T> WhereNotNull() where T : class
            => source.Where(static x => x is not null)!;
    }

    extension<T>(IEnumerable<T>)
    {
        public static IEnumerable<T> Identity => Enumerable.Empty<T>();

        public static IEnumerable<T> operator +(IEnumerable<T> left, IEnumerable<T> right)
            => left.Concat(right);
    }
}

// Gọi như member thật:
bool empty = Array.Empty<int>().IsEmpty;
var combined = seqA + seqB;
```

- Extension method kiểu `this T` **vẫn hợp lệ** và tương thích IL với dạng mới.
- Không thêm **field** qua extension (do đó không có auto-property cần backing field trên extension).
- Chi tiết: [methods.md](methods.md) · [oop.md](oop.md) · [Learn: extension members](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)

### 2.2 `field` keyword

Token ngữ cảnh `field` trỏ tới backing field compiler-synthesized — viết logic accessor mà không khai báo `_backing` tay.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

- Có thể chỉ viết body một phía (`get;` + `set => …` hoặc ngược lại).
- Nếu type đã có member tên `field`: dùng `@field` / `this.field` hoặc đổi tên.
- Preview từ .NET 9 → **GA trong C# 14**.

### 2.3 Null-conditional assignment

`?.` / `?[]` dùng được bên **trái** phép gán / compound assignment. Vế phải **chỉ** evaluate khi vế trái không null.

```csharp
customer?.Order = GetCurrentOrder();   // không gọi GetCurrentOrder nếu customer == null
customer?.Score += 10;
```

- `++` / `--` **không** kết hợp với `?.` theo quy tắc hiện tại.
- Learn: [null-conditional operators](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)

### 2.4 `nameof` với unbound generics

```csharp
nameof(List<>);      // "List"
nameof(Dictionary<,>); // "Dictionary"
```

Trước đây chỉ `nameof(List<int>)` (closed) mới hợp lệ cho mục đích tên generic mở.

### 2.5 Implicit Span conversions

C# 14 coi `Span<T>` / `ReadOnlySpan<T>` là first-class hơn: thêm implicit conversion với `T[]` và giữa các span, cải thiện extension receiver / type inference.

```csharp
void Print(ReadOnlySpan<char> s) => Console.WriteLine(s.ToString());

char[] buffer = ['a', 'b', 'c'];
Print(buffer);                 // array → ROS tự nhiên hơn
Span<char> slice = buffer;
ReadOnlySpan<char> view = slice;
```

Chi tiết: [memory-spans.md](memory-spans.md) · feature *First class span types*.

### 2.6 Lambda parameter modifiers

Có thể gắn `ref` / `in` / `out` / `scoped` / `ref readonly` trên lambda **không** cần ghi đủ kiểu tham số (suy từ delegate):

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

Trước đây modifier buộc khai báo đủ kiểu. `params` vẫn yêu cầu typed parameter list.

### 2.7 Partial constructors & events

`partial` mở rộng sang **instance constructors** và **events** (đúng một defining + một implementing declaration).

```csharp
public partial class Widget
{
    public partial Widget(string name);           // defining
    public partial event EventHandler? Changed;   // defining (field-like)
}

public partial class Widget
{
    public partial Widget(string name) : base()   // implementing — initializer chỉ ở đây
    {
        Name = name;
    }

    public partial event EventHandler? Changed    // implementing — bắt buộc add/remove
    {
        add { /* ... */ }
        remove { /* ... */ }
    }

    public string Name { get; }
}
```

- Chỉ implementing ctor được có `: this(...)` / `: base(...)`.
- Primary constructor: chỉ một phần declaration được dùng cú pháp primary.

### 2.8 User-defined compound assignment / `++` `--`

C# 14 cho phép định nghĩa toán tử gán kép (`+=`, `-=`, …) và tăng/giảm kiểu user-defined theo feature *user-defined compound assignment* — tối ưu kiểu mutable / collection-like mà không luôn đi qua `op_Addition` + gán lại.

```csharp
public struct Counter
{
    public int Value;

    // Minh họa tinh thần — chữ ký chính xác theo spec compound assignment
    public void operator +=(int delta) => Value += delta;
}
```

- Dùng khi kiểu lớn / refrence semantics cần mutate tại chỗ.
- Spec: [user-defined compound assignment](https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-14.0/user-defined-compound-assignment) · [operators.md](operators.md)

### 2.9 File-based apps & `#:` directives

Không chỉ “SDK feature”: C# 14 thêm preprocessor / file-level directives cho app một file:

```csharp
#:package Spectre.Console@0.49.1
#:property PublishAot=false

Console.WriteLine("no csproj needed");
```

```bash
dotnet run app.cs
dotnet project convert app.cs
```

Chi tiết đầy đủ: [main-function.md §8](main-function.md#8-file-based-apps-net-10--c-14) · [File-based apps](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps)

---

## 3. Runtime / BCL (.NET 10) — tóm tắt

Không liệt kê hết release notes; vài điểm developer hay chạm:

| Chủ đề | Ghi chú thực dụng |
|--------|-------------------|
| **Native AOT** | Trưởng thành thêm; file-based apps **bật `PublishAot` mặc định** khi publish. Đo trim/AOT warnings sớm. |
| **Performance** | Cải tiến JIT, GC, LINQ/BCL micro-opts — nâng TFM thường đủ hưởng; hot path vẫn đo bằng BenchmarkDotNet. |
| **SDK / CLI** | `dotnet run file.cs`, cải tiến workload/template; xác nhận SDK ≥ 10.0.x trên CI. |
| **ASP.NET / libraries** | Xem [What's new in .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) theo stack (ASP.NET, EF, …). |

```xml
<!-- Ví dụ tắt AOT khi publish file-based / hoặc csproj -->
<PublishAot>false</PublishAot>
```

---

## 4. C# 15 / .NET 11 — PREVIEW

> **PREVIEW** — cần .NET **11** preview SDK + `<LangVersion>preview</LangVersion>`. Surface **có thể đổi** trước GA. **Không** dùng cho production baseline của repo này.

Bật:

```xml
<TargetFramework>net11.0</TargetFramework>
<LangVersion>preview</LangVersion>
```

### 4.1 Union types *(preview)*

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");
string name = pet switch
{
    Dog d => d.Name,
    Cat c => c.Name,
    Bird b => b.Name,
    // exhaustiveness: đủ case → không cần discard
};
```

- Runtime cung cấp `UnionAttribute` / `IUnion` từ các preview gần đây — xác nhận bản preview bạn dùng.
- Learn: [What's new in C# 15](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [Union types blog](https://devblogs.microsoft.com/dotnet/csharp-15-union-types/)

### 4.2 Các preview khác (tóm tắt)

| Feature | Ý tưởng ngắn |
|---------|----------------|
| **Closed hierarchies** | `closed` class — tập derived cố định trong assembly → switch exhaustive |
| **Extension indexers** | `this[int]` trong `extension` block (C# 14 có extension members; indexer mở rộng ở 15) |
| **Collection expression arguments** | `[with(capacity: n), ..items]` truyền arg vào ctor/factory |
| **Labeled `break` / `continue`** | `break outer;` / `continue outer;` trên vòng lồng |
| **Memory safety model** | Nới một phần thao tác pointer khỏi `unsafe`; dereference vẫn `unsafe` — đa giai đoạn |

```csharp
// Closed hierarchy (preview) — minh họa
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;
```

---

## 5. Nhắc lại nền tảng đã final trước 14

Không phải “mới 14”, nhưng là baseline khi nói C# hiện đại trên LTS 10:

| Chủ đề | Từ | Ghi chú |
|--------|-----|---------|
| Top-level statements / await | C# 10 / .NET 6 | [main-function.md](main-function.md) |
| Global / implicit usings, file-scoped namespace | C# 10 | |
| `required`, raw string, list patterns… | C# 11 | [literals.md](literals.md) · [statements.md](statements.md) |
| Primary constructors (class/struct) | C# 12 | [oop.md](oop.md) |
| Collection expressions `[…]` | C# 12 | |
| `params` collections, lock object, `\e`… | C# 13 / .NET 9 | |

---

## 6. Tài nguyên & checklist nâng cấp

```bash
dotnet --version          # >= 10.0
dotnet new console -n Demo
# csproj: net10.0 → C# 14 mặc định
```

Preview C# 15:

```bash
# cài .NET 11 preview SDK từ https://dotnet.microsoft.com/download/dotnet
```

Tài nguyên:

- [C# 14 what's new](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14)
- [C# 15 what's new (preview)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15)
- [.NET 10 overview](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview)
- [File-based apps](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps)
- [Introducing C# 14 (blog)](https://devblogs.microsoft.com/dotnet/introducing-csharp-14/)

```text
Checklist nâng cấp → .NET 10 / C# 14
[ ] TFM net10.0; CI image SDK 10
[ ] Đọc breaking changes compiler .NET 10
[ ] Thử extension members / field chỉ nơi giảm boilerplate thật
[ ] Span: kiểm tra overload resolution nếu API thêm ROS/Span
[ ] Script/CLI nhỏ: cân nhắc file-based apps; AOT publish mặc định
[ ] Không đưa C# 15 unions / closed vào nhánh production chưa sẵn sàng preview
```

| Nhóm | Trạng thái |
|------|------------|
| Extension members, `field`, `?.=` , nameof unbound, Span, lambda mods, partial ctor/event, compound assignment, `#:` | **C# 14 final** |
| Unions, closed, extension indexers, collection `with(…)`, labeled break, memory safety… | **C# 15 preview** |
