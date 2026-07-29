# Project, SDK & NuGet

Tham chiếu sâu về **project SDK-style**, NuGet, và CLI `dotnet` trên baseline **.NET 10 / C# 14**.  
Lớp *build/identity* của ứng dụng .NET (tương tự packages/modules bên Go) — không phải cú pháp ngôn ngữ thuần.

> **Baseline:** .NET **10** SDK · TFM thường `net10.0` · ngôn ngữ mặc định **C# 14**.

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

Từ .NET Core, project dùng **SDK-style** `.csproj` (XML ngắn, convention-over-configuration):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

- **`Sdk="Microsoft.NET.Sdk"`**: console/classlib mặc định. Web: `Microsoft.NET.Sdk.Web`; Worker: `Microsoft.NET.Sdk.Worker`.  
- SDK tự include `**/*.cs`, resolve `PackageReference`, chạy `Restore` → `Compile` → …  
- Không cần liệt kê từng `.cs` (trừ exclude/include tường minh).  
- Solution (`.sln` / `.slnx`) gom project; `dotnet build MyApp.sln` theo graph phụ thuộc.

> SDK-style vẫn là MSBuild — chỉ có mặc định thông minh hơn.

---

## 2. `TargetFramework`, `LangVersion` & cấu hình cốt lõi

### 2.1 TFM

```xml
<TargetFramework>net10.0</TargetFramework>
<TargetFrameworks>net10.0;net8.0</TargetFrameworks> <!-- multi — mục 9 -->
```

- `net10.0` = .NET 10 (BCL/API surface tương ứng).  
- Legacy: `net481`, `netstandard2.0`… vẫn gặp ở thư viện đa nền.  
- TFM quyết định API lúc biên dịch và runtime lúc chạy.

### 2.2 `LangVersion`

```xml
<LangVersion>14.0</LangVersion>
<!-- hoặc latest / preview (cẩn thận) -->
```

- SDK .NET 10 mặc định gắn **C# 14** với `net10.0`.  
- Hạ `LangVersion` ≠ hạ được API runtime — thiếu API thì phải hạ TFM.  
- `preview` chỉ khi chủ động theo dõi breaking change.

### 2.3 Property thường gặp

| Property | Ý nghĩa |
|---|---|
| `OutputType` | `Exe` / `Library` / `WinExe` |
| `AssemblyName` / `RootNamespace` | Tên assembly / namespace mặc định |
| `Nullable` | `enable` / `disable` / `warnings` / `annotations` |
| `ImplicitUsings` | `enable` / `disable` |
| `TreatWarningsAsErrors` | Warning → lỗi CI |
| `Deterministic` | Build lặp lại được (`true` khuyến nghị) |

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <Deterministic>true</Deterministic>
</PropertyGroup>
```

---

## 3. Implicit usings & nullable

### 3.1 Implicit usings — **.NET 6+**

`ImplicitUsings=enable` → SDK sinh `global using` theo loại SDK (console: `System`, `System.Linq`, `System.Threading.Tasks`…).

```xml
<ItemGroup>
  <Using Include="System.Text.Json" />
  <Using Remove="System.Net.Http" />
</ItemGroup>
```

```csharp
global using System.Text; // C# 10+
```

### 3.2 Nullable reference types — **C# 8+**

```xml
<Nullable>enable</Nullable>
```

- Phân biệt `string` vs `string?`; cảnh báo dereference null.  
- Có thể `#nullable enable/disable` theo file.  
- Template `dotnet new` hiện đại thường đã bật cả hai — giữ nguyên trừ lý do mạnh.

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

- Semantic version / range (dùng range có chủ đích).  
- Analyzer/build-only: `PrivateAssets=all` để không chảy xuống consumer.

| Tình huống | Chọn |
|---|---|
| Code nội bộ, iterate nhanh | `ProjectReference` |
| Thư viện phiên bản hóa / bên thứ ba | `PackageReference` |
| Test cần internals của SUT | `ProjectReference` + `InternalsVisibleTo` |

> Tránh vừa project vừa package **cùng assembly** trong một graph.

---

## 5. NuGet restore

Restore tải package về cache (`%USERPROFILE%\.nuget\packages`) và ghi artifacts trong `obj/`.

```bash
dotnet restore
dotnet build              # thường restore ngầm
dotnet build --no-restore
dotnet list package --outdated
dotnet list package --vulnerable
dotnet nuget why Serilog
```

`nuget.config` — nên `<clear />` rồi khai báo source tường minh:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

- CI: cache global packages.  
- Restore giải dependency graph (direct + transitive); conflict theo quy tắc NuGet nearest-wins / unify.

---

## 6. Central Package Management (CPM)

**Tooling:** NuGet 6.2+ / .NET SDK 6.0.300+ (ổn định trên .NET 10).

`Directory.Packages.props` (root repo):

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
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

- Override: `VersionOverride="…"`. Opt-out project: `ManagePackageVersionsCentrally=false`.  
- Chỉ auto-import **một** `Directory.Packages.props` gần nhất.  
- Scaffold: `dotnet new packagesprops`.

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

CI: `dotnet tool restore` trước khi gọi tool. Local tools = cùng version cho cả team.

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
- Strong-name: cần `PublicKey=` trong thuộc tính.  
- Phổ biến cho unit test — đừng phá encapsulation giữa layer production.

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

Mỗi TFM nhân chi phí CI/test — chỉ multi-target khi thực sự cần runtime cũ.

---

## 10. `Directory.Build.props` / `.targets`

MSBuild tự import theo cây thư mục:

| File | Vai trò |
|---|---|
| `Directory.Build.props` | Property/item **đầu** (trước csproj) |
| `Directory.Build.targets` | Target **cuối** (sau csproj) |
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

File-based apps (.NET 10) cũng thừa hưởng các file này — cẩn thận khi đặt script cạnh monorepo lớn.

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
| Framework-dependent (mặc định) | Cần runtime .NET trên máy đích |
| `--self-contained` | Kèm runtime; artifact lớn hơn |
| `PublishSingleFile` | Một file (có thể extract native) |
| `PublishAot` | Native AOT (mục 12) |
| `PublishTrimmed` | Cắt IL — rủi ro reflection |

---

## 12. Native AOT (`PublishAot`) — overview & pitfalls

**Ổn định từ .NET 7+;** .NET 10 mở rộng compatibility.

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r win-x64
```

**Lợi ích:** startup nhanh, footprint nhỏ, binary native self-contained.

**Pitfalls:**

- Reflection/dynamic hạn chế — warning trim/AOT (`IL2026`, `IL3050`…).  
- Không phải mọi library AOT-friendly.  
- Compile lâu; cần toolchain native (MSVC / clang…).  
- `Assembly.Load*`, emit, serializer reflection-heavy → source generator / annotation.  
- File-based apps (.NET 10) **bật `PublishAot` mặc định** khi publish — tắt: `#:property PublishAot=false`.

```csharp
[RequiresUnreferencedCode("Uses reflection")]
public static void Risky() { /* ... */ }
```

> CLI tool / cold-start → AOT hấp dẫn. Plugin động / reflection nặng → cân nhắc JIT.

---

## 13. File-based apps — `dotnet run app.cs` (.NET 10)

**Áp dụng:** .NET **10 SDK+** — chạy/publish một `.cs` không cần `.csproj`.

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
dotnet publish app.cs
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

**Mặc định:** `PublishAot=true`, `PackAsTool=true`. Tôn trọng `Directory.Build.props` / CPM / `nuget.config` / `global.json`.  
Nếu có `.csproj` trong cwd, `dotnet run app.cs` không `--file` có thể coi `app.cs` là **argument** của project.

Dùng cho script/utility/prototype; app lớn → `dotnet project convert`.

---

## 14. Best practices & checklist

- Pin SDK bằng `global.json` trên CI/team.  
- Chuẩn hóa `Nullable` + `ImplicitUsings` + `TreatWarningsAsErrors` ở `Directory.Build.props`.  
- Solution lớn → CPM. Tooling team → local tools.  
- Phân biệt `ProjectReference` (nội bộ) vs package (biên giới version).  
- Publish: chọn FDD / self-contained / single-file / AOT có chủ đích; đọc warning AOT/trim trước khi ship.  
- Không nhét file-based app vào cây project nếu sợ “nhiễm” props.

```text
[ ] TargetFramework = net10.0
[ ] Nullable + ImplicitUsings
[ ] Directory.Build.props + nuget.config rõ nguồn
[ ] CPM nếu ≥ vài project
[ ] InternalsVisibleTo cho test (nếu cần)
[ ] CI: restore → build → test → (publish)
```
