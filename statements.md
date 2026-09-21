# Statements trong C#

> **Baseline:** .NET **10** / C# **14**. Top-level statements: C# **9** (mặc định template từ .NET 6). Labeled `break`/`continue`: **C# 15** (mặc định trên `net11.0` từ RC1).

**Statement** là đơn vị *thực thi* (khác *expression* cho ra giá trị). Hiểu semantics từng nhóm giúp tránh bug phạm vi, tài nguyên, và điều khiển luồng — đặc biệt với pattern matching, `using`/`await using`, và vòng lặp.

---

## Mục lục

- [Statements trong C#](#statements-trong-c)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan \& phân loại](#1-tổng-quan--phân-loại)
  - [2. Top‑level statements (C# 9)](#2-toplevel-statements-c-9)
  - [3. Khối lệnh `{ ... }`, phạm vi \& lifetime](#3-khối-lệnh----phạm-vi--lifetime)
  - [4. Declaration statements](#4-declaration-statements)
    - [4.1 Khai báo biến \& suy luận kiểu (`var`, target-typed)](#41-khai-báo-biến--suy-luận-kiểu-var-target-typed)
    - [4.2 `const` \& `readonly` (local)](#42-const--readonly-local)
    - [4.3 Deconstruction declaration](#43-deconstruction-declaration)
    - [4.4 `ref` local \& `ref readonly` local](#44-ref-local--ref-readonly-local)
    - [4.5 `using` declaration (C# 8+) \& `await using`](#45-using-declaration-c-8--await-using)
    - [4.6 `scoped` (C# 11, nâng cao)](#46-scoped-c-11-nâng-cao)
  - [5. Expression statements](#5-expression-statements)
  - [6. Selection statements: `if`/`else`, `switch`](#6-selection-statements-ifelse-switch)
    - [6.1 `if` / `else` \& pattern matching](#61-if--else--pattern-matching)
    - [6.2 `switch` statement vs switch expression](#62-switch-statement-vs-switch-expression)
    - [6.3 Pattern combinators \& thứ tự case](#63-pattern-combinators--thứ-tự-case)
  - [7. Iteration statements: `while`, `do`, `for`, `foreach`, `await foreach`](#7-iteration-statements-while-do-for-foreach-await-foreach)
    - [7.1 `while` / `do`](#71-while--do)
    - [7.2 `for`](#72-for)
    - [7.3 `foreach` — enumerator, Dispose, pitfall](#73-foreach--enumerator-dispose-pitfall)
    - [7.4 `foreach` vs `for` — khi nào dùng cái nào](#74-foreach-vs-for--khi-nào-dùng-cái-nào)
    - [7.5 `await foreach` (C# 8) — async streams](#75-await-foreach-c-8--async-streams)
  - [8. Jump statements: `break`, `continue`, `return`, `throw`, `goto`, `yield`](#8-jump-statements-break-continue-return-throw-goto-yield)
    - [8.1 `break` / `continue`](#81-break--continue)
    - [8.1.1 Labeled `break` / `continue` (C# 15)](#811-labeled-break--continue-c-15)
    - [8.2 `return`](#82-return)
    - [8.3 `throw`](#83-throw)
    - [8.4 `goto` \& labeled statement](#84-goto--labeled-statement)
    - [8.5 `yield return` / `yield break` (iterator method)](#85-yield-return--yield-break-iterator-method)
  - [9. Exception handling \& resource: `try`/`catch`/`finally`, `using`/`await using`](#9-exception-handling--resource-trycatchfinally-usingawait-using)
    - [9.1 `try` / `catch` / `finally` + filter `when`](#91-try--catch--finally--filter-when)
    - [9.2 `using` statement vs `using` declaration](#92-using-statement-vs-using-declaration)
    - [9.3 `using` vs `await using` — semantics \& pitfalls](#93-using-vs-await-using--semantics--pitfalls)
  - [10. Đồng bộ hoá: `lock` (con trỏ `System.Threading.Lock`)](#10-đồng-bộ-hoá-lock-con-trỏ-systemthreadinglock)
  - [11. Kiểm soát tràn \& môi trường: `checked`/`unchecked`, `unsafe`/`fixed`](#11-kiểm-soát-tràn--môi-trường-checkedunchecked-unsafefixed)
  - [12. Local functions](#12-local-functions)
  - [13. Empty \& labeled statements](#13-empty--labeled-statements)
  - [14. Mẹo \& best practices](#14-mẹo--best-practices)
    - [Phụ lục: Bộ ví dụ “từ đơn giản đến nâng cao”](#phụ-lục-bộ-ví-dụ-từ-đơn-giản-đến-nâng-cao)

---

## 1. Tổng quan & phân loại

- **Statement**: đơn vị thực thi của ngôn ngữ (khác *expression* là biểu thức cho giá trị). Một số cấu trúc vừa là statement vừa chứa expression (`if (expr)`, `return expr`, `await expr`).
- **WHY tách statement/expression:** compiler kiểm soát *luồng* (nhảy, phạm vi, dispose) ở mức statement; expression chỉ tính giá trị. Nhầm lẫn (ví dụ dùng `if` như expression) → phải dùng `?:` / `switch` expression.
- Các nhóm chính:
  - **Declaration**: khai báo biến/const/`using` declaration…
  - **Expression**: lời gọi hàm, gán, `await`…
  - **Selection**: `if`/`else`, `switch` (hỗ trợ **pattern matching** hiện đại).
  - **Iteration**: `while`, `do`, `for`, `foreach`, `await foreach`.
  - **Jump**: `break`, `continue`, `return`, `throw`, `goto`, `yield`.
  - **Exception & resource**: `try`/`catch`/`finally`, `using`, `await using`.
  - **Concurrency**: `lock` (C# 13+: nhận diện `System.Threading.Lock`).
  - **Misc**: `checked`/`unchecked`, `unsafe`/`fixed`, **top‑level statements**, **local functions**, **empty/labeled**.

---

## 2. Top‑level statements (C# 9)

**WHY:** giảm boilerplate `class Program { static void Main() { } }` cho app nhỏ và template mặc định. **Không** phải C# 14 — TLS ổn định từ **C# 9**; C# 10 bổ sung global usings / file-scoped namespace thường đi cùng template.

Chi tiết đầy đủ (nhiều `Main`, `Program`/`<Main>$`, file-based apps, pitfalls): **[main-function.md §3+](main-function.md#3-top-level-statements-c-9)**. Trang này chỉ nhắc semantics statement-level.

```csharp
// Program.cs — file entry
Console.WriteLine("Hello World!");
var name = args.Length > 0 ? args[0] : "guest";
```

Semantics quan trọng:

- **Một** file TLS / project. File TLS **luôn** là entry; `StartupObject` không chọn `Main` khác.
- Có thể `await` và `return int` trực tiếp — compiler suy ra chữ ký `Main` (`void` / `int` / `Task` / `Task<int>`).
- `args` luôn có (không `null`). Type/namespace trong cùng file phải đứng **sau** statements.
- App lớn: giữ TLS mỏng (host builder) rồi gọi type tường minh; hoặc viết `Program.Main` cổ điển.

```csharp
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);
await builder.Build().RunAsync();

// types sau statements
public sealed class Worker : BackgroundService
{
    protected override Task ExecuteAsync(CancellationToken stoppingToken) => Task.CompletedTask;
}
```

**Pitfall:** khai báo `class Program` *trước* statements, hoặc hai file TLS → lỗi compile. Xem [main-function.md §9](main-function.md#9-pitfalls-thường-gặp).

---

## 3. Khối lệnh `{ ... }`, phạm vi & lifetime

- Khối `{ ... }` tạo **scope mới** cho **biến cục bộ**. Lifetime của local kết thúc khi rời scope (stack slot có thể tái dùng; object trên heap sống theo GC).
- Biến local **không có `readonly`** (trừ `ref readonly`/`in`); `const` là compile-time.

```csharp
int x = 1;
{
    int x2 = x + 1;
    // x2 chỉ tồn tại trong khối
}
// x2 không còn ở đây
```

- **Shadowing**: có thể trùng tên ở scope trong (nên tránh vì giảm rõ ràng).
- `using` declaration (mục 4.5) gắn Dispose với **scope hiện tại** — thêm `{ }` chỉ để rút ngắn lifetime tài nguyên là pattern hợp lệ.

```csharp
void Process()
{
    {
        using var tmp = File.Create("scratch.bin");
        tmp.WriteByte(1);
    } // Dispose ở đây — file có thể xóa/mở lại ngay sau
    File.Delete("scratch.bin");
}
```

---

## 4. Declaration statements

### 4.1 Khai báo biến & suy luận kiểu (`var`, target-typed)

```csharp
var n = 42;                 // int
List<string> list = new();  // C# 9 target-typed new
```

- `var` yêu cầu initializer có kiểu rõ; không dùng `var` khi kiểu đích là một phần tài liệu API (`IDisposable stream = Open()`).
- `new()` target-typed suy từ vế trái / tham số — tiện nhưng IDE phải hiện kiểu khi review.

### 4.2 `const` & `readonly` (local)

```csharp
const double PI = 3.141592653589793;
```

> `readonly` áp dụng cho **field** (không phải local). Với local, dùng `in`/`ref readonly` để truyền chỉ-đọc. So sánh `const` vs `readonly` field: [literals.md §8](literals.md#8-hằng-số-const-vs-readonly).

### 4.3 Deconstruction declaration

```csharp
(var x, var y) = (10, 20);
(int a, int b) = GetPoint();
```

Cần `Deconstruct` trên type (hoặc tuple). Gán vào biến đã có: `(x, y) = point;` (không phải declaration).

### 4.4 `ref` local & `ref readonly` local

```csharp
int[] arr = {1,2,3};
ref int second = ref arr[1];
second = 99;                 // arr[1] đổi theo

ref readonly int ro = ref arr[0];
// ro = 5; // lỗi: readonly
```

**Pitfall:** `ref` local không được sống lâu hơn đối tượng được trỏ tới (ref safety). Không trả `ref` tới local của method khác.

### 4.5 `using` declaration (C# 8+) & `await using`

```csharp
using var stream = File.OpenRead(path); // auto Dispose khi ra khỏi scope
await using var conn = await OpenAsync(); // IAsyncDisposable
```

> Khác `using` statement dạng khối ở mục 9. **WHY declaration:** ít thụt lề, Dispose gắn hết method/block hiện tại — hợp khi tài nguyên dùng tới cuối scope. **WHY statement:** cần dispose *sớm hơn* hết method, hoặc chuỗi nested rõ ràng.

Chi tiết `using` vs `await using`: [§9.3](#93-using-vs-await-using--semantics--pitfalls).

### 4.6 `scoped` (C# 11, nâng cao)

Giới hạn lifetime của tham chiếu `ref`/`in` để tránh escape khỏi scope an toàn. Chỉ dùng khi thực sự cần tối ưu `ref` (span/stackalloc). Xem [methods.md](methods.md) / [memory-spans.md](memory-spans.md).

---

## 5. Expression statements

Các **biểu thức** có thể trở thành statement: gọi hàm, gán, tăng/giảm, `await`…

```csharp
x = y + 1;
count++;
DoWork();
await Task.Delay(100);
```

> **`throw`** cũng là statement (và từ C# 7 có dạng *throw expression* trong toán tử `?:`/`??`).

Expression *không* tự thành statement nếu chỉ là giá trị (`x + 1;` cảnh báo CS0201). Phải có side-effect (gọi, gán, `await`, `++`).

---

## 6. Selection statements: `if`/`else`, `switch`

**WHY pattern matching (C# 7–11+):** gom *kiểm tra kiểu + null + ràng buộc thuộc tính* thành một biểu thức, tránh `as` + `if` lặp và cast không an toàn. Compiler hiểu exhaustiveness tốt hơn chuỗi `if` thủ công.

### 6.1 `if` / `else` & pattern matching

```csharp
if (order is null) throw new ArgumentNullException(nameof(order));
if (value is >= 0 and <= 100)
    Console.WriteLine("In range");
else if (value is > 100)
    Console.WriteLine("Too large");
else
    Console.WriteLine("Negative");
```

Pattern thường dùng với `is`:

| Pattern | Ví dụ | Ý nghĩa |
|---|---|---|
| Constant / `null` | `x is null`, `x is 0` | So sánh hằng (null dùng pattern, không `==` nếu overload `==`) |
| Type + discard | `x is string` | Đúng kiểu (hoặc derived) |
| Type + designator | `x is string s` | Vừa test vừa gán `s` (scope: cả `if`/`else` theo luật C# 7+) |
| Property | `x is { Length: > 0 }` | Không cần cast nếu đã biết kiểu |
| Relational | `x is >= 0 and < 10` | C# 9 |
| Logical | `and` / `or` / `not` | C# 9 |
| List (C# 11) | `x is [1, 2, ..]` | Sequence pattern |
| Parenthesized | `x is not (null or "")` | Nhóm |

```csharp
if (payload is Customer { Active: true, Address: { City: "HN" } } c)
    Notify(c);

if (args is ["--file", var path, ..])
    Open(path);

if (node is not null and not ErrorNode)
    Visit(node);
```

**Pitfall:** `if (x is int i)` — `i` có scope xuyên sang `else` nhưng **không assigned** ở nhánh else; dùng `i` ở else là lỗi. `is not null` trên nullable value type khác `HasValue` khi so sánh boxing — ưu tiên pattern trên `T?`.

Điều kiện `if` phải là `bool` (không truthy như JS). Type có `true`/`false` operator mới dùng trực tiếp trong `if`.

**WHY `is null` thay `== null`:** type overload `==` có thể không coi null như bạn nghĩ; pattern `is null` dùng identity/null check của ngôn ngữ. Record/class overload `==` theo value — `is null` vẫn đúng cho reference null.

```csharp
if (obj is Customer { Orders.Count: > 0, Name: { Length: > 0 } name })
    Greet(name);

// list pattern + remainder
if (span is [0x47, 0x49, 0x46, ..]) // GIF
    DecodeGif(span);
```

Nhánh `else if` pattern: compiler **không** chứng minh exhaustiveness như `switch` expression — dễ thiếu case. Domain đóng (enum, union, `closed`): `switch` expression + warning thiếu arm an toàn hơn chuỗi `if`.

### 6.2 `switch` statement vs switch expression

**Switch expression** (C# 8) là *expression* — phải exhaustive (hoặc `_`), mỗi nhánh trả giá trị, **không** `break`:

```csharp
static string Classify(object? x) => x switch
{
    null            => "null",
    int i when i < 0 => "negative int",
    int             => "int",
    string { Length: 0 } => "empty string",
    string s        => $"string({s.Length})",
    _               => "other"
};
```

`switch` **statement** với pattern — nhiều lệnh, `break`/`return`/`goto`/`throw`:

```csharp
object x = Get();
switch (x)
{
    case null:
        Console.WriteLine("null");
        break;

    case int i when i < 0:
        Console.WriteLine("negative int");
        break;

    case string { Length: > 0 } s:
        Console.WriteLine($"string len={s.Length}");
        break;

    default:
        Console.WriteLine("other");
        break;
}
```

**Semantics:**

- Không có fall-through *ngầm* giữa case (phải `goto case` / `goto default` nếu muốn). Mỗi case kết thúc bằng `break`, `return`, `throw`, `continue` (trong vòng), hoặc `goto`.
- `goto case` chỉ với **hằng compile-time** (không pattern phức tạp).
- Switch expression: thứ tự arm **từ trên xuống**; `_` bắt buộc nếu compiler chưa chứng minh hết case (warning/error tùy context).
- Statement `switch` trên `enum` không bắt buộc `default` nhưng thiếu `default` + giá trị ngoài enum (cast) sẽ rơi khỏi switch **im lặng**.

**Khi nào expression vs statement:**

- Expression: ánh xạ 1–1 ra giá trị (`return`, gán, argument).
- Statement: side-effect nhiều dòng, `await` từng nhánh khác nhau, hoặc cần `break` khỏi vòng bao.

### 6.3 Pattern combinators & thứ tự case

Thứ tự **quan trọng**: case hẹp trước, rộng sau. `case string` trước `case object` — nếu đảo, `string` không bao giờ khớp (cảnh báo unreachable).

```csharp
string Describe(IEnumerable<int> xs) => xs switch
{
    []              => "empty",
    [var only]      => $"one={only}",
    [var a, var b]  => $"two={a},{b}",
    [0, .. var rest] => $"starts-zero, rest={rest.Length}",
    _               => "many"
};
```

`when` filter chạy **sau** khi pattern khớp; biểu thức `when` có side-effect thì phụ thuộc thứ tự case — tránh logic nặng trong `when`.

---

## 7. Iteration statements: `while`, `do`, `for`, `foreach`, `await foreach`

### 7.1 `while` / `do`

```csharp
while (reader.Read())
{
    // ...
}

do
{
    ReadInput();
} while (!IsValid());
```

- `while`: có thể không chạy lần nào. `do`: chạy **ít nhất một** lần rồi mới kiểm điều kiện.
- Điều kiện phải `bool`. Vòng vô hạn chủ đích: `while (true)` + `break`/`return` rõ ràng hơn `for (;;)`.

### 7.2 `for`

```csharp
for (int i = 0; i < n; i++)
{
    if ((i & 1) == 0) continue; // bỏ qua số chẵn
    sum += i;
}

// Nhiều biến khởi tạo/iterator
for (int i = 0, j = n-1; i < j; i++, j--) { Swap(a, i, j); }
```

Ba phần (`init`; `condition`; `iter`) đều tùy chọn. `init` có thể khai báo biến chỉ sống trong vòng. Phần `iter` chạy sau mỗi lần lặp (kể cả sau `continue`, **không** chạy sau `break`).

### 7.3 `foreach` — enumerator, Dispose, pitfall

```csharp
foreach (var line in File.ReadLines(path))
    Console.WriteLine(line);
```

**Semantics (ý tưởng compiler):**

```csharp
var e = source.GetEnumerator();
try
{
    while (e.MoveNext())
    {
        var line = e.Current;
        Console.WriteLine(line);
    }
}
finally
{
    (e as IDisposable)?.Dispose(); // hoặc Dispose() nếu enumerator struct IDisposable
}
```

- Hoạt động trên bất kỳ kiểu có `GetEnumerator()` phù hợp (pattern-based, không bắt buộc `IEnumerable`).
- **`break`/`continue`** vẫn hoạt động; `finally` của enumerator vẫn chạy khi `break`.
- Với `Dictionary<TKey,TValue>`: `foreach (var (k,v) in dict)` deconstruction.
- `foreach (ref var x in span)` / `foreach (var x in span)` — `Span<T>` không phải `IEnumerable` nhưng có enumerator.

**Pitfall:**

- Sửa collection đang `foreach` (`List.Add` trong vòng) → thường `InvalidOperationException` (version check). `for` + index hoặc copy snapshot thì được.
- Enumerator struct (ví dụ `List<T>.Enumerator`): `foreach` không box; gán enumerator ra biến `IEnumerator<T>` thì **box** + có thể copy sai.
- `foreach` capture `item` trong lambda: từ **C# 5** mỗi vòng một biến riêng (trước C# 5 mọi vòng chung một biến — bug kinh điển).

```csharp
var list = new List<int> { 1, 2, 3 };
// SAI nếu mutate
foreach (var x in list)
    if (x < 0) list.Remove(x); // nguy hiểm

// OK
for (int i = list.Count - 1; i >= 0; i--)
    if (list[i] < 0) list.RemoveAt(i);
```

### 7.4 `foreach` vs `for` — khi nào dùng cái nào

| | `foreach` | `for` (index) |
|---|---|---|
| Ý đồ | Duyệt *mọi* phần tử, không cần index | Cần chỉ số, bước, duyệt ngược, hai con trỏ |
| Collection | Mọi enumerable / span pattern | Cần indexer `this[int]` + `Count`/`Length` |
| Sửa tại chỗ | Thường **không** (version) | Có thể `RemoveAt`, swap |
| Hiệu năng mảng/`List`/`Span` | JIT thường tối ưu gần `for` | Kiểm soát bounds; hot-path `Span` hay dùng `for` |
| `IEnumerable` thuần | Bắt buộc | Không index được |
| Đọc dễ | Thường hơn | Nhiều biến vòng → dễ lệch off-by-one |

**WHY không luôn `for`:** nhiều API chỉ expose enumerator (LINQ, channel, file lines). **WHY không luôn `foreach`:** thuật toán hai con trỏ, stride, hoặc cần `ref` vào `span[i]`.

```csharp
// foreach: ý đồ "đọc từng dòng"
foreach (var line in File.ReadLines(path))
    if (line.Length > 0) yield return line;

// for: đảo mảng / Span
for (int i = 0, j = a.Length - 1; i < j; i++, j--)
    (a[i], a[j]) = (a[j], a[i]);

// hot path Span
int Sum(ReadOnlySpan<int> s)
{
    int t = 0;
    for (int i = 0; i < s.Length; i++) t += s[i];
    return t;
}
```

Trên mảng, `foreach` và `for` gần như tương đương sau JIT; đừng micro-optimize trước khi đo. Ưu tiên **ý đồ**.

**Enumerator `IDisposable`:** `foreach` trên `File.ReadLines` / `BlockingCollection.GetConsumingEnumerable` / custom enumerator — `Dispose` lúc `break` **quan trọng** (đóng file, đánh dấu consumer xong). `for (int i = 0; i < list.Count; i++)` **không** gọi enumerator Dispose — không thay `foreach` trên consuming enumerable.

```csharp
foreach (var job in queue.GetConsumingEnumerable(ct))
{
    if (job.Done) break; // Dispose enumerator → CompleteAdding phía consumer dừng Take
    Process(job);
}
```

`foreach (var x in array)` JIT thường thành vòng index, không gọi interface — đừng sợ `foreach` trên `T[]`/`List<T>`/`Span<T>` vì “chậm hơn for” nếu chưa đo.

### 7.5 `await foreach` (C# 8) — async streams

```csharp
await foreach (var msg in ReceiveAsync(channel))
    Console.WriteLine(msg);

static async IAsyncEnumerable<string> ReceiveAsync(ChannelReader<string> r)
{
    while (await r.WaitToReadAsync())
        while (r.TryRead(out var m)) yield return m;
}
```

Tương đương `GetAsyncEnumerator` + `await MoveNextAsync` + `await using` dispose. Cancellation: `.WithCancellation(ct)` và/hoặc `[EnumeratorCancellation]` — xem [async.md §10–11](async.md#10-async-streams--iasyncenumerablet--await-foreach).

**Pitfall:** `foreach` *đồng bộ* trên `IAsyncEnumerable` **không compile**. Đừng `.ToList()` lên stream vô hạn. `ConfigureAwait` trên `await foreach` (C# 8+): `await foreach (var x in src.ConfigureAwait(false))`.

---

## 8. Jump statements: `break`, `continue`, `return`, `throw`, `goto`, `yield`

### 8.1 `break` / `continue`

- `break` thoát khỏi vòng lặp hiện tại hoặc `switch`.
- `continue` bỏ phần còn lại của vòng lặp & bắt đầu vòng mới (`for`: vẫn chạy phần iterator).
- Baseline C# 14: chỉ ảnh hưởng vòng/`switch` **gần nhất**.

#### 8.1.1 Labeled `break` / `continue` (C# 15)

> **C# 15 / .NET 11**, mặc định trên `net11.0` từ RC1. Không cần `LangVersion=preview`. Baseline C# 14: thoát vòng ngoài bằng `goto` hoặc cờ boolean.

C# 15 cho phép gắn nhãn vòng / `switch` rồi `break outer;` / `continue outer;`:

```csharp
outer:
for (int i = 0; i < n; i++)
{
    for (int j = 0; j < m; j++)
    {
        if (done) break outer;       // thoát cả hai vòng
        if (skipRow) continue outer; // lần lặp i tiếp theo (chạy iterator của for i)
    }
}
```

**WHY:** `goto done` nhảy tới nhãn *statement* bất kỳ — dễ xuyên `finally`/`using` nếu đặt nhãn sai. Labeled `break`/`continue` **chỉ** điều khiển vòng/`switch` đã gắn nhãn (ý đồ rõ: “thoát/skip vòng này”).

C# 14 tương đương:

```csharp
bool leave = false;
for (int i = 0; i < n && !leave; i++)
{
    for (int j = 0; j < m; j++)
    {
        if (done) { leave = true; break; }
        if (skipRow) break; // chỉ thoát vòng j; cần continue i bằng cách khác
    }
}

// hoặc
for (int i = 0; i < n; i++)
{
    for (int j = 0; j < m; j++)
    {
        if (done) goto exit;
    }
}
exit:;
```

- Nhãn đứng **trước** vòng/`switch` cần điều khiển (cú pháp gần `goto` label, nhưng `break`/`continue` **không** nhảy tùy ý).
- Ưu tiên hơn `goto done` khi ý định là thoát/skip vòng ngoài.
- Không có trên C# 14 / `net10.0`. IDE0410 gợi ý thay cờ boolean hoặc `goto` bằng labeled jump khi toolchain là C# 15.

**Nhãn vs `goto` label:** cùng cú pháp `name:` nhưng labeled `break` **chỉ** hợp lệ khi nhãn gắn vòng/`switch`. `break somewhere;` tới nhãn statement thường (không phải vòng) là lỗi — đó là việc của `goto`. C# 15 không biến `break` thành `goto` tùy ý.

`continue outer` trên `for`: chạy phần iterator của `for outer` (`i++`), rồi kiểm điều kiện. `continue outer` trên `while`: nhảy lại điều kiện `while`, không “tăng i” tự động — phải tự quản biến.

`switch` gắn nhãn: `break thatSwitch;` thoát switch (không phải vòng bao). Ít dùng hơn vòng lồng.

### 8.2 `return`

```csharp
if (input is null) return;
return value; // với kiểu trả về non-void
```

Trong iterator: `yield break` kết thúc sequence; `return` thường không dùng kèm `yield` (trừ local function). Async: `return x` trở thành `Task` hoàn thành với `x`.

### 8.3 `throw`

```csharp
throw new InvalidOperationException("Bad state");

// C# hiện đại: throw expression
var x = value ?? throw new ArgumentNullException(nameof(value));
```

`throw;` trong `catch` giữ stack — xem [exceptions.md](exceptions.md). Helper `ThrowIf*` (.NET 6+) là statement gọi method ném exception.

### 8.4 `goto` & labeled statement

```csharp
start:
if (!TryStep()) goto start;

switch (kind)
{
    case 0: Handle0(); goto done;
    case 1: Handle1(); goto done;
    default: goto case 0;
}
done:;
```

> Dùng *tiết chế*; đa số trường hợp có thể thay bằng cấu trúc điều khiển rõ ràng hơn. `goto` **không** được nhảy vào khối chưa khởi tạo biến / xuyên một số ràng buộc definite assignment. Nhảy *ra* khỏi `try` vẫn chạy `finally`.

### 8.5 `yield return` / `yield break` (iterator method)

```csharp
public static IEnumerable<int> Evens(int from, int count)
{
    int n = from;
    while (count-- > 0)
    {
        if ((n & 1) == 0) yield return n;
        n++;
    }
}
```

Compiler sinh state machine (giống tinh thần async). **Lazy:** thân method chạy khi caller `MoveNext`. Không `yield` trong `try` kèm `catch` (được `try`/`finally`). Không `yield` + `async` trong cùng method — dùng `IAsyncEnumerable` + `await` + `yield` (async iterator). Chi tiết: [methods.md §13](methods.md).

---

## 9. Exception handling & resource: `try`/`catch`/`finally`, `using`/`await using`

### 9.1 `try` / `catch` / `finally` + filter `when`

```csharp
try
{
    await SaveAsync(); // có thể await trong try/catch
}
catch (HttpRequestException ex) when ((int?)ex.StatusCode == 429)
{
    await Task.Delay(1000);
    // retry nhẹ...
}
catch (Exception ex)
{
    Log(ex);
    throw; // giữ stack
}
finally
{
    Cleanup();
}
```

Filter `when` **không** unwind stack nếu không khớp (hữu ích cho logging/telemetry). Chi tiết: [exceptions.md §6](exceptions.md#6-exception-filter-với-when).

### 9.2 `using` statement vs `using` declaration

**Using statement** (khối **nội** scope) — Dispose đúng điểm đóng `}`:

```csharp
using (var stream = File.OpenRead(path))
using (var reader = new StreamReader(stream))
{
    Console.WriteLine(reader.ReadLine());
} // Dispose() được gọi ở đây theo thứ tự ngược
```

**Using declaration** (**C# 8+**, hiệu lực đến hết scope hiện tại):

```csharp
using var stream = File.OpenRead(path);
using var reader = new StreamReader(stream);
// ... dùng reader
// tự Dispose khi rời block hiện tại
```

**WHY statement:** file phải đóng trước khi rename/move ngay trong cùng method; hoặc giới hạn lock/handle ngắn. **WHY declaration:** method dùng resource suốt thân — ít nesting.

Nhiều `using` declaration: Dispose **ngược thứ tự khai báo** (như stack), giống using statement lồng.

### 9.3 `using` vs `await using` — semantics & pitfalls

| | `using` / `using var` | `await using` / `await using var` |
|---|---|---|
| Interface | `IDisposable` | `IAsyncDisposable` |
| Gọi | `Dispose()` | `await DisposeAsync()` |
| Trong async method | Được | Được (và thường **nên** nếu type có async dispose) |
| Trong sync method | Được | **Không** — không `await` được |

Type cài **cả hai** (`Stream`, `HttpClient` không điển hình; nhiều ADO.NET/`Channel` writer): `await using` ưu tiên `DisposeAsync` (tránh sync-over-async trong Dispose). `using` đồng bộ gọi `Dispose()` — có thể block.

```csharp
await using var conn = await OpenConnectionAsync(); // IAsyncDisposable
await using var cmd = conn.CreateCommand();         // nếu command async-disposable

await using (var tx = await conn.BeginTransactionAsync())
{
    await SaveAsync(tx);
    await tx.CommitAsync();
} // DisposeAsync ngay — không chờ hết method
```

**Pitfalls:**

1. **Quên `await using`** trên type chỉ (hoặc chủ yếu) dọn async → deadlock/thread pool starvation nếu `Dispose()` đợi I/O sync.
2. **Dispose ném exception** che exception gốc trong `try` — .NET hiện đại với `await using` vẫn có thể làm mất lỗi gốc nếu không cẩn thận; log cả hai khi debug.
3. **Lifetime declaration quá dài** giữ file/handle/connection hết method dù chỉ cần vài dòng → dùng khối `using ( )`.
4. `using` trên struct `IDisposable` (custom ref-like) — copy enumerator/dispose sai nếu không hiểu boxing; hiếm trong app code.
5. Không `return` resource đã `using` ra ngoài (dangling Dispose). Trả về thì **caller** `using`.

**Cả `IDisposable` lẫn `IAsyncDisposable`:** spec: `await using` gọi `DisposeAsync` (không bắt buộc gọi `Dispose`). Implementer: `DisposeAsync` nên làm việc async; `Dispose` có thể `DisposeAsync().AsTask().GetAwaiter().GetResult()` — **sync-over-async**, tránh nếu bạn đang trên UI/pool. Ưu tiên `await using` trong async method.

```csharp
public sealed class Pipe : IDisposable, IAsyncDisposable
{
    public void Dispose() => DisposeAsync().AsTask().GetAwaiter().GetResult(); // last resort
    public async ValueTask DisposeAsync()
    {
        await FlushAsync();
        _socket.Dispose();
    }
}
```

---

## 10. Đồng bộ hoá: `lock` (con trỏ `System.Threading.Lock`)

Đảm bảo **mutual exclusion** cho đoạn critical:

```csharp
private readonly object _sync = new();

void Add(int value)
{
    lock (_sync)
    {
        _total += value;
    }
}
```

- Tương đương `Monitor.Enter/Exit` (an toàn với exception) khi operand là `object`.
- **C# 13 / .NET 9+:** nếu operand kiểu `System.Threading.Lock`, compiler sinh đường `Lock.EnterScope` — **không** dùng sync-block `object`. Baseline .NET 10: field mới nên dùng `Lock`.
- **Tránh** lock trên `this` hoặc `Type` public (dễ deadlock do bên ngoài cũng lock).
- Với async: **không** `lock` quanh `await`. Dùng `SemaphoreSlim.WaitAsync` hoặc Channel.

Chi tiết so sánh `lock`/`Monitor` vs `System.Threading.Lock`, `Wait`/`Pulse`, try-enter: **[threading.md §4.1–4.2](threading.md#41-lockmonitor)**.

```csharp
using System.Threading;

private readonly Lock _gate = new(); // .NET 9+ / C# 13

void Add(int value)
{
    lock (_gate)
        _total += value;
}
```

---

## 11. Kiểm soát tràn & môi trường: `checked`/`unchecked`, `unsafe`/`fixed`

```csharp
checked
{
    int c = int.MaxValue + 1; // ném OverflowException
}

unchecked
{
    int d = int.MaxValue + 1; // tràn im lặng
}
```

Mặc định project thường `unchecked` cho `int` (wrap). `checked` theo khối, biểu thức `checked(a + b)`, hoặc property MSBuild. `decimal` không wrap như `int`.

**Unsafe & fixed** (khi cần interop/hiệu năng thấp‑level):

```csharp
unsafe
{
    int x = 10;
    int* p = &x;
    *p = 20;

    fixed (char* cp = "abc") { /* dùng cp */ }
}
```

> Chỉ bật khi thật sự cần; xem thêm [typesystem.md](typesystem.md) và [operators.md](operators.md) (pointer ops). Nới “khai báo pointer” khỏi `unsafe` là **Unsafe Evolution** (vẫn preview) — [memory-spans.md](memory-spans.md).

---

## 12. Local functions

Hàm cục bộ nằm bên trong method, **đặt tên được**, có thể `async` hoặc `iterator`:

```csharp
int SumSquares(ReadOnlySpan<int> a)
{
    static int Sqr(int x) => x * x; // static: không capture

    int sum = 0;
    foreach (var x in a) sum += Sqr(x);
    return sum;
}
```

- Khác lambda: local function **ít cấp phát hơn** khi không capture; hỗ trợ `ref/out`/`yield`; đệ quy dễ.
- `static` local: cấm capture — lỗi compile nếu vô tình đóng biến ngoài.
- So sánh sâu: [delegates-lambdas.md §11](delegates-lambdas.md#11-local-functions-vs-lambda).

---

## 13. Empty & labeled statements

- **Empty**: chỉ dấu `;` — đôi khi dùng làm **no‑op** (hiếm). Nguy hiểm: `if (ok); DoWork();` — `DoWork` luôn chạy.
- **Labeled**: `label:` đứng trước một statement để `goto` tới (mọi phiên bản) hoặc gắn vòng cho labeled `break` (C# 15).

```csharp
; // empty

retry:
if (!TryConnect())
{
    attempts++;
    if (attempts < 3) goto retry;
}
```

---

## 14. Mẹo & best practices

1. **Ưu tiên cấu trúc rõ ràng** thay vì `goto`. Labeled `break` trên C# 15; trên .NET 10 dùng cờ hoặc `goto` có kiểm soát.
2. **Luôn `break`** trong `switch` statement (trừ khi `goto`/`return`), tránh rơi qua. Switch **expression** không `break`.
3. **Bao try/catch ở rìa hệ thống**; ở sâu bên trong để exception “bubble up” (xem [exceptions.md](exceptions.md)).
4. **`using` declaration** cho scope dài; **`using` statement** khi cần gói nhóm nhỏ/điểm dispose cụ thể. Async resource → **`await using`**.
5. Với **async**, dùng `await foreach`/`await using` cho stream/tài nguyên bất đồng bộ.
6. `lock`: chốt **`Lock`** (.NET 9+) hoặc `object` **riêng tư**; tránh deadlock; `SemaphoreSlim` khi cần `await`. → [threading.md](threading.md).
7. Vòng lặp: **`for`** khi cần index/`Span`/mutate; **`foreach`** khi duyệt enumerable và đọc dễ.
8. Tận dụng **pattern matching** trong `if`/`switch`; đặt case hẹp trước. Đừng lạm dụng `when` có side-effect.
9. TLS: một file, types sau statements; chi tiết entry → [main-function.md](main-function.md).
10. Local `static` khi không cần capture — tránh closure ẩn.

---

### Phụ lục: Bộ ví dụ “từ đơn giản đến nâng cao”

**Validate & chuyển nhánh bằng pattern**

```csharp
static string Describe(object? x)
{
    if (x is null) return "null";
    if (x is int { } i and >= 0 and <= 100) return $"int[0..100]={i}";
    if (x is string { Length: > 0 } s) return $"string({s.Length})";
    return "other";
}
```

**Pipeline đọc file an toàn tài nguyên**

```csharp
static IEnumerable<string> ReadNonEmptyLines(string path)
{
    using var s = File.OpenRead(path);
    using var r = new StreamReader(s);
    string? line;
    while ((line = r.ReadLine()) is not null)
        if (line.Length > 0) yield return line;
}
```

**Tiêu thụ stream async**

```csharp
await foreach (var evt in SubscribeAsync(topic, ct))
{
    if (evt.Severity >= 3)
        await HandleAsync(evt, ct);
}
```

**Vòng lặp hai con trỏ**

```csharp
for (int i = 0, j = a.Length - 1; i < j; i++, j--)
    (a[i], a[j]) = (a[j], a[i]);
```

**`await using` + khối hẹp**

```csharp
async Task ReplaceFileAsync(string path, byte[] payload, CancellationToken ct)
{
    var tmp = path + ".tmp";
    await using (var fs = new FileStream(tmp, FileMode.Create, FileAccess.Write, FileShare.None, 4096, useAsync: true))
        await fs.WriteAsync(payload, ct);
    File.Move(tmp, path, overwrite: true);
}
```
