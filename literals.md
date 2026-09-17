# Literals

> **Baseline:** .NET **10** / C# **14**. Raw string / UTF-8 `u8`: C# 11. Escape `\e`: C# 13. Interpolated const: C# 10. Raw interpolation `$"""` / `$$"""`: C# 11.

**Literal** là giá trị viết trực tiếp trong mã nguồn (không thông qua biến hay biểu thức). C# hỗ trợ nhiều loại literal: **số** (nguyên & thực), **chuỗi/char**, **bool**, **`null`**, **`default`**, cùng các biến thể hiện đại như **dấu gạch dưới `_`** để phân tách chữ số, **nhị phân `0b`**, **raw string `""" ... """`**, **UTF-8 `"..."u8`**, và **string interpolation**.

Literal không “chạy” — compiler nhúng giá trị (hoặc handler nội suy) vào IL. Nhầm `const` với `static readonly`, hoặc `u8` với `string`, là pitfall phổ biến.

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
    - [5.3 Interpolated `$"..."` — handler, culture, alignment](#53-interpolated--handler-culture-alignment)
    - [5.4 Kết hợp `@$` hoặc `$@`](#54-kết-hợp--hoặc-)
  - [6. Raw string literal `"""` (C# 11) \& UTF-8 `"..."u8`](#6-raw-string-literal--c-11--utf-8-u8)
    - [6.1 Raw string `""" ... """` — indent \& quotes](#61-raw-string----indent--quotes)
    - [6.2 UTF-8 string literal `"..."u8` (C# 11)](#62-utf-8-string-literal-u8-c-11)
  - [7. Boolean \& `null` \& `default` literal](#7-boolean--null--default-literal)
  - [8. Hằng số (`const`) vs `readonly`](#8-hằng-số-const-vs-readonly)
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

Literal quá lớn cho mọi kiểu nguyên → lỗi compile. `int x = 3_000_000_000;` không vừa `int` — cần `U`/`L` hoặc kiểu đích `long`.

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

**Pitfall:** `float f = 0.1;` không compile (`0.1` là `double`); cần `0.1F`. `0.1 + 0.2 == 0.3` có thể **false** với `double`. Tiền tệ: luôn `M`.

`decimal` literal chính xác thập phân trong phạm vi 28–29 chữ số có nghĩa; overflow literal → lỗi compile. `float` literal lớn thành `Infinity` lúc compile? Thường overflow warning/error tùy giá trị. Đừng so `==` tiền tệ `double`.

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
**C# 13:** `\e` = ESC (U+001B) — hữu ích ANSI/VT.  
**Unicode**: `\uFFFF` (4 hex), `\xNN` (1–4 hex), `\U0000FFFF` (8 hex)

```csharp
char c1 = 'A';
char c2 = '\n';        // xuống dòng
char c3 = '\u03A9';    // Ω
char c4 = '\x263A';    // ☺
char esc = '\e';       // C# 13
```

> `\x` có độ dài **linh hoạt**; nên dùng `\u`/`\U` để **rõ ràng**. `\U` trong `char` phải nằm BMP; ngoài BMP dùng string `"\U0001F600"`.

---

## 5. Literal chuỗi (`string`) thường & verbatim `@` + interpolated `$`

### 5.1 Chuỗi thường (có escape)

```csharp
string s1 = "Hello\n\"World\"";
```

Interning: literal giống nhau có thể cùng instance (`ReferenceEquals`); **không** phụ thuộc intern cho logic `==` — `string ==` so nội dung.

### 5.2 Verbatim `@"..."` (không escape)

- Backslash & xuống dòng **giữ nguyên**; dấu `"` viết thành `""`.

```csharp
string path = @"C:\data\logs\app.txt";
string text = @"Line1
Line2 ""quoted""
Line3";
```

Verbatim **vẫn** xử lý `""` → `"`. Không giải `\n` (hai ký tự `\` và `n`). Regex/Windows path hợp verbatim; JSON/XML nhiều dấu `"` → cân nhắc **raw string**.

### 5.3 Interpolated `$"..."` — handler, culture, alignment

- Chèn biểu thức trong `{ ... }`. Có **căn lề** `,width` và **định dạng** `:format`.

```csharp
var name = "Alice";
var score = 1234.5;
string msg = $"Hi {name,-10} | {score,8:0.00}"; // căn trái 10, căn phải 8, định dạng
```

**WHY `$` thay `string.Format`:** compile-time kiểm tra hole; C# 10+ dùng `DefaultInterpolatedStringHandler` (ref struct) — ít cấp phát hơn `Format` khi nối vài hole (stack buffer, rồi `ToString`).

**Semantics:**

1. Mỗi hole đánh giá **một lần**, trái → phải.
2. `{expr,alignment:format}` — `alignment` dương: pad trái (căn phải); âm: pad phải (căn trái).
3. Format specifier đi tới `IFormattable` / handler — **culture**: interpolation mặc định dùng `CultureInfo.CurrentCulture` (khác `nameof` / ordinal).

```csharp
CultureInfo.CurrentCulture = new CultureInfo("vi-VN");
var n = 1234.5;
Console.WriteLine($"{n:N2}"); // "1.234,50" (vi) — không ổn định cho log/wire

// Invariant — API, file, JSON:
string wire = string.Create(CultureInfo.InvariantCulture, $"{n:N2}");
// hoặc FormattableString:
FormattableString fs = $"{n:N2}";
string inv = fs.ToString(CultureInfo.InvariantCulture);
```

**`FormattableString`:** gán `$"..."` vào `FormattableString` **không** gọi `ToString` ngay — giữ format + args (SQL/log structured, `Execute($"...{id}")` pattern). Gán vào `string` thì nội suy ngay.

```csharp
string s = $"user={id}";                 // handler → string
FormattableString f = $"user={id}";      // hole chưa ghép
object[] args = f.GetArguments();
```

**Escape hole:** `{{` / `}}` → `{` / `}`. C# 11 raw: tăng `$` để đổi delimiter (`$$""" {not a hole} {{expr}} """` — một `{` literal, `{{expr}}` là hole).

```csharp
var jsonLike = $"{{ \"n\": {n} }}"; // { "n": 1 }
var raw = $$"""
    { "n": {{n}} }
    """;
```

**Pitfall:** side-effect trong hole (`$"{++i} {++i}"`) — thứ tự xác định nhưng khó đọc. Exception trong hole → cả interpolation fail. `$"..."u8` → UTF-8 interpolated span (C# 11) — format hạn chế hơn string.

**Interpolated **const** (C# 10):** mọi hole là `const` → kết quả `const string` — xem §8.

**Handler & alloc:** C# 10 `DefaultInterpolatedStringHandler` — vài hole nhỏ có thể không alloc `string.Format` array. Vòng lặp cực nóng: `string.Create(length, state, span => ...)` hoặc `IBufferWriter`. Logging: `LoggerExtensions` nhận hole **không** gọi `ToString` nếu level tắt (template), khác `$"..."` **luôn** materialize nếu bạn truyền `string` đã nội suy.

```csharp
logger.LogInformation($"user={user}");          // nội suy ngay, kể cả Information tắt
logger.LogInformation("user={User}", user);     // structured — tốt hơn
```

### 5.4 Kết hợp `@$` hoặc `$@`

```csharp
string path2 = $@"C:\Users\{Environment.UserName}\docs";
```

> **Interpolated string** có thể là `FormattableString` khi gán vào loại đó. Thứ tự `@$` / `$@` tương đương.

---

## 6. Raw string literal `"""` (C# 11) & UTF-8 `"..."u8`

### 6.1 Raw string `""" ... """` — indent & quotes

- Không cần escape **backslash** hay `"`; giữ nguyên **xuống dòng & thụt lề** (cắt thụt lề chung).
- Hỗ trợ **interpolation**: `$""" ... {expr} ... """`.
- Để chèn dấu `{`/`}` *nguyên văn* trong chuỗi có nội suy, dùng **`{{`** hoặc **`}}`**, hoặc tăng số `$` (C# 11 `$$"""`).

**Quy tắc indent (WHY tránh “thụt lề file nguồn lọt vào chuỗi”):**

1. Dòng đóng `"""` quyết định **cột cắt**. Mọi dòng nội dung phải thụt ≥ cột đó (trừ dòng trống).
2. Dòng mở `"""` thường **một mình**; nội dung bắt đầu dòng sau.
3. Newline sau `"""` mở **không** nằm trong giá trị; newline trước `"""` đóng **không** nằm trong giá trị (single-line raw thì khác).

```csharp
var json = """
{
  "name": "Alice",
  "path": "C:\\data\\files\\a.txt"
}
""";
// json bắt đầu bằng `{`, không có indent thừa từ source

var who = "Bob";
var greet = $"""
Hello, {who}!
This is a "raw" string with no escaping.
{{This brace is literal}}.
""";
```

**Nhiều dấu `"`:** nếu nội dung chứa `"""`, mở/đóng bằng 4+ dấu:

```csharp
var embedded = """"
    code = """hello""";
    """";
```

Số `$` ≥ số `{` liên tiếp muốn coi là literal. `$$""" {x} {{x}} """` → `{x}` literal, hole `x`.

**Pitfall:** `"""` đóng lệch cột → CS8997/8999 (indent). Trộn tab/space trong indent raw → lỗi. Raw **không** biến `""` thành `"` (đó là verbatim). JSON chứa `"""` hiếm — tăng quote count.

### 6.2 UTF-8 string literal `"..."u8` (C# 11)

- Thêm suffix **`u8`** để nhận **`ReadOnlySpan<byte>`** (UTF-8). Rất hữu ích cho **I/O/Protocol**.

```csharp
ReadOnlySpan<byte> bytes = "PING\r\n"u8;
```

> Với **raw + UTF-8**: `"""..."""u8` cũng hợp lệ.

**WHY không `Encoding.UTF8.GetBytes("PING")`:** `u8` là literal — compiler nhúng byte UTF-8, **không** cấp phát `string` rồi encode lúc runtime; kiểu là `ReadOnlySpan<byte>` (ref struct) trỏ data trong assembly (hoặc tương đương).

**Semantics & ràng buộc:**

| | `"..."u8` | `string` |
|---|---|---|
| Kiểu | `ReadOnlySpan<byte>` | `string` (UTF-16) |
| Heap string | Không (span vào data) | Có intern/literal |
| Gán `string s = "x"u8` | **Lỗi** | — |
| Sống qua `async`/`return string` | Span không store field/`async` dễ dàng | OK |
| `+` nối | Không như string | Có |

```csharp
ReadOnlySpan<byte> ping = "PING\r\n"u8;
socket.Send(ping);

// SAI
// string s = "PING"u8;
// byte[] a = "PING"u8; // không implicit sang byte[] — dùng .ToArray() nếu cần copy

byte[] owned = "PING\r\n"u8.ToArray(); // copy khi phải store
```

**Pitfall lifetime:** `u8` span trỏ data tĩnh — an toàn hơn `stackalloc`. **Không** trả `Span` từ method rồi dùng sau khi “xong” nếu bạn `ToArray` không? Data literal sống cùng assembly — `return "ok"u8;` *có thể* hợp lệ vì payload tĩnh, nhưng kiểu trả `ReadOnlySpan<byte>` từ method public thường khó (ref struct). Public API: `ReadOnlyMemory<byte>` / `byte[]` / `Utf8String` pattern.

Interpolation UTF-8: `$"x={n}"u8` — handler UTF-8; format culture vẫn là mối quan tâm. Chỉ dùng khi hole format được thành UTF-8.

So sánh protocol:

```csharp
if (buffer.StartsWith("HTTP/1.1"u8)) { }
// vs Encoding.UTF8.GetBytes mỗi lần — cấp phát
```

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
- `null` chỉ gán được cho **reference type**, **nullable value type** (`int?`), pointer, và generic với ràng buộc phù hợp — **không** `int x = null`.
- `true`/`false` là `bool` (không implicit sang `int` như C).

**`null` vs `default`:**

| Ngữ cảnh | `null` | `default` / `default(T)` |
|---|---|---|
| `string?` | `null` | `null` |
| `int` | lỗi | `0` |
| `int?` | `null` | `null` |
| `DateTime` | lỗi | `DateTime.MinValue` (default struct) |
| Generic `T` | chỉ khi `T` nhận null | Luôn được — `default!` khi NRT |

```csharp
static T? FirstOrDefault<T>(T[] a)
    => a.Length == 0 ? default : a[0];

int n = default;           // 0 — không phải “chưa gán”
int? m = default;          // null
int? m2 = null;            // rõ ý null hơn default cho T?
```

**Pitfall NRT:** `string s = default;` cảnh báo — `default` của reference là `null`. Ưu tiên `null` khi ý đồ là null; `default` khi generic / “zero của T”. `default` **không** gọi ctor; struct field zeroed.

`new()` (C# 9) khác `default`: `new DateTime()` cũng default, nhưng `new List<int>()` là instance rỗng — `default(List<int>)` là **null**.

---

## 8. Hằng số (`const`) vs `readonly`

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

**WHY phân biệt `const` / `static readonly`:** `const` được **inline vào chỗ dùng** lúc compile (kể cả assembly khác). Đổi giá trị `const` public → **phải compile lại mọi consumer**. `static readonly` đọc field lúc runtime — đổi library, consumer cũ nhận giá trị mới khi load DLL mới (không cần recompile chỉ vì constant folding).

| | `const` | `static readonly` | instance `readonly` |
|---|---|---|---|
| Khi gán | Compile-time | Static ctor / khai báo | Ctor / khai báo field |
| Kiểu | Primitive, `string`, `null` | Mọi kiểu | Mọi kiểu |
| Cross-assembly | Copy giá trị vào IL caller | Tham chiếu field | — |
| `DateTime`, `array`, object | Không | Có (`readonly` array **vẫn mutate phần tử**) | Có |
| Generic / computed | Không | Có | Có |

```csharp
public static class Limits
{
    public const int MaxNameLength = 128;              // ổn — thật sự không đổi bao giờ
    public static readonly TimeSpan Timeout = TimeSpan.FromSeconds(30);
    public static readonly DateTime Epoch = new(2020, 1, 1, 0, 0, 0, DateTimeKind.Utc);
    public static readonly string[] Roles = { "admin", "user" }; // phần tử vẫn gán được!
}

void M()
{
    const int local = 3;           // local const
    // readonly int x = 1;         // KHÔNG có local readonly (dùng const hoặc in/ref readonly)
}
```

**Local:** chỉ `const`, không `readonly` (field-only). Truyền không đổi: `in` / `ref readonly`.

**Pitfall:** `public const decimal Vat = 0.1M` trong shared library — đổi thuế suất mà plugin không rebuild → vẫn 0.1. Dùng `static readonly` cho “hằng” có thể chỉnh giữa các version.

`readonly struct` / `readonly` field khác `const`: ngăn gán lại *biến/field*, không phải literal. `readonly` trên field reference: không gán lại *tham chiếu*, object bên trong vẫn mutate.

```csharp
readonly List<int> _items = new();
_items.Add(1);     // OK
// _items = new(); // lỗi
```

**Interned string `const`:** `const string A = "x"; const string B = "x";` có thể cùng intern. Đổi `const` public trong lib A, app B **không rebuild** → B vẫn mang bản cũ trong IL. Versioning protocol/magic number: `static readonly` hoặc file config, không `public const int Protocol = 3` nếu có thể tăng.

`enum` member là hằng compile-time (dùng trong `const`/`case`). `static readonly` enum field thì không dùng trong `case` label.

---

## 9. Target-typed literals & gợi ý kiểu

Nhiều literal **suy kiểu** theo ngữ cảnh đích:

```csharp
nint ni = 123;            // suy kiểu về native-int theo biến bên trái
Half h = (Half)1.5;       // hoặc target-typed qua ctor/Parse nếu hỗ trợ
TimeSpan t = default;     // default literal target-typed
```

Với số nguyên lớn, nếu không có suffix và **không vừa `int`**, compiler sẽ cân nhắc các kiểu lớn hơn **theo ngữ cảnh** (biểu thức, phép toán, gán). Dùng **suffix** để tránh mơ hồ.

Collection expression `[1, 2, 3]` (C# 12) cũng target-typed — xem [collections-generics.md](collections-generics.md).

---

### Bảng tóm tắt suffix & mặc định

| Nhóm | Mặc định | Suffix | Ghi chú |
|---|---|---|---|
| Integral | `int` | `U`→`uint`, `L`→`long`, `UL`→`ulong` | Hex `0x`, Bin `0b`, `_` |
| Floating | `double` | `F`→`float`, `D`→`double`, `M`→`decimal` | `e/E` cho float/double; **không** cho decimal |
| String | `string` | `$` (interpolated), `@` (verbatim), `"""` (raw), `u8` (UTF-8) | Có thể kết hợp `$@`/`@$`, `$"""`/`"""u8` |
| Char | `char` | — | Escape: `\n`, `\e` (C# 13), `\uFFFF`, `\xNN`, `\UXXXXXXXX` |
| Boolean | `bool` | — | `true`/`false` |
| Null/Default | — | — | `null`, `default`/`default(T)` |
