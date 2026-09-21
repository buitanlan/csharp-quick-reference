# Hàm Main & điểm vào chương trình

Mỗi ứng dụng C# executable cần **một** điểm vào (entry point). Baseline hiện tại: **.NET 10 / C# 14** — hỗ trợ `Main` cổ điển, **top-level statements** (C# **9+**; template mặc định từ .NET 6), và **file-based apps** (`dotnet run file.cs`, .NET 10+).

Khác Go (`func main()` không tham số / không return), C# cho phép nhiều chữ ký `Main` hợp lệ (`void`/`int`/`Task`/`Task<int>`, có hoặc không `args`, sync hoặc async). CLR thì chỉ hiểu **một** method tĩnh đồng bộ mang `.entrypoint` trong IL — `async Main` và top-level statements đều là **đường ngôn ngữ/compiler** sinh wrapper đó.

Tham số dòng lệnh: [§5](#5-tham-số-dòng-lệnh-args--environment). Top-level statements: [§3](#3-top-level-statements-c-9)–[§4](#4-tổng-hợp-program--main-bởi-compiler), [§7](#7-top-level-await--global-usings).

---

## Mục lục

- [Hàm Main \& điểm vào chương trình](#hàm-main--điểm-vào-chương-trình)
  - [Mục lục](#mục-lục)
  - [1. Chữ ký Main hợp lệ](#1-chữ-ký-main-hợp-lệ)
  - [2. Chọn entry point khi có nhiều Main](#2-chọn-entry-point-khi-có-nhiều-main)
  - [3. Top-level statements (C# 9+)](#3-top-level-statements-c-9)
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

Quy tắc ngôn ngữ:

- `Main` phải **`static`**. Type chứa `Main` và bản thân `Main` **không** bắt buộc `public` — `internal` / không modifier vẫn được (thường gặp với TLS).
- Type chứa entry **không** được generic; `Main` **không** được generic, không `params` tùy ý ngoài `string[]`, không `ref`/`in`/`out` trên `args`.
- Kiểu trả về: **chỉ** `void` / `int` / `Task` / `Task<int>`.
  - `void` / `Task` → exit code **0** trừ khi gán `Environment.ExitCode` hoặc gọi `Environment.Exit`.
  - `int` / `Task<int>` → giá trị trả về là exit code process.
- **`ValueTask` / `ValueTask<int>` không** phải entry hợp lệ. `async void Main` cũng không.
- `async Main` (C# **7.1+**): đây là feature **compiler**, không phải CLR. Runtime chỉ chạy method đồng bộ mang `.entrypoint`.

### 1.1 IL: wrapper đồng bộ và “đổi tên” async Main

CLR/PE header trỏ tới **một** method tĩnh không generic, không async. Compiler xử lý như sau (tinh thần Roslyn; tên chính xác có thể đổi theo version):

| Bạn viết | Entry IL thực tế (ý tưởng) |
|----------|----------------------------|
| `static int Main(string[] args)` | Chính method đó mang `.entrypoint` |
| `static async Task Main(...)` | Sinh wrapper **đồng bộ** (thường tên dạng `$Main` / generated) gọi `GetAwaiter().GetResult()`; method `async` giữ logic |
| Top-level statements | Class `Program` + method unspeakable `<Main>$` (§4); nếu có `await` thì thêm wrapper sync tương tự |

Hệ quả thực tế:

- Debugger / stack trace có thể hiện `$Main`, `<Main>$`, hoặc `Program.Main` — **đừng** parse tên method để phân nhánh logic.
- Không được **đổi tên** entry thành `Start` / `Run` rồi kỳ vọng runtime tìm ra: compiler chỉ nhận diện **`Main`** (hoặc TLS). Muốn tên khác → gọi từ `Main`.
- Có cả `Main` sync và `Main` async cùng type → mơ hồ, lỗi biên dịch. Chọn **một**.
- Exception chưa bắt trong async Main được unwrap qua `GetResult()` — stack hơi khác `await` trong app, nhưng process vẫn fail (thường exit ≠ 0). Tránh `.Result` / `.Wait()` thủ công trong `Main` sync nếu đã có `async Main`.

```csharp
// SAI — không phải entry hợp lệ
static string Main() => "nope";
static void Main(int code) { }
static async void Main() { }   // async void — không dùng làm Main
static async ValueTask Main() { }
static void Main<T>(string[] args) { }
```

`ildasm` / ILSpy: tìm method có flag `.entrypoint`. Đó mới là thứ OS/CLR gọi. Method `async` hiện `state machine` (`<Main>d__0` / tương đương) — **không** phải entry.

### 1.2 `[STAThread]`, WinExe, generic host

- WinForms/WPF: `[STAThread]` trên **entry sync** (wrapper). Đặt trên `async Main` nguồn có thể **không** dính wrapper — COM/`OpenFileDialog` vỡ. Template WinExe thường `Main` sync + `Application.Run`.
- `OutputType=WinExe` ẩn console; exit code vẫn có nhưng user không thấy.
- ASP.NET / generic host: `Main` gọi `builder.Build().Run()` (blocking) hoặc `RunAsync` + `await`. **Đừng** `async Main` fire-and-forget host. Lifetime: `IHostApplicationLifetime`, không `Environment.Exit` giữa request.
- `ModuleInitializer` chạy **trước** `Main` — không thay entry, dễ side-effect lúc load.

### 1.3 Thứ tự khởi động (console)

1. Load runtime / native AOT bring-up  
2. Static ctor của type chứa entry (nếu đụng member static) + module initializers  
3. **Entry sync** (`.entrypoint`) — với `async Main`/TLS await: wrapper `GetResult()`  
4. Body `Main` / `<Main>$`  
5. Return → process exit code; hoặc `Environment.Exit` cắt giữa chừng  

`static` field `HttpClient` trên `Program` khởi tạo **trước** dòng TLS đầu — lỗi ở đây không nằm trong `try` của TLS trừ khi bọc static ctor.

Bảng chọn chữ ký:

| Nhu cầu | Chọn |
|---------|------|
| CLI có code, sync | `int Main(string[] args)` |
| I/O async | `async Task<int> Main` hoặc TLS `await` + `return` |
| Không quan tâm code | `void` / `Task` + mặc định 0 |
| GUI WinForms | sync `Main` + `[STAThread]` trên **wrapper/entry** |
| Không args | bỏ `string[]` — vẫn lấy `GetCommandLineArgs` nếu cần |

---

## 2. Chọn entry point khi có nhiều Main

Khi project có nhiều method `Main` hợp lệ (ví dụ nhiều class demo trong cùng project), chỉ định entry bằng:

| Cơ chế | Nơi đặt | Ghi chú |
|--------|---------|---------|
| MSBuild `StartupObject` | `.csproj` / `Directory.Build.props` | Khuyến nghị cho SDK-style; map sang `/main:` |
| Compiler `/main:` / `-main` | CSC / `dotnet build -p:StartupObject=` | Tên **đầy đủ** của type chứa `Main` |
| Property `MainEntryPoint` | Một số template / tooling cũ | Ít gặp hơn `StartupObject` |

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net10.0</TargetFramework>
  <StartupObject>MyApp.Cli.Program</StartupObject>
</PropertyGroup>
```

- Giá trị là **tên type** (namespace + class/struct), **không** phải tên method (`MyApp.Program.Main` là sai). Nested type: `Outer+Inner` theo metadata, trong source thường `Outer.Inner` — dùng tên compiler chấp nhận (CS1555 nếu không tìm thấy).
- Type phải chứa **đúng một** `Main` hợp lệ. Hai overload `Main()` và `Main(string[])` trên cùng type → vẫn mơ hồ.
- `dotnet build -p:StartupObject=MyApp.Cli.Program` tương đương ghi trong csproj — tiện CI/tạm thời, đừng để lệch với file project lâu dài.
- Có **top-level statements** → TLS **luôn** là entry; `StartupObject` / `-main` **không** chọn được `Main` khác (§9). Muốn demo nhiều entry: xóa TLS hoặc tách project / dùng `#if` cẩn thận (dễ rối).
- Không chỉ định khi có nhiều `Main` → **CS0017**. Không có `Main` nào (và không TLS) với `OutputType=Exe` → **CS5001**.
- `OutputType=Library` không cần entry; test project thường là library hoặc có `Main` riêng — đừng để `StartupObject` trỏ nhầm sang fixture test.

Production: một executable = một entry rõ. Nhiều tool trong một repo → nhiều project `Exe`, không nhồi nhiều `Main` rồi “chọn lúc build”.

`StartupObject` **không** đổi `OutputType` và **không** tạo entry nếu type không có `Main` hợp lệ. Sai tên (typo namespace sau khi rename) → CS1555 lúc compile, không lúc chạy. Nested class: ưu tiên type top-level `Program` cho đỡ khổ `/main:`.

Khi publish Native AOT, entry vẫn là generated sync method — `StartupObject` chọn **type**, trimmer giữ `Main` của type đó. Nhiều `Main` “chết” (không được chọn) có thể bị trim; đừng để logic khởi tạo quan trọng trong `Main` không phải entry.

---

## 3. Top-level statements (C# 9+)

Từ **C# 9** (.NET 5, 11/2020), có thể viết executable code trực tiếp ở root file — không cần class `Program` / method `Main` tường minh.

> **Không phải C# 10.** Nhiều tài liệu gắn TLS với .NET 6 vì **template** `dotnet new console` đổi mặc định từ .NET 6 (C# 10 SDK). Bản thân ngôn ngữ: **C# 9**. C# 10 bổ sung `global using`, file-scoped namespace, implicit usings **SDK** — tiện cho TLS nhưng là feature khác.

Template hiện đại (.NET 6 → 10) mặc định TLS. Phù hợp utility, script, Minimal APIs, Azure Functions / Lambda nhỏ, và app lớn nếu type nằm **sau** statements.

Lịch version hay nhầm:

| Mốc | Việc |
|-----|------|
| **C# 9** / .NET 5 | TLS + top-level `await` / `args` / `return` — **đây là feature ngôn ngữ** |
| **C# 10** / .NET 6 | `global using`, file-scoped namespace, implicit usings **SDK**; template console **đổi mặc định** sang TLS (nên người học gắn nhầm “TLS = C# 10”) |
| **C# 11–13** | Không đổi mô hình entry; thêm raw string, `required`, `file` types… dùng **cùng** file TLS |
| **C# 14** / .NET 10 | File-based apps + `#:` — TLS trong **một file không csproj**, không phải phiên bản mới của TLS |

Local function, lambda, `using var` trong TLS hợp lệ như trong `Main`. Attribute trên entry (`[STAThread]`) **không** gắn lên statements — phải `partial class Program` + method, hoặc bỏ TLS.

### 3.1 Quy tắc entry

- **Một** file trong project được phép có top-level statements. File thứ hai → lỗi biên dịch (không phải warning).
- File TLS là entry **duy nhất**. Bạn vẫn *có thể* viết `static void Main(...)` ở file khác, nhưng nó **không** làm entry — compiler cảnh báo **CS7022**. `StartupObject` không “cứu” được.
- Với TLS, **không** chọn entry bằng `StartupObject` / `-main`. Muốn `Main` cổ điển: xóa statements ở root (chỉ còn using + types) hoặc chuyển code vào method.

### 3.2 Cấu trúc file

Thứ tự trong file TLS (compiler bắt buộc):

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

Đặt `class Program { }` **trước** `Console.WriteLine` → lỗi cú pháp. File-scoped namespace (`namespace MyApp;`) bao cả file nên **không** trộn TLS trong cùng file namespace — để TLS ở file không namespace (thường `Program.cs` global) và types ở file khác.

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

`return;` không giá trị khi đã có nhánh `return 0` vẫn ra `int` entry. Không khai báo `args` thủ công — đã có sẵn; shadow bằng local `args` là anti-pattern.

---

## 4. Tổng hợp Program / `<Main>$` bởi compiler

Với top-level statements, compiler sinh:

- Một class (thường tên `Program` trong **global namespace** — SDK hiện tại). Có thể `partial` thêm member ở file khác.
- Method entry có tên dạng **`<Main>$`** (unspeakable / không gõ được trong C#) chứa body TLS. Nếu TLS `async`, thêm wrapper sync mang `.entrypoint` (§1.1).

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

- Tham chiếu type `Program` (vd. `partial`) được; **không** phụ thuộc tên `<Main>$` — reflection theo `"Main"` sẽ **thất bại** trên TLS.
- Pattern phổ biến Minimal APIs / integration test: TLS + `public partial class Program { }` ở file khác để `WebApplicationFactory<Program>` thấy type.
- Stack trace hiện `<Main>$` là bình thường, không phải “code lạ”.
- `Program` mặc định **không public** — test/host cần `partial` public hoặc `InternalsVisibleTo`.
- Đừng `new Program()` kỳ vọng chạy lại entry; logic nằm ở generated method, không phải instance ctor.

Pattern test Minimal APIs:

```csharp
// Program.cs — TLS
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.MapGet("/", () => "ok");
app.Run();

// Program.Partial.cs — cùng assembly
public partial class Program { }
```

`WebApplicationFactory<Program>` cần type `Program` **visible**. TLS mặc định `internal` → `InternalsVisibleTo` hoặc `public partial`. Không tìm method `Main` bằng tên: TLS không có `Main` speakable.

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

- `args` **luôn** khác `null`; không có đối số → `Length == 0`. Không cần `args ?? []`.
- Chỉ chứa đối số **người dùng** truyền — **không** gồm tên executable. Đây là khác biệt lớn nhất với `Environment.GetCommandLineArgs()`.

```bash
dotnet run -- foo bar
# TLS / Main args:  args[0]=foo  args[1]=bar
# GetCommandLineArgs(): [0]=đường dẫn host/exe  [1]=foo  [2]=bar
```

Nhầm `args[0]` với “tên chương trình” (kiểu C `argv[0]`) là bug rất phổ biến.

### 5.2 Khi Main không nhận `args`

Vẫn lấy được dòng lệnh qua BCL:

```csharp
// Chuỗi đầy đủ (có quoting theo OS)
string line = Environment.CommandLine;

// Mảng: [0] = đường dẫn executable / host, [1..] = args người dùng
string[] all = Environment.GetCommandLineArgs();
string exe = all[0];
string[] userArgs = all.AsSpan(1).ToArray();
```

| API | `[0]` / nội dung | Khi nào dùng |
|-----|------------------|--------------|
| `Main(string[] args)` / TLS `args` | User arguments; `[0]` = arg đầu **nếu có** | Parse CLI của app |
| `Environment.GetCommandLineArgs()` | `[0]` = đường dẫn process/host; `[1..]` = user args | Cần path exe, hoặc `Main()` không khai báo `args` |
| `Environment.CommandLine` | Một string thô; quoting theo Windows/Unix | Log / debug, không parse flag |

> **`dotnet run`:** phần trước `--` thuộc CLI SDK; đối số app đặt **sau** `--`. Không có `--`, token có thể bị `dotnet run` nuốt (trông giống option của SDK). File-based: `dotnet run file.cs -- arg1 arg2` hoặc `dotnet run --file file.cs -- arg1`.

Native AOT / single-file / `dotnet file.cs`: `GetCommandLineArgs()[0]` có thể là path native exe, host `dotnet`, hoặc path file `.cs` — **đừng** hardcode giả định. Cần directory app: `AppContext.BaseDirectory` thường ổn định hơn `[0]`.

Quoting: Windows (`CommandLineToArgvW`) và Unix (shell split) khác nhau với dấu `"` / `\`. `args` đã được runtime tách; đừng split lại `Environment.CommandLine` trừ khi hiểu platform.

`args` thô không tách `--flag value` — CLI thật dùng `System.CommandLine` (.NET 10 có trong SDK/workload tùy template), Spectre.Console.Cli, Cocona, hoặc `switch` cho tool cực nhỏ.

### 5.3 Off-by-one — bảng nhanh

```text
Lệnh:  mytool.exe --verbose file.txt

Main args / TLS args:     [0]=--verbose  [1]=file.txt
GetCommandLineArgs():     [0]=C:\...\mytool.exe  [1]=--verbose  [2]=file.txt
```

`args.Length == GetCommandLineArgs().Length - 1` (thường). `Main()` không tham số: **không** có biến `args`; TLS **luôn** có. Copy-paste `args[0]` từ tutorial TLS vào `Main()` không `args` → không compile.

`Environment.GetCommandLineArgs()` trên Windows dùng `GetCommandLineW` + split; Unix: `argv` từ host. `ProcessStartInfo.ArgumentList` (không quote tay) an toàn hơn nối string khi **spawn** process con.

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
Environment.ExitCode = 1;   // đặt code, vẫn chạy hết Main (void/Task)
Environment.Exit(1);        // thoát process NGAY — không return
```

### 6.1 `return` / `ExitCode` / `Environment.Exit`

| Cách | Hành vi | Dùng khi |
|------|---------|----------|
| `return n` từ `int`/`Task<int> Main` (hoặc TLS) | Unwind stack bình thường: `finally` / `using` / `await using` chạy | **Mặc định** cho CLI |
| `Environment.ExitCode = n` | Process thoát với `n` khi `Main` là `void`/`Task` và không `Exit` | Muốn set code giữa đường, vẫn cleanup |
| `Environment.Exit(n)` | Gọi `Environment.Exit` → **kết thúc process ngay**. `finally` trên stack hiện tại **không** được đảm bảo chạy như `return`; thread khác bị cắt | Hạn chế tối đa — “kill switch”, không phải flow thường |

**Quan trọng:** `Environment.Exit` trong library = khó test, nuốt cleanup, bỏ qua `IAsyncDisposable`. Ưu tiên Main mỏng + `return` / `Task<int>`:

```csharp
static async Task<int> Main(string[] args)
{
    try { return await RunAsync(args); }
    catch (Exception ex) { Console.Error.WriteLine(ex); return 1; }
}
```

### 6.2 Nền tảng và host

- **Unix:** code process thường **8 bit** (0–255). `return 256` có thể thành `0`; `return -1` thành 255. Giữ 0–125 cho app, tránh đụng tín hiệu.
- **Windows:** `int` 32-bit; tool vẫn nên dùng 0/1/2 cho portable.
- Unhandled exception: host .NET thường thoát ≠ 0 (thường 1, hoặc mã HRESULT/negative tùy host). Đừng dựa vào số cụ thể — bắt và `return 1` nếu CLI cần ổn định.
- `WinExe` / GUI: ít khi consume exit code; vẫn nên 0 khi shutdown sạch.
- Generic host (`WebApplication.RunAsync`): lifetime/stop token quan trọng hơn `return`; đừng `Environment.Exit` giữa request.

Gán **cả** `return 0` lẫn `ExitCode = 1` — hành vi phụ thuộc thứ tự/host; chọn **một** kênh (ưu tiên `return`).

Ctrl+C: console mặc định terminate; `Console.CancelKeyPress` + `CancellationToken` rồi `return 130` (128+SIGINT) nếu CLI cần phân biệt “bị hủy” vs lỗi. `Environment.Exit` trong handler cancel = không flush log có chủ đích.

`Environment.FailFast` còn mạnh hơn `Exit`: không `finally`, không finalizer — chỉ crash dump / fail-fast, không dùng cho CLI thường.

---

## 7. Top-level await & global usings

### 7.1 Top-level await

TLS (và `async Main`) cho phép `await` trực tiếp. Runtime giữ process sống đến khi Task entry hoàn tất (wrapper `.GetResult()`).

```csharp
using var client = new HttpClient();
var html = await client.GetStringAsync("https://example.com");
Console.WriteLine(html.Length);
```

- Không cần `async` keyword ở “đầu file” — việc có `await` khiến compiler sinh `async Task` / `Task<int>` entry + wrapper sync.
- Tránh fire-and-forget (`_ = DoAsync()`) ở TLS trừ khi chủ đích; process **thoát khi entry xong**, không đợi task nền.
- Console/CLI: `ConfigureAwait(false)` ít lợi hơn ASP.NET; vẫn hữu ích trong **library** gọi từ Main.
- `await foreach` / `await using` hợp lệ ở TLS như trong `async` method.

### 7.2 Global usings

SDK thường bật `<ImplicitUsings>enable</ImplicitUsings>` (`System`, `System.Linq`, `System.Threading.Tasks`, … — tập phụ thuộc SDK Web/Worker). Hoặc `GlobalUsings.cs`:

```csharp
global using System.Net.Http.Json;
global using static System.Console;
```

- `global using` ở file khác áp dụng toàn project; `using` cục bộ trong file TLS phải **trước** statements.
- Implicit usings làm TLS ngắn nhưng dễ che dependency — khi đọc code lạ, kiểm tra `.csproj` / `GlobalUsings`. File-based apps cũng nhận implicit usings theo SDK ảo.
- `using static` + TLS: `WriteLine(...)` không qualifier — tiện script, dễ đụng tên.

---

## 8. File-based apps (.NET 10 / C# 14)

**.NET 10 SDK** giới thiệu *file-based apps*: một file `.cs` chạy **không** cần `.csproj`. SDK sinh project ảo từ file + chỉ thị `#:`.

> Không nhầm với **single-file publish** (`PublishSingleFile`) hay script cũ `.csx` / `dotnet-script`. Đây là mô hình **app chuẩn** (restore/build/publish), không phải REPL. **`dotnet publish` bật Native AOT mặc định** — khác project SDK thông thường (`PublishAot` tắt mặc định).

### 8.1 Chạy

```bash
dotnet run file.cs
dotnet run --file file.cs
dotnet file.cs          # shorthand

dotnet run file.cs -- arg1 arg2
```

> Nếu thư mục hiện tại **đã có** `.csproj`, `dotnet run file.cs` (không `--file`) có thể chạy **project** và truyền `file.cs` như **argument** — dùng `--file` (hoặc `dotnet file.cs`) để tránh nhầm. Đây là pitfall vận hành số 1.

```csharp
// hello.cs — Unix: #!/usr/bin/env -S dotnet --  rồi chmod +x
Console.WriteLine($"Hello, {args.FirstOrDefault() ?? "world"}!");
```

Shebang phải **dòng đầu**. `#:` ngay sau shebang.

### 8.2 Chỉ thị `#:` (C# 14 / preprocessor file-based)

Đặt ở đầu file (sau shebang nếu có). Chúng **không** phải `#if` / `#define` cổ điển — SDK đọc để sinh csproj ảo:

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:package Spectre.Console@0.49.1
#:property PublishAot=false
#:project ../Shared/Shared.csproj
```

| Directive | Việc |
|-----------|------|
| `#:sdk` | SDK (mặc định `Microsoft.NET.Sdk`) |
| `#:package` | NuGet — `Name@Version` hoặc `@*` (floating — cẩn thận lock) |
| `#:property` | MSBuild property (`PublishAot`, `Nullable`, `LangVersion`, …) |
| `#:project` | Project reference |
| `#:include` | Thêm file khác vào compile (SDK 11 / .NET 11; kiểm tra SDK của bạn) |

```bash
dotnet build file.cs
dotnet publish file.cs      # Native AOT bật mặc định
dotnet pack file.cs         # PackAsTool=true mặc định
dotnet project convert file.cs   # nâng lên .csproj khi app lớn
```

**AOT mặc định:** `publish` native — reflection, `Assembly.Load`, một số serializer **vỡ im lặng hoặc warning**. Prototype CLI: `#:property PublishAot=false` cho đến khi đo được trim. Chi tiết AOT: [projects-packages.md §12](projects-packages.md#12-native-aot-publishaot--overview--pitfalls).

File-based **thừa hưởng** `Directory.Build.props` / `Directory.Packages.props` / `nuget.config` / `global.json` của thư mục — đặt `hello.cs` trong monorepo lớn có thể “nhiễm” `TreatWarningsAsErrors`, TFM, CPM. Cô lập bằng thư mục trống hoặc `#:property` tường minh.

### 8.3 Entry trong file-based app

Cùng quy tắc ngôn ngữ: TLS **hoặc** `Main` cổ điển trong **một** file. Phù hợp học C#, CLI nhỏ, prototype — khi cần nhiều file / team / CI phức tạp → `dotnet project convert`. Convert giữ `#:package` thành `PackageReference`; rà lại `PublishAot` vì project thường **không** bật AOT mặc định (hành vi đổi so với file-based publish).

`#:` không phải C# 15. `#:include` trên SDK mới: nhiều file vẫn **một** entry (TLS chỉ một file trong tập compile — file include thường là type, không thêm TLS thứ hai). Hai file cùng statements → lỗi như project thường.

Shebang + `dotnet run file.cs` trên Windows (không exec bit) vẫn chạy qua `dotnet`; Unix `./file.cs` cần `chmod +x` và kernel shebang.

`#:property LangVersion=preview` trên file-based **không** biến máy thành SDK 11 — vẫn cần SDK 11. Baseline file-based = C# 14 / `net10.0`. C# 15 (union, `closed`, …) cần `#:property TargetFramework=net11.0` trên SDK 11 (RC1+: ngôn ngữ mặc định là 15, không cần `preview`). `LangVersion=preview` chỉ để thử Unsafe Evolution.

`dotnet pack file.cs` → tool NuGet (`PackAsTool`); cài `dotnet tool install --add-source`. Không phải thay `dotnet run` lúc dev.

---

## 9. Pitfalls thường gặp

1. **Nhiều file TLS** trong cùng project → lỗi biên dịch (chỉ một file được phép). Tách type ra file không có statements.
2. **Trộn TLS + `Main`**: TLS thắng; `Main` tường minh bị bỏ qua + **CS7022**. `StartupObject` **không** áp dụng khi có TLS. Muốn `Main` cổ điển: xóa body TLS.
3. **Nhiều `Main` hợp lệ** không `StartupObject` → **CS0017**. Ghi **tên type**, không tên method.
4. **`async void Main`** / `ValueTask Main` — không hợp lệ làm entry. Dùng `Task`/`Task<int>`.
5. **`args` vs `GetCommandLineArgs`**: `args[0]` là user arg đầu; `GetCommandLineArgs()[0]` là path exe/host. Off-by-one khi port từ C/`argv`.
6. **`Environment.Exit` trong library** / thay cho `return`: bỏ `finally`/`using`, khó test. CLI: `return` code; `Exit` chỉ kill switch.
7. **File-based app trong thư mục có `.csproj`**: `dotnet run file.cs` có thể chạy project — dùng `--file`. Publish AOT **mặc định** (khác csproj).
8. **Đặt type / namespace trước statements** trong file TLS → lỗi cú pháp. Types **sau** statements; namespace file-scoped đừng ôm TLS.
9. Fire-and-forget Task ở cuối TLS → process thoát sớm; `await` hết hoặc `WaitAsync` có chủ đích.
10. Phụ thuộc tên `<Main>$` / `$Main` hoặc giả định `Program` luôn `public` — artifact compiler. Test host: `partial class Program`.
11. Gắn TLS với “C# 10” — **sai version**; TLS = **C# 9**, template đổi ở .NET 6.
12. Exit code Unix wrap 8-bit; `256` → `0`. Unhandled exception ≠ ổn định như `return 1`.

---

## 10. Best practices & checklist

- Production CLI: `Task<int> Main` / TLS `return` + `RunAsync` tách riêng (dễ test, không `Exit`).
- Một entry rõ; nhiều demo `Main` → `StartupObject` **hoặc** (tốt hơn) project `Exe` riêng.
- App mới: TLS (template SDK). Script một file: file-based apps → `dotnet project convert` khi lớn / cần CI chuẩn.
- Parse arg: đừng invent parser khi tool có flag; `System.CommandLine` hoặc thư viện CLI.
- File-based: luôn ý thức **AOT publish mặc định** và `--file`.

```text
Checklist
[ ] Đúng một entry (TLS hoặc một Main được chọn)
[ ] Exit code có chủ đích; return / ExitCode — tránh Environment.Exit trừ khi cần
[ ] Phân biệt Main args vs GetCommandLineArgs[0]
[ ] Await hết công việc async trước khi process kết thúc
[ ] File-based: #: đúng; --file nếu cạnh .csproj; PublishAot có chủ đích
[ ] Không nhiều TLS file; không kỳ vọng Main cạnh TLS làm entry
[ ] Không phụ thuộc tên IL <Main>$ / $Main
[ ] Nhớ TLS là C# 9, không phải C# 10
```

| Chủ đề | Version |
|--------|---------|
| `Main` `void`/`int` + `args` | từ đầu |
| `async Task` / `Task<int> Main` (wrapper IL sync) | C# 7.1 |
| Top-level statements / await | **C# 9** / phổ biến template **.NET 6** |
| Implicit / global usings | .NET 6+ / C# 10 |
| File-based apps, `#:`, AOT publish mặc định | **.NET 10 / C# 14** |

Learn: [Main](https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/main-command-line) · [TLS](https://learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/top-level-statements) · [File-based apps](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps)
