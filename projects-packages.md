# Project, SDK & NuGet

Tham chiếu sâu về **project SDK-style**, NuGet, và CLI `dotnet` trên baseline **.NET 10 / C# 14**.  
Lớp *build/identity* của ứng dụng .NET (tương tự packages/modules bên Go) — không phải cú pháp ngôn ngữ thuần. Compiler C# nhận **source + references**; MSBuild/SDK quyết định TFM, `LangVersion`, restore, publish. Sai ở lớp này thì “code đúng vẫn không build / chạy sai runtime”.

> **Baseline:** .NET **10** LTS (GA 11/2025 · hỗ trợ đến **14/11/2028**) · TFM `net10.0` · ngôn ngữ mặc định **C# 14**.  
> C# **15** / .NET **11** = Preview (Preview 7 · 08/2026) — `<LangVersion>preview</LangVersion>` + SDK 11; không phải baseline repo. Surface preview **đổi trước GA** — đừng pin production vào union / `closed` / labeled `break`.

---

## Mục lục

- [Project, SDK \& NuGet](#project-sdk--nuget)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan: SDK-style csproj](#1-tổng-quan-sdk-style-csproj)
  - [2. `TargetFramework`, `LangVersion` \& cấu hình cốt lõi](#2-targetframework-langversion--cấu-hình-cốt-lõi)
  - [3. Implicit usings \& nullable](#3-implicit-usings--nullable)
  - [4. `ProjectReference` vs `PackageReference`](#4-projectreference-vs-packagereference)
  - [5. NuGet restore](#5-nuget-restore)
  - [6. Central Package Management (CPM)](#6-central-package-management-cpm)
  - [7. Global tools \& local tools](#7-global-tools--local-tools)
  - [8. `InternalsVisibleTo`](#8-internalsvisibleto)
  - [9. Multi-targeting (tóm tắt)](#9-multi-targeting-tóm-tắt)
  - [10. `Directory.Build.props` / `.targets`](#10-directorybuildprops--targets)
  - [11. CLI: `dotnet new` / `build` / `run` / `test` / `publish`](#11-cli-dotnet-new--build--run--test--publish)
  - [12. Native AOT (`PublishAot`) — overview \& pitfalls](#12-native-aot-publishaot--overview--pitfalls)
  - [13. File-based apps — `dotnet run app.cs` (.NET 10)](#13-file-based-apps--dotnet-run-appcs-net-10)
  - [14. Best practices \& checklist](#14-best-practices--checklist)

---

## 1. Tổng quan: SDK-style csproj

Từ .NET Core, project dùng **SDK-style** `.csproj` (XML ngắn, convention-over-configuration). SDK là bộ props/targets MSBuild (`Sdk="…"`), không phải “format khác MSBuild”. Old-style (liệt kê từng `.cs`, `packages.config`) vẫn gặp ở .NET Framework cổ — **đừng** trộn mental model.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

- **`Sdk="Microsoft.NET.Sdk"`**: console/classlib mặc định. Web: `Microsoft.NET.Sdk.Web` (IIS/Kestrel targets, implicit ASP.NET usings); Worker: `Microsoft.NET.Sdk.Worker`; Razor/Blazor: SDK tương ứng. Sai SDK → thiếu target, publish lạ, implicit usings khác.
- SDK tự include `**/*.cs` (và content theo SDK), resolve `PackageReference`, chuỗi `Restore` → `Compile` → `CopyToOutput` → …  
- Không cần liệt kê từng `.cs` trừ `Compile Remove` / `EnableDefaultCompileItems=false`. File generated trong `obj/` đã được exclude đúng cách nếu không custom bừa.
- Solution (`.sln` / `.slnx`) gom project; `dotnet build MyApp.sln` theo graph phụ thuộc. SDK-style **không** thay solution — chỉ làm từng csproj ngắn.
- `EnableDefaultItems`: tắt khi generate code ra cây source và bị compile hai lần.

> SDK-style vẫn là MSBuild — property/item/target đầy đủ. “Không ghi trong csproj” ≠ “không tồn tại”: mặc định nằm trong SDK. Debug bằng `dotnet msbuild -pp` / `-bl` khi restore/publish “bí”.

File-based apps (.NET 10) **không** có csproj trên đĩa nhưng vẫn đi qua cùng pipeline (project ảo + `#:`) — xem §13.

### 1.1 Old-style vs SDK-style

| | Old-style (.NET Framework) | SDK-style |
|--|------------------------------|-----------|
| Include source | Từng `<Compile Include>` | Glob `**/*.cs` |
| Package | `packages.config` / `HintPath` | `PackageReference` + restore assets |
| Verbose | Hàng trăm dòng XML | Vài property |
| Debug “thiếu file” | Nhìn csproj | `Compile Remove`, glob, `obj/` generated |

Port: `dotnet migrate` / tạo csproj mới rồi copy source — đừng sửa tay `ToolsVersion=15` nửa vời. SDK-style **chạy được** trên .NET Framework (`net481`) với SDK hiện đại, nhưng TFM + App.config khác `net10.0`.

Imports ẩn: `Sdk="Microsoft.NET.Sdk"` = props đầu file + targets cuối. Ghi `<Import>` trùng SDK → restore/compile hai lần, warning khó đọc.

`Microsoft.NET.Sdk.Web` kéo Kestrel/IIS integration, `Content` wwwroot, implicit `Microsoft.AspNetCore.*` usings — **đừng** dùng cho classlib thuần. Worker SDK thêm `BackgroundService` hosting. Razor SDK compile `.cshtml`/`.razor`. Chọn sai SDK là nguyên nhân “sao không thấy `WebApplication`”.

`<EnableDefaultCompileItems>false</EnableDefaultCompileItems>` khi codegen emit `.cs` vào project dir và glob compile trùng. `Compile Remove="**/Generated/**"` tinh hơn tắt hết glob.

---

## 2. `TargetFramework`, `LangVersion` & cấu hình cốt lõi

### 2.1 TFM

```xml
<TargetFramework>net10.0</TargetFramework>
<TargetFrameworks>net10.0;net8.0</TargetFrameworks> <!-- multi — mục 9 -->
```

- `net10.0` = .NET 10: **API surface BCL** lúc compile + runtime tối thiểu lúc chạy. App `net10.0` **không** chạy trên runtime 8.
- Legacy: `net481`, `netstandard2.0`… vẫn gặp ở thư viện đa nền. `netstandard2.0` không có API hiện đại (`Span` một phần nhờ polyfill, không có Generic Math đầy đủ, …).
- OS-specific: `net10.0-windows` (WinForms/WPF), `net10.0-android`, … — TFM kèm workload. Đừng dùng `-windows` nếu muốn Linux CI.
- TFM **không** đồng nghĩa `LangVersion`. Có thể `net8.0` + C# 12 (SDK 8) hoặc (không khuyến nghị) `net10.0` + `LangVersion=12` để tạm tránh syntax mới.

Bảng TFM hay gặp (app 2026):

| TFM | Runtime | Ghi chú |
|-----|---------|---------|
| `net10.0` | .NET 10 LTS | **Baseline repo** |
| `net9.0` / `net8.0` | STS / LTS cũ | EOS ~11/2026 — đừng mở app mới |
| `netstandard2.0` | Nhiều runtime | Thư viện legacy; API hẹp |
| `net481` | .NET Framework | Windows; không phải Core |
| `net10.0-windows` | .NET 10 + Windows | WinForms/WPF |

`TargetFrameworkIdentifier` / `NETCoreApp` preprocessor: `#if NET10_0_OR_GREATER` do SDK define theo **TFM đang compile**, không theo SDK máy.

### 2.2 `LangVersion`

```xml
<LangVersion>14.0</LangVersion>
<!-- latest = bản GA cao nhất SDK hiểu; preview = C# 15 trên SDK 11 -->
```

- SDK .NET 10 mặc định gắn **C# 14** với `net10.0` — **không cần** ghi `LangVersion` trừ khi hạ (compat) hoặc bật preview.
- Hạ `LangVersion` ≠ hạ được API runtime: thiếu `TimeSpan.FromSeconds(double)` variant hay LINQ mới thì phải hạ **TFM** hoặc polyfill, không phải “đổi số ngôn ngữ”.
- `latest` theo **SDK máy build**, không theo TFM — CI SDK 11 + `latest` có thể biên dịch C# 15 vào binary `net10.0` (một số feature cần runtime 11). Pin `14.0` trên nhánh production nếu team sợ “lọt preview”.
- `preview` + TFM `net11.0` chỉ khi theo dõi **C# 15** (union, `closed`, labeled `break`, collection `with(…)` …) — **đổi trước GA**. Bảng feature: §14.

`global.json` pin **SDK** (compiler + targets), khác `LangVersion` (cờ compiler). Cả hai cần nhất quán trên CI.

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature",
    "allowPrerelease": false
  }
}
```

`allowPrerelease: true` trên máy dev dễ kéo SDK 11 preview → compile C# 15 **nhầm** nhánh GA. CI production: `false` + version 10.x. `rollForward: latestFeature` = 10.0.x mới; `latestMajor` có thể nhảy 11 khi GA — thường **không** muốn trên LTS pin.

Giá trị `LangVersion` thường gặp: `14.0` (pin GA), `latest` (GA cao nhất **SDK hiểu**), `preview` (C# 15), `13.0`/`12.0` (hạ syntax, API TFM vẫn .NET 10). `ISO-2`/`ISO-1` cổ — đừng dùng.

### 2.3 Property thường gặp

| Property | Ý nghĩa |
|---|---|
| `OutputType` | `Exe` / `Library` / `WinExe` |
| `AssemblyName` / `RootNamespace` | Tên assembly / namespace mặc định |
| `Nullable` | `enable` / `disable` / `warnings` / `annotations` |
| `ImplicitUsings` | `enable` / `disable` |
| `TreatWarningsAsErrors` | Warning → lỗi CI |
| `Deterministic` | Build lặp lại được (`true` khuyến nghị) |
| `InvariantGlobalization` | Gọt ICU — hay đi với AOT/container |
| `PublishAot` / `PublishTrimmed` / `PublishSingleFile` | Artifact publish (§11–12) |

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <Deterministic>true</Deterministic>
</PropertyGroup>
```

Đặt các property “chuẩn repo” ở `Directory.Build.props` (§10), không copy 20 lần. Override local bằng csproj khi một project (generated, legacy) không chịu nổi `TreatWarningsAsErrors`.

---

## 3. Implicit usings & nullable

### 3.1 Implicit usings — **.NET 6+**

`ImplicitUsings=enable` → SDK sinh `global using` theo **loại SDK** (console: `System`, `System.Linq`, `System.Threading.Tasks`…; Web thêm ASP.NET). Không phải “compiler tự đoán”: file generated trong `obj/` — đọc được khi IDE “từ đâu ra `HttpClient`?”.

```xml
<ItemGroup>
  <Using Include="System.Text.Json" />
  <Using Remove="System.Net.Http" />
</ItemGroup>
```

```csharp
global using System.Text; // C# 10+ — file GlobalUsings.cs
```

Tắt implicit rồi `using` tay khi viết thư viện public muốn dependency **nhìn thấy** trong source. App: giữ enable cho ngắn.

### 3.2 Nullable reference types — **C# 8+**

```xml
<Nullable>enable</Nullable>
```

- Phân biệt `string` vs `string?`; cảnh báo dereference null. Đây là **annotation + warning**, không phải runtime check (trừ NRT helpers / Roslyn analyzers).
- `#nullable enable/disable` theo file khi port từng phần; mục tiêu cuối: enable toàn solution.
- Template `dotnet new` hiện đại thường đã bật cả hai — **giữ nguyên** trừ lý do mạnh (generated code, interop).
- Nâng TFM không “tự sửa” warning nullable cũ — bật `TreatWarningsAsErrors` sẽ lộ nợ.

---

## 4. `ProjectReference` vs `PackageReference`

### 4.1 Project reference

```xml
<ItemGroup>
  <ProjectReference Include="..\Shared\Shared.csproj" />
</ItemGroup>
```

- Build theo phụ thuộc; đổi source phản ánh ngay.  
- Phù hợp monorepo / cùng solution; dễ kết hợp `InternalsVisibleTo`.
- Cycle reference → lỗi MSBuild. Multi-target: MSBuild chọn TFM nearest — hiểu `SetTargetFramework` khi debug.

### 4.2 Package reference

```xml
<ItemGroup>
  <PackageReference Include="Serilog" Version="4.2.0" />
  <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

- Semantic version / range (dùng range có chủ đích; CPM thường pin exact).  
- Analyzer/build-only: `PrivateAssets=all` để **không** chảy xuống consumer (tránh “cài library bị kéo Roslyn”).

| Tình huống | Chọn |
|---|---|
| Code nội bộ, iterate nhanh | `ProjectReference` |
| Thư viện phiên bản hóa / bên thứ ba | `PackageReference` |
| Test cần internals của SUT | `ProjectReference` + `InternalsVisibleTo` |

> Tránh vừa project vừa package **cùng assembly** trong một graph (hai bản `Shared.dll` — bind fail lúc runtime).

---

## 5. NuGet restore

Restore tải package về cache (`%USERPROFILE%\.nuget\packages` / `~/.nuget/packages`) và ghi assets trong `obj/` (`project.assets.json`). Restore **không** phải “copy DLL vào bin” — compile đọc assets.

```bash
dotnet restore
dotnet build              # thường restore ngầm
dotnet build --no-restore
dotnet list package --outdated
dotnet list package --vulnerable
dotnet nuget why Serilog
```

`nuget.config` — nên `<clear />` rồi khai báo source tường minh (tránh máy dev “dính” feed lạ):

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

- CI: cache global packages theo hash `packages.lock.json` nếu bật lock.  
- Conflict: nearest-wins / unify — `dotnet nuget why` khi “sao ra bản cũ”.  
- `RestoreLockedMode` trên CI khi đã commit lock file.

---

## 6. Central Package Management (CPM)

**Tooling:** NuGet 6.2+ / .NET SDK 6.0.300+ (ổn định trên .NET 10). Solution ≥ vài project: **nên** CPM — một chỗ bump `Serilog`, không grep 15 csproj.

`Directory.Packages.props` (root repo, cạnh `Directory.Build.props`):

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <!-- Pin cả transitive: chặt, có thể vỡ restore nếu graph khó -->
    <!-- <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled> -->
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.2.0" />
    <PackageVersion Include="xunit" Version="2.9.3" />
  </ItemGroup>
</Project>
```

Trong `.csproj` — **không** ghi `Version`:

```xml
<PackageReference Include="Serilog" />
```

Quy tắc dễ sai:

- Ghi `Version=` trên `PackageReference` khi CPM bật → lỗi restore (trừ `VersionOverride`).  
- Override có chủ đích: `VersionOverride="…"`. Opt-out một project: `ManagePackageVersionsCentrally=false` (legacy / generated).  
- Chỉ auto-import **một** `Directory.Packages.props` **gần nhất** theo cây thư mục — repo lồng (submodule, samples) dễ lấy nhầm file.  
- Transitive pinning: thống nhất bản transitive; bật khi compliance/security đòi hỏi, đo restore trước.  
- File-based apps trong repo CPM: `#:package Name@version` vs version trung tâm — kiểm tra SDK có tôn trọng `PackageVersion` không; đừng giả định.  
- Scaffold: `dotnet new packagesprops`.

CPM **không** thay `Directory.Build.props` (TFM/nullable) và **không** pin SDK (`global.json`). Ba lớp: SDK · ngôn ngữ/TFM · phiên bản package.

`dotnet list package` với CPM vẫn chạy theo project; bump version = sửa **một** `PackageVersion`. PR “nâng Serilog” không còn 12 file csproj — dễ review. Central file conflict git: merge `Directory.Packages.props` cẩn thận hơn merge Version rải rác.

Package **không** có trong `PackageVersion` nhưng csproj `PackageReference Include="Foo"` → restore fail (thiếu version). Analyzer/SDK pack (`Microsoft.CodeAnalysis.NetAnalyzers`) cũng nên vào CPM nếu dùng nhiều project.

---

## 7. Global tools & local tools

```bash
# Global
dotnet tool install -g dotnet-ef
dotnet tool list -g

# Local (khuyến nghị team) — commit .config/dotnet-tools.json
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool restore
dotnet tool run dotnet-ef -- --help
```

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": { "version": "10.0.0", "commands": ["dotnet-ef"] }
  }
}
```

CI: `dotnet tool restore` trước khi gọi tool. Global tool trên agent dùng chung = version lệch. Local tools = cùng version cho cả team. Tool `net8.0` vẫn chạy trên SDK 10 trong nhiều trường hợp — vẫn pin major theo runtime bạn test.

---

## 8. `InternalsVisibleTo`

Cho assembly “friend” thấy thành viên `internal`:

```csharp
using System.Runtime.CompilerServices;
[assembly: InternalsVisibleTo("MyApp.Tests")]
```

```xml
<ItemGroup>
  <InternalsVisibleTo Include="MyApp.Tests" />
</ItemGroup>
```

- Khớp **assembly name** (không phải tên project nếu `AssemblyName` khác).  
- Strong-name: cần `PublicKey=` trong thuộc tính (public key token **không** đủ ở một số toolchain — dùng full public key).  
- Phổ biến cho unit test — đừng phá encapsulation giữa layer production (`InternalsVisibleTo` cho mọi service = `public` trá hình).

---

## 9. Multi-targeting (tóm tắt)

```xml
<TargetFrameworks>net10.0;net8.0;netstandard2.0</TargetFrameworks>
```

```csharp
#if NET10_0_OR_GREATER
    // API .NET 10+
#elif NET8_0_OR_GREATER
    // fallback
#endif
```

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
  <PackageReference Include="System.Memory" Version="4.5.5" />
</ItemGroup>
```

Mỗi TFM nhân chi phí CI/test/pack. Thư viện public: multi-target khi **thực sự** còn consumer cũ. App nội bộ: một TFM `net10.0`. `NET10_0_OR_GREATER` do SDK define — đừng tự `#define` trùng.

---

## 10. `Directory.Build.props` / `.targets`

MSBuild tự import theo cây thư mục (đi lên từ csproj):

| File | Vai trò |
|---|---|
| `Directory.Build.props` | Property/item **đầu** (trước csproj) — mặc định repo |
| `Directory.Build.targets` | Target **cuối** (sau csproj) — hook pack/publish |
| `Directory.Packages.props` | Version NuGet (CPM) |
| `global.json` | Pin SDK version |

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>14.0</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

File-based apps (.NET 10) cũng thừa hưởng các file này — **cẩn thận** khi đặt script cạnh monorepo lớn (`TreatWarningsAsErrors`, TFM, CPM). `Import` vòng / props đặt `TargetFramework` quá sớm có thể đè template.

---

## 11. CLI: `dotnet new` / `build` / `run` / `test` / `publish`

```bash
dotnet new console -n MyApp -o MyApp --framework net10.0
dotnet new classlib -n MyLib
dotnet new xunit -n MyApp.Tests
dotnet new sln -n MySolution
dotnet sln add MyApp/MyApp.csproj

dotnet build -c Release
dotnet run --project MyApp -- arg1 arg2
dotnet test --filter "FullyQualifiedName~MyNamespace"
dotnet watch run --project MyApp

dotnet publish MyApp -c Release -o ./publish
dotnet publish -c Release -r win-x64 --self-contained true
dotnet publish -c Release -r linux-x64 -p:PublishSingleFile=true
```

| Chế độ publish | Ý |
|---|---|
| Framework-dependent (mặc định csproj) | Cần runtime .NET trên máy đích |
| `--self-contained` | Kèm runtime; artifact lớn hơn |
| `PublishSingleFile` | Một file (có thể extract native) — **không** phải file-based apps |
| `PublishAot` | Native AOT (mục 12) |
| `PublishTrimmed` | Cắt IL — rủi ro reflection |

`dotnet run --` tách arg app (xem [main-function.md §5](main-function.md#5-tham-số-dòng-lệnh-args--environment)). RID (`-r`) bắt buộc với AOT/self-contained.

---

## 12. Native AOT (`PublishAot`) — overview & pitfalls

**Ổn định từ .NET 7+;** .NET 10 mở rộng compatibility (JIT/GC/BCL micro-opts khi nâng TFM — hot path vẫn đo BenchmarkDotNet). SDK ≥ 10.0.x trên CI. Binary **không** còn IL + JIT (trừ runtime bring-up): không `Assembly.Load` plugin tùy ý, không emit.

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r win-x64
```

**Lợi ích:** startup nhanh, footprint nhỏ, binary native self-contained — CLI, serverless cold-start, container scratch.

**Pitfalls (đọc trước khi bật trên production):**

- **Reflection/dynamic hạn chế** — warning trim/AOT (`IL2026`, `IL3050`, `IL2104`…). Warning trên CI phải xử lý, không “tắt ILLink”.  
- **Không phải mọi library AOT-friendly.** Serializer reflection-heavy (Newtonsoft cũ, `BinaryFormatter` đã obsolete), plugin `Assembly.Load*`, `Regex` compiled một số path, COM — cần source generator / `JsonSerializerContext` / annotation `[DynamicallyAccessedMembers]`.  
- Compile **lâu**; cần toolchain native (MSVC trên Windows, clang/gcc + native deps trên Linux). CI image “SDK-only” thiếu C++ workload → fail lúc publish, không lúc `dotnet build`.  
- Trimming cắt method “chỉ gọi qua reflection” → fail **lúc chạy**. Test trên **artifact AOT**, không chỉ `Debug` JIT.  
- `stackalloc` / `Span` ổn; `ref struct` trong generic cần `allows ref struct` (C# 13) — không phải bug AOT nhưng hay lộ khi trim.  
- **File-based apps (.NET 10) bật `PublishAot` mặc định** khi `dotnet publish file.cs` — **khác** csproj. Tắt: `#:property PublishAot=false`. Convert sang project rồi quên AOT → hành vi publish đổi (JIT FDD).  
- `InvariantGlobalization` / ICU: AOT + globalization đầy đủ làm binary to; app không format theo culture → invariant.

```csharp
[RequiresUnreferencedCode("Uses reflection")]
public static void Risky() { /* ... */ }
```

> CLI tool / cold-start → AOT hấp dẫn. Plugin động / reflection nặng / “load DLL lúc chạy” → giữ JIT (FDD hoặc self-contained không AOT). Đừng bật AOT chỉ vì “.NET 10 mới”.

### 12.1 Checklist AOT trước khi ship

```text
[ ] Publish RID đúng (win-x64 / linux-x64 / …) — AOT không “AnyCPU”
[ ] Zero warning ILLink/ILCompiler trên cấu hình Release
[ ] Test **binary publish**, không chỉ `dotnet test` JIT
[ ] JSON: source gen (`JsonSerializerContext`), không reflection mặc định nếu trim
[ ] Không Assembly.Load plugin; không emit
[ ] File-based: nhớ mặc định AOT — tắt tường minh nếu chưa sẵn sàng
[ ] So sánh size/startup với FDD self-contained (đôi khi FDD đủ)
```

`IsAotCompatible` / `EnableAotAnalyzer` trên classlib giúp bắt warning sớm khi thư viện bị app AOT kéo vào.

Trimmer mặc định **aggressive** hơn JIT: `MakeGenericType` lúc chạy, `Enum.Parse` một số path, COM, `ConfigurationBinder` bind phức tạp — đọc [Native AOT compatibility](https://learn.microsoft.com/dotnet/core/deploying/native-aot/). `PublishAot` **kéo** `PublishTrimmed`. Tắt trim nhưng giữ AOT không phải mô hình hỗ trợ.

Debug AOT: `DOTNET_ReadyToRun`, dump, log ILC — chậm iteration. Dev loop: `dotnet run` JIT; CI job publish AOT + smoke test.

---

## 13. File-based apps — `dotnet run app.cs` (.NET 10)

**Áp dụng:** .NET **10 SDK+** — chạy/publish một `.cs` không cần `.csproj`. SDK materialize project ảo (restore NuGet, compile, có thể AOT). **Không** phải `.csx` / `dotnet-script` (hệ scripting khác, không cùng publish model).

```csharp
#:package Spectre.Console@0.49.1
#:property PublishAot=false

using Spectre.Console;
AnsiConsole.MarkupLine("[green]Hello[/]");
```

```bash
dotnet run app.cs
dotnet run --file app.cs   # an toàn khi thư mục có .csproj
dotnet app.cs              # shorthand
dotnet run app.cs -- arg1
dotnet publish app.cs      # AOT ON mặc định
dotnet pack app.cs         # PackAsTool=true mặc định
dotnet project convert app.cs
```

| Directive `#:` | Việc |
|---|---|
| `#:package Id@version` | NuGet |
| `#:project path` | ProjectReference |
| `#:property Name=Value` | MSBuild property |
| `#:sdk …` | Đổi SDK (vd. Web) |
| `#:include other.cs` | Thêm file (SDK mới hơn — kiểm tra version) |

**Mặc định khác csproj:** `PublishAot=true`, `PackAsTool=true`. Tôn trọng `Directory.Build.props` / CPM / `nuget.config` / `global.json` — script trong monorepo “dính” `TreatWarningsAsErrors` là bình thường.

Nếu có `.csproj` trong cwd, `dotnet run app.cs` không `--file` có thể coi `app.cs` là **argument** của project (chạy `Main` project, `args[0]=app.cs`). Dùng `--file` hoặc `dotnet app.cs`. Chi tiết entry/args: [main-function.md §8](main-function.md#8-file-based-apps-net-10--c-14).

Dùng cho script/utility/prototype; app lớn / nhiều file / team → `dotnet project convert`. Sau convert: rà `PublishAot` (project **không** mặc định AOT) và CPM (`PackageReference` không `Version` nếu repo đang CPM).

`#:sdk Microsoft.NET.Sdk.Web` + TLS `WebApplication.CreateBuilder` = Minimal API một file; publish AOT Web **khó hơn** console (trim endpoint/JSON). Prototype: `PublishAot=false`. `#:package` floating `@*` = restore non-reproducible — pin version như CPM.

Cache restore file-based thường dưới thư mục user/temp SDK — xóa `bin`/`obj` cạnh file **không** luôn xóa cache ảo; `dotnet clean file.cs` / xóa thư mục generated khi “package không lên”.

---

## 14. Best practices & checklist

- Pin SDK bằng `global.json` trên CI/team (`rollForward` có chủ đích, ví dụ `latestFeature` trong band 10.x).  
- Chuẩn hóa `Nullable` + `ImplicitUsings` + `TreatWarningsAsErrors` + `LangVersion` (pin **14.0** trên nhánh GA) ở `Directory.Build.props`.  
- Solution lớn → CPM. Tooling team → local tools.  
- Phân biệt `ProjectReference` (nội bộ) vs package (biên giới version).  
- Publish: chọn FDD / self-contained / single-file / AOT **có chủ đích**; đọc warning AOT/trim trước khi ship. File-based apps **bật `PublishAot` mặc định**.  
- Không nhét file-based app vào cây project nếu sợ “nhiễm” props — hoặc `#:property` tường minh.  
- .NET 8/9 EOS ~ **10/11/2026** — production dài hạn nên đã ở **10**. Không đưa C# 15 unions / `closed` vào nhánh GA.

### 14.1 Đọc bảng C# 14 / 15 như thế nào

Bảng dưới **không** phải changelog đầy đủ và **không** thay topic file. Repo này tra cứu **theo chủ đề** (`oop.md`, `operators.md`, …): mỗi hàng là *cổng version* — feature nằm rải trong file tương ứng, kèm pitfall.

- **C# 14 final** đi với baseline **.NET 10**. Dùng được trên `net10.0` + SDK 10, không cần `LangVersion=preview`. Extension members, `field`, `?.=`, `nameof` unbound, chuyển đổi Span, modifier trên lambda, partial ctor/event, compound assignment, `#:` (file-based) — GA.  
- **C# 15 preview** cần SDK **11** + thường `net11.0` / `LangVersion=preview`. Union, `closed`, extension indexer, collection `with(…)`, labeled `break`/`continue`, một số memory safety — **đổi trước GA** (Preview 7 · 08/2026). Copy snippet preview vào app `net10.0` production → không compile hoặc cần preview compiler “lọt” binary không hỗ trợ runtime.  
- Nâng **TFM** `net8.0` → `net10.0` mang API BCL + (mặc định) C# 14. Nâng **SDK** trên CI mà không pin `LangVersion` có thể kéo syntax mới hơn TFM — đó là lý do checklist ghi pin 14.0.  
- Breaking compiler .NET 10 (overload `Span`/`ReadOnlySpan`, analyzer mới) lộ khi **đổi TFM/SDK**, không chỉ khi “viết C# 14”. Đo build warning = 0 trước khi bật `TreatWarningsAsErrors` trên nhánh chính.

C# 14 **final** có thể dùng ngay trên SDK 10; đừng ghi `preview` “cho chắc”. C# 15 **preview** không backport đầy đủ lên net10.0: union/`closed` cần toolchain 11. Extension **members** (C# 14) ≠ extension **indexer** (C# 15 preview) — nhầm bảng là compile fail.

Nâng 8 → 10: đọc [breaking changes](https://learn.microsoft.com/dotnet/core/compatibility/10.0) (BCL + SDK + container images), không chỉ `LangVersion`. Analyzer `CA`/`IDE` mới có thể ồn — baseline `.editorconfig` trước khi `TreatWarningsAsErrors`.

```text
Checklist nâng cấp → .NET 10 / C# 14
[ ] TFM net10.0; CI image SDK 10; global.json pin
[ ] LangVersion 14.0 trên production (tránh latest/preview)
[ ] Nullable + ImplicitUsings
[ ] Directory.Build.props + nuget.config rõ nguồn
[ ] CPM nếu ≥ vài project; csproj không còn Version= khi CPM
[ ] Đọc breaking changes compiler .NET 10
[ ] Span: kiểm tra overload resolution nếu API thêm ROS/Span
[ ] Script/CLI nhỏ: cân nhắc file-based apps; AOT publish mặc định
[ ] Publish: FDD vs AOT vs single-file có chủ đích; test artifact AOT nếu bật
[ ] Không đưa C# 15 preview vào production
[ ] InternalsVisibleTo cho test (nếu cần)
[ ] CI: restore → build → test → (publish)
```

| Nhóm ngôn ngữ | Trạng thái | Topic |
|------|------------|-------|
| Extension members, `field`, `?.=` , `nameof` unbound, Span conversions, lambda mods, partial ctor/event, compound assignment, `#:` | **C# 14 final** | `oop` / `operators` / `memory-spans` / `delegates-lambdas` / `preprocessor` / `main-function` |
| Unions, `closed`, extension indexers, collection `with(…)`, labeled `break`/`continue`, memory safety… | **C# 15 preview** | `typesystem` / `oop` / `collections-generics` / `statements` / `memory-spans` |

Tài nguyên: [C# 14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14) · [.NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview) · [C# 15 preview](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [File-based apps](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps) · [Native AOT](https://learn.microsoft.com/dotnet/core/deploying/native-aot/)
