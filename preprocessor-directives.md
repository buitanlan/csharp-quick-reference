# Preprocessor directives

Preprocessor directives ảnh hưởng tới **quá trình biên dịch**: bật/tắt code, cảnh báo, nullable, cấu hình file-based apps, v.v.  
Chúng **không** là runtime API — biểu thức trong `#if` không đọc biến runtime.

> **Baseline:** .NET **10** / C# **14**. Phần lớn directive có từ C# 1.0; `#nullable` từ C# 8; `#:` / `#!` (file-based apps) từ C# 14 / .NET 10.

---

## Mục lục

- [Preprocessor directives](#preprocessor-directives)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan \& quy tắc cú pháp](#1-tổng-quan--quy-tắc-cú-pháp)
  - [2. `#define` / `#undef`](#2-define--undef)
  - [3. `#if` / `#elif` / `#else` / `#endif`](#3-if--elif--else--endif)
  - [4. Symbol chuẩn: `DEBUG`, `TRACE`, TFM](#4-symbol-chuẩn-debug-trace-tfm)
  - [5. `ConditionalAttribute` vs preprocessor](#5-conditionalattribute-vs-preprocessor)
    - [5.1 Semantics: code biến mất vs lời gọi bị strip](#51-semantics-code-biến-mất-vs-lời-gọi-bị-strip)
    - [5.2 Argument evaluation \& pitfalls](#52-argument-evaluation--pitfalls)
    - [5.3 Khi nào chọn cái nào](#53-khi-nào-chọn-cái-nào)
  - [6. `#region` / `#endregion`](#6-region--endregion)
  - [7. `#warning` / `#error`](#7-warning--error)
  - [8. `#line`](#8-line)
  - [9. `#pragma warning` / `#pragma checksum`](#9-pragma-warning--pragma-checksum)
  - [10. `#nullable` \& nullable context](#10-nullable--nullable-context)
    - [10.1 Hai context: annotations vs warnings](#101-hai-context-annotations-vs-warnings)
    - [10.2 `enable` / `disable` / `restore`](#102-enable--disable--restore)
  - [11. File-based apps: `#!` \& `#:` (C# 14)](#11-file-based-apps----c-14)
    - [11.1 Ai xử lý `#!` / `#:` / `#if`](#111-ai-xử-lý-----if)
    - [11.2 Các `#:` phổ biến \& thứ tự](#112-các--phổ-biến--thứ-tự)
    - [11.3 Pitfalls file-based](#113-pitfalls-file-based)
  - [12. Best practices](#12-best-practices)

---

## 1. Tổng quan & quy tắc cú pháp

- Mỗi directive bắt đầu bằng `#`, **một dòng riêng** (cho phép khoảng trắng đầu dòng).
- Có thể ghi chú sau directive: `#if DEBUG // chỉ debug`.
- `#define` / `#undef` phải đứng **trước** mọi token không-phải-directive trong file (thường đặt sát đầu file).
- Symbol chỉ có hai trạng thái: **defined** / **undefined** — không gán giá trị số/string như C/C++.
- Ưu tiên định nghĩa symbol qua **MSBuild** (`DefineConstants`, cấu hình Debug/Release) thay vì `#define` rải rác trong source.

```csharp
// Trong .csproj (khuyến nghị cho project-wide symbols):
// <DefineConstants>$(DefineConstants);FEATURE_X</DefineConstants>
```

Preprocessor C# **không** macro thay thế token (`#define PI 3.14` kiểu C **không tồn tại**). Chỉ có *symbol boolean* cho `#if` và `[Conditional]`.

---

## 2. `#define` / `#undef`

**Mục đích:** Định nghĩa / hủy symbol dùng trong `#if` và `[Conditional]`.  
**Phiên bản C#:** 1.0  

```csharp
#define FEATURE_EXPERIMENTAL
#undef FEATURE_LEGACY

#if FEATURE_EXPERIMENTAL
    // biên dịch khi FEATURE_EXPERIMENTAL được define
#endif
```

**Ghi chú:**

- `#define` trong file chỉ ảnh hưởng **file đó**.
- Project/build có thể đã define sẵn (`DEBUG` ở cấu hình Debug) — `#undef DEBUG` trong file sẽ hủy cho file đó.
- Không dùng `#define` để “feature flag sản phẩm” dài hạn; xem [Best practices](#12-best-practices).

---

## 3. `#if` / `#elif` / `#else` / `#endif`

**Mục đích:** Bao gồm / loại bỏ đoạn code khỏi compilation unit.  
**Phiên bản C#:** 1.0  

```csharp
#if DEBUG
    Console.WriteLine("debug path");
#elif TRACE
    Console.WriteLine("trace-only path");
#else
    Console.WriteLine("release path");
#endif
```

**Biểu thức hợp lệ:**

| Thành phần | Ý nghĩa |
|---|---|
| `SYMBOL` | `true` nếu symbol đã define |
| `!`, `&&`, `\|\|` | phủ định / AND / OR |
| `==`, `!=` | so sánh với `true`/`false` (ít dùng) |
| `true` / `false` | hằng boolean |

```csharp
#if (NET10_0_OR_GREATER && DEBUG) || FORCE_DIAG
    // ...
#endif
```

**Không được:**

- Dùng biến runtime, gọi method, đọc config.
- Kỳ vọng nhánh `#else` “chạy khi DEBUG=false lúc runtime” — code nhánh bị loại **không còn trong IL**.

**Lồng nhau:** được phép; mỗi `#if` cần `#endif` tương ứng. IDE thường tô xám nhánh không active theo cấu hình hiện tại.

**Pitfall:** `#if` quanh `using` / type — nhánh không active không được bind; API thiếu trên TFM phải nằm trong `#if` đúng, không “comment mentally”.

---

## 4. Symbol chuẩn: `DEBUG`, `TRACE`, TFM

### `DEBUG` / `TRACE`

- **`DEBUG`**: thường bật tự động ở cấu hình **Debug** (SDK-style project).
- **`TRACE`**: thường bật ở cả Debug và Release (phụ thuộc template/property `DefineTrace`).
- Dùng cho logging/`Debug.Assert`/`Trace.WriteLine`, không dùng làm feature flag nghiệp vụ.

```csharp
#if DEBUG
    System.Diagnostics.Debug.Assert(invariant);
#endif
```

`Debug.Assert` / `Debug.WriteLine` đã gắn `[Conditional("DEBUG")]` — thường **không** cần bọc `#if DEBUG` quanh lời gọi (xem §5). `#if` vẫn cần nếu bạn khai báo *type/field* chỉ tồn tại lúc debug.

### Target Framework Moniker (TFM)

SDK định nghĩa symbol theo TFM, ví dụ:

- `NET`, `NET8_0`, `NET8_0_OR_GREATER`, `NET10_0_OR_GREATER`, …
- `NETFRAMEWORK`, `NETSTANDARD2_0`, …

```csharp
#if NET10_0_OR_GREATER
    // API chỉ có từ .NET 10
#elif NET8_0_OR_GREATER
    // fallback .NET 8/9
#endif
```

Hữu ích khi **multi-target** một library; với app single-TFM baseline .NET 10 thường ít cần.

---

## 5. `ConditionalAttribute` vs preprocessor

Hai cơ chế “có điều kiện” nhưng **semantics khác nhau**. Nhầm lẫn → hoặc nhánh Release vẫn chứa API nhạy cảm, hoặc helper debug *vẫn compile* nhưng caller kỳ vọng type biến mất.

### 5.1 Semantics: code biến mất vs lời gọi bị strip

| | `#if` / preprocessor | `[Conditional("DEBUG")]` |
|---|---|---|
| Thời điểm | Loại bỏ **cả khối source** khỏi compilation | Method **vẫn compile** vào assembly; **lời gọi** bị loại nếu symbol không define |
| Phạm vi | Bất kỳ đoạn: type, field, `using`, statement | Method `void` (và một số attribute: `Conditional` trên attribute class) |
| IL của callee | Không tồn tại nếu bọc hết method | Method **còn** trong DLL (có thể gọi reflection) |
| IL của caller | Không có opcode nhánh tắt | **Không** có `call` tới method đó |
| Side-effect ở arg | Code chết — không evaluate | Compiler **loại cả lời gọi** → **không** evaluate argument |
| Return value | Có thể bao hàm bất kỳ | **Chỉ `void`** — không `[Conditional]` trên `Func<T>` |
| Use case | API/TFM/platform khác nhau | `Debug.WriteLine`, helper chẩn đoán |

```csharp
using System.Diagnostics;

public static class Diag
{
    [Conditional("DEBUG")]
    public static void Log(string message) => Console.WriteLine(message);
}

void Run()
{
    Diag.Log(BuildExpensiveMessage()); // Release: lời gọi bị loại (không gọi BuildExpensiveMessage)
}
```

**WHY `Conditional` tồn tại:** giữ *một* surface API (`Debug.Assert(condition)`) trong mọi build; Release không trả giá điều kiện (và không gọi). `#if` quanh từng `Assert` sẽ phình code.

### 5.2 Argument evaluation & pitfalls

```csharp
[Conditional("DEBUG")]
static void Trace(object x) => Console.WriteLine(x);

int n = 0;
Trace(n++);        // Release: n KHÔNG tăng — lời gọi biến mất hoàn toàn

#if DEBUG
Trace(n++);        // tương đương ý đồ, nhưng phải lặp #if
#endif
```

**Pitfall 1 — dựa vào side-effect argument:** `Debug.Assert(Save() != null)` — Release **không** gọi `Save()`. Assert chỉ cho *kiểm tra*, không cho *công việc bắt buộc*.

**Pitfall 2 — `[Conditional]` không xóa method:**

```csharp
[Conditional("DEBUG")]
public static void DumpSecrets(string token) => Console.WriteLine(token);
// Release: caller bị strip, nhưng DumpSecrets vẫn có trong metadata → reflection vẫn gọi được
```

Bí mật / API không được tồn tại ở Release → `#if DEBUG` **cả method** (hoặc không ship vào binary).

**Pitfall 3 — không áp dụng overload có return:**

```csharp
// SAI — Conditional chỉ cho void
// [Conditional("DEBUG")]
// public static string Describe() => "...";
```

**Pitfall 4 — interface:** method có Conditional không được implement interface member (CS0629), và không được đặt Conditional trên interface method. Nếu contract cần diagnostics, dùng method helper riêng hoặc #if ở call site.

**Pitfall 5 — `Conditional` trên attribute:** `[Conditional("DEBUG")]` trên *class attribute* → attribute đó bị loại khỏi metadata nếu symbol off (`[Obsolete]` không dùng kiểu này).

```csharp
[Conditional("DEBUG")]
[AttributeUsage(AttributeTargets.Method)]
public sealed class DebugOnlyAttribute : Attribute { }

[DebugOnly] // Release: attribute không gắn
void M() { }
```

### 5.3 Khi nào chọn cái nào

- Cần **API / type / using** khác nhau theo platform/TFM → `#if`.
- Chỉ muốn tắt **lời gọi chẩn đoán** mà giữ signature method → `[Conditional]`.
- Tránh nhân đôi logic nghiệp vụ lớn trong `#if`/`#else`.
- Kết hợp hợp lệ: method `[Conditional("DEBUG")]` *bên trong* vẫn có thể `#if NET10_0` cho API mới — hai trục khác nhau (cấu hình vs TFM).

```csharp
public static class TraceOs
{
    [Conditional("DEBUG")]
    public static void Info(string msg)
    {
#if NET10_0_OR_GREATER
        Console.WriteLine($"[net10] {msg}");
#else
        Console.WriteLine(msg);
#endif
    }
}
```

**Tóm tắt quyết định:**

```
Cần type/API biến mất khỏi IL Release / TFM khác?
  ├─ Có → #if
  └─ Không, chỉ tắt lời gọi chẩn đoán?
        ├─ Method void, gọi trực tiếp → [Conditional]
        └─ Có return / gọi qua interface → #if quanh call-site, hoặc API riêng
```

`Debug.WriteLine` / `Trace.WriteLine` đã `[Conditional]` — đừng bọc thêm `#if DEBUG` trừ khi bạn cần *khối* nhiều statement không phải một lời gọi.

---

## 6. `#region` / `#endregion`

**Mục đích:** Gom nhóm code để IDE collapse/expand. **Không** ảnh hưởng IL.  
**Phiên bản C#:** 1.0  

```csharp
#region Public API
public void Start() { }
public void Stop() { }
#endregion
```

**Ghi chú:** Lạm dụng region để che “code mùi” là anti-pattern; ưu tiên tách class/file nhỏ hơn.

---

## 7. `#warning` / `#error`

**Mục đích:** Sinh warning / error compile-time có chủ đích.  
**Phiên bản C#:** 1.0  

```csharp
#warning TODO: migrate off legacy auth before next release

#if NETFRAMEWORK
#error This library requires .NET 8+ (not .NET Framework).
#endif
```

- `#warning`: build vẫn có thể thành công (trừ khi treat warnings as errors).
- `#error`: **fail build** — hợp lý cho cấu hình/TFM không được hỗ trợ, hoặc guard tạm thời khi refactor.

---

## 8. `#line`

**Mục đích:** Ghi đè số dòng / tên file trong diagnostic (phổ biến với **source generator** / Razor / tool sinh code).  
**Phiên bản C#:** 1.0  

```csharp
#line 200 "GeneratedFile.cs"
int x = "oops"; // diagnostic báo dòng 200, file GeneratedFile.cs
#line default   // trở lại mapping thật
#line hidden    // ẩn khỏi debugger step-through (một số tooling)
```

Hiếm khi viết tay trong app code thường ngày.

---

## 9. `#pragma warning` / `#pragma checksum`

### `#pragma warning`

**Mục đích:** Tắt / bật lại warning theo mã (CS… / IDE…).  

```csharp
#pragma warning disable CS0168 // biến khai báo nhưng không dùng
void Foo()
{
    int unused;
}
#pragma warning restore CS0168
```

```csharp
#pragma warning disable IDE0005, CS8618
// phạm vi hẹp…
#pragma warning restore IDE0005, CS8618
```

**Quy tắc:** disable **phạm vi nhỏ nhất**, kèm comment lý do; tránh `disable` cả file trừ generated code.

### `#pragma checksum`

Dùng cho debugger / ASP.NET để gắn checksum file nguồn (thường do tooling sinh). Ít viết thủ công:

```csharp
#pragma checksum "file.cs" "{406EA660-64CF-4C3B-A9B0-...}" "hex..."
```

---

## 10. `#nullable` & nullable context

**Mục đích:** Điều khiển **nullable annotation context** và **nullable warning context** theo vùng/file.  
**Phiên bản C#:** 8.0  

Baseline .NET 10 template thường bật nullable project-wide (`<Nullable>enable</Nullable>`). Directive hữu ích khi migrate dần hoặc với generated code.

```csharp
#nullable enable
string? maybe = null;     // OK
string must = null;       // warning

#nullable disable
string old = null;        // không warning NRT

#nullable restore         // về cấu hình Nullable của project
```

### 10.1 Hai context: annotations vs warnings

Nullable **không** phải một công tắc duy nhất:

| Context | Việc compiler làm |
|---|---|
| **Annotation** | `string` nghĩa là non-null; `string?` nghĩa là có thể null. Metadata `NullableAttribute` được ghi (khi enable). |
| **Warning** | Phát CS86xx khi bạn vi phạm annotation (gán null, dereference). |

Tách hai context để migrate: bật annotation (API đúng `?`) trước, bật warning sau; hoặc ngược lại trong generated code.

`<Nullable>` trong `.csproj`: `enable` / `disable` / `warnings` / `annotations` — cùng mô hình với directive.

### 10.2 `enable` / `disable` / `restore`

| Directive | Annotations | Warnings |
|---|---|---|
| `#nullable enable` | Bật | Bật |
| `#nullable disable` | Tắt (`string` không còn nghĩa non-null) | Tắt |
| `#nullable restore` | Khôi phục mặc định project | Khôi phục mặc định project |
| `#nullable enable annotations` | Bật | Giữ nguyên warning context |
| `#nullable disable annotations` | Tắt | Giữ nguyên |
| `#nullable enable warnings` | Giữ nguyên | Bật |
| `#nullable disable warnings` | Giữ nguyên | Tắt |
| `#nullable restore annotations` / `restore warnings` | Khôi phục từng phần | |

`#nullable disable` trong generated file: tránh hàng nghìn warning, nhưng **caller** thấy API không có `?` — dễ hiểu nhầm non-null. Generator hiện đại nên `#nullable enable` + annotate đúng.

```csharp
#nullable enable annotations
public string? Find(string id) => lookup.GetValueOrDefault(id);

#nullable enable warnings
var x = Find("a").Length; // warning nếu Find trả string?
```

**`#nullable enable` vs project `disable`:** file có thể opt-in từng phần. Restore về **cấu hình project**, không nhất thiết enable và không nhớ directive trước đó.

**Tương tác với preprocessor:** `#if` có thể bao quanh `#nullable`, nhưng **đừng** dùng `#if DEBUG` để bật/tắt nullable khác nhau giữa Debug/Release — dễ lệch hành vi phân tích giữa môi trường (cùng code, khác cảnh báo / khác ý nghĩa `string`).

**Pitfall `?` khi annotations disable:** `string?` có thể cảnh báo CS8632 (nullable annotation không có ngữ cảnh). Bật annotation hoặc bỏ `?`.

Oblivious vs nullable: code cũ không `?` khi disable = *oblivious* (compiler không biết null hay không). Library oblivious + consumer enable → warning khi dereference tùy flow.

**Không có stack nullable context:** restore không pop trạng thái trước. Lặp restore vẫn cho cùng mặc định project. Ví dụ dưới giả định `<Nullable>enable</Nullable>`:

```csharp
#nullable disable
#nullable enable
string a = null;       // warning
#nullable restore      // về project enable
string b = null;       // vẫn warning
#nullable restore      // vẫn project enable
```

Generated code: `#nullable disable warnings` giữ annotation để caller thấy `string?`, nhưng file gen không spam CS86xx. `#nullable disable` cả hai → caller mất thông tin null.

`<Nullable>enable</Nullable>` + file `#nullable disable` = file đó oblivious; đừng làm vậy cho public API.

---

## 11. File-based apps: `#!` & `#:` (C# 14)

Từ **C# 14 / .NET 10**, file-based apps (`dotnet run app.cs`) hỗ trợ directive cấu hình **không phải** conditional compilation cổ điển. Chi tiết entry/TLS: [main-function.md §8](main-function.md#8-file-based-apps-net-10--c-14).

### 11.1 Ai xử lý `#!` / `#:` / `#if`

| Directive | Ai xử lý | Vai trò | Compiler C# thấy? |
|---|---|---|---|
| `#!` | OS / shell (shebang) | Cho phép `./app.cs` trên Unix | Thường bỏ qua / không là C# token |
| `#:`… | SDK xử lý cấu hình build; compiler nhận diện directive | Package, property, SDK thay csproj | Có; cần chế độ file-based và đúng vị trí |
| `#if` / `#define`… | **Compiler** | Conditional compilation như cũ | Có |

**WHY `#:` không phải `#if`:** SDK đọc package/property để tạo build của file-based app. Compiler vẫn nhận diện cú pháp directive và kiểm tra vị trí; nó không tự restore NuGet hoặc dựng project.

Trong project-based compilation thông thường, `#:` gây **error CS9298** nếu không bật chế độ file-based phù hợp. Chuyển cấu hình sang csproj khi convert app.

```csharp
#!/usr/bin/env dotnet
#:sdk Microsoft.NET.Sdk
#:package Spectre.Console@*
#:property Nullable=enable
#:property PublishAot=false

#if DEBUG
Console.WriteLine("file-based + DEBUG");
#endif
```

Shebang phải **dòng đầu** (Unix). Windows `dotnet run app.cs` bỏ qua `#!`.

### 11.2 Các `#:` phổ biến & thứ tự

- `#:package Package@version` — NuGet (`@*` / version range tùy SDK)
- `#:property Name=Value` — MSBuild property (`Nullable`, `PublishAot`, `DefineConstants`, `TargetFramework`, …)
- `#:sdk Some.Sdk` — đổi SDK (ví dụ `Microsoft.NET.Sdk.Web`)
- `#:project path` — tham chiếu project
- `#:include path` — thêm source từ **SDK 10.0.300+** / .NET 11 Preview 3+; include DLL cần .NET 11. [File-based directives](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps).

**Thứ tự:** `#!` → các `#:` → `using` / TLS. `#:` sau statement C# thường không hợp lệ (phải đầu file, trước token C# — tương tự `#define`).

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:package Microsoft.AspNetCore.OpenApi@10.*
#:property TargetFramework=net10.0

var app = WebApplication.Create(args);
app.MapGet("/", () => "ok");
app.Run();
```

Muốn symbol biên dịch: `#:property DefineConstants=FEATURE_X` (cộng dồn tùy SDK) hoặc `#define` trong file. `#:package` **không** define symbol.

### 11.3 Pitfalls file-based

1. **Nhầm `#:` với preprocessor** — không viết `#:if DEBUG`. Dùng `#if DEBUG` như project thường; `DEBUG` vẫn theo cấu hình `dotnet run`.
2. **Copy `#:` vào class library `.csproj`** — build thông thường báo lỗi; chuyển cấu hình sang csproj, dùng `dotnet project convert` khi app lớn.
3. **Version lock** — `@*` tiện prototype, CI/production nên pin version.
4. **Nhiều file** — file-based mặc định một file; `#:include` / convert project khi tách type.
5. **`#nullable` vs `#:property Nullable=`** — property là mặc định project ảo; `#nullable` trong file vẫn ghi đè vùng. Nên `#:property Nullable=enable` + code annotated.
6. **TFM** — không ghi `#:property` → SDK chọn mặc định (.NET 10 trên toolchain hiện tại); multi-target không phải use case file-based.

```csharp
#:property Nullable=enable
#:property DefineConstants=TRACE

#nullable enable
string? q = args.Length > 0 ? args[0] : null;
#if TRACE
Console.WriteLine(q);
#endif
```

---

## 12. Best practices

1. **Không dùng preprocessor làm feature flag sản phẩm**  
   Ưu tiên cấu hình runtime (`IConfiguration`, options, feature management). `#if` nhân bản binary path → khó test đủ nhánh, khó ship một build.

2. **Giữ `#if` cho biên giới thật sự của compile**  
   TFM/API khác nhau, platform (`WINDOWS`/`LINUX` nếu có), bỏ debug-only *types*.

3. **Định nghĩa symbol ở project/CI**, không rải `#define` trong nhiều file.

4. **`[Conditional]` cho diagnostics**; `#if` khi cả khối type/API phải biến mất. Đừng dựa side-effect argument của `Debug.Assert`.

5. **`#pragma warning`**: phạm vi hẹp + lý do; prefer sửa root cause.

6. **`#region`**: tổ chức nhẹ; không thay thế thiết kế module tốt.

7. **Nullable**: bật project-wide; `#nullable` chỉ để migrate / generated code — tránh Debug≠Release. Hiểu tách **annotations** vs **warnings**.

8. **File-based apps**: `#:` cho script/prototype (package/property/SDK); `#if` vẫn cho TFM/DEBUG. Khi lớn hãy `dotnet project convert` sang `.csproj`.

9. **Đo / review IL** khi nghi ngờ nhánh `#if` (đảm bảo API nhạy cảm không lọt vào Release). Reflection vẫn thấy method `[Conditional]` .
