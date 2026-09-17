# Hệ thống kiểu dữ liệu (Common Type System)

> **Baseline:** .NET **10** / C# **14**. Mục [18. Union types](#18-union-types-c-15-preview) là **PREVIEW (.NET 11 / C# 15)** — chưa GA.

C# là một ngôn ngữ `strongly typed`, có nghĩa là các kiểu dữ liệu được sử dụng rất chặt chẽ, và bạn luôn phải 
xác định kiểu cụ thể của một biến, hằng hoặc biểu thức. Vì C# là ngôn ngữ được thiết kế cho .NET nên nó hỗ trợ 
đầy đủ hệ thống kiểu trong .NET, bao gồm các kiểu dữ liệu primitive và cả một tập rất lớn các kiểu phức tạp được
khai báo trong hệ thống thư viện của .NET.

Khi nói về một kiểu dữ liệu, ta sẽ có các thông tin sau:
  - Kích cỡ không gian bộ nhớ mà kiểu dữ liệu đó chiếm.
  - Các giá trị nhỏ nhất và lớn nhất của kiểu dữ liệu.
  - Các thành phần bên trong kiểu dữ liệu (phương thức, các trường, thuộc tính...).
  - Kiểu dữ liệu cơ sở mà kiểu dữ liệu này thừa kế.
  - Các interface được implement.
  - Các toán tử mà kiểu dữ liệu hỗ trợ.
  
Trình biên dịch có những thông tin trên về tất cả các kiểu dữ liệu, nhờ đó nó hỗ trợ an toàn kiểu (type safe), 
bạn không thể gán các giá trị không tương thích vào cho các biến, ví dụ không thể gán một giá trị số thực vào
một biến kiểu số nguyên (nhưng ngược lại thì được).

Các thông tin kiểu dữ liệu trên cũng được lưu vào file thực thi (.exe hoặc .dll). Trình runtime của .NET (CLR) 
sẽ dùng các thông tin này để đảm bảo an toàn kiểu khi nó cấp phát và thu hồi bộ nhớ.

---

## Mục lục

- [Hệ thống kiểu dữ liệu (Common Type System)](#hệ-thống-kiểu-dữ-liệu-common-type-system)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan CTS/CLS \& Runtime](#1-tổng-quan-ctscls--runtime)
    - [1.1 CTS — hệ thống kiểu chung](#11-cts--hệ-thống-kiểu-chung)
    - [1.2 CLS — quy tắc tương tác giữa ngôn ngữ](#12-cls--quy-tắc-tương-tác-giữa-ngôn-ngữ)
    - [1.3 CLR, IL, metadata](#13-clr-il-metadata)
  - [2. Bức tranh bộ nhớ: Stack/Managed Heap \& GC](#2-bức-tranh-bộ-nhớ-stackmanaged-heap--gc)
    - [2.1 Stack](#21-stack)
    - [2.2 Managed Heap \& GC](#22-managed-heap--gc)
    - [2.3 Value type không luôn nằm trên stack](#23-value-type-không-luôn-nằm-trên-stack)
  - [3. Phân loại kiểu dữ liệu](#3-phân-loại-kiểu-dữ-liệu)
    - [3.1 Value types](#31-value-types)
      - [Nullable Value Types](#nullable-value-types)
      - [Boxing \& Unboxing](#boxing--unboxing)
    - [3.2 Reference types](#32-reference-types)
    - [3.3 Built-in \& Primitive](#33-built-in--primitive)
  - [4. Kiểu đặc biệt: `object`, `string`, `dynamic`, `void`, `null`](#4-kiểu-đặc-biệt-object-string-dynamic-void-null)
  - [5. Struct, `readonly struct`, `ref struct` (byref-like)](#5-struct-readonly-struct-ref-struct-byref-like)
  - [6. Enum \& `[Flags]`](#6-enum--flags)
  - [7. Tuples \& `ValueTuple`, deconstruction](#7-tuples--valuetuple-deconstruction)
  - [8. Records (record class / record struct)](#8-records-record-class--record-struct)
  - [9. Mảng (arrays): 1D, nhiều chiều, jagged, `Span<T>`](#9-mảng-arrays-1d-nhiều-chiều-jagged-spant)
  - [10. Nullable Reference Types (NRT)](#10-nullable-reference-types-nrt)
    - [10.1 Annotation attributes](#101-annotation-attributes)
    - [10.2 Pitfalls NRT](#102-pitfalls-nrt)
  - [11. Khai báo biến \& suy luận kiểu (`var`, target-typed, default literal)](#11-khai-báo-biến--suy-luận-kiểu-var-target-typed-default-literal)
  - [12. Giá trị mặc định (default values)](#12-giá-trị-mặc-định-default-values)
  - [13. Generics \& ràng buộc (`where`, `new()`, `struct/class/unmanaged/notnull`), phương sai (variance)](#13-generics--ràng-buộc-where-new-structclassunmanagednotnull-phương-sai-variance)
    - [13.1 Ràng buộc (`where`)](#131-ràng-buộc-where)
    - [13.2 `allows ref struct` (C\# 13)](#132-allows-ref-struct-c-13)
    - [13.3 Phương sai (variance)](#133-phương-sai-variance)
  - [14. Namespace, `using`, `global using`, `extern alias`](#14-namespace-using-global-using-extern-alias)
  - [15. Chuyển đổi \& ép kiểu: implicit/explicit, user-defined, pattern matching](#15-chuyển-đổi--ép-kiểu-implicitexplicit-user-defined-pattern-matching)
    - [15.1 Chuyển đổi chuẩn](#151-chuyển-đổi-chuẩn)
    - [15.2 User-defined conversion](#152-user-defined-conversion)
    - [15.3 Pattern matching](#153-pattern-matching)
  - [16. Unsafe \& unmanaged types (overview), function pointers](#16-unsafe--unmanaged-types-overview-function-pointers)
  - [17. Sơ đồ “type tree” (ASCII)](#17-sơ-đồ-type-tree-ascii)
  - [18. Union types (C# 15) — **PREVIEW**](#18-union-types-c-15-preview)

---

## 1. Tổng quan CTS/CLS & Runtime

Tất cả các kiểu dữ liệu trong .NET được thiết kế để dùng bởi bất kỳ ngôn ngữ .NET nào, vì vậy người 
ta gọi nó là "Hệ thống kiểu chung" (CTS). Có hai đặc điểm quan trọng với CTS:

- Hỗ trợ thừa kế: tất cả các kiểu dữ liệu trong CTS đều hỗ trợ thừa kế, một kiểu dữ liệu có thể thừa kế từ một
kiểu khác, và tất cả các kiểu dữ liệu, bao gồm cả các kiểu nguyên thủy (primitive type) đều thừa kế trực tiếp
hoặc gián tiếp từ `System.Object` (`object`).
- Các kiểu dữ liệu trong .NET được chia làm hai loại: [value type](#31-value-types) (kiểu giá trị) 
và [reference type](#32-reference-types) (kiểu tham chiếu). Các kiểu dữ liệu được khai báo với `struct` là `value type`;
`class` và `record class` là reference type; `record struct` vẫn là value type.

> - **CTS (Common Type System)**: chuẩn định nghĩa tất cả kiểu trong .NET (giá trị/tham chiếu, kế thừa, generic…).  
> - **CLS (Common Language Specification)**: tập con quy tắc để ngôn ngữ khác nhau tương tác (interop) thuận lợi.  
> - **CLR**: runtime thực thi IL, quản lý **GC**, **JIT**, kiểm tra an toàn kiểu (type-safety).  
> - **C#** biên dịch → **IL** (MSIL/CIL), chứa **metadata** (thông tin type, thuộc tính, method…).

**Vì sao / Khi nào dùng:** hiểu CTS/CLS khi viết thư viện dùng chung nhiều ngôn ngữ .NET (F#, VB, C++/CLI) hoặc khi debug IL/metadata. Ứng dụng C# thuần thì CTS “ẩn” sau compiler; CLS chỉ thành vấn đề khi public API “lạ” với ngôn ngữ khác.

### 1.1 CTS — hệ thống kiểu chung

CTS trả lời: *kiểu là gì trong CLR?* Mọi ngôn ngữ .NET (C#, F#, VB…) ánh xạ kiểu của mình vào cùng một mô hình:

| Khái niệm CTS | Ví dụ C# |
|---|---|
| Value type | `int`, `struct`, `enum`, `record struct` |
| Reference type | `class`, `interface`, `delegate`, `string`, array, `record class` |
| Generic type | `List<T>`, `Span<T>` |
| Byref-like | `ref struct` (`Span<T>`) — không box, không lên heap như object độc lập |

CTS cho phép **kế thừa đơn** (một base class) + **nhiều interface**. Primitive (`int`, `bool`…) cũng là kiểu CTS đầy đủ: có method (`int.Parse`), implement interface (`IComparable<int>`), và vẫn thừa kế `object` (qua `ValueType`).

**So sánh với “kiểu C#”:** một số cấu trúc C# **không** phải kiểu CTS độc lập — ví dụ `dynamic` (compile-time trick quanh `object` + DLR), `var` (suy luận, biến mất sau compile), nullable annotation `string?` (metadata + warning, không đổi IL runtime của reference type).

### 1.2 CLS — quy tắc tương tác giữa ngôn ngữ

CLS là **tập con** của CTS: những gì *mọi* ngôn ngữ CLS-compliant phải hiểu. C# cho phép nhiều thứ CTS mà CLS **cấm** trên **public API** của assembly muốn interop rộng.

Ví dụ **không CLS-compliant** (vẫn hợp lệ C#):

```csharp
[assembly: CLSCompliant(true)]

public class Api
{
    public uint Count { get; set; }          // unsigned — VB/F# cổ điển khó dùng
    public void Run(sbyte x) { }             // sbyte không CLS
    public void Process(int n) { }
    public void Process(ref int n) { }       // overload chỉ khác ref — không CLS
}
```

Compiler cảnh báo `CS3001`/`CS3002`… khi bật `[assembly: CLSCompliant(true)]`. **Bên trong** assembly (internal/private) bạn vẫn dùng `uint`, pointer, generic constraint “lạ”.

**So sánh CTS vs CLS:**

| | CTS | CLS |
|---|---|---|
| Phạm vi | *Mọi* kiểu CLR có thể biểu diễn | Tập con “an toàn interop” |
| Mục tiêu | Runtime thống nhất | Public API đa ngôn ngữ |
| Vi phạm | Không compile / không load | Cảnh báo; C# khác vẫn gọi được |
| Ví dụ ngoài CLS | `uint`, pointer, overload chỉ khác `ref`/`out` | — |

**Pitfall:** `[CLSCompliant(true)]` trên thư viện NuGet công khai — đừng expose `uint` id, `sbyte`, hay generic unconstrained `T` trên public surface nếu consumer có thể là VB. Ứng dụng nội bộ C#-only thì CLS ít quan trọng.

### 1.3 CLR, IL, metadata

Luồng: **source C# → Roslyn → IL + metadata → JIT (RyuJIT) → native**. Metadata mô tả type, member, generic, custom attribute; CLR dùng để:

- Kiểm tra type-safety khi load (verifier).
- Cấp phát object đúng layout, chạy GC.
- Reflection (`typeof`, `GetType()`).

**Vì sao metadata quan trọng:** NRT, `required`, nullable annotations, `DynamicallyAccessedMembers`… sống chủ yếu ở metadata/attribute — runtime không “biết null” trên reference type. Đó là lý do NRT chỉ là *cảnh báo compile-time*, không chặn `null` lúc chạy.

---

## 2. Bức tranh bộ nhớ: Stack/Managed Heap & GC

### 2.1 Stack

- **Stack**: lưu biến local/value type (có thể “nằm trong” stack frame), tham chiếu tới object, con trỏ return… vòng đời theo scope call.  
- Mỗi **thread** có stack riêng; cấp phát/hủy **LIFO**, không qua GC.  
- Giới hạn kích thước (thường vài MB) → đệ quy sâu / `stackalloc` lớn → `StackOverflowException`.

**Vì sao stack nhanh:** bump pointer theo frame, không mark/sweep. Đổi lại: không chia sẻ giữa thread, không sống sau khi method return.

### 2.2 Managed Heap & GC

- **Managed Heap**: nơi **object/reference type** (và **boxed value**) sống; GC thu hồi khi không còn tham chiếu.  
- **Value type trong object**: tồn tại trong heap **bên trong** object chứa (ví dụ field của class).  
- **Large Object Heap (LOH)** cho object lớn (≈ ≥ 85KB).  
- **Generations** (Gen 0/1/2): tối ưu chi phí thu gom. Có một nguyên tắc là: Những đối tượng có tuổi đời càng ngắn thì xác suất nó không còn được sử dụng càng cao, những đối tượng static hoặc lưu trữ dữ liệu lâu dài có thể sẽ "sống" hết vòng đời ứng dụng, việc nhóm các đối tượng theo tuổi đời do vậy sẽ giúp tối ưu chi phí giải phóng.

| Thế hệ | Ý nghĩa |
|---|---|
| **Gen 0** | Object mới; GC thường xuyên, rẻ |
| **Gen 1** | Sống sót 1 lần; vùng đệm |
| **Gen 2** | Lâu đời; full GC đắt |
| **LOH / POH** | Object lớn / pinned; compact đắt hoặc không compact |

**Semantics GC (rút gọn):** allocation trên ephemeral segment gần như bump-pointer; collection = mark (+ compact). Object còn *root* (static, local còn sống, register, GCHandle) thì sống. **Pin** (`fixed`, interop) cản compact → pin ngắn.

Chi tiết lifetime/`Span`: [memory-spans.md](memory-spans.md).

### 2.3 Value type không luôn nằm trên stack

> **Lưu ý quan trọng**: Value types không phải lúc nào cũng nằm trên stack. Vị trí lưu trữ phụ thuộc vào **ngữ cảnh**:
> - Biến local value type trong method → nằm trên stack (trừ khi bị capture trong closure hoặc async method)
> - Value type là field của class → nằm trên heap (bên trong object)
> - Value type bị boxed → nằm trên heap
> - Value type trong mảng → nằm trên heap

```csharp
int local = 1;                          // stack (thường)
var box = (object)local;                // boxing → heap
var arr = new int[] { 1, 2, 3 };        // mảng trên heap; phần tử int nằm trong mảng
async Task Capture()
{
    int n = 42;                         // compiler có thể đưa n lên heap (state machine)
    await Task.Yield();
    Console.WriteLine(n);
}
```

**Pitfall:** “struct nhỏ = không GC” là sai nếu struct nằm trong class, array, hoặc bị box. Đo allocation (`dotMemory`, `GC.GetAllocatedBytesForCurrentThread`) thay vì đoán.

**Vì sao / Khi nào dùng:** nghĩ stack/heap khi tối ưu hot-path (tránh box, tránh closure capture struct lớn) — không phải khi thiết kế domain model thông thường.

---

## 3. Phân loại kiểu dữ liệu

### 3.1 Value types

- Thừa kế **ngầm** từ `System.ValueType` (cuối vẫn từ `object`) nhưng **không** hỗ trợ kế thừa tuỳ ý.  
- Gồm: các số nguyên/thực/decimal/bool/char, `enum`, `struct`, `DateTime`, `Guid`, `ValueTuple`,…  
- **Copy-by-value** khi gán/tham số (nếu không dùng `ref/in/out`).  
- **Hiệu năng**: tốt cho dữ liệu nhỏ, bất biến; nhưng copy struct lớn tốn chi phí → cân nhắc `in`, `ref readonly`.

**Semantics copy:** gán `Point a = b` copy *toàn bộ field*. Mutate `a` không đụng `b`. Đó là lý do mutable struct trong collection/`foreach` dễ bug (bạn sửa **bản copy**).

```csharp
var p1 = new Point(1, 2);
var p2 = p1;          // copy
p2 = p2 with { X = 9 }; // record struct: p1 vẫn (1,2)
```

**So sánh với reference type:** hai biến class trỏ cùng object; hai biến struct là hai bản độc lập (trừ `ref`).

**Vì sao / Khi nào dùng value type:** tọa độ, money (`decimal` wrapper), ID nhỏ, `DateTime`, key dictionary. Tránh struct > ~16–24 byte nếu copy thường xuyên; tránh mutable struct public.

#### Nullable Value Types

- `T?` với `T` là value type ⇒ `Nullable<T>`. Ví dụ: `int?`, `DateTime?`.  
- Thuộc tính: `.HasValue`, `.Value`, hoặc dùng `??`, `??=`, pattern matching `is null`.  
- Toán tử nâng (lifted operators) hoạt động với nullable (`int? a + int? b`).

`T?` **là struct** (`Nullable<T>`): `HasValue` + `value`. `null` ở đây là *trạng thái*, không phải reference.

```csharp
int? a = null;
int? b = 3;
int? sum = a + b;          // null (lifted)
int n = b ?? 0;            // 3
if (b is int x)            // pattern: HasValue
    Console.WriteLine(x);
```

**Pitfall:** `.Value` khi `!HasValue` → `InvalidOperationException`. Ưu tiên `??` / pattern. `int?` **không** thay NRT (`string?`).

#### Boxing & Unboxing

- **Boxing**: value → object (heap).  
- **Unboxing**: object → value (copy từ box).  
- Tốn cấp phát/copy → tránh trong hot-path; lưu ý khi dùng `ArrayList`, `object`, “params object[]”.

```csharp
int x = 42;
object o = x;         // boxing
int y = (int)o;       // unboxing (InvalidCastException nếu sai kiểu)
```

**Chi phí thật:** (1) cấp phát object trên heap, (2) copy bits vào box, (3) unbox copy ra, (4) object sống đến GC Gen 0+. Trong vòng lặp, boxing biến “struct rẻ” thành “GC pressure”.

```csharp
// ❌ mỗi lần Add box một int
ArrayList legacy = new();
legacy.Add(1);
legacy.Add(2);

// ❌ interface trên struct thường box, trừ khi generic
IComparable c = 5;                 // box
IComparable<int> c2 = 5;           // vẫn box vì gán vào interface typed sẵn
int cmp = 5.CompareTo(3);          // không box — gọi trực tiếp

// ✅ generic constraint tránh box
static int Compare<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b);

Compare(5, 3);                     // không box
```

**So sánh:** `params object[]` box mọi value argument. `params ReadOnlySpan<int>` thì không — xem [methods.md §11](methods.md#11-params).

**Pitfall:** `struct` implement interface rồi gán `IDisposable d = myStruct` → box; `d.Dispose()` không sửa bản gốc. Dùng generic `where T : IDisposable` hoặc gọi trực tiếp.

**Vì sao / Khi nào chấp nhận boxing:** logging (`object`), reflection, API cũ. Không chấp nhận trên hot-path, LINQ trên `IEnumerable` của struct (enumerator box nếu đi qua interface).

### 3.2 Reference types

- Gồm: `class`, `interface`, `delegate`, `string`, **array**, `record class`.  
- **Copy-by-reference**: gán truyền địa chỉ object; 2 biến trỏ cùng object.  
- Quản lý bởi GC; có **nullable reference types (NRT)** ở mức ngôn ngữ để an toàn null (phần 10).

```csharp
var a = new Person { Name = "A" };
var b = a;                 // cùng object
b.Name = "B";
Console.WriteLine(a.Name); // "B"

b = new Person { Name = "C" }; // chỉ b đổi tham chiếu; a vẫn "B"
```

**Semantics:** biến reference = *handle*. `null` = handle rỗng → `NullReferenceException` khi dereference. Equality mặc định = **cùng instance** (`ReferenceEquals`), trừ khi override/`record`.

**Vì sao / Khi nào dùng class:** identity, đa hình, vòng đời phức tạp, dữ liệu lớn (tránh copy). Dùng `record class` khi muốn value-equality trên heap.

### 3.3 Built-in & Primitive

- **Numeric**: sbyte/byte, short/ushort, int/uint, long/ulong, nint/nuint, float, double, decimal, Half (System.Half).  
- **Others**: bool, char, string, object.  
- **BigInteger** (System.Numerics) cho số “vô hạn”.  
- Chú ý **decimal** (base-10) cho tiền tệ/chính xác, **double/float** (IEEE754) cho khoa học/hiệu năng.

| Kiểu | Điểm mấu chốt |
|---|---|
| `int`/`long` | Mặc định số nguyên; `int` 32-bit, `long` 64-bit |
| `nint`/`nuint` | Kích thước pointer (32/64 tùy process) — interop |
| `float`/`double` | IEEE754; **không** so sánh `==` tiền tệ |
| `decimal` | 128-bit thập phân; chậm hơn, đúng tiền |
| `Half` | 16-bit; ML/GPU, không thay `double` |

```csharp
double d = 0.1 + 0.2;
Console.WriteLine(d == 0.3);          // False
decimal m = 0.1m + 0.2m;
Console.WriteLine(m == 0.3m);         // True
```

**Pitfall:** `float` → `decimal` implicit không có; `int` → `double` implicit (có thể mất chính xác với số lớn). Overflow `int` mặc định *wrap* (unchecked) — xem `checked` ở [exceptions.md](exceptions.md).

---

## 4. Kiểu đặc biệt: `object`, `string`, `dynamic`, `void`, `null`

- **`object`**: gốc của mọi kiểu; có `ToString()`, `Equals()`, `GetHashCode()`. Bạn có thể override ở class/struct.  
- **`string`**: immutable, interning một phần (literal), thao tác nhiều → dùng `StringBuilder`. So sánh: `StringComparison`.  
- **`dynamic`**: defer binding qua DLR (runtime); linh hoạt nhưng mất an toàn kiểu/IntelliSense, có chi phí.  
- **`void`**: chỉ dùng làm **kiểu trả về**; trong IL là `System.Void`.  
- **`null`**: literal biểu diễn “không tham chiếu/không giá trị”; với value type chỉ có trong `Nullable<T>`.

```csharp
object o = "hi";
string s = (string)o;                  // explicit; fail → InvalidCastException
string? t = o as string;               // fail → null, không ném

dynamic d = 1;
d = d + "2";                           // runtime: "12" (binder)
// int n = d;                          // RuntimeBinderException nếu không convert được
```

**So sánh `object` vs `dynamic`:** cả hai hay box value; `object` cần cast tường minh (compile-time), `dynamic` gọi member lúc chạy. `dynamic` **không** phải thay cho generic.

**Pitfall `string`:** `==` trên `string` là value-equality (overload), khác class thường. Interning chỉ literal/`String.Intern` — đừng giả định mọi chuỗi trùng nội dung là cùng instance.

**Vì sao / Khi nào dùng `dynamic`:** COM, JSON lỏng, DLR. Không dùng cho API nội bộ C# — generic + interface rõ hơn.

---

## 5. Struct, `readonly struct`, `ref struct` (byref-like)

- **`struct`**: value type do người dùng định nghĩa; phù hợp dữ liệu nhỏ, bất biến, nhiều instance. Không nên vượt ~16–24 byte nếu dùng nhiều.  
- **`readonly struct`**: mọi field readonly; tối ưu copy/defensive-copy; an toàn bất biến.  
- **`ref struct`**: *byref-like* (ví dụ `Span<T>`, `ReadOnlySpan<T>`) với **ràng buộc nghiêm**:  
  - Không boxed, không dùng làm field của class, không dùng trong async/iterator *qua điểm treo* (`await`/`yield`), không capture lambda, không trong `IEnumerable<T>` thông thường.  
  - Mục tiêu: truy cập bộ nhớ hiệu quả, an toàn (stack-only).  
  - C# 13+: `ref struct` có thể implement interface nhưng **không** convert sang interface (sẽ box). Generic: `allows ref struct` — [§13.2](#132-allows-ref-struct-c-13).

```csharp
public readonly struct Point(int x, int y)
{
    public int X { get; } = x;
    public int Y { get; } = y;
}
```

**Semantics `readonly struct`:** compiler giả định không mutate qua `in`/readonly ref → **ít defensive copy**. `struct` thường + method không `readonly` khi truyền `in` có thể copy ngầm.

```csharp
public struct Mutable
{
    public int X;
    public void Bump() => X++;          // không readonly → in Mutable có thể copy
}

public readonly struct Immutable
{
    public int X { get; init; }
    public readonly int Twice => X * 2; // readonly method
}
```

**Vì sao / Khi nào dùng:**
- `struct` — dữ liệu nhỏ copy rẻ, không identity.
- `readonly struct` — mặc định nên dùng nếu struct bất biến.
- `ref struct` — chỉ khi giữ `Span`/buffer stack; xem [memory-spans.md](memory-spans.md).

---

## 6. Enum & `[Flags]`

- `enum` là value type, **nền tảng** là kiểu số (mặc định `int`).  
- Dùng `[Flags]` cho bitmask; giá trị nên là mũ 2 (1,2,4,8…).

```csharp
[Flags]
public enum FileAccess { None=0, Read=1, Write=2, Execute=4 }
var rights = FileAccess.Read | FileAccess.Write;
bool canWrite = rights.HasFlag(FileAccess.Write);
```

**Semantics:** enum **không** giới hạn giá trị — `(FileAccess)99` compile được. `[Flags]` chủ yếu ảnh hưởng `ToString()` (in `Read, Write`) và ý định API; `|` `&` vẫn chạy không cần attribute.

```csharp
FileAccess weird = (FileAccess)99;
Console.WriteLine(Enum.IsDefined(weird)); // False

// HasFlag box trên runtime cũ; .NET hiện đại nội tuyến. Vẫn có thể viết:
bool canWrite2 = (rights & FileAccess.Write) != 0;
```

**Pitfall:** mặc định underlying `int`; interop native `byte`/`uint` phải khai báo `enum E : byte`. Đừng dùng enum cho tập giá trị *mở* liên tục thay đổi — union/closed hierarchy (C# 15 preview) hoặc class hierarchy rõ hơn.

**Vì sao / Khi nào dùng:** tập đóng nhỏ, flags quyền. Không dùng enum làm “state machine lớn” nếu cần dữ liệu kèm theo (`Open(float percent)` → record/union).

---

## 7. Tuples & `ValueTuple`, deconstruction

- `ValueTuple<T1,...>` là **value type** (khác `Tuple<>` tham chiếu). Hỗ trợ **deconstruction**:

```csharp
(int x, int y) Get() => (10, 20);
var (a, b) = Get();
```

- Đặt tên phần tử: `(int X, int Y)` → `p.X`. Dùng cho trả nhiều giá trị, nhưng **API công khai** lớn nên cân nhắc type rõ nghĩa.

```csharp
(int Code, string Message) result = (404, "missing");
var (code, _) = result;                // discard

public readonly record struct Point(int X, int Y);
var p = new Point(1, 2);
var (x, y) = p;                        // deconstruct nếu có Deconstruct
```

**So sánh:**

| | `ValueTuple` | `Tuple<>` | named type / record |
|---|---|---|---|
| Heap | Không (struct) | Có | class: có / record struct: không |
| Tên field | metadata (mất khi qua `object`) | `Item1`… | ổn định |
| Public API | tạm ổn 2–3 field | tránh | **nên** |

**Pitfall:** `(int X, int Y)` vs `(int A, int B)` **cùng** `ValueTuple<int,int>` lúc runtime — tên không phải kiểu. Equality so sánh giá trị, không so tên.

**Vì sao / Khi nào dùng:** trả nhanh `(ok, value)`, destructure nội bộ. Public library: `record struct` hoặc type riêng.

---

## 8. Records (record class / record struct)

- **Record** cung cấp **equality theo giá trị**, `with`-expression, phù hợp mô hình dữ liệu bất biến.  
- `record class User(string Id, string Name);`  
- `record struct` là **value type** có semantics tương tự (từ C# 10).

```csharp
var u1 = new User("1","Alice");
var u2 = u1 with { Name = "Bob" };
Console.WriteLine(u1 == u2); // false (so sánh theo giá trị)
```

**Semantics:** compiler sinh `Equals`/`GetHashCode`/`ToString`/`Deconstruct`/`with`. `record class` vẫn là reference type — `with` **cấp phát object mới**. `record struct` copy-by-value; `with` copy struct.

```csharp
public record class User(string Id, string Name);
public readonly record struct Money(decimal Amount, string Currency);

User a = new("1", "Ann");
User b = new("1", "Ann");
Console.WriteLine(a == b);              // True (value equality)
Console.WriteLine(ReferenceEquals(a, b)); // False
```

**So sánh class thường:** class → identity; record class → dữ liệu. Đừng override `Equals` lung tung trên record trừ khi hiểu positional vs field extra.

**Pitfall:** record class chứa `List<T>` mutable — equality so **tham chiếu list**, hai list cùng phần tử vẫn có thể `!=`. `with` là *shallow copy*.

**Vì sao / Khi nào dùng:** DTO, message, value object. Entity có identity (cùng Id nhưng khác instance) → class + equality theo Id, không phải record positional mặc định.

---

## 9. Mảng (arrays): 1D, nhiều chiều, jagged, `Span<T>`

- **1D**: `T[]` phổ biến nhất.  
- **Nhiều chiều**: `T[,]` (rectangular).  
- **Jagged**: `T[][]` (mảng các mảng, linh hoạt).  
- Array là **reference type**, covariant (có rủi ro `ArrayTypeMismatchException`).

```csharp
object[] arr = new string[2];   // hợp lệ compile-time
arr[0] = 123;                   // runtime: ArrayTypeMismatchException
```

**So sánh layout:** `T[,]` một khối chữ nhật; `T[][]` mỗi hàng một mảng (hàng dài khác nhau, cache kém hơn nếu không đều). Hầu hết API hiện đại dùng `T[]` + `Span<T>` / `Range`.

**`Span<T>`/`ReadOnlySpan<T>`** (byref-like, `ref struct`): Cho phép truy cập bộ nhớ hiệu quả, an toàn, không cấp phát; dùng cho xử lý buffer, text. Không lưu trữ lâu dài, không dùng qua async/iterator.

```csharp
static void PrintLetters(ReadOnlySpan<char> span)
{
    foreach (var ch in span)
    {
        Console.Write($"{ch} ");
    }
    Console.WriteLine();
}

string text = "Hello";
char[] buffer = { 'W', 'o', 'r', 'l', 'd' };

// Cả 2 cái này đều compile được nhờ implicit conversion
PrintLetters(text);
PrintLetters(buffer);
```

> Từ C# 14, việc truyền string/T[] vào API nhận Span<T> / ReadOnlySpan<T> trở nên ‘tự nhiên’ hơn nhờ các implicit conversion mới.

**Pitfall covariance:** generic `List<T>` **không** covariant theo cách array (`List<string>` không phải `List<object>`). Đó là *tính năng* — tránh lỗ hổng array.

**Vì sao / Khi nào dùng mảng vs List:** buffer cố định, interop, hot-path. `List<T>` khi size đổi. Xem [collections-generics.md](collections-generics.md), [memory-spans.md](memory-spans.md).

---

## 10. Nullable Reference Types (NRT)

- Bật bằng `#nullable enable` hoặc trong project (`<Nullable>enable</Nullable>`).  
- Phân biệt `string` (non-null) và `string?` (có thể null); compiler sinh *warnings* giúp tránh `NullReferenceException`.  
- Chú ý các **annotation attributes** trong `System.Diagnostics.CodeAnalysis` như `NotNull`, `MaybeNull`, `MemberNotNull`, `NotNullWhen(bool)`,… để mô tả hợp đồng nullability cho API phức tạp.

```csharp
#nullable enable
string? name = GetNameOrNull();
if (name is not null)
{
    Console.WriteLine(name.Length); // an toàn
}
```

**Semantics:** NRT **không** đổi representation runtime của `string` — vẫn là reference, `null` vẫn gán được nếu bỏ qua warning. Compiler luồng-nhạy: sau `if (x is null) return;`, `x` được *state* non-null.

**So sánh `T?`:**

| | Value type `int?` | Reference `string?` |
|---|---|---|
| Runtime | `Nullable<int>` | vẫn `string` |
| `null` | `HasValue == false` | reference null |
| Ép buộc | runtime exception `.Value` | warning / NRE |

### 10.1 Annotation attributes

```csharp
public static bool TryGet(
    Dictionary<string, string> map,
    string key,
    [NotNullWhen(true)] out string? value)
    => map.TryGetValue(key, out value);

if (TryGet(map, "k", out var v))
    Console.WriteLine(v.Length); // v non-null khi true

[return: NotNullIfNotNull(nameof(s))]
public static string? Normalize(string? s)
    => s?.Trim();
```

`MaybeNull` / `AllowNull` / `DisallowNull` / `MemberNotNull` / `DoesNotReturnIf` — dùng khi luồng null không diễn đạt nổi bằng `T?` thuần.

### 10.2 Pitfalls NRT

- `null!` / `default!` **tắt** cảnh báo — chỉ cho deserialization/ORM.  
- Generic `T` không `class`/`struct`: `T?` nghĩa *khác* (unconstrained).  
- Array `string[]` vẫn nhận `null` phần tử; NRT trên array yếu.  
- `#nullable disable` trong file generated — đừng copy vào domain code.

**Vì sao / Khi nào dùng:** bật NRT toàn project (.NET 10 mặc định trên template hiện đại). Annotate biên API (`Try*`, factory). Không dựa NRT thay validation runtime trên input ngoài.

---

## 11. Khai báo biến & suy luận kiểu (`var`, target-typed, default literal)

- **`var`**: suy luận tại compile-time từ vế phải (vẫn *strongly-typed*). 
- **Target-typed `new`** (C# 9): `List<int> list = new();`  
- **Default literal** (C# 7.1): `T x = default;` (tự suy kiểu T).  
- **Anonymous types**: `new { Name = "A", Age = 1 }` (chỉ dùng nội bộ).

```csharp
var list = new List<int>();     // List<int>
List<int> list2 = new();        // target-typed new
int[] nums = [1, 2, 3];         // collection expression — C# 12+
string name = default!;         // null + NRT suppression — cẩn thận
```

**Pitfall:** `var` với `null` không suy được (`var x = null` lỗi). `var` với `dynamic` vẫn `dynamic`. Anonymous type không thể lộ qua public API (kiểu nội bộ assembly).

**Vì sao / Khi nào dùng `var`:** khi vế phải đã nêu kiểu (`new Dictionary<...>`). Khi kiểu là ý nghĩa chính (`IEnumerable<Customer>` vs implementation) — viết tường minh.

---

## 12. Giá trị mặc định (default values)

- `default(T)`:
  - Value type: tất cả bit 0 (`0`, `false`, `\0`, struct với field default).  
  - Reference type: `null`.  
- Nullable: `default(int?)` == `null`.

```csharp
int x = default;        // 0
string? s = default;    // null
DateTime dt = default;  // 01/01/0001 ...
```

**Semantics generic:**

```csharp
static T? Zero<T>() => default; // class → null; int → 0; int? → null
```

**Pitfall:** `default(DateTime)` không phải “chưa set” hữu ích — dùng `DateTime?`. `enum` default = `0` dù bạn không đặt tên `None = 0`. `record struct` default = field 0, **không** chạy primary ctor.

**Vì sao / Khi nào dùng:** khởi tạo generic, `out` chưa gán, array phần tử. Domain: prefer `null`/optional/`required` thay vì “0 nghĩa là thiếu”.

---

## 13. Generics & ràng buộc (`where`, `new()`, `struct/class/unmanaged/notnull`), phương sai (variance)

### 13.1 Ràng buộc (`where`)

- `where T : class` / `struct` / `unmanaged` / `notnull`  
- `where T : new()` (có ctor không tham số)  
- `where T : SomeBase, ISomeInterface` (đa ràng buộc)  

```csharp
T Create<T>() where T : new() => new T();
bool Equal<T>(T a, T b) where T : IEquatable<T> => a.Equals(b);
```

**`unmanaged`**: chỉ chứa các field unmanaged (không tham chiếu) → phù hợp interop/unsafe/fixed size.

**Semantics:** ràng buộc là *thu hẹp* tập `T`. `struct` loại trừ `Nullable<U>` (vì `Nullable<U>` không thỏa `struct` constraint theo nghĩa “non-nullable value type”). `class` gồm class, interface, delegate, array — không gồm struct.

```csharp
static void NeedsUnmanaged<T>(T value) where T : unmanaged { }
NeedsUnmanaged(1);                 // OK
// NeedsUnmanaged("a");            // lỗi — string là reference
```

**Pitfall:** `new()` không thấy primary ctor có tham số — type chỉ có `Person(string name)` **không** thỏa `new()`. Factory/`Activator` khác `new()`.

### 13.2 `allows ref struct` (C# 13)

**Anti-constraint:** *mở* thêm `ref struct` (`Span<T>`) làm type argument — mặc định generic **cấm** `ref struct`.

```csharp
static void Process<T>(scoped T value)
    where T : allows ref struct
{
    // T không box được; không gán field class; không capture qua await
}

static string Lower(ReadOnlySpan<char> input)
    => string.Create(input.Length, input, static (dst, src) => src.ToLowerInvariant(dst));
    // TState = ReadOnlySpan<char> nhờ allows ref struct trên string.Create
```

Compiler áp **ref-safety** lên mọi chỗ dùng `T`: không box, không array `T[]` nếu `T` có thể là ref struct, không field trên class.

**So sánh:** `where T : struct` **không** gồm `ref struct`. Phải ghi thêm `allows ref struct`.

**Vì sao / Khi nào dùng:** thư viện generic trên `Span`/`ReadOnlySpan` (parser, hash). BCL: `string.Create<TState>(..., TState state, ...)` với `TState : allows ref struct`. Không thêm “cho vui” — API trở nên khó dùng (caller phải `scoped`).

Xem [memory-spans.md](memory-spans.md).

### 13.3 Phương sai (variance)

- `out` (covariant — chỉ *xuất* `T`) cho **interfaces/delegates**: `IEnumerable<out T>`.  
- `in` (contravariant — chỉ *nhập* `T`): `IComparer<in T>`.  
- Giúp tái sử dụng kiểu generic giữa kế thừa: `IEnumerable<string>` có thể dùng nơi yêu cầu `IEnumerable<object>`.

> Lưu ý: **Array covariance** tồn tại nhưng *nguy hiểm* (mục 9).

**PECS (Producer Extends, Consumer Super)** — cùng ý Java, thuật ngữ C# là `out`/`in`:

- **Producer** (`IEnumerable<out T>`): “đưa ra T” → `IEnumerable<string>` dùng như `IEnumerable<object>` (string *là* object).
- **Consumer** (`IComparer<in T>`): “nhận T” → `IComparer<object>` dùng như `IComparer<string>` (so sánh object thì so được string).

```csharp
IEnumerable<string> names = ["a", "b"];
IEnumerable<object> objs = names;          // covariant out

IComparer<object> cmp = Comparer<object>.Default;
IComparer<string> sc = cmp;                // contravariant in
```

**Pitfall:** class generic (`List<T>`) **invariant**. `List<string>` không gán `List<object>` — đúng, vì `Add(object)` sẽ phá type safety. Chỉ interface/delegate được khai báo `in`/`out`. Chi tiết: [collections-generics.md §10.2](collections-generics.md#102-phương-sai-variance-outin--pecs).

---

## 14. Namespace, `using`, `global using`, `extern alias`

- **Namespace**: tổ chức type theo không gian tên; từ C# 10 có **file-scoped namespace**:

```csharp
namespace MyApp.Core; // file-scoped
```

- **`using`**: import namespace; **`global using`** (C# 10) áp dụng cho toàn project.  
- **`extern alias`**: phân biệt 2 assembly có cùng namespace/type:

```csharp
extern alias LibA;
extern alias LibB;
using A = LibA::Company.Product;
using B = LibB::Company.Product;
```
- **`using static`**: cho phép bạn sử dụng trực tiếp các thành phần static bên trong các lớp mà không cần chỉ định tên lớp.  

```csharp
using static System.Console;
using static System.Math;
class Program
{
    static void Main()
    {
        WriteLine(Sqrt(3*3 + 4*4));
    }
}
```

Trong ví dụ trên chúng ta không cần viết Console.WriteLine hay Math.Sqrt.

**Pitfall:** `global using` quá rộng làm IntelliSense/ambiguity (`Timer` WinForms vs Threading). File-scoped namespace không lồng type ngoài file. `extern alias` cần `Aliases` trong `csproj` — hiếm, chỉ khi duplicate type.

**Vì sao / Khi nào dùng file-scoped:** mặc định file một namespace. `extern alias` chỉ khi hai NuGet đụng namespace.

---

## 15. Chuyển đổi & ép kiểu: implicit/explicit, user-defined, pattern matching

### 15.1 Chuyển đổi chuẩn

- **Implicit**: an toàn, không mất dữ liệu (`int -> long`).  
- **Explicit**: có thể mất dữ liệu, cần cast (`double -> int`).  
- **Parse/TryParse** cho string → số/ngày…

**Thứ tự hay gặp:** identity → implicit numeric → enum ↔ underlying → implicit reference (derived → base) → boxing → explicit ngược lại → user-defined.

```csharp
long l = 3;                 // implicit int → long
int i = (int)l;             // explicit; truncate nếu vượt
int n = checked((int)l);    // OverflowException nếu tràn

object o = "x";
string s = (string)o;       // explicit reference; sai kiểu → InvalidCastException
string? s2 = o as string;   // không ném; null nếu fail — chỉ reference type
```

**So sánh `as` vs `is` vs cast:** `as` không ném, chỉ class/interface/nullable. Pattern `is string s` vừa test vừa bind — ưa dùng hơn `as` + null check. Cast `(T)` ném khi sai.

**C# 14:** implicit `T[]`/`string` → `Span`/`ReadOnlySpan` — [§9](#9-mảng-arrays-1d-nhiều-chiều-jagged-spant), [memory-spans.md §7](memory-spans.md#7-c-14--implicit-span-conversions).

### 15.2 User-defined conversion

Trong class/struct, bạn có thể định nghĩa:

```csharp
public readonly struct Dollars
{
    public decimal Amount { get; }
    public Dollars(decimal a) => Amount = a;
    public static implicit operator Dollars(decimal a) => new(a);
    public static explicit operator decimal(Dollars d) => d.Amount;
}
```

**Pitfall:** `implicit` chỉ khi **không mất thông tin** và không ném. Conversion ném/`lossy` → `explicit`. Compiler **không** xâu chuỗi hai user-defined conversion. Tránh implicit giữa domain type dễ nhầm (`UserId` ↔ `int`).

**Vì sao / Khi nào dùng:** wrapper mỏng (`Degrees`, `Meters`). Không thay factory có validation phức tạp.

### 15.3 Pattern matching

- `is` pattern, **`switch` expression**, property/relational/list patterns.  
- Vừa kiểm tra kiểu, vừa “rút” giá trị một cách an toàn, ngắn gọn.

```csharp
object x = Get();
if (x is Person { Age: >= 18 } p)
    Console.WriteLine($"{p.Name} is adult");
```

```csharp
static string Classify(object value) => value switch
{
    null => "none",
    int n and > 0 => $"pos {n}",
    string { Length: > 0 } s => s,
    IEnumerable<int> xs => $"seq {xs.Count()}",
    _ => "other"
};

int[] row = [1, 2, 3];
if (row is [1, .. var rest, 3])
    Console.WriteLine(rest.Length); // 1 — list pattern
```

**So sánh với cast:** pattern không ném; `switch` exhaustiveness trên union/`closed` (C# 15 preview) mạnh hơn `if-else` + `_`.

**Pitfall:** `switch` trên `object` **không** exhaustive trừ `closed`/union. `is T` với `T` nullable value: `is int?` ít dùng — `is int n` đã phủ `HasValue`.

Union **Try-Both**: [§18.3](#183-pattern-matching--tính-đầy-đủ-exhaustiveness--try-both).

---

## 16. Unsafe & unmanaged types (overview), function pointers

- **Unsafe context** (`unsafe { ... }`): dùng con trỏ (`T*`), `stackalloc`, `fixed`. Chỉ dùng khi **thật cần** (interop/hiệu năng đặc biệt).  
- **Unmanaged types**: không chứa reference; có thể dùng trong `sizeof`, `stackalloc`, `unmanaged` constraint.  
- **Function pointers** (C# 9, unsafe): `delegate*<int, void>` — hiệu năng cao khi interop/native, nhưng mất an toàn kiểu ở C# mức cao; đa phần nên dùng **delegate**.

**Vì sao / Khi nào dùng:** P/Invoke, serialization zero-copy. Mặc định: `Span`/`MemoryMarshal`. **C# 15 PREVIEW** tách “khai báo pointer” khỏi “dereference” — [memory-spans.md §9.1](memory-spans.md#91-memory-safety-c-15-preview).

---

## 17. Sơ đồ “type tree” (ASCII)

```
object
├─ Value types (System.ValueType)
│  ├─ Primitives: bool, char, sbyte/byte, short/ushort, int/uint, long/ulong, nint/nuint, float, double, decimal, Half
│  ├─ Enums: enum E : int { ... }
│  ├─ Structs: DateTime, Guid, ValueTuple<...>, custom struct/readonly struct
│  └─ Byref-like: ref struct (Span<T>, ReadOnlySpan<T>)
└─ Reference types
   ├─ class (String, Exception, Stream, List<T>, ...)
   │  ├─ record class
   │  └─ arrays: T[], T[,], T[][] (covariant, ref type)
   ├─ interface (IDisposable, IEnumerable<T>, ...)
   ├─ delegate (Action, Func<...>, custom delegates)
   └─ dynamic (runtime-bound)
```

Union (C# 15 preview) không thay cây này — chúng *ghép* case type đã có. `closed` hierarchy vẫn là class/record trên nhánh reference.

---

## 18. Union types (C# 15) — PREVIEW

> **PREVIEW (.NET 11 / C# 15)** — chưa phải baseline .NET 10 / C# 14.  
> Yêu cầu: .NET 11 Preview (hoặc tương đương) + `<LangVersion>preview</LangVersion>`. Cú pháp/semantics có thể đổi trước GA.

C# 15 giới thiệu **union types** — kiểu có thể là đúng một trong số các kiểu thành viên đã xác định (tập đóng). Tương tự *discriminated unions* (F#) / *union types* (TypeScript), theo phong cách C#.

### 18.1 Cú pháp khai báo

Dùng từ khóa `union` để khai báo:

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);
```

`Pet` là một union type có thể chứa giá trị thuộc một trong ba kiểu: `Cat`, `Dog`, hoặc `Bird`.

**Vì sao / Khi nào dùng:** mô hình “đúng một trong N dạng” *không* cần base class chung (kết quả parse, event, đơn vị đo). Cần cây OOP + member dùng chung → `closed` ở [oop.md §2.6](oop.md#26-closed-hierarchies-c-15-preview).

### 18.2 Gán giá trị & chuyển đổi ngầm định

Mỗi kiểu thành viên (case type) có thể được chuyển đổi ngầm định (implicit conversion) sang union type:

```csharp
Pet pet  = new Dog("Rex");        // hợp lệ
Pet pet2 = new Cat("Whiskers");   // hợp lệ
// Pet pet3 = new Shark("Jaws"); // lỗi compile-time nếu Shark không thuộc union
```

**Semantics:** union là wrapper (thường box case value-type trừ khi custom non-boxing `TryGetValue`). Tập **đóng** — thêm case là đổi kiểu, mọi `switch` phải cập nhật.

### 18.3 Pattern matching & tính đầy đủ (exhaustiveness) — Try-Both

Compiler biết tất cả các case của union, nên **bắt buộc** xử lý đủ mọi trường hợp trong `switch` mà không cần arm `_` / `default`:

```csharp
string name = pet switch
{
    Dog d  => d.Name,
    Cat c  => c.Name,
    Bird b => b.Name,
};
```

Nếu thêm một case mới vào `Pet`, compiler sẽ cảnh báo tại tất cả `switch` chưa xử lý case đó.

**Try-Both matching (Preview 7+):** khi pattern áp lên giá trị union, compiler thử pattern trên **chính instance union**; nếu fail thì thử trên **`Value` chứa bên trong**. Do đó `pet is Dog d` và pattern trên wrapper đều có thể khớp — xác nhận bản preview (có thể tinh chỉnh trước GA).

Áp dụng type / declaration / list / recursive pattern. Ý tưởng: vừa nhận diện `Pet`, vừa nhận diện `Cat` bên trong.

```csharp
public record class Dog(string Name);
public record class Cat(int Lives);
public union Pet(Dog, Cat);

Pet pet = new Cat(9);

if (pet is Pet)                         // khớp CHÍNH union
    Console.WriteLine("got a pet");

if (pet is Cat { Lives: > 0 } cat)      // Try-Both: fail trên wrapper → thử Value
    Console.WriteLine($"cat has {cat.Lives} lives");

if (pet is Dog)                         // false — Value là Cat
    Console.WriteLine("dog");
```

**So sánh với pattern thường:** trên `object o = cat`, `o is Pet` chỉ đúng nếu runtime type là union wrapper. Trên biến kiểu `Pet`, `is Pet` luôn true (chính instance). `is Cat` unwrap.

Custom union (struct discriminator, tránh box): implement `HasValue` + `TryGetValue(out T)` — compiler ưu tiên non-boxing access thay vì `Value` kiểu `object`.

Runtime: `UnionAttribute` / `IUnion` (`System.Runtime.CompilerServices`) — BCL từ các preview gần đây. Learn: [C# 15 unions](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [blog](https://devblogs.microsoft.com/dotnet/csharp-15-union-types/).

Hierarchy OOP đóng (cùng exhaustiveness nhưng *kế thừa*): xem `closed` ở [oop.md §2.6](oop.md#26-closed-hierarchies-c-15-preview).

**Pitfall preview:** `var` pattern / một số property pattern không unwrap như type pattern — đọc speclet từng bản SDK. Đừng dùng union trên production .NET 10.

### 18.4 Đặc điểm nổi bật

| Đặc điểm | Mô tả |
|-----------|-------|
| Không cần thừa kế chung | Các kiểu thành viên không cần có lớp cha hay interface chung. |
| Tập đóng (closed set) | Không thể thêm case từ bên ngoài → tăng type safety. |
| Nullable awareness | Nếu case types có nullable, compiler yêu cầu xử lý trường hợp `null`. |
| Try-Both | Pattern thử wrapper rồi `Value` (Preview 7+). |

### 18.5 So sánh với các kỹ thuật trước đây

Trước C# 15, để mô hình hóa một "loại có thể là A hoặc B", người ta thường dùng:
- **Interface/abstract class**: đòi hỏi thừa kế, tập mở (open set), không có exhaustiveness.
- **OneOf<T1,T2,...>** (thư viện bên thứ ba): tương tự về ý tưởng nhưng không tích hợp sâu vào ngôn ngữ.
- **`closed` hierarchy**: vẫn OOP; case *phải* derived trong cùng assembly.

| | Union | `closed` class | Interface |
|---|---|---|---|
| Case không cùng base | ✅ | ❌ | ❌ (cần implement) |
| Exhaustive switch | ✅ | ✅ | ❌ |
| Thêm case ngoài assembly | ❌ | ❌ | ✅ |
| Shared members | hạn chế | ✅ (base) | ✅ |

Union types C# 15 (preview) giải quyết các hạn chế đó với sự hỗ trợ trực tiếp từ compiler — chỉ dùng trên toolchain preview, không phụ thuộc vào baseline .NET 10.
