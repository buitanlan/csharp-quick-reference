# Phương thức (Method) trong C#

> **Baseline:** .NET **10** / C# **14**. Extension members / `params` collections: xem §11–12. Extension indexer: **C# 15** → [oop.md §6.4](oop.md#64-extension-indexers-c-15).

Trong C#, **phương thức (method)** là đơn vị cơ bản để đóng gói logic và định nghĩa hành vi cho một kiểu (`class`, `struct`, `record`, v.v.).  

---

## Mục lục

1. [Khai báo phương thức](#1-khai-báo-phương-thức)
2. [Gọi phương thức](#2-gọi-phương-thức)
3. [Tham số (parameters)](#3-tham-số-parameters)
4. [Truyền theo giá trị và truyền theo tham chiếu](#4-truyền-theo-giá-trị-và-truyền-theo-tham-chiếu)
   - [4.1 Truyền theo giá trị](#41-truyền-theo-giá-trị-by-value--mặc-định)
   - [4.2 Truyền theo tham chiếu](#42-truyền-theo-tham-chiếu-by-reference)
   - [4.3 Copy vs alias](#43-copy-vs-alias--semantics)
5. [Safe context](#5-safe-context)
6. [Modifier trên tham số – tổng quan](#6-modifier-trên-tham-số--tổng-quan)
7. [`ref`](#7-ref)
8. [`out`](#8-out)
9. [`ref readonly`](#9-ref-readonly)
10. [`in`](#10-in)
11. [`params`](#11-params)
12. [`this` và extension method / extension members](#12-this-và-extension-method--extension-members)
13. [Iterator method – `yield return` / `yield break`](#13-iterator-method--yield-return--yield-break)
    - [13.8 Pitfalls iterator \& async streams](#138-pitfalls-iterator--so-sánh-với-async-streams)
14. [Tài liệu liên quan](#14-tài-liệu-liên-quan)

---

## 1. Khai báo phương thức

### 1.1 Cú pháp tổng quát

```csharp
[attributes]
[modifiers] return_type MethodName(parameter_list)
{
    // Thân phương thức (method body)
}
```

Trong đó:

- `attributes`: các attribute như `[Obsolete]`, `[TestMethod]`, v.v. (có hoặc không).
- `modifiers` (bộ sửa đổi):
  - Access: `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`
  - Hành vi: `static`, `virtual`, `override`, `abstract`, `sealed`, `extern`, `unsafe`, v.v.
- `return_type`: kiểu trả về (`int`, `string`, `void`, `T`, v.v.).
- `MethodName`: tên phương thức (theo convention PascalCase).
- `parameter_list`: danh sách tham số (có thể rỗng).

Ví dụ:

```csharp
public class Calculator
{
    // instance method
    public int Add(int x, int y)
    {
        return x + y;
    }

    // static method
    public static int Multiply(int x, int y)
    {
        return x * y;
    }
}
```

### 1.2 Overload method

Cùng tên, **khác chữ ký** (kiểu/ số lượng/ thứ tự tham số) ⇒ nhiều overload:

```csharp
public class Logger
{
    public void Log(string message) { /* ... */ }

    public void Log(string message, Exception ex) { /* ... */ }

    public void Log(Exception ex) { /* ... */ }
}
```

**Semantics:** overload resolution lúc **compile** theo kiểu argument (không phải runtime type). `return type` **không** tham gia chữ ký. C# 13 `[OverloadResolutionPriority]` cho thư viện ưu tiên một overload (`Span` vs `T[]`).

```csharp
void F(object o) => Console.WriteLine("object");
void F(string s) => Console.WriteLine("string");
F((object)"x"); // "object" — kiểu tĩnh là object
```

**Pitfall:** `null` khớp nhiều overload reference → lỗi ambiguous hoặc chọn `string` hơn `object`. Optional parameter + overload dễ “nuốt” call (`F(1)` vs `F(int, int = 0)`).

**Vì sao / Khi nào overload:** cùng ý nghĩa, khác kiểu (`Read` stream vs span). Khác hành vi → tên khác (`TryParse` vs `Parse`).

### 1.3 Expression-bodied method

Dùng khi thân method chỉ là **một expression**:

```csharp
public int Square(int x) => x * x;

public override string ToString()
    => $"{Name} ({Age})";
```

### 1.4 Generic method

Method có tham số kiểu `T`, `TKey`, `TValue`…:

```csharp
public T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

int maxInt = Max(3, 10);         // T => int
string maxStr = Max("a", "z");   // T => string
```

**Vì sao generic method (không chỉ generic type):** thuật toán độc lập chỗ chứa (`Max`, `Swap`). Ràng buộc `where T : IComparable<T>` tránh box — [typesystem.md §13](typesystem.md#13-generics--ràng-buộc-where-new-structclassunmanagednotnull-phương-sai-variance).

**Pitfall:** suy luận `T` thất bại khi argument `null` không kiểu — ghi `Max<string>(a, b)`.

---

## 2. Gọi phương thức

### 2.1 Gọi phương thức instance

Cần **instance** của kiểu:

```csharp
var calc = new Calculator();
int sum = calc.Add(3, 5);
```

### 2.2 Gọi phương thức static

Gọi qua **tên type**:

```csharp
int abs = Math.Abs(-10);
int prod = Calculator.Multiply(2, 3);
```

### 2.3 Named arguments & optional parameters

**Named arguments** – chỉ rõ tên tham số:

```csharp
void SendEmail(string to, string subject, string body = "", bool isHtml = false) { }

SendEmail(
    to: "user@example.com",
    subject: "Hello",
    isHtml: true);
```

**Optional parameters** – tham số có giá trị mặc định:

```csharp
void Log(string message, LogLevel level = LogLevel.Info) { }

Log("Started");                       // level = Info
Log("Critical!", LogLevel.Critical);  // override
```

Lưu ý: giống `const`, giá trị mặc định được embed vào call site (assembly gọi) tại compile-time.

**Pitfall:** đổi default trong thư viện **không** cập nhật caller đã compile — giống đổi `const`. Optional phải đứng sau required. Không default `new List<T>()` (phải compile-time constant) — dùng `null` rồi `??=` trong thân.

**Vì sao / Khi nào named:** nhiều bool/`option` — `SendEmail(isHtml: true)` đọc được. Optional: tiện ích; public contract ổn định thì overload rõ hơn default “ma”.

### 2.4 Method chaining / Fluent API

Method trả về chính nó (hoặc object khác) để gọi liên tiếp:

```csharp
var builder = new StringBuilder()
    .Append("Hello ")
    .AppendLine("world");
```

---

## 3. Tham số (parameters)

**Tham số (parameter)** là biến trong khai báo method.  
**Đối số (argument)** là giá trị truyền vào khi gọi method.

```csharp
void Greet(string name, int times) // name, times là parameters
{
    for (int i = 0; i < times; i++)
        Console.WriteLine($"Hello {name}");
}

Greet("Alice", 3); // "Alice", 3 là arguments
```

Các kiểu tham số cơ bản:

- Tham số bình thường (truyền theo giá trị).
- Tham số có modifier: `ref`, `out`, `in`, `params`, `this`.
- Tham số generic (dựa trên `T` của method/type).
- Tham số optional (có default value).

---

## 4. Truyền theo giá trị và truyền theo tham chiếu

### 4.1 Truyền theo giá trị (by value) – mặc định

Khi **không** dùng `ref` / `out` / `in`, C# luôn truyền **theo giá trị**:

- Với **value type**: copy **giá trị**.
- Với **reference type**: copy **reference**, *không phải* copy object.

```csharp
void ChangeInt(int x) { x = 20; }

void ChangeName(Person p)
{
    p.Name = "Bob";       // thay đổi object
    p = new Person();     // chỉ đổi local p
}

int a = 10;
ChangeInt(a);
Console.WriteLine(a); // 10 (không đổi)

var person = new Person { Name = "Alice" };
ChangeName(person);
Console.WriteLine(person.Name); // "Bob"
```

### 4.2 Truyền theo tham chiếu (by reference)

Dùng `ref`, `out`, `in`:

- Method nhận **tham chiếu** tới biến của caller.
- Thay đổi trên tham số ⇒ tác động trực tiếp biến gốc.

```csharp
void Increment(ref int x) => x++;

int n = 10;
Increment(ref n);
Console.WriteLine(n); // 11
```

### 4.3 Copy vs alias — semantics

Hai mô hình, đừng nhầm:

| | **Copy** (mặc định) | **Alias** (`ref` / `out` / `in` / `ref readonly`) |
|---|---|---|
| Value type | Copy **bits** — callee sửa không đụng caller | Cùng một ô nhớ |
| Reference type | Copy **handle** — cùng object, nhưng gán lại biến không đổi caller | Cùng biến handle — `p = new()` đổi caller |
| Chi phí | Struct lớn = copy đắt | Gần như pointer; `in` đôi khi *vẫn* copy (defensive) |
| Lifetime | Bản copy chết với callee | Alias không được sống lâu hơn biến gốc |

```csharp
void RebindCopy(Person p) => p = new Person { Name = "X" };
void RebindAlias(ref Person p) => p = new Person { Name = "X" };

var a = new Person { Name = "A" };
RebindCopy(a);
Console.WriteLine(a.Name);          // "A" — chỉ copy handle
RebindAlias(ref a);
Console.WriteLine(a.Name);          // "X" — alias biến a
```

**`in` / `ref readonly`:** alias **chỉ đọc**. Mục tiêu: tránh copy struct lớn, không phải “const sâu” — callee vẫn gọi method mutate nếu struct không `readonly` (compiler có thể **defensive copy**).

**Vì sao / Khi nào alias:** `Swap`, `TryParse` (`out`), struct lớn (`in`), sửa slot mảng (`ref return`). Mặc định copy **đúng** cho API thường — dễ suy luận, không lifetime.

**Không nhầm với `ref struct`:** `Span<T>` *là* byref-like type, không phải modifier tham số. Xem [memory-spans.md](memory-spans.md).

---

## 5. Safe context

**Safe context** là vùng code:

- Không dùng con trỏ (`*`, `&` trên kiểu pointer, `T*`, `void*`),
- Không tự dereference con trỏ unmanaged; vẫn có thể dùng API an toàn như Span được thư viện xây trên native memory, với lifetime do owner quản lý,
- Là chế độ mặc định của C#.

Dù dùng `ref`, `out`, `in`, `ref readonly` thì:

- Vẫn an toàn (CLR đảm bảo type-safe, bounds-check…),
- Không phá vỡ quản lý bộ nhớ của GC.

Chỉ khi bạn dùng từ khóa `unsafe` thì method mới ở **unsafe context**:

```csharp
public unsafe void Foo(int* p) { *p = 42; }

unsafe
{
    int value = 10;
    int* p = &value;
    *p = 20;
}
```

**So sánh:** `ref`/`Span` = managed alias, GC-aware. `T*` = không bounds-check. **Unsafe Evolution** (preview, không phải C# 15 mặc định) nới khai báo pointer khỏi `unsafe`, dereference vẫn unsafe — [memory-spans.md §9.1](memory-spans.md#91-memory-safety-c-15-preview).

**Vì sao / Khi nào `unsafe` trên method:** P/Invoke.fill buffer. Không đánh `unsafe` cả class nếu chỉ một method cần.

---

## 6. Modifier trên tham số – tổng quan

Các **parameter modifier** quyết định **cách truyền** tham số:

| Modifier | Đọc | Ghi | Biến caller phải gán trước? | Method bắt buộc gán? | Ghi chú |
|----------|-----|-----|-----------------------------|----------------------|---------|
| *(không)* | ✅ (copy) | chỉ local | — | — | by value (value type copy / ref copy) |
| `ref` | ✅ | ✅ | **Có** | Không | alias đọc/ghi |
| `out` | sau khi gán | ✅ | Không | **Có** (mọi path) | output / `TryXxx` |
| `in` | ✅ | ❌ | Có (hoặc temporary) | Không | by-ref readonly (tham số) |
| `ref readonly` | ✅ | ❌ | Khuyến khích biến; temporary có thể phát warning | Không | ref local/return và tham số C# 12+ |
| `params` | ✅ | — | — | — | varargs; C# 13+: nhiều kiểu collection |
| `this` | — | — | — | — | chỉ tham số đầu — classic extension method |

- `ref` – truyền tham chiếu **đọc/ghi**.
- `out` – truyền tham chiếu, dành cho **giá trị đầu ra**.
- `in` – truyền tham chiếu **chỉ-đọc** (readonly by-ref).
- `params` – tham số “danh sách” (varargs).
- `this` – dùng để khai báo **extension method** (classic).

Trong các mục sau, ta sẽ đi chi tiết từng modifier quan trọng.

> **Lambda:** modifier `ref`/`in`/`out`/`ref readonly` trên lambda — xem [delegates-lambdas.md](./delegates-lambdas.md) (C# 14 cho phép không ghi kiểu tường minh).

---

## 7. `ref`

### 7.1 `ref` parameter

- Biến gọi **phải được gán giá trị trước** khi truyền vào.
- Tham số trong method có thể **đọc & gán** – ảnh hưởng trực tiếp biến gốc.

```csharp
void Swap(ref int a, ref int b)
{
    int t = a;
    a = b;
    b = t;
}

int x = 1, y = 2;
Swap(ref x, ref y);

Console.WriteLine(x); // 2
Console.WriteLine(y); // 1
```

Lưu ý:

- Cần ghi `ref` ở **cả khai báo và chỗ gọi**.
- Không dùng `ref` với `const` hoặc biểu thức không phải lvalue.

**Vì sao / Khi nào dùng `ref` parameter:** `Swap`, tăng counter, sửa phần tử. Không dùng `ref` chỉ vì “sợ copy `int`” — `int` copy rẻ hơn nhiễu API.

### 7.2 `ref` local và `ref` return

**Ref local**: alias tới một vị trí dữ liệu.

```csharp
int[] numbers = { 1, 2, 3 };

ref int second = ref numbers[1];
second = 42;

Console.WriteLine(numbers[1]); // 42
```

**Ref return**: method trả về **tham chiếu**, không phải copy:

```csharp
ref int FindFirstPositive(int[] arr)
{
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] > 0)
            return ref arr[i];

    throw new InvalidOperationException();
}

int[] data = { -1, -2, 5, 3 };
ref int v = ref FindFirstPositive(data);
v = 10;

Console.WriteLine(data[2]); // 10
```

Hạn chế:

- Không được trả `ref` tới biến local (vì sẽ bị thu hồi khỏi stack).
- Chủ yếu dùng với mảng, `Span<T>`, struct field.

```csharp
ref int First(int[] xs) => ref xs[0];

int[] data = [1, 2];
First(data) = 9;                    // gán qua ref return — data[0] == 9
int copy = First(data);             // thiếu `ref` → COPY, sửa copy không đụng mảng
```

**Pitfall:** `ref` return + `await` — không giữ `ref` qua `await`. `ref` field chỉ trong `ref struct`.

**Vì sao / Khi nào `ref` return:** buffer/dictionary slot, tránh copy struct lớn khỏi mảng. Public API rộng: indexer/`Span` thường rõ hơn `ref` return.

---

## 8. `out`

`out` dành cho **tham số đầu ra**, thường dùng với pattern `TryXxx`.

Đặc điểm:

- Biến truyền vào có thể **chưa gán** giá trị.
- Method **bắt buộc** gán giá trị cho `out` trước khi `return`.
- Sau khi method kết thúc, biến `out` sẽ mang giá trị mới.

```csharp
bool TryParseInt(string text, out int value)
{
    return int.TryParse(text, out value);
}

// C# 7: declare-out
if (TryParseInt("123", out int number))
{
    Console.WriteLine(number);
}
else
{
    Console.WriteLine("Không phải số");
}
```

Có thể dùng discard `_` nếu không cần giá trị:

```csharp
if (!int.TryParse(input, out _))
{
    Console.WriteLine("Invalid number");
}
```

Pattern `TryXxx` (vd `int.TryParse`, `Dictionary.TryGetValue`) rất phổ biến và nên được áp dụng cho API của bạn khi cần.

**Semantics vs `ref`:** `out` **không** cần biến đã gán; compiler coi mọi path phải ghi. Đọc `out` trước khi gán → lỗi. `out var` / `out _` (discard) là idiom C# 7+.

```csharp
static bool TryDiv(int a, int b, out int q)
{
    if (b == 0) { q = 0; return false; } // vẫn phải gán q
    q = a / b;
    return true;
}
```

**So sánh trả tuple vs `out`:** `(bool ok, int value)` rõ, nhưng `Try` + `out` là convention BCL (`if (TryGet(..., out var x))`). Không trộn cả hai trên cùng API.

**Pitfall:** `out` trên async method — **không** được `async Task` với `out`/`ref` (signature không sống qua state machine). Cần kết quả async → `Task<(bool, T)>` / `ValueTask<T>` — xem [async.md](async.md) (**không** nhầm `ValueTask` với modifier tham số).

**Vì sao / Khi nào dùng `out`:** `TryParse`, `TryGetValue`, nhiều output nhỏ. Một giá trị thành công duy nhất → return `T?` / `bool` đủ.

---

## 9. `ref readonly`

`ref readonly` là **tham chiếu chỉ-đọc**:

- Dùng cho **ref local** và **ref return**.
- Giúp tránh copy struct lớn nhưng vẫn không cho phép sửa dữ liệu.

Ví dụ trả `ref readonly`:

```csharp
public readonly struct BigStruct
{
    public int A { get; }
    public int B { get; }
}

private readonly BigStruct[] _items;

public ref readonly BigStruct GetItem(int index)
{
    return ref _items[index]; // ref readonly return
}
```

Dùng:

```csharp
ref readonly BigStruct item = ref GetItem(0);
// item = new BigStruct(); // lỗi
Console.WriteLine(item.A);
```

So sánh:

- `ref`          → tham chiếu **đọc/ghi**.
- `ref readonly` → tham chiếu **chỉ-đọc**.
- `in`           → dành cho **tham số** (by-ref readonly parameter).

**`ref readonly` trên tham số (C# 12+):** khuyến khích caller truyền biến bằng `in` hoặc `ref`. Bỏ modifier hay truyền temporary có thể biên dịch với warning CS9192/CS9193, không phải luôn là error. Nếu bật warnings-as-errors thì các warning đó làm build fail.

```csharp
static int Norm(ref readonly Matrix4x4 m) => 0;
Matrix4x4 m = default;
_ = Norm(in m);                         // hoặc Norm(ref m) tùy overload
```

**Pitfall:** `ref readonly` không chặn mutate qua **mutable struct method** (defensive copy). Dùng `readonly struct` + `readonly` instance method.

**Vì sao / Khi nào dùng:** indexer/mảng struct lớn, `ref return` không cho ghi. Tham số: ưu tiên `in`.

---

## 10. `in`

`in` (C# 7.2) cho phép:

- Truyền **bằng tham chiếu**,  
- Nhưng **chỉ-đọc** bên trong method.

Phù hợp cho **struct lớn**, truyền nhiều mà không muốn copy.

```csharp
public readonly struct Matrix4x4
{
    // 16 phần tử, khá to để copy
}

public float Determinant(in Matrix4x4 m)
{
    // m là tham chiếu chỉ-đọc
    // m = default; // lỗi
    return 0; // ví dụ
}
```

Gọi method:

```csharp
Matrix4x4 mat = /* ... */;
float det = Determinant(in mat); // rõ ràng
float det2 = Determinant(mat);   // compiler có thể chèn 'in' ngầm
```

Lưu ý:

- Compiler có thể copy trong một số trường hợp để đảm bảo an toàn.
- Dùng `in` chủ yếu vì lý do hiệu năng (struct lớn), không nên lạm dụng khi struct nhỏ.

**Defensive copy:** nếu `Matrix4x4` không `readonly struct` và callee gọi method instance không `readonly`, compiler copy rồi gọi trên bản sao — **mất** lợi ích `in`.

```csharp
public struct Nasty { public int X; public void Bump() => X++; }

static void Touch(in Nasty n)
{
    n.Bump(); // có thể copy; n của caller không đổi (và bạn tưởng mình tối ưu)
}
```

**So sánh `in` vs copy `int`:** `in int` **chậm hơn** `int` (indirection). Ngưỡng thực dụng: struct ≳ 16–24 byte, hot path.

**Vì sao / Khi nào dùng `in`:** `Matrix`, `decimal` wrapper lớn, `ReadOnlySpan` đã là ref struct (không cần `in`). API mới trên buffer: nhận `ReadOnlySpan<T>` hơn `in T[]`.

---

## 11. `params`

`params` cho phép truyền **0, 1 hoặc nhiều đối số** vào một tham số “danh sách”.

### 11.1 `params` với mảng (classic)

```csharp
void Log(params string[] messages)
{
    foreach (var message in messages)
        Console.WriteLine(message);
}

Log("A");
Log("A", "B", "C");
Log(); // messages.Length == 0

string[] arr = { "X", "Y" };
Log(arr); // vẫn hợp lệ
```

Quy tắc:

- `params` **phải là tham số cuối cùng** trong danh sách.
- Mỗi method chỉ có **tối đa một** tham số `params`.
- Trước C# 13: kiểu phải là **mảng một chiều** (`T[]`).

### 11.2 `params` collections (C# 13+)

Từ **C# 13**, `params` chấp nhận nhiều kiểu “collection-like”, không chỉ `T[]`:

- `Span<T>`, `ReadOnlySpan<T>`
- `IEnumerable<T>`, `ICollection<T>`, `IList<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`, …
- Kiểu có create method tương thích **collection expression** (cùng attribute như collection expressions)
- Struct/class implement `IEnumerable<T>` với ctor không tham số + `Add(T)` instance

```csharp
void Sum(params ReadOnlySpan<int> values)
{
    var total = 0;
    foreach (var v in values)
        total += v;
    Console.WriteLine(total);
}

Sum(1, 2, 3);           // không bắt buộc cấp phát int[]
Sum([10, 20, 30]);      // collection expression (C# 12+)

void WriteAll(params IEnumerable<string> lines)
{
    foreach (var line in lines)
        Console.WriteLine(line);
}

WriteAll("a", "b");
WriteAll(new List<string> { "x", "y" });
```

**Hiệu năng:**

- `params T[]` / nhiều collection tạo instance mới khi truyền từng đối số rời → tránh trong hot-path.
- `params ReadOnlySpan<T>` / `params Span<T>` giúp **tránh cấp phát** khi compiler có thể stackalloc / truyền span từ mảng sẵn có.
- Overload với và không `params` có thể gây nhầm; compiler chọn theo luật overload riêng.

**Semantics C# 13:** đối số rời `Sum(1,2,3)` với `params ReadOnlySpan<int>` → compiler **không** bắt buộc `new int[]` — có thể đưa lên stack. `params IEnumerable<T>` vẫn có thể cấp phát. Collection expression `Sum([1,2,3])` đi theo target type.

```csharp
void A(params int[] xs) { }
void B(params ReadOnlySpan<int> xs) { }
void C(params IEnumerable<int> xs) { }

A(1, 2);                 // cấp phát int[] (trừ khi nội tuyến đặc biệt)
B(1, 2);                 // ưu tiên zero-alloc
C(1, 2);                 // thường enumerator/array — kém hơn span
B(stackalloc int[] { 1, 2 }); // OK với Span/ROS
```

**Pitfall:** `params` + overload `Foo(int)` vs `Foo(params int[])` — `Foo(1)` chọn không-params. `params` **không** kết hợp `ref`/`out`. Một `params` cuối danh sách.

**Vì sao / Khi nào dùng `params ReadOnlySpan<T>`:** API logging/hash/sum hot. Public đơn giản, không đo: `params T[]` vẫn ổn. Truyền collection có sẵn: nhận `IEnumerable<T>` / `ReadOnlySpan<T>` **không** `params` rõ hơn.

```csharp
// Ưu tiên span khi API nhạy hiệu năng
public static bool AllPositive(params ReadOnlySpan<int> xs)
{
    foreach (var x in xs)
        if (x <= 0) return false;
    return true;
}
```

---

## 12. `this` và extension method / extension members

### 12.1 Classic: `this` trên tham số đầu (C# 3+)

`this` trong tham số đầu tiên của một **static method** cho phép khai báo **extension method** — “gắn thêm” method cho type đã tồn tại mà không sửa code type đó.

Quy tắc:

- Method phải trong **static class** (top-level).
- Method phải **static**.
- Tham số đầu tiên: `this SomeType value`.

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);

    public static string ToSlug(this string value)
    {
        return value
            .Trim()
            .ToLowerInvariant()
            .Replace(" ", "-");
    }
}
```

### 12.2 Gọi extension method

Sau khi `using` đúng namespace, có thể gọi như instance method:

```csharp
using MyApp.Extensions;

string? name = null;
if (name.IsNullOrEmpty())
{
    Console.WriteLine("No name");
}

string title = "Hello World C#";
string slug = title.ToSlug(); // "hello-world-c#"
```

Thực chất, compiler dịch thành:

```csharp
StringExtensions.IsNullOrEmpty(name);
StringExtensions.ToSlug(title);
```

### 12.3 Extension members — khối `extension` (C# 14)

C# 14 thêm cú pháp **`extension` block** trong static class: ngoài method, còn **extension property**, **static extension**, **operator**. Classic `this` và `extension` block **cùng IL / tương thích** — caller không phân biệt.

```csharp
public static class EnumerableExt
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();

        public T? FirstOrNull() => source.FirstOrDefault();
    }

    extension<T>(IEnumerable<T>)
    {
        public static IEnumerable<T> Empty => Enumerable.Empty<T>();
    }
}

var xs = new[] { 1, 2 };
_ = xs.IsEmpty;
_ = IEnumerable<int>.Empty;
```

Chi tiết OOP (so sánh, generic block, pitfalls): [oop.md — §8 Extension members](./oop.md#8-extension-members-c-14).

> **C# 15:** extension **indexer** trong `extension` block — mặc định trên `net11.0` từ RC1, không có trên C# 14.

### 12.4 Lưu ý khi thiết kế extension

- Không phá vỡ tính bất biến của type (đặc biệt `string` immutable).
- Tránh behavior bất ngờ; member của type gốc **luôn thắng** extension cùng chữ ký.
- Quan tâm namespace để tránh xung đột tên và overload không mong muốn.
- Prefer `extension` block khi cần **property / static / operator**; giữ classic `this` cho method đơn giản hoặc codebase cũ.

**Vì sao / Khi nào dùng extension:** thêm API lên type không sửa được (`string`, `IEnumerable<T>`). Không dùng để giấu logic domain — method instance trên type của bạn rõ hơn.

### 12.5 Lambda (chỉ dẫn hướng)

Method nhận `Func`/`Action`/delegate tùy chỉnh — lambda là cách viết ngắn. Modifier tham số trên lambda (`ref`/`out`/…): xem [delegates-lambdas.md](./delegates-lambdas.md), không lặp sâu tại đây.

---

## 13. Iterator method – `yield return` / `yield break`

### 13.1 Iterator method là gì?

**Iterator method** là phương thức:

- Trả về **một sequence** (`IEnumerable`, `IEnumerable<T>`, `IEnumerator`, `IEnumerator<T>`),
- Dùng từ khóa **`yield return`** để trả từng phần tử *từng bước một*,
- Có thể dùng **`yield break`** để kết thúc sequence sớm.

Nhờ đó bạn không phải tự cài đặt `IEnumerator<T>` một cách thủ công.

Ví dụ:

```csharp
public IEnumerable<int> EvenNumbers(int from, int count)
{
    int current = from;

    for (int i = 0; i < count; i++)
    {
        if (current % 2 == 0)
            yield return current;

        current++;
    }
}
```

Dùng:

```csharp
foreach (var n in EvenNumbers(1, 10))
{
    Console.WriteLine(n);
}
```

### 13.2 `IEnumerable<T>` và `IEnumerator<T>` – nền tảng của iterator

`foreach` hoạt động dựa trên `IEnumerable<T>` và `IEnumerator<T>`:

```csharp
public interface IEnumerable<out T>
{
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<out T> : IDisposable
{
    T Current { get; }
    bool MoveNext();
    void Reset();
}
```

Khi `foreach (var item in sequence)`:

- Compiler gọi `GetEnumerator()`,
- Lặp: `MoveNext()` → `Current`,
- Kết thúc: `Dispose()`.

Iterator method với `yield` giúp bạn không phải viết bộ cài đặt này.

### 13.3 `yield return` – trả từng phần tử

```csharp
public IEnumerable<int> Range(int start, int count)
{
    for (int i = 0; i < count; i++)
    {
        yield return start + i;
    }
}
```

Sequence kết quả (ví dụ `Range(10, 5)`): `10, 11, 12, 13, 14`.

Đặc điểm:

- **Deferred execution**: logic chỉ thực thi khi bắt đầu duyệt sequence, không phải lúc gọi method.
- Mỗi lần `MoveNext()`: chạy tới `yield return` kế tiếp, lưu state, chờ lần gọi tiếp theo.

### 13.4 `yield break` – kết thúc sớm

```csharp
public IEnumerable<int> UpTo(int max)
{
    for (int i = 0; ; i++)
    {
        if (i > max)
            yield break;

        yield return i;
    }
}
```

Hoặc khi không có dữ liệu:

```csharp
public IEnumerable<string> FindNonEmpty(IEnumerable<string?> source)
{
    if (source is null)
        yield break;

    foreach (var s in source)
    {
        if (!string.IsNullOrWhiteSpace(s))
            yield return s!;
    }
}
```

### 13.5 Iterator & state machine (ý tưởng)

Compiler biến một method như:

```csharp
public IEnumerable<int> Simple()
{
    yield return 1;
    yield return 2;
}
```

thành một class implement `IEnumerable<int>` + `IEnumerator<int>` với trường `_state` và `switch` trong `MoveNext()`.

Bạn chỉ cần nhớ:

- Viết code tuần tự với `yield`,
- Compiler lo việc sinh state machine giúp bạn.

### 13.6 Deferred execution & nhiều lần duyệt

```csharp
var seq = Range(0, 3);

Console.WriteLine("Before foreach");

foreach (var x in seq)
{
    Console.WriteLine(x);
}

Console.WriteLine("After foreach");
```

- `Range` chưa chạy cho tới khi bắt đầu `foreach`.
- Mỗi `foreach` tạo enumerator mới → logic chạy lại.

Nếu muốn **chỉ tính một lần**, hãy materialize:

```csharp
var list = Range(0, 3).ToList(); // thực thi ngay
```

### 13.7 Resource & `try/finally` trong iterator

Iterator method vẫn hỗ trợ `using` / `try/finally`:

```csharp
public IEnumerable<string> ReadLines(string path)
{
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    string? line;
    while ((line = reader.ReadLine()) != null)
    {
        yield return line;
    }
}
```

Khi `foreach` kết thúc (dù bình thường hay exception), enumerator được dispose và `using` đảm bảo resource được giải phóng.

### 13.8 Pitfalls iterator & so sánh với async streams

Validation trong thân iterator cũng bị deferred. Nếu muốn báo argument sai **ngay lúc gọi**, kiểm tra trong wrapper thông thường rồi trả local iterator:

```csharp
static IEnumerable<int> RepeatChecked(int value, int count)
{
    ArgumentOutOfRangeException.ThrowIfNegative(count);
    return Iterate();

    IEnumerable<int> Iterate()
    {
        for (int i = 0; i < count; i++) yield return value;
    }
}
```

- **Không yield return** trong catch, finally hoặc try có catch; try/finally được phép. Yield break có thể dùng trong catch/try có catch nhưng không trong finally.
- `ref`/`Span` không sống qua `yield return` (C# 13: dùng được `ref struct` *ngoài* đoạn có `yield`).  
- Iterator đồng bộ không dùng async; **async iterator** kết hợp async + yield và trả IAsyncEnumerable<T>/IAsyncEnumerator<T>. [Async streams](async.md).
- `ValueTask` / `Task` **không** thuộc chương này: method trả `Task` không dùng `yield` theo nghĩa iterator CLR. Trả sequence sync → `IEnumerable` + `yield`; I/O từng phần tử → async streams.

```csharp
public IEnumerable<int> TakePositive(IEnumerable<int> src)
{
    foreach (var n in src)
    {
        if (n < 0) yield break;
        yield return n;
    }
}

// Materialize nếu gọi 2 lần đắt:
var once = TakePositive(ReadDb()).ToList();
```

**Vì sao / Khi nào dùng iterator:** sinh dãy lười (pipeline LINQ-like), đọc file từng dòng. Cần random access / `Count` O(1) → `List`/`T[]`. Hot-path không muốn state machine → `Span` + vòng `for`.

---

## 14. Tài liệu liên quan

- **OOP / extension members**: [oop.md — §8](./oop.md#8-extension-members-c-14) — *khối `extension`, property/static/operator.* Indexer extension: [oop.md §6.4](./oop.md#64-extension-indexers-c-15) (C# 15).
- **Delegate & Lambda**: [delegates-lambdas.md](./delegates-lambdas.md) — *callback, closure, modifier trên lambda (C# 14).*
- **Chương bất đồng bộ**: [async.md](./async.md) — *async/await, **Task vs ValueTask**, exception/cancellation, async streams (`IAsyncEnumerable<T>`, `await foreach`).*  
  `ValueTask` **không** phải parameter modifier và **không** thuộc file này: dùng khi hot-path async hay hoàn thành đồng bộ (tránh alloc `Task`) — quy tắc pool/`AsTask`/không await hai lần nằm hết ở `async.md`.
- **Toán tử**: [operators.md](./operators.md) — *compound assignment / instance `++` (C# 14).*
- **Span / `ref` lifetime**: [memory-spans.md](memory-spans.md).
