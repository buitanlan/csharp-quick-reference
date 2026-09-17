# Memory, Span & unsafe

Tham chiếu nâng cao về **bộ nhớ managed**, `ref`/`Span`/`Memory`, và **unsafe** trên baseline **.NET 10 / C# 14**.  
Tập trung semantics, lifetime, pitfalls (tương tự chương pointers bên Go) — không phải tutorial GC đầy đủ.

> **Baseline:** .NET **10** / C# **14** (first-class span conversions). Nhiều API (`Span`, `Memory`, `scoped`) từ C# 7.2–11.  
> **C# 15 PREVIEW:** [§9.1 Memory safety](#91-memory-safety-c-15-preview) — `unsafe` gắn với *dereference*, không còn với *sự tồn tại pointer*.

---

## Mục lục

- [Memory, Span \& unsafe](#memory-span--unsafe)
  - [Mục lục](#mục-lục)
  - [1. Stack vs Heap \& GC (generational)](#1-stack-vs-heap--gc-generational)
  - [2. `ref` locals, `ref` returns, `ref` fields](#2-ref-locals-ref-returns-ref-fields)
  - [3. `Span<T>` \& `ReadOnlySpan<T>`](#3-spant--readonlyspant)
  - [4. `Memory<T>` \& `ReadOnlyMemory<T>`](#4-memoryt--readonlymemoryt)
  - [5. Span vs Memory — chọn cái nào](#5-span-vs-memory--chọn-cái-nào)
  - [6. `stackalloc` \& fixed buffers](#6-stackalloc--fixed-buffers)
  - [7. C\# 14 — implicit Span conversions](#7-c-14--implicit-span-conversions)
  - [8. `scoped` (C\# 11) — lifetime](#8-scoped-c-11--lifetime)
  - [9. `unsafe` \& pointers — overview](#9-unsafe--pointers--overview)
    - [Khi **KHÔNG** dùng unsafe](#khi-không-dùng-unsafe)
    - [9.1 Memory safety (C# 15 preview)](#91-memory-safety-c-15-preview)
  - [10. `ArrayPool<T>` \& `MemoryMarshal`](#10-arraypoolt--memorymarshal)
  - [11. Pitfalls thường gặp](#11-pitfalls-thường-gặp)
  - [12. Cheat sheet chọn API](#12-cheat-sheet-chọn-api)

---

## 1. Stack vs Heap & GC (generational)

### 1.1 Stack

- Mỗi thread có **stack**: frame method, biến local, return address, đôi khi value type local.  
- Cấp phát/hủy LIFO theo scope — **không** qua GC.  
- Giới hạn kích thước → StackOverflow nếu đệ quy sâu / `stackalloc` quá lớn.

**Vì sao / Khi nào quan tâm:** `stackalloc`, đệ quy, struct lớn trên local. Không phải mọi value type đều ở đây — xem [typesystem.md §2](typesystem.md#2-bức-tranh-bộ-nhớ-stackmanaged-heap--gc).

### 1.2 Managed heap

- **Reference type** (`class`, mảng, boxed value…) sống trên **managed heap**.  
- Biến local kiểu tham chiếu trên stack chỉ giữ **reference**.  
- Value type là field của object / phần tử mảng → nằm **trong** heap object đó.

> Value type **≠** “luôn trên stack”. Ngữ cảnh quyết định vị trí lưu trữ.

### 1.3 GC thế hệ

| Thế hệ | Đặc điểm |
|---|---|
| **Gen 0** | Object mới; thu gom thường xuyên, rẻ |
| **Gen 1** | Sống sót 1 lần GC; đệm short/long-lived |
| **Gen 2** | Sống lâu; full GC đắt hơn |
| **LOH** | Object lớn (≈ ≥ 85 KB); compact đắt |

**Giả thuyết thế hệ:** hầu hết object chết trẻ → quét Gen 0 mang lại nhiều bộ nhớ với chi phí thấp.

- **Allocation:** bump pointer trên ephemeral segment (nhanh).  
- **Collection:** mark (+ compact tùy chế độ).  
- **Pinned** (`fixed`, GCHandle, interop) cản compact → tránh pin lâu.  
- Finalizer trì hoãn thu hồi — ưu tiên `using` / `IAsyncDisposable`.

```csharp
var list = new List<byte>(4096); // heap
int x = 42;                      // value local — thường stack (trừ capture/async)
```

**Hot path thực dụng:** giảm allocation → `Span`, `stackalloc`, `ArrayPool`, `ValueTask` (xem [async.md](async.md)); tránh LINQ/`string` tạm trong vòng nóng.

---

## 2. `ref` locals, `ref` returns, `ref` fields

### 2.1 `ref` local & `ref` return — **C# 7+**

Tham chiếu tới ô nhớ **đã tồn tại** — không copy value lớn. Đây là **alias**, không phải copy — đối lập mặc định by-value ở [methods.md §4](methods.md#4-truyền-theo-giá-trị-và-truyền-theo-tham-chiếu).

```csharp
ref int Find(Span<int> data, int value)
{
    for (int i = 0; i < data.Length; i++)
        if (data[i] == value) return ref data[i];
    throw new InvalidOperationException();
}

int[] arr = { 1, 2, 3 };
ref int slot = ref Find(arr, 2);
slot = 99; // sửa arr[1]
```

- `ref` / `in` / `out` trên tham số; `in` = readonly ref; `ref readonly` = tham chiếu chỉ đọc.  
- Compiler chặn trả `ref` tới local sắp chết (*safe-to-return*).

```csharp
ref int Bad()
{
    int local = 1;
    return ref local; // lỗi biên dịch
}
```

**Semantics:** `ref int slot = ref arr[1]` — `slot` là tên khác của `arr[1]`. Gán `slot = 99` ghi vào mảng. `int copy = arr[1]` là **bản sao**.

### 2.2 `ref` fields trong `ref struct` — **C# 11+**

```csharp
ref struct ByRefPair
{
    public ref int Left;
    public ref int Right;
    public ByRefPair(ref int left, ref int right)
    {
        Left = ref left;
        Right = ref right;
    }
}
```

- `ref` field **chỉ** trong `ref struct`. Kết hợp `scoped` để siết lifetime (mục 8).

### 2.3 `ref struct` (byref-like)

```csharp
public ref struct Utf8Parser
{
    private ReadOnlySpan<byte> _input;
    public Utf8Parser(ReadOnlySpan<byte> input) => _input = input;
}
```

**Không** được: boxing lên `object`/interface thông thường; field của `class`; capture lambda; sống qua `async`/`yield` như biến treo qua suspension.  
(`allows ref struct` generic — C# 13+ — chỉ trong ngữ cảnh hạn chế; [typesystem.md §13.2](typesystem.md#132-allows-ref-struct-c-13).)

**Vì sao / Khi nào dùng `ref`:** tránh copy struct lớn; sửa phần tử mảng/`Span` tại chỗ. Không dùng để “trả nhiều giá trị” — `out`/`tuple` rõ hơn.

---

## 3. `Span<T>` & `ReadOnlySpan<T>`

### 3.1 Bản chất

`Span<T>` / `ReadOnlySpan<T>` là **`ref struct`**: descriptor `(ref T, length)` tới nhớ liên tục — **không sở hữu**, không cấp phát thêm.

Có thể trỏ: `T[]`, `stackalloc`, native/pinned, `string` (`ReadOnlySpan<char>`), buffer unmanaged qua API phù hợp.

```csharp
int[] data = { 10, 20, 30, 40 };
Span<int> all = data;              // C# 14 implicit (mục 7)
Span<int> mid = data.AsSpan(1, 2); // {20,30}
mid[0] = 99;                       // data[1] == 99

ReadOnlySpan<char> name = "Alice"; // C# 14
if (name.StartsWith("Al"))
    Console.WriteLine(name[2..]);
```

**Semantics:** `Span` = *view*. Slice O(1) — không copy phần tử. Ghi `span[i]` ghi vào bộ nhớ gốc (mảng/stack/native). `ReadOnlySpan` cấm ghi qua API span (vẫn có thể ghi gốc nếu bạn giữ `T[]`).

### 3.2 API & vì sao nhanh

- Indexer, `Length`, `Clear`/`Fill`, `CopyTo`/`TryCopyTo`, `Slice` **O(1)**.  
- Parsing: `int.TryParse(ReadOnlySpan<char>, …)` — tránh `Substring`.  
- Zero-alloc cho lát cắt; JIT hay tối ưu biên kiểm trong vòng quen thuộc.

```csharp
static bool TryReadInt(ReadOnlySpan<char> text, out int value)
    => int.TryParse(text, out value);

static ReadOnlySpan<char> AfterColon(ReadOnlySpan<char> line)
{
    int i = line.IndexOf(':');
    return i < 0 ? [] : line[(i + 1)..].Trim();
}
```

**So sánh `Substring`:** `s.Substring(1, 3)` cấp phát `string` mới. `s.AsSpan(1, 3)` / implicit C# 14 không cấp phát.

**Pitfall:** `foreach` trên `Span` OK; **không** đưa `Span` vào `IEnumerable<T>` (box). Không dùng làm field class.

**Vì sao / Khi nào dùng Span:** parse, slice, hot-path đồng bộ. Cần lưu qua `await` → `Memory`.

---

## 4. `Memory<T>` & `ReadOnlyMemory<T>`

Khi cần **lưu** lát cắt qua field / async / heap:

| Kiểu | `ref struct`? | Field / async? |
|---|---|---|
| `Span<T>` / `ReadOnlySpan<T>` | Có | Không |
| `Memory<T>` / `ReadOnlyMemory<T>` | Không | Có |

```csharp
async Task ConsumeAsync(ReadOnlyMemory<byte> memory)
{
    ReadOnlySpan<byte> span = memory.Span; // dùng đồng bộ, ngắn hạn
    int sum = 0;
    foreach (var b in span) sum += b;

    await Task.Delay(1);
    // sau await: lấy memory.Span mới — đừng giữ Span cũ
}
```

- `Memory<T>.Span` tạo `Span` **tạm**. Owner: mảng, `IMemoryOwner<T>`, pool.  
- Pipeline mạng/file async thường truyền `ReadOnlyMemory<byte>`.

**Semantics:** `Memory<T>` là *token* (object + offset + length, hoặc GCHandle/owner). Có thể copy `Memory` struct (nhẹ) giữa field/queue. **Không** tự pin; `Pin()` trả `MemoryHandle` — nhớ `Dispose`.

```csharp
public sealed class BufferSlot
{
    public required Memory<byte> Payload { get; init; } // OK — không phải Span
}

static async Task WriteAsync(Stream stream, ReadOnlyMemory<byte> data)
{
    await stream.WriteAsync(data); // API hiện đại nhận Memory
}
```

**Pitfall:** `memory.Span` sau `await` có thể trỏ vùng đã `Return` pool nếu owner giải phóng. Lifetime owner ≥ mọi `Span` lấy ra.

**Vì sao / Khi nào dùng Memory:** `async`, lưu field, `Channel<ReadOnlyMemory<byte>>`. Xử lý CPU thuần trong method sync → nhận `Span` (caller `.Span` tại chỗ).

---

## 5. Span vs Memory — chọn cái nào

Cùng *ý tưởng* (lát cắt nhớ liên tục), khác **lifetime & nơi sống**:

| Câu hỏi | `Span<T>` | `Memory<T>` |
|---|---|---|
| Sống trên heap như field class? | Không (`ref struct`) | Có |
| Qua `await` / `yield`? | Không (treo qua suspension) | Có |
| Tạo từ `stackalloc`? | Có | Không trực tiếp |
| Zero-alloc slice đồng bộ | ✅ | Token; `.Span` khi dùng |
| Implicit C# 14 từ `T[]`/`string` | ✅ | `T[]` → `Memory<T>` (API BCL); string → `ReadOnlyMemory<char>` |
| Chi phí | 2 field (ref + len), stack | struct nhỏ + object gốc |

```csharp
// Đồng bộ: Span
static int Checksum(ReadOnlySpan<byte> data)
{
    int s = 0;
    foreach (var b in data) s += b;
    return s;
}

// Bất đồng bộ: Memory — bóc Span từng khúc
static async Task<int> ChecksumAsync(ReadOnlyMemory<byte> data)
{
    int s = 0;
    s += Checksum(data.Span);     // xong trước await
    await Task.Yield();
    return s;
}
```

**Quy tắc thực dụng:**
1. API **sync** nội bộ → `ReadOnlySpan<T>` / `Span<T>`.
2. API **async** hoặc lưu trữ → `ReadOnlyMemory<T>` / `Memory<T>`.
3. Cả hai: overload `Span` + `Memory` (Memory gọi `.Span` khi sync) — C# 14 conversion làm `T[]` tìm đúng overload (xem mục 7).

**Vì sao không chỉ Memory:** mỗi lần `.Span` là view mới; `ref struct` cho compiler *chứng minh* không escape — tối ưu + an toàn stackalloc. Memory không chứng minh được điều đó.

---

## 6. `stackalloc` & fixed buffers

### 6.1 `stackalloc`

```csharp
Span<byte> buffer = stackalloc byte[256]; // safe — C# 7.2+
buffer.Clear();
```

- Nhanh, không GC. Không trả `Span` từ `stackalloc` ra ngoài method.  
- Tránh size lớn / theo input không chặn. Pattern: ngưỡng rồi fallback pool:

```csharp
Span<byte> bytes = length <= 512
    ? stackalloc byte[length]
    : pool.Rent(length).AsSpan(0, length);
```

**Pitfall:** `stackalloc` theo `userLength` không chặn = DoS / StackOverflow. Ngưỡng cứng (256–1024) là bắt buộc trên input ngoài.

**Vì sao / Khi nào dùng:** buffer tạm < ~1 KB, không async. Lớn hơn → `ArrayPool`.

### 6.2 Fixed buffer & `InlineArray`

```csharp
unsafe struct Packet
{
    public fixed byte Header[8]; // cần unsafe — interop
    public int Length;
}

[System.Runtime.CompilerServices.InlineArray(8)] // C# 12 — ưu tiên khi được
struct Header8 { private byte _element0; }
```

**So sánh:** `fixed` buffer = C layout, cần `unsafe` để index. `InlineArray` = managed, indexer an toàn, dùng với `Span` — baseline 14 nên ưu tiên `InlineArray` trừ P/Invoke.

---

## 7. C# 14 — implicit Span conversions

**Phiên bản C#:** **14** (.NET 10) — *first-class span types*.

Implicit span conversions (standard) → overload resolution, type inference, extension method “hiểu” Span sâu hơn; ít `.AsSpan()` thủ công.

| Từ | Sang |
|---|---|
| `T[]` | `Span<T>` |
| `T[]` | `ReadOnlySpan<U>` (covariance khi hợp lệ) |
| `Span<T>` | `ReadOnlySpan<U>` |
| `ReadOnlySpan<T>` | `ReadOnlySpan<U>` |
| `string` | `ReadOnlySpan<char>` |

```csharp
static int Sum(ReadOnlySpan<int> values)
{
    var total = 0;
    foreach (var v in values) total += v;
    return total;
}

int[] nums = { 1, 2, 3 };
Console.WriteLine(Sum(nums)); // array → ReadOnlySpan

static void Show(ReadOnlySpan<char> s) => Console.WriteLine(s.Length);
Show("hello"); // string → ReadOnlySpan<char>
```

**Semantics:** conversion *standard* (không user-defined) — tham gia overload resolution như numeric implicit. `string` → `ReadOnlySpan<char>` **không** copy ký tự.

```csharp
static void F(string s) => Console.WriteLine("string");
static void F(ReadOnlySpan<char> s) => Console.WriteLine("span");

F("x"); // C# 14: có thể chọn span — kiểm tra overload của bạn
```

> Nâng C# 14 có thể đổi overload resolution chỗ có nhiều overload `T[]`/`Span`/`ReadOnlySpan` — chạy test kỹ.

**Pitfall:** extension method trên `ReadOnlySpan<char>` bỗng thắng extension trên `string` (hoặc ngược) sau khi nâng ngôn ngữ. API thư viện: đánh `[OverloadResolutionPriority]` (C# 13) nếu cần giữ `string` overload.

**Vì sao / Khi nào dựa conversion:** viết API mới nhận `ReadOnlySpan<T>` — caller truyền `T[]`/`string` tự nhiên. Codegen cũ còn `.AsSpan()` vẫn đúng, chỉ dài hơn.

---

## 8. `scoped` (C# 11) — lifetime

`scoped` giới hạn **không cho tham chiếu escape** khỏi scope — API `ref`/`Span` linh hoạt mà vẫn an toàn.

```csharp
void Process(scoped Span<int> data)
{
    data[0] = 1;
    // không gán vào field sống lâu hơn (theo quy tắc escape)
}

Span<int> stack = stackalloc int[4];
Process(stack); // OK nhờ scoped trên tham số
```

- **`scoped` parameter:** cấm return/ref escape; cho phép truyền `stackalloc`.  
- **`scoped` local** + **ref fields:** chứng minh không lưu ref nguy hiểm.

**Semantics (ý tưởng):** compiler gán *lifetime* cho mỗi ref/`ref struct`. `scoped` = “lifetime ≤ method hiện tại”. Không `scoped`, tham số `Span` có thể bị coi là *gọi lên từ caller* — không nhận `stackalloc` vì stackalloc chết khi method này return… thực ra stackalloc của *caller* sống suốt caller; vấn đề là callee **trả** span đó ra ngoài callee trong khi callee đã return. `scoped` nói: tôi **không** trả / lưu escape.

```csharp
ref struct Holder { public Span<int> Data; }

// Không scoped: compiler sợ Process lưu span vào field tĩnh / trả ra
void Ok(scoped Span<int> data)
{
    Span<int> local = data; // scoped local — không return local
    local.Fill(0);
}

// typical pattern C# 13 generic:
void Hash<TBuffer>(scoped TBuffer buffer)
    where TBuffer : allows ref struct { /* ... */ }
```

**So sánh không ghi `scoped`:** nhiều API `Span` tham số *đã* scoped ngầm (ngôn ngữ coi tham số `ref struct` theo quy tắc mặc định). Ghi tường minh khi compiler báo CS8352/CS9077… hoặc khi nhận `stackalloc` từ caller.

> Thư viện low-level: nếu compiler báo lifetime, đừng bỏ `ref struct`/`scoped` chỉ để “cho compile”.

**Pitfall:** `scoped` không phải “thread-safe” hay “pinned”. Nó chỉ là ràng buộc escape lúc compile.

**Vì sao / Khi nào dùng:** viết API nhận `Span`/`ref struct`/generic `allows ref struct`. Caller `stackalloc` mới truyền được.

---

## 9. `unsafe` & pointers — overview

```xml
<AllowUnsafeBlocks>true</AllowUnsafeBlocks>
```

```csharp
unsafe void Fill(byte* dest, int length, byte value)
{
    for (int i = 0; i < length; i++) dest[i] = value;
}

unsafe void Checksum(byte[] data)
{
    fixed (byte* p = data) // pin — giữ khối ngắn
    {
        byte b = p[0];
    }
}

// Function pointer — C# 9+
unsafe
{
    delegate*<int, int, int> add = &Add;
    int r = add(1, 2);
}
static int Add(int a, int b) => a + b;
```

**Semantics baseline 14:** pointer arithmetic, dereference, `fixed` buffer index — tất cả trong `unsafe`. CLR **không** bounds-check `p[i]` trên `T*` → buffer overrun = UB / lỗ hổng.

### Khi **KHÔNG** dùng unsafe

- Nghiệp vụ thường, web/CRUD.  
- “Tối ưu” khi chưa đo profiler.  
- Khi `Span`/`Memory`/`MemoryMarshal`/`Unsafe` (managed helpers) đủ.  
- Team không quen review memory safety.
- Cần `async` giữ pointer qua `await` (pin + lifetime = ác mộng).
- Input chưa tin cậy mà bạn tính pointer bằng số do user đưa.

> **Mặc định hiện đại:** `Span` / `ref struct` / `stackalloc` safe. `unsafe` cho interop / layout / buffer cực đoan sau khi đo.

**So sánh `System.Runtime.CompilerServices.Unsafe`:** helper managed (`Add`, `As`, `ReadUnaligned`) — không cần khối `unsafe` nhưng **vẫn** có thể phá type safety. Không “an toàn hơn” chỉ vì thiếu từ khóa.

**Vì sao / Khi nào dùng unsafe:** P/Invoke, mmap, SIMD thủ công, serializer đã đo. Luôn `fixed` ngắn, kiểm length, không lưu `T*` sau khi unpin.

### 9.1 Memory safety (C# 15 preview)

> **PREVIEW (.NET 11 / C# 15).** Opt-in: SDK 11 + `<LangVersion>preview</LangVersion>`. Surface/enforcement có thể đổi trước GA (kể cả .NET 12).  
> Mục tiêu dài hạn: `unsafe` = *thao tác truy cập nhớ CLR không quản* + hợp đồng *requires-unsafe* lan ra caller — không phải “file này có con trỏ”.

**Baseline 14:** khai báo `T*`, `&x`, `fixed`, `sizeof`, dereference — đều trong `unsafe`.

**Preview 15 — pointer relaxations** (khi compile `preview`): các thao tác sau **không** cần `unsafe` context:

- Khai báo kiểu pointer và lấy địa chỉ `&`
- Câu `fixed` (pin)
- Đổi `stackalloc` sang pointer
- `sizeof` trên unmanaged type

**Vẫn `unsafe`:** dereference / truy cập nhớ:

- `*p`
- `p->member`
- `p[i]`
- index fixed-buffer
- gọi function pointer

```csharp
int number = 42;
int* pointer = &number;          // C# 15 preview: không cần unsafe

int[] numbers = [10, 20, 30];
fixed (int* first = numbers)     // preview: fixed ngoài unsafe
{
    unsafe
    {
        Console.WriteLine(*first); // dereference: vẫn unsafe
        Console.WriteLine(first[1]);
    }
}
```

**Compat / requires-unsafe (đang hoàn thiện):** bản đầy đủ (sau này) — `unsafe` trên member = caller phải ở `unsafe` hoặc cũng đánh dấu; assembly opt-in `MemorySafetyRulesAttribute`; từ khóa `safe` cho `extern`/explicit-layout. Preview 5–7: **nới pointer** đã có; **enforcement caller** chưa đủ — đừng dựa vào để audit production.

Opt-in thực nghiệm (có thể đổi tên feature flag):

```xml
<LangVersion>preview</LangVersion>
<!-- một số SDK preview: -->
<Features>$(Features);updated-memory-safety-rules</Features>
```

Learn: [What's new in C# 15 — Memory safety](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15) · [blog](https://devblogs.microsoft.com/dotnet/explore-csharp-15/).

**Vì sao / Khi nào thử preview:** interop/`sizeof` giảm ceremony. **Không** dùng trên .NET 10 LTS production. Dereference vẫn là điểm review.

---

## 10. `ArrayPool<T>` & `MemoryMarshal`

### 10.1 `ArrayPool<T>`

```csharp
var pool = ArrayPool<byte>.Shared;
byte[] rented = pool.Rent(minimumLength: 4096);
try
{
    Span<byte> use = rented.AsSpan(0, 4096); // Rent có thể trả mảng LỚN HƠN
}
finally
{
    pool.Return(rented, clearArray: true); // clear nếu dữ liệu nhạy cảm
}
```

- Luôn theo dõi `length` logic riêng. Quên `Return` → áp lực GC. Sau `Return` **không** dùng lại.  
- `IMemoryOwner<T>` / `MemoryPool<T>`: ownership rõ hơn cho `Memory<T>`.

**Pitfall:** `Rent(4096)` có thể 8192 — đừng `rented.Length` làm payload size. Double-return / return sai mảng = corrupt pool.

### 10.2 `MemoryMarshal` (tóm tắt)

```csharp
Span<byte> raw = stackalloc byte[16];
Span<int> asInts = MemoryMarshal.Cast<byte, int>(raw);
ref byte first = ref MemoryMarshal.GetReference(raw);
ReadOnlySpan<byte> utf16 = MemoryMarshal.AsBytes("abcd".AsSpan());
```

- Mạnh cho serializer/hash/interop — dễ sai alignment/lifetime.  
- `CollectionsMarshal.AsSpan(List<T>)`: vô hiệu nếu list reallocate sau đó.

**Vì sao / Khi nào dùng Marshal:** reinterpret bytes. Không dùng để ghi đè `string` (phá intern/immutability).

---

## 11. Pitfalls thường gặp

### 11.1 `ref struct` không lên heap

```csharp
Span<int> span = stackalloc int[2];
object box = span;          // lỗi
List<Span<int>> list = [];  // lỗi
```

### 11.2 Async & iterators

State machine `async`/`yield` có thể box locals → cấm `Span`/`ref struct` sống qua `await`/`yield return`.

```csharp
async Task BadAsync(Memory<byte> mem)
{
    Span<byte> span = mem.Span;
    await Task.Yield();
    span[0] = 1; // không hợp lệ
}

async Task GoodAsync(Memory<byte> mem)
{
    Process(mem.Span);     // xong trước await
    await Task.Yield();
    Process(mem.Span);     // Span mới sau await
}
```

C# 13+ cho phép `ref struct` trong async method **nếu không** vượt qua `await` trong cùng block.

### 11.3 Lifetime `stackalloc`

```csharp
Span<int> Leak()
{
    Span<int> s = stackalloc int[8];
    return s; // lỗi — stack frame chết
}
```

### 11.4 Ownership & string bất biến

- `Span`/`Memory` là *view* — mảng bị pool-return / native free khi vẫn đang dùng → corrupt.  
- Đừng `MemoryMarshal` rồi **ghi** vào span lấy từ `string` (phá bất biến).  
- `Span` không phải key collection thông thường — hash/equal theo nội dung dùng API/`string` phù hợp.

### 11.5 C# 14 overload bất ngờ

Thêm overload `ReadOnlySpan<char>` cạnh `string` có thể đổi call-site im lặng — regression test.

---

## 12. Cheat sheet chọn API

| Nhu cầu | Chọn |
|---|---|
| Lát cắt đồng bộ, zero-alloc | `Span<T>` / `ReadOnlySpan<T>` |
| Field / qua `await` | `Memory<T>` / `ReadOnlyMemory<T>` |
| Buffer tạm nhỏ, nóng | `stackalloc` → `Span` |
| Buffer tạm lớn / động | `ArrayPool<T>` |
| API thư viện (text/binary) | `ReadOnlySpan` / `ReadOnlyMemory` |
| Interop C / layout cố định | `unsafe` + `fixed` / `SafeHandle` |
| Đổ khuôn bytes ↔ struct | `MemoryMarshal.Cast` / `AsBytes` |
| Pointer khai báo (C# 15 preview) | ngoài `unsafe`; `*p` vẫn `unsafe` |

```csharp
static int ParseCsvLine(ReadOnlySpan<char> line)
{
    int count = 0;
    while (!line.IsEmpty)
    {
        int comma = line.IndexOf(',');
        ReadOnlySpan<char> cell = comma < 0 ? line : line[..comma];
        if (int.TryParse(cell, out _)) count++;
        if (comma < 0) break;
        line = line[(comma + 1)..];
    }
    return count;
}

int n = ParseCsvLine("1,2,3,4"); // C# 14: string → span
```

```csharp
static async Task<int> ChecksumAsync(ReadOnlyMemory<byte> data)
{
    int sum = 0;
    for (int offset = 0; offset < data.Length; offset += 4096)
    {
        var slice = data.Slice(offset, Math.Min(4096, data.Length - offset));
        foreach (var b in slice.Span) sum += b;
        await Task.Yield(); // không giữ Span qua await
    }
    return sum;
}
```

**Tóm lại:** `Span` = *view stack-bound*; `Memory` = *token lưu được*; `scoped` = *không escape*; `unsafe` = van an toàn — chỉ mở khi thực sự cần (C# 15 preview: mở van đúng chỗ dereference).
