# Lập trình hàm trong C#

> **Baseline:** .NET **10** / C# **14**. Union làm kiểu kết quả đóng: **C# 15** / `net11.0` — [typesystem.md §18](typesystem.md#18-union-types-c-15).  
> Delegate, closure, expression tree: [delegates-lambdas.md](delegates-lambdas.md). Toán tử chuỗi: [linq.md](linq.md). `record` / `with`: [typesystem.md §8](typesystem.md#8-records-record-class--record-struct). Exception cho lỗi không lường: [exceptions.md](exceptions.md).

C# là ngôn ngữ **đa kiểu**. File này là kiểu **hàm**: hàm thuần, dữ liệu không sửa tại chỗ, và tổ hợp hàm. Class, interface, `virtual` vẫn ở [oop.md](oop.md). Không cần thư viện monad bên ngoài để viết được.

---

## Mục lục

- [Lập trình hàm trong C#](#lập-trình-hàm-trong-c)
  - [Mục lục](#mục-lục)
  - [1. FP trong C# nghĩa là gì](#1-fp-trong-c-nghĩa-là-gì)
  - [2. Hàm thuần và biên tác dụng phụ](#2-hàm-thuần-và-biên-tác-dụng-phụ)
  - [3. Dữ liệu bất biến](#3-dữ-liệu-bất-biến)
    - [3.1 `record`, `with`, `readonly`](#31-record-with-readonly)
    - [3.2 Collection bất biến và view](#32-collection-bất-biến-và-view)
  - [4. Hàm là giá trị](#4-hàm-là-giá-trị)
  - [5. Áp dụng từng phần và tổ hợp](#5-áp-dụng-từng-phần-và-tổ-hợp)
  - [6. Pipeline: map, filter, fold](#6-pipeline-map-filter-fold)
  - [7. Nhánh bằng pattern](#7-nhánh-bằng-pattern)
  - [8. Giá trị vắng và kết quả lỗi](#8-giá-trị-vắng-và-kết-quả-lỗi)
    - [8.1 Null không phải Option](#81-null-không-phải-option)
    - [8.2 `Result` trên .NET 10](#82-result-trên-net-10)
    - [8.3 Union (C# 15)](#83-union-c-15)
  - [9. `SelectMany`, `Task`, và “monad”](#9-selectmany-task-và-monad)
  - [10. Lazy, ghi nhớ, đệ quy](#10-lazy-ghi-nhớ-đệ-quy)
  - [11. Chia sẻ dữ liệu giữa các thread](#11-chia-sẻ-dữ-liệu-giữa-các-thread)
  - [12. Khi không ép kiểu hàm](#12-khi-không-ép-kiểu-hàm)
  - [13. Checklist](#13-checklist)

---

## 1. FP trong C# nghĩa là gì

Ba ý, không phải một bộ cú pháp mới:

| Ý | Trong C# |
|---|---|
| Hàm thuần | Cùng đối số → cùng kết quả; không sửa thứ bên ngoài |
| Giá trị không sửa tại chỗ | `record` + `with`, `string`, `Immutable*`, `Frozen*` |
| Hàm là giá trị | `Func` / `Action`, method group, lambda — [delegates-lambdas.md](delegates-lambdas.md) |

Từ vựng ngôn ngữ khác, và công cụ C# tương ứng:

| Nói trong FP | Viết trong C# |
|---|---|
| map | `Select` |
| filter | `Where` |
| fold / reduce | `Aggregate` (hoặc `Sum` / `CountBy` khi đủ) |
| bind | `SelectMany` |
| partial application | lambda đóng một tham số |
| pipeline | chuỗi method, hoặc `x.Pipe(f)` tự viết |
| algebraic data type | `record` + pattern; **C# 15** `union` / `closed` |

C# **không** có: toán tử pipe `|>`, currying mặc định, tail-call bắt buộc, kiểu `Option` trong BCL, hay cấm side effect ở type system. Ép code “trông Haskell” bằng wrapper đủ lớp thường chậm hơn và khó đọc hơn `record` + LINQ.

**Vì sao / Khi nào:** lõi tính toán (giá, luật, biến đổi DTO) tách khỏi I/O. Biên (HTTP, EF, file, UI) được phép không thuần. Đừng viết lại EF thành fold.

---

## 2. Hàm thuần và biên tác dụng phụ

Hàm **thuần** (pure): kết quả chỉ phụ thuộc tham số. Thay lời gọi bằng giá trị trả về không đổi nghĩa chương trình (referential transparency). Không đọc đồng hồ, không ghi field, không I/O, không `Random` dùng chung.

```csharp
static int Add(int a, int b) => a + b;          // thuần

static int NextId() => _n++;                    // không: sửa static
static string Stamp() => $"{DateTime.UtcNow}";  // không: đồng hồ
static int Roll(Random rng) => rng.Next();      // không: rng đổi trạng thái
```

`Random` truyền vào vẫn không thuần, vì lần gọi sau thấy trạng thái khác. Muốn thuần: nhận `seed` và **trả** generator mới, hoặc giữ `Random` ở biên và chỉ đưa số đã bốc vào lõi.

Biên tác dụng phụ là chỗ được phép không thuần. Lõi nhận dữ liệu đã đọc, trả giá trị hoặc `Result`. Caller ghi DB / gửi HTTP.

```csharp
// biên
OrderDto dto = await db.Orders.FirstAsync(o => o.Id == id, ct);
Result<Invoice> invoice = Price(dto);          // lõi thuần
if (!invoice.IsOk) return Results.BadRequest(invoice.Error);
await db.SaveInvoiceAsync(invoice.OrThrow(), ct);
```

**Pitfall:** method `static` không có nghĩa là thuần. `static` chỉ nghĩa là không có `this`. Field `static` mutable, `HttpContext`, `AsyncLocal` đều là kênh ngầm. Log bên trong “hàm tính giá” làm test phụ thuộc thứ tự và khó chạy lại.

Hàm thuần **được** cấp phát object mới. Cấp phát không phải side effect. Sửa object người gọi còn giữ mới là side effect.

---

## 3. Dữ liệu bất biến

Sửa tại chỗ (`list.Add`, `obj.Name =`) làm người giữ tham chiếu khác thấy đổi — kể cả code không ở cùng thread. Kiểu hàm trả **bản mới** và giữ bản cũ.

`string` đã bất biến: `s + "!"` là chuỗi khác; `s` không đổi. Mảng và `List<T>` thì không.

### 3.1 `record`, `with`, `readonly`

`record` (class hoặc struct) sinh equality theo giá trị và `with`. `with` **sao chép nông** rồi gán property được chỉ định. Bản cũ không đổi.

```csharp
public sealed record Money(decimal Amount, string Currency);

Money usd = new(10m, "USD");
Money more = usd with { Amount = 12m }; // usd.Amount vẫn 10
```

`init` chỉ gán lúc khởi tạo. `readonly` trên field / `readonly struct` cấm gán lại field sau ctor (trừ constructor). `record struct` vẫn là struct: gán biến là copy. Chi tiết equality: [oop.md §4](oop.md#4-equality--tostring), [typesystem.md §8](typesystem.md#8-records-record-class--record-struct).

**Pitfall sao chép nông:** property kiểu reference mutable được **dùng chung**.

```csharp
public sealed record Bag(List<int> Items);

var a = new Bag([1]);
var b = a with { };          // list mới? không — cùng List
b.Items.Add(2);              // a.Items cũng có 2
```

Sửa: `ImmutableArray<int>` / `ImmutableList<int>`, hoặc copy trong `with` (`Items = Items.ToList()` vẫn mutable — chỉ hết alias nếu không ai giữ list cũ). Đừng nhét `List<T>` vào `record` rồi coi là bất biến.

Với record class, `with` gọi cơ chế clone/copy constructor rồi mới gán các member trong initializer; không gọi lại constructor thông thường. Copy constructor kiểm dữ liệu cũ nên **không bảo đảm** giá trị mới hợp lệ. Đặt validation trong init accessor hoặc hàm WithAmount chuyên dụng. Record struct sao chép giá trị.

### 3.2 Collection bất biến và view

Ba thứ khác nhau. Đừng đổi tên cho nhau.

| | Sửa phần tử | Cập nhật | Đọc |
|---|---|---|---|
| `List<T>` / mảng | tại chỗ | rẻ | rẻ |
| `IReadOnlyList<T>` | API không cho; **gốc vẫn sửa được** nếu còn `List` | — | view |
| `ImmutableList<T>` | không | cấu trúc cây mới, chia sẻ nút | index O(log n) |
| `ImmutableArray<T>` | không | thay đổi thường tạo/copy mảng mới | index O(1) |
| `FrozenDictionary` / `FrozenSet` | không | không có cập nhật từng phần — xây lại | rất nhanh sau khi xây |

Persistent collection (`Immutable*`): `Add` không sửa bản cũ. Nhiều lần `Add` đơn lẻ trên `ImmutableList` đắt — dùng `ToBuilder()`, sửa builder, `ToImmutable()` một lần. Package và semantics: [collections-generics.md §3](collections-generics.md#3-collections-bất-biến-systemcollectionsimmutable).

```csharp
using System.Collections.Immutable;

ImmutableList<int> empty = ImmutableList<int>.Empty;
ImmutableList<int> one = empty.Add(1); // empty vẫn rỗng

ImmutableArray<int> nums = [1, 2, 3]; // collection expression, .NET 8+
```

`Frozen*` xây một lần từ dữ liệu đã xong, rồi chỉ đọc. Không phải undo. Cache đọc nhiều: Frozen. Luồng sự kiện / snapshot từng bước: `Immutable*`.

Collection expression `[1, 2, 3]` **không** bất biến theo mặc định. Kiểu đích quyết định: `List<int> x = [1, 2, 3]` vẫn `Add` được. [collections-generics.md §7](collections-generics.md#7-collection-expressions-c-12--args-c-15).

---

## 4. Hàm là giá trị

Func<T,TResult> nhận đối số và trả giá trị; Action<T> trả void. Predicate<T> có cùng chữ ký với Func<T,bool> nhưng là **delegate type khác**, không có implicit conversion giữa hai instance; có thể bọc lời gọi trong lambda.

```csharp
Func<int, int> square = static x => x * x;
int nine = square(3);

IEnumerable<int> doubled = nums.Select(static n => n * 2);
```

`static` trên lambda **cấm capture**. Capture biến ngoài thì lambda thành object trên heap, sống lâu bằng delegate. Biến vòng `for` bị mọi lambda nhìn giá trị cuối: [delegates-lambdas.md §7](delegates-lambdas.md#7-closure--capturing--lifetime--heap).

Method group (`Select(Parse)`) không cấp phát closure nếu không cần target, nhưng overload resolution khác lambda. Local function `static` cũng không capture — compiler dễ inline hơn lambda khi hàm chỉ dùng một chỗ: [delegates-lambdas.md §11](delegates-lambdas.md#11-local-functions-vs-lambda).

Hàm bậc cao nhận hoặc trả hàm:

```csharp
static IEnumerable<T> Where<T>(IEnumerable<T> src, Func<T, bool> pred)
{
    foreach (var x in src)
        if (pred(x)) yield return x;
}
```

Đó là `Enumerable.Where`. Đừng viết lại trừ khi đo được delegate là nút thắt; lúc đó dùng `foreach` / `Span`.

Expression tree (`Expression<Func<…>>`) **không** phải hàm chạy được cho đến khi `Compile()` hoặc provider dịch. EF nhận cây, không nhận `Func` đã biên dịch. [delegates-lambdas.md §10](delegates-lambdas.md#10-expression-trees-vs-funcaction), [linq.md §6](linq.md#6-iqueryable--biểu-thức-expression).

---

## 5. Áp dụng từng phần và tổ hợp

C# không curry sẵn. `void M(int a, int b)` không tự thành `Func<int, Func<int, int>>`. Áp dụng từng phần = đóng tham số đã biết trong lambda:

```csharp
static Func<int, int> Add(int a) => b => a + b;

Func<int, int> add10 = Add(10);
int result = add10(3); // 13
```

Mỗi `Add(10)` cấp phát một closure giữ `a`. Trong vòng nóng, đừng tạo `Func` mới mỗi phần tử nếu lambda không cần trạng thái — dùng `static` lambda hoặc method group.

Tổ hợp `g ∘ f` nghĩa là chạy `f` trước, rồi `g`:

```csharp
static Func<T, TResult> Compose<T, TMid, TResult>(
    Func<T, TMid> f, Func<TMid, TResult> g) => x => g(f(x));

Func<string, int> length = Compose<string, string, int>(
    static s => s.Trim(),
    static s => s.Length);
```

Thứ tự ngược với “đọc từ trái sang phải”. Pipeline thì đọc xuôi:

```csharp
static TResult Pipe<T, TResult>(this T value, Func<T, TResult> f) => f(value);

int n = "  hi  ".Pipe(static s => s.Trim()).Pipe(static s => s.Length);
```

`Pipe` là đường truyền giá trị, không phải thư viện. Chuỗi LINQ (`Where` rồi `Select`) đã là pipeline trên **chuỗi**; `Pipe` là pipeline trên **một giá trị**. Đừng dựng cả hai cho cùng một việc.

Curry đủ tham số (`Func<A, Func<B, Func<C, R>>>`) chỉ đáng khi API thật sự xây hàm dần (callback cấu hình). Hàm nghiệp vụ ba tham số: giữ `(a, b, c)` hoặc `record` một tham số. Tuple tham số `(a, b)` tổ hợp dễ hơn curry.

---

## 6. Pipeline: map, filter, fold

| Bước | LINQ | Không làm |
|---|---|---|
| Đổi từng phần tử | `Select` | `Select` để `Add` vào list ngoài |
| Giữ một số phần tử | `Where` | `Where` rồi dựa vào side effect để “lọc” |
| Bẹt chuỗi con | `SelectMany` | `Select` trả `IEnumerable` rồi quên bẹt |
| Gộp thành một giá trị | `Aggregate`, `Sum`, `CountBy` | `Aggregate` trên `IQueryable` — thường không dịch |
| Sắp | `OrderBy` / `ThenBy` | `OrderBy` lần hai và tưởng đó là then |

```csharp
decimal total = lines
    .Where(static l => l.Quantity > 0)
    .Select(static l => l.Quantity * l.Price)
    .Aggregate(0m, static (sum, x) => sum + x);
```

`Aggregate` không seed trên dãy rỗng thì ném. Seed `0m` cho tổng tiền. Đếm theo khóa không cần giữ phần tử: `CountBy` / `AggregateBy` — [linq.md §4.4](linq.md#44-grouping-groupby-tolookup).

**Deferred:** `Where`/`Select` chưa chạy cho đến khi duyệt. Gọi hai lần là chạy hai lần. Hàm “thuần” trên pipeline vẫn thuần **mỗi lần chạy**, nhưng nguồn không thuần (stream, EF) thì lần hai đổi nghĩa. Chốt bằng `ToList` / `ToArray` / `ToImmutableArray` khi rời biên. [linq.md §2](linq.md#2-deferred-vs-immediate-execution).

**Pitfall:** `Select` chứa `list.Add` hoặc `Console.WriteLine` biến map thành vòng lặp giấu. Người đọc bỏ `Select` vì “không dùng kết quả” thì mất side effect. Muốn side effect: `foreach` ở biên, nhìn thấy.

`IQueryable` chỉ thuần ở mức “chưa gửi SQL”. Lambda trong `Where` là cây biểu thức, không phải hàm C# tùy ý. Method riêng của bạn trong predicate EF thường không dịch.

---

## 7. Nhánh bằng pattern

`if` gán cờ (`bool ok`, `object? err`) dễ quên nhánh. Biểu thức `switch` trả giá trị và, với kiểu đóng, compiler bắt đủ nhánh.

```csharp
static string Band(int score) => score switch
{
    < 0 or > 100 => throw new ArgumentOutOfRangeException(nameof(score)),
    < 50 => "fail",
    < 80 => "pass",
    _ => "high"
};
```

`throw` trong nhánh là biểu thức. Input sai kiểu dữ liệu (điểm ngoài 0–100) là bug — ném được. “Không tìm thấy đơn” là kết quả nghiệp vụ — trả kiểu ở mục 8, đừng ném để điều khiển luồng thường.

Pattern trên `record` positional:

```csharp
static decimal Discount(Money m) => m switch
{
    { Currency: "USD", Amount: > 100m } => 5m,
    { Currency: "USD" } => 0m,
    _ => 0m
};
```

Nhánh `_` có thể che currency lạ. Union/closed giúp compiler xét tập case; enum vẫn có giá trị không đặt tên (do cast/deserialize), nên xử lý giá trị ngoài danh sách. Switch expression thiếu case thường phát warning, không tự bảo đảm build fail. [Patterns](statements.md#6-selection-statements-ifelse-switch).

---

## 8. Giá trị vắng và kết quả lỗi

### 8.1 Null không phải Option

`T?` (nullable reference, bật NRT) là cảnh báo biên dịch, không phải giá trị `Some`/`None` mà runtime kiểm tra. Gán `null!` hoặc tắt nullable là xuyên thủng. `string?` vẫn là một tham chiếu có thể null, không có `Map` trong BCL.

Dùng `T?` khi vắng mặt là **một** trạng thái rõ (không có hàng). Đừng nhồi thêm nghĩa “lỗi mạng” vào cùng `null`.

```csharp
User? user = users.FirstOrDefault(u => u.Id == id);
if (user is null) return Results.NotFound();
```

`FirstOrDefault` trên `int` trả `0` khi thiếu — `0` vừa là giá trị vừa là “không có”. Lúc đó `int?` hoặc `bool` + out, không phải `default`. [typesystem.md §10](typesystem.md#10-nullable-reference-types-nrt).

### 8.2 `Result` trên .NET 10

Lỗi **dự kiến** (validate, parse, hết hàng) nên thành giá trị, để chữ ký hàm nói thật. Lỗi **không** dự kiến (null nội bộ, đĩa hỏng giữa invariant) vẫn là exception. [exceptions.md §1](exceptions.md#1-tổng-quan--triết-lý).

```csharp
public readonly record struct Result<T>
{
    private readonly T _value;
    public string? Error { get; }
    public bool IsOk { get; }

    private Result(T value, string? error, bool ok)
    {
        _value = value;
        Error = error;
        IsOk = ok;
    }

    public static Result<T> Ok(T value) => new(value, null, true);
    public static Result<T> Fail(string error)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(error);
        return new(default!, error, false);
    }

    public T OrThrow() => IsOk
        ? _value
        : throw new InvalidOperationException(Error ?? "Result chưa được khởi tạo.");
}

static Result<int> ParsePort(string text) =>
    int.TryParse(text, out int port) && port is > 0 and < 65536
        ? Result<int>.Ok(port)
        : Result<int>.Fail("port");
```

Readonly record struct tránh allocation object khi không boxing. `_value = default!` trong Fail không được đọc **khi IsOk là false**; dùng OrThrow làm cửa truy cập. `default(Result<T>)` cũng có IsOk=false nhưng Error=null: cần quy ước rõ trạng thái chưa khởi tạo hoặc dùng class nếu phải cấm nó. Không có exhaustiveness cho struct này.

Đừng bọc `Result` bằng exception (“Fail thì ném”) rồi bắt ngay ở caller — mất chữ ký. Đừng `Result<Result<T>>` trừ khi hai lớp lỗi thật sự khác nhau.

Async: `Task<Result<T>>`, không phải `Result<Task<T>>`. Task là việc sẽ xong; Result là nghĩa của giá trị sau khi xong.

### 8.3 Union (C# 15)

`union` là tập đóng thật: `switch` thiếu case thì cảnh báo. Không thuộc `net10.0`.

```csharp
public readonly record struct Ok<T>(T Value);
public readonly record struct Fail(string Message);
public union Result<T>(Ok<T>, Fail);

static string Show<T>(Result<T> r) => r switch
{
    Ok<T> ok => ok.Value?.ToString() ?? "",
    Fail f => f.Message
};
```

Cây OOP chung member: `closed` thay vì union. [typesystem.md §18](typesystem.md#18-union-types-c-15), [oop.md §2.6](oop.md#26-closed-hierarchies-c-15). JSON của union **không** tự thêm `$type` — đừng dùng union làm hợp đồng HTTP mới nếu hai case cùng hình JSON.

---

## 9. `SelectMany`, `Task`, và “monad”

`SelectMany` trên `IEnumerable<T>` là bind: hàm trả một chuỗi, rồi bẹt.

```csharp
IEnumerable<(int X, int Y)> pairs =
    xs.SelectMany(x => ys.Select(y => (x, y)));

// query syntax là cùng bind
var pairs2 = from x in xs
             from y in ys
             select (x, y);
```

Query syntax `from` thứ hai **là** `SelectMany`, không phải `Select`. [linq.md §4.2](linq.md#42-projection-select-selectmany).

`Task` không cùng luật với `IEnumerable`:

| | `IEnumerable` / iterator | `Task` |
|---|---|---|
| Lúc tạo | iterator chưa chạy tới khi duyệt; source khác tùy implementation | Task.Run đã schedule, async method bắt đầu ngay; new Task(delegate) chưa Start thì chưa chạy |
| Lần hai | chạy lại (deferred) | cùng một việc đã lên lịch |
| “Bind” | `SelectMany` | `await` (và `ContinueWith`, tránh nếu có `await`) |

Đừng mô tả `await` là “LINQ của task” rồi `Select` trên `Task` bằng thư viện để né `async`. `async`/`await` đã là cú pháp bind của việc bất đồng bộ. [async.md](async.md).

Nullable không có `SelectMany` trong ngôn ngữ. Viết `?.` cho một bước; chuỗi dài thì `if (x is null) return` hoặc hàm cục bộ. Đừng kéo thư viện monad vào service chỉ để `Map` một `string?`.

**Pitfall:** gọi kiểu hàm là “monad” không làm code an toàn hơn. An toàn đến từ dữ liệu không sửa tại chỗ, chữ ký không giấu lỗi, và biên I/O hẹp.

---

## 10. Lazy, ghi nhớ, đệ quy

`Lazy<T>` tính một lần, rồi giữ kết quả. Mặc định khóa: nhiều thread cùng `.Value` chỉ chạy factory một lần.

```csharp
Lazy<Regex> rule = new(static () => new Regex("^[a-z]+$", RegexOptions.CultureInvariant));
bool ok = rule.Value.IsMatch(text);
```

Factory không thuần (đọc file, gọi mạng) thì `Lazy` chỉ chạy một lần — lỗi cũng bị **giữ**. Lần sau `.Value` ném lại exception đã cache (`ExecutionAndPublication`). Muốn thử lại: đừng dùng `Lazy` cho I/O dễ fail, hoặc tạo `Lazy` mới.

Iterator (`yield`) cũng lạnh: body chạy lúc `foreach`, không lúc gọi hàm trả `IEnumerable`. [methods.md §13](methods.md#13-iterator-method--yield-return--yield-break).

Ghi nhớ (memo) một hàm thuần: cùng key → cùng giá trị, cache được. `ConcurrentDictionary.GetOrAdd` có thể chạy factory **hai lần** khi tranh — factory phải thuần và rẻ, hoặc dùng `GetOrAdd` với `Lazy<T>` bên trong nếu factory đắt.

Đệ quy trên cây **nông** thì rõ. C# **không** bảo đảm tail-call. Hàm đuôi vẫn có thể `StackOverflowException` khi sâu hàng chục nghìn.

```csharp
static int Fact(int n, int acc = 1) =>
    n <= 1 ? acc : Fact(n - 1, n * acc); // đuôi, vẫn có thể tràn stack
```

Dãy dài: vòng hoặc `Aggregate`, không đệ quy. Cây AST độ sâu người viết: đệ quy ổn. Parser trên input độc hại: đệ quy là đường DoS — giới hạn độ sâu hoặc dùng stack tường minh.

Baseline .NET 10 cho phép `Func<Span<T>, TResult>` nhờ `allows ref struct` trên delegate BCL. Lambda có thể nhận Span qua tham số, nhưng **không được capture Span local**, dù static hay không. Span không trở thành IEnumerable chỉ nhờ dùng delegate. [Lifetime](memory-spans.md).

---

## 11. Chia sẻ dữ liệu giữa các thread

Bản đã xây xong và **không còn đường sửa** thì nhiều thread đọc không cần `lock` trên từng phần tử. Việc cần hàng rào là **công bố tham chiếu**: thread khác phải thấy object hoàn chỉnh, không thấy nửa khởi tạo.

```csharp
private ImmutableList<int> _published = ImmutableList<int>.Empty;

void Publish(ImmutableList<int> next) => Volatile.Write(ref _published, next);
ImmutableList<int> Snapshot() => Volatile.Read(ref _published);
```

Mẫu dùng ImmutableList là reference type để publish bằng Volatile. **Không khai báo `volatile ImmutableArray<T>`**: compiler cấm vì đó là struct. Với ImmutableArray, dùng lock, wrapper reference hoặc ImmutableInterlocked phù hợp. Bất biến collection là nông: element mutable vẫn cần đồng bộ. Builder không tự thread-safe.

`FrozenDictionary` sau `ToFrozenDictionary()` cùng luật: xây trên một thread, công bố, rồi chỉ đọc. [collections-generics.md §4.2](collections-generics.md#42-frozen-systemcollectionsfrozen).

Hàm thuần không `lock`. Khóa nằm ở biên khi lấy snapshot hoặc khi hàng đợi message. `Channel` đưa bản ghi (`record`) sang worker: worker không sửa message. [threading.md §7](threading.md#7-mẫu-producerconsumer).

`AsyncLocal` không phải state hàm. Nó là kênh ngầm theo execution context — làm hàm trông thuần nhưng phụ thuộc request. [threading.md §5](threading.md#5-threadstatic--threadlocalt-vs-asynclocalt).

---

## 12. Khi không ép kiểu hàm

- Vòng `Span` / `for` trên hot path đã đo. Delegate + iterator cấp phát và chặn một số tối ưu.
- EF và `IQueryable`: thành phần là biểu thức dịch được, không phải `Func` đã capture `HttpClient`.
- UI: một số API bắt buộc sửa property (`INotifyPropertyChanged`). Snapshot `record` bên dưới, gán control ở biên.
- `Action` / `void` trong pipeline “thuần” là dấu side effect đang trốn.
- Thư viện FP bên ngoài (LanguageExt và tương tự) không có trong BCL. Dùng thì cả codebase dùng một kiểu `Option`/`Either`; đừng trộn với `T?` và `Result<T>` tự viết.
- Native AOT: cây biểu thức `Compile()` và phản xạ trong thư viện FP dễ cảnh báo trim. Lõi `record` + method thường thì không.

---

## 13. Checklist

```text
[ ] Lõi tính toán không đọc đồng hồ, static mutable, hay I/O
[ ] record không chứa List/mảng rồi coi with là bản sao sâu
[ ] Invariant tiền không chỉ nằm trong ctor nếu có with
[ ] Select/Where không giấu side effect
[ ] Pipeline deferred được materialize trước khi ra khỏi biên
[ ] Lỗi dự kiến là Result / union; lỗi lập trình là exception
[ ] Task<Result<T>>, không Result<Task<T>>
[ ] Đệ quy sâu có trần hoặc đổi thành vòng
[ ] Snapshot đưa sang thread khác đã xây xong rồi mới công bố tham chiếu
[ ] Không thêm thư viện monad cho một chỗ Map
```

**Tóm lại:** hàm thuần trên dữ liệu không sửa tại chỗ, tổ hợp bằng `Func` và LINQ, lỗi dự kiến nằm trong kiểu trả về. I/O và UI đứng ở biên. C# không biến những quy tắc đó thành lỗi biên dịch — kỷ luật nằm ở chữ ký và ở chỗ không sửa object dùng chung.
