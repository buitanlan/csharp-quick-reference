# Keywords

> **Baseline:** .NET **10** / C# **14**. `extension` / `field`: C# 14. `union` / `closed`: **C# 15 preview**.  
> Mục 1–80: reserved + vài contextual đã tách thành mục. Mục 81: bảng contextual còn lại — gồm **`record`**, **`async`/`await`**, **`yield`**, **`var`**, **`nameof`**.

C# phân token thành vài lớp — **không** phải mọi chữ “keyword” trong docs đều cấm dùng làm tên biến:

| Lớp | Ý nghĩa | Ví dụ |
|-----|---------|--------|
| **Reserved** | Keyword mọi ngữ cảnh; không đặt identifier (trừ `@int`) | `class`, `if`, `void`, `int`, `return` |
| **Contextual** | Chỉ keyword ở vị trí nhất định; chỗ khác là identifier hợp lệ | `async`, `await`, `yield`, `var`, `record`, `when`, `where`, `get`, `from` |
| **Preprocessor** | Chỉ thị compiler, **không** thuộc grammar biểu thức C#; viết `#` đầu dòng | `#if`, `#nullable`, `#region`, `#:` (file-based, C# 14) — [preprocessor-directives.md](preprocessor-directives.md) |

Không nhầm ba lớp:

| Viết | Là gì | Không phải |
|------|--------|------------|
| `if (x)` | keyword reserved | `#if` |
| `await foo` | contextual | method tên `await` vẫn được (đừng) |
| `#:package X@1` | preprocessor file-based C# 14 | `using` NuGet trong C# |
| `record R(...)` | contextual | reserved như `class` |

`@` escape identifier trùng reserved (`int @class = 1`). Contextual (`file`, `from`) thường không cần `@` trừ khi đúng slot keyword. `from` là tên biến hợp lệ trong method bình thường; trong `from x in xs` thì là query. `async` đặt tên biến được, nhưng **không** nên — dễ đọc nhầm.

Preprocessor **không** xuất hiện trong IL như keyword: `#if DEBUG` cắt source trước compile; `#:` (C# 14) chỉ file-based SDK đọc, không phải `if` runtime. Trộn `#if` với TLS/`Main` để “chọn entry” rất rối — dùng `StartupObject` / tách project.

Gợi ý tra: reserved → mục 1–78 (+ `union` 79 preview). Contextual “lớn” đã tách: `extension`, `field`, `closed`. Còn lại (`record`, `async`, `await`, `yield`, `var`, `nameof`, LINQ, accessor) → **§81**. Pitfall/when nằm ở **Ghi chú**, không lặp lại cả topic `async.md`.

Trang này là **mục lục + pitfall ngắn**; semantics đầy đủ ở topic (`statements`, `oop`, `async`, `linq`, …) — không phải changelog C# 14/15. Đọc keyword mỏng → Ghi chú; contextual không có mục riêng → **§81**.

---

## 1. `abstract`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:**  
  - Đánh dấu class/member trừu tượng, không có triển khai đầy đủ.  
  - Class `abstract` không thể `new`.  
  - Member `abstract` bắt buộc phải được `override` trong lớp con.

**Ví dụ:**

```csharp
public abstract class Shape
{
    public abstract double Area();

    public virtual void Print()
        => Console.WriteLine($"Area = {Area()}");
}

public sealed class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius) => Radius = radius;

    public override double Area() => Math.PI * Radius * Radius;
}
```

**Ghi chú:**  
Thường dùng khi muốn định nghĩa “hợp đồng + một phần behavior chung” cho một nhóm class.  
`abstract` member không có body (trừ default interface — khác). Class có abstract member phải `abstract`. Không `new` abstract class; `sealed` + `abstract` cấm. Factory/`Activator` trên abstract → runtime fail.

---

## 2. `as`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Cast an toàn giữa reference type / nullable value type. Thất bại ⇒ `null`, không ném exception.

**Ví dụ:**

```csharp
object obj = "hello";

string? s = obj as string;        // "hello"
FileStream? fs = obj as FileStream; // null

if (s != null)
{
    Console.WriteLine(s.ToUpper());
}
```

**Ghi chú:**  
Luôn nhớ kiểm tra `null` sau khi dùng `as`. Nếu muốn lỗi rõ ràng hơn, dùng cast thường `(T)obj`.  
`as` chỉ cho reference / `Nullable<T>` — không `as` sang `int`. Pattern `is T t` vừa test vừa bind, thường sạch hơn `as` + if. `as` không chạy user-defined conversion (khác cast).

---

## 3. `base`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Dùng trong class dẫn xuất để gọi ctor hoặc member của base class (đặc biệt trong `override`).

**Ví dụ:**

```csharp
public class Animal
{
    public string Name { get; }

    public Animal(string name) => Name = name;

    public virtual void Speak()
        => Console.WriteLine($"{Name} makes a sound");
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }

    public override void Speak()
    {
        base.Speak(); // gọi behavior chung
        Console.WriteLine($"{Name} barks");
    }
}
```

**Ghi chú:**  
Không dùng được trong `struct`. Nếu base không có ctor mặc định, lớp con phải gọi `base(...)`.  
`base.Method()` trong `override` tránh recursion vô hạn khi quên. Primary ctor + `base(...)`: [oop.md](oop.md). Không `base` trong static.

---

## 4. `bool`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Kiểu Boolean với hai giá trị `true` / `false`. Không cho dùng int thay bool như C/C++.

**Ví dụ:**

```csharp
bool isActive = true;

if (isActive)
{
    Console.WriteLine("Enabled");
}
else
{
    Console.WriteLine("Disabled");
}
```

**Ghi chú:**  
Không dùng `0`/`1` thay `bool` như C. `if (flag == true)` thừa — viết `if (flag)`. `bool?` cho tri-state (unset); đừng nhầm `default(bool)` (`false`) với “chưa gán”.

---

## 5. `break`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Thoát khỏi vòng lặp (`for`, `foreach`, `while`, `do`) hoặc `switch` ngay lập tức.

**Ví dụ:**

```csharp
for (int i = 0; i < 100; i++)
{
    if (i == 10)
        break;
}

switch (statusCode)
{
    case 200:
        Console.WriteLine("OK");
        break;
    default:
        Console.WriteLine("Other");
        break;
}
```

**Ghi chú:**  
Quá nhiều `break`/`continue` trong cùng một vòng lặp có thể làm flow khó đọc.  
**C# 15 preview:** `break outer;` / `continue outer;` trên vòng có nhãn — xem [statements.md §8.1.1](statements.md#811-labeled-break--continue-c-15-preview).  
`break` trong `switch` không thoát vòng bao ngoài — đó là lý do labeled break preview. `break` không dùng trong `if`.

---

## 6. `byte`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên không dấu 8-bit (`0..255`), thường dùng cho buffer, stream, dữ liệu nhị phân.

**Ví dụ:**

```csharp
byte b = 255;
byte[] buffer = new byte[1024];
```

**Ghi chú:**  
Phép toán trên `byte` trả về `int`, cần cast ngược nếu muốn gán lại vào `byte`.  
`byte` không dấu 0–255; `sbyte` có dấu — nhầm interop. `byte[]` ≠ `Span<byte>` / `Memory<byte>` — API hiện đại ưu tiên span. Overflow 255+1 wrap nếu unchecked.

---

## 7. `case`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Định nghĩa nhánh trong `switch` (statement).

**Ví dụ:**

```csharp
switch (status)
{
    case 200:
        Console.WriteLine("OK");
        break;
    case 404:
        Console.WriteLine("Not Found");
        break;
    default:
        Console.WriteLine("Unknown");
        break;
}
```

**Ghi chú:**  
Trong `switch` cũ, mỗi `case` phải kết thúc bằng `break`/`return`/`goto`… Trên `switch expression` (C# 8+) không dùng `case` kiểu này nữa.  
`case` rỗng xếp chồng = OR (`case 1: case 2:`). Pattern `case int n when n > 0:` — `when` contextual. Không fall-through có lệnh như C.

---

## 8. `catch`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Bắt exception được ném từ khối `try`.

**Ví dụ:**

```csharp
try
{
    File.ReadAllText("data.txt");
}
catch (FileNotFoundException ex)
{
    Console.WriteLine("File không tồn tại");
}
catch (IOException ex)
{
    Console.WriteLine("Lỗi IO khác");
}
catch (Exception ex)
{
    Console.WriteLine("Lỗi không xác định");
}
```

**Ghi chú:**  
Tránh `catch (Exception) { }` bỏ trống – rất khó debug. Nên log hoặc wrap thành exception có ý nghĩa cụ thể.  
Thứ tự: kiểu cụ thể trước, `Exception` sau. Filter `when` chạy **trước** khi coi là bắt — [exceptions.md](exceptions.md). Không `catch` `StackOverflowException` / fatal.

---

## 9. `char`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Ký tự Unicode 16-bit (`System.Char`). Một số ký tự (emoji, ký hiệu phức tạp) cần 2 `char` (surrogate pair).

**Ví dụ:**

```csharp
char c = 'A';
bool isLetter = char.IsLetter(c); // true
```

**Ghi chú:**  
Đừng giả định “1 ký tự người dùng = 1 `char`”; với Unicode phức tạp nên dùng API trên `string` hoặc `System.Text.Rune`.  
`char` là UTF-16 code unit. Literal `'A'`. `char.IsDigit` ≠ “chữ số mọi script” lúc parse `int`. Arithmetic `char` promote `int`.

---

## 10. `checked`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Bật kiểm tra overflow cho toán tử số học/ép kiểu integral. Overflow ⇒ ném `OverflowException`.

**Ví dụ:**

```csharp
int x = int.MaxValue;

checked
{
    int y = x + 1; // OverflowException
}
```

**Ghi chú:**  
Dùng ở chỗ cần đảm bảo không overflow (tài chính, số quan trọng). Có thể bật mặc định trong project và dùng `unchecked` cho các chỗ đặc biệt.  
`checked` expression `checked(x + y)` khác khối. `decimal` không dùng checked overflow như `int`. Constant overflow lúc compile vẫn lỗi kể cả unchecked context một số case.

---

## 11. `class`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khai báo **reference type** (class).

**Ví dụ:**

```csharp
public class Person
{
    public string Name { get; set; } = "";
    public int Age { get; set; }

    public void SayHello()
        => Console.WriteLine($"Hi, I'm {Name} ({Age})");
}
```

**Ghi chú:**  
Class là reference type → được cấp phát trên heap, truyền qua reference. Với type nhỏ, immutable, nhạy hiệu năng, cân nhắc `struct` hoặc `record struct`.  
`class` vs `record class`: record thêm equality/`with`. `sealed` mặc định khi không cần inherit. C# 15 `closed` preview: [oop.md](oop.md). `static class` không instance.

---

## 12. `const`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khai báo hằng compile-time. Giá trị được inline vào IL của caller.

**Ví dụ:**

```csharp
public const double Pi = 3.14159265358979;
private const string AppName = "MyApp";
```

**Ghi chú:**  
Thay đổi giá trị `public const` trong library không tự update cho code client đã build; cần rebuild client. Với giá trị có thể thay đổi, dùng `static readonly`.  
Chỉ cho phép giá trị compile-time (`int`, `string`, `enum`…). Không `const DateTime`. `const` local trong method cũng inline.

---

## 13. `continue`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Bỏ phần còn lại của vòng lặp hiện tại, nhảy tới lần lặp tiếp theo.

**Ví dụ:**

```csharp
foreach (var item in items)
{
    if (item == null) continue;
    Process(item);
}
```

**Ghi chú:** **C# 15 preview** — `continue outer;` với nhãn vòng ngoài: [statements.md §8.1.1](statements.md#811-labeled-break--continue-c-15-preview).  
`continue` chỉ vòng **đang chạy**, không phải `switch`. Trong `foreach` nhảy tới phần tử kế. Lạm dụng `continue` + điều kiện phức = khó đọc hơn early-filter LINQ/`if` ngược.

---

## 14. `decimal`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số thập phân 128-bit, độ chính xác cao, rất phù hợp tài chính / tiền tệ.

**Ví dụ:**

```csharp
decimal price = 19.99m;
decimal qty   = 3;
decimal total = price * qty; // 59.97m
```

**Ghi chú:**  
Chậm hơn `double`; không lý tưởng cho tính toán khoa học nặng.  
Scale 28–29 chữ số thập phân; literal `m`. JSON/`double` round-trip có thể mất `decimal`. Không `NaN`. Arithmetic `decimal` không SIMD như `double`.

---

## 15. `default`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:**  
  - Trong `switch`: nhánh mặc định.  
  - Trong expression: `default(T)` hoặc `default` (C# 7.1+) ⇒ giá trị mặc định của kiểu (`0`, `false`, `null`, struct rỗng…).

**Ví dụ:**

```csharp
int x = default;      // 0
string? s = default;  // null

T Create<T>() => default!;
```

**Ghi chú:**  
`default` trên `switch` statement ≠ `default` expression. `default` của `string` / class là `null`; của `int` là `0` — hay nhầm với `FirstOrDefault`. Generic: `default!` chỉ tắt warning NRT, không tạo instance.

---

## 16. `delegate`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khai báo kiểu đại diện cho method (function pointer an toàn). Dùng cho callback, event handler, v.v.

**Ví dụ:**

```csharp
public delegate void LogHandler(string message);

public class Worker
{
    public LogHandler? Logger { get; set; }

    public void DoWork()
    {
        Logger?.Invoke("Working...");
    }
}
```

**Ghi chú:**  
Trong code hiện đại, thường dùng `Action<>`, `Func<>` thay vì tự khai báo delegate, trừ khi cần type public có tên rõ ràng.  
`delegate` multicast (`+`); `event` hạn chế truy cập. `unmanaged` function pointer (`delegate*`) khác — [memory-spans.md](memory-spans.md). Covariance/contravariance `in`/`out` trên generic delegate.

---

## 17. `do`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Bắt đầu vòng lặp `do { ... } while (cond)` – thân vòng lặp **chạy ít nhất 1 lần**.

**Ví dụ:**

```csharp
int i = 0;

do
{
    Console.WriteLine(i);
    i++;
} while (i < 3);
```

**Ghi chú:**  
Dùng khi body **phải** chạy trước khi biết điều kiện (prompt, đọc stream lần đầu). Loop vô hạn: `while (true)` phổ biến hơn `do` + `true`. Đừng quên `;` sau `while (...)`.

---

## 18. `double`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số thực 64-bit theo chuẩn IEEE 754; dùng nhiều cho tính toán khoa học, đồ họa, đo đạc.

**Ví dụ:**

```csharp
double x = 0.1 + 0.2;
Console.WriteLine(x); // 0.30000000000000004
```

**Ghi chú:**  
Luôn tồn tại sai số floating-point; khi so sánh nên dùng epsilon, không so sánh trực tiếp bằng `==` cho số thực.  
Literal không hậu tố = `double`. `NaN != NaN`; dùng `double.IsNaN`. Tiền tệ: `decimal`. `float`/`double` không `checked` overflow (ra `Infinity`).

---

## 19. `else`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Nhánh “ngược lại” của `if`.

**Ví dụ:**

```csharp
if (score >= 90)
    grade = "A";
else if (score >= 80)
    grade = "B";
else
    grade = "C";
```

**Ghi chú:**  
`else` gắn với `if` gần nhất — thiếu `{}` dễ “else dính nhầm”. C# không có `elif`; chuỗi `else if`. Nhánh rỗng: thường pattern `if` ngược hoặc early `return` sạch hơn.

---

## 20. `enum`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Định nghĩa kiểu liệt kê (enum) với underlying integral type (mặc định `int`).

**Ví dụ:**

```csharp
public enum OrderStatus
{
    Pending = 0,
    Processing = 1,
    Completed = 2,
    Cancelled = 3
}
```

**Ghi chú:**  
Enum vẫn là số bên dưới ⇒ cast được giá trị không hợp lệ; nếu nhận từ bên ngoài cần validate.  
`[Flags]` + bit. Underlying mặc định `int`; có thể `byte`/`long`. `ToString`/`Parse` culture. Không dùng enum cho tập giá trị hay đổi (thêm member = breaking binary nếu số đổi).

---

## 21. `event`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khai báo event .NET, cho phép subscribe/unsubscribe handler.

**Ví dụ:**

```csharp
public class Button
{
    public event EventHandler? Click;

    protected virtual void OnClick()
        => Click?.Invoke(this, EventArgs.Empty);

    public void SimulateClick() => OnClick();
}
```

**Ghi chú:**  
Cẩn thận memory leak nếu subscriber không hủy đăng ký ở các scenario sống lâu (winforms, WPF, v.v.).  
`event` chỉ cho phép `+=`/`-=` từ ngoài (không `Invoke` trừ cùng type). `async void` handler: [async.md](async.md). Field-like event không thread-safe tuyệt đối lúc subscribe — thường đủ UI.

---

## 22. `explicit`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Toán tử chuyển kiểu tường minh, bắt buộc dùng cast.

**Ví dụ:**

```csharp
public readonly struct Meter
{
    public double Value { get; }
    public Meter(double value) => Value = value;

    public static explicit operator Meter(double v) => new(v);
    public static explicit operator double(Meter m) => m.Value;
}

Meter m = (Meter)5.0;
double d = (double)m;
```

**Ghi chú:**  
`explicit` khi chuyển kiểu **có thể mất thông tin** hoặc không hiển nhiên — bắt caller viết `(T)`. Cặp với `implicit` (§34): quá nhiều implicit làm overload rối. User-defined conversion phải `static` trong chính type nguồn hoặc đích.

---

## 23. `extension`

- **Loại:** contextual · **C#:** 14.0  
- **Mục đích:** Khai báo **extension block** — nhóm extension members (method, property, operator, static) cho một receiver type trong `static class`. Thay/bổ sung cho extension method kiểu `this` (C# 3+).

**Ví dụ:**

```csharp
public static class StringExtensions
{
    extension(string s)
    {
        public bool IsEmpty => s.Length == 0;

        public string Truncate(int max) =>
            s.Length <= max ? s : s[..max];
    }

    extension(string) // static extensions: không đặt tên receiver
    {
        public static string Empty => string.Empty;
    }
}
```

**Ghi chú:**  
- Extension method cổ điển (`this T`) vẫn hợp lệ và tương thích nhị phân với extension members.  
- Chi tiết thiết kế & `this`: [methods.md — `this` và extension method](methods.md#12-this-và-extension-method); property/`field` liên quan OOP: [oop.md](oop.md).  
- Indexer extension: **C# 15 preview**, không phải C# 14 `extension` block method. Không dùng `extension` làm tên type trừ khi contextual slot cho phép.

---

## 24. `extern`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Chỉ ra method được implement bên ngoài (thường là native DLL, dùng với `DllImport`).

**Ví dụ:**

```csharp
using System.Runtime.InteropServices;

class NativeMethods
{
    [DllImport("kernel32.dll")]
    public static extern void Sleep(uint milliseconds);
}
```

**Ghi chú:**  
`extern` không có body C#. Cần `DllImport` / `LibraryImport` (source gen, .NET 7+) cho native. Sai calling convention / charset → fail lúc chạy, không lúc compile. `extern alias` (cú pháp khác) phân hai assembly cùng identity — hiếm.

---

## 25. `false`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Hằng Boolean `false`.

**Ví dụ:**

```csharp
bool ok = false;
if (!ok)
{
    Console.WriteLine("Not OK");
}
```

**Ghi chú:**  
Hằng, không phải biến. Overload `operator false` (cặp `true`) cho type kiểu DB bool — hiếm; đừng nhầm với `false` literal. `default(bool)` là `false`.

---

## 26. `field`

- **Loại:** contextual · **C#:** 14.0  
- **Mục đích:** Trong accessor của **auto-property**, tham chiếu **backing field** do compiler sinh — không cần khai báo field thủ công.

**Ví dụ:**

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}

public int Score
{
    get => field;
    set => field = value < 0 ? 0 : value;
}
```

**Ghi chú:**

- Chỉ hợp lệ trong `get`/`set`/`init` của property dùng auto-backing-field (không trộn với field `_msg` tự viết trên cùng property).
- Giữ được cú pháp auto-property + logic validate/transform ngắn.
- Chi tiết property / OOP: [oop.md — Truy cập backing field (C#14)](oop.md#52-truy-cập-backing-field-c14).
- `field` contextual: `int field = 1;` ngoài accessor vẫn là identifier. Trùng tên với property `field` trong cùng accessor = warning/lỗi tùy version — đổi tên param/`value`.

---

## 27. `finally`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khối cleanup luôn chạy sau `try`/`catch` (dù có exception hay không).

**Ví dụ:**

```csharp
FileStream? stream = null;
try
{
    stream = File.OpenRead("data.txt");
    // xử lý
}
finally
{
    stream?.Dispose();
}
```

**Ghi chú:**  
Cẩn thận không ném exception mới từ `finally` (dễ che mất exception gốc). Hiện đại hơn là dùng `using` / `using var`.  
`finally` chạy khi `return` trong `try`, **không** đảm bảo với `Environment.Exit` / `FailFast`. `async` + `finally`: [async.md](async.md) / [exceptions.md](exceptions.md).

---

## 28. `fixed`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Trong `unsafe`, pin object để lấy địa chỉ cố định (pointer) cho interop/native.

**Ví dụ:**

```csharp
unsafe
{
    int[] arr = { 1, 2, 3 };
    fixed (int* p = &arr[0])
    {
        // p trỏ tới arr[0], GC không được di chuyển arr trong khối fixed
        Console.WriteLine(*p);
    }
}
```

**Ghi chú:**  
Chỉ trong `unsafe`. Pin xong khối `fixed` thì pointer **hết hạn** — đừng lưu `p` ra ngoài. Buffer lớn: `stackalloc`/`Span` hoặc heap, không `fixed` mảng rồi quên lifetime. `fixed` statement khác `fixed` buffer trong `struct` (`fixed byte buf[16]`).

---

## 29. `float`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số thực 32-bit (nhẹ hơn nhưng kém chính xác hơn `double`).

**Ví dụ:**

```csharp
float f = 1.23f;
```

**Ghi chú:**  
Literal phải `f`/`F` — `1.23` là `double`, gán vào `float` cần convert. Sai số nặng hơn `double`; GPU/interop hay dùng. So sánh: epsilon, không `==`. Tiền tệ: `decimal`.

---

## 30. `for`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Vòng lặp có phần khởi tạo, điều kiện, bước tăng/giảm rõ ràng.

**Ví dụ:**

```csharp
for (int i = 0; i < items.Length; i++)
{
    Console.WriteLine(items[i]);
}
```

**Ghi chú:**  
Phạm vi biến `i` là cả câu `for`. Closure trên `i` rồi enumerate sau (LINQ/deferred) hay bắt **giá trị cuối** — copy local trong vòng. `for` trên `Count` collection bị mutate: undefined. Index thuần: `for`; chỉ đọc phần tử: `foreach`.

---

## 31. `foreach`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Vòng lặp tiện dụng trên mọi `IEnumerable` / `IEnumerable<T>`.

**Ví dụ:**

```csharp
foreach (var item in items)
{
    Console.WriteLine(item);
}
```

**Ghi chú:**  
`item` **không** gán lại được (là iteration variable). Sửa list đang `foreach` → `InvalidOperationException` (hầu hết collection BCL). `await foreach` trên `IAsyncEnumerable`. `ref foreach` / `Span` — [memory-spans.md](memory-spans.md) / [statements.md](statements.md).

---

## 32. `goto`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Nhảy tới label hoặc `case`/`default` trong `switch`.

**Ví dụ:**

```csharp
switch (option)
{
    case 0:
        Console.WriteLine("Zero");
        goto case 1;
    case 1:
        Console.WriteLine("One");
        break;
}
```

**Ghi chú:**  
Thường được xem là “code smell”, trừ vài pattern rất hiếm (ví dụ thoát lồng nhiều vòng).  
**C# 15 preview** có `break outer` — ưu tiên hơn `goto` cho vòng lồng. `goto case` trong `switch` vẫn hợp lệ. Không `goto` xuyên `finally` theo cách bỏ cleanup.

---

## 33. `if`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Câu lệnh rẽ nhánh cơ bản.

**Ví dụ:**

```csharp
if (user is null)
    throw new ArgumentNullException(nameof(user));
```

**Ghi chú:**  
Điều kiện phải `bool` — không “if (ptr)” như C. Pattern: `if (obj is string s)`. Nhiều nhánh: `switch` / switch expression. Early-return giảm `else` lồng.

---

## 34. `implicit`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Toán tử chuyển kiểu ngầm định (không cần cast).

**Ví dụ:**

```csharp
public readonly struct Meter
{
    public double Value { get; }
    public Meter(double value) => Value = value;

    public static implicit operator Meter(double v) => new(v);
}

Meter m = 5.0; // implicit
```

**Ghi chú:**  
Chỉ khi conversion **an toàn / hiển nhiên** (mất thông tin → `explicit`). Chuỗi implicit làm overload resolution bất ngờ. Không định nghĩa cả hai chiều implicit vòng. BCL: `int` → `long` implicit; ngược lại phải cast.

---

## 35. `in`

- **Loại:** reserved · **C#:** 1.0 (thêm ý nghĩa mới ở C# 7.2)  
- **Mục đích:**  
  - Trong tham số: `in` ⇒ readonly by-ref.  
  - Trong `foreach`: một phần cú pháp (`foreach (var x in xs)`).

**Ví dụ (C# 7.2+):**

```csharp
public static double Distance(in Point a, in Point b)
{
    // a, b truyền by-ref nhưng không gán lại được
    // tránh copy struct lớn
}
```

**Ghi chú:**  
`in` trên tham số = `ref readonly` (caller không cần `in` lúc gọi, compiler có thể copy nếu type không `readonly struct`). `foreach (var x in xs)` — `in` ở đây **không** phải modifier tham số. Generic: `in T` = contravariance trên interface/delegate.

---

## 36. `int`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 32-bit có dấu, kiểu integer phổ biến nhất.

**Ví dụ:**

```csharp
int count = 42;
```

**Ghi chú:**  
Alias `System.Int32`. Literal không hậu tố thường là `int` nếu vừa. Overflow mặc định **unchecked** (wrap); `checked` khi cần. `int?` / `default` = 0 — phân biệt với “thiếu giá trị” khi parse.

---

## 37. `interface`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Định nghĩa hợp đồng (method, property, event…) mà class/struct phải thực hiện.

**Ví dụ:**

```csharp
public interface ILogger
{
    void Log(string message);
}
```

**Ghi chú:**  
Không có field instance (trừ default interface members C# 8+ — cẩn thận DIAM). Class implement mọi member hoặc `abstract`. `interface` cho hợp đồng; `abstract class` khi có state/behavior chung. Generic variance: `in`/`out` trên `T`.

---

## 38. `internal`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Access modifier – chỉ thấy được trong cùng assembly.

**Ví dụ:**

```csharp
internal class InternalHelper { }
```

**Ghi chú:**  
Mặc định thành viên class là `private`; type top-level không modifier = `internal`. Test: `InternalsVisibleTo`. `internal` ≠ `file` (C# 11, cùng file). `protected internal` = union (protected **hoặc** same-assembly).

---

## 39. `is`

- **Loại:** reserved · **C#:** 1.0 (pattern matching từ C# 7.0)  
- **Mục đích:**  
  - Kiểm tra kiểu (`obj is T`).  
  - Từ C# 7: pattern matching (`is T v`, `is null`, `is > 0`, v.v.).

**Ví dụ:**

```csharp
if (obj is string s && s.Length > 0)
{
    Console.WriteLine(s);
}
```

**Ghi chú:**  
`is` không ném; thất bại → `false` (khác cast). `as` + null-check ≈ `is T t` (C# 7+). Pattern list/relational: [statements.md](statements.md) / [operators.md](operators.md). `is null` không gọi `==` overloaded — an toàn hơn `== null` trên type có operator.

---

## 40. `lock`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Đồng bộ truy cập giữa các thread (`Monitor.Enter/Exit`).

**Ví dụ:**

```csharp
private readonly object _sync = new();
private int _counter;

public void Increment()
{
    lock (_sync)
    {
        _counter++;
    }
}
```

**Ghi chú:**  
`lock` trên `this` / `typeof(T)` / string interned → deadlock với code ngoài. Dùng `private readonly object _sync = new()`. Không `await` trong `lock`. Async: `SemaphoreSlim`. Chi tiết: [threading.md](threading.md).

---

## 41. `long`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 64-bit có dấu (`System.Int64`).

**Ví dụ:**

```csharp
long big = 1_000_000_000_000L;
```

**Ghi chú:**  
Literal lớn hơn `int.MaxValue` cần `L` hoặc kiểu đích `long`. `ticks` / file size hay dùng `long`. `long` → `int` phải explicit (mất dữ liệu). JSON/`Number` lớn: kiểm tra bound.

---

## 42. `namespace`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Tổ chức không gian tên cho type.

**Ví dụ:**

```csharp
namespace MyApp.Core
{
    public class Service { }
}
```

**Ghi chú:**  
File-scoped `namespace MyApp;` (C# 10) — một namespace/file, ít indent. Nested namespace ≠ folder bắt buộc, nhưng convention khớp. TLS: **không** bọc statements trong namespace cùng file — [main-function.md](main-function.md). `using` không phải namespace member.

---

## 43. `new`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:**  
  - Tạo instance: `new Type(...)`.  
  - Hide member base: `new void Foo()`.  
  - Constraint generic: `where T : new()` - yêu cầu kiểu T phải có constructor không tham số (public parameterless constructor).

**Ví dụ:**

```csharp
var p = new Person("Alice");

public class Base
{
    public void Print() => Console.WriteLine("Base");
}

public class Derived : Base
{
    public new void Print() => Console.WriteLine("Derived");
}

// Generic constraint với new():
public class Factory<T> where T : new()
{
    public T Create() => new T(); // Đảm bảo T có constructor không tham số
}
```

**Ghi chú:**  
Ba nghĩa: (1) `new T()`, (2) hide member, (3) `where T : new()`. `new` hide **không** polymorphic — gọi qua base vẫn base; muốn override dùng `virtual`. Constraint `new()` loại `ref struct` / type không có ctor public parameterless. Target-typed `new()` (C# 9): `List<int> x = new();`.

---

## 44. `null`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Giá trị “không tham chiếu tới object nào” cho reference type & nullable value type.

**Ví dụ:**

```csharp
string? name = null;
if (name is null)
{
    // ...
}
```

**Ghi chú:**  
NRT: `string?` vs `string`. `null` không gán vào non-nullable value type (`int` — dùng `int?`). So sánh: `is null` / `is not null`. `Nullable<T>.HasValue`. `default` reference = `null`.

---

## 45. `object`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Kiểu gốc của mọi reference type (alias cho `System.Object`).

**Ví dụ:**

```csharp
object o = 42;   // boxing
int x = (int)o;  // unboxing
```

**Ghi chú:**  
Boxing `int` → `object` cấp phát heap; hot-path tránh. Unboxing sai kiểu → `InvalidCastException`. Mọi type (kể cả `struct`) kế thừa chuỗi API `object` (`Equals`, `GetHashCode`) — override cặp khi làm key.

---

## 46. `operator`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Định nghĩa toán tử overload (`+`, `-`, `==`, conversion…) cho type custom.

**Ví dụ:**

```csharp
public readonly struct Money
{
    public decimal Value { get; }

    public Money(decimal value) => Value = value;

    public static Money operator +(Money a, Money b)
        => new(a.Value + b.Value);
}
```

**Ghi chú:**  
Overload `==` thì overload `!=` và thường `Equals`/`GetHashCode`. Conversion: `implicit`/`explicit operator`. Compound assignment C# 14: [operators.md](operators.md). Không overload `&&` trực tiếp (đi qua `true`/`false`/`&`).

---

## 47. `out`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Tham số output; method phải gán trước khi return.

**Ví dụ:**

```csharp
if (int.TryParse("123", out int value))
{
    Console.WriteLine(value);
}
```

**Ghi chú:**  
Caller không cần khởi tạo biến `out`. `out var` / `out int x` inline (C# 7). Generic: `out T` = covariance. Đừng dùng `out` cho API mới nếu có thể trả tuple / `bool Try…`. `out` khác `ref` (phải ghi trước return).

---

## 48. `params`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Cho phép truyền số lượng đối số biến đổi (varargs).

**Ví dụ:**

```csharp
void Log(params string[] messages)
{
    foreach (var m in messages)
        Console.WriteLine(m);
}

Log("A", "B", "C");
```

**Ghi chú:**  
Chỉ **một** `params`, phải **cuối** danh sách. C# 13+/14: `params` collections (`params ReadOnlySpan<T>`, `params IEnumerable<T>`) — [methods.md §11](methods.md#11-params). Gọi `Log(array)` không spread thêm lớp. Overload `params` vs cố định: compiler ưu tiên cố định.

---

## 49. `private`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Access modifier – chỉ trong cùng type.

**Ví dụ:**

```csharp
public class User
{
    private string _passwordHash = "";
}
```

**Ghi chú:**  
Mặc định member class/struct/record. Nested type `private` chỉ outer thấy. Không có “private cho file” — dùng `file` (C# 11) cho type. Property `private set` vẫn `get` public.

---

## 50. `protected`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Access modifier – trong type và lớp dẫn xuất.

```csharp
public class Base
{
    protected void OnChanged() { }
}
```

**Ghi chú:**  
Derived **khác assembly** vẫn thấy `protected`. `private protected` = derived **cùng** assembly. `protected internal` rộng hơn. Dùng cho hook `OnXxx`, không phải API public.

---

## 51. `public`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Access modifier – public ở khắp nơi nếu nhìn thấy type/assembly.

```csharp
public class ApiClient { }
```

**Ghi chú:**  
Public type trong library = surface NuGet — breaking khi đổi. Minimal API: thu hẹp `public`. Top-level type không ghi modifier = `internal`, không phải `public`.

---

## 52. `readonly`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Field chỉ gán trong ctor hoặc tại điểm khai báo.

```csharp
public class Config
{
    public readonly string ConnectionString;

    public Config(string conn) => ConnectionString = conn;
}
```

**Ghi chú:**  
`readonly struct` / `readonly` member (C# 7.2/8): method không mutate. `init` property ≠ `readonly` field. `ref readonly` trả về. Static `readonly` chạy lúc type init — khác `const` (inline IL).

---

## 53. `ref`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Tham số by-ref (đọc/ghi), C# 7+ có `ref local`, `ref return`.

```csharp
void Swap(ref int a, ref int b)
{
    int t = a; a = b; b = t;
}
```

**Ghi chú:**  
Caller phải `ref` lúc gọi. `ref struct` (`Span<T>`) không hộp, không lên heap. `ref` local/return: lifetime — [memory-spans.md](memory-spans.md). `ref` khác `out` (đọc/ghi vs phải ghi). C# 13: `allows ref struct`.

---

## 54. `return`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Trả giá trị (nếu có) và kết thúc method/local function.

```csharp
int Double(int x) => x * 2;
```

**Ghi chú:**  
`void` method: `return;` không giá trị. TLS: `return n` suy `int` Main — [main-function.md](main-function.md). `ref return`: `return ref field`. Iterator: `yield return` ≠ `return` (kết thúc iterator = `yield break` / hết method).

---

## 55. `sbyte`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 8-bit có dấu; không CLS-compliant, hiếm dùng.

```csharp
sbyte x = -5;
```

**Ghi chú:**  
Không CLS-compliant — API public nên `int`/`byte`. Phép toán promote lên `int`. Interop/binary protocol mới gặp. Overflow 127+1 → wrap nếu unchecked.

---

## 56. `sealed`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:**  
  - `sealed class`: cấm kế thừa.  
  - `sealed override`: không cho override tiếp.

Khi được áp dụng cho một lớp (class), từ khóa sealed sẽ ngăn các lớp khác kế thừa từ lớp đó. Trong ví dụ sau, lớp FinalType kế thừa từ lớp BaseType, nhưng không có lớp nào có thể kế thừa từ lớp FinalType.

```csharp
class BaseType {}
sealed class FinalType : BaseType {}
```

- Bạn cũng có thể dùng từ khóa `sealed` cho một phương thức (method) hoặc thuộc tính (property) đang override một phương thức/thuộc tính `virtual` trong lớp cơ sở (base class). Điều này cho phép bạn vẫn cho phép các lớp khác kế thừa từ lớp của bạn, nhưng ngăn chúng override một số phương thức/thuộc tính `virtual` cụ thể.
- Bạn không thể áp dụng `sealed` chung với `abstract` khi khai báo lớp, vì bạn buộc phải cho phép thừa kế từ lớp `abstract` mới có thể dùng được.

---

## 57. `short`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 16-bit có dấu.

```csharp
short s = 10;
```

**Ghi chú:**  
Promote lên `int` khi tính toán — gán lại cần cast. Interop/`Int16` / binary. `short` + `short` → `int`. Ít dùng hơn `int` trừ khi packing memory.

---

## 58. `sizeof`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Trả kích thước (byte) của kiểu.

```csharp
int size = sizeof(int); // 4
```

**Ghi chú:**  
`sizeof` kiểu known-unmanaged: compile-time. `sizeof(T)` generic cần `unsafe` hoặc `Unsafe.SizeOf<T>`. Không gồm padding “ý nghĩa” marshal — `Marshal.SizeOf` khác. Reference type: không `sizeof(string)` theo nghĩa độ dài chuỗi.

---

## 59. `stackalloc`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Cấp phát mảng trên stack, thường dùng với `Span<T>` hoặc pointer.

```csharp
Span<int> span = stackalloc int[100];
```

**Ghi chú:**  
Stack — đừng `stackalloc` theo input user không bound (stack overflow). C# 7.2+: `Span<T>` không cần `unsafe`. Lifetime kết thúc khi method return; đừng trả `Span` trỏ stack ra ngoài. Chi tiết: [memory-spans.md](memory-spans.md).

---

## 60. `static`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Thành viên/kiểu thuộc về type, không thuộc instance.

```csharp
public static class MathHelper
{
    public static int Square(int x) => x * x;
}
```

**Ghi chú:**  
`static class` không instance, không kế thừa. Local function/lambda `static` (C# 8/9) cấm capture. `using static`. C# 11: `static abstract` trên interface (generic math). `Main` phải `static`.

---

## 61. `string`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Chuỗi Unicode immutable (alias `System.String`).

```csharp
string s = "Hello";
s += " world"; // tạo string mới
```

**Ghi chú:**  
Immutable — `+=` trong vòng lặp = nhiều alloc; dùng `StringBuilder` / interpolation. So sánh: `StringComparison` rõ, không dựa culture mặc định. `string?` NRT. Alias `String` cùng kiểu.

---

## 62. `struct`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Khai báo value type tùy biến.

```csharp
public readonly struct Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
}
```

**Ghi chú:**  
Value type: copy khi gán/truyền (trừ `ref`/`in`). Mutate copy ≠ mutate gốc — ưu tiên `readonly struct`. `struct` không tham số ctor mặc định luôn có (C# 10+ có thể định nghĩa parameterless). `ref struct`: [memory-spans.md](memory-spans.md). `record struct`: [typesystem.md](typesystem.md).

---

## 63. `switch`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Câu lệnh rẽ nhánh nhiều nhánh; C# 8+ có thêm switch expression.

```csharp
switch (day)
{
    case DayOfWeek.Monday:
        Console.WriteLine("Start");
        break;
    default:
        Console.WriteLine("Other");
        break;
}
```

**Ghi chú:**  
Statement: mỗi `case` phải `break`/`return`/`goto` (fall-through C bị cấm, trừ empty case xếp chồng). Expression `day switch { … }` (C# 8): exhaustive hơn. Pattern: [statements.md](statements.md). **C# 15 preview:** labeled `break` vòng — không phải `switch`.

---

## 64. `this`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Tham chiếu tới instance hiện tại; trong extension method đứng trước tham số đầu tiên.

```csharp
public class Person
{
    public string Name { get; set; } = "";
    public void Introduce()
        => Console.WriteLine($"I'm {this.Name}");
}

public static class StringExtensions
{
    public static bool IsEmpty(this string? value)
        => string.IsNullOrEmpty(value);
}
```

**Ghi chú:**  
Ctor chain: `this(...)`. Extension cổ điển: `this T` tham số đầu. Tránh `lock (this)`. Primary ctor / capture `this` trong lambda: lifetime. `this` không dùng trong `static` member (trừ extension receiver).

---

## 65. `throw`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Ném exception.

```csharp
if (id <= 0)
    throw new ArgumentOutOfRangeException(nameof(id));
```

**Ghi chú:**  
`throw;` trong `catch` giữ stack; `throw ex` reset. Expression: `?? throw`. Không dùng exception cho control flow thường. Chi tiết: [exceptions.md](exceptions.md). `nameof` trên param: refactor-safe.

---

## 66. `true`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Hằng Boolean `true`.

```csharp
while (true)
{
    if (ShouldStop()) break;
}
```

**Ghi chú:**  
Hằng. Cặp `operator true`/`false` cho type custom (SQL-style). `true` literal kiểu `bool`. `while (true)` + `break` hợp lệ; `CancellationToken` sạch hơn loop vô hạn server.

---

## 67. `try`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Bắt đầu khối có xử lý ngoại lệ (`catch`, `finally`).

```csharp
try
{
    DoWork();
}
catch (Exception ex)
{
    Log(ex);
}
```

**Ghi chú:**  
Cần ít nhất `catch` hoặc `finally`. Filter: `catch (Exception ex) when (ex.InnerException is IOException)`. `try` không làm “rẻ” — đừng bọc hot-path. [exceptions.md](exceptions.md).

---

## 68. `typeof`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Lấy `System.Type` của một kiểu.

```csharp
Type t1 = typeof(string);
Type t2 = typeof(List<int>);
```

**Ghi chú:**  
`typeof` bind compile-time; `GetType()` runtime (có thể derived). `typeof(List<>)` unbound — C# 14 `nameof` unbound generic liên quan [operators.md §13](operators.md). Không `typeof` trên biến (`x.GetType()`). Open generic: `typeof(Dictionary<,>)`.

---

## 69. `uint`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 32-bit không dấu; không CLS-compliant.

```csharp
uint u = 10u;
```

**Ghi chú:**  
Không CLS-compliant — API public nên `int` nếu đủ range. Hậu tố `u`/`U`. Mix signed/unsigned → promote bất ngờ (so sánh `< 0` với `uint` luôn false sau convert). Bit flags 32-bit không âm.

---

## 70. `ulong`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 64-bit không dấu.

```csharp
ulong u = 10UL;
```

**Ghi chú:**  
Không CLS; API public thường `long`. Hậu tố `UL`. Không âm — trừ `ulong` wrap. Mix `long`/`ulong` dễ warning signed/unsigned.

---

## 71. `unchecked`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Tắt kiểm tra overflow.

```csharp
unchecked
{
    int x = int.MaxValue + 1; // wrap, không ném exception
}
```

**Ghi chú:**  
Mặc định project thường unchecked. Cặp với `checked` (§10) — bật checked toàn project rồi `unchecked` chỗ bit-twiddle / hash. `unchecked((int)0xFFFFFFFF)` cast bit pattern.

---

## 72. `unsafe`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Cho phép dùng pointer, `stackalloc`, `fixed` – giống C/C++ style.

```csharp
unsafe void Foo(int* p)
{
    *p = 42;
}
```

**Ghi chú:**  
Cần `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>`. Pointer không GC-safe ngoài `fixed`. Ưu tiên `Span<T>` / `ref` trước khi `unsafe`. [memory-spans.md](memory-spans.md). AOT/trim: `unsafe` vẫn compile nhưng không “thêm an toàn”.

---

## 73. `ushort`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Số nguyên 16-bit không dấu.

```csharp
ushort u = 10;
```

**Ghi chú:**  
Không CLS. Promote `int` khi tính. Port/length 16-bit, BMP Unicode code unit ≠ `char` semantics đầy đủ (surrogate). Overflow wrap nếu unchecked.

---

## 74. `using`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:**  
  - Import namespace (`using System;`).  
  - Quản lý lifetime tài nguyên (`using var` hoặc `using (...) { ... }`).

```csharp
using System;

using var stream = File.OpenRead("data.txt");
// dùng stream...
```

**Ghi chú:**  
Ba vai: `using Ns;` / `global using` / `using static` (C# 6/10) và `using` statement/`using var` (IDisposable). `await using` cho `IAsyncDisposable`. File-scoped không liên quan `using`. [exceptions.md](exceptions.md) · implicit usings: [projects-packages.md](projects-packages.md).

---

## 75. `virtual`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Cho phép member được override trong lớp con.

```csharp
public class Base
{
    public virtual void Do() { }
}
```

**Ghi chú:**  
Chỉ class (không `struct` instance virtual). `override` trên derived; thiếu `override` + cùng chữ ký = hide (`new`) — bug phổ biến. `abstract` = virtual không body. `sealed override` chặn lớp cháu.

---

## 76. `void`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Kiểu trả về “không có gì”.

```csharp
void Log(string message) => Console.WriteLine(message);
```

**Ghi chú:**  
Không phải kiểu giá trị — không `var x = void`. `Task`/`Task<int>` khác `void` (async). `async void` chỉ event handler — [async.md](async.md). `Main` `void` → exit 0 trừ `ExitCode`. Pointer: `void*`.

---

## 77. `volatile`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Field `volatile` đảm bảo read/write luôn đi thẳng bộ nhớ, cải thiện visibility giữa thread.

```csharp
public volatile bool _stopped;
```

**Ghi chú:**  
**Không** thay `lock` / `Interlocked` cho read-modify-write (`flag++` vẫn race). Chỉ visibility, không atomicity rộng. `volatile` trên `int`/`reference`; không `long` an toàn mọi nền (dùng `Interlocked`). Hầu hết code mới: `lock`, `volatile` field cờ stop đơn giản, hoặc `CancellationToken`. [threading.md](threading.md).

---

## 78. `while`

- **Loại:** reserved · **C#:** 1.0  
- **Mục đích:** Vòng lặp kiểm tra điều kiện trước mỗi lần lặp.

```csharp
int i = 0;
while (i < 10)
{
    Console.WriteLine(i++);
}
```

**Ghi chú:**  
Điều kiện kiểm tra **trước** body (khác `do`). `while (true)` cần `break`/`return`/cancel. Collection: `foreach` rõ hơn index. **C# 15 preview:** `break outer` trên vòng có nhãn.

---

## 79. `union`

- **Loại:** reserved · **C#:** 15.0 — **PREVIEW (.NET 11 / C# 15)**  
- **Mục đích:** Khai báo một **union type** — kiểu có thể chứa đúng một trong số các kiểu thành viên đã xác định (tập đóng). Xem chi tiết tại [Union types (C# 15) — PREVIEW](typesystem.md#18-union-types-c-15-preview).

**Ví dụ:**

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");  // implicit conversion

string name = pet switch
{
    Dog d  => d.Name,
    Cat c  => c.Name,
    Bird b => b.Name,
};
```

**Ghi chú:**  
- **Không thuộc baseline .NET 10 / C# 14** — cần preview toolchain.  
- Các kiểu thành viên được chuyển đổi ngầm định sang union type.  
- Compiler bắt buộc xử lý đầy đủ tất cả các case trong `switch` (exhaustiveness).  
- Yêu cầu .NET 11 Preview + `<LangVersion>preview</LangVersion>`.

---

## 80. `closed`

- **Loại:** contextual · **C#:** 15.0 — **PREVIEW (.NET 11 / C# 15)**  
- **Mục đích:** Đánh dấu class/record hierarchy **đóng** trong assembly — mọi derived type phải cùng assembly; `switch` exhaustive. Xem [oop.md §2.6](oop.md#26-closed-hierarchies-c-15-preview).

**Ví dụ:**

```csharp
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;
```

**Ghi chú:** Khác `union` (ghép kiểu không cần thừa kế). Không thuộc baseline C# 14.  
Mọi derived phải **cùng assembly**; library public `closed` hạn chế consumer extend — đúng ý exhaustiveness, sai ý plugin. Preview: [oop.md §2.6](oop.md#26-closed-hierarchies-c-15-preview).

---

## 81. Contextual keywords & alias (không đủ chỗ từng mục)

Các token dưới **không** luôn reserved; chỉ là keyword trong ngữ cảnh (chỗ khác có thể là tên biến). Chi tiết đầy đủ nằm ở topic file — bảng chỉ **một dòng ví dụ** để nhận diện. Năm token hay tra nhất: **`record`**, **`async`/`await`**, **`yield`**, **`var`**, **`nameof`**.

| Token | Vai trò ngắn | Ví dụ | Topic |
|-------|----------------|-------|-------|
| `record` | `record class` / `record struct` — value-ish equality, `with` | `public record Person(string Name);` | [oop.md](oop.md) · [typesystem.md](typesystem.md) §8 |
| `required` | member bắt buộc init (C# 11) | `public required string Name { get; init; }` | [oop.md](oop.md) §1.8 / §5.4 |
| `file` | access modifier cùng file (C# 11) | `file class HiddenHelper { }` | [oop.md](oop.md) §1.3 |
| `scoped` | lifetime `ref`/`ref struct` (C# 11) | `void F(scoped ref Span<int> s)` | [statements.md](statements.md) §4.6 · [memory-spans.md](memory-spans.md) |
| `when` | filter `catch` / pattern | `catch (IOException ex) when (ex.HResult == 5)` | [exceptions.md](exceptions.md) · [statements.md](statements.md) |
| `with` | copy record; `with(...)` collection **C# 15 preview** | `var p2 = p with { Name = "B" };` | [typesystem.md](typesystem.md) · [collections-generics.md](collections-generics.md) |
| `and` / `or` / `not` | pattern combinator (C# 9) | `x is > 0 and < 10` | [operators.md](operators.md) · [statements.md](statements.md) |
| `async` | đánh dấu method/lambda bất đồng bộ | `async Task RunAsync() { … }` | [async.md](async.md) |
| `await` | chờ awaitable; TLS/async Main được | `var n = await http.GetStringAsync(url);` | [async.md](async.md) · [main-function.md](main-function.md) |
| `yield` | iterator `yield return` / `yield break` | `yield return item;` | [methods.md](methods.md) §13 |
| `var` | suy luận kiểu **local** (không phải field) | `var list = new List<int>();` | [typesystem.md](typesystem.md) §11 |
| `nameof` | tên symbol; C# 14 unbound generic | `throw …(nameof(arg));` / `nameof(List<>)` | [operators.md](operators.md) §13 |
| `nint` / `nuint` | integer kích thước pointer | `nint p = 0;` | [typesystem.md](typesystem.md) §3.3 |
| `unmanaged` | generic constraint (blittable) | `where T : unmanaged` | [typesystem.md](typesystem.md) §13 |
| `allows` | `allows ref struct` (C# 13) | `where T : allows ref struct` | [typesystem.md](typesystem.md) §13 · [memory-spans.md](memory-spans.md) |
| `dynamic` | DLR binding lúc chạy | `dynamic d = json; d.Name` | [typesystem.md](typesystem.md) §4 |
| `get` / `set` / `init` | accessor property; `init` chỉ lúc khởi tạo | `public int X { get; init; }` | [oop.md](oop.md) |
| `add` / `remove` | accessor `event` tùy chỉnh | `public event Action E { add { } remove { } }` | [oop.md](oop.md) |
| `value` | implicit param trong `set`/`init`/`add`/`remove` | `set => _x = value;` | [oop.md](oop.md) |
| `partial` | type/member ghép nhiều file | `public partial class Program { }` | [oop.md](oop.md) · [main-function.md](main-function.md) |
| `where` (constraint) | generic constraint | `where T : class, new()` | [typesystem.md](typesystem.md) |
| `from` / `select` / `where` (query) | LINQ query syntax | `from x in xs where x > 0 select x` | [linq.md](linq.md) |
| `let` / `join` / `group` / `into` | LINQ query | `group o by o.Id into g` | [linq.md](linq.md) |
| `orderby` / `ascending` / `descending` | LINQ sort | `orderby x.Name descending` | [linq.md](linq.md) |
| `on` / `equals` / `by` | LINQ `join` / `group` | `join o in orders on c.Id equals o.CustomerId` | [linq.md](linq.md) |
| `alias` / `notnull` | `using` alias; constraint `notnull` | `where T : notnull` | [typesystem.md](typesystem.md) |

`extension` / `field` đã có mục 23–26 (C# 14). `closed` mục 80 (C# 15 preview). Preprocessor (`#if`, `#:`) **không** nằm bảng này — [preprocessor-directives.md](preprocessor-directives.md).

### 81.1 Năm contextual hay tra (`record` / `async` / `await` / `yield` / `var` / `nameof`)

- **`record`**: positional `record Person(string Name)` sinh ctor, `Deconstruct`, equality theo giá trị, `with`. `record class` reference; `record struct` value. Không phải reserved — `int record = 1` hợp lệ (đừng). Chi tiết [oop.md](oop.md) / [typesystem.md](typesystem.md) §8. C# 15 `closed record` preview.
- **`async`**: modifier method/lambda/local function trả `Task`/`Task<T>`/`IAsyncEnumerable<T>`/`ValueTask`. **Không** biến `Main` thành CLR async — compiler sinh wrapper (§ `main-function`). `async void` chỉ event. [async.md](async.md).
- **`await`**: chỉ trong `async` (hoặc TLS/`await foreach`). Unwrap `Task` exception. TLS: có `await` ⇒ entry `async Task`. Không `await` trong `lock`. [async.md](async.md) · [main-function.md](main-function.md) §7.
- **`yield`**: `yield return` / `yield break` — iterator, deferred, không phải `return` giá trị method. Không `yield` trong `try` có `catch` (được `try`/`finally`). [methods.md](methods.md) §13 · [linq.md](linq.md) custom operators.
- **`var`**: suy luận **biến local** (và range `foreach`). Không thay `field`/`parameter` (trừ lambda implicit). `var` ≠ `dynamic`. Kiểu phải xác định lúc compile (`var x = null` sai). [typesystem.md](typesystem.md) §11.
- **`nameof`**: đổi tên symbol → string compile-time, không reflection. C# 14: `nameof(List<>)` unbound generic. Argument exception: `nameof(param)`. [operators.md](operators.md) §13.

---

