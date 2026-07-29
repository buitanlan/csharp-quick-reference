# Memory, Span & unsafe

Tham chiếu nâng cao về **bộ nhớ managed**, `ref`/`Span`/`Memory`, và **unsafe** trên baseline **.NET 10 / C# 14**.  
Tập trung semantics, lifetime, pitfalls (tương tự chương pointers bên Go) — không phải tutorial GC đầy đủ.

> **Baseline:** .NET **10** / C# **14** (first-class span conversions). Nhiều API (`Span`, `Memory`, `scoped`) từ C# 7.2–11.

---

## Mục lục

- [Memory, Span \& unsafe](#memory-span--unsafe)
  - [Mục lục](#mục-lục)
  - [1. Stack vs Heap \& GC (generational)](#1-stack-vs-heap--gc-generational)
  - [2. `ref` locals, `ref` returns, `ref` fields](#2-ref-locals-ref-returns-ref-fields)
  - [3. `Span<T>` \& `ReadOnlySpan<T>`](#3-spant--readonlyspant)
  - [4. `Memory<T>` \& `ReadOnlyMemory<T>`](#4-memoryt--readonlymemoryt)
  - [5. `stackalloc` \& fixed buffers](#5-stackalloc--fixed-buffers)
  - [6. C\# 14 — implicit Span conversions](#6-c-14--implicit-span-conversions)
  - [7. `scoped` (C\# 11) — lifetime](#7-scoped-c-11--lifetime)
  - [8. `unsafe` \& pointers — overview](#8-unsafe--pointers--overview)
  - [9. `ArrayPool<T>` \& `MemoryMarshal`](#9-arraypoolt--memorymarshal)
  - [10. Pitfalls thường gặp](#10-pitfalls-thường-gặp)
  - [11. Cheat sheet chọn API](#11-cheat-sheet-chọn-api)

---

## 1. Stack vs Heap & GC (generational)

### 1.1 Stack

- Mỗi thread có **stack**: frame method, biến local, return address, đôi khi value type local.  
- Cấp phát/hủy LIFO theo scope — **không** qua GC.  
- Giới hạn kích thước → StackOverflow nếu đệ quy sâu / `stackalloc` quá lớn.

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

**Hot path thực dụng:** giảm allocation → `Span`, `stackalloc`, `ArrayPool`, `ValueTask`; tránh LINQ/`string` tạm trong vòng nóng.

---

## 2. `ref` locals, `ref` returns, `ref` fields

### 2.1 `ref` local & `ref` return — **C# 7+**

Tham chiếu tới ô nhớ **đã tồn tại** — không copy value lớn.

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

- `ref` field **chỉ** trong `ref struct`. Kết hợp `scoped` để siết lifetime (mục 7).

### 2.3 `ref struct` (byref-like)

```csharp
public ref struct Utf8Parser
{
    private ReadOnlySpan<byte> _input;
    public Utf8Parser(ReadOnlySpan<byte> input) => _input = input;
}
```

**Không** được: boxing lên `object`/interface thông thường; field của `class`; capture lambda; sống qua `async`/`yield` như biến treo qua suspension.  
(`allows ref struct` generic — C# 13+ — chỉ trong ngữ cảnh hạn chế.)

---

## 3. `Span<T>` & `ReadOnlySpan<T>`

### 3.1 Bản chất

`Span<T>` / `ReadOnlySpan<T>` là **`ref struct`**: descriptor `(ref T, length)` tới nhớ liên tục — **không sở hữu**, không cấp phát thêm.

Có thể trỏ: `T[]`, `stackalloc`, native/pinned, `string` (`ReadOnlySpan<char>`), buffer unmanaged qua API phù hợp.

```csharp
int[] data = { 10, 20, 30, 40 };
Span<int> all = data;              // C# 14 implicit (mục 6)
Span<int> mid = data.AsSpan(1, 2); // {20,30}
mid[0] = 99;                       // data[1] == 99

ReadOnlySpan<char> name = "Alice"; // C# 14
if (name.StartsWith("Al"))
    Console.WriteLine(name[2..]);
```

### 3.2 API & vì sao nhanh

- Indexer, `Length`, `Clear`/`Fill`, `CopyTo`/`TryCopyTo`, `Slice` **O(1)**.  
- Parsing: `int.TryParse(ReadOnlySpan<char>, …)` — tránh `Substring`.  
- Zero-alloc cho lát cắt; JIT hay tối ưu biên kiểm trong vòng quen thuộc.

```csharp
static bool TryReadInt(ReadOnlySpan<char> text, out int value)
    => int.TryParse(text, out value);
```

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

---

## 5. `stackalloc` & fixed buffers

### 5.1 `stackalloc`

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

### 5.2 Fixed buffer & `InlineArray`

```csharp
unsafe struct Packet
{
    public fixed byte Header[8]; // cần unsafe — interop
    public int Length;
}

[System.Runtime.CompilerServices.InlineArray(8)] // C# 12 — ưu tiên khi được
struct Header8 { private byte _element0; }
```

---

## 6. C# 14 — implicit Span conversions

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

> Nâng C# 14 có thể đổi overload resolution chỗ có nhiều overload `T[]`/`Span`/`ReadOnlySpan` — chạy test kỹ.

---

## 7. `scoped` (C# 11) — lifetime

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

> Thư viện low-level: nếu compiler báo lifetime, đừng bỏ `ref struct`/`scoped` chỉ để “cho compile”.

---

## 8. `unsafe` & pointers — overview

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

### Khi **KHÔNG** dùng unsafe

- Nghiệp vụ thường, web/CRUD.  
- “Tối ưu” khi chưa đo profiler.  
- Khi `Span`/`Memory`/`MemoryMarshal`/`Unsafe` (managed helpers) đủ.  
- Team không quen review memory safety.

> **Mặc định hiện đại:** `Span` / `ref struct` / `stackalloc` safe. `unsafe` cho interop / layout / buffer cực đoan sau khi đo.

---

## 9. `ArrayPool<T>` & `MemoryMarshal`

### 9.1 `ArrayPool<T>`

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

### 9.2 `MemoryMarshal` (tóm tắt)

```csharp
Span<byte> raw = stackalloc byte[16];
Span<int> asInts = MemoryMarshal.Cast<byte, int>(raw);
ref byte first = ref MemoryMarshal.GetReference(raw);
ReadOnlySpan<byte> utf16 = MemoryMarshal.AsBytes("abcd".AsSpan());
```

- Mạnh cho serializer/hash/interop — dễ sai alignment/lifetime.  
- `CollectionsMarshal.AsSpan(List<T>)`: vô hiệu nếu list reallocate sau đó.

---

## 10. Pitfalls thường gặp

### 10.1 `ref struct` không lên heap

```csharp
Span<int> span = stackalloc int[2];
object box = span;          // lỗi
List<Span<int>> list = [];  // lỗi
```

### 10.2 Async & iterators

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

### 10.3 Lifetime `stackalloc`

```csharp
Span<int> Leak()
{
    Span<int> s = stackalloc int[8];
    return s; // lỗi — stack frame chết
}
```

### 10.4 Ownership & string bất biến

- `Span`/`Memory` là *view* — mảng bị pool-return / native free khi vẫn đang dùng → corrupt.  
- Đừng `MemoryMarshal` rồi **ghi** vào span lấy từ `string` (phá bất biến).  
- `Span` không phải key collection thông thường — hash/equal theo nội dung dùng API/`string` phù hợp.

---

## 11. Cheat sheet chọn API

| Nhu cầu | Chọn |
|---|---|
| Lát cắt đồng bộ, zero-alloc | `Span<T>` / `ReadOnlySpan<T>` |
| Field / qua `await` | `Memory<T>` / `ReadOnlyMemory<T>` |
| Buffer tạm nhỏ, nóng | `stackalloc` → `Span` |
| Buffer tạm lớn / động | `ArrayPool<T>` |
| API thư viện (text/binary) | `ReadOnlySpan` / `ReadOnlyMemory` |
| Interop C / layout cố định | `unsafe` + `fixed` / `SafeHandle` |
| Đổ khuôn bytes ↔ struct | `MemoryMarshal.Cast` / `AsBytes` |

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

**Tóm lại:** `Span` = *view stack-bound*; `Memory` = *token lưu được*; `unsafe` = van an toàn — chỉ mở khi thực sự cần.
