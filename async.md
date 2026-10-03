# Lập trình bất đồng bộ trong C#  
*(async/await, Task, async streams)*

> **Baseline:** .NET **10** / C# **14**. Thread / Channels đồng bộ thấp hơn → [threading.md](threading.md).

Chương này tập trung vào **lập trình bất đồng bộ (asynchronous)**:

- `async` / `await` và cơ chế state machine bên dưới,
- Các kiểu trả về: `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `async void`,
- Cancellation, progress, exception trong async,
- **Async streams**: `IAsyncEnumerable<T>` và `await foreach`,
- Timer / delay patterns (`Task.Delay`, `PeriodicTimer`), Channel + async.

**WHY async:** giải phóng thread khi chờ I/O — không biến CPU-bound thành “nhanh hơn”. Nhầm `async` với `Task.Run` / thread mới là nguồn deadlock và thread-pool starve.

---

## Mục lục

- [Lập trình bất đồng bộ trong C#](#lập-trình-bất-đồng-bộ-trong-c)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan: Vì sao cần async?](#1-tổng-quan-vì-sao-cần-async)
  - [2. Async method là gì?](#2-async-method-là-gì)
  - [3. Kiểu trả về của async method](#3-kiểu-trả-về-của-async-method)
    - [3.1 `Task` \& `Task<T>` – phổ biến nhất](#31-task--taskt--phổ-biến-nhất)
    - [3.2 `ValueTask` / `ValueTask<T>` — quy tắc dùng](#32-valuetask--valuetaskt--quy-tắc-dùng)
    - [3.3 `async void` – trường hợp đặc biệt](#33-async-void--trường-hợp-đặc-biệt)
  - [4. `await` hoạt động như thế nào?](#4-await-hoạt-động-như-thế-nào)
  - [5. Async state machine](#5-async-state-machine)
    - [5.1 Fast path vs yield, boxing, locals](#51-fast-path-vs-yield-boxing-locals)
    - [5.2 Nhiều `await`, `try`/`finally`, giới hạn `ref struct`](#52-nhiều-await-tryfinally-giới-hạn-ref-struct)
  - [6. Capture context \& `ConfigureAwait`](#6-capture-context--configureawait)
  - [7. Exception trong async method](#7-exception-trong-async-method)
    - [7.1 Khi `await` một Task](#71-khi-await-một-task)
    - [7.2 Khi không `await` Task](#72-khi-không-await-task)
  - [8. Cancellation \& IProgress](#8-cancellation--iprogress)
    - [8.1 CancellationToken](#81-cancellationtoken)
    - [8.2 Best practices CancellationToken](#82-best-practices-cancellationtoken)
    - [8.3 Báo tiến độ với `IProgress<T>`](#83-báo-tiến-độ-với-iprogresst)
  - [9. Best practices khi dùng async/await](#9-best-practices-khi-dùng-asyncawait)
  - [10. Async streams – `IAsyncEnumerable<T>` \& `await foreach`](#10-async-streams--iasyncenumerablet--await-foreach)
    - [10.1 Vấn đề trước khi có async streams](#101-vấn-đề-trước-khi-có-async-streams)
    - [10.2 `IAsyncEnumerable<T>` \& `IAsyncEnumerator<T>`](#102-iasyncenumerablet--iasyncenumeratort)
    - [10.3 Async iterator method](#103-async-iterator-method)
    - [10.4 Duyệt bằng `await foreach`](#104-duyệt-bằng-await-foreach)
  - [11. Async streams: Cancellation, exception, best practices](#11-async-streams-cancellation-exception-best-practices)
    - [11.1 Cancellation](#111-cancellation)
      - [Pattern 1: `[EnumeratorCancellation]` + `WithCancellation`](#pattern-1-enumeratorcancellation--withcancellation)
      - [Pattern 2: Truyền `CancellationToken` bình thường](#pattern-2-truyền-cancellationtoken-bình-thường)
    - [11.2 Exception \& dispose](#112-exception--dispose)
    - [11.3 So sánh \& best practices](#113-so-sánh--best-practices)
  - [12. Channel + async](#12-channel--async)
  - [13. `PeriodicTimer` vs `Task.Delay`](#13-periodictimer-vs-taskdelay)
  - [14. Ghép tác vụ và giới hạn đồng thời](#14-ghép-tác-vụ-và-giới-hạn-đồng-thời)

---

## 1. Tổng quan: Vì sao cần async?

Bài toán kinh điển:

- Bạn cần gọi HTTP, truy vấn DB, đọc/ghi file, gọi một service từ xa…
- Nếu dùng API đồng bộ (blocking), thread sẽ phải **chờ** I/O → lãng phí tài nguyên, UI treo, server khó scale.

Ví dụ blocking:

```csharp
// Blocking – thread đứng im chờ
var data = httpClient.GetStringAsync(url).Result;
```

Cách hiện đại với async/await:

```csharp
// Non-blocking – thread được trả lại để làm việc khác
var data = await httpClient.GetStringAsync(url);
```

Điểm quan trọng:

- **Bạn viết code trông như code tuần tự**, try/catch bình thường,
- **Compiler** chuyển method thành state machine và đăng ký continuation. API I/O bất đồng bộ cho phép trả thread trong lúc chờ; chỉ thêm `async` vào một method blocking không tạo ra lợi ích đó.

`.Result` / `.Wait()` trên UI thread (hoặc ASP.NET cũ có SyncContext) dễ **deadlock**: thread giữ context, Task continuation cần context đó. Async all the way — xem §6 và §9.

---

## 2. Async method là gì?

**Async method** là method:

- Có từ khóa `async`,
- Thường (nên) có ít nhất một `await`,
- Trả về `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>` hoặc `void` (special case).

Ví dụ:

```csharp
public async Task<string> DownloadAsync(string url)
{
    using var client = new HttpClient();
    string content = await client.GetStringAsync(url);
    return content;
}
```

Đặc trưng:

- **Không phải** cứ `async` là chạy trên thread khác – nó **chạy trên thread hiện tại** cho tới khi gặp `await` trên một tác vụ chưa hoàn thành.
- Tại mỗi `await`:
  - Nếu tác vụ đã xong → chạy tiếp như bình thường.
  - Nếu chưa xong → method **tạm thoát ra**, trả về một `Task` cho caller, và khi tác vụ xong, nó sẽ quay lại chạy từ sau `await`.
- Một hàm async không có `await` vẫn chạy **hết trên thread gọi**, rồi trả `Task` đã hoàn thành. Compiler cảnh báo **CS1998**. `async` không đưa việc sang thread khác — muốn vậy thì `Task.Run`: [threading.md §3.2](threading.md#32-taskrun--quan-hệ-với-pool).

```csharp
public async Task<int> NoAwaitAsync() // CS1998
{
    return 1; // chạy đồng bộ; Task đã RanToCompletion trước khi caller nhận
}
```

---

Sơ đồ tuần tự (sequence diagram)

```csharp
async Task<int> FooAsync()
{
    var x = await BarAsync();      // (A)
    return x + 1;                  // (B)
}

// Gọi:
Task<int> t = FooAsync();          // (C)
int r = await t;                   // (D)
```

```
Caller                  FooAsync (SM)              BarAsync Task/Awaiter           Context/ThreadPool
  |                           |                              |                               |
  |---- call FooAsync() ----->|                              |                               |
  |                           |-- create SM & Task ----------|                               |
  |                           |-- await BarAsync() ----------> create awaiter                |
  |                           |                              |-- IsCompleted? ---------------|
  |                           |                              |            |                  |
  |                           |          (YES) inline path   |            | (NO) async path  |
  |                           |<----- GetResult() -----------|            |                  |
  |                           |-- do (B) & set result -------|            |                  |
  |<----------- Task done ----|                              |            |                  |
  |                           |                                           |-- OnCompleted(SM.MoveNext)
  |                           |                                           |-- later: completion happens
  |                           |<-------------------- MoveNext ------------(posted via captured context)
  |                           |-- GetResult(); do (B); set result ------- |
  |<----------- Task done ----|                                           |
```

---

## 3. Kiểu trả về của async method

Các kiểu trả về hợp lệ:

1. `Task`
2. `Task<T>`
3. `ValueTask`
4. `ValueTask<T>`
5. `void` (chỉ dùng cho event handler)

Async method còn có thể trả custom **task-like type** với async method builder phù hợp (`AsyncMethodBuilderAttribute`). Chỉ có `GetAwaiter()` chưa đủ để làm kiểu trả về của async method. Async iterator trả `IAsyncEnumerable<T>` hoặc `IAsyncEnumerator<T>` (§10).

### 3.1 `Task` & `Task<T>` – phổ biến nhất

```csharp
public async Task DoWorkAsync()
{
    await Task.Delay(500);
}

public async Task<int> CalculateAsync()
{
    await Task.Delay(500);
    return 42;
}
```

Dùng:

```csharp
await DoWorkAsync();
int result = await CalculateAsync();
```

Điểm mạnh:

- Dễ compose, `await` được nhiều lần,
- Tích hợp với các API `Task`-based trong .NET (`WhenAll`, `WhenAny`, `WaitAsync`, Channel, EF…).

`Task` là class — mỗi lần `async` chưa hoàn thành đồng bộ thường **cấp phát** Task + box state machine khi yield.

### 3.2 `ValueTask` / `ValueTask<T>` — quy tắc dùng

Tối ưu khi kết quả **thường đã sẵn** (cache hit / sync path) và muốn tránh cấp phát `Task`:

```csharp
public async ValueTask<int> GetCachedOrComputeAsync()
{
    if (TryGetFromCache(out var value))
        return value; // sync, không allocate Task

    int computed = await ComputeAsync();
    SaveToCache(computed);
    return computed;
}
```

`ValueTask<T>` là **struct** (có thể bọc `T` hoàn thành, `Task<T>`, hoặc `IValueTaskSource<T>`). Hot path sync: không object `Task`.

**Khi nên dùng:**

- Hot path, tỷ lệ hoàn thành đồng bộ cao, đo được áp lực GC từ `Task`.
- Implement interface/`IValueTaskSource` trong thư viện hạ tầng (Channel reader, `PipeReader`, socket).

**Khi không nên:**

- API ứng dụng thông thường → `Task`/`Task<T>` đơn giản, dễ compose.
- Cần `await` **nhiều lần**, lưu vào field, hoặc `Task.WhenAll` trực tiếp trên nhiều `ValueTask` (phải `.AsTask()` khi cần).

**Quy tắc (bắt buộc nhớ):**

1. **Await tối đa một lần.** Consume xong (await / `.Result` / `.GetAwaiter().GetResult()`) → xong. Await lần hai = undefined (sai kết quả, exception, hoặc reuse underlying source).
2. **Không** `await` đồng thời hai nơi trên cùng instance.
3. **Không** `.GetAwaiter()` rồi bỏ — phải `GetResult` hoặc await.
4. Sau await, **đừng** dùng lại biến `ValueTask` đó.
5. Cần fan-out / store / `WhenAll`: `vt.AsTask()` **một lần**, rồi dùng `Task`.
6. `ValueTask` hoàn thành sync: `.IsCompletedSuccessfully` rồi `.Result` hợp lệ — vẫn chỉ một lần.
7. Generic overload `Preserve()` (.NET) khi phải inspect trước khi await — chỉ khi API cho phép; mặc định coi như single-consumption.

```csharp
ValueTask<int> vt = GetCachedOrComputeAsync();
int a = await vt;
// int b = await vt; // SAI — không await ValueTask đã consume

ValueTask<int> vt2 = GetCachedOrComputeAsync();
Task<int> asTask = vt2.AsTask(); // khi cần API Task-based
await Task.WhenAll(asTask, OtherAsync());
```

```csharp
async Task<int> BadWhenAll()
{
    ValueTask<int> a = GetAsync();
    ValueTask<int> b = GetAsync();
    // return await Task.WhenAll(a, b); // không compile
    return (await a.AsTask()) + (await b.AsTask());
}
```

Nhược điểm: API phức tạp hơn, dễ dùng sai → **chỉ dùng khi có lý do hiệu năng đo được**. Public app API: `Task` trừ khi bạn viết infrastructure.

**`IValueTaskSource` (hạ tầng):** Channel/Pipe tái dùng object nguồn — `ValueTask` trỏ tới source + token version. Consume xong, source **recycle**. Await lần hai = version lệch → undefined. Đây là lý do quy tắc “một lần” không chỉ lý thuyết.

```csharp
ValueTask<int> vt = reader.ReadAsync(ct);
if (vt.IsCompletedSuccessfully)
    return vt.Result; // sync consume — vẫn chỉ một lần
return await vt;
```

Đừng `vt.Result` khi `!IsCompleted` — block hoặc ném. `Preserve()` (.NET 5+) tách kết quả khỏi `IValueTaskSource` (sync thì giữ `T`, chưa xong thì bọc `Task`) để được inspect / `await` nhiều lần. Bản gốc sau `Preserve` không được dùng — source có thể đã recycle.

```csharp
ValueTask<int> raw = reader.ReadAsync(ct);
ValueTask<int> stable = raw.Preserve(); // chỉ dùng stable từ đây

int n = stable.IsCompletedSuccessfully
    ? stable.Result
    : await stable;
// await stable lần nữa được; await raw thì không
```

### 3.3 `async void` – trường hợp đặc biệt

```csharp
public async void OnButtonClick(object sender, EventArgs e)
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex)
    {
        Log(ex); // không có Task để caller await/catch — chỉ bắt được ở đây
    }
}
```

Bỏ `try`: exception thoát khỏi `async void` vào `SynchronizationContext` (UI có thể crash), không vào `TaskScheduler.UnobservedTaskException`.

**Chỉ nên dùng cho event handler**, vì:

- Không thể `await` → caller không biết khi nào xong.
- Exception không đi qua `Task`, khó bắt → có thể crash app.

Best practice: **mọi method async nên trả về `Task` hoặc `Task<T>`**, trừ event handler UI.

---

## 4. `await` hoạt động như thế nào?

Cú pháp cơ bản:

```csharp
var result = await SomeAsyncOperation();
```

Một biểu thức **awaitable** phải có:

- `GetAwaiter()` trả về một awaiter,
- Awaiter có:
  - `bool IsCompleted { get; }`
  - `void OnCompleted(Action continuation)` hoặc `UnsafeOnCompleted`
  - `T GetResult()` (hoặc `void GetResult()`)

Các kiểu phổ biến:

- `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`
- Custom type có `GetAwaiter()`: awaiter cần `IsCompleted`, `GetResult()` và triển khai `INotifyCompletion` (hoặc `ICriticalNotifyCompletion`).

Quy trình (đơn giản hoá):

1. Gọi `var awaiter = expr.GetAwaiter();`
2. Nếu `awaiter.IsCompleted`:
   - Gọi `awaiter.GetResult()` **ngay**,
   - Tiếp tục chạy code sau `await`.
3. Nếu chưa completed:
   - Đăng ký callback: `awaiter.OnCompleted(continuation)`,
   - Async method **kết thúc tạm thời**, trả về một `Task` chưa hoàn thành,
   - Khi tác vụ hoàn thành → runtime gọi `continuation` → tiếp tục method từ sau `await`.

`GetResult()` trên faulted awaiter **ném** exception đã unwrap (không `AggregateException` cho single Task).

---

## 5. Async state machine

**WHY compiler sinh SM:** method phải “dừng” giữa chừng mà vẫn giữ local, rồi resume — không stack-split như coroutine native. SM là struct (đầu) implement `IAsyncStateMachine`, field `_state`, locals/awaiter, `MoveNext()`.

Ví dụ method:

```csharp
public async Task<int> FooAsync()
{
    Console.WriteLine("A");
    await Task.Delay(1000);
    Console.WriteLine("B");
    return 42;
}
```

Compiler sẽ:

- Tạo một struct/class ẩn cài đặt `IAsyncStateMachine`,
- Sinh ra trường `_state` để theo dõi “đang ở đoạn nào”,
- Sinh `AsyncTaskMethodBuilder<int>` để quản lý `Task<int>` trả về,
- Sinh `MoveNext()` với một `switch(_state)`.

**Lưu trữ:** build Release thường sinh state machine dạng struct; Debug có thể sinh class. Khi phải suspend, trạng thái cần sống qua lần gọi hiện tại. .NET hiện đại thường lưu nó trong `AsyncStateMachineBox<TStateMachine>` kết hợp với Task, thay vì luôn cấp phát một box riêng và một Task riêng.

Ý tưởng pseudo-code (giản lược):

```csharp
struct FooAsyncStateMachine : IAsyncStateMachine
{
    public int _state;
    public AsyncTaskMethodBuilder<int> _builder;
    private TaskAwaiter _awaiter;

    public void MoveNext()
    {
        int result;
        try
        {
            if (_state == -1)
            {
                Console.WriteLine("A");
                _awaiter = Task.Delay(1000).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _state = 0;
                    _builder.AwaitOnCompleted(ref _awaiter, ref this);
                    return; // caller nhận Task chưa xong
                }
            }

            if (_state == 0)
            {
                _awaiter.GetResult(); // ném nếu Delay fault/cancel
            }

            Console.WriteLine("B");
            result = 42;
        }
        catch (Exception ex)
        {
            _builder.SetException(ex);
            return;
        }

        _builder.SetResult(result);
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

`FooAsync` thực tế trông như:

```csharp
public Task<int> FooAsync()
{
    var sm = new FooAsyncStateMachine();
    sm._builder = AsyncTaskMethodBuilder<int>.Create();
    sm._state = -1;
    sm._builder.Start(ref sm);
    return sm._builder.Task;
}
```

**Hệ quả thực tế:**

| Hiện tượng | Liên quan SM |
|---|---|
| Local “sống” qua `await` | Field trên SM (heap nếu yielded) |
| `try/finally` qua `await` | `finally` chạy khi SM hoàn thành / exception |
| Không `lock` quanh `await` | Monitor gắn thread; resume có thể thread khác |
| `Span` / `ref struct` local qua `await` | **Cấm** — không store trên SM |
| Nhiều `await` | Nhiều `_state`; mỗi yield có thể allocate awaiter |

Bạn không cần nhớ IL, giữ mindset:

> `async`/`await` = compiler sinh state machine để chạy code không blocking, nhìn vẫn như code tuần tự. Chi phí thật sự ở **yield** (I/O chưa xong), không phải từ khóa `async`.

### 5.1 Fast path vs yield, boxing, locals

Hai đường trong `MoveNext`:

1. **Fast path** — `awaiter.IsCompleted == true` (cache, `Task.FromResult`, I/O đã xong): `GetResult()` inline, `_state` không nhảy, **không** đăng ký continuation. Method `async` có thể chạy **hoàn toàn đồng bộ** — vẫn trả `Task` completed (hoặc `ValueTask` không alloc).
2. **Yield path** — chưa xong: `AwaitUnsafeOnCompleted` đăng ký callback, `return` khỏi `MoveNext`. Caller nhận Task pending. Khi I/O xong, thread pool (hoặc SyncContext) gọi `MoveNext` lần nữa với `_state` đã lưu.

**State machine qua nhiều await:** trạng thái được tái dùng; không sinh một state machine mới cho từng `await`. Cách cấp phát phụ thuộc builder, runtime và cấu hình build; không nên suy ra số allocation chỉ từ số từ khóa `await`.

```csharp
async Task FastAsync()
{
    await Task.CompletedTask; // IsCompleted → không yield, thường không box
}

async Task SlowAsync()
{
    await Task.Delay(1); // yield → box SM + Task
}
```

Locals cần sống qua `await` trở thành **field** SM (kể cả biến bạn tưởng “chết” trước await — compiler bảo toàn definite assignment). Tên field dạng `<>8__1` — debugger “de-mangle” thành tên local.

### 5.2 Nhiều `await`, `try`/`finally`, giới hạn `ref struct`

Mỗi `await` một `_state` (0, 1, 2…). `switch (_state)` nhảy tới continuation đúng chỗ. `try` bao nhiều await: exception từ bất kỳ `GetResult` đều vào cùng `catch` của SM rồi `SetException`.

`finally` qua `await`: compiler tách — `finally` chạy khi method **kết thúc** (thành công, exception, hoặc iterator dispose), không chạy lúc yield giữa chừng (tài nguyên vẫn “mở” khi đang chờ I/O — đúng ý `await using` giữ connection suốt await).

```csharp
async Task DemoAsync()
{
    Console.WriteLine("enter");
    try
    {
        await Task.Delay(100);
        await Task.Delay(100);
    }
    finally
    {
        Console.WriteLine("leave"); // sau cả hai Delay, hoặc khi exception
    }
}
```

**Cấm** trên SM (không store được): `Span<T>`, `ReadOnlySpan<T>`, `ref` local, hầu hết `ref struct`. Phải dùng xong *trước* `await`, hoặc `Memory<T>` / heap buffer.

```csharp
async Task OkAsync(Memory<byte> buf)
{
    await stream.ReadAsync(buf); // Memory OK
}

async Task BadAsync()
{
    Span<byte> s = stackalloc byte[16];
    // await stream.ReadAsync(s); // lỗi compile
    Fill(s);
}
```

`lock` không được chứa `await` (compiler cấm) vì SM resume **thread khác** — `Monitor` gắn thread. Giới hạn đồng thời qua lần chờ: `SemaphoreSlim`.

```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

async Task UpdateAsync(CancellationToken ct)
{
    await _gate.WaitAsync(ct);
    try
    {
        await SaveAsync(ct); // được await — không giữ lock OS
    }
    finally
    {
        _gate.Release();
    }
}
```

---

## 6. Capture context & `ConfigureAwait`

Trong môi trường có **`SynchronizationContext`** (WPF, WinForms, ASP.NET “cũ”):

```csharp
private async void Button_Click(object sender, EventArgs e)
{
    label.Text = "Loading...";
    var data = await client.GetStringAsync(url);
    // Sau await, chạy lại trên UI thread
    label.Text = data;
}
```

Mặc định:

- `await` sẽ **capture context hiện tại** (`SynchronizationContext` hoặc `TaskScheduler`),
- Khi tiếp tục, nó cố quay lại đúng context (UI thread) để bạn được phép update UI.

Để **không capture context**:

```csharp
var data = await client.GetStringAsync(url).ConfigureAwait(false);
```

**Semantics `ConfigureAwait(false)`:** continuation chạy trên thread pool (hoặc thread hoàn thành I/O), **không** marshal về UI/ASP.NET classic. `ConfigureAwait(true)` = mặc định.

**`ConfigureAwait(ConfigureAwaitOptions)` (.NET 8+)** trên `Task` / `Task<T>`. `ValueTask` / `ValueTask<T>` chỉ có overload nhận `bool`. App thường vẫn chỉ cần `false`. Ba cờ hay gặp (kết hợp bằng `|`):

| Cờ | Việc |
|---|---|
| `ContinueOnCapturedContext` | Giống `ConfigureAwait(true)` — cố về SyncContext |
| `ForceYielding` | Dù awaitable **đã xong**, continuation vẫn chạy sau, không inline trên thread hiện tại |
| `SuppressThrowing` | `await` **không ném**. Chỉ hợp lệ trên `Task` không generic. Fault/cancel đọc ở `IsFaulted` / `IsCanceled` |

```csharp
await Task.Delay(1).ConfigureAwait(ConfigureAwaitOptions.ForceYielding);

Task op = DoWorkAsync();
await op.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing);
if (op.IsFaulted)
    Log(op.Exception!.InnerException);
```

`SuppressThrowing` trên `Task<T>` không hợp lệ — không có giá trị để trả khi lỗi. `ForceYielding` tránh continuation ăn stack của thread vừa hoàn thành I/O; đừng bật mọi `await`.

### Library vs app (.NET Core / .NET 5+)

| Ngữ cảnh | SyncContext điển hình | Gợi ý |
|---|---|---|
| **Class library / SDK** | Không biết host | Nên `ConfigureAwait(false)` hầu hết chỗ — tránh buộc continuation về UI/legacy context của caller |
| **ASP.NET Core** | Thường **không** có SyncContext tùy biến | `ConfigureAwait(false)` ít khác biệt hành vi; vẫn hữu ích nếu library được gọi từ UI host |
| **Console / Worker / Minimal API** | Thường không | Mặc định `await` thường ổn |
| **WPF / WinForms / MAUI** | Có UI SyncContext | App code: thường **không** `false` sau await nếu cần đụng UI; library thuần: `false` |

**Deadlock cổ điển (ASP.NET Framework / UI + `.Result`):**

```csharp
// UI thread
label.Text = http.GetStringAsync(url).Result;
// GetStringAsync continuation muốn về UI; UI đang block .Result → deadlock
```

Sửa: `await` trên UI; hoặc `GetStringAsync(url).ConfigureAwait(false)` **bên trong library** để continuation không cần UI — `.Result` trên UI vẫn xấu (block).

**Pitfall mix:**

```csharp
async Task LoadAsync()
{
    var data = await FetchAsync().ConfigureAwait(false);
    // có thể không còn UI thread
    label.Text = data; // WinForms: sai thread
}
```

Sau `false`, marshal chủ động: `Dispatcher.Invoke` / `IProgress<T>` (capture UI lúc tạo `Progress<T>`).

**Tóm lại:** trên .NET hiện đại (Core+), “bắt buộc ConfigureAwait everywhere trong app” **không còn** là quy tắc vàng như thời ASP.NET Framework; vẫn **nên** dùng trong **thư viện** tái sử dụng. Xem thêm [threading.md — SynchronizationContext](threading.md#9-synchronizationcontext--configureawait).

`await foreach` / `await using`: `ConfigureAwait(false)` trên enumerable/disposable (extension) — không quên stream/resource.

---

## 7. Exception trong async method

### 7.1 Khi `await` một Task

Trong async method:

```csharp
public async Task DoAsync()
{
    try
    {
        await SomeAsyncOperation();
    }
    catch (Exception ex)
    {
        // xử lý ở đây
    }
}
```

- Nếu `SomeAsyncOperation()` trả về `Task` faulted:
  - `await` sẽ ném ra **exception gốc** (không bọc `AggregateException` – trừ khi bạn gọi `.Result`/`.Wait()`).
- Nhiều exception trên một Task (hiếm, `WhenAll`): `await` ném **một** (thường đầu tiên); còn lại trong `task.Exception`.

```csharp
Task<int> all = Task.WhenAll(FailAsync("a"), FailAsync("b"));
try
{
    await all; // ném một exception, thường "a"
}
catch (Exception first)
{
    AggregateException agg = all.Exception!; // InnerExceptions có cả "a" và "b"
    _ = first;
    _ = agg.InnerExceptions;
}
```

Chi tiết unwrap vs `AggregateException`: [exceptions.md §10](exceptions.md#10-ngoại-lệ-trong-asyncawait--song-song).

### 7.2 Khi không `await` Task

```csharp
var task = DoAsync(); // fire-and-forget
```

- Nếu `DoAsync` ném exception sau đó:
  - Nếu không ai `await` hoặc inspect `task.Exception`,
  - Exception có thể bị nuốt, chỉ log ra `TaskScheduler.UnobservedTaskException` (tùy runtime).

Với `async void`:

- Exception “bật” lên `SynchronizationContext` → có thể crash ứng dụng UI nếu không handle.

**Kết luận:** Trừ event handler, tất cả async method nên trả `Task`/`Task<T>` và **luôn được await** hoặc theo dõi kết quả.

---

## 8. Cancellation & IProgress

### 8.1 CancellationToken

API async “đàng hoàng” thường nhận thêm `CancellationToken`:

```csharp
public async Task DoWorkAsync(CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();
    await Task.Delay(1000, cancellationToken);
    // ... các thao tác khác dùng token
}
```

Caller:

```csharp
var cts = new CancellationTokenSource();

var task = DoWorkAsync(cts.Token);
cts.Cancel(); // yêu cầu hủy

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Đã hủy");
}
```

**Semantics:** cancel là **hợp tác** — token không abort thread. API phải *quan sát* token (`Delay`, `ReadAsync`, `ThrowIfCancellationRequested` trong vòng CPU). Không truyền token = không hủy được giữa chừng (trừ khi API tự timeout).

`TaskCanceledException` : `OperationCanceledException` — bắt base type. Phân biệt cancel vs timeout vs fault: [exceptions.md §11](exceptions.md#11-cancellation-vs-exception).

### 8.2 Best practices CancellationToken

1. **Truyền token xuống mọi API hỗ trợ** (`HttpClient`, EF, `ReadAsync`, `Task.Delay`, Channel…).
2. **Không nuốt** `OperationCanceledException` trừ khi bạn cố ý chuyển thành kết quả “không lỗi” ở biên app.
3. Phân biệt: cancel theo yêu cầu user/host → `OperationCanceledException` / `TaskCanceledException`; lỗi thật → exception khác.
4. `CancellationTokenSource` — `using` / dispose đúng; cân nhắc `CancelAfter(TimeSpan)` cho timeout.
5. Liên kết nhiều nguồn: `CancellationTokenSource.CreateLinkedTokenSource(userCt, shutdownCt)`.
6. Kiểm tra hợp tác trong vòng lặp CPU: `ThrowIfCancellationRequested()` định kỳ (không chỉ dựa vào một `Delay`).
7. Sau khi cancel, **không** tiếp tục dùng resource nửa vời — cleanup trong `finally` / `await using`.
8. Default parameter `CancellationToken cancellationToken = default` ở public API async là convention tốt.
9. **Không** `Cancel()` rồi giả sử method dừng ngay — race với code giữa hai checkpoint.
10. `CancellationToken.None` / `default` = không bao giờ cancel; `Register(callback)` cho cleanup native.

```csharp
await using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(cts.Token, hostCt);
await DoWorkAsync(linked.Token);
```

Timeout: `CancelAfter` hủy **cả** token — cùng OCE như user cancel. Nếu cần phân biệt, CTS riêng + `catch (OperationCanceledException ex) when (timeoutCts.IsCancellationRequested)`.

### 8.3 Báo tiến độ với `IProgress<T>`

```csharp
public async Task DownloadWithProgressAsync(IProgress<int> progress)
{
    for (int i = 0; i <= 100; i += 10)
    {
        await Task.Delay(200);
        progress.Report(i);
    }
}
```

Caller (UI):

```csharp
var progress = new Progress<int>(percent =>
{
    progressBar.Value = percent; // chạy trên UI context
});

await DownloadWithProgressAsync(progress);
```

`new Progress<T>(handler)` capture `SynchronizationContext` **lúc tạo** — `Report` marshal về UI dù worker `ConfigureAwait(false)`.

---

## 9. Best practices khi dùng async/await

1. **Async all the way down**  
   - Đã “đi async” thì đi từ UI tới DAL.  
   - Tránh `.Result`, `.Wait()` vì dễ deadlock.

2. **Tránh `async void`** trừ event handler  
   - Dùng `Task`/`Task<T>` để caller có thể `await` & catch exception.

3. **Luôn `await` hoặc quản lý Task**  
   - Nếu không await, hãy ghi chú rõ ràng (fire-and-forget) và có logger.

4. **Không mix async với blocking sync**  
   - Tránh `Thread.Sleep`, I/O sync bên trong async.  
   - Dùng API async tương ứng (`ReadAsync`, `WriteAsync`, `SendAsync`, `SaveChangesAsync`…).

5. **Không dùng async trong property getter**  
   - Property nên “nhẹ” và trả về ngay.  
   - Dùng method `GetXxxAsync()` thay vì `async` property.

6. **Trong library: dùng `ConfigureAwait(false)`**  
   - App .NET Core+: thường không bắt buộc mọi chỗ; xem mục 6.

7. **Đặt tên method rõ ràng**  
   - Convention: method async đặt hậu tố `Async`: `GetUserAsync`, `SaveAsync`.

8. **Không `lock` quanh `await`**  
   - Giữ critical section ngắn; dùng `SemaphoreSlim.WaitAsync` nếu cần giới hạn đồng thời qua await.

9. **`ValueTask`:** một lần consume; API công khai ưu tiên `Task`.

10. **Cancellation:** token xuyên suốt; đừng nuốt OCE ở tầng giữa.

---

## 10. Async streams – `IAsyncEnumerable<T>` & `await foreach`

### 10.1 Vấn đề trước khi có async streams

Trước C# 8:

- Dùng `IEnumerable<T>` / `yield return` → **stream sync**, không `await` được bên trong.
- Dùng `Task<IEnumerable<T>>` → chờ lấy một sequence; sequence có thể lazy, nhưng các bước duyệt tiếp theo vẫn đồng bộ và không có `MoveNextAsync`.

Nhưng cần:

- Đọc từng dòng file từ server qua network,
- Nhận từng message từ socket,
- Stream log / sự kiện từ DB…

→ cần **stream async**, xử lý từng phần tử khi nó đến, không phải đợi tất cả.

### 10.2 `IAsyncEnumerable<T>` & `IAsyncEnumerator<T>`

```csharp
public interface IAsyncEnumerable<out T>
{
    IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken cancellationToken = default);
}

public interface IAsyncEnumerator<out T> : IAsyncDisposable
{
    ValueTask<bool> MoveNextAsync();
    T Current { get; }
}
```

Khác:

- `MoveNextAsync()` → `ValueTask<bool>`, phải `await` (tuân quy tắc ValueTask: mỗi lần `MoveNextAsync` một consume).
- Có `DisposeAsync()` cho cleanup async.
- `GetAsyncEnumerator(ct)` — token này hủy **việc duyệt**, không nhất thiết hủy nguồn nếu bạn không nối token.

**WHY không `IEnumerable<Task<T>>`:** pull model khác; `await foreach` + backpressure tự nhiên (producer chỉ chạy khi consumer `MoveNext`).

### 10.3 Async iterator method

Khai báo:

- `async IAsyncEnumerable<T>` +
- `yield return` bên trong +
- Có thể `await` trong thân.

Compiler sinh **một** state machine kết hợp async và iterator, triển khai các interface cần thiết. Nếu luôn materialize toàn bộ, hãy đo trước khi chọn stream thay cho `Task<List<T>>`.

Ví dụ:

```csharp
public async IAsyncEnumerable<int> CountAsync(int from, int to, int delayMs)
{
    for (int i = from; i <= to; i++)
    {
        await Task.Delay(delayMs);
        yield return i;
    }
}
```

**Pitfall:** không `yield` trong `catch` (giống iterator sync); `try/finally` được. Không buffer vô hạn trong iterator nếu consumer chậm — áp Channel bounded nếu cần tách tốc độ.

### 10.4 Duyệt bằng `await foreach`

```csharp
await foreach (var item in CountAsync(1, 5, 1000))
{
    Console.WriteLine(item);
}
```

Tương đương (giản lược):

```csharp
await using var e = CountAsync(1, 5, 1000).GetAsyncEnumerator();
while (await e.MoveNextAsync())
{
    var item = e.Current;
    Console.WriteLine(item);
}
```

`break` / exception → `DisposeAsync` enumerator (hủy iterator, chạy `finally` producer).

```csharp
await foreach (var x in source.ConfigureAwait(false))
{
    await HandleAsync(x).ConfigureAwait(false);
}
```

---

## 11. Async streams: Cancellation, exception, best practices

### 11.1 Cancellation

Có hai pattern phổ biến:

#### Pattern 1: `[EnumeratorCancellation]` + `WithCancellation`

```csharp
public async IAsyncEnumerable<int> CountAsync(
    int delayMs,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(delayMs, ct);
        yield return i;
    }
}
```

Caller:

```csharp
var cts = new CancellationTokenSource();

await foreach (var x in CountAsync(500, cts.Token)
                     .WithCancellation(cts.Token))
{
    Console.WriteLine(x);
    if (x >= 10) cts.Cancel();
}
```

**WHY attribute:** token trên `GetAsyncEnumerator` (từ `WithCancellation`) **không** tự thành tham số method. `[EnumeratorCancellation]` bảo compiler **hợp nhất** token tham số với token enumerator (linked). Thiếu attribute: `CountAsync(500, userCt)` nhận `userCt` nhưng `foreach.WithCancellation(other)` **không** chảy vào `ct` của iterator.

Nên: vừa tham số `ct` (gọi trực tiếp), vừa `[EnumeratorCancellation]` (foreach). Truyền cùng nguồn hoặc để default + `WithCancellation` ở call site.

#### Pattern 2: Truyền `CancellationToken` bình thường

```csharp
public async IAsyncEnumerable<int> CountAsync(
    int delayMs, CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(delayMs, ct);
        yield return i;
    }
}
```

Caller:

```csharp
var cts = new CancellationTokenSource();

await foreach (var x in CountAsync(500, cts.Token))
{
    Console.WriteLine(x);
    if (x >= 10) cts.Cancel();
}
```

Đủ khi **không** dùng `WithCancellation`. Library public: pattern 1 linh hoạt hơn.

### 11.2 Exception & dispose

Async iterator hỗ trợ xử lý ngoại lệ, nhưng `yield return` không được nằm trong `catch`, `finally` hoặc `try` có `catch`. Mẫu `try/finally` sau là hợp lệ:

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
{
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    try
    {
        while (await reader.ReadLineAsync(ct) is { } line)
            yield return line;
    }
    finally
    {
        Console.WriteLine("Done reading.");
    }
}
```

- Khi `await foreach` kết thúc (bình thường, exception hoặc cancel),
- `DisposeAsync()` được gọi, `finally` đảm bảo chạy.

Exception giữa `yield` → consumer bắt được trên `MoveNextAsync`/`await foreach`. Cancel → `OperationCanceledException` từ `Delay`/`ThrowIf…`.

### 11.3 So sánh & best practices

**`Task<IEnumerable<T>>` vs `IAsyncEnumerable<T>`**:

- `Task<IEnumerable<T>>` → chờ sequence, rồi duyệt đồng bộ; không bắt buộc sequence đã được materialize.
- `IAsyncEnumerable<T>` → xử lý từng phần tử khi chúng sẵn sàng.

Chọn async stream khi:

- Dữ liệu lớn / vô hạn,
- Muốn pipeline xử lý streaming (log, message, event…).

**Best practices với async streams:**

1. Dùng khi **từng phần tử** có ý nghĩa (log, stream dữ liệu).
2. Luôn hỗ trợ **cancellation** (token + `[EnumeratorCancellation]`).
3. Bọc `await foreach` trong `try/catch` nếu cần.
4. Đảm bảo cleanup: `using` / `await using` + `finally`.
5. Không dùng async stream khi thực chất bạn luôn cần “lấy hết rồi xử lý” → dùng `Task<List<T>>` là đủ.
6. Không lạm dụng trong logic thuần CPU sync.
7. Backpressure thật (producer nhanh): **Channel bounded** giữa iterator và worker — iterator `yield` không tự giới hạn queue.

**LINQ trên `IAsyncEnumerable`:** BCL (`System.Linq.AsyncEnumerable` / .NET 10) có `Where`/`Select`/`ToListAsync` async. Đừng `.ToEnumerable()` rồi LINQ sync — mất streaming. `ToListAsync` materialize hết — chỉ khi cần list.

```csharp
await foreach (var x in source.Where(static i => i > 0).Take(10))
    Console.WriteLine(x);
```

**Không** `GetAsyncEnumerator` rồi quên `DisposeAsync` — `await foreach` / `await using` bắt buộc. Gọi lại cùng method iterator là **lần chạy mới**, không phải phát lại bộ nhớ đệm:

```csharp
async IAsyncEnumerable<int> OnceAsync()
{
    Console.WriteLine("run");
    yield return 1;
}

IAsyncEnumerable<int> seq = OnceAsync();
await foreach (var _ in seq) { } // in "run"
await foreach (var _ in seq) { } // in "run" lần nữa
```

---

## 12. Channel + async

`System.Threading.Channels` là hàng đợi **async-native**: producer/consumer không chiếm thread khi chờ. So với `BlockingCollection` (block thread): [threading.md §7](threading.md#7-mẫu-producerconsumer).

```csharp
using System.Threading.Channels;

var channel = Channel.CreateBounded<WorkItem>(64);

async Task ProduceAsync(CancellationToken ct)
{
    try
    {
        await foreach (var item in source.WithCancellation(ct))
            await channel.Writer.WriteAsync(item, ct);
    }
    catch (Exception ex)
    {
        channel.Writer.TryComplete(ex);
        throw;
    }
    finally { channel.Writer.TryComplete(); }
}

async Task ConsumeAsync(CancellationToken ct)
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
        await ProcessAsync(item, ct);
}
```

**WHY Channel trong async:** `WriteAsync`/`ReadAsync` là awaitable — worker `await` khi đầy/rỗng, trả thread. `BlockingCollection.Add` **block** thread pool → cạn pool dưới tải.

- Backpressure: dùng **bounded** channel khi producer có thể nhanh hơn consumer.
- Kết hợp `IAsyncEnumerable` (nguồn) → Channel (fan-in/fan-out) → worker async.
- `Complete(exception)` để lan lỗi tới phía đọc (`ReadAsync` ném).
- `TryWrite`/`TryRead` cho hot path không await.
- Luôn `Complete()` — không Complete → `ReadAllAsync` treo.

```csharp
using var pipelineCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
async Task GuardAsync(Func<CancellationToken, Task> run)
{
    try { await run(pipelineCts.Token); }
    catch { pipelineCts.Cancel(); throw; }
}
await Task.WhenAll(GuardAsync(ProduceAsync), GuardAsync(ConsumeAsync));
```

Nhiều consumer cùng `Reader` cạnh tranh phần tử; không có bảo đảm fairness. Có thể fan-out qua N channel riêng. `SingleReader`/`SingleWriter = true` là cam kết chỉ có một bên đọc/ghi đồng thời. Hủy chung khi một bên lỗi giúp producer không treo trên channel đầy nếu consumer đã dừng.

**`FullMode` bounded:**

| Mode | Khi đầy |
|---|---|
| `Wait` | `WriteAsync` chờ (backpressure — mặc định nên dùng) |
| `DropWrite` | Bỏ item mới; `TryWrite` vẫn trả `true`, `WriteAsync` vẫn hoàn thành thành công |
| `DropOldest` | Bỏ item cũ nhất, nhận mới |
| `DropNewest` | Bỏ item mới nhất đã trong kênh |

Telemetry/log: `DropOldest` có thể chấp nhận. Thanh toán/lệnh: **Wait**, không drop.

```csharp
var log = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
{
    FullMode = BoundedChannelFullMode.DropOldest,
    SingleReader = true,
    SingleWriter = true
});

log.Writer.TryWrite(1);
log.Writer.TryWrite(2); // kênh đầy: bỏ 1, giữ 2. TryWrite trả true
// Wait (mặc định) thì WriteAsync thứ hai sẽ await, không bỏ 1
```

**Lỗi qua channel:** producer ở trên sở hữu việc hoàn tất writer, kể cả khi lỗi. `ReadAsync` trên channel đã hoàn tất có lỗi ném `ChannelClosedException` với lỗi gốc trong `InnerException`; `ReadAllAsync` truyền lỗi hoàn tất ra vòng `await foreach`. Với các mode drop, đăng ký callback `itemDropped` của `Channel.CreateBounded` nếu cần đếm dữ liệu bị bỏ.

`Complete()` hai lần → exception. `TryComplete` an toàn hơn shutdown đua.

**Không** `Write` sau `Complete`. Đọc hết + complete → `ReadAllAsync` kết thúc bình thường. Complete kèm exception → `await foreach` ném — `finally` consumer vẫn chạy.

Fan-in: nhiều producer, **một** `Complete` khi *tất cả* xong (`Task.WhenAll` producers rồi `Complete`). Complete sớm → producer khác `WriteAsync` fail.

---

## 13. `PeriodicTimer` vs `Task.Delay`

### `Task.Delay`

Một lần chờ / debounce / timeout đơn giản:

```csharp
await Task.Delay(TimeSpan.FromSeconds(1), ct);

// Timeout đua với thao tác:
var work = DoWorkAsync(ct);
var completed = await Task.WhenAny(work, Task.Delay(timeout, ct));
if (completed != work)
{
    ct.ThrowIfCancellationRequested(); // phân biệt hủy với hết giờ
    throw new TimeoutException(); // work vẫn chạy; cần quản lý completion của nó
}
await work;
```

`WhenAny` chỉ cho biết tác vụ nào xong trước. Hết giờ mà không `Cancel` token của `work` thì việc nền tiếp tục. Cách dừng việc: `CancelAfter` ở cuối mục này.

Tránh `Thread.Sleep` trong async method. Mỗi `Delay` tạo `Task` + timer — vòng `while + Delay` **mỗi vòng một Task**.

### Drift của `while + Delay`

```csharp
while (!ct.IsCancellationRequested)
{
    await PollAsync(ct);
    await Task.Delay(TimeSpan.FromSeconds(5), ct);
}
```

Nếu `PollAsync` mất 2s, chu kỳ thực tế ≈ 7s. Đặt Delay trước/sau đều **không** khóa wall-clock. Bù trừ thủ công (`Stopwatch`) dễ sai.

### `PeriodicTimer` (.NET 6+)

Vòng lặp định kỳ **bắt nhịp tick** thay vì “delay sau việc”:

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

while (await timer.WaitForNextTickAsync(ct))
{
    await PollAsync(ct);
}
```

**Semantics:** các tick xảy ra giữa hai lần chờ được **gộp thành một tick**, không xếp hàng từng tick để chạy bù. Chỉ một consumer được gọi `WaitForNextTickAsync` tại một thời điểm. `Dispose()` làm lần chờ trả `false`; hủy token chỉ hủy lần chờ đó, không tự dispose timer. [API PeriodicTimer](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer.waitfornexttickasync).

| | `Task.Delay` trong `while` | `PeriodicTimer` |
|---|---|---|
| Một lần chờ / debounce / timeout | **Phù hợp** | Không cần |
| Polling / heartbeat / host loop | Được nhưng dễ lệch chu kỳ + alloc Task mỗi vòng | **Đúng use case** |
| Hủy | Token trên `Delay` | Token trên `WaitForNextTickAsync` |
| Dispose | Không bắt buộc | `using` / `Dispose` timer |
| Overrun (work > period) | Tự “trượt” thêm Delay | Nhiều tick bị gộp thành một, không chạy bù tất cả |

Worker dài hạn: kết hợp `PeriodicTimer` + linked CTS từ `IHostApplicationLifetime` / shutdown token.

```csharp
public sealed class PollWorker(ILogger<PollWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try { await PollAsync(stoppingToken); }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }
            catch (Exception ex) { log.LogError(ex, "poll failed"); }
        }
    }
}
```

**Không** thay `PeriodicTimer` cho `Task.Delay` một lần (startup jitter, retry backoff — Delay/exponential rõ hơn). Không dùng `System.Threading.Timer` callback đồng bộ trong async (reentrancy) trừ khi wrap `TimeProvider` / `PeriodicTimer`.

**So sánh thêm:**

```csharp
// Retry backoff — Delay, không PeriodicTimer
TimeSpan delay = TimeSpan.FromMilliseconds(200);
for (int i = 0; i < 5; i++)
{
    if (await TryOnceAsync(ct)) return;
    await Task.Delay(delay, ct);
    delay *= 2;
}

// Debounce — Delay + CTS reset
CancellationTokenSource? debounce = null;
async Task OnKeyAsync()
{
    debounce?.Cancel();
    using var current = new CancellationTokenSource();
    debounce = current;
    try
    {
        await Task.Delay(300, current.Token);
        await SearchAsync(current.Token);
    }
    catch (OperationCanceledException) when (current.IsCancellationRequested) { }
    finally
    {
        if (ReferenceEquals(debounce, current)) debounce = null;
    }
}
```

`TimeProvider.System.CreateTimer` / `PeriodicTimer` testable (.NET 8 `TimeProvider`): inject clock trong unit test — `Task.Delay` thật làm test chậm. `IHostedService` production: `PeriodicTimer` + `stoppingToken`.

Mẫu debounce giả định lời gọi được tuần tự hóa trên UI context; nếu gọi từ nhiều thread cần đồng bộ field `debounce`. Cancellation chỉ có hiệu lực nếu `SearchAsync` tôn trọng token.

**Pitfall `WhenAny` + `Delay` timeout:** work vẫn chạy sau timeout. Yêu cầu hủy qua CTS nếu API hỗ trợ, và tiếp tục quan sát completion/lỗi để cleanup tài nguyên. Cancellation là hợp tác, không bảo đảm work dừng ngay.

```csharp
using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
timeoutCts.CancelAfter(timeout);
try
{
    await DoWorkAsync(timeoutCts.Token);
}
catch (OperationCanceledException ex) when (!ct.IsCancellationRequested && timeoutCts.IsCancellationRequested)
{
    throw new TimeoutException("Operation timed out.", ex);
}
```

Gọn hơn `WhenAny`+Delay: một token, work thật sự dừng (nếu API tôn trọng token).

---

## 14. Ghép tác vụ và giới hạn đồng thời

- **Task.WhenAll** chờ mọi task kết thúc, không tự hủy task còn lại khi một task lỗi. Task kết quả Faulted nếu có lỗi; nếu không lỗi nhưng có task bị hủy thì Canceled. `await` thường ném một lỗi, còn `all.Exception` giữ tập lỗi.
- **Task.WhenAny** trả task hoàn tất đầu tiên; cần `await` task thắng để nhận kết quả/lỗi. Các task còn lại vẫn chạy, phải được quan sát và cleanup.
- **Task.WaitAsync(timeout, ct)** (.NET 6+) giới hạn thời gian **chờ**, không tự hủy operation gốc. Token của operation cần được truyền riêng nếu muốn cancellation hợp tác.

```csharp
Task operation = DoWorkAsync(ct);
await operation.WaitAsync(TimeSpan.FromSeconds(5), ct);
// Nếu hết giờ, operation có thể vẫn chạy: owner vẫn phải quản lý completion.

await Parallel.ForEachAsync(items, new ParallelOptions
{
    MaxDegreeOfParallelism = 8,
    CancellationToken = ct
}, async (item, token) => await ProcessAsync(item, token));
```

Không tạo Task cho toàn bộ nguồn rất lớn rồi chỉ WhenAll: số operation đang mở có thể tăng không giới hạn. Dùng Parallel.ForEachAsync, SemaphoreSlim hoặc bounded Channel theo yêu cầu thứ tự và backpressure. Không chạy đồng thời nhiều query trên cùng DbContext.
