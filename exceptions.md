# Exception & Error Handling trong C#

> **Baseline:** .NET **10** / C# **14**.

Ngoại lệ là luồng **bất thường** — đắt (stack trace) và dễ nuốt sai trong async. Chương này nhấn **`ThrowIf*`**, filter `when`, **rethrow**, **await unwrap**, và **cancellation ≠ lỗi nghiệp vụ**.

---

## Mục lục

- [Exception \& Error Handling trong C#](#exception--error-handling-trong-c)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan \& triết lý](#1-tổng-quan--triết-lý)
  - [2. Cú pháp cơ bản: `try`/`catch`/`finally`/`throw`](#2-cú-pháp-cơ-bản-trycatchfinallythrow)
  - [3. Guard `ThrowIf*` (.NET 6+)](#3-guard-throwif-net-6)
  - [4. Hệ phân cấp ngoại lệ \& các ngoại lệ phổ biến](#4-hệ-phân-cấp-ngoại-lệ--các-ngoại-lệ-phổ-biến)
  - [5. Bắt `Exception` đúng cách](#5-bắt-exception-đúng-cách)
    - [5.1 Bắt cụ thể trước, tổng quát sau](#51-bắt-cụ-thể-trước-tổng-quát-sau)
    - [5.2 Không bắt `Exception` vô tội vạ](#52-không-bắt-exception-vô-tội-vạ)
    - [5.3 `catch` rỗng là code smell](#53-catch-rỗng-là-code-smell)
  - [6. Rethrow, bọc \& bảo toàn stack trace](#6-rethrow-bọc--bảo-toàn-stack-trace)
    - [6.1 Rethrow đúng](#61-rethrow-đúng)
    - [6.2 Bọc ngoại lệ với InnerException](#62-bọc-ngoại-lệ-với-innerexception)
    - [6.3 Bảo toàn stack trace thủ công (nâng cao)](#63-bảo-toàn-stack-trace-thủ-công-nâng-cao)
  - [7. Exception filter với `when`](#7-exception-filter-với-when)
  - [8. Quản lý tài nguyên: `finally`, `using`, `await using`](#8-quản-lý-tài-nguyên-finally-using-await-using)
    - [8.1 Sử dụng `finally` để đảm bảo thu hồi](#81-sử-dụng-finally-để-đảm-bảo-thu-hồi)
    - [8.2 `using` \& `await using` (khuyến nghị)](#82-using--await-using-khuyến-nghị)
  - [9. Thiết kế ngoại lệ tuỳ biến (Custom exceptions)](#9-thiết-kế-ngoại-lệ-tuỳ-biến-custom-exceptions)
    - [9.1 Khi nào cần?](#91-khi-nào-cần)
    - [9.2 Hướng dẫn](#92-hướng-dẫn)
  - [10. Ngoại lệ trong async/await \& song song](#10-ngoại-lệ-trong-asyncawait--song-song)
    - [10.1 `await` *unwrap* ngoại lệ](#101-await-unwrap-ngoại-lệ)
    - [10.2 `.Result` / `.Wait()` bọc trong `AggregateException`](#102-result--wait-bọc-trong-aggregateexception)
    - [10.3 Nhiều task: `Task.WhenAll`](#103-nhiều-task-taskwhenall)
    - [10.4 `async void` \& unobserved exceptions](#104-async-void--unobserved-exceptions)
    - [10.5 TPL/Parallel/PLINQ](#105-tplparallelplinq)
  - [11. Cancellation vs Exception](#11-cancellation-vs-exception)
  - [12. Global handling \& logging](#12-global-handling--logging)
    - [12.1 Ứng dụng desktop (WPF/WinForms)](#121-ứng-dụng-desktop-wpfwinforms)
    - [12.2 ASP.NET Core](#122-aspnet-core)
    - [12.3 Worker/Service](#123-workerservice)
  - [13. Hiệu năng \& quy tắc sử dụng ngoại lệ](#13-hiệu-năng--quy-tắc-sử-dụng-ngoại-lệ)
  - [14. Kiểm tra tràn số: `checked` / `unchecked`](#14-kiểm-tra-tràn-số-checked--unchecked)
  - [15. Cheat sheet \& checklist](#15-cheat-sheet--checklist)

---

## 1. Tổng quan & triết lý

- Ngoại lệ là **luồng điều khiển bất thường** dùng để báo lỗi **không mong đợi**.
- **Không dùng ngoại lệ cho luồng điều khiển bình thường** (ví dụ kiểm tra tồn tại file, parse… → ưu tiên `TryXxx`).
- Ngoại lệ **tốn chi phí** do phải xây dựng lại stack trace và thông tin Exception: ném/bắt tạo stack trace → tránh ở hot-path.
- **Cancellation** dùng cùng hệ thống exception (`OperationCanceledException`) nhưng **ý nghĩa** khác lỗi — xem §11.

---

## 2. Cú pháp cơ bản: `try`/`catch`/`finally`/`throw`

```csharp
try
{
    DoWork();
}
catch (IOException ex)
{
    Log(ex);
    // xử lý/khôi phục
}
finally
{
    Cleanup(); // luôn chạy, kể cả khi có/không có exception
}
```

- `finally` **luôn** chạy (trừ khi process bị terminate).
- `throw;` **ném lại** ngoại lệ hiện tại, giữ stack trace.

Ném ngoại lệ:

```csharp
if (arg is null)
    throw new ArgumentNullException(nameof(arg));

throw new InvalidOperationException("State is invalid");
```

Helper guard gọn hơn `if` + `throw` — **vẫn là exception path**, không thay `TryParse` — §3.

---

## 3. Guard `ThrowIf*` (.NET 6+)

API tĩnh trên `ArgumentNullException`, `ArgumentOutOfRangeException`, `ArgumentException`, `ObjectDisposedException`, … ném **đúng type** với tên tham số, giảm boilerplate.

**WHY:** một dòng, JIT có thể coi là cold path; `nameof` không lệch. **Không** rẻ hơn `throw` tay — vẫn exception. Dùng cho *contract* (null, range), không cho parse user input (`TryParse`).

```csharp
public void Register(string name, int count, string? email)
{
    ArgumentNullException.ThrowIfNull(name);
    ArgumentException.ThrowIfNullOrWhiteSpace(name); // .NET 7+
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(count);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(count, 1000);
    ObjectDisposedException.ThrowIf(_disposed, this);
}
```

| Helper (tiêu biểu) | Từ | Ném khi |
|---|---|---|
| `ArgumentNullException.ThrowIfNull(arg)` | .NET 6 | `arg` null (`[NotNull]` cho NRT) |
| `ArgumentNullException.ThrowIfNull(arg, paramName)` | .NET 6 | tùy tên |
| `ArgumentException.ThrowIfNullOrEmpty(str)` | .NET 7 | null hoặc `""` |
| `ArgumentException.ThrowIfNullOrWhiteSpace(str)` | .NET 8 | null/empty/whitespace |
| `ArgumentOutOfRangeException.ThrowIfNegative` / `ThrowIfNegativeOrZero` / `ThrowIfZero` | .NET 8 | số |
| `ThrowIfGreaterThan` / `LessThan` / `Equal` / `NotEqual` | .NET 8 | so sánh |
| `ObjectDisposedException.ThrowIf(condition, instance)` | .NET 7 | `condition` true |

```csharp
public sealed class Connection : IDisposable
{
    private bool _disposed;
    public void Send(ReadOnlySpan<byte> payload)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        ArgumentOutOfRangeException.ThrowIfZero(payload.Length);
        // ...
    }
    public void Dispose() => _disposed = true;
}
```

**Pitfalls:**

1. **`ThrowIfNull` trên `int?` / value:** `ThrowIfNull(object?)` — `int x` không null; `int?` box/`object`. Dùng `ThrowIfNegative` cho số, không `ThrowIfNull(count)` với `int`.
2. **Message tùy biến:** helper message chuẩn BCL. Cần câu nghiệp vụ → `throw new ArgumentOutOfRangeException(nameof(x), x, "must be even")`.
3. **Không** dùng `ThrowIf*` trong vòng hot *kỳ vọng fail* (parse từng dòng log) — `Try*` / `if`.
4. Expression trees / một số source-gen cần `throw new` tường minh — hiếm.
5. `ThrowIfNull(arg)` sau khi đã dùng `arg!` — thừa; đặt **đầu method**.

`Debug.Assert` **không** thay guard production (bị strip Release — [preprocessor-directives.md §5](preprocessor-directives.md#5-conditionalattribute-vs-preprocessor)).

**Chọn `if` + `throw` vs `ThrowIf*`:**

```csharp
// Contract — ThrowIf*
public Buffer(int size)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(size);
    _data = new byte[size];
}

// Nghiệp vụ / message giàu — throw tường minh
public void Withdraw(decimal amount)
{
    ArgumentOutOfRangeException.ThrowIfNegative(amount);
    if (amount > _balance)
        throw new InvalidOperationException($"Insufficient funds: {_balance} < {amount}");
}
```

`ThrowIfNull<T>(T? arg)` generic class constraint — NRT: sau lời gọi, `arg` được xem non-null (`[NotNull]`). Giúp flow analysis, không phải phép màu runtime (vẫn NRE nếu caller tắt NRT và truyền null qua reflection).

---

## 4. Hệ phân cấp ngoại lệ & các ngoại lệ phổ biến

```
System.Object
└─ System.Exception
   ├─ System.SystemException
   │  ├─ NullReferenceException
   │  ├─ IndexOutOfRangeException
   │  ├─ InvalidOperationException
   │  ├─ ArgumentException
   │  │  ├─ ArgumentNullException
   │  │  └─ ArgumentOutOfRangeException
   │  ├─ NotSupportedException
   │  ├─ FormatException
   │  ├─ OverflowException
   │  ├─ DivideByZeroException
   │  ├─ TimeoutException
   │  └─ ...
   ├─ System.IO.IOException (và các ngoại lệ con)
   ├─ System.Net.Http.HttpRequestException
   ├─ System.Threading.Tasks.TaskCanceledException
   ├─ System.OperationCanceledException
   └─ (ngoại lệ tuỳ miền / thư viện khác)
```

**Gợi ý nhận diện nhanh** (không đầy đủ):
- **Lập trình phòng vệ/đầu vào**: `ArgumentNullException`, `ArgumentOutOfRangeException`, `FormatException`
- **Trạng thái sai**: `InvalidOperationException`, `NotSupportedException`
- **I/O**: `IOException` (file lock, mất kết nối…), `FileNotFoundException`, `DirectoryNotFoundException`
- **Số học**: `DivideByZeroException`, `OverflowException` (khi `checked`)
- **Thời gian**: `TimeoutException`
- **Mạng/HTTP**: `HttpRequestException`
- **Huỷ**: `OperationCanceledException` / `TaskCanceledException` — §11

`NullReferenceException` = bug (thiếu guard/NRT), không bắt để “sửa null” ở tầng sâu.

---

## 5. Bắt `Exception` đúng cách

### 5.1 Bắt cụ thể trước, tổng quát sau

```csharp
try
{
    Save();
}
catch (IOException ex)
{
    // xử lý cho I/O
}
catch (Exception ex) // tổng quát, đặt sau
{
    // fallback / log
    throw; // thường rethrow để không nuốt lỗi
}
```

Thứ tự: compiler chọn `catch` **đầu tiên khớp kiểu** (không phải “gần nhất trên hierarchy” nếu viết sai thứ tự — `catch (Exception)` trước `IOException` là lỗi compile unreachable).

### 5.2 Không bắt `Exception` vô tội vạ
- Chỉ bắt ở **biên rìa** (UI, job runner, middleware) để log và hiển thị thông báo thân thiện.
- Ở tầng trong (domain, repository…), hãy để lỗi **bubble up**.

### 5.3 `catch` rỗng là code smell

```csharp
catch
{
    // nuốt lỗi, gây khó debug
}
```

Hãy **ít nhất log** hoặc chuyển sang trạng thái an toàn. `catch { }` còn nuốt `OutOfMemoryException` / `StackOverflowException` (một số không bắt được) và **OCE** — hủy bị nuốt thành “thành công”.

---

## 6. Rethrow, bọc & bảo toàn stack trace

### 6.1 Rethrow đúng

```csharp
catch (Exception)
{
    // Do something...
    throw; // ✅ giữ nguyên stack
}
```

**Tránh**: `throw ex;` vì sẽ **reset** stack trace về dòng `throw ex` — mất nơi lỗi gốc.

```csharp
catch (Exception ex)
{
    Log(ex);
    throw;      // stack: Save → File.Write → ...
    // throw ex; // stack: bắt đầu tại catch này
}
```

`throw;` chỉ hợp lệ trong `catch` đang active. Ngoài `catch`: phải `throw new` hoặc `ExceptionDispatchInfo` (§6.3).

### 6.2 Bọc ngoại lệ với InnerException

```csharp
try
{
    ParseConfig();
}
catch (FormatException ex)
{
    throw new ConfigurationException("Config invalid", ex);
}
```

- Cung cấp **ngữ cảnh** giàu thông tin (`ConfigurationException`) nhưng **không làm mất** exception gốc (`InnerException`).
- Caller bắt `ConfigurationException` vẫn `ex.InnerException` để log root cause.
- Đừng bọc nhiều lớp vô nghĩa (`Wrapper1(Wrapper2(IOException))`) — một lớp nghiệp vụ đủ.

### 6.3 Bảo toàn stack trace thủ công (nâng cao)

Khi bắt rồi **lưu** exception, ném lại **sau** (sau retry logic, sau `finally` khác):

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? edi = null;
try { ... }
catch (Exception ex)
{
    edi = ExceptionDispatchInfo.Capture(ex);
}
// ... cleanup không liên quan
edi?.Throw(); // giữ stack gốc — không reset như throw ex
```

```csharp
try { ... }
catch (Exception ex)
{
    ExceptionDispatchInfo.Capture(ex).Throw();
    throw; // unreachable — một số analyzer muốn throw sau Capture.Throw()
}
```

`await` faulted task đã dùng EDI bên trong — bạn không cần Capture khi chỉ `await`.

**Bảng rethrow:**

| Câu | Stack | Dùng khi |
|---|---|---|
| `throw;` | Giữ | Trong `catch`, log rồi bubble |
| `throw ex;` | **Reset** tại đây | **Không** — gần như luôn sai |
| `throw new X("...", ex)` | Stack mới + Inner | Thêm ngữ cảnh domain |
| `ExceptionDispatchInfo.Capture(ex).Throw()` | Giữ | Ném lại *ngoài* `catch` / sau delay |

```csharp
Exception? pending = null;
try { await StepAsync(); }
catch (Exception ex) { pending = ex; }
await CleanupAsync();
if (pending is not null)
    ExceptionDispatchInfo.Capture(pending).Throw();
```

---

## 7. Exception filter với `when`

Lọc điều kiện **trước khi** vào khối `catch` — **không** unwind stack nếu `when` false (CLR thử `catch` kế).

```csharp
try
{
    await SendAsync();
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    await Task.Delay(1000);
    // retry nhẹ
}
```

**WHY không `catch` rồi `if` + `throw;`:**

| | `when (cond)` | `catch { if (!cond) throw; }` |
|---|---|---|
| Unwind | Chỉ khi `when` true | Unwind **trước**, rồi rethrow |
| Stack | Giữ nguyên lần ném | `throw;` giữ stack nhưng đã vào catch (filter debug / finally ngoài) |
| `when` false | Catch khác / bubble **như chưa bắt** | Phải rethrow |
| Side-effect `when` | Chạy khi đang xét filter (stack chưa unwind) | — |

**Pitfall `when`:**

1. **Exception trong `when`:** filter ném → che exception gốc (hành vi CLR). `when` phải **thuần**, không I/O, không null-deref.
2. **Gọi method nặng** trong `when` — chạy trên đường exception (hiếm nhưng đắt).
3. **`when (ex is ...)`** lặp type đã có trên `catch`.
4. Bắt OCE bằng filter token: `catch (OperationCanceledException) when (ct.IsCancellationRequested)` — §11.

```csharp
catch (IOException ex) when (ex.HResult == unchecked((int)0x80070020)) // sharing violation
{
    await Task.Delay(50);
    retry = true;
}
```

Logging “mọi exception nhưng không bắt”: `catch (Exception ex) when (LogAndFalse(ex))` với `bool LogAndFalse(Exception ex) { Log(ex); return false; }` — log rồi bubble, **không** swallow. Dùng tiết chế (mọi exception path gọi log).

**Filter vs nhiều `catch`:**

```csharp
catch (HttpRequestException ex) when ((int?)ex.StatusCode is 408 or 429 or >= 500)
{
    return Result.Retry;
}
catch (HttpRequestException ex) when ((int?)ex.StatusCode is >= 400 and < 500)
{
    return Result.FailClient(ex);
}
```

Hai `catch` cùng type + `when` khác = phân nhánh **không** unwind nhánh sai. Viết `catch (HttpRequestException ex) { if (retryable) ... else throw; }` thì đã vào catch (debugger break on thrown vẫn “caught”).

`when` **không** thay `if` trong logic thành công — chỉ trên đường exception.

---

## 8. Quản lý tài nguyên: `finally`, `using`, `await using`

### 8.1 Sử dụng `finally` để đảm bảo thu hồi

```csharp
Stream? s = null;
try
{
    s = File.OpenRead(path);
    // ...
}
finally
{
    s?.Dispose();
}
```

`finally` chạy khi `return`, `break`, exception. Không chạy nếu process kill / `FailFast` / (một số) `StackOverflowException`.

### 8.2 `using` & `await using` (khuyến nghị)

```csharp
using var stream = File.OpenRead(path);
using var reader = new StreamReader(stream);
string text = reader.ReadToEnd();
```

Async:

```csharp
await using var conn = await OpenConnectionAsync();
var rows = await conn.QueryAsync(...);
```

- Dựa trên `IDisposable` / `IAsyncDisposable`.
- **Đảm bảo** giải phóng kể cả có ngoại lệ.

> Xem thêm: mẫu *Dispose pattern* với resource unmanaged (nếu cần). So sánh declaration vs statement: [statements.md §9](statements.md#92-using-statement-vs-using-declaration).

Dispose trong `finally` ném **che** exception gốc (`ExceptionDispatchInfo` / `AggregateException` tùy runtime). Tránh `throw` trong `Dispose` nếu có thể.

---

## 9. Thiết kế ngoại lệ tuỳ biến (Custom exceptions)

### 9.1 Khi nào cần?
- Cần thêm **ngữ cảnh nghiệp vụ** (domain-specific).
- Muốn caller **bắt riêng** lỗi của domain.

### 9.2 Hướng dẫn
- Kế thừa trực tiếp từ `Exception` (hoặc một nhánh hợp lý).
- Đặt hậu tố `Exception`, có ctor chuẩn.

```csharp
[Serializable]
public class ConfigurationException : Exception
{
    public ConfigurationException() { }
    public ConfigurationException(string message) : base(message) { }
    public ConfigurationException(string message, Exception inner) : base(message, inner) { }
    protected ConfigurationException(
      System.Runtime.Serialization.SerializationInfo info,
      System.Runtime.Serialization.StreamingContext context) : base(info, context) { }
}
```

> Với .NET hiện đại, `[Serializable]`/ctor serialization chỉ cần nếu bạn thật sự cần cross-appdomain/interop cũ.

---

## 10. Ngoại lệ trong async/await & song song

### 10.1 `await` *unwrap* ngoại lệ

`Task` faulted lưu exception trong `Task.Exception` kiểu **`AggregateException`**. `await` **không** ném wrapper đó — gọi `GetResult()` / EDI → ném **inner đầu tiên** (exception gốc).

```csharp
try
{
    await TaskThatFailsAsync();
}
catch (InvalidOperationException ex)
{
    // ex là exception gốc, KHÔNG phải AggregateException
}
catch (Exception ex)
{
    Console.WriteLine(ex is AggregateException); // false với await một Task
}
```

**WHY:** `try/catch` giống sync. `.Result` giữ mô hình TPL cổ (`AggregateException`).

Nhiều inner (WhenAll): `await` vẫn ném **một**; inner khác trên `task.Exception.InnerExceptions`.

```csharp
var t = Task.FromException(new IOException("disk"));
try { await t; }
catch (IOException) { /* khớp */ }

try { t.Wait(); }
catch (AggregateException aex) when (aex.InnerException is IOException) { }
```

### 10.2 `.Result` / `.Wait()` bọc trong `AggregateException`

```csharp
try
{
    TaskThatFailsAsync().Wait(); // hoặc .Result
}
catch (AggregateException aex)
{
    foreach (var ex in aex.Flatten().InnerExceptions)
        Log(ex);
}
```

`Flatten()` bung lồng `AggregateException`. Deadlock: [async.md §6](async.md#6-capture-context--configureawait). `GetAwaiter().GetResult()` unwrap giống `await` nhưng **block** — vẫn deadlock UI, khác `.Result` ở type ném (gốc vs Aggregate).

### 10.3 Nhiều task: `Task.WhenAll`

```csharp
try
{
    await Task.WhenAll(tasks);
}
catch
{
    // một hay nhiều task lỗi → Exception từ task đầu tiên; có thể duyệt tasks để lấy tất cả
    var faults = tasks.Where(t => t.IsFaulted).Select(t => t.Exception).ToList();
}
```

Cancel mixed fault: một canceled + một faulted → `WhenAll` faulted (ưu tiên lỗi). Inspect từng `Status`.

### 10.4 `async void` & unobserved exceptions
- `async void` **chỉ** dùng cho event handler.
- Lỗi trong `async void` đi vào `SynchronizationContext` → có thể **crash** app nếu không bắt.
- Unobserved `Task`: GC finalize có thể log `UnobservedTaskException` — **luôn await** hoặc `_ = Observe(task)`.

### 10.5 TPL/Parallel/PLINQ
- `Parallel.For/ForEach` & `Parallel.Invoke` ném `AggregateException`.
- PLINQ (`AsParallel()`) lỗi cũng gói trong `AggregateException`.
- `await` Parallel.ForEachAsync: exception giống async (unwrap) tùy API — kiểm tra docs; thường một exception surface, còn lại trên task.

---

## 11. Cancellation vs Exception

- Huỷ là **tình huống dự kiến** (user, shutdown, timeout chủ đích) → `CancellationToken`.
- Khi token bị huỷ: ném `OperationCanceledException` (hoặc `TaskCanceledException` : OCE) **kèm token** (`ex.CancellationToken`).
- **Không** dùng OCE cho lỗi nghiệp vụ (“user không tồn tại”). **Không** bắt `Exception` rồi nuốt OCE như lỗi đã xử lý.

```csharp
public async Task WorkAsync(CancellationToken ct)
{
    ct.ThrowIfCancellationRequested();
    await Task.Delay(500, ct);
}
```

Caller **phân biệt** giữa lỗi thật và huỷ:

```csharp
try
{
    await WorkAsync(cts.Token);
}
catch (OperationCanceledException) when (cts.IsCancellationRequested)
{
    // coi như “bình thường” — user/host hủy
}
```

**Filter `when` quan trọng:** OCE có thể đến từ **token khác** (timeout lồng, HTTP hủy trong). `catch (OperationCanceledException)` không điều kiện có thể nuốt hủy *bên trong* thư viện trong khi caller vẫn muốn fail.

```csharp
catch (OperationCanceledException ex) when (ex.CancellationToken == cts.Token)
{
    // đúng nguồn hủy của chúng ta
}
```

| | Cancellation (OCE / TCE) | Exception lỗi |
|---|---|---|
| Ý nghĩa | Dừng **có chủ đích** | Thất bại |
| Log | Debug / metrics “canceled”; không Error storm | Error / Warning |
| HTTP | 499 / 408 tùy host — không 500 | 5xx / ProblemDetails |
| Retry | Không retry như transient fault (trừ timeout có policy) | Retry theo loại |
| `throw;` vs nuốt | Biên app: nuốt có ý; giữa pipeline: **rethrow** để host dừng | Không nuốt |

**Timeout:** `CancelAfter` → vẫn OCE, không `TimeoutException` trừ khi bạn dịch. `Task.WaitAsync(timeout)` (.NET 6+) có thể ném `TimeoutException` — **khác** cancel token. Đừng `catch (Exception)` gộp timeout + bug.

```csharp
try
{
    await WorkAsync(ct).WaitAsync(TimeSpan.FromSeconds(5), ct);
}
catch (TimeoutException)
{
    // hết giờ WaitAsync
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // user cancel
}
```

`TaskCanceledException` khi `Delay(ct)` bị cancel — bắt `OperationCanceledException`. `HttpClient` hủy: thường OCE, đôi khi `HttpRequestException` bọc — đọc inner.

Middleware ASP.NET: hủy request client → OCE; **đừng** log Error mỗi lần (scan/bot).

**`TaskCanceledException` vs `OperationCanceledException`:**

- `Task.Delay(ct)` / nhiều API Task: ném `TaskCanceledException`.
- `ct.ThrowIfCancellationRequested()`: ném `OperationCanceledException` (không nhất thiết TCE).
- `catch (OperationCanceledException)` bắt **cả hai**. `catch (TaskCanceledException)` **trượt** OCE thuần.

```csharp
catch (TaskCanceledException)
{
    // Delay/HttpClient một số path — KHÔNG bắt ThrowIfCancellationRequested
}
```

Luôn bắt base `OperationCanceledException` trừ khi bạn cố ý chỉ TCE.

**Linked CTS:** `CreateLinkedTokenSource(a, b)` — cancel **một** nguồn → OCE với token **linked** (không phải `a` hay `b` nguyên bản). Filter `ex.CancellationToken == a` có thể **false**. Dùng `a.IsCancellationRequested || b.IsCancellationRequested` hoặc `linked.Token`.

---

## 12. Global handling & logging

### 12.1 Ứng dụng desktop (WPF/WinForms)
- `AppDomain.CurrentDomain.UnhandledException` – bắt ngoại lệ **không bắt được** (cuối).
- WPF: `Application.DispatcherUnhandledException`
- WinForms: `Application.ThreadException`

> Dùng để **log & báo lỗi thân thiện**, không nên tiếp tục chạy nếu trạng thái không an toàn.

### 12.2 ASP.NET Core
- Dùng middleware `UseExceptionHandler` cho production; `UseDeveloperExceptionPage` cho dev.
- Log qua `ILogger`. Trả về response chuẩn hoá (ProblemDetails).
- OCE do abort request: filter / `IExceptionFilter` **không** biến thành 500.

### 12.3 Worker/Service
- Quấn entrypoint bằng `try/catch` tổng → log + exit code phù hợp.
- Lập lịch job: đảm bảo **retry policy** hợp lý (exponential backoff, circuit breaker).
- `catch (Exception)` trong `BackgroundService.ExecuteAsync` **tách** OCE shutdown.

---

## 13. Hiệu năng & quy tắc sử dụng ngoại lệ

- **Không dùng ngoại lệ cho luồng thường** → thiết kế API theo cặp:
  - `Parse` **ném lỗi** khi input sai nghiêm trọng;
  - `TryParse` **không ném**, trả `bool` + `out`.
- **Guard clauses** sớm với `ArgumentNullException.ThrowIfNull(arg);`.
- **Thông báo rõ ràng**: message ngắn gọn, nêu *ngữ cảnh* và *cách khắc phục* nếu có.
- **Không lạm dụng bắt/đổi ngoại lệ**: chỉ bọc khi bạn **thêm giá trị ngữ cảnh**.
- **Đừng nuốt lỗi** (catch rỗng) – luôn log/lộ diện ở rìa hệ thống. Đừng nuốt OCE.
- **Đo lường**: exceptions đắt đỏ → tránh ném trong vòng lặp nội, hot path.
- `throw;` không `throw ex;`. Filter `when` cho phân loại; `when` không side-effect độc.

---

## 14. Kiểm tra tràn số: `checked` / `unchecked`

```csharp
int a = int.MaxValue;
int b = 1;

checked
{
    // ném OverflowException
    int c = a + b;
}

unchecked
{
    // tràn im lặng (wrap-around)
    int d = a + b;
}
```

- Có thể bật `checked` theo **khối**, **biểu thức**, hoặc **mức project**.

---

## 15. Cheat sheet & checklist

**Bắt đầu**  
- [ ] Bắt ngoại lệ **cụ thể** trước, tổng quát sau.  
- [ ] Không bắt `Exception` tràn lan; chỉ ở rìa app để log/hiển thị.  
- [ ] Luôn `throw;` khi rethrow, **không** `throw ex;`.  
- [ ] Dùng `when` để lọc điều kiện (retry, throttle, đúng token hủy).  
- [ ] Guard: `ThrowIf*` đầu method; `Try*` cho input dự kiến sai.

**Tài nguyên**  
- [ ] Ưu tiên `using` / `await using` cho `IDisposable` / `IAsyncDisposable`.  
- [ ] Chỉ dùng finalizer khi cần tài nguyên unmanaged.

**Async/Parallel**  
- [ ] Trong async, **luôn** `await` Task (tránh nuốt lỗi).  
- [ ] `await` → exception **gốc**; `.Wait`/`.Result` → `AggregateException`.  
- [ ] Tránh `async void` (trừ event).  
- [ ] `WhenAll` → biết cách thu thập nhiều lỗi (`IsFaulted`, `Exception`).

**Cancellation**  
- [ ] OCE ≠ 500 / Error log. Filter theo token.  
- [ ] Không `catch (Exception)` nuốt hủy.

**Thiết kế API**  
- [ ] Phân biệt lỗi business vs lỗi hệ thống vs hủy.  
- [ ] Dùng `TryXxx` cho lỗi dự kiến/nhẹ; `Xxx` ném exception cho điều kiện bất thường.  
- [ ] Custom exception có tên rõ, ctor đầy đủ, bọc `InnerException` khi cần.

**Hiệu năng & logging**  
- [ ] Log đủ ngữ cảnh (request-id, user, input chính).  
- [ ] Tránh ném/bắt liên tục trong hot path.  
- [ ] Theo dõi `UnhandledException`/`UnobservedTaskException` để phát hiện rò rỉ lỗi.

---

> Kết hợp chương này với **Methods**, **Async**, **Type System** sẽ giúp bạn xây dựng API rõ ràng, an toàn và dễ vận hành.
