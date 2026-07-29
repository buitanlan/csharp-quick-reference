# Hàm Main & điểm vào chương trình

Mỗi ứng dụng C# executable cần **một** điểm vào (entry point). Baseline hiện tại: **.NET 10 / C# 14** — hỗ trợ `Main` cổ điển, **top-level statements** (C# 10+), và **file-based apps** (`dotnet run file.cs`, .NET 10+).

Khác Go (`func main()` không tham số / không return), C# cho phép nhiều chữ ký `Main` hợp lệ (`void`/`int`/`Task`/`Task<int>`, có hoặc không `args`, sync hoặc async).

> Stub cũ: [top-level-statements.md](main-function/top-level-statements.md) · [parameter.md](main-function/parameter.md) — nội dung đã gộp vào trang này.

---

## Mục lục

- [Hàm Main \& điểm vào chương trình](#hàm-main--điểm-vào-chương-trình)
  - [Mục lục](#mục-lục)
  - [1. Chữ ký Main hợp lệ](#1-chữ-ký-main-hợp-lệ)
  - [2. Chọn entry point khi có nhiều Main](#2-chọn-entry-point-khi-có-nhiều-main)
  - [3. Top-level statements (C# 10+)](#3-top-level-statements-c-10)
  - [4. Tổng hợp Program / `<Main>$` bởi compiler](#4-tổng-hợp-program--main-bởi-compiler)
  - [5. Tham số dòng lệnh: `args` \& Environment](#5-tham-số-dòng-lệnh-args--environment)
  - [6. Mã thoát (exit codes)](#6-mã-thoát-exit-codes)
  - [7. Top-level await \& global usings](#7-top-level-await--global-usings)
  - [8. File-based apps (.NET 10 / C# 14)](#8-file-based-apps-net-10--c-14)
  - [9. Pitfalls thường gặp](#9-pitfalls-thường-gặp)
  - [10. Best practices \& checklist](#10-best-practices--checklist)

---

## 1. Chữ ký Main hợp lệ

Tám chữ ký được công nhận làm entry point:

```csharp
static void Main() { }
static int Main() { }
static void Main(string[] args) { }
static int Main(string[] args) { }
static async Task Main() { }
static async Task<int> Main() { }
static async Task Main(string[] args) { }
static async Task<int> Main(string[] args) { }
```

Quy tắc:

- `Main` phải **`static`**. Type chứa `Main` và bản thân `Main` **không** bắt buộc `public`.
- Kiểu trả về: `void` / `int` / `Task` / `Task<int>`.
  - `void` / `Task` → exit code **0** (trừ khi `Environment.Exit`).
  - `int` / `Task<int>` → giá trị trả về là exit code.
- `async Main` (C# 7.1+): compiler sinh entry đồng bộ chờ Task hoàn tất (implementation chi tiết có thể đổi theo version).
- Một assembly executable chỉ có **một** entry được chọn. Nhiều `Main` → §2.

```csharp
// SAI — không phải entry hợp lệ
static string Main() => "nope";
static void Main(int code) { }
static async void Main() { }   // async void — không dùng làm Main
```

---

## 2. Chọn entry point khi có nhiều Main

Khi project có nhiều method `Main` hợp lệ (ví dụ nhiều class demo trong cùng project), chỉ định entry bằng:

| Cơ chế | Nơi đặt | Ghi chú |
|--------|---------|---------|
| MSBuild `StartupObject` | `.csproj` | Khuyến nghị cho SDK-style |
| Compiler `/main:` / `-main` | CSC / CLI | Tên đầy đủ của type chứa `Main` |
| Property `MainEntryPoint` | Một số template / tooling cũ | Ít gặp hơn `StartupObject` |

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net10.0</TargetFramework>
  <StartupObject>MyApp.Cli.Program</StartupObject>
</PropertyGroup>
```

- Giá trị là **tên type** (namespace + class), không phải tên method. `dotnet build -p:StartupObject=…` tương đương `/main:`.
- Type phải chứa đúng một `Main` hợp lệ.
- Có **top-level statements** → TLS **luôn** là entry; `StartupObject` / `-main` không chọn được `Main` khác (§9).

---

## 3. Top-level statements (C# 10+)

Từ C# 10, có thể viết executable code trực tiếp ở root file — không cần class `Program` / method `Main` tường minh. `dotnet new console` mặc định dùng TLS.

```csharp
Console.WriteLine("Hello World!");
```

Phù hợp utility, script, Azure Functions / Lambda nhỏ, và cả app lớn hơn nếu tổ chức type phía sau statements.

### 3.1 Quy tắc entry

- **Một** file trong project được phép có top-level statements.
- File TLS là entry point duy nhất. Bạn vẫn *có thể* viết `static void Main(...)` ở file khác, nhưng nó **không** làm entry — compiler cảnh báo CS7022.
- Với TLS, **không** chọn entry bằng `StartupObject` / `-main`.

### 3.2 Cấu trúc file

Thứ tự trong file TLS:

1. (Tùy chọn) shebang / `#:` directives — chủ yếu với **file-based apps** (§8)
2. `using` / `global using` (nếu khai báo cục bộ)
3. Top-level statements
4. Type / namespace declarations (nếu có) — **sau** statements

```csharp
using System.Text;

var builder = new StringBuilder();
foreach (var arg in args)
    builder.AppendLine(arg);
Console.WriteLine(builder);

public static class Helpers { }  // types phải SAU statements
```

### 3.3 `args`, `await`, `return`

Trong TLS luôn có biến ẩn `args` kiểu `string[]` (không bao giờ `null`). Có thể `await` và `return int`:

```csharp
if (args.Length == 0)
{
    Console.Error.WriteLine("usage: tool <file>");
    return 2;
}

await Task.Delay(10);
Console.WriteLine(args[0]);
return 0;
```

Chữ ký `Main` được compiler suy ra:

| TLS chứa | Implicit entry (ý tưởng) |
|----------|--------------------------|
| không `await`, không `return` | `static void Main(string[] args)` |
| có `return`, không `await` | `static int Main(string[] args)` |
| có `await`, không `return` | `static async Task Main(string[] args)` |
| có `await` và `return` | `static async Task<int> Main(string[] args)` |

---

## 4. Tổng hợp Program / `<Main>$` bởi compiler

Với top-level statements, compiler sinh:

- Một class (thường tên `Program` trong global namespace — có thể đổi bằng thuộc tính / internals tùy version, nhưng mặc định SDK là `Program`).
- Method entry có tên dạng `<Main>$` (unspeakable name) chứa body TLS.

Minh họa tinh thần (không phải IL chính xác 100%):

```csharp
// Bạn viết:
Console.WriteLine("Hello");

// Compiler roughly:
internal class Program
{
    private static void <Main>$(string[] args)
    {
        Console.WriteLine("Hello");
    }
}
```

Hệ quả:

- Tham chiếu type `Program` (vd. `partial`) được; **không** phụ thuộc tên `<Main>$`.
- Pattern phổ biến Minimal APIs: TLS + `public partial class Program { }` ở file khác (tests / `WebApplicationFactory`).
- Stack trace hiện `<Main>$` là bình thường.

---

## 5. Tham số dòng lệnh: `args` & Environment

### 5.1 `string[] args` trên Main / TLS

```csharp
static int Main(string[] args)
{
    // args không bao giờ null
    if (args.Length == 0)
    {
        Console.Error.WriteLine("missing args");
        return 2;
    }
    Console.WriteLine(string.Join(", ", args));
    return 0;
}
```

- `args` **luôn** khác `null`; không có đối số → `Length == 0`.
- Chỉ chứa đối số người dùng truyền — **không** gồm tên executable (khác `Environment.GetCommandLineArgs()`).

```bash
dotnet run -- foo bar
# args[0]=foo args[1]=bar
```

### 5.2 Khi Main không nhận `args`

Vẫn lấy được dòng lệnh qua BCL:

```csharp
// Chuỗi đầy đủ (có quoting theo OS)
string line = Environment.CommandLine;

// Mảng: [0] = đường dẫn executable / host, [1..] = args
string[] all = Environment.GetCommandLineArgs();
string exe = all[0];
string[] userArgs = all.AsSpan(1).ToArray();
```

| API | Nội dung |
|-----|----------|
| `Main(string[] args)` / TLS `args` | Chỉ user arguments |
| `Environment.GetCommandLineArgs()` | Executable + user arguments |
| `Environment.CommandLine` | Một string; parsing quoting theo platform |

> **Lưu ý**: Với `dotnet run`, phần trước `--` thuộc CLI; đối số app đặt **sau** `--`. File-based apps: `dotnet run file.cs -- arg1 arg2`.

`args` thô không tách flag — CLI thật dùng `System.CommandLine`, Spectre.Console.Cli, Cocona, hoặc tự `switch` cho tool cực nhỏ.

---

## 6. Mã thoát (exit codes)

Convention phổ biến (Unix / CLI):

| Code | Ý nghĩa thông dụng |
|------|--------------------|
| 0 | Thành công |
| 1 | Lỗi chung |
| 2 | Sai cách dùng / argument (usage) |

```csharp
static int Main(string[] args) => args.Length == 0 ? 2 : 0;          // return
static async Task<int> Main(string[] args) => await RunAsync(args); // async
Environment.ExitCode = 1;   // đặt code, vẫn chạy hết Main
Environment.Exit(1);        // thoát process NGAY
```

**Quan trọng:** `Environment.Exit` kết thúc ngay — tránh trong library. Ưu tiên Main mỏng + `return` / `Task<int>`:

```csharp
static async Task<int> Main(string[] args)
{
    try { return await RunAsync(args); }
    catch (Exception ex) { Console.Error.WriteLine(ex); return 1; }
}
```

---

## 7. Top-level await & global usings

### 7.1 Top-level await

TLS (và `async Main`) cho phép `await` trực tiếp. Runtime giữ process sống đến khi Task entry hoàn tất.

```csharp
using var client = new HttpClient();
var html = await client.GetStringAsync("https://example.com");
Console.WriteLine(html.Length);
```

- Không cần `async` keyword ở “đầu file” — việc có `await` khiến compiler sinh `async Task` / `Task<int>` entry.
- Tránh fire-and-forget (`_ = DoAsync()`) ở TLS trừ khi chủ đích; process có thể thoát trước khi hoàn tất nếu entry không await.

### 7.2 Global usings

SDK thường bật `<ImplicitUsings>enable</ImplicitUsings>` (`System`, `System.Linq`, …). Hoặc `GlobalUsings.cs`:

```csharp
global using System.Net.Http.Json;
global using static System.Console;
```

- `global using` ở file khác áp dụng toàn project; `using` cục bộ trong file TLS phải **trước** statements.
- Implicit usings làm TLS ngắn nhưng dễ che dependency — khi đọc code lạ, kiểm tra `.csproj` / `GlobalUsings`.

---

## 8. File-based apps (.NET 10 / C# 14)

**.NET 10 SDK** giới thiệu *file-based apps*: một file `.cs` chạy **không** cần `.csproj`. SDK sinh project ảo từ file + chỉ thị `#:`.

> Không nhầm với **single-file publish** (`PublishSingleFile`) hay script cũ `.csx` / `dotnet-script`. Đây là mô hình app chuẩn, AOT publish mặc định khi `dotnet publish`.

### 8.1 Chạy

```bash
dotnet run file.cs
dotnet run --file file.cs
dotnet file.cs          # shorthand

dotnet run file.cs -- arg1 arg2
```

> Nếu thư mục hiện tại **đã có** `.csproj`, `dotnet run file.cs` (không `--file`) có thể chạy **project** và truyền `file.cs` như argument — dùng `--file` để tránh nhầm.

```csharp
// hello.cs — Unix: #!/usr/bin/env -S dotnet --  rồi chmod +x
Console.WriteLine($"Hello, {args.FirstOrDefault() ?? "world"}!");
```

### 8.2 Chỉ thị `#:` (C# 14 / preprocessor file-based)

Đặt ở đầu file (sau shebang nếu có):

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:package Spectre.Console@0.49.1
#:property PublishAot=false
#:project ../Shared/Shared.csproj
```

| Directive | Việc |
|-----------|------|
| `#:sdk` | SDK (mặc định `Microsoft.NET.Sdk`) |
| `#:package` | NuGet — `Name@Version` hoặc `@*` |
| `#:property` | MSBuild property |
| `#:project` | Project reference |
| `#:include` | Thêm file khác vào compile (SDK mới hơn / .NET 11 preview+; kiểm tra SDK của bạn) |

```bash
dotnet build file.cs
dotnet publish file.cs      # Native AOT bật mặc định
dotnet pack file.cs         # PackAsTool=true mặc định
dotnet project convert file.cs   # nâng lên .csproj khi app lớn
```

### 8.3 Entry trong file-based app

Cùng quy tắc ngôn ngữ: TLS **hoặc** `Main` cổ điển trong **một** file. Phù hợp học C#, CLI nhỏ, prototype — khi cần nhiều file / team / CI phức tạp → `dotnet project convert`.

---

## 9. Pitfalls thường gặp

1. **Nhiều file TLS** trong cùng project → lỗi biên dịch (chỉ một file được phép có top-level statements).
2. **Trộn TLS + `Main`**: TLS thắng; `Main` tường minh bị bỏ qua + cảnh báo CS7022. Đừng dùng `StartupObject` để “cứu” — không áp dụng khi có TLS.
3. **Nhiều `Main` hợp lệ** không chỉ định `StartupObject` → lỗi CS0017.
4. **`async void Main`** — không hợp lệ làm entry.
5. **`args` vs `GetCommandLineArgs`**: nhầm index 0 (user arg vs đường dẫn exe).
6. **`Environment.Exit` trong library** / sau khi đăng ký cleanup phức tạp → khó test, bỏ qua luồng return bình thường.
7. **File-based app trong thư mục có `.csproj`**: nhầm lệnh `dotnet run` — dùng `--file`.
8. **Đặt type trước statements** trong file TLS → lỗi cú pháp.
9. Fire-and-forget Task ở cuối TLS → process thoát sớm.
10. Phụ thuộc tên method `<Main>$` hoặc giả định `Program` luôn public — tránh; coi là artifact compiler.

---

## 10. Best practices & checklist

- Production CLI: `Task<int> Main` / TLS `return` + `RunAsync` tách riêng.
- Một entry rõ; nhiều demo `Main` → `StartupObject` hoặc project riêng.
- App mới: TLS (template SDK). Script một file: file-based apps → `dotnet project convert` khi lớn.

```text
Checklist
[ ] Đúng một entry (TLS hoặc một Main được chọn)
[ ] Exit code có chủ đích; tránh Environment.Exit trừ khi cần
[ ] Phân biệt Main args vs GetCommandLineArgs
[ ] Await hết công việc async trước khi process kết thúc
[ ] File-based: #: đúng; dùng --file nếu cạnh .csproj
[ ] Không nhiều TLS file; không kỳ vọng Main cạnh TLS làm entry
```

| Chủ đề | Version |
|--------|---------|
| `Main` `void`/`int` + `args` | từ đầu |
| `async Task` / `Task<int> Main` | C# 7.1 |
| Top-level statements / await | C# 10 / .NET 6 |
| Implicit / global usings | .NET 6+ |
| File-based apps, `#:` | **.NET 10 / C# 14** |

Learn: [Main](https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/main-command-line) · [TLS](https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/top-level-statements) · [File-based apps](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps) · [C# 14 hub](csharp14-dotnet10.md)
