# Toán tử (Operators) trong C# — Bản chi tiết hiện đại

> **Baseline:** .NET **10** / C# **14** — null-conditional assignment, `nameof` unbound generics, user-defined compound assignment / instance `++` `--`.

Toán tử quyết định *thứ tự đánh giá*, *null-safety*, và *bằng nhau*. Nhiều bug production đến từ precedence (`??` vs `?:`), `==` vs `Equals`, và side-effect bị bỏ qua khi receiver null (C# 14 assignment).

## Mục lục

- [Toán tử (Operators) trong C# — Bản chi tiết hiện đại](#toán-tử-operators-trong-c--bản-chi-tiết-hiện-đại)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan \& nguyên tắc](#1-tổng-quan--nguyên-tắc)
  - [2. Bảng ưu tiên (precedence) \& kết hợp (associativity)](#2-bảng-ưu-tiên-precedence--kết-hợp-associativity)
    - [2.1 Pitfall precedence thường gặp](#21-pitfall-precedence-thường-gặp)
  - [3. Toán tử số học](#3-toán-tử-số-học)
  - [4. Toán tử tăng/giảm `++`/`--`](#4-toán-tử-tănggiảm---)
  - [5. So sánh \& bằng/khác — `==` vs `Equals`](#5-so-sánh--bằngkhác----vs-equals)
  - [6. Bit \& logic: `&` `|` `^` `~` `&&` `||` `!`](#6-bit--logic-------)
  - [7. Dịch bit: `<<` `>>` `>>>`](#7-dịch-bit---)
  - [8. Gán \& gán hợp (compound assignment)](#8-gán--gán-hợp-compound-assignment)
  - [9. Điều kiện: `?:` (ternary) vs `??`](#9-điều-kiện--ternary-vs-)
  - [10. Null: `??`, `??=`, null-conditional `?.` `?[]`, null-forgiving `!`](#10-null---null-conditional---null-forgiving-)
    - [10.1 Null-conditional assignment (C# 14)](#101-null-conditional-assignment-c-14)
  - [11. Pattern \& cast: `is`, `as`, cast `(<T>)`](#11-pattern--cast-is-as-cast-t)
    - [`is` (pattern matching)](#is-pattern-matching)
    - [`as`](#as)
    - [Cast tường minh `(T)`](#cast-tường-minh-t)
  - [12. Range \& Index: `..`, `^`](#12-range--index--)
  - [13. Toán tử ngữ nghĩa đặc biệt](#13-toán-tử-ngữ-nghĩa-đặc-biệt)
    - [`nameof` với unbound generics (C# 14)](#nameof-với-unbound-generics-c-14)
  - [14. Operator overloading](#14-operator-overloading)
    - [14.1 Overload cổ điển (`static`)](#141-overload-cổ-điển-static)
    - [14.2 User-defined compound assignment (C\# 14)](#142-user-defined-compound-assignment-c-14)
    - [14.3 Instance `++`/`--` (C\# 14)](#143-instance----c-14)
  - [15. User-defined conversions `implicit`/`explicit`](#15-user-defined-conversions-implicitexplicit)
  - [16. Lifted operators \& nullable](#16-lifted-operators--nullable)
  - [17. Unsafe/pointer operators: `*` `&` `->` `[]` `fixed`](#17-unsafepointer-operators------fixed)
  - [18. Best practices \& cảnh báo thường gặp](#18-best-practices--cảnh-báo-thường-gặp)

---

## 1. Tổng quan & nguyên tắc

- **Biểu thức** được đánh giá theo *precedence* và *associativity* → hiểu đúng để tránh bug.
- Nhiều toán tử hỗ trợ **overload** trên struct/class; với **nullable** (`T?`), đa số toán tử được “nâng” (*lifted*) để hoạt động tự nhiên.
- Toán tử **logic ngắn mạch**: `&&`, `||` — chỉ đánh giá vế phải khi cần.
- Cẩn trọng **overflow** trong số học nguyên → dùng `checked` khi cần.
- **Bằng nhau** có ba lớp hay nhầm: `==` (toán tử, có thể overload), `object.Equals` / `IEquatable<T>`, `ReferenceEquals`. Chọn sai → bug dictionary/set hoặc null.

---

## 2. Bảng ưu tiên (precedence) & kết hợp (associativity)

Từ **cao** → **thấp** (tóm tắt nhóm chính):

1. **Postfix**: `x++` `x--` `x!` (null-forgiving) `a[b]` `a.b` `a?.b` `a?[^i]` `a()` `new T()` `typeof` `checked` `unchecked` `default` `nameof` `stackalloc` — *trái → phải*
   > Lưu ý: `default`, `nameof`, và `stackalloc` là **contextual keywords** (từ khóa phụ thuộc ngữ cảnh) - chỉ có ý nghĩa đặc biệt trong ngữ cảnh nhất định, có thể dùng làm identifier ở chỗ khác.
2. **Unary**: `+x` `-x` `!x` `~x` `++x` `--x` `&x` `*x` `await x` `^i` (index) — *phải → trái*
3. **Multiplicative**: `*` `/` `%` — *trái → phải*
4. **Additive**: `+` `-` — *trái → phải*
5. **Shift**: `<<` `>>` `>>>` — *trái → phải*
6. **Relational & type test**: `<` `>` `<=` `>=` `is` `as` — *trái → phải*
7. **Equality**: `==` `!=` — *trái → phải*
8. **Bitwise AND/XOR/OR**: `&` `^` `|` — *trái → phải*
9. **Conditional AND/OR**: `&&` `||` — *trái → phải*
10. **Null-coalescing**: `??` — *trái → phải*
11. **Conditional**: `?:` — *phải → trái*
12. **Assignment**: `=` `+=` `-=` `*=` `/=` `%=` `&=` `|=` `^=` `<<=` `>>=` `>>>=` `??=` — *phải → trái*
13. **Lambda**: `=>` (ràng buộc riêng, thường thấp)

> Khi nghi ngờ, **dùng ngoặc** để làm rõ ý đồ. Compiler không “đoán” ý bạn — nó chỉ áp bảng trên.

### 2.1 Pitfall precedence thường gặp

```csharp
// 1) ?? cao hơn ?:  →  (a ?? b) ? c : d   KHÔNG phải  a ?? (b ? c : d)
var x = a ?? b ? c : d;

int n = args.Length > 0 ? int.Parse(args[0]) : 0; // OK: ?:
string s = args.Length > 0 ? args[0] : "default";
string t = GetName() ?? "default";                 // OK: ?? cho null

// Muốn: nếu a null thì chọn (b ? c : d)
var y = a ?? (b ? c : d);

// 2) + chuỗi vs số
var msg = "sum=" + 1 + 2;     // "sum=12"  (trái→phải: string + int → string)
var msg2 = "sum=" + (1 + 2);  // "sum=3"

// 3) & | vs && ||  — bitwise/logic không ngắn mạch thấp hơn == nhưng CAO hơn &&
bool flags = (mask & 0x1) != 0 && enabled; // ngoặc & — không viết mask & 0x1 != 0 && ...
// Thực tế `!=` cao hơn `&`!  mask & 0x1 != 0  ≡  mask & (0x1 != 0)  ≡ mask & true/int...
int bit = mask & 0x1 != 0; // cảnh báo / ý nghĩa sai — LUÔN (mask & 0x1) != 0

// 4) is / as vs ==
if (x is string s2 && s2.Length > 0) { } // && thấp hơn is — OK
// if (x is string == false) // không compile như ý; dùng is not string

// 5) await vs cast
await (Task<int>)obj;  // await thấp (unary) vs cast — ngoặc khi mix

// 6) ! null-forgiving vs !=
x! != y   // (x!) != y
!(x != y)

// 7) ?? vs +
var z = a + b ?? 0; // (a + b) ?? 0  — nếu a+b không nullable thì ?? vô nghĩa
var z2 = a + (b ?? 0);
```

**Quy tắc nhớ:** `??` *trên* `?:`; `==` *trên* `&`/`|`; `&` *trên* `&&`. Mix bit-flag với so sánh → **luôn ngoặc**. Mix `??` với ternary → **luôn ngoặc**.

---

## 3. Toán tử số học

- Nhị phân: `+` `-` `*` `/` `%`
- Đơn ngôi: `+` `-`

```csharp
int a = 7 / 2;      // 3 (chia nguyên)
double b = 7 / 2.0; // 3.5
int c = -(-5);      // 5
```

**Overflow**: dùng `checked` để ném `OverflowException`:

```csharp
checked { int x = int.MaxValue + 1; } // ném
```

`decimal` phù hợp tài chính; `%` hoạt động trên số nguyên và `decimal`. Chia nguyên `int` cắt về 0 (không “floor” với số âm như một số ngôn ngữ). `%` với số âm: dấu theo dividend (`-7 % 3 == -1`).

---

## 4. Toán tử tăng/giảm `++`/`--`

- **Prefix**: `++x` trả giá trị **sau khi tăng**.
- **Postfix**: `x++` trả giá trị **trước khi tăng**.

```csharp
int x = 1;
int a = ++x; // x=2, a=2
int b = x++; // x=3, b=2
```

Tránh dùng trong biểu thức phức tạp gây khó đọc (`a[i++] = i++`).

**Overload:** từ trước có `static` `operator ++`/`--` (trả instance mới). **C# 14** thêm **instance** `void operator ++()` / `--()` để mutate in-place — xem [§14.3](#143-instance----c-14).

---

## 5. So sánh & bằng/khác — `==` vs `Equals`

`<` `>` `<=` `>=` `==` `!=`

```csharp
string s1 = "a", s2 = new string("a");
bool eq = s1 == s2; // true: string so sánh theo **nội dung** (ordinal)
```

Ba cơ chế **không thay thế nhau**:

| Cơ chế | Ý nghĩa mặc định | Null | Overload / override |
|---|---|---|---|
| `==` / `!=` | Reference type: **tham chiếu**; value type: giá trị; `string`/`record`: nội dung | `==` **static** — `x == null` an toàn (không NRE) | `operator ==` |
| `Equals(object?)` | Virtual; `object`: tham chiếu; `ValueType`: reflection-ish field | `a.Equals(b)` NRE nếu `a` null | `override Equals` |
| `Equals(T)` (`IEquatable<T>`) | Tránh box value type | Tùy implement | Interface |
| `ReferenceEquals(a,b)` | Cùng instance (hoặc cả hai null) | An toàn | Không overload |
| `GetHashCode` | Phải **khớp** `Equals` | — | Luôn đi đôi |

```csharp
object a = "hi", b = new string("hi".ToCharArray());
Console.WriteLine(a == b);          // True — runtime type string, operator string ==
Console.WriteLine(a.Equals(b));     // True
Console.WriteLine(ReferenceEquals(a, b)); // False

string? n = null;
Console.WriteLine(n == null);       // True, không NRE
// n.Equals(null);                  // NRE

int x = 1, y = 1;
Console.WriteLine(x == y);          // True (value)
Console.WriteLine(x.Equals(y));     // True
```

**`string`:** `==` ordinal (culture-insensitive). So sánh ngôn ngữ: `string.Equals(a, b, StringComparison.CurrentCultureIgnoreCase)` hoặc `Compare`.

**`record` (class):** `==` gọi `Equals` theo value equality (thành phần). `record struct` tương tự.

**Collection:** `List<T>` **không** overload `==` — hai list cùng nội dung: `==` false (khác instance). Dùng `SequenceEqual`.

**WHY override `Equals` thì phải `==`?** Không bắt buộc, nhưng **nên** đồng bộ: nếu `Equals` true mà `==` false (class thường), LINQ/`HashSet` dùng `Equals`+hash, còn `if (a == b)` dùng toán tử → hai thế giới. Guideline: class identity (`==` tham chiếu, `Equals` có thể value — như `string` là ngoại lệ); struct: `==` khớp `Equals`.

```csharp
public sealed class UserId : IEquatable<UserId>
{
    public string Value { get; }
    public UserId(string value) => Value = value;

    public bool Equals(UserId? other) => other is not null && Value == other.Value;
    public override bool Equals(object? obj) => obj is UserId id && Equals(id);
    public override int GetHashCode() => StringComparer.Ordinal.GetHashCode(Value);

    public static bool operator ==(UserId? a, UserId? b)
        => a is null ? b is null : a.Equals(b);
    public static bool operator !=(UserId? a, UserId? b) => !(a == b);
}
```

**Pitfall NaN:** `double.NaN == double.NaN` là **false**; `Equals` trên `double` cũng false với NaN. Dùng `double.IsNaN`.

**Pitfall floating:** `==` trên `float`/`double` exact bit — tiền tệ dùng `decimal` hoặc epsilon có chủ đích.

**`Equals` tĩnh `object.Equals(a, b)`:** an toàn null (cả hai null → true; một null → false; không NRE). Khác `a.Equals(b)`. Dictionary key null: tùy comparer.

```csharp
object.Equals(null, null);          // True
EqualityComparer<string>.Default.Equals(null, null); // True
```

**GetHashCode contract:** `Equals` true ⇒ hash bằng. Đổi field dùng trong `Equals` khi object đang trong `Dictionary` → mất key. `record` sinh `Equals`/`GetHashCode` theo primary properties — mutable record làm key là bẫy.

**`==` trên generic `T`:** không dùng `==` với `T` không ràng `class` (trừ `where T : class`). Dùng `EqualityComparer<T>.Default` — chọn `IEquatable`, tránh box struct.

---

## 6. Bit & logic: `&` `|` `^` `~` `&&` `||` `!`

- Với **số nguyên**: `&` `|` `^` `~` là bitwise.
- Với **bool**: `&` `|` là **logic không ngắn mạch**; `&&` `||` là **ngắn mạch**.

```csharp
bool b = Expensive() & Cheap();  // cả hai chạy
bool c = Expensive() && Cheap(); // nếu Expensive() == false → bỏ qua Cheap()
```

`^` với bool = XOR logic.

**WHY `&` trên bool:** khi *cả hai* side-effect cần chạy (hiếm). Đa số dùng `&&`/`||`. Nullable bool: `&`/`|` lifted — xem §16.

---

## 7. Dịch bit: `<<` `>>` `>>>`

- `<<` dịch trái (điền 0).
- `>>` dịch phải **theo dấu** (arithmetic shift) cho kiểu có dấu (`int`, `long`).
- `>>>` **dịch phải không dấu** (logical shift) — **C# 11**.

```csharp
int  a = -8;        // 111..1000
int  b = a >> 1;    // 111..1100  (giữ bit dấu)
uint c = 0xF0u >>> 2; // 0x3C (C# 11)
```

Shift count được mask (int: 5 bit thấp) — `1 << 32` không phải 2^32.

---

## 8. Gán & gán hợp (compound assignment)

`=`, `+=`, `-=`, `*=`, `/=`, `%=` `&=`, `|=`, `^=`, `<<=`, `>>=`, `>>>=` (C# 11), `??=`

```csharp
int x = 1;
x += 2; // x = x + 2
dict[key] ??= new List<int>(); // khởi tạo nếu null
```

**Semantics quan trọng:** gán hợp đánh giá **LHS một lần** rồi đọc-sửa-ghi. Với indexer/property phức tạp, khác viết tay `a[i++] = a[i++] + 1` (đánh giá index hai lần).

```csharp
list[GetIndex()] += 1;
// ≈
// var i = GetIndex();  // một lần
// list[i] = list[i] + 1;

// KHÔNG tương đương:
list[GetIndex()] = list[GetIndex()] + 1; // GetIndex() hai lần
```

`??=` chỉ gán khi vế trái **null** (`== null`); vế phải không chạy nếu đã có giá trị.

**C# 14:** type có thể khai báo **user-defined** `+=`, `-=`, … dạng instance `void operator +=(T rhs)` để **mutate in-place** thay vì `x = x + y` (tránh tạm / cấp phát). Chi tiết [§14.2](#142-user-defined-compound-assignment-c-14).

`+=` trên `event`/`delegate` là combine multicast — không phải số học. `+=` trên `string` cấp phát chuỗi mới.

---

## 9. Điều kiện: `?:` (ternary) vs `??`

```csharp
var label = (age >= 18) ? "Adult" : "Minor";
```

- Kết quả phải có kiểu suy ra được; có thể tham gia **overload resolution**.
- `?:` chọn theo **điều kiện bool**. `??` chọn theo **null**.

| | `?:` | `??` |
|---|---|---|
| Điều kiện | `bool` | `== null` (vế trái) |
| Vế phải khi “sai” | Luôn có (nhánh false) | Chỉ chạy khi trái null |
| Kiểu điển hình | Hai nhánh cùng kiểu / chung base | Trái nullable, phải cùng kiểu non-null hơn |
| Precedence | Thấp hơn `??` | Cao hơn `?:` |

```csharp
string? name = Get();
string display = name != null ? name : "guest"; // ?:  — verbose
string display2 = name ?? "guest";              // ??  — đúng ý null

// SAI ý: ?? không thay if (count == 0)
int n = count ?? 0; // chỉ khi count là int?

// Mix — LUÔN ngoặc
var r = flag ? a ?? b : c;
var r2 = (flag ? a : b) ?? c;
```

Throw expression: `x ?? throw new ArgumentNullException(nameof(x))` — vế phải `??` có thể `throw`.

**Pitfall kiểu:** `condition ? 1 : null` → `int?`. `condition ? "a" : 1` không compile (không common type).

---

## 10. Null: `??`, `??=`, null-conditional `?.` `?[]`, null-forgiving `!`

```csharp
string? name = GetNameOrNull();
int len = name?.Length ?? 0; // ?. tránh NRE, ?? cung cấp mặc định

dict?["key"]?.ToString();    // ?[] với indexer

obj!.Property // null-forgiving: cam kết với compiler là không null (cẩn thận)
```

- `??` và `??=` chỉ kiểm `== null`.
- `?.` trả **null** nếu vế trái null, không ném NRE. Chuỗi `a?.B?.C` rất hữu dụng cho truy cập sâu.
- `?.` trên value type → kết quả `T?` (`name?.Length` là `int?`).
- `!` **không** sinh IL kiểm null — chỉ tắt cảnh báo NRT. Sai → NRE runtime.

### 10.1 Null-conditional assignment (C# 14)

`?.` và `?[]` dùng được ở **vế trái** của gán / gán hợp. Vế phải **chỉ** đánh giá khi receiver **không** null.

```csharp
// trước C# 14
if (customer is not null)
{
    customer.Order = GetCurrentOrder();
}

// C# 14
customer?.Order = GetCurrentOrder();
// nếu customer == null → không gọi GetCurrentOrder(), không gán
```

**WHY:** tránh `if` lặp khi gán property/indexer trên graph có thể null. **Semantics:** đây là *conditional assignment*, không phải “gán null vào customer”.

Compound assignment cũng được:

```csharp
customer?.Score += 10;
buffer?[i] = value;
list?[index] += delta;
```

**Không** áp dụng cho `++` / `--` dạng null-conditional (`customer?.Count++` — không hợp lệ). Dùng `if (customer is not null) customer.Count++;`.

**Pitfalls:**

1. **Side-effect RHS biến mất** khi receiver null — logging, factory, `Interlocked` trong RHS sẽ không chạy.

```csharp
customer?.Order = CreateOrderAndLog(); // null customer → không log, không tạo order
```

2. `??=` kết hợp `?.` dễ đọc nhầm: `customer?.Tag ??= "x"` — nếu `customer` null, không gán gì; nếu `Tag` null mới gán `"x"`.
3. Không thay `if` khi nhánh null phải làm việc khác (`throw`, default object).
4. Event: `handler?.Invoke(...)` đã có từ trước (gọi, không phải assignment). Đừng nhầm với `obj?.Event += H` — **không** hợp lệ (subscribe cần receiver chắc chắn).

---

## 11. Pattern & cast: `is`, `as`, cast `(<T>)`

### `is` (pattern matching)

```csharp
if (x is Person { Age: >= 18 } p) { /* ... */ }
```

- Hỗ trợ **type pattern**, **property/relational/list patterns**, kết hợp `and/or/not` (C# 9+).
- `is` không dùng user-defined conversion (khác cast). `is` true ⇒ gán designator an toàn.

### `as`

- Cast **an toàn**: trả `null` nếu không chuyển được (reference/nullable).

```csharp
var p = obj as Person;
if (p != null) Console.WriteLine(p.Name);
```

Không dùng `as` với value type không nullable (`obj as int` lỗi). Dùng `obj as int?` hoặc `is int`.

### Cast tường minh `(T)`

- Có thể ném `InvalidCastException` nếu không tương thích.

```csharp
var p2 = (Person)obj; // nếu obj không phải Person → ném
```

**Chọn:** pattern `is` khi cần nhánh; `as` khi “có thì dùng, không thì bỏ”; cast khi *invariant* sai thì bug.

---

## 12. Range & Index: `..`, `^`

- `^i` → chỉ số **từ cuối** (1 = phần tử cuối).
- `a[start..end]` → **slice** từ `start` (inclusive) tới `end` (exclusive).

```csharp
int[] arr = {0,1,2,3,4};
var last = arr[^1];    // 4
var mid  = arr[1..^1]; // {1,2,3}
```

Hoạt động với `string`, `Span<T>`, `Index`, `Range`, và các type tự cài indexer `this[Index]`/`this[Range]`. `arr[..]` copy/`AsSpan` tùy target typed.

---

## 13. Toán tử ngữ nghĩa đặc biệt

- **`await`**: tạm ngưng method async cho đến khi awaitable hoàn thành. (*Xem phần [Async](async.md)*)
- **`nameof(x)`**: lấy **tên** định danh dạng chuỗi, không bị rename runtime (an toàn refactor).
- **`sizeof(T)`**: kích thước byte của kiểu unmanaged (với managed struct, thường cần `unsafe`).
- **`typeof(T)`**: trả `System.Type` của `T`.
- **`checked` / `unchecked`**: bật/tắt kiểm tra overflow số học.
- **`default`**: literal/expr tạo giá trị mặc định của `T`.
- **`new`**: tạo instance; cũng là **operator** ở mức ngữ nghĩa.
- **`stackalloc`**: cấp phát trên stack (unsafe) cho buffer `Span<T>`/con trỏ.

```csharp
var s = nameof(Person.Name);     // "Name"
var t = typeof(List<string>);    // System.Type
Span<byte> buf = stackalloc byte[256]; // stack buffer
```

`nameof` **không** đánh giá argument (`nameof(DoWork())` lấy tên `DoWork`, không gọi). `nameof(this.X)` → `"X"`.

### `nameof` với unbound generics (C# 14)

Trước C# 14 chỉ dùng được **closed** generic (`List<int>`). Từ C# 14, `nameof` nhận **unbound** generic type:

```csharp
Console.WriteLine(nameof(List<>));          // "List"
Console.WriteLine(nameof(Dictionary<,>));   // "Dictionary"
Console.WriteLine(nameof(Nullable<>));      // "Nullable"
```

**WHY:** logging, diagnostic ID, source generator — không muốn (hoặc không thể) chọn type argument. Metadata tên type **không** gồm arity trong chuỗi `nameof` (`Dictionary<,>` → `"Dictionary"`, không phải `"Dictionary`2"`).

```csharp
// SAI trước C# 14: nameof(List<>) — lỗi
// Vẫn khác typeof:
Type open = typeof(List<>);          // Type mở, có metadata arity
string name = nameof(List<>);        // chỉ "List"

void LogHandler<T>() => Console.WriteLine(nameof(T)); // "T" — tên tham số, không phải argument
```

`nameof` trên alias / `using` identifier lấy tên bạn viết. Unbound chỉ cho **type** generic, không phải method generic `nameof(M<>)` theo nghĩa type.

**Pitfall unbound:** `nameof(List<int>)` vẫn `"List"` (tên type, không gồm argument). Arity phân biệt `Action` vs `Action<T>` trong reflection là `Action` vs `Action`1` — `nameof` **không** cho `1`. Log kèm `typeof(T).Name` khi cần arity.

```csharp
void Diagnose(Type t) =>
    Console.WriteLine($"{nameof(IDictionary<,>)} closed={t.IsGenericType && !t.IsGenericTypeDefinition}");
```

Unbound `nameof` hữu ích source-gen: `[LoggerMessage(EventName = nameof(MyHandler<>))]` không ép `MyHandler<int>`.

---

## 14. Operator overloading

### 14.1 Overload cổ điển (`static`)

- Cho phép định nghĩa lại nghĩa của toán tử trên **class/struct** (không phải interface).
- Khai báo `public static <ret> operator +(T a, T b) { ... }`.

**Overload được** (tiêu biểu): `+ - ! ~ ++ -- true false * / % & | ^ << >> == != < > <= >=`
**C# 14 thêm:** compound `+=` `-=` … và instance `++`/`--` (xem dưới).
**Không overload được:** `=` `&&` `||` `?:` `??` `?.` `?[]` `=>` `.` `[]` `()` v.v.

Quy tắc quan trọng:

- Nếu overload `==` → nên override `Equals`/`GetHashCode` và overload `!=`.
- `true`/`false` giúp type dùng trong `if`, kết hợp với `&`/`|`.
- Tôn trọng kỳ vọng của người dùng (tính giao hoán/bất biến).

```csharp
public readonly struct Vector2(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;

    public static Vector2 operator +(Vector2 a, Vector2 b)
        => new(a.X + b.X, a.Y + b.Y);

    public static bool operator ==(Vector2 a, Vector2 b)
        => a.X == b.X && a.Y == b.Y;
    public static bool operator !=(Vector2 a, Vector2 b) => !(a == b);
    public override bool Equals(object? o) => o is Vector2 v && this == v;
    public override int GetHashCode() => HashCode.Combine(X, Y);
}
```

Extension operators (khai báo trong `extension` block): xem [oop.md §8](./oop.md#8-extension-members-c-14).

### 14.2 User-defined compound assignment (C# 14)

Mặc định `x += y` ≈ `x = x + y` (có thể cấp phát/copy). C# 14 cho phép **instance** operator `void`, mutate `this`:

```csharp
public class GateAttendance
{
    public string GateId { get; }
    public int Count { get; private set; }

    public GateAttendance(string gateId, int count = 0)
    {
        GateId = gateId;
        Count = count;
    }

    // Vẫn có thể giữ static + làm fallback khi không dùng được instance op
    public static GateAttendance operator +(GateAttendance g, int n)
        => new(g.GateId, g.Count + n);

    // C# 14: in-place
    public void operator +=(int partySize) => Count += partySize;
}

var gate = new GateAttendance("A");
gate += 5;   // gọi void operator += — không tạo instance mới
gate += 3;
```

Luật nhanh:

- `public`, **không** `static`, trả **`void`**, **một** tham số (vế phải).
- Khi LHS là **biến** và có compound op phù hợp → ưu tiên instance op; không có thì fallback `x = x op y`.
- Phù hợp buffer lớn, tensor, counter mutable — **không** hợp kiểu thiết kế hoàn toàn immutable (trừ khi chấp nhận đổi mô hình).
- LHS không phải biến (property get-only, `x + y += z`) → không dùng instance op.

### 14.3 Instance `++`/`--` (C# 14)

```csharp
public class Counter
{
    public int Value { get; private set; }

    // Classic static: luôn trả instance mới
    public static Counter operator ++(Counter c)
        => new() { Value = c.Value + 1 };

    // C# 14 instance: mutate in-place
    public void operator ++() => Value++;
    public void operator --() => Value--;
}

var c = new Counter();
++c;   // ưu tiên instance void operator ++ khi c là biến
c++;   // compiler vẫn dùng instance op khi hợp lệ; giá trị biểu thức = Value trước tăng
```

Luật nhanh:

- Instance: `public void operator ++()` / `--()` — **không** tham số, **không** `static`.
- Prefix trên biến → ưu tiên instance; nếu không phải biến / không có instance op → dùng static unary.
- Một khai báo instance phục vụ cả prefix và postfix (compiler lấy giá trị trước/sau tùy ngữ cảnh).
- Reference type: instance op trên `null` → `NullReferenceException`.

---

## 15. User-defined conversions `implicit`/`explicit`

Cho phép chuyển đổi giữa type của bạn và type khác:

```csharp
public readonly struct Dollars
{
    public decimal Amount { get; }
    public Dollars(decimal amount) => Amount = amount;

    public static implicit operator Dollars(decimal a) => new(a);
    public static explicit operator decimal(Dollars d) => d.Amount;
}

Dollars d = 10.5m;           // implicit
decimal v = (decimal)d;      // explicit
```

- Dùng `implicit` cho **an toàn** (không mất dữ liệu); `explicit` khi có thể mất thông tin/chi phí.
- Tránh tạo chuyển đổi **mơ hồ** gây lỗi overload resolution.
- Conversion **không** tham gia `is`/`as` pattern theo nghĩa user-defined (cast tường minh mới gọi operator).

---

## 16. Lifted operators & nullable

Với `T?` (nullable **value** type), hầu hết toán tử nhị phân được “nâng”: compiler sinh phiên bản nhận `T?`, trả `T?` (hoặc `bool` cho so sánh).

**WHY:** viết `a + b` khi `a`, `b` là `int?` mà không gọi `.Value` mọi chỗ. **Semantics:** với số học, **một** toán hạng `null` → kết quả `null` (SQL-like), **không** ném.

```csharp
int? a = null, b = 10;
int? c = a + b;   // => null (nếu bất kỳ toán hạng null)
bool d = a == b;  // => false (trừ khi cả hai null → true)
bool e = a == null; // true — so sánh với null literal
bool f = a > b;   // false (relational: null tham gia → false, kể cả a > null)
bool g = a > null;  // false
```

Quy tắc nhanh:

| Nhóm | Cả hai có giá trị | Có `null` |
|---|---|---|
| `+ - * / %` `& \| ^` shift | `op` trên `.Value` | Kết quả `null` |
| `==` | So sánh giá trị | Cả hai null → true; một null → false |
| `!=` | Ngược `==` | Cả hai null → false; một null → true |
| `< > <= >=` | So sánh giá trị | **false** (không `null` bool) |
| `!` trên `bool?` | — | `null` → `null` |

```csharp
bool? p = null, q = true;
bool? r = p & q;  // false?  — lifted bool &: null & true → null? 
                  // Thực tế: null & true = null; null & false = false; true & true = true
bool? s = p && q; // && trên bool? không ngắn mạch giống bool — hạn chế; ưu tiên HasValue
```

**Không lifted:** `++`/`--` trên `T?` tăng `.Value` nếu `HasValue`. User-defined operator trên `T` được lift nếu operand `T?` và operator trả value type.

```csharp
int? n = 3;
n++;          // 4
int? m = null;
m++;          // vẫn null
int definite = (a + b) ?? 0; // thay mặc định trước khi cần non-null
```

**Pitfall:** `if (a > b)` khi `a` hoặc `b` null → nhánh false — dễ hiểu nhầm “a không lớn hơn” vs “không so sánh được”. Dùng `a is int av && b is int bv && av > bv` hoặc `.HasValue`.

Nullable **reference** (`string?`) **không** dùng lifted operators số học — `?.` / `??` / NRT warnings.

---

## 17. Unsafe/pointer operators: `*` `&` `->` `[]` `fixed`

Trong **unsafe context**:

- `*p` truy cập giá trị, `&x` lấy địa chỉ, `p->Member` truy cập member qua con trỏ struct, `p[i]` index pointer arithmetic.
- `fixed` cố định object/array để lấy địa chỉ ổn định cho GC.

```csharp
unsafe
{
    int x = 10;
    int* p = &x;
    *p = 20;
}
```

> Chỉ dùng khi thật cần (interop/hiệu năng đặc biệt).

---

## 18. Best practices & cảnh báo thường gặp

1. **Ưu tiên biểu thức rõ ràng**: thêm ngoặc khi tổ hợp `&&`, `||`, `??`, `?:`, `&` với so sánh.
2. **Chuỗi &**: với chuỗi, `+` cấp phát; trong vòng lặp dùng `StringBuilder` / interpolation.
3. **So sánh chuỗi**: rõ ràng `StringComparison`/`StringComparer` thay vì mặc định.
4. **`==` vs `Equals`**: `==` an toàn null, có thể overload khác `Equals`. Dictionary dùng `Equals`+hash. `ReferenceEquals` khi cần identity.
5. **Nullable**: tận dụng `?.` + `??` để tránh NRE; tránh lạm dụng `!` (null-forgiving). Lifted: `null + x` là `null`; relational với null là `false`.
6. **Overflow**: trong tính toán tài chính, dùng `decimal`; bật `checked` khi cần chính xác.
7. **Overload toán tử**: nhất quán với trực giác; luôn đi đôi `==`/`!=`/`Equals`/`GetHashCode`. Compound/`++` instance (C# 14) khi cần hiệu năng in-place — đừng trộn mơ hồ với API immutable.
8. **Shift mới `>>>`**: đảm bảo target .NET/C# hỗ trợ (C# 11+).
9. **Range/Index**: coi chừng `IndexOutOfRangeException`; nhớ `end` là **exclusive**.
10. **`as` vs cast vs `is`**: `is` pattern khi phân nhánh; `as` khi chấp nhận `null`; cast khi chắc chắn đúng.
11. **IQueryable**: một số toán tử ở LINQ có semantics khác (dịch SQL) — xem chương LINQ.
12. **Null-conditional assignment (C# 14)**: RHS không chạy khi receiver null — side-effect có thể “biến mất”.
13. **Compound assignment**: LHS đánh giá **một lần** — dựa vào đó với indexer.
14. **`nameof(List<>)` (C# 14)**: chuỗi tên, không phải `typeof` mở.
