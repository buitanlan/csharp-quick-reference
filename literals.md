# Literals

> **Baseline:** .NET **10** / C# **14**. Raw string / UTF-8 `u8`: C# 11. Escape `\e`: C# 13.

**Literal** là giá trị viết trực tiếp trong mã nguồn (không thông qua biến hay biểu thức). C# hỗ trợ nhiều loại literal: **số** (nguyên & thực), **chuỗi/char**, **bool**, **`null`**, **`default`**, cùng các biến thể hiện đại như **dấu gạch dưới `_`** để phân tách chữ số, **nhị phân `0b`**, **raw string `""" ... """`**, **UTF-8 `"..."u8`**, và **string interpolation**.

---

## Mục lục

- [Literals](#literals)
  - [Mục lục](#mục-lục)
  - [1. Literal số nguyên (integral)](#1-literal-số-nguyên-integral)
  - [2. Literal số thực (floating-point \& decimal)](#2-literal-số-thực-floating-point--decimal)
  - [3. Dấu gạch dưới `_` trong số](#3-dấu-gạch-dưới-_-trong-số)
  - [4. Literal ký tự (`char`) và escape sequences](#4-literal-ký-tự-char-và-escape-sequences)
  - [5. Literal chuỗi (`string`) thường \& verbatim `@` + interpolated `$`](#5-literal-chuỗi-string-thường--verbatim---interpolated-)
    - [5.1 Chuỗi thường (có escape)](#51-chuỗi-thường-có-escape)
    - [5.2 Verbatim `@"..."` (không escape)](#52-verbatim--không-escape)
    - [5.3 Interpolated `$"..."` (nội suy)](#53-interpolated--nội-suy)
    - [5.4 Kết hợp `@$` hoặc `$@`](#54-kết-hợp--hoặc-)
  - [6. Raw string literal `"""` (C# 11) \& UTF-8 `"..."u8`](#6-raw-string-literal--c-11--utf-8-u8)
    - [6.1 Raw string `""" ... """`](#61-raw-string---)
    - [6.2 UTF-8 string literal `"..."u8` (C# 11)](#62-utf-8-string-literal-u8-c-11)
  - [7. Boolean \& `null` \& `default` literal](#7-boolean--null--default-literal)
  - [8. Hằng số (`const`) và literal](#8-hằng-số-const-và-literal)
  - [9. Target-typed literals \& gợi ý kiểu](#9-target-typed-literals--gợi-ý-kiểu)
    - [Bảng tóm tắt suffix \& mặc định](#bảng-tóm-tắt-suffix--mặc-định)

---

## 1. Literal số nguyên (integral)

- **Cơ số**: thập phân (mặc định), **hex `0x`**, **nhị phân `0b`**.  
- **Kiểu mặc định**: `int` nếu vừa; nếu không vừa → `uint` / `long` / `ulong` theo **suffix** hoặc **ngữ cảnh**.
- **Suffix** để chỉ kiểu rõ ràng (không phân biệt thứ tự chữ cái, nên viết hoa cho dễ đọc):  
  - `U` → `uint`  
  - `L` → `long`  
  - `UL` (hoặc `LU`) → `ulong`
- Có thể dùng dấu gạch dưới để giúp dễ đọc hơn, dấu gạch dưới không ảnh hưởng đến giá trị.

```csharp
int    a = 42;        // thập phân
int    b = 0x2A;      // hex
int    c = 0b_0010_1010; // nhị phân, có '_'

long   big = 9_000_000_000L; // L → long
uint   u   = 4000000000U;    // U → uint
ulong  ul  = 18_000_000_000UL;
```

> **Không có** suffix dành riêng cho `nint`/`nuint`. Dùng **gợi ý kiểu (target-typed)** hoặc cast.

---

## 2. Literal số thực (floating-point & decimal)

- **Mặc định** cho số thực: `double`.  
- **Suffix**: `F/f` → `float`, `D/d` → `double`, `M/m` → `decimal`.  
- Hỗ trợ **kí pháp khoa học** cho `float`/`double`: `1.23e-2`. `decimal` **không** dùng `e`.

```csharp
double dx = 3.14;       // mặc định double
float  fx = 3.14F;      // float
decimal money = 123_456.78M; // decimal cho tài chính

double e  = 1.2e3;      // 1200
// decimal không có e/E:
decimal dm = 1_200.00M;
```

> `float`/`double` theo **IEEE 754** (nhị phân) → có sai số; `decimal` theo **thập phân** → phù hợp tiền tệ.

---

## 3. Dấu gạch dưới `_` trong số

- Dùng để **nhóm chữ số** cho dễ đọc (C# 7+).  
- Hợp lệ ở **giữa** chữ số, ở **phần nguyên, phần thập phân, và số mũ** (với float/double).  
- **Không** đặt ở đầu/cuối, ngay sau prefix `0x/0b`, trước/ sau dấu chấm thập phân, hay trước suffix.

```csharp
int    n  = 1_000_000;
double pi = 3.1415_9265;
double ee = 1_23e4_5;   // 1.23 × 10^45
int    hx = 0xDEAD_BEEF;
```

---

## 4. Literal ký tự (`char`) và escape sequences

`char` là **một đơn vị UTF-16** (0..0xFFFF). Kí tự > 0xFFFF cần **chuỗi** (hoặc `System.Rune`).

**Escape chuẩn**: `\'` `\"` `\\` `\0` `\a` `\b` `\f` `\n` `\r` `\t` `\v`  
**Unicode**: `\uFFFF` (4 hex), `\xNN` (1–4 hex), `\U0000FFFF` (8 hex)

```csharp
char c1 = 'A';
char c2 = '\n';        // xuống dòng
char c3 = '\u03A9';    // Ω
char c4 = '\x263A';    // ☺
```

> `\x` có độ dài **linh hoạt**; nên dùng `\u`/`\U` để **rõ ràng**.

---

## 5. Literal chuỗi (`string`) thường & verbatim `@` + interpolated `$`

### 5.1 Chuỗi thường (có escape)

```csharp
string s1 = "Hello\n\"World\"";
```

### 5.2 Verbatim `@"..."` (không escape)

- Backslash & xuống dòng **giữ nguyên**; dấu `"` viết thành `""`.

```csharp
string path = @"C:\data\logs\app.txt";
string text = @"Line1
Line2 ""quoted""
Line3";
```

### 5.3 Interpolated `$"..."` (nội suy)

- Chèn biểu thức trong `{ ... }`. Có **căn lề** `,width` và **định dạng** `:format`.

```csharp
var name = "Alice";
var score = 1234.5;
string msg = $"Hi {name,-10} | {score,8:0.00}"; // căn trái 10, căn phải 8, định dạng
```

### 5.4 Kết hợp `@$` hoặc `$@`

```csharp
string path2 = $@"C:\Users\{Environment.UserName}\docs";
```

> **Interpolated string** có thể là `FormattableString` khi gán vào loại đó.

---

## 6. Raw string literal `"""` (C# 11) & UTF-8 `"..."u8`

### 6.1 Raw string `""" ... """`

- Không cần escape **backslash** hay `"`; giữ nguyên **xuống dòng & thụt lề** (cắt thụt lề chung).  
- Hỗ trợ **interpolation**: `$""" ... {expr} ... """`.  
- Để chèn dấu `{`/`}` *nguyên văn* trong chuỗi có nội suy, dùng **`{{`** hoặc **`}}`**.

```csharp
var json = """
{
  "name": "Alice",
  "path": "C:\\data\\files\\a.txt"
}
""";

var who = "Bob";
var greet = $"""
Hello, {who}!
This is a "raw" string with no escaping.
{{This brace is literal}}.
""";
```

> Bạn có thể dùng **nhiều dấu `"`** mở/đóng khi cần nhúng chuỗi có `"""` bên trong.

### 6.2 UTF-8 string literal `"..."u8` (C# 11)

- Thêm suffix **`u8`** để nhận **`ReadOnlySpan<byte>`** (UTF-8). Rất hữu ích cho **I/O/Protocol**.

```csharp
ReadOnlySpan<byte> bytes = "PING\r\n"u8;
```

> Với **raw + UTF-8**: `"""..."""u8` cũng hợp lệ.

---

## 7. Boolean & `null` & `default` literal

```csharp
bool ok = true;
bool fail = false;

string? none = null;

// default literal (C# 7.1+)
int    x = default;       // 0
string s = default;       // null
var p = default(DateTime); // 01/01/0001 00:00:00
```

- `default` **target-typed** theo biến/kiểu bên trái. `default(T)` luôn tường minh.

---

## 8. Hằng số (`const`) và literal

- `const` chỉ nhận **literal/hằng compile-time**: số, `char`, `bool`, `string`, `null`, `decimal`…

```csharp
const int PORT = 5432;
const string AppName = "MyApp";
const decimal Vat = 0.10M;
```

- **Interpolated const string (C# 10+)**: cho phép nếu **mọi thành phần là hằng**.

```csharp
const string Vendor = "ACME";
const string FullName = $"{Vendor}-Service"; // OK từ C# 10
```

> Không thể `const` với `DateTime.Now`/`Guid.NewGuid()`… vì **không** là hằng compile-time.

---

## 9. Target-typed literals & gợi ý kiểu

Nhiều literal **suy kiểu** theo ngữ cảnh đích:

```csharp
nint ni = 123;            // suy kiểu về native-int theo biến bên trái
Half h = (Half)1.5;       // hoặc target-typed qua ctor/Parse nếu hỗ trợ
TimeSpan t = default;     // default literal target-typed
```

Với số nguyên lớn, nếu không có suffix và **không vừa `int`**, compiler sẽ cân nhắc các kiểu lớn hơn **theo ngữ cảnh** (biểu thức, phép toán, gán). Dùng **suffix** để tránh mơ hồ.

---

### Bảng tóm tắt suffix & mặc định

| Nhóm | Mặc định | Suffix | Ghi chú |
|---|---|---|---|
| Integral | `int` | `U`→`uint`, `L`→`long`, `UL`→`ulong` | Hex `0x`, Bin `0b`, `_` |
| Floating | `double` | `F`→`float`, `D`→`double`, `M`→`decimal` | `e/E` cho float/double; **không** cho decimal |
| String | `string` | `$` (interpolated), `@` (verbatim), `"""` (raw), `u8` (UTF-8) | Có thể kết hợp `$@`/`@$`, `$"""`/`"""u8` |
| Char | `char` | — | Escape: `\n`, `\uFFFF`, `\xNN`, `\UXXXXXXXX` |
| Boolean | `bool` | — | `true`/`false` |
| Null/Default | — | — | `null`, `default`/`default(T)` |
