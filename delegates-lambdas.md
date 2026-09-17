# Delegate & Lambda

> **Baseline:** .NET **10** / C# **14** — modifier `ref`/`in`/`out`/`scoped` trên lambda không cần kiểu tường minh (§6.4).  
> Đọc thêm: *Events*, *LINQ*, *Type System*, *Methods*, *Async*.

Delegate là kiểu *tham chiếu tới phương thức*; lambda là cú pháp tạo function. Hiểu **multicast**, **closure lifetime**, và **expression tree vs `Func`** tránh leak, exception nuốt handler, và LINQ-to-Objects chạy nhầm thành SQL.

---

## Mục lục

1. [Delegate là gì?](#1-delegate-là-gì)
2. [Khai báo \& sử dụng delegate](#2-khai-báo--sử-dụng-delegate)
3. [Delegate dựng sẵn: `Action<>`, `Func<>`, `Predicate<T>`](#3-delegate-dựng-sẵn-action-func-predicatet)
4. [Multicast delegate \& Invocation List](#4-multicast-delegate--invocation-list)
5. [Method group \& overload resolution](#5-method-group--overload-resolution)
6. [Lambda expressions](#6-lambda-expressions)
   6.1 [Expression lambda vs Statement lambda](#61-expression-lambda-vs-statement-lambda) · 6.2 [Anonymous methods](#62-anonymous-methods) · 6.3 [Async lambda](#63-async-lambda) · 6.4 [Parameter modifiers không cần kiểu tường minh (C# 14)](#64-parameter-modifiers-không-cần-kiểu-tường-minh-c-14)
7. [Closure \& Capturing — lifetime \& heap](#7-closure--capturing--lifetime--heap)
8. [Variance trong delegate (`in`/`out`)](#8-variance-trong-delegate-inout)
9. [Ref/Out/In trong delegate \& ref return](#9-refoutin-trong-delegate--ref-return)
10. [Expression Trees vs `Func`/`Action`](#10-expression-trees-vs-funcaction)
11. [Local functions vs Lambda](#11-local-functions-vs-lambda)
12. [Delegates vs Events](#12-delegates-vs-events)
13. [Interop \& Function Pointers](#13-interop--function-pointers)
14. [Hiệu năng \& Best Practices](#14-hiệu-năng--best-practices)
15. [Cheat sheet \& ví dụ tổng hợp](#15-cheat-sheet--ví-dụ-tổng-hợp)

---

## 1. Delegate là gì?

- **Delegate** là *kiểu tham chiếu tới phương thức* (type-safe function pointer).
- Một biến delegate trỏ tới **phương thức** (static/instance) có **chữ ký** tương ứng.
- Dùng làm *callback*, *event handler*, *pipeline*…

```csharp
public delegate int Transformer(int x);

int Square(int x) => x * x;

Transformer t = Square;
int r = t(5); // hoặc t.Invoke(5) → 25
```

**WHY kiểu riêng:** compiler kiểm chữ ký; multicast (`+`) và event dựa trên cùng hạ tầng `MulticastDelegate`. Function pointer `delegate*` (unsafe) nhanh hơn nhưng không multicast/GC-track.

---

## 2. Khai báo & sử dụng delegate

### 2.1 Khai báo custom delegate

```csharp
public delegate bool Filter<in T>(T item); // có thể khai báo variance
```

- Delegate là **sealed class** kế thừa `System.MulticastDelegate`.
- Thuộc tính hữu ích: `.Target` (instance được capture — `null` nếu static), `.Method` (MethodInfo).

### 2.2 Gán & gọi

```csharp
Filter<string> f = s => !string.IsNullOrWhiteSpace(s);
bool ok = f("hello");
```

Gọi `f.Invoke(...)` tương đương `f(...)`. **Null:** `f?.Invoke(...)` — đừng `f()` khi có thể null (`NullReferenceException`).

### 2.3 So sánh & bằng nhau

- Hai delegate bằng nhau khi **cùng Target + Method** (với multicast: cùng chuỗi invocation).
- `delegateA == delegateB` kiểm tra semantically equal.

```csharp
Action a = M;
Action b = M;
Console.WriteLine(a == b); // True (cùng method group static)

var o = new C();
Action c1 = o.Inst;
Action c2 = o.Inst;
Console.WriteLine(c1 == c2); // True — cùng target + method
```

---

## 3. Delegate dựng sẵn: `Action<>`, `Func<>`, `Predicate<T>`

- `Action` không trả về: `Action`, `Action<T1>`, …, `Action<T1,...,T16>`
- `Func<..., TResult>`: tham số bất kỳ + **kết quả là phần tử cuối**.
- `Predicate<T>` tương đương `Func<T, bool>` (di sản, dùng trong `List<T>.Find`…)

```csharp
Action<string> log = Console.WriteLine;
Func<int,int,int> add = (a,b) => a + b;
Predicate<string> nonEmpty = s => !string.IsNullOrEmpty(s);
```

> Khuyên dùng `Func`/`Action` thay vì tạo delegate mới, trừ khi bạn cần **ý nghĩa tên** rõ ràng hoặc **variance** khác.

`Predicate<T>` **không** gán chéo implicit sang `Func<T,bool>` (khác kiểu delegate) — cần `new Func<T,bool>(pred)` hoặc lambda bọc.

---

## 4. Multicast delegate & Invocation List

- Delegate có thể **multicast** (kết hợp nhiều handler): sử dụng `+`, `-` hoặc `+=`, `-=`.
- Khi gọi, **thứ tự thực thi** theo thứ tự thêm vào (FIFO).
- Giá trị trả về của multicast = giá trị handler **cuối cùng**; handler giữa bị bỏ return (side-effect vẫn chạy).
- Nếu một handler ném exception → lời gọi **dừng tại đó** — handler sau **không** chạy.

**WHY:** event nhiều subscriber; pipeline logging. **Không** dùng multicast `Func<T>` nếu mọi handler đều cần đóng góp kết quả — iterate `GetInvocationList()`.

```csharp
Action all = null!;
all += () => Console.WriteLine("A");
all += () => throw new Exception("Boom");
all += () => Console.WriteLine("B"); // không tới nếu không catch

foreach (Action h in all.GetInvocationList())
{
    try { h(); }
    catch (Exception ex) { Console.Error.WriteLine(ex.Message); }
}
```

**Semantics combine/remove:**

- `+=` / `Delegate.Combine` tạo instance **mới** (delegate bất biến). Race: copy local rồi `+=` không atomic — event dùng `add`/`remove` accessor (thường `lock` hoặc `Interlocked`).
- `-=` gỡ **một** khớp đầu tiên (cùng method+target). Lambda **khác instance** mỗi lần viết `() => ...` — `-=` không gỡ được nếu không giữ cùng reference.

```csharp
EventHandler h = (_, _) => Console.WriteLine("x");
btn.Click += h;
btn.Click -= h;           // OK

btn.Click += (_, _) => { }; 
btn.Click -= (_, _) => { }; // KHÔNG gỡ — hai lambda khác nhau
```

**Invocation list & thread:**

```csharp
var snapshot = handler;          // copy tham chiếu
snapshot?.Invoke(this, args);    // tránh NRE nếu thread khác -= hết giữa check và call
```

Vẫn có thể handler bị gỡ *sau* copy nhưng *trước* invoke từng mục — snapshot list ổn định cho lần gọi đó.

**Return multicast:**

```csharp
Func<int> f = () => 1;
f += () => 2;
Console.WriteLine(f()); // 2 — 1 bị bỏ
```

---

## 5. Method group & overload resolution

**Method group** có thể gán trực tiếp cho delegate nếu chữ ký phù hợp:

```csharp
int Parse(string s) => int.Parse(s);
Func<string,int> f = Parse; // method group conversion
```

- Khi có **nhiều overload**, compiler chọn theo *overload resolution*.
- Nếu mơ hồ, cần **cast** đích rõ ràng: `(Func<string,int>)Parse`.
- Method group **không** cấp phát closure; `x => Parse(x)` có thể cấp phát lambda (tùy capture). Ưu tiên method group trên hot path.

---

## 6. Lambda expressions

```csharp
// Lambda cơ bản
(int x) => x * x
x => x * x              // suy kiểu
(string s) => { Console.WriteLine(s); return s.Length; } // statement lambda
```

> **C# 9**: *static lambda* — `static x => ...` không capture được biến ngoài.  
> **C# 10**: *Lambda improvements* — **type tự nhiên** (natural type), có thể **chỉ định kiểu trả về** và **gán attribute** cho lambda.  
> **C# 12**: hỗ trợ **default parameter** cho lambda.  
> **C# 14**: **modifier** (`ref`/`in`/`out`/`ref readonly`/`scoped`) trên tham số lambda **không** bắt buộc ghi kiểu tường minh — xem [§6.4](#64-parameter-modifiers-không-cần-kiểu-tường-minh-c-14).

### 6.1 Expression lambda vs Statement lambda

- **Expression lambda**: thân là *biểu thức* → trả về kết quả expression. Có thể convert sang `Expression<T>` **nếu** thân là expression (không statement).
- **Statement lambda**: thân là *khối lệnh* `{ ... }` → có `return`/`await`/`try-catch`. **Không** thành expression tree.

### 6.2 Anonymous methods

Cú pháp cũ (C# 2.0), vẫn hữu dụng khi cần `goto`, nhiều `return`, hoặc không muốn khai báo tham số:

```csharp
delegate(int x) { return x * x; }
```

### 6.3 Async lambda

- Gắn từ khóa `async`: `async x => { await ...; }`
- Kiểu trả về: `Task` / `Task<T>` / **chỉ `void` cho event handler** (không nên dùng `async void` trong các trường hợp khác).
- Không thể vừa `async` vừa `yield` (iterator).

```csharp
Func<Task<int>> f = async () => { await Task.Delay(10); return 42; };
```

Exception trong `async void` lambda event: giống `async void` method — [async.md](async.md) / [exceptions.md](exceptions.md).

### 6.4 Parameter modifiers không cần kiểu tường minh (C# 14)

Trước C# 14, nếu tham số lambda có modifier (`ref`, `out`, `in`, `ref readonly`, `scoped`) thì **bắt buộc** khai báo kiểu đầy đủ. C# 14 cho phép **suy kiểu** từ delegate đích:

```csharp
delegate bool TryParse<T>(string text, out T result);

// C# 14 — không cần ghi string / int
TryParse<int> parse1 = (text, out result) => int.TryParse(text, out result);

// Trước C# 14 — bắt buộc kiểu tường minh khi có modifier
TryParse<int> parse2 = (string text, out int result) => int.TryParse(text, out result);
```

**WHY:** `TryParse`/`TryGetValue` pattern dài dòng; modifier đã có trên delegate — lặp kiểu là noise. Compiler lấy kiểu từ *target* (`TryParse<int>`), không từ natural type độc lập.

Các modifier được hỗ trợ theo kiểu này: `ref`, `in`, `out`, `ref readonly`, `scoped`.

```csharp
delegate void Update(ref int value);
Update bump = (ref x) => x++;           // C# 14

delegate void ReadOnlyView(in Matrix4x4 m);
ReadOnlyView use = (in m) => { /* ... */ };
```

**Ngoại lệ:** modifier **`params`** trên lambda vẫn yêu cầu **danh sách tham số typed tường minh** (không suy kiểu được như trên).

**Pitfall:** không có target type thì không suy được — `var f = (out x) => ...` lỗi. Phải `TryParse<int> f = ...` hoặc kiểu tường minh.

Kết hợp ghi chú phiên bản trước:

```csharp
// C# 10 — natural type + return type
var square = int (int x) => x * x;

// C# 10 — attribute trên lambda / tham số
Handler h = [Obsolete] (int x) => x;

// C# 12 — default parameter
var greet = (string name = "world") => $"Hello, {name}";

// C# 14 — modifier + suy kiểu
TryParse<double> parseD = (text, out result) => double.TryParse(text, out result);
```

Chi tiết `ref`/`out`/`in` trên method: [methods.md](./methods.md). Delegate với `ref` return: [§9](#9-refoutin-trong-delegate--ref-return).

---

## 7. Closure & Capturing — lifetime & heap

- Lambda/anonymous method có thể **capture biến ngoài phạm vi** (tạo *closure*).
- Capture **biến**, không capture **giá trị** → thay đổi sau đó sẽ phản ánh trong lambda.
- Compiler sinh **class closure ẩn**, lưu các biến captured trên **heap** → có thể gây **cấp phát**.

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    int copy = i;                 // ✅ dùng bản sao để tránh bẫy
    actions.Add(() => Console.WriteLine(copy));
}
foreach (var a in actions) a();   // 0 1 2
```

**WHY heap:** local thường stack; lambda có thể chạy **sau khi** method return (callback, event, `Task.Run`) → biến phải sống lâu hơn frame → display class trên heap.

**Lifetime — pitfall lớn:**

1. **Giữ object sống:** lambda đăng ký event/static cache capture `this` / form / `HttpContext` → memory leak cho tới khi unsubscribe.
2. **`this` implicit:** `() => _field` capture instance. `static` lambda + truyền field như tham số, hoặc local function `static`.
3. **Nhiều biến một display class:** capture `a` và `b` → cả hai lên heap, kể cả biến chỉ dùng sync (compiler gom). Tách method/`static` để thu hẹp.
4. **Loop variable:** `for` **một** biến `i` — mọi lambda thấy giá trị *cuối* nếu không `copy`. `foreach` C# 5+: mỗi vòng biến riêng.
5. **`ref struct` / `ref` local / `Span`:** không capture (ref safety).
6. **Async + capture:** state machine đã trên heap; thêm closure lồng tăng object.

```csharp
sealed class LeakyPublisher
{
    public event Action? Tick;
    public void Start(Widget w)
    {
        // Widget sống bằng lifetime event của publisher
        Tick += () => w.Refresh();
    }
}

void Safe(Widget w)
{
    Action h = () => w.Refresh();
    pub.Tick += h;
    // ...
    pub.Tick -= h; // cùng instance
}
```

**Static lambda**: `static () => ...` **không capture** gì → không cấp phát closure; lỗi compile nếu lỡ dùng biến ngoài.

```csharp
int k = 2;
var good = new[] { 1, 2, 3 }.Select(static x => x * 2);
// var bad = Enumerable.Range(0, 3).Select(static x => x * k); // lỗi
var ok = Enumerable.Range(0, 3).Select(x => x * k); // capture k → heap
```

**Cache delegate:** `private static readonly Func<int,int> Twice = static x => x * 2;` tránh alloc mỗi lần gọi.

---

## 8. Variance trong delegate (`in`/`out`)

Bạn có thể khai báo delegate generic **covariant/contravariant**:

```csharp
public delegate TResult Factory<out TResult>();          // covariant kết quả
public delegate void Consumer<in T>(T item);             // contravariant tham số
```

- Tương tự như `Func<in ..., out TResult>` và `Action<in ...>`.
- Cho phép *thay thế kiểu* thuận tiện khi kế thừa (ví dụ `IAnimal` ↔ `Cat`).

```csharp
Func<string> g = () => "hi";
Func<object> f = g; // OK — out TResult covariant
```

---

## 9. Ref/Out/In trong delegate & ref return

- Delegate có thể nhận **`ref`/`out`/`in`** parameters, nhưng phải **khớp đúng** ở method/lambda gán vào.
- **C# 14:** lambda có thể ghi `(text, out result) => ...` **không** cần kiểu tường minh nếu delegate đích đã biết kiểu — xem [§6.4](#64-parameter-modifiers-không-cần-kiểu-tường-minh-c-14).
- Delegate có thể **trả về `ref`**:

```csharp
public delegate ref int RefPicker(int[] data);

static ref int First(ref int a) => ref a; // ví dụ ref return

RefPicker pick = (int[] arr) => ref arr[0];
ref int r = ref pick(new[] { 10, 20 });
r = 99;  // thay đổi phần tử mảng
```

- **Không** thể `async` với `ref` return.
- Cần thận trọng vòng đời đối tượng được trả `ref` (không trả ref tới local).

---

## 10. Expression Trees vs `Func`/`Action`

- **Lambda** có thể “nâng” thành **biểu thức** dùng trong metaprogramming/ORM:

```csharp
using System.Linq.Expressions;

Expression<Func<int,int>> expr = x => x * x;
Func<int,int> compiled = expr.Compile();
int r = compiled(5); // 25
```

Hai thế giới **không** hoán đổi implicit:

| | `Func<T>` / `Action` (delegate) | `Expression<Func<T>>` |
|---|---|---|
| Là gì | IL gọi được ngay | Cây dữ liệu (`Body`, `Parameters`) |
| Lambda statement `{ }` | Được | **Không** (chỉ expression lambda) |
| `async` | Được | Không |
| Capture | Closure runtime | Cây có thể chứa `Constant` tới object captured |
| EF Core / LINQ to SQL | Client eval hoặc **không dịch** nếu nhận `Func` | Provider **visit** cây → SQL |
| Chi phí | 1 delegate (+ closure) | Build tree; `Compile()` = JIT method |

**WHY nhầm `Func` vào `IQueryable`:** `Where(Func)` chọn overload `IEnumerable` → kéo **toàn bộ** bảng rồi lọc in-memory. `Where(Expression<Func>)` dịch server-side.

```csharp
IQueryable<User> q = db.Users;

q = q.Where(u => u.Active);           // Expression — SQL WHERE
Func<User, bool> pred = u => u.Active;
q = q.Where(pred);                    // thường chuyển IEnumerable — PITFALL
```

**Giới hạn cây:** không `await`, không `ref`, không assignment, không `??` / pattern đầy đủ tùy version provider. Method không dịch được → runtime exception hoặc client eval (EF cũ).

Sửa/chắp biểu thức động: `ExpressionVisitor` hoặc thư viện (`PredicateBuilder`). `Compile()` cache lại — đừng compile mỗi request.

```csharp
Expression<Func<User, bool>> Active = u => u.Active;
// Compose (ý tưởng): AndAlso hai Expression.Lambda
```

`Expression` **không** thay delegate trong hot path thuần CPU — tree + compile đắt hơn lambda thường.

**Compile & cache:**

```csharp
static readonly Func<User, bool> Compiled =
    ((Expression<Func<User, bool>>)(u => u.Active)).Compile();
```

Mỗi `Compile()` ≈ sinh method động. ASP.NET: compile một lần (static/lazy), không per-request. EF **không** cần `Compile()` — provider visit tree.

**Pitfall statement lambda:** `Expression<Func<int,int>> e = x => { return x * x; };` **không compile** — phải `x => x * x`.

**Pitfall capture trong tree:** `int min = 3; Expression<Func<int,bool>> e = x => x > min;` — cây chứa `Constant` tới closure; đổi `min` sau khi build tree **có thể** vẫn thấy giá trị mới (closure) tùy compile — đừng phụ thuộc; capture snapshot `var local = min` rồi đóng `local` nếu cần ổn định. EF: capture local thường dịch thành constant SQL.

---

## 11. Local functions vs Lambda

**Local function**: hàm cục bộ *đặt tên được*, nằm trong thân method (C# 7+).

```csharp
int Sum(int[] a)
{
    int Impl(int i, int acc)
        => i == a.Length ? acc : Impl(i+1, acc + a[i]);
    return Impl(0, 0);
}
```

So với lambda:

- **Hiệu năng**: local function thường **ít cấp phát** hơn khi không capture; hỗ trợ `ref/out`, `in`, iterator `yield`.
- **Đọc/Debug**: tên rõ ràng, gọi đệ quy dễ.
- Lambda **ngắn gọn** cho callback/simple mapping; có thể gắn `async` trực tiếp.
- Local **không** convert sang `Expression<>`. Cần LINQ provider → lambda/expression.

> Nhiều trường hợp local function là lựa chọn **tối ưu** thay vì lambda capture.

---

## 12. Delegates vs Events

- **Delegate**: kiểu tham chiếu phương thức, *có thể* gọi trực tiếp (`Invoke`), `=` ghi đè cả list, expose public field nguy hiểm.
- **Event**: **bao bọc** delegate, **giới hạn** truy cập từ ngoài chỉ `+=`/`-=`; *chỉ* publisher mới được `Invoke`.

**WHY event:** nếu `public Action? OnTick` public, subscriber làm `OnTick = null` xóa hết handler khác, hoặc `OnTick(...)` giả mạo publisher.

```csharp
public class Notifier
{
    public event EventHandler<string>? Message;
    protected virtual void OnMessage(string m)
        => Message?.Invoke(this, m);
}

public class Bad
{
    public Action? Tick; // field delegate — ai cũng Invoke / gán đè
}
```

| | Public delegate field | `event` |
|---|---|---|
| Ngoài class `+=`/`-=` | Có | Có |
| Ngoài class `Invoke` | Có | **Không** |
| Ngoài class `=` | Có (xóa multicast) | **Không** |
| `add`/`remove` tùy chỉnh | Không | Có (lock, weak) |
| Interface | Field không thuộc interface theo nghĩa event | `event` trong interface |

```csharp
public event EventHandler? Changed
{
    add { /* lock / Interlocked.Combine */ }
    remove { /* ... */ }
}
```

**`EventHandler` vs `Action`:** convention .NET: `sender` + `EventArgs` (hoặc `EventHandler<T>`). `Action` gọn hơn cho in-process không cần sender.

**Leak:** static event / singleton publisher + handler capture UI → unsubscribe `Dispose`/`Unloaded`. Weak event pattern (WPF) khi không kiểm soát vòng đời.

**Thread:** `event` không tự thread-safe. Snapshot: `var h = Message; h?.Invoke(...)`.

Multicast exception: một subscriber ném → các subscriber sau không chạy — `GetInvocationList` + try/catch từng cái (mục 4).

**Field-like event** compiler sinh `lock(this)` kiểu cũ hoặc `Interlocked` tùy version — **đừng** `lock(this)` thêm. Custom `add`/`remove` phải thread-safe nếu nhiều thread `+=`.

```csharp
event EventHandler? Changed
{
    add
    {
        EventHandler? h;
        var e = _changed;
        do
        {
            h = e;
            var n = (EventHandler?)Delegate.Combine(h, value);
            e = Interlocked.CompareExchange(ref _changed, n, h);
        } while (e != h);
    }
    remove { /* tương tự Delegate.Remove */ }
}
```

**Covariant event?** Event không covariant trên `T` args một cách tùy tiện — `EventHandler<Derived>` không gán `EventHandler<Base>` (invoke safety). Dùng `EventHandler<T>` với `T` cụ thể.

Interface event: implementer có thể explicit; subscriber chỉ `+=` qua interface, không `Invoke`.

---

## 13. Interop & Function Pointers

- Interop P/Invoke: khai báo delegate phù hợp **gọi ngược (callback)**.
- **Marshal**: `Marshal.GetDelegateForFunctionPointer` / `GetFunctionPointerForDelegate`.
- **Function pointers** (*unsafe*, C# 9+): `delegate*<int, void>` — nhanh hơn, nhưng mất nhiều an toàn.

```csharp
unsafe delegate*<int, void> fp;
```

> Dùng khi thật cần hiệu năng/interop thấp-level; còn lại cứ dùng delegate .NET. Giữ delegate sống (GCHandle / field) khi native gọi ngược — không để GC thu.

---

## 14. Hiệu năng & Best Practices

1. **Ưu tiên method group** khi không cần capture: `DoSomethingAsync` thay vì `x => DoSomethingAsync(x)` để tránh closure.
2. **Static lambda** nếu có thể: `static x => Transform(x)` — không capture, không cấp phát.
3. **Cache delegate dùng lặp lại** (đặc biệt trong loop/hot-path).
4. **Tránh lạm dụng LINQ/lambda** trong hot-path → cân nhắc `for`/`Span<T>`.
5. **Unsubscribe** event để tránh rò rỉ. Giữ **cùng** instance handler khi `-=`.
6. **Expression Trees** chỉ cho nơi cần dịch/biến đổi; không thay thế runtime delegate thông thường. Đừng truyền `Func` vào `IQueryable`.
7. **`async void`** chỉ dùng cho event; mọi async lambda còn lại trả `Task`/`Task<T>`.
8. **Ref safety**: cẩn thận với `ref/out/in` trong delegate; không trả `ref` tới local.
9. **Recursion**: lambda recursion cần tự tham chiếu (khó); local function thuận tiện hơn.
10. **Multicast `Func`**: nhớ chỉ return handler cuối; exception dừng list.
11. **C# 14:** `(text, out result) =>` khi target type đã rõ — `var` không suy modifier.

---

## 15. Cheat sheet & ví dụ tổng hợp

### 15.1 Callback đơn giản

```csharp
void Process<T>(IEnumerable<T> src, Func<T,bool> predicate, Action<T> onHit)
{
    foreach (var x in src)
        if (predicate(x)) onHit(x);
}

Process(new[]{1,2,3,4}, x => x%2==0, Console.WriteLine);
// output: 2 4
```

### 15.2 Ghép pipeline không cấp phát (static lambda)

```csharp
var data = Enumerable.Range(1, 1_000_000);
var sum = data.Where(static x => (x & 1) == 0)     // static: không capture
              .Select(static x => x * 2)
              .Sum();
```

### 15.3 Xử lý multicast an toàn

```csharp
public static void SafeInvoke<T>(this EventHandler<T>? evt, object sender, T args)
{
    if (evt is null) return;
    foreach (EventHandler<T> h in evt.GetInvocationList())
        try { h(sender, args); } catch (Exception ex) { /* log */ }
}
```

### 15.4 Biểu thức động cho EF (Expression Tree)

```csharp
Expression<Func<User,bool>> Build(string? q)
{
    q ??= "";
    return u => u.Name.Contains(q) || u.Email.Contains(q);
}
var expr = Build("alice");
var users = await db.Users.Where(expr).ToListAsync();
```

### 15.5 Local function vs Lambda capture

```csharp
int SumSquares(ReadOnlySpan<int> a)
{
    // Local function: không capture → không cấp phát
    static int Sqr(int x) => x * x;

    var s = 0;
    foreach (var x in a) s += Sqr(x);
    return s;
}
```
