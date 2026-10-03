# Collections & Generics

> **Baseline:** .NET **10** / C# **14**. Collection expression arguments (`with(...)`) là **C# 15** (mặc định trên `net11.0` từ RC1) — không có trên .NET 10.

---

## Mục lục

- [Collections \& Generics](#collections--generics)
  - [Mục lục](#mục-lục)
  - [1. Các interface cốt lõi của Collections](#1-các-interface-cốt-lõi-của-collections)
  - [2. Nhóm Collections thường dùng (mutable)](#2-nhóm-collections-thường-dùng-mutable)
    - [2.1 `List<T>`](#21-listt)
    - [2.2 `LinkedList<T>`](#22-linkedlistt)
    - [2.3 `Queue<T>`](#23-queuet)
    - [2.4 `Stack<T>`](#24-stackt)
    - [2.5 `Dictionary<TKey,TValue>`](#25-dictionarytkeytvalue)
    - [2.6 `SortedDictionary<TKey,TValue>` vs `SortedList<TKey,TValue>`](#26-sorteddictionarytkeytvalue-vs-sortedlisttkeytvalue)
    - [2.7 `HashSet<T>` / `SortedSet<T>`](#27-hashsett--sortedsett)
  - [3. Collections bất biến (`System.Collections.Immutable`)](#3-collections-bất-biến-systemcollectionsimmutable)
  - [4. `List` vs Frozen vs Concurrent](#4-list-vs-frozen-vs-concurrent)
    - [4.1 Collections đồng thời (thread-safe)](#41-collections-đồng-thời-thread-safe)
    - [4.2 Frozen (`System.Collections.Frozen`)](#42-frozen-systemcollectionsfrozen)
    - [4.3 Bảng so sánh](#43-bảng-so-sánh)
  - [5. Readonly \& View: `ReadOnlyCollection<T>`, `IReadOnlyList<T>`…](#5-readonly--view-readonlycollectiont-ireadonlylistt)
  - [6. Mảng \& các tiện ích hiệu năng: `Array`, `ArrayPool<T>`, `Span<T>`, `Memory<T>`](#6-mảng--các-tiện-ích-hiệu-năng-array-arraypoolt-spant-memoryt)
  - [7. Collection expressions (C# 12+) \& args (C# 15)](#7-collection-expressions-c-12--args-c-15)
  - [8. So sánh \& băm: `IEquatable<T>`, `IComparable<T>`, `IEqualityComparer<T>`…](#8-so-sánh--băm-iequatablet-icomparablet-iequalitycomparert)
  - [9. Hiệu năng \& best practices khi dùng collections](#9-hiệu-năng--best-practices-khi-dùng-collections)
  - [10. Generics nâng cao](#10-generics-nâng-cao)
    - [10.1 Ràng buộc (`where`) \& mẫu thiết kế](#101-ràng-buộc-where--mẫu-thiết-kế)
    - [10.2 Phương sai (variance): `out`/`in` — PECS](#102-phương-sai-variance-outin--pecs)
    - [10.3 Generic math \& `static abstract` members](#103-generic-math--static-abstract-members)
    - [10.4 Comparer/Equality custom cho collections](#104-comparerequality-custom-cho-collections)
  - [11. Cheat sheet chọn cấu trúc dữ liệu](#11-cheat-sheet-chọn-cấu-trúc-dữ-liệu)
    - [Ví dụ tổng hợp](#ví-dụ-tổng-hợp)

---

## 1. Các interface cốt lõi của Collections

```
IEnumerable<T>
└─ IEnumerator<T> (GetEnumerator())
ICollection<T> : IEnumerable<T> (Count, Add/Remove/Contains, CopyTo)
└─ IList<T> (indexer, Insert/RemoveAt)          // list dạng mảng
└─ ISet<T>  (hợp, giao, hiệu)                   // tập hợp
IDictionary<TKey,TValue> (Add, TryGetValue, Keys, Values)
IReadOnlyCollection<T> / IReadOnlyList<T> / IReadOnlyDictionary<TKey,TValue>
```

**Nguyên tắc API**:  

- **Expose tối thiểu** cần thiết (ví dụ trả `IReadOnlyList<T>` thay vì `List<T>`).  
- **Duyệt**: mọi collection nên hỗ trợ `foreach` qua `IEnumerable<T>`.  
- **Try-pattern**: `bool TryGetValue(TKey key, out TValue value)` để tránh ném exception trong luồng thường.

**Semantics:** `IEnumerable<T>` chỉ cam kết *duyệt được* — không `Count` O(1), không random access. `IReadOnlyList<T>` thêm indexer + `Count`. `ICollection<T>` cho phép mutate (`Add`) — **đừng** trả `ICollection<T>` nếu caller không được sửa.

**Vì sao / Khi nào dùng interface mỏng:** public API thư viện. Nội bộ implementation có thể giữ `List<T>` để `Add`/`EnsureCapacity`.

**Pitfall:** `IEnumerable<T>` có thể là iterator deferred (LINQ) — duyệt 2 lần chạy lại query. Materialize (`ToList`) khi cần snapshot.

---

## 2. Nhóm Collections thường dùng (mutable)

### 2.1 `List<T>`

- Mảng động, tiếp giáp bộ nhớ → **O(1)** truy cập ngẫu nhiên, **Append amortized O(1)**.
- **Insert/Remove ở giữa**: O(n) do dồn phần tử.
- API đáng chú ý: `Capacity`, `EnsureCapacity`, `AddRange`, `InsertRange`, `RemoveAll`, `BinarySearch`, `Sort`, `AsReadOnly`.

```csharp
var list = new List<int>(capacity: 1024);
list.AddRange(new[] {1,2,3});
list.Sort(); // O(n log n)
int idx = list.BinarySearch(2); // yêu cầu list đã Sort
```

**Mẹo**: biết trước kích thước? → set `Capacity` để tránh **realloc** nhiều lần.

**Semantics:** khi `Count == Capacity`, `Add` cấp phát mảng mới (~×2) và copy — amortized O(1) nhưng spike GC. `list[i]` không bound-check-elide như `Span` trong mọi JIT, nhưng rất gần mảng.

```csharp
var xs = new List<int> { 1, 2, 3 };
foreach (ref var n in CollectionsMarshal.AsSpan(xs))
    n++; // sửa tại chỗ — vô hiệu nếu Add làm realloc sau đó
```

**So sánh:** mặc định cho dãy mutable 1 thread. Không thread-safe. Frozen/Immutable khi chia sẻ đọc; `ConcurrentBag` không thay `List`.

**Vì sao / Khi nào dùng:** 90% “danh sách”. Tránh `List<object>` + boxing; tránh `Insert(0, …)` lặp lại (dùng `LinkedList`/`Stack`/`Queue` tùy chiều).

---

### 2.2 `LinkedList<T>`

- Danh sách liên kết đôi. **Insert/Remove O(1)** khi đã có node. **Tìm kiếm O(n)**.
- Không tiếp giáp bộ nhớ → cache kém; hiếm khi nhanh hơn `List<T>` trừ khi **add/remove nội bộ cực nhiều** và đã có node.

```csharp
var ll = new LinkedList<int>();
var n2 = ll.AddLast(2);
ll.AddBefore(n2, 1); // O(1)
```

**Pitfall:** `foreach` + xóa node đang duyệt cần `LinkedListNode`. Đừng chọn `LinkedList` “vì O(1) insert” nếu bạn vẫn `Find` O(n) mỗi lần.

**Vì sao / Khi nào dùng:** LRU thủ công, queue có xóa giữa. Benchmark trước — `List` thường thắng nhờ cache.

---

### 2.3 `Queue<T>`

- Hàng đợi FIFO. `Enqueue`/`Dequeue` amortized **O(1)**.  
- Ưu tiên theo key: `PriorityQueue<TElement,TPriority>` (.NET 6+) — không thay `Queue<T>` cho FIFO thuần.

```csharp
var q = new Queue<string>();
q.Enqueue("a");
var x = q.Dequeue(); // "a"
```

**Pitfall:** `Dequeue` trên rỗng → `InvalidOperationException`; dùng `TryDequeue`. Không thread-safe — producer/consumer → `Channel`/`ConcurrentQueue`.

**Vì sao / Khi nào dùng:** BFS, buffer tuần tự 1 thread.

---

### 2.4 `Stack<T>`

- Ngăn xếp LIFO. `Push`/`Pop` amortized **O(1)**.

```csharp
var st = new Stack<int>();
st.Push(10);
int top = st.Pop();
```

**Vì sao / Khi nào dùng:** DFS, undo, parse ngoặc. `TryPop` thay `Pop` trên biên rỗng.

---

### 2.5 `Dictionary<TKey,TValue>`

- Bảng băm. **Lookup trung bình O(1)**.  
- **Khóa** cần equality/hashing tốt (`IEquatable<T>`, `GetHashCode`).  
- Tránh `dict.ContainsKey(k)` rồi `dict[k]`: dùng `TryGetValue` **một lần tra**.

```csharp
var dict = new Dictionary<string,int>(StringComparer.Ordinal);
if (dict.TryGetValue("key", out var value))
{
    // dùng value
}
```

**Mẹo**: chọn `StringComparer.Ordinal`/`OrdinalIgnoreCase` thay vì mặc định để rõ ràng *culture*.

**Semantics:** hash % bucket; collision → chain/contiguous. Worst-case O(n) nếu hash xấu hoặc tấn công hash (string comparer ordinal giảm rủi ro culture). `Key` **không** được đổi field ảnh hưởng hash sau khi đưa vào dictionary.

```csharp
dict["k"] = dict.TryGetValue("k", out var c) ? c + 1 : 1; // 2 lần hash nếu không cẩn
dict["k"] = 1;
dict.TryAdd("k", 2);                 // false, không ghi đè
```

**Pitfall:** `dict[missing]` ném `KeyNotFoundException`. Key bị sửa **những field tham gia Equals/GetHashCode** có thể không tìm lại được. `List<int>` dùng equality theo identity mặc định nên sửa phần tử không đổi hash; comparer theo nội dung lại có nguy cơ này. Đọc song song với ghi cần khóa hoặc ConcurrentDictionary.

**Vì sao / Khi nào dùng:** lookup 1 thread / “ghi rồi đọc” trên cùng thread. Nhiều thread ghi → Concurrent hoặc lock. Data cố định sau startup → Frozen.

---

### 2.6 `SortedDictionary<TKey,TValue>` vs `SortedList<TKey,TValue>`

- `SortedDictionary` dùng **cây đỏ-đen** (lookup **O(log n)**, thêm/xóa **O(log n)** ổn định).  
- `SortedList` dùng **mảng đã sort** (lookup **O(log n)**, thêm/xóa **O(n)** do dịch phần tử) nhưng **tiêu tốn ít bộ nhớ** hơn & truy cập theo **index**.

Chọn gì?

- Nhiều **insert/remove rải rác** → `SortedDictionary`.  
- Ít thay đổi, cần **truy cập theo index** hoặc bộ nhớ chặt → `SortedList`.

**Vì sao không dùng mặc định:** `Dictionary` nhanh hơn nếu không cần thứ tự key. Sorted chỉ khi duyệt theo thứ tự / range.

---

### 2.7 `HashSet<T>` / `SortedSet<T>`

- `HashSet<T>`: tập hợp không trùng; các phép **Union/Intersect/Except**.

```csharp
var a = new HashSet<int>{1,2,3};
var b = new HashSet<int>{3,4};
a.IntersectWith(b); // a = {3}
```

- `SortedSet<T>`: sắp xếp tự nhiên theo `IComparer<T>`; hỗ trợ **range view** (`GetViewBetween`).

**Pitfall:** `IntersectWith` **mutate** `a`. Cần tập mới → copy trước hoặc LINQ `Intersect` (cấp phát). Equality phần tử = comparer của set, không phải `==` của bạn trừ khi khớp.

**Vì sao / Khi nào dùng:** membership, loại trùng. `FrozenSet` khi tập cố định, tra cứu cực nhiều.

---

## 3. Collections bất biến (`System.Collections.Immutable`)

> **Baseline net10.0:** `System.Collections.Immutable` có sẵn trong shared framework/reference pack, không cần thêm NuGet chỉ để dùng namespace. Target cũ hoặc cần phiên bản thư viện khác thì kiểm tra package tương ứng.

- `ImmutableList<T>`, `ImmutableDictionary<TKey,TValue>`, `ImmutableHashSet<T>`…  
- **Mọi thao tác sinh cấu trúc mới**; bên trong dùng **persistent data structure** để chia sẻ nút → tiết kiệm bộ nhớ so với copy thô.
- Phù hợp: **đồng thời, chia sẻ giữa thread**, **state lịch sử** (time-travel), **functional style**.

```csharp
using System.Collections.Immutable;
var list = ImmutableList<int>.Empty;
var list2 = list.Add(1).Add(2); // list vẫn rỗng
```

**Builder**: `var b = list.ToBuilder(); ...; var newList = b.ToImmutable();` — tối ưu nhiều thao tác.

**So sánh với Frozen:** Immutable *cập nhật rẻ* (chia sẻ cấu trúc) nhưng lookup thường **chậm hơn** `Dictionary`/`FrozenDictionary`. Frozen *xây đắt, đọc rẻ, không chỉnh từng phần tử*.

**Vì sao / Khi nào dùng Immutable:** snapshot, undo, message giữa thread không lock. Cache đọc-nhiều sau init → Frozen. View không mutate API → `IReadOnly*` (gốc vẫn mutable).

---

## 4. `List` vs Frozen vs Concurrent

Ba họ giải **ba bài toán khác nhau** — không thay thế nhau.

| Nhu cầu | Chọn |
|---|---|
| Mutable, 1 thread (hoặc lock ngoài) | `List<T>` / `Dictionary<,>` |
| Nhiều thread **ghi** xen kẽ | `Concurrent*` hoặc `lock` |
| Xây **một lần**, đọc rất nhiều, không sửa | `FrozenDictionary` / `FrozenSet` |
| Sửa tạo phiên bản mới, chia sẻ an toàn | `Immutable*` |

### 4.1 Collections đồng thời (thread-safe)

- `ConcurrentDictionary<TKey,TValue>`: tra cứu an toàn, API `GetOrAdd`, `AddOrUpdate`.
- `ConcurrentQueue<T>` giữ FIFO; `ConcurrentStack<T>` giữ LIFO; `ConcurrentBag<T>` không bảo đảm thứ tự. Lịch chạy thread và thứ tự hoàn thành xử lý vẫn có thể khác thứ tự lấy phần tử.
- `BlockingCollection<T>`: bọc trên concurrent collection với **bounded capacity** & blocking producers/consumers.
- `System.Threading.Channels` (có sẵn trên net10.0): channel cho producer/consumer async — tốt cho I/O pipeline.
- **`PriorityQueue<TElement,TPriority>`** (.NET 6+): min-heap, ưu tiên nhỏ nhất được lấy trước theo comparer; cùng priority không bảo đảm FIFO. Duyệt bằng `UnorderedItems`, không theo thứ tự ưu tiên. Không thread-safe.

```csharp
var cd = new ConcurrentDictionary<string,int>();
int v = cd.AddOrUpdate("k", 1, (_, old) => old + 1);
```

**Semantics `GetOrAdd`:** factory **có thể chạy thừa** (hai thread cùng miss) — factory phải **idempotent / rẻ / không side-effect độc**. Giá trị thắng là một trong các kết quả được store atomic.

```csharp
// ❌ factory có side-effect đắt / không idempotent
cd.GetOrAdd("k", _ => LoadFromDb()); // có thể Load hai lần
```

> **Lock thủ công** (`lock`) vẫn hữu ích cho thao tác phức tạp nhiều bước cần tính nguyên tử.

**Pitfall:** `ConcurrentDictionary` **không** làm `if (!d.ContainsKey) d[k]=…` atomic — đúng API là `TryAdd`/`GetOrAdd`. Enumerate vừa sửa: snapshot yếu, không freeze. `ConcurrentBag` không FIFO.

**Vì sao / Khi nào dùng Concurrent:** cache chia sẻ có ghi. Chỉ đọc sau init → Frozen nhanh hơn và đơn giản hơn (không lock).

### 4.2 Frozen (`System.Collections.Frozen`)

**`.NET 8+`**, namespace `System.Collections.Frozen`. Xây một lần (`ToFrozenDictionary` / `FrozenDictionary.Create`), sau đó **chỉ đọc** — runtime chọn layout tối ưu theo dữ liệu (small map, string keys…).

```csharp
using System.Collections.Frozen;

var source = new Dictionary<string, int>(StringComparer.Ordinal)
{
    ["ok"] = 200,
    ["no"] = 404,
};

FrozenDictionary<string, int> codes = source.ToFrozenDictionary(StringComparer.Ordinal);
int n = codes["ok"]; // nhanh, thread-safe đọc

// .NET 10: Create từ span — tránh List tạm
FrozenDictionary<string, int> codes2 = FrozenDictionary.Create(
    StringComparer.Ordinal,
    (ReadOnlySpan<KeyValuePair<string, int>>)
    [
        new("ok", 200),
        new("no", 404),
    ]);
```

**Semantics:** không `Add` sau freeze. “Cập nhật” = xây Frozen mới từ nguồn mutable. Lookup thường thắng `Dictionary` trên tập lớn, đọc lặp; **không** luôn thắng trên tập rất nhỏ / key `int` tuần tự — **đo**.

**So sánh chi phí:** freeze tốn CPU/memory lúc init (chấp nhận được ở startup). `ImmutableDictionary.Add` từng key rẻ hơn rebuild Frozen; Frozen lookup rẻ hơn Immutable.

**Vì sao / Khi nào dùng:** bảng mã, route, MIME, config sau load, từ điển dịch. Không dùng nếu dataset đổi liên tục.

### 4.3 Bảng so sánh

| | `List`/`Dictionary` | `ConcurrentDictionary` | `FrozenDictionary` | `ImmutableDictionary` |
|---|---|---|---|---|
| Ghi | ✅ rẻ | ✅ concurrent | ❌ rebuild | ✅ persistent |
| Đọc nhiều thread không ghi | ⚠️ không an toàn nếu có ghi song song | ✅ | ✅ | ✅ |
| Lookup điển hình | baseline | + overhead sync | thường nhanh nhất (steady) | chậm hơn dict |
| Khởi tạo | rẻ | rẻ | đắt hơn | trung bình |
| API mutate tại chỗ | ✅ | ✅ | ❌ | ❌ (trả instance mới) |

```csharp
// Chọn theo vòng đời
Dictionary<string, string> building = new(StringComparer.Ordinal);
Fill(building);
FrozenDictionary<string, string> published = building.ToFrozenDictionary(StringComparer.Ordinal);
// building có thể bỏ — published phục vụ request
```

---

## 5. Readonly & View: `ReadOnlyCollection<T>`, `IReadOnlyList<T>`…

- ReadOnlyCollection<T> là **view chỉ đọc**, không phải immutable snapshot: thay đổi IList gốc vẫn phản ánh vào view.
- **Interface `IReadOnlyList<T>`/`IReadOnlyDictionary<TKey,TValue>`**: hợp đồng chỉ-đọc; trả về từ API để **giấu** implement thật.

```csharp
IReadOnlyList<int> GetIds() => _ids; // _ids là List<int>
```

**Pitfall lớn:** trả `_ids` như `IReadOnlyList<T>` **không** ngăn caller cast lại `List<T>` và `Add`. Muốn cứng: copy, `ToFrozenSet`, hoặc `ImmutableList`. `AsReadOnly()` vẫn là view — gốc đổi, view đổi.

**Vì sao / Khi nào dùng view:** encapsulation nội bộ, caller tin cậy. Biên assembly không tin → copy/Frozen/Immutable.

---

## 6. Mảng & các tiện ích hiệu năng: `Array`, `ArrayPool<T>`, `Span<T>`, `Memory<T>`

- **`Array`**: thao tác khối: `Array.Copy`, `Clear`, `BinarySearch`, `Sort`.  
- **`ArrayPool<T>`**: *rent/return* mảng để **giảm GC** trong luồng nóng.
```csharp
var pool = System.Buffers.ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096);
try
{
    // dùng buffer
}
finally
{
    pool.Return(buffer, clearArray: true);
}
```

- **`Span<T>` / `ReadOnlySpan<T>`** (byref-like): lát cắt không cấp phát; dùng với string (as `ReadOnlySpan<char>`), file I/O, parsing…  
- **`Memory<T>` / `ReadOnlyMemory<T>`**: tương tự `Span` nhưng **lưu trữ được** (dùng cho async, field).

> Chi tiết sâu (lifetime, `stackalloc`, unsafe, pitfalls): [memory-spans.md](memory-spans.md).

**CollectionsMarshal** (nâng cao): `CollectionsMarshal.AsSpan(list)` để truy cập nội bộ `List<T>` *không an toàn phiên bản* → chỉ dùng khi hiểu rõ ràng buộc.

**Vì sao / Khi nào dùng pool:** buffer > vài trăm byte, hot path. Quên `Return` = leak pool; dùng sau `Return` = corrupt.

---

## 7. Collection expressions (C# 12+) & args (C# 15)

**C# 12** — cú pháp `[...]` tạo collection theo *target type* (thay `new List<int> { ... }` / `new[] { ... }` trong nhiều chỗ):

```csharp
int[] arr = [1, 2, 3];
List<string> names = ["a", "b"];
Span<int> slice = [1, 2, 3];
int[] merged = [..arr, 4, 5]; // spread
```

- Compiler chọn constructor / `CollectionBuilder` / empty phù hợp với kiểu đích.  
- Hỗ trợ spread `..` để nối sequence.  
- Ưu tiên khi khởi tạo ngắn; vẫn dùng `new List<T>(capacity)` khi cần capacity tường minh (hoặc `with(capacity:…)` ở C# 15).

**Semantics:** `[...]` **không** có kiểu riêng — kiểu đến từ đích (`List<int> x = [1]` khác `int[] y = [1]`). `Span<int> s = [1,2,3]` có thể `stackalloc`/inline — **không** sống lâu hơn method. Spread `..xs` enumerates `xs` lúc tạo.

```csharp
IEnumerable<int> xs = [1, 2, 3]; // thường thành mảng rồi wrap
HashSet<int> set = [1, 1, 2];    // 2 phần tử — HashSet loại trùng
```

**Pitfall:** `var x = [1, 2, 3]` không biên dịch (CS9176): collection expression cần target type, ví dụ `int[] x = [1, 2, 3]`. Overload nhận List và array có thể gây ambiguity; dùng target type tường minh khi cần. Span chứa stack/local storage không được escape; một số ReadOnlySpan từ literal hằng có thể dùng static storage và trả về hợp lệ.

**Vì sao / Khi nào dùng:** khởi tạo ngắn, test, merge `[..a, ..b]`. Hot-path biết capacity → `new List<T>(n)` hoặc C# 15 `with(capacity:…)`.

### Collection expression arguments — **C# 15**

> **C# 15 / .NET 11**, mặc định trên `net11.0` từ RC1. Không cần `LangVersion=preview`. Không compile trên C# 14.

Truyền đối số constructor/factory qua phần tử `with(...)` **đứng đầu** collection expression:

```csharp
string[] values = ["one", "two", "three"];

List<string> names = [with(capacity: values.Length * 2), ..values];

HashSet<string> set = [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO", "hello"];
// OrdinalIgnoreCase → thường còn 1 phần tử
```

**Ràng buộc chính:**

- `with(...)` phải là phần tử **đầu tiên**.  
- Không dùng cho **array** / **span** targets.  
- Arg không được `dynamic`.  
- Với `[CollectionBuilder]`, args truyền vào factory **trước** `ReadOnlySpan<T>` phần tử.

**Vì sao / Khi nào dùng:** `List` cần capacity, `HashSet`/`Dictionary` cần comparer ngay lúc tạo — tránh `new HashSet(...) { ... }` dài. Trên .NET 10: `new HashSet<string>(StringComparer.OrdinalIgnoreCase) { "Hello" }` hoặc `EnsureCapacity`.

---

## 8. So sánh & băm: `IEquatable<T>`, `IComparable<T>`, `IEqualityComparer<T>`…

- **`IEquatable<T>`**: so sánh bằng nhau theo **giá trị** (tối ưu tránh boxing).  
- **`IComparable<T>`**: thứ tự sắp xếp.  
- **`IEqualityComparer<T>` / `IComparer<T>`**: truyền vào `Dictionary`/`HashSet`/`SortedSet` để **định nghĩa quy tắc**.

```csharp
public sealed class PersonIdComparer : IEqualityComparer<Person>
{
    public bool Equals(Person? x, Person? y) => x?.Id == y?.Id;
    public int GetHashCode(Person obj) => obj.Id.GetHashCode();
}

// Dùng
var set = new HashSet<Person>(new PersonIdComparer());
```

**`GetHashCode` chuẩn**: dùng `HashCode.Combine(a,b,...)`; đảm bảo: bằng nhau ⇒ hash bằng nhau.

**Tuples/records**: đã có equality/hashing theo giá trị; tận dụng cho key phức tạp: `Dictionary<(int,int),TValue>`.

**Pitfall:** Equals và GetHashCode phải nhất quán, ổn định suốt thời gian key nằm trong collection. `StringComparer.CurrentCulture` chụp culture lúc tạo comparer, không tự đổi theo thread về sau. Key kỹ thuật thường nên dùng Ordinal/OrdinalIgnoreCase; comparer tự đọc CurrentCulture mỗi lần có thể phá tính ổn định. [Equality](oop.md#4-equality--tostring).

**Vì sao / Khi nào dùng comparer ngoài type:** cùng `Person` lúc thì so Id, lúc thì so Email — không nhúng một `Equals` duy nhất.

---

## 9. Hiệu năng & best practices khi dùng collections

- **Chọn đúng cấu trúc** (xem *Cheat sheet*).  
- **Capacity**: biết trước kích thước? set `Capacity`/`EnsureCapacity` (`List<T>`, `Dictionary<,>`).  
- **Tránh cấp phát**: dùng `ArrayPool<T>`, `Span<T>`, tránh tạo iterator/closure trong hot-path.  
- **`foreach`** trên `List<T>` dùng **struct enumerator** (không cấp phát). Trên `IEnumerable<T>` “trừu tượng” có thể boxing; trong hot-path cân nhắc `for`/`Span`.  
- **`Dictionary`**: dùng `TryGetValue` thay vì `ContainsKey` + indexer; khai báo `StringComparer` phù hợp.  
- **`SortedList`** vs `SortedDictionary`**: xem mục 2.6.  
- **LINQ**: rõ ràng, ngắn; nhưng dễ **cấp phát**. Trong đường nóng → cân nhắc vòng `for`/`foreach`.  
- **Immutable**: dùng khi chia sẻ giữa thread nhiều đọc; biến đổi nhiều → dùng `Builder`.  
- **Concurrent**: thao tác đơn giản → concurrent collections; thao tác phức tạp nhiều bước → *lock*.
- **Frozen**: init một lần, đọc hot.

---

## 10. Generics nâng cao

### 10.1 Ràng buộc (`where`) & mẫu thiết kế

```csharp
public T Create<T>() where T : new() => new T();

public T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

public interface IRepository<T> where T : class
{
    T? FindById(Guid id);
    void Add(T entity);
}
```

- Ràng buộc đặc biệt: `unmanaged`, `notnull`, `struct`, `class`, `new()`, **`allows ref struct`** (C# 13 — generic nhận `Span<T>` / `ref struct`).  
- **Ràng buộc nhiều**: `where T : SomeBase, ISvc, new()`.

**Vì sao ràng buộc:** JIT có thể gọi interface trên struct **không box** khi `where T : IComparable<T>`. Không ràng buộc + `IComparable` → box. Chi tiết anti-constraint: [typesystem.md §13.2](typesystem.md#132-allows-ref-struct-c-13).

### 10.2 Phương sai (variance): `out`/`in` — PECS

- **Covariant `out`**: cho **output-only**. Ví dụ `IEnumerable<out T>` → `IEnumerable<string>` nạp vào nơi cần `IEnumerable<object>`.

Variance conversion chỉ áp dụng cho reference type: `IEnumerable<int>` không chuyển thành `IEnumerable<object>`; `Cast<object>()` tạo pipeline có boxing.
- **Contravariant `in`**: cho **input-only**. Ví dụ `IComparer<in T>` có thể so sánh `object` cho `string`.

```csharp
IEnumerable<string> ss = new List<string>();
IEnumerable<object> oo = ss; // ok nhờ 'out T'
```

> Không áp dụng cho class/struct generic, chỉ **interface/delegate**.

**PECS** (*Producer Extends, Consumer Super* — thuật ngữ Java; C# dùng `out`/`in`):

| Vai trò | Java PECS | C# | Ý nghĩa |
|---|---|---|---|
| Producer (chỉ đọc T ra) | `? extends T` | `IFoo<out T>` | `Foo<Derived>` dùng như `Foo<Base>` |
| Consumer (chỉ ghi T vào) | `? super T` | `IFoo<in T>` | `Foo<Base>` dùng như `Foo<Derived>` |
| Cả hai (List) | invariant | `List<T>` invariant | không gán chéo |

```csharp
void PrintAll(IEnumerable<object> items)
{
    foreach (var x in items) Console.WriteLine(x);
}

IEnumerable<string> names = ["a"];
PrintAll(names); // producer: string extends object

void SortNames(List<string> names, IComparer<string> cmp) => names.Sort(cmp);

IComparer<object> byToString = Comparer<object>.Create(
    (a, b) => string.CompareOrdinal(a?.ToString(), b?.ToString()));
SortNames(["b", "a"], byToString); // consumer: comparer<object> super string
```

**Vì sao `List<T>` invariant:** nếu `List<string>` là `List<object>`, `list.Add(new object())` phá mảng string — cùng lỗ hổng [array covariance](typesystem.md#9-mảng-arrays-1d-nhiều-chiều-jagged-spant).

**Pitfall:** khai báo `out T` rồi có method `void Add(T)` → lỗi compile. Delegate `Func<out T>` covariant; `Action<in T>` contravariant.

**Vì sao / Khi nào dùng:** API chỉ duyệt → `IEnumerable<out T>` / `IReadOnlyList<out T>`. API chỉ so sánh/ghi → `IComparer<in T>` / `Action<in T>`. Storage hai chiều → invariant `List<T>`.

### 10.3 Generic math & `static abstract` members

Từ .NET 7/C# 11: interface có **`static abstract`** cho toán học tổng quát (`System.Numerics`):

```csharp
using System.Numerics;

T Sum<T>(IEnumerable<T> xs) where T : INumber<T>
{
    T s = T.Zero;
    foreach (var x in xs) s += x;
    return s;
}
```

- Viết thuật toán số học generic **không cần** overloading thủ công từng kiểu.

**Semantics:** `INumber<T>` yêu cầu `T` tự làm toán tử `+`, `T.Zero`, parse… JIT chuyên biệt hóa theo `T` (`int` vs `double`) — không phải dispatch ảo cổ điển trên instance.

```csharp
static T Clamp<T>(T value, T min, T max)
    where T : IComparable<T>
    => value.CompareTo(min) < 0 ? min
     : value.CompareTo(max) > 0 ? max
     : value;

static T Average<T>(ReadOnlySpan<T> xs)
    where T : INumber<T>
{
    if (xs.IsEmpty) return T.Zero;
    T sum = T.Zero;
    foreach (var x in xs) sum += x;
    return sum / T.CreateChecked(xs.Length);
}

_ = Sum([1, 2, 3]);                 // int
_ = Sum([1.5, 2.5]);                // double
_ = Average<decimal>([1.0m, 2.0m]);
```

**Họ interface hay dùng:** `INumber<T>`, `IBinaryInteger<T>`, `IFloatingPoint<T>`, `IAdditionOperators<T,T,T>`, `IMinMaxValue<T>`.

**Pitfall:** `T.CreateChecked` ném khi overflow; `CreateTruncating`/`CreateSaturating` khác semantics. Không giả định `INumber<T>` = “không NaN” (`double`). Mixing `INumber<T>` với `IEnumerable` box enumerator nếu không concrete.

**So sánh:** trước .NET 7 phải `Add(int)`, `Add(double)`, … hoặc `dynamic` (chậm, không an toàn). Generic math = một thuật toán, nhiều kiểu. Static **non-virtual** trên interface (helper gọi `I.M()`, không dispatch theo `T`) là việc khác — C# 15 không còn đòi runtime DIM: [oop.md §3.3](oop.md#33-static-trên-interface-non-virtual-vs-static-abstract).

**Vì sao / Khi nào dùng:** thư viện số, thống kê, shader-like. Business money → `decimal` tường minh thường rõ hơn generic.

### 10.4 Comparer/Equality custom cho collections

- `Dictionary<TKey,TValue>(IEqualityComparer<TKey>)`  
- `SortedSet<T>(IComparer<T>)`

```csharp
var dictCI = new Dictionary<string,int>(StringComparer.OrdinalIgnoreCase);
var setDesc = new SortedSet<int>(Comparer<int>.Create((a,b) => b.CompareTo(a)));
```

**Pitfall:** đổi comparer sau khi đã có data = không hỗ trợ. Frozen phải truyền comparer **lúc** `ToFrozenDictionary(comparer)`.

---

## 11. Cheat sheet chọn cấu trúc dữ liệu

| Bài toán | Gợi ý |
|---|---|
| Truy cập theo chỉ số, thêm cuối nhiều | `List<T>` (+ đặt `Capacity`) |
| Thêm/xóa nhiều ở giữa (đã có node) | `LinkedList<T>` |
| FIFO / LIFO | `Queue<T>` / `Stack<T>` |
| Tra cứu theo khóa | `Dictionary<TKey,TValue>` (+ `TryGetValue`, comparer phù hợp) |
| Tập hợp không trùng | `HashSet<T>` |
| Cần thứ tự sort & tra cứu | `SortedDictionary<TKey,TValue>` |
| Sort ổn định, ít cập nhật, cần index | `SortedList<TKey,TValue>` |
| Chia sẻ thread-safe, nhiều đọc, **không ghi** sau init | `FrozenDictionary` / `FrozenSet` |
| Chia sẻ thread-safe, **có ghi** | `Concurrent*` / `lock` |
| Snapshot / undo / persistent | `Immutable*` collections |
| Producer/consumer tốc độ cao | `System.Threading.Channels` / `BlockingCollection<T>` |
| Giảm GC với buffer | `ArrayPool<T>`, `Span<T>`/`Memory<T>` |

---

### Ví dụ tổng hợp

```csharp
// Đếm số lần xuất hiện (case-insensitive) và xuất top N theo tần suất giảm dần
IReadOnlyList<(string Word, int Count)> TopNWords(IEnumerable<string> words, int n)
{
    var freq = new Dictionary<string,int>(StringComparer.OrdinalIgnoreCase);
    foreach (var w in words)
        freq[w] = (freq.TryGetValue(w, out var c) ? c : 0) + 1;

    // sort theo Count giảm dần, rồi theo Word tăng dần
    var list = freq.ToList();
    list.Sort((a,b) => b.Value.CompareTo(a.Value) != 0
        ? b.Value.CompareTo(a.Value)
        : StringComparer.OrdinalIgnoreCase.Compare(a.Key, b.Key));

    if (n < list.Count) list.RemoveRange(n, list.Count - n);
    return list.Select(p => (p.Key, p.Value)).ToList();
}
```

Lookup mã HTTP cố định — Frozen:

```csharp
static readonly FrozenDictionary<int, string> Status =
    new Dictionary<int, string>
    {
        [200] = "OK",
        [404] = "Not Found",
        [500] = "Error",
    }.ToFrozenDictionary();

static string Label(int code)
    => Status.TryGetValue(code, out var s) ? s : "Unknown";
```

---

**Kết luận**: Nắm vững **interface cốt lõi**, chọn đúng **cấu trúc dữ liệu**, hiểu **equality/hashing**, và sử dụng **generics nâng cao** (ràng buộc, variance/PECS, generic math) sẽ giúp code C# của bạn **đúng, nhanh, và dễ bảo trì**.
