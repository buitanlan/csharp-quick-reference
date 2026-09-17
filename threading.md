# Lập trình Thread

> **Baseline:** .NET **10** / C# **14**. `System.Threading.Lock` từ C# **13** / .NET **9+**. Chi tiết async/await → [async.md](async.md).

Thread là tài nguyên OS; .NET hiện đại ưu tiên **async I/O** và **pool**, chỉ tạo `Thread` khi cần kiểm soát thấp-level. Nhầm `Parallel` với async, nhầm `ThreadLocal` với `AsyncLocal`, nhầm `lock(object)` với `Lock` — pitfall chương này.

---

## Mục lục

- [Lập trình Thread](#lập-trình-thread)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan:](#1-tổng-quan)
  - [2. `Thread` cơ bản](#2-thread-cơ-bản)
    - [2.1 Tạo \& start thread](#21-tạo--start-thread)
    - [2.2 Background vs Foreground](#22-background-vs-foreground)
    - [2.3 Join/Interrupt/Sleep/Yield](#23-joininterruptsleepyield)
    - [2.4 Priority \& đặt tên](#24-priority--đặt-tên)
  - [3. Thread Pool](#3-thread-pool)
    - [3.1 Queue công việc](#31-queue-công-việc)
    - [3.2 `Task.Run` \& quan hệ với pool](#32-taskrun--quan-hệ-với-pool)
    - [3.3 Điều chỉnh min/max threads](#33-điều-chỉnh-minmax-threads)
  - [4. Đồng bộ hóa \& bộ công cụ](#4-đồng-bộ-hóa--bộ-công-cụ)
    - [4.1 `lock`/`Monitor`](#41-lockmonitor)
    - [4.2 `System.Threading.Lock` vs `Monitor` (C# 13 / .NET 9+)](#42-systemthreadinglock-vs-monitor-c-13--net-9)
    - [4.3 `Interlocked` \& `Volatile`](#43-interlocked--volatile)
    - [4.4 `ManualResetEventSlim`/`AutoResetEvent`](#44-manualreseteventslimautoresetevent)
    - [4.5 `SemaphoreSlim`](#45-semaphoreslim)
    - [4.6 `ReaderWriterLockSlim`](#46-readerwriterlockslim)
    - [4.7 `SpinLock`/`SpinWait`](#47-spinlockspinwait)
  - [5. `ThreadStatic` / `ThreadLocal<T>` vs `AsyncLocal<T>`](#5-threadstatic--threadlocalt-vs-asynclocalt)
  - [6. Cancellation: kiểu hợp tác](#6-cancellation-kiểu-hợp-tác)
  - [7. Mẫu Producer/Consumer](#7-mẫu-producerconsumer)
    - [7.1 `BlockingCollection<T>`](#71-blockingcollectiont)
    - [7.2 `System.Threading.Channels`](#72-systemthreadingchannels)
    - [7.3 Channels vs `BlockingCollection`](#73-channels-vs-blockingcollection)
  - [8. Parallel.For / PLINQ — khi nào *không* dùng](#8-parallelfor--plinq--khi-nào-không-dùng)
  - [9. `SynchronizationContext` \& `ConfigureAwait`](#9-synchronizationcontext--configureawait)
  - [10. Chẩn đoán \& đo đạc](#10-chẩn-đoán--đo-đạc)
  - [11. Best practices \& cảnh báo](#11-best-practices--cảnh-báo)

---

## 1. Tổng quan:

- **Thread**: luồng OS thực thi code CPU. Bạn có thể tạo thủ công với `new Thread(...)`.
- **Thread Pool**: nhóm luồng dùng chung. `Task.Run`, `ThreadPool.QueueUserWorkItem` **mượn** luồng tại đây để chạy.

> Trong .NET hiện đại, ưu tiên **`async/await`** và **Task/TPL**, chỉ quay về **Thread** khi cần kiểm soát thấp‑level hoặc tác vụ đặc thù.

---

## 2. `Thread` cơ bản

### 2.1 Tạo & start thread

```csharp
using System;
using System.Threading;

void Work(object? state)
{
    Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] Start: {state}");
    Thread.Sleep(500);
    Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] Done");
}

var t = new Thread(Work); // ParameterizedThreadStart (object?)
t.Start("job-1");
t.Join();
```

**C# hiện đại** (delegate/lambda mạnh mẽ):

```csharp
var t2 = new Thread(() =>
{
    Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] Heavy work...");
    Thread.Sleep(200);
});
t2.Start();
t2.Join();
```

Mỗi `new Thread` ≈ 1MB stack (thứ tự) + kernel object — đắt hơn pool. Capture closure: [delegates-lambdas.md §7](delegates-lambdas.md).

### 2.2 Background vs Foreground

```csharp
var bg = new Thread(() => Thread.Sleep(10_000)) { IsBackground = true };
bg.Start();
// Process có thể thoát dù bg chưa xong (background không giữ cho process sống).
```

Pool thread là background. Worker phải hoàn thành trước shutdown: `Join`, `IHost`, `CancellationToken`.

### 2.3 Join/Interrupt/Sleep/Yield

```csharp
var t = new Thread(() =>
{
    try
    {
        while (true) Thread.Sleep(1000); // chờ lâu
    }
    catch (ThreadInterruptedException) { Console.WriteLine("Interrupted"); }
});
t.Start();
Thread.Sleep(500);
t.Interrupt(); // đánh thức khỏi Sleep/Wait
t.Join();
```

`Abort` đã obsolete/loại khỏi .NET Core+. `Interrupt` chỉ khi thread ở wait — không phải cancel hợp tác. Ưu tiên `CancellationToken`.

### 2.4 Priority & đặt tên

```csharp
var t = new Thread(() => { /* ... */ })
{
    Name = "Worker#1",
    Priority = ThreadPriority.AboveNormal
};
t.Start();
```

> **Không** nên lạm dụng Priority. Lập lịch OS & thread pool đã tối ưu tốt.

---

## 3. Thread Pool

Thread Pool là một tập hợp các thread chạy sẵn để xử lý các nhiệm vụ, sử dụng thread pool thay vì tạo mới các **Thread**
giúp chúng ta kiểm soát được số lượng thread có trong hệ thống. Vì việc chuyển đổi giữa các thread tốn kém tài nguyên, do
vậy khi hệ thống càng có nhiều thread, thời gian dành cho việc chuyển đổi càng chiếm nhiều thời gian, thread pool giúp giữ 
việc dùng các thread được hiệu quả.

### 3.1 Queue công việc

```csharp
using System.Threading;

ThreadPool.QueueUserWorkItem(_ =>
{
    Console.WriteLine($"Work on pool thread {Thread.CurrentThread.ManagedThreadId}");
});
```

### 3.2 `Task.Run` & quan hệ với pool

```csharp
await Task.Run(() => CpuBound());
```

- `Task.Run` **mượn** worker thread từ pool.
- Tránh chạy tác vụ **blocking dài** trên pool (sẽ làm cạn pool) — dùng `TaskCreationOptions.LongRunning` để tách thread:

```csharp
var longTask = Task.Factory.StartNew(
    () => LongBlocking(), 
    CancellationToken.None,
    TaskCreationOptions.LongRunning, // gợi ý tạo dedicated thread
    TaskScheduler.Default);
```

`Task.Run` **không** làm I/O async nhanh hơn — chỉ chuyển CPU-bound khỏi request thread. Bọc sync I/O trong `Task.Run` trên ASP.NET: tốn 2 thread (request + pool) — sửa API async.

### 3.3 Điều chỉnh min/max threads

```csharp
ThreadPool.GetMinThreads(out var minW, out var minIO);
ThreadPool.SetMinThreads(workerThreads: Math.Max(minW, Environment.ProcessorCount*2), completionPortThreads: minIO);

// Xem số còn trống
ThreadPool.GetAvailableThreads(out var availW, out var availIO);
Console.WriteLine($"Available worker={availW} IOCP={availIO}");
```

> Nâng MinThreads có thể giảm độ trễ burst, nhưng **thận trọng** để tránh thừa luồng → context switch nhiều.

---

## 4. Đồng bộ hóa & bộ công cụ

> Tham khảo các bài học về đồng bộ hóa trong khóa học .NET nền tảng.

### 4.1 `lock`/`Monitor`

```csharp
private readonly object _gate = new();
private int _counter;

void Inc()
{
    lock (_gate)
    {
        _counter++;
    }
}
```

Tương đương (giản lược):

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(_gate, ref lockTaken);
    _counter++;
}
finally
{
    if (lockTaken) Monitor.Exit(_gate);
}
```

- `lock` trên `object` → `Monitor.Enter/Exit` (đảm bảo Exit khi có exception).
- **Không lock trên `this`, `typeof(T)`, hay string interned** (caller ngoài có thể tranh chấp / deadlock).
- `Monitor.Wait` / `Pulse` / `PulseAll` cho điều kiện trong critical section — dễ lỗi; thường ưu tiên `SemaphoreSlim`/`Channels`.
- `lock` **không** `await` được — compiler cấm `await` trong `lock`. Resume thread khác = deadlock / orphan lock.

`Monitor.TryEnter(o, timeout)` khi không muốn chờ vô hạn. Recursion: cùng thread `lock` lại cùng object → reentrant (Monitor). `Lock` (.NET 9) cũng reentrant theo tài liệu runtime — đừng dựa vào lock đệ quy làm thiết kế.

### 4.2 `System.Threading.Lock` vs `Monitor` (C# 13 / .NET 9+)

Kiểu **dedicated lock** thay cho lock trên `object` tùy ý. Compiler nhận diện `lock (lockObj)` khi `lockObj` là `System.Threading.Lock` và sinh code dùng API tối ưu hơn `Monitor` trên object sync-block.

**WHY:** mọi `object` có sync-block “ẩn” — dễ `lock(this)` / `lock(typeof(T))` / `lock("literal")` (intern). `Lock` là primitive **chỉ** để khóa; ý đồ rõ, runtime tối ưu (thin lock) trên .NET 9+.

```csharp
using System.Threading;

private readonly Lock _gate = new();
private int _counter;

void Inc()
{
    lock (_gate) // C# 13: hạ tầng Lock, không qua Monitor trên object thường
    {
        _counter++;
    }
}
```

API tường minh (hữu ích khi cần scope hẹp / try-enter):

```csharp
void TryInc()
{
    using (_gate.EnterScope())
    {
        _counter++;
    }
}

bool TryIncOnce()
{
    if (!_gate.TryEnter())
        return false;
    try
    {
        _counter++;
        return true;
    }
    finally
    {
        _gate.Exit();
    }
}
```

| | `lock (object)` + `Monitor` | `System.Threading.Lock` |
|---|---|---|
| Identity | Mọi `object` đều có thể làm gate | Kiểu chuyên dụng, rõ ý đồ |
| Lạm dụng | Dễ lock nhầm `this`/type/string | Khó nhầm hơn |
| Runtime | Sync block / Monitor | Đường tối ưu hơn trên runtime mới |
| `Wait`/`Pulse` | Có trên `Monitor` | **Không** thay Wait/Pulse — condition variable dùng primitive khác (`Monitor` trên object riêng, hoặc Channel) |
| `EnterScope` / `TryEnter` | `Monitor.TryEnter` | API first-class, `using` scope |
| Yêu cầu | Mọi .NET | .NET **9+** (API); `lock` nhận diện từ **C# 13** |
| `await` | Cấm trong `lock` | Vẫn **cấm** — không biến lock thành async mutex |

**Pitfall:** `lock ((object)_gate)` **boxing/cast** có thể đi đường Monitor trên object wrapper — đừng cast `Lock` về `object` để `Monitor.Enter`. Giữ kiểu `Lock`.

```csharp
object boxed = _gate;
lock (boxed) // KHÔNG phải Lock path — object khác / sai ý
{ }
```

Condition wait: giữ `object` riêng cho `Monitor.Wait` **hoặc** (khuyến nghị) không Wait/Pulse — producer/consumer = Channel.

**Khuyến nghị (baseline .NET 10):** field đồng bộ mới dùng `Lock`; code cũ `object` gate vẫn đúng — không bắt buộc rewrite hàng loạt. Statement-level: [statements.md §10](statements.md#10-đồng-bộ-hoá-lock-con-trỏ-systemthreadinglock).

**Deadlock cổ điển với `lock`/`Lock`:**

```csharp
void Transfer(Account a, Account b, int n)
{
    lock (a.Gate)
    lock (b.Gate) // nếu thread kia lock b rồi a → deadlock
    {
        a.Balance -= n;
        b.Balance += n;
    }
}
```

Sửa: thứ tự lock ổn định (`id` nhỏ trước), hoặc một lock toàn cục, hoặc Channel lệnh tuần tự. **Không** lock rồi gọi callback/user code (reentrancy / lock inversion).

`lock` giữ **ngắn**: copy dữ liệu ra, I/O/`await` **ngoài** lock. Giữ lock lúc `HttpClient.Send` = treo mọi thread khác cần cùng gate.

**So sánh nhanh mutex async:**

| Nhu cầu | Công cụ |
|---|---|
| Critical sync ngắn, không await | `Lock` / `lock` |
| Critical có `await` | `SemaphoreSlim(1)` + `WaitAsync` |
| N worker đồng thời | `SemaphoreSlim(N)` |
| Hàng đợi công việc | Channel |

`Mutex` OS (named) — cross-process; đắt, không dùng in-process thay `Lock`.

### 4.3 `Interlocked` & `Volatile`

```csharp
int x = 0;
Interlocked.Increment(ref x);
Interlocked.Add(ref x, 10);
var old = Interlocked.Exchange(ref x, 123);
```

`Volatile.Read/Write` đảm bảo **thứ tự nhìn thấy** giữa threads:

```csharp
using System.Threading;
volatile bool _done; // hoặc Volatile.Read/Write cho field thường

void Worker()
{
    while (!Volatile.Read(ref _done)) { /* spin */ }
}
void Stop() => Volatile.Write(ref _done, true);
```

Đủ cho flag/counter đơn; invariant nhiều field → `lock`/`Lock`.

### 4.4 `ManualResetEventSlim`/`AutoResetEvent`

```csharp
var evt = new ManualResetEventSlim(false);

new Thread(() =>
{
    Console.WriteLine("Init...");
    Thread.Sleep(500);
    evt.Set(); // mở cổng cho tất cả waiter
}).Start();

evt.Wait(); // chặn tới khi Set()
Console.WriteLine("Go!");
```

- `AutoResetEvent` đánh thức **một** waiter mỗi lần `Set()`.
- Bản `Slim` hiệu năng tốt cho in‑process; không cross-process.

### 4.5 `SemaphoreSlim`

Giới hạn **đồng thời N** tác vụ — **async-friendly** (`WaitAsync`):

```csharp
var sem = new SemaphoreSlim(3);
await sem.WaitAsync();
try
{
    await WorkAsync();
}
finally
{
    sem.Release();
}
```

Đây là mutex async khi `initialCount: 1` — thay `lock` khi critical section có `await`.

### 4.6 `ReaderWriterLockSlim`

Đọc song song nhiều, ghi độc quyền:

```csharp
var rw = new ReaderWriterLockSlim();
void Write(Action action)
{
    rw.EnterWriteLock();
    try { action(); } finally { rw.ExitWriteLock(); }
}
T Read<T>(Func<T> f)
{
    rw.EnterReadLock();
    try { return f(); } finally { rw.ExitReadLock(); }
}
```

Đừng `await` khi đang giữ RW lock. Contention ghi cao → `Lock` đơn giản hơn.

### 4.7 `SpinLock`/`SpinWait`

- **Spin** hữu ích khi lock **rất ngắn** và contention **thấp** (tránh context switch).
- **Cẩn thận** starvation; thường `lock` / `Lock` / `SemaphoreSlim` đủ tốt.

---

## 5. `ThreadStatic` / `ThreadLocal<T>` vs `AsyncLocal<T>`

Ba cơ chế “giá trị theo ngữ cảnh” — **không** thay thế nhau.

```csharp
[ThreadStatic]
static int _counterPerThread; // mỗi thread có bản sao riêng

var local = new ThreadLocal<int>(() => 42);
Console.WriteLine(local.Value); // 42 (mỗi thread khởi tạo riêng)
```

- `ThreadStatic` **không** chạy field initializer per-thread (giá trị default của T). Dùng `ThreadLocal<T>` khi cần factory.
- `AsyncLocal<T>` lan truyền theo **async execution context** (không phải theo thread thuần).

**WHY lệch sau `await`:** continuation có thể chạy **thread pool khác**. `ThreadLocal` / `[ThreadStatic]` **mất** giá trị — thread mới default. `AsyncLocal` **đi theo** luồng logic (copy-on-write ExecutionContext).

```csharp
var tl = new ThreadLocal<string>(() => "unset");
var al = new AsyncLocal<string>();

async Task DemoAsync()
{
    tl.Value = "thread-A";
    al.Value = "flow-1";
    Console.WriteLine($"{Thread.CurrentThread.ManagedThreadId} {tl.Value} {al.Value}");

    await Task.Delay(1).ConfigureAwait(false);

    // tl.Value có thể "unset" (thread khác)
    // al.Value vẫn "flow-1"
    Console.WriteLine($"{Thread.CurrentThread.ManagedThreadId} {tl.Value} {al.Value}");
}
```

| | `[ThreadStatic]` / `ThreadLocal<T>` | `AsyncLocal<T>` |
|---|---|---|
| Key | OS/managed **thread** | **ExecutionContext** (async flow) |
| Sau `await` / `Task.Run` | Giá trị **khác** (thread khác) | **Giữ** (flow xuống child async) |
| `Task.Run` | Worker không thấy giá trị caller | **Copy** context — thấy `AsyncLocal` (trừ `SuppressFlow`) |
| Use case | Affinity thread, native TLS, pooled thread reuse cẩn thận | Request id, `IHttpContextAccessor`-like, logging correlation |
| Pool reuse | **Phải clear** — thread pool tái sử dụng, giá trị cũ rò | Flow rõ; vẫn clear khi kết thúc request |
| Chi phí | Rẻ | Capture context trên `await` (đã có với async) |

**Pitfall `AsyncLocal` + `Task.Run`:** work nền **thừa hưởng** request culture/user — có thể leak identity. `using (ExecutionContext.SuppressFlow()) { Task.Run(...) }` khi fire-and-forget không muốn context.

**Pitfall `ThreadLocal` trên pool:** không `Dispose` + factory nặng → leak; `Value` còn sau job → request sau đọc nhầm. `try/finally { local.Value = default; }`.

`AsyncLocal` setter: thay đổi **sau** khi đã queue continuation không luôn lan ra sau — hiểu copy-on-write (con thấy snapshot lúc yield). Đừng dùng `AsyncLocal` như global mutable không document.

Ambient context ASP.NET Core: `IHttpContextAccessor` dựa ExecutionContext — cùng họ `AsyncLocal`, không `ThreadLocal`.

**ExecutionContext & `ConfigureAwait`:** `await` (mặc định) capture **ExecutionContext** (gồm `AsyncLocal`, security, culture) *kể cả* khi `ConfigureAwait(false)` — `false` chỉ bỏ **SynchronizationContext** / UI marshal, **không** xóa `AsyncLocal`. Muốn cắt hẳn: `ExecutionContext.SuppressFlow` hoặc không set `AsyncLocal` trước khi queue work nền.

```csharp
AsyncLocal<string> trace = new();
trace.Value = "req-1";

await Task.Yield();
Console.WriteLine(trace.Value); // "req-1" — EC chảy theo

using (ExecutionContext.SuppressFlow())
{
    _ = Task.Run(() => Console.WriteLine(trace.Value ?? "(none)"));
}
```

`ThreadLocal.Values` (mọi thread đã touch) — debug; production đừng iterate. `Dispose` `ThreadLocal` khi không dùng (unregister slot).

---

## 6. Cancellation: kiểu hợp tác

```csharp
var cts = new CancellationTokenSource();
var t = Task.Run(async () =>
{
    while (true)
    {
        cts.Token.ThrowIfCancellationRequested();
        await Task.Delay(100, cts.Token);
    }
}, cts.Token);

cts.Cancel();
try { await t; }
catch (OperationCanceledException) { Console.WriteLine("Canceled"); }
```

- Với `Thread`, không còn `Abort` trong .NET hiện đại (không an toàn). Hãy **hợp tác** qua `CancellationToken`/cờ tự quản.
- Token trên `Task.Run(..., token)` **không** abort body đang chạy — chỉ chuyển Task sang canceled nếu **chưa** bắt đầu, hoặc kết hợp với throw trong body.

---

## 7. Mẫu Producer/Consumer

### 7.1 `BlockingCollection<T>`

```csharp
using System.Collections.Concurrent;

var queue = new BlockingCollection<int>(boundedCapacity: 100);

// Producer
var prod = Task.Run(() =>
{
    for (int i = 0; i < 1000; i++) queue.Add(i);
    queue.CompleteAdding();
});

// Consumers (thread pool)
var consumers = Enumerable.Range(0, Environment.ProcessorCount).Select(_ => Task.Run(() =>
{
    foreach (var item in queue.GetConsumingEnumerable())
        Process(item);
})).ToArray();

await Task.WhenAll(consumers.Prepend(prod));
```

`Add`/`Take` **block thread** khi đầy/rỗng. Hợp code **sync** CPU worker. Trên ASP.NET async pipeline: block pool → tránh.

### 7.2 `System.Threading.Channels`

Hiệu năng cao, **async-friendly** (không block thread khi chờ). Phù hợp pipeline I/O ↔ CPU. Chi tiết async → [async.md §12](async.md#12-channel--async).

**Bounded** (backpressure — writer chờ khi đầy):

```csharp
using System.Threading.Channels;

var ch = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait, // hoặc DropWrite / DropOldest / …
    SingleWriter = false,
    SingleReader = false
});

_ = Task.Run(async () =>
{
    for (int i = 0; i < 1000; i++)
        await ch.Writer.WriteAsync(i);
    ch.Writer.Complete();
});

await foreach (var item in ch.Reader.ReadAllAsync())
    Process(item);
```

**Unbounded** (đơn giản hơn, cẩn thận OOM nếu producer nhanh hơn consumer):

```csharp
var open = Channel.CreateUnbounded<string>(new UnboundedChannelOptions
{
    SingleReader = true,
    SingleWriter = true // tối ưu khi đúng 1 reader/writer
});
```

**Pattern thường gặp:**

- Nhiều producer → một consumer (aggregate).
- Một producer → nhiều consumer: mỗi consumer `ReadAsync` vòng lặp (cạnh tranh trên cùng reader) hoặc fan-out qua nhiều channel.
- Luôn `Writer.Complete()` (hoặc `Complete(exception)`) khi hết dữ liệu; consumer thoát khỏi `ReadAllAsync`.
- Truyền `CancellationToken` vào `WriteAsync`/`ReadAsync`/`ReadAllAsync`.

### 7.3 Channels vs `BlockingCollection`

| | `BlockingCollection<T>` | `Channel<T>` |
|---|---|---|
| Chờ đầy/rỗng | **Block thread** (`Add`/`Take`) | **`await`** (`WriteAsync`/`ReadAsync`) |
| Async I/O pipeline | Dễ cạn thread pool | **Đúng chỗ** |
| CPU worker sync | Ổn (thread dành riêng / LongRunning) | Vẫn dùng được (`TryRead` vòng) nhưng API nghiêng async |
| Bounded backpressure | `boundedCapacity` | `CreateBounded` + `FullMode` |
| Complete | `CompleteAdding` | `Writer.Complete(ex?)` |
| Consume stream | `GetConsumingEnumerable` | `ReadAllAsync` → `IAsyncEnumerable` |
| .NET | Classic TPL / Concurrent | `System.Threading.Channels` (cũng dùng nội bộ Kestrel/pipeline) |

**Chọn Channel khi:** producer/consumer có `async`, ASP.NET/worker host, backpressure không được chiếm thread. **Chọn BlockingCollection khi:** toàn sync, đã có thread worker, code cũ Concurrent.

**Không** mix: `Task.Run` + `BlockingCollection.Take` trong hot path request. Không Channel unbounded làm “cứ Add” — OOM.

`ConcurrentQueue` **không** wait — spin/`Task.Delay` tự viết = tệ hơn Channel.

---

## 8. Parallel.For / PLINQ — khi nào *không* dùng

Cho **CPU-bound** trên nhiều core — không thay async I/O.

```csharp
using System.Threading.Tasks;

Parallel.For(0, items.Length, i => ProcessCpu(items[i]));

Parallel.ForEach(items, new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    CancellationToken = ct
}, item => ProcessCpu(item));
```

**PLINQ** (xem thêm [linq.md](linq.md)):

```csharp
var results = source
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .WithCancellation(ct)
    .Select(HeavyCompute)
    .ToArray();
```

**WHY Parallel:** chia vòng CPU (hash, encode, số) lên N core. TPL dùng pool — `MaxDegreeOfParallelism` tránh oversubscribe.

### Khi *không* dùng Parallel / PLINQ

1. **I/O-bound** (`HttpClient`, EF, file async) trong body — N worker **block** pool chờ mạng. Dùng `Task.WhenAll` / Channel + async, không `Parallel.For`.
2. **Workload nhỏ** — partition + delegate overhead > lợi ích (vài µs/item, N nhỏ). Đo `Stopwatch`.
3. **ASP.NET request** song song CPU nặng — cướp pool của request khác. Offload queue / giới hạn DOP thấp / isolate process.
4. **Thứ tự / side-effect** — `Parallel` không giữ order; mutate `List` chung không lock → race. PLINQ `AsOrdered()` trả giá.
5. **Lambda capture + shared mutable** — cần `lock` hoặc `localFinally` aggregate.
6. **Đã async sẵn** — `Parallel.ForEachAsync` (.NET 6) cho async body **có** DOP; vẫn không thay thế khi mỗi item là I/O nhẹ (dùng `Channel` + N consumer). `Parallel.For` **sync** + `.Result` bên trong = deadlock/starve.
7. **Single core / DOP 1** — không có lợi.
8. **Non-thread-safe API** (COM STA, một số native) — affinity, không song song.

```csharp
// SAI — I/O trong Parallel
Parallel.ForEach(urls, url => http.GetStringAsync(url).Result);

// ĐÚNG hướng I/O
await Parallel.ForEachAsync(urls, new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (url, token) =>
    {
        var s = await http.GetStringAsync(url, token);
        Process(s);
    });
```

`ForEachAsync` vẫn dùng pool; DOP quá cao + HTTP = socket/rate limit. I/O thuần: Channel + 2–8 worker thường ổn định hơn “DOP = 100”.

PLINQ `AsParallel().Select(async ...)` **không** await task — trả `Task` chưa chạy xong. Đừng mix LINQ async như vậy.

**`Parallel.For` vs `Task.WhenAll` vs `Parallel.ForEachAsync`:**

| | Dùng khi | Không dùng khi |
|---|---|---|
| `Parallel.For` / `ForEach` | CPU thuần, body **sync**, N lớn | I/O, `async`, ASP.NET không giới hạn |
| `Task.WhenAll(tasks)` | Tập task **đã** tạo (I/O fan-out vừa) | `tasks` = 100k `Task.Run` CPU (oversubscribe) |
| `Parallel.ForEachAsync` | Body async, cần **DOP** cứng | Mỗi item I/O + DOP = số item (DDoS chính mình) |

```csharp
// CPU: Parallel
Parallel.For(0, n, i => hashes[i] = SHA256.HashData(chunks[i]));

// I/O: WhenAll với throttle Channel/SemaphoreSlim — không Parallel.For
var sem = new SemaphoreSlim(8);
var jobs = urls.Select(async url =>
{
    await sem.WaitAsync(ct);
    try { return await http.GetStringAsync(url, ct); }
    finally { sem.Release(); }
});
string[] pages = await Task.WhenAll(jobs);
```

`Partitioner.Create` tùy chỉnh khi item lệch kích thước (một item 10s, còn lại 1ms) — default chunk có thể lệch load.

- Tránh I/O blocking bên trong `Parallel`/`AsParallel` (cạn thread pool).
- Side-effect / shared mutable state → cần đồng bộ hoặc dùng local aggregate.
- Đo trước: overhead partition có thể lớn hơn lợi ích với workload nhỏ.

---

## 9. `SynchronizationContext` & `ConfigureAwait`

- UI (WPF/WinForms) và một số host gắn **`SynchronizationContext`**: continuation sau `await` có thể được **marshal** về context đó.
- ASP.NET Core / console / worker hiện đại: thường **không** có SyncContext tùy biến → `ConfigureAwait(false)` ít thay đổi hành vi hơn so với .NET Framework + ASP.NET cũ.
- Chi tiết capture context, library vs app: xem [async.md — Capture context & ConfigureAwait](async.md#6-capture-context--configureawait).

```csharp
// Cập nhật UI: đảm bảo chạy trên UI thread
SynchronizationContext.Current?.Post(_ => label.Text = "done", null);
// WPF: Dispatcher.InvokeAsync(...); WinForms: Control.BeginInvoke(...)
```

---

## 10. Chẩn đoán & đo đạc

```csharp
ThreadPool.GetMaxThreads(out var maxW, out var maxIO);
ThreadPool.GetAvailableThreads(out var availW, out var availIO);
Console.WriteLine($"Pool: availW={availW}/{maxW}, availIO={availIO}/{maxIO}");
```

- Dùng **`Stopwatch`** đo thời gian; **PerfView/dotnet-trace** để phân tích contention/CPU.
- **`ConcurrentQueue`**/`Channels` có counters hữu ích (EventSource).
- Thread pool starve: available ≈ 0 + hàng đợi dài; giảm sync-over-async.

---

## 11. Best practices & cảnh báo

- **Ưu tiên async I/O**; chỉ dùng thread cho CPU-bound hoặc API không async.
- **Không** tạo quá nhiều threads — để thread pool điều phối (work‑stealing, hill‑climbing).
- Tác vụ **blocking dài** → `TaskCreationOptions.LongRunning` hoặc **Thread** riêng.
- **Đồng bộ tối thiểu**: `Interlocked` khi đủ; `Lock`/`lock` khi cần critical section; tránh lock khi đang `await`.
- Dọn dẹp đúng: `CancellationToken`, `using` cho resource, `try/finally`.
- **UI**: thread affinity — cập nhật qua `SynchronizationContext` / `Dispatcher` (mục 9).
- **Đo đạc trước tối ưu**; stress test để phát hiện race/deadlock.
- Pipeline I/O nặng: **async + Channels**; phần CPU: `Parallel.ForEach` / PLINQ **chỉ khi đo được lợi**.
- Gate mới trên .NET 9+: ưu tiên `System.Threading.Lock` thay `object` tùy ý.
- Correlation/request: `AsyncLocal`, không `ThreadLocal`.
- `Parallel` không phải “turbo” cho HTTP/EF.
