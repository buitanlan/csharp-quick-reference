# LINQ (Language Integrated Query)

> **Baseline:** .NET **10** / C# **14**. BCL/LINQ micro-opts khi nâng TFM — hot path vẫn đo BenchmarkDotNet.  
> Extension members (C# 14) **không** thay Standard Query Operators: LINQ vẫn là extension method trên `IEnumerable<T>` / `IQueryable<T>`. Toán tử custom: §9.

**LINQ** đem cú pháp truy vấn vào C#, thống nhất cách làm việc với **tập hợp đối tượng**, **XML**, **CSDL**, **JSON**, **in-memory** và cả **stream async** (thông qua `IAsyncEnumerable<T>` + gói mở rộng). Cốt lõi không phải “SQL trong C#” mà là **toán tử trên chuỗi** (`Where`/`Select`/…) + **hai mô hình thực thi**: in-memory (`IEnumerable`) vs dịch biểu thức (`IQueryable`).

---

## Mục lục

- [LINQ (Language Integrated Query)](#linq-language-integrated-query)
  - [Mục lục](#mục-lục)
  - [1. Tư duy LINQ \& hai cú pháp](#1-tư-duy-linq--hai-cú-pháp)
  - [2. Deferred vs Immediate execution](#2-deferred-vs-immediate-execution)
  - [3. LINQ to Objects vs IQueryable (EF/LINQ Providers)](#3-linq-to-objects-vs-iqueryable-eflinq-providers)
  - [4. Nhóm toán tử chuẩn (Standard Query Operators)](#4-nhóm-toán-tử-chuẩn-standard-query-operators)
    - [4.1 Filtering: `Where`, `OfType`](#41-filtering-where-oftype)
    - [4.2 Projection: `Select`, `SelectMany`](#42-projection-select-selectmany)
    - [4.3 Sorting: `OrderBy`, `ThenBy`, `Reverse`](#43-sorting-orderby-thenby-reverse)
    - [4.4 Grouping: `GroupBy`, `ToLookup`](#44-grouping-groupby-tolookup)
    - [4.5 Joining: `Join`, `GroupJoin`, Left Join](#45-joining-join-groupjoin-left-join)
    - [4.6 Set: `Distinct`, `Union`, `Intersect`, `Except`](#46-set-distinct-union-intersect-except)
    - [4.7 Quantifiers: `Any`, `All`, `Contains`](#47-quantifiers-any-all-contains)
    - [4.8 Element: `First`, `Single`, `Last`, `ElementAt`](#48-element-first-single-last-elementat)
    - [4.9 Partitioning: `Skip`, `Take`, `SkipWhile`, `TakeWhile`](#49-partitioning-skip-take-skipwhile-takewhile)
    - [4.10 Aggregation: `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`](#410-aggregation-count-sum-min-max-average-aggregate)
    - [4.11 Generation/Conversion: `Range`, `Repeat`, `Empty`, `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`…](#411-generationconversion-range-repeat-empty-tolist-toarray-todictionary-tohashset)
    - [4.12 `Zip`, `Chunk`, `Append/Prepend`, `SequenceEqual`, `DefaultIfEmpty`](#412-zip-chunk-appendprepend-sequenceequal-defaultifempty)
  - [5. Query syntax ↔ method syntax (bảng quy chiếu)](#5-query-syntax--method-syntax-bảng-quy-chiếu)
  - [6. IQueryable \& biểu thức (Expression)](#6-iqueryable--biểu-thức-expression)
  - [7. Async LINQ \& Streams](#7-async-linq--streams)
  - [8. PLINQ (Parallel LINQ)](#8-plinq-parallel-linq)
  - [9. Custom LINQ operators (viết toán tử riêng với extension methods)](#9-custom-linq-operators-viết-toán-tử-riêng-với-extension-methods)
  - [10. Best practices \& Pitfalls](#10-best-practices--pitfalls)
  - [11. Cheat sheet nhanh](#11-cheat-sheet-nhanh)

---

## 1. Tư duy LINQ & hai cú pháp

LINQ có **hai cú pháp tương đương**:

- **Method syntax** (khuyên dùng): chuỗi extension methods trên `IEnumerable<T>`/`IQueryable<T>`.
- **Query syntax**: tựa SQL; compile-time dịch sang method syntax.

```csharp
// Method syntax
var q1 = people.Where(p => p.Age >= 18)
               .OrderBy(p => p.LastName)
               .Select(p => new { p.LastName, p.FirstName });

// Query syntax (dịch tương đương)
var q2 = from p in people
         where p.Age >= 18
         orderby p.LastName
         select new { p.LastName, p.FirstName };
```

Chọn cú pháp nào?  
- **Method syntax** nhất quán, đầy đủ toán tử (`DistinctBy`, `Chunk`, `MaxBy`… không có keyword).  
- **Query syntax** dễ đọc với `from/where/select/group/join` nhiều range; `from` thứ hai = `SelectMany` (§4.2).

Cả hai đều **không chạy** cho đến khi enumerate nếu chuỗi chỉ gồm toán tử deferred (§2). `var` ở đây là query object, không phải `List`.

---

## 2. Deferred vs Immediate execution

Đây là chỗ LINQ **dễ đúng lúc debug, sai lúc production**.

- **Deferred execution**: `Where`, `Select`, `SelectMany`, `OrderBy`, `GroupBy`, `Join`, `Skip`/`Take` (LINQ to Objects)… chỉ **xây dựng pipeline** (iterator / expression tree). Chưa đọc nguồn cho đến khi **enumerate**: `foreach`, `ToList`, `Count()`, `Any()`, `First`, …  
- **Immediate execution**: `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`, `Count`/`Sum`/`Average` (không predicate trên IQueryable vẫn hit DB), `First`/`Single`/`ElementAt`, `All`/`Any`/`Contains` — **kích hoạt** thực thi ngay.

```csharp
var q = numbers.Where(x => x % 2 == 0); // chưa chạy

// chạy tại đây (mỗi lần duyệt lại chạy lại):
foreach (var n in q) Console.WriteLine(n);

var list = q.ToList(); // chạy và materialize một lần
```

### 2.1 Multiple enumeration

Gán `var q = source.Where(...)` rồi gọi `q.Count()` **và** `foreach (q)` → nguồn bị đọc **hai lần**. Với mảng thì “chỉ chậm”; với `IEnumerable` là I/O, `yield` generator, hoặc EF `IQueryable` → **hai round-trip** / đọc stream hai lần (lần hai có thể throw).

```csharp
IEnumerable<string> lines = ReadLines(path).Where(l => l.Length > 0);
var n = lines.Count();          // đọc hết file
foreach (var l in lines) { }    // đọc lại — hoặc fail nếu stream đã dispose
```

Chốt kết quả: `ToList()` / `ToArray()` khi cần snapshot, khi nguồn có side-effect, hoặc khi truyền query ra ngoài method (lifetime DbContext).

### 2.2 Capture & “query chạy lúc nào”

Deferred **đóng** lambda + biến captured. Đổi biến **sau** khi tạo query, **trước** khi enumerate → predicate thấy giá trị **lúc chạy**, không lúc tạo:

```csharp
int min = 10;
var q = nums.Where(n => n > min);
min = 100;
var list = q.ToList(); // lọc theo 100, không phải 10
```

Cùng cơ chế closure với vòng `for` (§10). EF: giá trị parameter được đóng vào expression lúc **translate/execute**, không lúc gõ `Where`.

### 2.3 `GroupBy` deferred vs `ToLookup` immediate

`GroupBy` trên `IEnumerable` **deferred** — chưa nhóm cho đến khi iterate (và khi iterate, LINQ to Objects thường **buffer toàn bộ** nhóm: deferred ≠ streaming từng phần tử xong là quên). `ToLookup` **chạy ngay**, cho indexer `lookup[key]`.

### 2.4 IQueryable

Deferred ở đây nghĩa là **chưa gửi SQL**. `ToListAsync()` / `CountAsync()` mới hit DB. Log SQL (`ToQueryString()`, EF logging) trước khi tối ưu — đừng đoán.

### 2.5 Iterator: “deferred” vẫn tốn state

`Where`/`Select` trên Objects là iterator: mỗi lần `GetEnumerator()` tạo state machine mới. Pipeline dài (`Where.Select.Where.Select`) = nhiều object nhỏ — thường rẻ hơn I/O, nhưng hot-path micro-benchmark có thể thua `for`. `ICollection<T>.Count` đi tắt; `IEnumerable` custom không `Count` → `Count()` **duyệt hết** (immediate + đắt).

`OrderBy` deferred nhưng lần enumerate đầu **buffer + sort toàn bộ** — không phải streaming. `GroupBy` tương tự (§2.3, §4.4). Chỉ `Where`/`Select`/`Take` (Objects) gần với kéo từng phần tử.

---

## 3. LINQ to Objects vs IQueryable (EF/LINQ Providers)

- **LINQ to Objects**: chạy trên `IEnumerable<T>` (in-memory). Lambda là **delegate** (`Func<T,bool>`) — mọi method C# gọi được.  
- **`IQueryable<T>`**: biểu diễn truy vấn **có thể dịch** sang hệ đích (SQL, OData…). Lambda là **expression tree** (`Expression<Func<…>>`).
  - **EF Core**: chỉ dịch được **tập con** toán tử/method; nếu không dịch được → (tuỳ version/config) ném runtime exception, hoặc **client eval** (kéo cột về memory rồi lọc — dễ N+1 / kéo cả bảng).
  - Tránh gọi **method tuỳ ý** trong predicate/select vì **không thể dịch sang SQL**.

**Quy tắc vàng**: Với EF, giữ toàn bộ truy vấn **trên server** trước khi materialize (`ToListAsync`). `AsNoTracking()` nếu chỉ đọc. `Select` DTO sớm để không kéo navigation thừa.

Nhận diện: `db.Users.Where(...)` là `IQueryable`; `db.Users.AsEnumerable().Where(...)` / `ToList()` giữa chừng là **LINQ to Objects** từ đó trở đi — SQL đã “đóng” (thường `SELECT *` / đến đoạn materialize).

### 3.1 Bảng “có dịch được không” (tinh thần EF Core)

| Trong lambda | Thường |
|--------------|--------|
| So sánh property, `&&` `\|\|`, `Contains` trên string, `StartsWith` | SQL |
| `list.Contains(e.Id)` local list nhỏ | `IN` |
| `Regex`, `File.*`, custom static helper, local function | **Không** / client |
| `DateTime.Now` | Tùy version — hay dùng UTC server / `EF.Functions` |
| `ToString()` format, interpolation phức tạp | Không ổn định |
| Navigation `.Any(x => …)` | `EXISTS` |
| `GroupBy` + `Sum`/`Count` | `GROUP BY` nếu hình dạng đúng |
| `Select` DTO/`record` cùng assembly | Thường OK |
| Gọi method instance trên entity (`user.IsAdmin()`) | **Không** trừ mapped |

Khi nghi: `query.ToQueryString()` (EF) hoặc log. “Chạy được” ≠ “chạy trên server”.

---

## 4. Nhóm toán tử chuẩn (Standard Query Operators)

Nhóm dưới là **method** trên `Enumerable` / `Queryable`. Query syntax chỉ phủ một phần (bảng §5) — không có keyword cho `LeftJoin`, `CountBy`, `Chunk`, `DistinctBy`.

Bổ sung sau .NET Framework, có trên baseline **net10.0** trừ dòng .NET 11:

| Đời | Toán tử | Nhóm |
|---|---|---|
| .NET 6 | `Chunk`, `DistinctBy`, `ExceptBy`, `IntersectBy`, `UnionBy`, `MinBy`, `MaxBy`, `TryGetNonEnumeratedCount`, `Take`/`ElementAt` với `Range`/`Index`, `Zip` 3 dãy | §4.6–4.12 |
| .NET 7 | `Order`, `OrderDescending` | §4.3 |
| .NET 9 | `CountBy`, `AggregateBy`, `Index` | §4.2, §4.4, §4.10 |
| .NET 10 | `LeftJoin`, `RightJoin` (bắt buộc result selector) | §4.5 |
| .NET 11 | `FullJoin`; `Join`/`GroupJoin`/`LeftJoin`/`RightJoin` trả tuple, thêm comparer | §4.5 — **không** có trên net10.0 |

### 4.1 Filtering: `Where`, `OfType`

```csharp
var adults = people.Where(p => p.Age >= 18);
var stringsOnly = objects.OfType<string>(); // bỏ phần tử không đúng kiểu
```

`OfType<T>` vừa filter vừa cast (bỏ không khớp). `Cast<T>` throw nếu sai kiểu. EF: `OfType<T>` hữu ích TPH/TPT; `Where(x => x is T)` có thể không dịch.

### 4.2 Projection: `Select`, `SelectMany`

`Select` **1→1**: mỗi phần tử nguồn một kết quả (cùng cardinality).  
`SelectMany` **1→N rồi flatten**: mỗi phần tử nguồn một **chuỗi** con, nối thành một chuỗi phẳng (cross join khi hai `from`).

```csharp
var names = people.Select(p => $"{p.FirstName} {p.LastName}");

var allTags = posts.SelectMany(p => p.Tags); // flatten IEnumerable<IEnumerable<T>>
// cross join:
var pairs = from x in xs
            from y in ys
            select (x, y);
// ≡ xs.SelectMany(x => ys, (x, y) => (x, y));
```

Nhầm lẫn thường gặp:

| Viết | Kết quả |
|------|---------|
| `posts.Select(p => p.Tags)` | `IEnumerable<List<string>>` (lồng) |
| `posts.SelectMany(p => p.Tags)` | `IEnumerable<string>` (phẳng) |
| `SelectMany` + collection `null` | `NullReferenceException` lúc enumerate — dùng `p.Tags ?? []` |
| Query syntax `from a in xs from b in a.Children` | `SelectMany`, không phải `Select` |

`SelectMany` overload `(x, y) => result` giữ **cả** phần tử ngoài và trong (như `from`/`select` có đủ range). EF: `SelectMany` navigation → `JOIN`/`CROSS APPLY`; filter trước `SelectMany` nếu không muốn explode rồi mới `Where`.

Projection anonymous type / tuple: LINQ to Objects ổn; EF: `Select` DTO/`record` để materialize — đừng `Select` entity rồi loop lazy navigation (N+1).

`SelectMany((x, i) => …)` có index — Objects; EF ít dịch. Flatten JSON/`List<List<T>>`: luôn `SelectMany`, không `Select` rồi `foreach` lồng nếu đã ở pipeline.

**.NET 9+ `Index()`** — cùng ý `Select((x, i) => (i, x))`, trả `(int Index, T Item)` mà không cần selector:

```csharp
foreach (var (i, person) in people.Index())
    Console.WriteLine($"{i}: {person.Name}");
```

Deferred. Index bắt đầu 0 theo thứ tự enumerate, không phải chỉ số gốc nếu phía trước đã `Where`. EF: `Index()` thường không dịch — đánh số sau `ToList` hoặc `ROW_NUMBER` trên SQL.

---

### 4.3 Sorting: `OrderBy`, `ThenBy`, `Reverse`

```csharp
var ordered = people.OrderBy(p => p.LastName)
                    .ThenBy(p => p.FirstName);

var desc = people.OrderByDescending(p => p.Age);
```

**`OrderBy` ổn định** (LINQ to Objects): phần tử bằng nhau giữ thứ tự ban đầu. Gọi `OrderBy` lần hai **thay** sort, không phải then — dùng `ThenBy`.  
`IQueryable`: sort trên server; collation SQL ≠ `StringComparer.Ordinal` — đừng giả định culture.  
**.NET 7+:** `Order` / `OrderDescending` khi phần tử tự so sánh được (`IComparable<T>`), không cần key:

```csharp
var asc = nums.Order();
var desc = names.OrderDescending();
```

### 4.4 Grouping: `GroupBy`, `ToLookup`

```csharp
var groups = people.GroupBy(p => p.City); // IEnumerable<IGrouping<string, Person>>

ILookup<string, Person> byCity = people.ToLookup(p => p.City);
var inHanoi = byCity["Hanoi"]; // lookup O(1) theo key; key thiếu → rỗng, không throw
```

**`GroupBy` (Objects):** deferred; lần enumerate đầu **đọc hết nguồn**, nhét vào lookup nội bộ, rồi yield từng `IGrouping`. Không phải streaming từng group khi nguồn vô hạn. Key `null` được phép (một nhóm). Equality key: anonymous type / record / tuple — `GroupBy(x => x.Name.ToLower())` tạo string mới mỗi phần tử (ổn) nhưng comparer mặc định ordinal culture-sensitive với `string` — cân nhắc `GroupBy(x => x.Name, StringComparer.OrdinalIgnoreCase)`.

Overload `GroupBy(key, element, resultSelector)` gom ngay, tránh giữ entity gốc:

```csharp
var counts = orders.GroupBy(o => o.CustomerId, (id, g) => new { id, n = g.Count() });
```

**EF `GroupBy`:** phải dịch được sang `GROUP BY`. `GroupBy` rồi `Select(g => g.Sum(...))` thường OK; `GroupBy` rồi materialize `IGrouping` entity đầy đủ có thể **không dịch** hoặc SQL nặng. Filter `Where` **trước** `GroupBy` (SQL `WHERE`) vs `Where` trên group sau ( `HAVING` / client). `AsEnumerable()` trước `GroupBy` = nhóm in-memory sau khi kéo rows.

**`ToLookup`:** immediate, indexer, cho phép duplicate key (khác `ToDictionary` throw). Dùng khi cần tra nhiều lần theo key; `GroupBy` khi compose thêm LINQ (deferred).

Query syntax: `group x by k into g` — `into` tiếp tục query trên các nhóm (`g.Key`, `g.Count()`).

Nhóm rỗng: `GroupBy` không yield nhóm không có phần tử (khác SQL `GROUP BY` trên bảng dim). Muốn mọi key kể cả 0 count: left join tập key × `GroupBy` / `ToLookup` + duyệt key phía ngoài.

**.NET 9+ `CountBy` / `AggregateBy`:** đếm hoặc gộp **theo key trong một lần**, không cấp phát `IGrouping` + list phần tử. Đúng khi chỉ cần số / tổng, không cần các phần tử trong nhóm. Deferred đến lúc enumerate, rồi **đọc hết** nguồn (cùng họ buffer với `GroupBy`).

```csharp
// KeyValuePair<dept, count>
foreach (var (dept, n) in employees.CountBy(e => e.Department))
    Console.WriteLine($"{dept}: {n}");

// seed chung cho mọi key
var totals = orders.AggregateBy(
    o => o.CustomerId,
    seed: 0m,
    (sum, o) => sum + o.Amount);

// seed theo key (nhóm mới bắt đầu từ giá trị riêng)
var totals2 = orders.AggregateBy(
    o => o.CustomerId,
    seedSelector: id => 0m,
    (sum, o) => sum + o.Amount);
```

Comparer key tùy chọn (tham số cuối). `CountBy` không có nhóm count 0. EF: `GroupBy` + `Count`/`Sum` vẫn là đường SQL; `CountBy`/`AggregateBy` trên `IQueryable` kiểm tra bản EF trước khi giả định dịch.

`IGrouping<TKey,T>` implement `IEnumerable<T>` — `g.Where`/`g.Select` là Objects trên nhóm **đã materialize** (sau khi query chạy). EF: đừng `GroupBy(e => e).Select(g => g.First())` kiểu “lấy entity đầy đủ mỗi nhóm” nếu SQL không dịch — dùng `Select` cột + key, hoặc window SQL thô.

---

### 4.5 Joining: `Join`, `GroupJoin`, Left Join

```csharp
// Inner join
var q = from c in customers
        join o in orders on c.Id equals o.CustomerId
        select new { c.Name, o.Id };

// Group join (customers + collection orders)
var q2 = from c in customers
         join o in orders on c.Id equals o.CustomerId into g
         select new { c, Orders = g };

// Left join = group join + DefaultIfEmpty
var left = from c in customers
           join o in orders on c.Id equals o.CustomerId into g
           from o in g.DefaultIfEmpty()
           select new { c, o }; // o có thể null
```

**Composite key**: dùng **anonymous type** hoặc **tuple** (cùng tên/thứ tự):

```csharp
join o in orders on new { c.Id, c.Region } equals new { Id = o.CustomerId, o.Region }
```

`equals` **không** đối xứng như SQL `ON` tùy ý — trái = outer, phải = inner. Method syntax: `Join` / `GroupJoin`. Nested loop `SelectMany` + `Where` = inner join không tối ưu bằng `Join` (hash) trên Objects.

**.NET 10 — `LeftJoin` / `RightJoin`.** Cùng hash join, không phải `GroupJoin` + `DefaultIfEmpty`. Không có keyword query syntax. Phần không khớp là `default`: class → `null`; struct/tuple → giá trị default (biến **không** null — đừng `?.` lên chính struct).

```csharp
var left = customers.LeftJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, o) => new { c, o }); // o null nếu customer không có order (order là class)

var right = customers.RightJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, o) => new { c, o }); // mọi order; c default nếu không có customer
```

`RightJoin` giữ **mọi** phần tử dãy thứ hai (`orders`). Một outer nhiều inner → nhiều dòng, giống `Join`, không gói thành nhóm (`GroupJoin` mới gói).

EF Core 10 dịch `LeftJoin`/`RightJoin` thành `LEFT`/`RIGHT JOIN` khi selector và key dịch được. Query syntax left join (§ trên) vẫn đúng và là cách duy nhất trên net8/net9.

**.NET 11** (không có trên net10.0): `FullJoin` (cả hai phía, bên thiếu = `default`). `Join`, `GroupJoin`, `LeftJoin`, `RightJoin`, `FullJoin` thêm overload **không** result selector — trả `(TOuter, TInner)` hoặc nhóm — và `IEqualityComparer` tùy chọn. Có trên `Enumerable`, `Queryable`, `AsyncEnumerable`.

```csharp
// SDK 11 / net11.0 — tuple, không selector
foreach (var (product, category) in products.LeftJoin(
    categories, p => p.Category, c => c.Name))
{
    _ = category; // default khi product không khớp category
}

foreach (var (product, category) in products.FullJoin(
    categories, p => p.Category, c => c.Name))
{
    // product default nếu category không có product, và ngược lại
}
```

### 4.6 Set: `Distinct`, `Union`, `Intersect`, `Except`

```csharp
var unique = items.Distinct(comparer); // truyền comparer nếu cần
var union = a.Union(b);
var inter = a.Intersect(b);
var diff  = a.Except(b);
```

Mặc định: `EqualityComparer<T>.Default` (reference cho class không override). Entity EF: `Distinct` trên DTO/`Select` cột, không phải instance tracker.

**.NET 6+ theo key** — đừng tự viết `GroupBy` chỉ để lấy phần tử đầu:

```csharp
var unique = users.DistinctBy(u => u.Email, StringComparer.OrdinalIgnoreCase);

// ExceptBy / IntersectBy: dãy thứ HAI là key, không phải phần tử
var fresh = all.ExceptBy(existingIds, x => x.Id);
var kept = all.IntersectBy(wantedIds, x => x.Id);

// UnionBy: dãy thứ hai vẫn là phần tử; trùng key thì giữ phần tử của dãy đầu
var merged = a.UnionBy(b, x => x.Id);
```

`ExceptBy(existing, x => x.Id)` khi `existing` là `IEnumerable<User>` **không** compile — tham số đó là `IEnumerable<TKey>`. `UnionBy` thì ngược lại. EF dịch `DistinctBy` tùy version — xem SQL; `Except` entity nguyên con thường không phải ý bạn muốn.

### 4.7 Quantifiers: `Any`, `All`, `Contains`

```csharp
bool anyAdult = people.Any(p => p.Age >= 18);
bool allAdult = people.All(p => p.Age >= 18);
bool has42 = numbers.Contains(42);
```

`Any()` không predicate: “có phần tử?” — rẻ hơn `Count() > 0` (Objects dừng sớm; EF `EXISTS`). `All` trên rỗng = **true**. `Contains` trên `IQueryable` với local collection → SQL `IN` (giới hạn số phần tử SQL).

### 4.8 Element: `First`, `Single`, `Last`, `ElementAt`

```csharp
var first = numbers.First();               // throw nếu rỗng
var firstOr = numbers.FirstOrDefault();    // default(T) nếu rỗng

var only = numbers.Single(n => n == 5);    // throw nếu != 1 phần tử phù hợp
var onlyOr = numbers.SingleOrDefault();
```

`First` vs `Single`: `Single` phải **đúng một** (EF `TOP 2`). API “get by id”: `Single`/`SingleOrDefault` nếu id unique; list/filter: `FirstOrDefault`. `Last` trên `IQueryable` cần `OrderBy` — không thì SQL không xác định. `default(T)` với `int` = 0: đừng nhầm “không có” với giá trị 0 — dùng `FirstOrDefault` + nullable / `bool` pattern.

**.NET 6+ `MinBy` / `MaxBy`:** trả **phần tử**, không trả key. Rỗng thì ném, giống `Min`/`Max`. Hòa key: lấy phần tử gặp **trước**.

```csharp
Person youngest = people.MinBy(p => p.Age)!; // ném nếu people rỗng
```

### 4.9 Partitioning: `Skip`, `Take`, `SkipWhile`, `TakeWhile`

```csharp
var page = items.Skip((pageIndex-1)*pageSize).Take(pageSize);
var untilNeg = numbers.TakeWhile(n => n >= 0);
```

EF: `Skip`/`Take` cần `OrderBy` ổn định (paging). `SkipWhile`/`TakeWhile` thường **không** dịch SQL. .NET 6+: `Take(Range)` / `..` trên Objects.

### 4.10 Aggregation: `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`

```csharp
int count = items.Count();
int sum   = numbers.Sum();
var total = numbers.Aggregate(0, (acc, x) => acc + x);
```

Immediate. `Aggregate` trên EF **hiếm khi** dịch — giữ Objects. `Min`/`Max` rỗng throw; `MinBy`/`MaxBy` (net6+) trả phần tử (§4.8). `Count` predicate vs `Where`+`Count`: Objects tương đương; EF thường cùng `COUNT`/`SUM(CASE`. Đếm theo nhóm không cần danh sách phần tử: `CountBy` (§4.4), không `GroupBy` rồi `Count`.

### 4.11 Generation/Conversion: `Range`, `Repeat`, `Empty`, `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`…

```csharp
var r = Enumerable.Range(1, 5);      // 1..5
var rep = Enumerable.Repeat("A", 3); // A A A
var empty = Enumerable.Empty<int>();

var list = q.ToList();
var dict = people.ToDictionary(p => p.Id); // chú ý key trùng → throw
var set  = items.ToHashSet(StringComparer.OrdinalIgnoreCase);
```

`ToDictionary` key trùng throw; `ToLookup` thì không. `Enumerable.Empty<T>()` cached — tốt hơn `new T[0]` khi return rỗng.

**.NET 6+ `TryGetNonEnumeratedCount`:** lấy `Count` **không** duyệt khi nguồn là `ICollection<T>` / mảng / `List<T>` (và một số iterator biết trước độ dài). `false` thì chưa đếm — `Where`/`Select` thường rơi vào nhánh này:

```csharp
if (!source.TryGetNonEnumeratedCount(out int n))
    n = source.Count(); // duyệt
```

Đừng gọi `Count()` “cho chắc” trên generator / EF: `TryGet` `false` trên `IQueryable` không có nghĩa là được phép `Count()` sync. EF dùng `CountAsync`.

### 4.12 `Zip`, `Chunk`, `Append/Prepend`, `SequenceEqual`, `DefaultIfEmpty`

```csharp
var zipped = xs.Zip(ys, (x, y) => (x, y)); // ghép 2 dãy theo vị trí; dừng khi một bên hết
var triples = xs.Zip(ys, zs);              // .NET 6+: (T1, T2, T3), không selector; cũng dừng ở dãy ngắn nhất
var chunks = numbers.Chunk(100);           // IEnumerable<T[]> — khối cuối có thể ngắn hơn size
var withHead = seq.Prepend(head);
bool same = seq1.SequenceEqual(seq2);
var withDefault = seq.DefaultIfEmpty(0);   // nếu rỗng → có 1 phần tử 0
```

`DefaultIfEmpty` = mảnh left join kiểu query syntax. Trên .NET 10 ưu tiên `LeftJoin` (§4.5) khi hai dãy và một key. `Chunk` là Objects; EF paging dùng `Skip`/`Take` có `OrderBy`. `Chunk(0)` hoặc size âm thì ném.

---

## 5. Query syntax ↔ method syntax (bảng quy chiếu)

| Query syntax | Method syntax |
|---|---|
| `from x in xs select x` | `xs.Select(x => x)` |
| `from x in xs where P(x) select x` | `xs.Where(x => P(x))` |
| `from x in xs orderby x.Key select x` | `xs.OrderBy(x => x.Key)` |
| `from x in xs orderby x.A, x.B descending select x` | `xs.OrderBy(x => x.A).ThenByDescending(x => x.B)` |
| `from x in xs group x by k` | `xs.GroupBy(x => k)` |
| `from x in xs join y in ys on x.K equals y.K select ...` | `xs.Join(ys, x => x.K, y => y.K, (x,y) => ...)` |
| `from x in xs from y in ys select ...` | `xs.SelectMany(x => ys, (x,y) => ...)` |
| `let t = expr select ...` | (giới thiệu biến trung gian → lồng `Select`) |
| `into g ...` | tiếp tục với kết quả `GroupJoin`/`group` |

`let` = `Select` anonymous giữ biến — EF dịch được nếu `expr` dịch được. Query syntax **không** có `Distinct`/`Chunk`/`LeftJoin` keyword → chấm method xen: `(from … select x).Distinct()`, hoặc `customers.LeftJoin(...)` (§4.5).

---

## 6. IQueryable & biểu thức (Expression)

- `IQueryable<T>` mở rộng `IEnumerable<T>` bằng **`Expression`** + `Provider`. Cùng “hình” `Where(x => …)` nhưng overload `Where(Expression<Func<…>>)` — compiler nhét **cây**, không compile sẵn IL của predicate.  
- `AsQueryable()` trên `List<T>` cho `IQueryable` **không** có SQL — provider LINQ to Objects. Test “query EF” bằng `AsQueryable()` **không** chứng minh dịch SQL.  
- Trong EF Core:
  - Tránh phương thức **không thể dịch** (`Regex.IsMatch`, `string.Format` phức tạp, custom helpers, `DateTime.Now` vs `DateTime.UtcNow` tùy version, local function, `int.Parse`).  
  - Tránh **materialize sớm** (`ToList()` giữa chừng) nếu muốn server lọc/sort — sau `AsEnumerable()`/`ToList()` mọi `Where` chạy in-memory.  
  - Dùng `AsNoTracking()` cho chỉ-đọc; `AsSplitQuery()` khi Include lớn (cartesian explosion).  
  - `EF.Functions` / `EF.Property` / `EF.Constant` khi cần hint dịch.  
  - Client eval ẩn: `Select` gọi method C# trên entity sau khi SQL thiếu cột → exception hoặc query tệ. Bật logging SQL.  
  - Subquery `Any`/`Count` trong `Where` → `EXISTS`; navigation collection `Where` không `Include` vẫn filter — khác `Include` rồi lọc in-memory.

```csharp
// Dịch được (tinh thần)
db.Users.Where(u => u.Email.EndsWith("@a.com")).Select(u => u.Id);

// Thường vỡ / client
db.Users.Where(u => MyUtil.IsCorp(u.Email));
db.Users.Where(u => File.Exists(u.Path));
```

Composition: giữ `IQueryable` qua layer (`IQueryable<User> Query()`) dễ leak DbContext và stack filter không dịch. Ưu tiên method nhận `IQueryable` trả `IQueryable` **trong cùng unit of work**, materialize ở biên (handler).

`Expression<Func<T,bool>>` compile sang SQL từng node: member access, constant, `Enumerable.Contains` local. Closure bắt `DateTime.Now` trong Objects = lúc enumerate; trong EF = parameter **lúc execute** (thường OK) vs `DateTime.Now` **trong cây** có thể translate thành SQL `GETDATE` hoặc fail — đừng nhét clock vào expression nếu chưa kiểm SQL.

`IQueryable` implement `IEnumerable` — `foreach` trên `DbSet` **sync-over-store** (blocking). Dùng async execute APIs.

---

## 7. Async LINQ & Streams

- **Enumerable async**: dùng `IAsyncEnumerable<T>` + `await foreach`.  
- **Toán tử async cho `IAsyncEnumerable<T>`** trên net10: gói `System.Linq.Async` (`WhereAwait`, `SelectAwait`, `ToListAsync` trên stream). **EF Core** `ToListAsync` / `SingleAsync` là execute SQL, không phải gói đó.  
- **.NET 11:** `Join` / `LeftJoin` / `RightJoin` / `FullJoin` có trên `AsyncEnumerable` trong BCL. net10 chưa có — đừng gọi `LeftJoin` trên `IAsyncEnumerable` với SDK 10.

```csharp
await foreach (var line in ReadLinesAsync(path).Where(x => x.Length > 0))
    Console.WriteLine(line);

// EF Core
var users = await db.Users.Where(u => u.Active).ToListAsync();
```

Đừng `.Result` trên `ToListAsync` (deadlock sync-over-async). Cancellation: `ToListAsync(ct)`.

---

## 8. PLINQ (Parallel LINQ)

- `AsParallel()` chạy LINQ trên nhiều core. Dùng cho **tác vụ CPU-bound** thuần trên **in-memory** đã materialize, **không** I/O, **không** EF.

```csharp
var result = data.AsParallel()
                 .WithDegreeOfParallelism(Environment.ProcessorCount)
                 .AsOrdered()                 // nếu cần giữ thứ tự — trả giá merge
                 .Where(ComputeHeavy)         // thuần, không side-effects
                 .Select(Transform)
                 .ToList();
```

### 8.1 Khi **không** dùng PLINQ

- Nguồn `IQueryable` / EF: `AsParallel()` **không** song song hóa SQL; thường kéo query rồi (lỡ) parallelize client — hoặc API không hợp lệ. Parallel DB = nhiều query/`Task`, không PLINQ.  
- I/O (`HttpClient`, disk): thread pool noose; dùng `async` / `Parallel.ForEachAsync`.  
- Chuỗi ngắn / predicate rẻ: overhead partition > lợi. Đo BenchmarkDotNet.  
- Cần thứ tự ổn + `AsOrdered()` trên pipeline dài: có thể **chậm hơn** tuần tự.  
- Side-effect / `List.Add` không khóa / `Random` instance / `DateTime.Now` trong predicate → data race. `ConcurrentBag` vẫn thường kém hơn `Select` thuần rồi materialize.  
- ASP.NET request path: tranh CPU với request khác; giới hạn DOP, thường **không** PLINQ per-request.  
- `AsSequential()` khi đoạn sau không parallel-safe.

**Cảnh báo**: exception trong worker bọc `AggregateException`. `WithMergeOptions` / `WithExecutionMode(ForceParallelism)` chỉ khi đã đo. Không phải “bật là nhanh”.

`AsParallel()` trên `IEnumerable` đã là query EF materialize nhầm (`.AsEnumerable().AsParallel()`) = kéo hết bảng rồi CPU — thảm họa. Đúng: SQL filter/`Take` trước, `ToListAsync`, rồi (nếu đo được) parallel **CPU** trên list nhỏ.

`WithCancellation(token)` để abort; không cancel → thread pool kẹt lúc shutdown ASP.NET.

---

## 9. Custom LINQ operators (viết toán tử riêng với extension methods)

LINQ to Objects dựa vào **extension methods** trả `IEnumerable<T>` (iterator + `yield return`) — **deferred** nếu bạn `yield`, **immediate** nếu `ToList` bên trong (đừng làm vậy trừ khi toán tử vốn immediate).

```csharp
public static class LinqEx
{
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source)
        where T : struct
    {
        foreach (var x in source)
            if (x.HasValue) yield return x.Value;
    }

    public static IEnumerable<T> DistinctBy<T, TKey>(this IEnumerable<T> src, Func<T,TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var x in src)
            if (seen.Add(keySelector(x)))
                yield return x;
    }
}
```

Gợi ý: `DistinctBy`, `MaxBy`, `MinBy`, `Chunk`, `CountBy`, `AggregateBy`, `Index`, `LeftJoin`, `RightJoin` đã có trong BCL baseline net10.0. `FullJoin` và overload tuple là .NET 11. Viết đè tên chuẩn (`Where`, `DistinctBy`) = nightmare overload.

### 9.1 Extension members (C# 14) vs toán tử LINQ

C# **14** thêm `extension(T)` blocks (method/property/operator trên receiver) — xem [methods.md §12](methods.md#12-this-và-extension-method--extension-members) / [oop.md](oop.md). Dùng khi:

- Property/operator trên type bạn không sở hữu (`extension(string s) { public bool IsBlank => … }`).  
- Nhóm API theo receiver, không phải chuỗi query.

**Không** thay `IEnumerable` operators: query LINQ cần `this IEnumerable<T>` (hoặc `IQueryable`) + `Func`/`Expression` + `yield` để compose với `Where`/`Select`. Extension member **property** không nhận lambda selector; không tự thành `IQueryable` translator.

Muốn toán tử chạy trên EF: không đủ `yield` trên `IEnumerable` — phải `IQueryable` + `Expression` (hoặc EF.Functions). Custom `WhereX(this IQueryable<T>)` trả `query.Where(expr)` với cây dịch được. Gọi custom `IEnumerable` extension trên `IQueryable` → bind nhầm **Objects** (cast/`AsEnumerable` ẩn) → kéo cả bảng.

```csharp
// Nguy hiểm: IQueryable bind vào extension IEnumerable
public static IEnumerable<User> Active(this IEnumerable<User> s)
    => s.Where(u => u.Active); // nếu gọi db.Users.Active() có thể client-eval

public static IQueryable<User> Active(this IQueryable<User> s)
    => s.Where(u => u.Active); // SQL WHERE
```

Prefer BCL / package (`System.Linq.Async`) trước khi viết thêm.

C# 14 extension **indexer** không có (indexer extension = **C# 15** — [oop.md](oop.md)). Đừng chờ `xs[1..]` custom qua extension block thay `ElementAt` trong query EF.

---

## 10. Best practices & Pitfalls

1. **Deferred**: multiple enumeration; materialize khi snapshot / I/O / rời DbContext.  
2. **Closure**: lambda bắt biến vòng lặp / `min` đổi trước `ToList` — copy local (`var age = item.Age`) trước khi đóng query. `foreach` hiện đại mỗi iter một biến; `for` + `i` vẫn cổ điển nguy hiểm khi **defer** `queries.Add(() => list[i])`.  
3. **Null-safety & rỗng**: `First()` ném nếu rỗng; `FirstOrDefault` + `default(T)` với value type.  
4. **Distinct/Dictionary**: comparer hoặc `DistinctBy`; `ToDictionary` throw trùng key.  
5. **Chuỗi**: `StringComparer.Ordinal/OrdinalIgnoreCase` — SQL collation khác.  
6. **Hiệu năng**: LINQ rõ ràng, có delegate/iterator overhead; hot-path `for`/`Span<T>`. `Count != 0` vs `Any()`.  
7. **EF/IQueryable**: method không dịch; tránh client-eval; `Where`/`Select` trước `ToListAsync`; đừng `IEnumerable` extension trên `DbSet` nếu muốn SQL.  
8. **`Select` vs `SelectMany`**: lồng vs phẳng; `from` thứ hai = `SelectMany`.  
9. **`GroupBy`**: Objects buffer; EF `GROUP BY` hạn chế; `ToLookup` immediate.  
10. **`Single()`**: đúng 1 phần tử; nghi thì `FirstOrDefault` + kiểm tra.  
11. **PLINQ**: CPU-bound in-memory, không I/O/EF, đo trước, không side-effect.  
12. **Compose nhỏ**: `var adults = …Where; var names = adults.Select` — dễ log `ToQueryString()`.

### 10.1 Closure trong query — ví dụ `for`

```csharp
var actions = new List<Func<int>>();
for (int i = 0; i < 3; i++)
    actions.Add(() => i);          // mọi delegate thấy i == 3 sau vòng
for (int i = 0; i < 3; i++)
{
    var copy = i;
    actions.Add(() => copy);       // 0,1,2
}
```

LINQ deferred = cùng bẫy: `queries.Add(source.Where(_ => _.Id == i))` rồi đổi `i`. `foreach (var item in items)` từ C# 5 mỗi vòng một `item` — an toàn hơn `for`. EF parameter: đóng giá trị lúc **execute**; loop `foreach (var id in ids) db.Users.Where(u => u.Id == id)` = N query (hoặc dùng `Contains`).

---

## 11. Cheat sheet nhanh

```csharp
// Top N theo điểm giảm dần, cùng điểm thì theo tên tăng dần
var top = students
    .OrderByDescending(s => s.Score)
    .ThenBy(s => s.Name, StringComparer.Ordinal)
    .Take(10)
    .ToList();

// Left join — .NET 10 method. Query syntax (mọi TFM): GroupJoin + DefaultIfEmpty, xem §4.5.
var totals = customers.LeftJoin(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, o) => (c, o))
    .GroupBy(x => x.c)
    .Select(g => new
    {
        Customer = g.Key,
        Total = g.Where(x => x.o != null).Sum(x => x.o!.Amount)
    });

// Grouping tháng-năm, đếm số đơn
var monthly = orders
    .GroupBy(o => new { o.Date.Year, o.Date.Month })
    .Select(g => new { g.Key.Year, g.Key.Month, Count = g.Count() })
    .OrderBy(x => x.Year).ThenBy(x => x.Month);

// Distinct theo khoá tuỳ biến
var distinct = users.DistinctBy(u => u.Email.ToLowerInvariant());

// Chia trang
IEnumerable<T> Page<T>(IEnumerable<T> src, int page, int size)
    => src.Skip((page-1)*size).Take(size);
```

---

**Kết luận**: LINQ giúp code **khai báo, ngắn gọn, dễ đọc**, đồng thời đủ mạnh để **kết hợp** với EF/Providers, **song song** (PLINQ), và **async streams**. Nắm chắc **toán tử chuẩn**, hiểu **deferred vs immediate**, phân biệt **IEnumerable/IQueryable**, `Select`/`SelectMany`/`GroupBy`, closure lúc enumerate, **khi không** dùng PLINQ, và custom operator **không** nhầm extension members C# 14 — chìa khoá để truy vấn vừa **đúng** vừa **nhanh**.
