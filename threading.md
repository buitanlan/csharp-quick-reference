# Lập trình Thread

> **Baseline:** .NET **10** / C# **14**. `System.Threading.Lock` từ C# **13** / .NET **9+**. Chi tiết async/await → [async.md](async.md).

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
    - [4.2 `System.Threading.Lock` (C# 13 / .NET 9+)](#42-systemthreadinglock-c-13--net-9)
    - [4.3 `Interlocked` \& `Volatile`](#43-interlocked--volatile)
    - [4.4 `ManualResetEventSlim`/`AutoResetEvent`](#44-manualreseteventslimautoresetevent)
    - [4.5 `SemaphoreSlim`](#45-semaphoreslim)
    - [4.6 `ReaderWriterLockSlim`](#46-readerwriterlockslim)
    - [4.7 `SpinLock`/`SpinWait`](#47-spinlockspinwait)
  - [5. Biến theo thread: `ThreadStatic`, `ThreadLocal<T>`, `AsyncLocal<T>`](#5-biến-theo-thread-threadstatic-threadlocalt-asynclocalt)
  - [6. Cancellation: kiểu hợp tác](#6-cancellation-kiểu-hợp-tác)
  - [7. Mẫu Producer/Consumer](#7-mẫu-producerconsumer)
    - [7.1 `BlockingCollection<T>` (sử dụng Task để chạy Thread trong ThreadPool)](#71-blockingcollectiont-sử-dụng-task-để-chạy-thread-trong-threadpool)
    - [7.2 `System.Threading.Channels`](#72-systemthreadingchannels)
  - [8. Parallel.For / PLINQ (con trỏ)](#8-parallelfor--plinq-con-trỏ)
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

### 2.2 Background vs Foreground

```csharp
var bg = new Thread(() => Thread.Sleep(10_000)) { IsBackground = true };
bg.Start();
// Process có thể thoát dù bg chưa xong (background không giữ cho process sống).
```

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

### 4.2 `System.Threading.Lock` (C# 13 / .NET 9+)

Kiểu **dedicated lock** thay cho lock trên `object` tùy ý. Compiler nhận diện `lock (lockObj)` khi `lockObj` là `System.Threading.Lock` và sinh code dùng API tối ưu hơn `Monitor` trên object sync-block.

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
| Lạm dụng | Dễ lock nhầm `this`/type | Khó nhầm hơn |
| Runtime | Sync block / Monitor | Đường tối ưu hơn trên runtime mới |
| `Wait`/`Pulse` | Có trên `Monitor` | Không thay thế Wait/Pulse — dùng primitive khác |
| Yêu cầu | Mọi .NET | .NET **9+** (API); `lock` nhận diện từ **C# 13** |

**Khuyến nghị (baseline .NET 10):** field đồng bộ mới dùng `Lock`; code cũ `object` gate vẫn đúng — không bắt buộc rewrite hàng loạt.

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

Giới hạn **đồng thời N** tác vụ:

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

### 4.7 `SpinLock`/`SpinWait`

- **Spin** hữu ích khi lock **rất ngắn** và contention **thấp** (tránh context switch).  
- **Cẩn thận** starvation; thường `lock` / `Lock` / `SemaphoreSlim` đủ tốt.

---

## 5. Biến theo thread: `ThreadStatic`, `ThreadLocal<T>`, `AsyncLocal<T>`

```csharp
[ThreadStatic]
static int _counterPerThread; // mỗi thread có bản sao riêng

var local = new ThreadLocal<int>(() => 42);
Console.WriteLine(local.Value); // 42 (mỗi thread khởi tạo riêng)
```

- `ThreadStatic` **không** chạy field initializer per-thread (giá trị default của T).  
- `AsyncLocal<T>` lan truyền theo **async context** (không phải theo thread thuần).

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

---

## 7. Mẫu Producer/Consumer

### 7.1 `BlockingCollection<T>` (sử dụng Task để chạy Thread trong ThreadPool)

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

### 7.2 `System.Threading.Channels`

Hiệu năng cao, **async-friendly** (không block thread khi chờ). Phù hợp pipeline I/O ↔ CPU. Chi tiết async → [async.md](async.md).

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

So với `BlockingCollection`: Channel không chiếm thread khi chờ; ưu tiên cho code async hiện đại.

---

## 8. Parallel.For / PLINQ (con trỏ)

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

- Tránh I/O blocking bên trong `Parallel`/`AsParallel` (cạn thread pool).  
- Side-effect / shared mutable state → cần đồng bộ hoặc dùng local aggregate.  
- Đo trước: overhead partition có thể lớn hơn lợi ích với workload nhỏ.

---

## 9. `SynchronizationContext` & `ConfigureAwait`

- UI (WPF/WinForms) và một số host gắn **`SynchronizationContext`**: continuation sau `await` có thể được **marshal** về context đó.
- ASP.NET Core / console / worker hiện đại: thường **không** có SyncContext tùy biến → `ConfigureAwait(false)` ít thay đổi hành vi hơn so với .NET Framework + ASP.NET cũ.
- Chi tiết capture context, library vs app: xem [async.md — Capture context & ConfigureAwait](async.md#6-capture-context--configureawaitfalse).

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

---

## 11. Best practices & cảnh báo

- **Ưu tiên async I/O**; chỉ dùng thread cho CPU-bound hoặc API không async.  
- **Không** tạo quá nhiều threads — để thread pool điều phối (work‑stealing, hill‑climbing).  
- Tác vụ **blocking dài** → `TaskCreationOptions.LongRunning` hoặc **Thread** riêng.  
- **Đồng bộ tối thiểu**: `Interlocked` khi đủ; `Lock`/`lock` khi cần critical section; tránh lock khi đang `await`.  
- Dọn dẹp đúng: `CancellationToken`, `using` cho resource, `try/finally`.  
- **UI**: thread affinity — cập nhật qua `SynchronizationContext` / `Dispatcher` (mục 9).  
- **Đo đạc trước tối ưu**; stress test để phát hiện race/deadlock.  
- Pipeline I/O nặng: **async + Channels**; phần CPU: `Parallel.ForEach` / PLINQ.  
- Gate mới trên .NET 9+: ưu tiên `System.Threading.Lock` thay `object` tùy ý.
