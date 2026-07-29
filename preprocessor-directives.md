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
  - [6. `#region` / `#endregion`](#6-region--endregion)
  - [7. `#warning` / `#error`](#7-warning--error)
  - [8. `#line`](#8-line)
  - [9. `#pragma warning` / `#pragma checksum`](#9-pragma-warning--pragma-checksum)
  - [10. `#nullable` \& nullable context](#10-nullable--nullable-context)
  - [11. File-based apps: `#!` \& `#:` (C# 14)](#11-file-based-apps----c-14)
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

Hai cơ chế “có điều kiện” nhưng **semantics khác nhau**:

| | `#if` / preprocessor | `[Conditional("DEBUG")]` |
|---|---|---|
| Thời điểm | Loại bỏ **cả khối source** khỏi compilation | Method vẫn compile; **lời gọi** bị loại nếu symbol không define |
| Phạm vi | Bất kỳ đoạn code | Chỉ áp dụng cho method `void` (và một số attribute) |
| Side-effect ở arg | Không tồn tại (code chết) | **Đối số vẫn được evaluate** rồi lời gọi bị strip? — thực tế compiler loại lời gọi; **không evaluate** arg của lời gọi bị loại |
| Use case | API khác nhau theo TFM, platform | `Debug.WriteLine`, helper chẩn đoán |

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

**Quy tắc chọn:**

- Cần **API / type / using** khác nhau theo platform → `#if`.
- Chỉ muốn tắt **lời gọi chẩn đoán** mà giữ signature method → `[Conditional]`.
- Tránh nhân đôi logic nghiệp vụ lớn trong `#if`/`#else`.

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

#nullable restore         // về lại trạng thái trước khối / theo project
```

| Directive | Ý nghĩa ngắn |
|---|---|
| `#nullable enable` | Bật annotations + warnings |
| `#nullable disable` | Tắt cả hai |
| `#nullable restore` | Khôi phục context bao ngoài / project |
| `#nullable enable annotations` | Chỉ annotations |
| `#nullable enable warnings` | Chỉ warnings |

**Tương tác với preprocessor:** `#if` có thể bao quanh `#nullable`, nhưng **đừng** dùng `#if DEBUG` để bật/tắt nullable khác nhau giữa Debug/Release — dễ lệch hành vi phân tích giữa môi trường.

---

## 11. File-based apps: `#!` & `#:` (C# 14)

Từ **C# 14 / .NET 10**, file-based apps (`dotnet run app.cs`) hỗ trợ directive cấu hình **không phải** conditional compilation cổ điển:

| Directive | Ai xử lý | Vai trò |
|---|---|---|
| `#!` | OS / shell (shebang) | Cho phép `./app.cs` trên Unix |
| `#:`… | **Build system** (SDK), compiler **bỏ qua** | Cấu hình package, property, SDK… thay `.csproj` |
| `#if` / `#define`… | **Compiler** | Conditional compilation như cũ |

Trong **project-based** compilation, `#:` thường gây warning (không dùng trong `.csproj` apps).

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

Các `#:` phổ biến:

- `#:package Package@version` — NuGet
- `#:property Name=Value` — MSBuild property
- `#:sdk Some.Sdk` — đổi SDK (ví dụ Web)
- `#:project path` — tham chiếu project
- `#:include path` — include thêm file (SDK mới hơn; kiểm tra phiên bản SDK)

**Lưu ý:** `#:` **không** thay `#if` — không define symbol biên dịch; muốn symbol thì `#:property DefineConstants=...` hoặc `#define` trong file.

---

## 12. Best practices

1. **Không dùng preprocessor làm feature flag sản phẩm**  
   Ưu tiên cấu hình runtime (`IConfiguration`, options, feature management). `#if` nhân bản binary path → khó test đủ nhánh, khó ship một build.

2. **Giữ `#if` cho biên giới thật sự của compile**  
   TFM/API khác nhau, platform (`WINDOWS`/`LINUX` nếu có), bỏ debug-only assertions.

3. **Định nghĩa symbol ở project/CI**, không rải `#define` trong nhiều file.

4. **`[Conditional]` cho diagnostics**; `#if` khi cả khối type/API phải biến mất.

5. **`#pragma warning`**: phạm vi hẹp + lý do; prefer sửa root cause.

6. **`#region`**: tổ chức nhẹ; không thay thế thiết kế module tốt.

7. **Nullable**: bật project-wide; `#nullable` chỉ để migrate / generated code — tránh Debug≠Release.

8. **File-based apps**: dùng `#:` cho script/prototype; khi lớn hãy `dotnet project convert` sang `.csproj`.

9. **Đo / review IL** khi nghi ngờ nhánh `#if` (đảm bảo API nhạy cảm không lọt vào Release).
