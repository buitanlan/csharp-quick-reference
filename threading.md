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
      - [3.2.1 Overload và option cố định](#321-overload-và-option-cố-định)
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

Mọi thread trong một process **nhìn chung heap**. Hai thread cùng `x++` không có hàng rào thì mất cập nhật (đọc–cộng–ghi không nguyên tử) và có thể không thấy ghi của nhau (sắp xếp lại lệnh / cache). Công cụ ở mục 4 tồn tại để đóng hai lỗ đó: **loại trừ** (một thread trong critical section) và **thứ tự nhìn thấy** (ghi bên này thành hiện bên kia).

> Trong .NET hiện đại, ưu tiên **`async/await`** và **Task/TPL**, chỉ quay về **Thread** khi cần kiểm soát thấp‑level hoặc tác vụ đặc thù. Async I/O **không chiếm thread** lúc chờ mạng; `new Thread` và `Task.Run` thì chiếm.

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

`Start` chỉ được gọi **một lần**. `ThreadStart` không tham số; `ParameterizedThreadStart` nhận `object?` — kiểu không an toàn, lambda capture rõ hơn.

Exception **không** bắt trong body của `new Thread` là unhandled: process **chết** (sau `AppDomain.UnhandledException`). Khác `Task.Run`: exception nằm trên `Task` cho đến khi `await` / `.Exception`; không ai quan sát thì mặc định **không** hạ process (.NET 4.5+).

```csharp
var t = new Thread(() => throw new InvalidOperationException("boom"));
t.Start();
t.Join(); // process đã được lên lịch terminate — đừng dựa vào Join để “nuốt” lỗi
```

### 2.2 Background vs Foreground

`new Thread` mặc định **foreground** (`IsBackground == false`): `Main` return rồi process **vẫn sống** đến khi mọi foreground thread kết thúc. Background không giữ process; lúc thoát, thread đó bị cắt — `finally` có thể không chạy.

```csharp
var fg = new Thread(() => Thread.Sleep(10_000)); // foreground: process đứng ~10s sau khi Main return
fg.Start();

var bg = new Thread(() => Thread.Sleep(10_000)) { IsBackground = true };
bg.Start();
// Process có thể thoát dù bg chưa xong.
```

Pool thread là background. Worker phải hoàn thành trước shutdown: `Join`, `IHost`, `CancellationToken`. `Join` một foreground từ `Main` là cách sync console đợi xong; service thì hủy token lúc `Stopping`, đừng dựa vào process kill để dọn tài nguyên.

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

`Abort` đã obsolete/loại khỏi .NET Core+. `Interrupt` chỉ ném `ThreadInterruptedException` khi thread đang **Sleep / Join / Wait** — đang chạy CPU thì interrupt bị ghi nhớ đến lần wait sau. Không phải cancel hợp tác. Ưu tiên `CancellationToken`.

Ba cách “nhường CPU”, không cái nào là lock:

| | Việc thực sự | Khi nào |
|---|---|---|
| `Thread.Sleep(ms)` | **Block** thread ít nhất ~`ms` (độ phân giải timer, thường 15ms trừ khi timeBeginPeriod / timer hiện đại) | Backoff thô, test. **Không** `Sleep` trên pool để chờ việc |
| `Thread.Sleep(0)` | Nhường phần timeslice còn lại cho thread **cùng priority** đang sẵn sàng | Hiếm khi đúng công cụ |
| `Thread.Yield()` | Gợi ý OS chạy thread **đang ready trên cùng CPU**; không có thì trả `false` ngay | Spin ngắn, trước khi `SpinWait` |

```csharp
bool finished = t.Join(TimeSpan.FromSeconds(2)); // false = hết giờ, thread vẫn chạy
if (!finished)
    Console.WriteLine("still working"); // không có Abort để cắt — cần token trong body
```

`Sleep` trên thread pool = một worker biến mất trong lúc chờ. Muốn chờ đồng hồ: `Task.Delay` (async) hoặc `PeriodicTimer`.

### 2.4 Priority & đặt tên

```csharp
var t = new Thread(() => { /* ... */ })
{
    Name = "Worker#1",
    Priority = ThreadPriority.AboveNormal
};
t.Start();
```

> `Priority` là gợi ý cho scheduler, không phải hạn ngạch CPU. Đặt `Name` trước `Start` giúp debugger/dump có tên ngay. Trên .NET hiện đại có thể đổi tên; giới hạn chỉ gán một lần thuộc .NET Framework. Tránh đặt tên pool thread theo từng request vì cùng thread được tái dùng; dùng logging scope/correlation ID.

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

Callback **không** trả `Task`. Exception trong callback là unhandled → process chết, giống `new Thread`. Không có `await` để bắt. Việc cần lỗi, hủy, kết quả → `Task.Run`.

`QueueUserWorkItem` **chảy** `ExecutionContext` (`AsyncLocal`, culture). `UnsafeQueueUserWorkItem` bỏ bước đó — rẻ hơn, và **mất** correlation đang set ở caller:

```csharp
var trace = new AsyncLocal<string> { Value = "req-9" };

ThreadPool.QueueUserWorkItem(_ =>
    Console.WriteLine(trace.Value)); // "req-9"

ThreadPool.UnsafeQueueUserWorkItem(
    _ => Console.WriteLine(trace.Value ?? "(none)"),
    state: 0,
    preferLocal: false);
```

Hàng đợi đầy / mọi worker đang block: item **nằm chờ**. Pool có thể tiêm thêm worker (mục 3.3) nhưng không ngay một thread mỗi item — đó là độ trễ, không phải mất item.

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
    TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default);
```

`LongRunning` **không** có trên `Task.Run`. `StartNew` thiếu `DenyChildAttach` và `TaskScheduler.Default` thì không còn là “`Task.Run` + thread riêng” — xem §3.2.1.

#### 3.2.1 Overload và option cố định

`Task.Run` **không** có tham số `TaskCreationOptions`. Chỉ có hình dạng delegate và `CancellationToken` tùy chọn:

| Overload | Delegate chạy trên pool | `Task` trả về |
|---|---|---|
| `Run(Action)` | `void` đồng bộ | hoàn thành khi delegate return |
| `Run(Func<T>)` | sync, có kết quả | `Task<T>` |
| `Run(Func<Task>)` | lambda `async` / method trả `Task` | **unwrap** — một `Task`, không phải `Task<Task>` |
| `Run(Func<Task<T>>)` | async có kết quả | `Task<T>`, không phải `Task<Task<T>>` |

Mỗi dòng có bản kèm `CancellationToken`. Token **chỉ** được xét lúc task **bắt đầu**:

- Đã hủy trước khi delegate chạy → delegate **không** chạy, task ở trạng thái Canceled.
- Hủy sau khi delegate đã chạy không tự dừng body. Với delegate đồng bộ, task nhận cancellation khi body ném `OperationCanceledException` gắn token đã hủy và trùng token của task; trường hợp khác thường Faulted. Với `Func<Task>`, Task.Run unwrap trạng thái task bên trong: async method ném OCE tạo task Canceled ngay cả khi token khác token truyền cho Task.Run.
- Token **không** được đưa vào `Action` sync. Body sync muốn hợp tác thì phải capture token từ ngoài (closure), không phải từ tham số `Run`.

```csharp
var cts = new CancellationTokenSource();
cts.Cancel();
await Task.Run(() => Console.WriteLine("no"), cts.Token); // không in; await ném TaskCanceledException
```

Bên trong, `Task.Run` luôn là:

```csharp
Task.Factory.StartNew(
    action,
    CancellationToken.None,
    TaskCreationOptions.DenyChildAttach, // cố định — không tắt được
    TaskScheduler.Default);              // cố định — luôn pool, bỏ qua scheduler hiện tại
// bản Func<Task> thêm .Unwrap()
```

Hai option đó **không** chọn lại. Muốn cờ khác thì `StartNew`, và phải **tự** giữ hai cái `Task.Run` đã gắn — nếu không thì hành vi đổi im lặng.

| `TaskCreationOptions` | `Task.Run` | Ý nghĩa khi dùng với `StartNew` |
|---|---|---|
| `DenyChildAttach` | **luôn bật** | Task con tạo bằng `AttachedToParent` **không** gắn vào task này. Parent không đợi con |
| *(scheduler)* `Default` | **luôn** | Xếp hàng thread pool. `StartNew` bỏ scheduler thì dùng `TaskScheduler.Current`: đang **bên trong** task chạy trên scheduler UI thì việc mới cũng vào UI. Đứng trên thread UI mà không nằm trong task thì `Current` vẫn thường là `Default` |
| `LongRunning` | không có | Gợi ý thread riêng, khỏi pool. Vẫn chỉ là gợi ý; đừng dùng cho việc ngắn |
| `PreferFairness` | không có | Xếp hàng global (FIFO) thay vì local queue LIFO của worker. Ít cần |
| `AttachedToParent` | không có — bên trong `Task.Run` cờ này trên task con bị bỏ qua | Con gắn parent: parent không hoàn thành cho đến khi con xong. Con đợi parent → deadlock |
| `HideScheduler` | không có | Trong body, `TaskScheduler.Current` thành `Default` dù task đang chạy trên scheduler khác |
| `RunContinuationsAsynchronously` | không có | Continuation không chạy inline trên thread vừa hoàn thành task. Tránh đệ quy / giữ thread lạ |
| `None` | — | `StartNew` mặc định khi bỏ tham số options |

```csharp
// Đang chạy trong task của scheduler UI: StartNew không chỉ scheduler → quay lại UI
Task.Factory.StartNew(() => cpu());

// Task.Run: luôn pool, kể cả khi gọi từ task UI
await Task.Run(() => cpu());

var cts = new CancellationTokenSource();
cts.Cancel();
await Task.Run(() => Console.WriteLine("no"), cts.Token); // không in; await ném TaskCanceledException

// “Run + LongRunning” — phải lặp lại hai option cố định
Task.Factory.StartNew(
    () => LongBlocking(),
    CancellationToken.None,
    TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default);
```

**Pitfall async không unwrap:** `Task.Run(() => SomeAsync())` khi `SomeAsync` trả `Task` thì `Run(Func<Task>)` unwrap — `await` đợi cả phía trong. `Task.Run(() => { _ = SomeAsync(); })` là `Action`: task ngoài xong ngay khi gọi, lỗi phía trong không ai đợi.

`Task.Run` không làm I/O async nhanh hơn. Bọc sync I/O trong Task.Run vẫn chiếm một pool worker suốt thời gian chờ; nếu caller await thì caller được giải phóng, không phải luôn có hai thread bị giữ. Ưu tiên API I/O async thực sự.

`Task.Run` có overload `Func<Task>`: lambda `async` được **unwrap** thành một `Task`, không phải `Task<Task>`. Exception trong `await` bên trong vẫn nằm trên `Task` trả về.

```csharp
Task job = Task.Run(async () =>
{
    await Task.Delay(10);
    throw new InvalidOperationException("observed");
});
// await job → ném InvalidOperationException, process không chết chỉ vì lỗi này
```

Starve pool: N worker cùng `Thread.Sleep` / `.Result` / `lock` chờ nhau, trong khi pool chưa kịp tạo thêm thread. Triệu chứng: request mới xếp hàng dù CPU thấp. Sửa việc block, đừng chỉ tăng `SetMaxThreads`.

### 3.3 Điều chỉnh min/max threads

```csharp
ThreadPool.GetMinThreads(out var minW, out var minIO);
ThreadPool.SetMinThreads(workerThreads: Math.Max(minW, Environment.ProcessorCount*2), completionPortThreads: minIO);

// Xem số còn trống
ThreadPool.GetAvailableThreads(out var availW, out var availIO);
Console.WriteLine($"Available worker={availW} IOCP={availIO}");
```

> Nâng MinThreads có thể giảm độ trễ burst, nhưng **thận trọng** để tránh thừa luồng → context switch nhiều.

Hai số **không** cùng một việc:

| | Worker | Completion port (IOCP) |
|---|---|---|
| Chạy | `Task.Run`, `QueueUserWorkItem`, `Parallel`, tiếp tục sau `await` khi không có sync context | Callback hoàn thành I/O (Windows IOCP). Lúc `await` socket, **không** có thread nào đứng chờ |
| Đừng | Block dài (Sleep, `.Result`, lock chờ I/O) | Block trong callback hoàn thành — pool I/O cũng cạn |

Pool điều chỉnh số worker theo throughput (hill climbing) và phát hiện blocking/starvation. `SetMinThreads` **không tạo sẵn hoặc giữ luôn đủ số thread**: nó nâng ngưỡng tạo worker theo nhu cầu trước khi áp dụng cơ chế điều tiết thông thường. Min quá cao có thể gây tạo dư thread khi tải tăng, tăng context switch và bộ nhớ. Chỉ đổi khi đã đo được vấn đề (§10).

`SetMaxThreads` thấp hơn số việc block đồng thời → hàng đợi vô hạn, không thêm được worker. Hầu như không cần đụng max.

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

Ba target cấm trong thực tế — ai cũng `lock` được cùng object, kể cả code không thuộc class của bạn:

```csharp
lock (this) { }                  // caller ngoài: lock (yourInstance)
lock (typeof(Counter)) { }       // mọi code trong AppDomain lock được type object
lock ("config") { }              // literal intern — "config" chỗ khác là CÙNG instance
```

Field `private readonly object _gate = new();` chỉ class bạn giữ được tham chiếu.

`TryEnter` khi chờ vô hạn là sai (UI, request):

```csharp
bool taken = false;
try
{
    Monitor.TryEnter(_gate, TimeSpan.FromMilliseconds(50), ref taken);
    if (!taken) return; // bận — bỏ hoặc thử lại phía trên, đừng spin
    _counter++;
}
finally
{
    if (taken) Monitor.Exit(_gate);
}
```

`Wait`/`Pulse` là condition variable trên **cùng** object đang `lock`. `Wait` **thả** lock rồi ngủ; khi được `Pulse` nó lấy lại lock trước khi trả về. Điều kiện phải kiểm trong `while` (spurious wakeup, và Pulse có thể xảy ra trước khi Wait).

```csharp
private readonly object _gate = new();
private readonly Queue<int> _q = new();

void Produce(int n)
{
    lock (_gate)
    {
        _q.Enqueue(n);
        Monitor.Pulse(_gate); // đánh thức một waiter; PulseAll nếu nhiều điều kiện
    }
}

int Consume()
{
    lock (_gate)
    {
        while (_q.Count == 0)
            Monitor.Wait(_gate);
        return _q.Dequeue();
    }
}
```

Quên `Pulse`, `Pulse` **ngoài** lock, hoặc `if` thay `while` → treo hoặc lấy queue rỗng. Producer/consumer mới: Channel (mục 7), không viết Wait/Pulse.

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

**Pitfall:** `System.Threading.Lock` là reference type, cast sang `object` không boxing. Tuy nhiên `lock ((object)_gate)` dùng Monitor, còn `lock (_gate)` dùng Lock: **hai cơ chế khóa độc lập trên cùng object**, không bảo vệ lẫn nhau. Giữ kiểu Lock nhất quán.

```csharp
object boxed = _gate;
lock (boxed) // KHÔNG phải Lock path — object khác / sai ý
{ }
```

Condition wait: giữ `object` riêng cho `Monitor.Wait` **hoặc** (khuyến nghị) không Wait/Pulse — producer/consumer = Channel.

**Khuyến nghị (baseline .NET 10):** field đồng bộ mới dùng `Lock`; code cũ `object` gate vẫn đúng — không bắt buộc rewrite hàng loạt. Statement-level: [statements.md §10](statements.md#10-đồng-bộ-hoá-lock-với-systemthreadinglock).

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

X++ không nguyên tử: hai thread cùng đọc 5 rồi cùng ghi 6, mất một lần tăng. Interlocked.Increment bảo đảm atomic read-modify-write; cách phát lệnh tùy JIT/architecture. Phù hợp một biến đếm, không biến nhiều cập nhật thành một giao dịch.

`CompareExchange` là CAS: ghi `next` chỉ khi ô nhớ vẫn bằng `expected`. Đây là vòng lock-free khi “tăng” không có API sẵn (hoặc cập nhật có điều kiện):

```csharp
static void Add(ref int target, int delta)
{
    int seen;
    do
    {
        seen = Volatile.Read(ref target);
    }
    while (Interlocked.CompareExchange(ref target, seen + delta, seen) != seen);
}

int published = 0;
bool won = Interlocked.CompareExchange(ref published, 1, 0) == 0; // đúng một thread thắng
```

Với `++`, gọi `Increment` — đừng tự viết CAS. CAS thất bại thì **thử lại**; giữa hai lần đọc, logic nặng trong vòng = spin nóng.

`Volatile.Read/Write` (và modifier `volatile`) đảm bảo **thứ tự nhìn thấy** của **một** field giữa threads — không ghép hai field thành một giao dịch, và **không** làm `x++` nguyên tử:

```csharp
using System.Threading;
bool _done; // field thường, mọi truy cập qua Volatile.Read/Write

void Worker()
{
    while (!Volatile.Read(ref _done)) { /* spin — xem SpinWait mục 4.7 */ }
}
void Stop() => Volatile.Write(ref _done, true);
```

Không `volatile` / `Volatile.Write`: compiler hoặc CPU có thể giữ `_done` trong thanh ghi, worker không bao giờ thấy `true`. `volatile bool _done; _done = true` là một ghi có hàng rào; `if (_done) _flag = 1` vẫn là hai ghi riêng — reader có thể thấy `_flag` cũ dù đã thấy `_done`. Nhiều field phải đúng cùng lúc → `lock` / `Lock`.

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

`ManualResetEventSlim` là **cổng**: `Set` mở, mọi `Wait` đang chờ và mọi `Wait` sau đó đều qua, cho đến `Reset`. `AutoResetEvent` là **vé**: `Set` chỉ cho **đúng một** `Wait` qua rồi tự đóng. `Set` khi chưa có ai chờ thì vé được giữ — `Wait` kế tiếp không block.

```csharp
var gate = new ManualResetEventSlim(false);
var ticket = new AutoResetEvent(false);

gate.Set();
gate.Wait();
gate.Wait();   // cả hai qua — cổng vẫn mở
gate.Reset();  // đóng lại; Wait sau đó block

ticket.Set();
ticket.WaitOne(); // lấy vé, event về unsignaled
// ticket.WaitOne(); // block — cần Set nữa

using (gate) { }
ticket.Dispose();
```

Hai worker + `AutoResetEvent`: mỗi `Set` từ producer đánh thức **một** consumer. Dùng `Manual` cho “init xong, mọi người chạy”. Dùng `Auto` cho “có một đơn vị việc”. Nhiều vé cùng lúc, hoặc `await`: `SemaphoreSlim`, không xếp nhiều `AutoResetEvent`.

`ManualResetEvent` / `AutoResetEvent` là WaitHandle dựa trên OS. Muốn event có tên để chia sẻ process trên Windows, tạo `EventWaitHandle` với `EventResetMode` và `name`; hai class trên không có constructor nhận tên. `WaitAny` / `WaitAll` nhận WaitHandle, không nhận trực tiếp `ManualResetEventSlim`.

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

`SemaphoreSlim(initial, max)` — `initial` là số vé đang có, `max` là trần. `Release` đẩy count vượt `max` → `SemaphoreFullException`. Không nhớ thread nào đã `Wait`: `Release` thừa từ thread khác cũng tăng count (khác `lock`, vốn gắn với thread giữ).

```csharp
var slots = new SemaphoreSlim(initialCount: 3, maxCount: 3);

async Task WorkAsync(int id, CancellationToken ct)
{
    await slots.WaitAsync(ct); // vé thứ 4 chờ cho đến khi có Release
    try
    {
        await Task.Delay(100, ct);
    }
    finally
    {
        slots.Release();
    }
}

async Task TryWorkAsync(CancellationToken ct)
{
    if (!await slots.WaitAsync(TimeSpan.Zero, ct))
        return; // không lấy được vé — không Release
    try
    {
        await Task.Delay(10, ct);
    }
    finally
    {
        slots.Release();
    }
}
```

`WaitAsync(0)` / `TimeSpan.Zero` là try-enter. Quên `Release` (exception trước `finally`) = vé mất vĩnh viễn, mọi waiter đứng. `using` không có sẵn — `try/finally` là hợp đồng. `Semaphore` (kernel, không Slim) cross-process và **không** có `WaitAsync`.

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

Cùng thread giữ read lock rồi xin write lock ném `LockRecursionException`, kể cả `SupportsRecursion`. Đường nâng cấp đúng là **upgradeable read**: chỉ một upgradeable reader tại một thời điểm, vẫn chạy song song với reader thường.

```csharp
void UpdateIfStale()
{
    rw.EnterUpgradeableReadLock();
    try
    {
        if (!IsStale) return;
        rw.EnterWriteLock();
        try { Rebuild(); }
        finally { rw.ExitWriteLock(); }
    }
    finally { rw.ExitUpgradeableReadLock(); }
}
```

Read nhiều, ghi hiếm, và cần “xem rồi mới quyết định ghi” → `ReaderWriterLockSlim`. Chỉ ghi hoặc critical section ngắn → `Lock` ít đường deadlock hơn. Không có bản async: đừng `await` giữa Enter và Exit.

### 4.7 `SpinLock`/`SpinWait`

**Spin** = thread không ngủ, chiếm CPU đến khi điều kiện đúng. Hữu ích khi critical section **vài chục lệnh** và contention **thấp** — rẻ hơn context switch của `lock`. Section dài, contention cao, hoặc máy 1 core: spin làm chậm chính thread đang giữ lock (cùng core). Mặc định vẫn `lock` / `Lock` / `SemaphoreSlim`.

`SpinLock` là **struct**. Copy ra biến khác = hai khóa không liên quan. Giữ một field, đừng truyền theo giá trị. `enableThreadOwnerTracking: false` trên hot path (nhanh hơn, không phát hiện enter đệ quy — đệ quy sẽ spin mãi). `true` (mặc định) ném nếu cùng thread enter lần hai.

```csharp
private SpinLock _spin = new(enableThreadOwnerTracking: false);
private int _counter;

void Inc()
{
    bool taken = false;
    try
    {
        _spin.Enter(ref taken);
        _counter++;
    }
    finally
    {
        if (taken) _spin.Exit();
    }
}
```

Quên `Exit` = mọi thread khác spin đến chết CPU. `Enter` có timeout: `TryEnter(TimeSpan, ref taken)`.

`SpinWait` là backoff khi **chờ một flag**, không phải khóa nhiều lệnh. `SpinOnce` bắt đầu bằng pause CPU, sau đó `Yield` / `Sleep` — không đốt một core mãi như `while (!flag) ;`.

```csharp
private volatile bool _ready;

void WaitUntilReady()
{
    var wait = new SpinWait();
    while (!_ready)
        wait.SpinOnce();
}

// tương đương có hạn giờ:
bool ok = SpinWait.SpinUntil(() => _ready, TimeSpan.FromMilliseconds(50));
```

`SpinUntil` hết giờ trả `false`, không ném. Flag phải `volatile` / `Volatile` (mục 4.3) nếu không người chờ có thể không thấy ghi. Đây không thay `await` — thread vẫn bận. Chờ I/O hoặc việc dài: `ManualResetEventSlim` / `Task`.

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

AsyncLocal truyền giá trị xuống child flow; gán trong child không tự truyền ngược về caller sau await. Nếu giá trị là reference tới object mutable, child/caller vẫn có thể cùng sửa object đó. Không coi copy-on-write ExecutionContext là deep copy dữ liệu.

Ambient context ASP.NET Core: `IHttpContextAccessor` dựa ExecutionContext — cùng họ `AsyncLocal`, không `ThreadLocal`.

**ExecutionContext & `ConfigureAwait`:** `await` (mặc định) capture **ExecutionContext** (gồm `AsyncLocal`, security, culture) *kể cả* khi `ConfigureAwait(false)` — `false` chỉ bỏ **SynchronizationContext** / UI marshal, **không** xóa `AsyncLocal`. Muốn cắt hẳn: `ExecutionContext.SuppressFlow` hoặc không set `AsyncLocal` trước khi queue work nền.

Không await **bên trong** scope SuppressFlow: việc Undo/Dispose phải diễn ra trên thread đã suppress. Queue work trong scope rồi await Task ở ngoài; vẫn quan sát lỗi của task nền.

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

`ThreadLocal.Values` chỉ dùng được nếu constructor có `trackAllValues: true`; mặc định đọc property này ném `InvalidOperationException`. Dispose ThreadLocal khi hết dùng; nếu các giá trị giữ resource thì phải dọn chúng riêng.

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

Body CPU phải tự kiểm token: ném OCE để báo cancellation, hoặc thoát bình thường nếu đó là contract. Async method ném OCE tạo task Canceled; delegate đồng bộ của Task.Run có thêm quy tắc token matching (§3.2.1). Await không tự quyết định trạng thái của task.

`new Thread` không nhận token. Truyền vào closure và poll — `Sleep` vẫn block đến hết khoảng đó, rồi mới thấy hủy:

```csharp
using var cts = new CancellationTokenSource();

var t = new Thread(() =>
{
    while (!cts.Token.IsCancellationRequested)
    {
        DoPiece();
        Thread.Sleep(50); // hủy giữa Sleep: trễ tối đa 50ms
    }
});
t.Start();
cts.Cancel();
t.Join();
```

`CancelAfter` (hoặc ctor nhận `TimeSpan`) tự `Cancel` trên thread pool timer. Linked token hủy khi **bất kỳ** nguồn nào hủy — đăng ký, request, timeout:

```csharp
using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(2));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(timeout.Token, stoppingToken);

linked.Token.Register(() => Console.WriteLine("stop"), useSynchronizationContext: false);
```

Callback Register thường chạy đồng bộ trên đường Cancel; nếu token đã hủy có thể chạy ngay lúc Register. Callback blocking làm chậm caller và có thể gây deadlock với khóa ứng dụng. Dispose registration khi hết cần; useSynchronizationContext:false tránh marshal về UI.

`ThrowIfCancellationRequested` ném `OperationCanceledException` (subclass của nó là `TaskCanceledException` khi từ `Task`). `catch (OperationCanceledException)` bắt cả hai. Bắt `Exception` rồi nuốt = coi hủy là lỗi nghiệp vụ.

Chi tiết async (`WaitAsync` ném vs trả canceled): [async.md §8](async.md#8-cancellation--iprogress).

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

using var pipelineCts = new CancellationTokenSource();
async Task ProduceAsync(CancellationToken token)
{
    try
    {
        for (int i = 0; i < 1000; i++)
            await ch.Writer.WriteAsync(i, token);
    }
    catch (Exception ex) { ch.Writer.TryComplete(ex); throw; }
    finally { ch.Writer.TryComplete(); }
}

async Task ConsumeAsync(CancellationToken token)
{
    await foreach (var item in ch.Reader.ReadAllAsync(token)) Process(item);
}
async Task GuardAsync(Func<CancellationToken, Task> run)
{
    try { await run(pipelineCts.Token); }
    catch { pipelineCts.Cancel(); throw; }
}
await Task.WhenAll(GuardAsync(ProduceAsync), GuardAsync(ConsumeAsync));
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

`FullMode` khi bounded đầy — bảng và `Complete(ex)`: [async.md §12](async.md#12-channel--async). `Wait` (mặc định) là backpressure. `DropOldest` chỉ khi mất mẫu cũ được phép (telemetry), không phải lệnh thanh toán:

```csharp
var telemetry = Channel.CreateBounded<int>(new BoundedChannelOptions(8)
{
    FullMode = BoundedChannelFullMode.DropOldest,
    SingleReader = true
});

telemetry.Writer.TryWrite(1); // không await; đầy thì bỏ phần tử cũ nhất
```

Nhiều consumer **cạnh tranh một** `Reader` — mỗi item về đúng một vòng `ReadAsync`:

```csharp
async Task ConsumeAsync(ChannelReader<int> reader, CancellationToken ct)
{
    while (await reader.WaitToReadAsync(ct))
    {
        while (reader.TryRead(out int item))
            Process(item);
    }
}

var consumers = Enumerable.Range(0, 4)
    .Select(_ => ConsumeAsync(ch.Reader, CancellationToken.None));
```

`WaitToReadAsync` trả `false` khi writer đã `Complete` và kênh rỗng — vòng ngoài thoát. `TryRead` vét những gì đang có, không block. Một `ReadAllAsync` duy nhất cũng được nếu chỉ một consumer; đừng vừa `ReadAllAsync` vừa `TryRead` trên cùng reader từ hai tác vụ mà không hiểu item chỉ được lấy một lần.

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
5. **Lambda capture + shared mutable** — cần `lock` hoặc `localFinally` aggregate. Mỗi partition giữ tổng **riêng**, cộng vào kết quả một lần lúc xong — không `lock` từng phần tử:

```csharp
long sum = 0;
Parallel.For(0, data.Length,
    localInit: () => 0L,
    body: (i, _, local) => local + data[i],
    localFinally: local => Interlocked.Add(ref sum, local));
```

`Partitioner.Create(0, n, rangeSize)` khi vài phần tử nặng hơn hẳn (default chunk có thể dồn việc lớn vào một worker):

```csharp
var ranges = Partitioner.Create(0, n, rangeSize: 1024);
Parallel.ForEach(ranges, range =>
{
    for (int i = range.Item1; i < range.Item2; i++)
        Process(i);
});
```
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

Deadlock cổ điển trên UI (và ASP.NET Framework cũ, nơi có sync context): thread UI **block** chờ task, trong khi continuation của `await` cần chính thread đó.

```csharp
async Task<string> LoadAsync()
{
    await Task.Delay(10);          // continuation xin quay lại UI context
    return "ok";
}

// đang đứng trên UI thread:
string s = LoadAsync().Result;     // UI bị .Result chiếm → continuation không vào được → treo
string s2 = LoadAsync().GetAwaiter().GetResult(); // cùng kiểu deadlock
```

Sửa ở app: `await LoadAsync()` trên UI, đừng `.Result`. Sửa ở thư viện: `await ... .ConfigureAwait(false)` để continuation không đòi context. ASP.NET Core không gắn sync context nên `.Result` thường **không** deadlock kiểu này — nó vẫn block một thread pool và có thể starve (mục 3). `Post` xếp delegate rồi trả về; `Send` / `Dispatcher.Invoke` chạy đồng bộ và deadlock nếu gọi từ chính thread UI đang chờ ngược lại.

---

## 10. Chẩn đoán & đo đạc

```csharp
ThreadPool.GetMaxThreads(out var maxW, out var maxIO);
ThreadPool.GetAvailableThreads(out var availW, out var availIO);
long pending = ThreadPool.PendingWorkItemCount; // .NET Core 3.0+: số work item đang chờ
int workers = ThreadPool.ThreadCount;
Console.WriteLine($"Pool: workers={workers} availW={availW}/{maxW} pending={pending} availIO={availIO}/{maxIO}");
```

Đọc lúc tải, không phải lúc idle (`availW` gần `maxW` là bình thường khi rảnh).

| Số | Ý nghĩa |
|---|---|
| `pending` tăng, latency tăng, CPU thấp, ThreadCount tăng dần | Dấu hiệu starvation do worker blocking; `availW` vẫn có thể rất lớn vì nó tính từ max, không phải số worker đang rảnh thực tế |
| `ThreadCount` leo tới hàng trăm, CPU thấp | Thread block, pool tiêm thêm (mục 3.3). Tăng min không sửa nguyên nhân |
| `availIO` gần 0 | Callback I/O bị block, hoặc sync I/O chiếm completion port |

- Dùng **`Stopwatch`** đo thời gian; **PerfView/dotnet-trace** để phân tích contention/CPU. Contention `Monitor` / `Lock` hiện trong event `Microsoft-Windows-DotNETRuntime` (ContentionStart).
- **`ConcurrentQueue`**/`Channels` có counters hữu ích (EventSource).
- Chẩn đoán starvation bằng queue, throughput, CPU và stack của worker; không chỉ dựa vào `GetAvailableThreads`. Giảm sync-over-async khi đó là nguyên nhân.

---

## 11. Best practices & cảnh báo

- **Ưu tiên async I/O**; chỉ dùng thread cho CPU-bound hoặc API không async.
- **Không** tạo quá nhiều threads — để thread pool điều phối (work‑stealing, hill‑climbing).
- Tác vụ **blocking dài** → `TaskCreationOptions.LongRunning` hoặc **Thread** riêng.
- **Đồng bộ tối thiểu**: `Interlocked` khi đủ; `Lock`/`lock` khi cần critical section; tránh lock khi đang `await`.
- Dọn dẹp đúng: `CancellationToken`, `using` cho resource, `try/finally`.
- **UI**: thread affinity — cập nhật qua `SynchronizationContext` / `Dispatcher` (mục 9).
- **Đo đạc trước tối ưu**; stress test để phát hiện race/deadlock.
- Exception trong `new Thread` / `QueueUserWorkItem` hạ process. Exception trong `Task` nằm trên task — `await` nó.
- Pipeline I/O nặng: **async + Channels**; phần CPU: `Parallel.ForEach` / PLINQ **chỉ khi đo được lợi**.
- Gate mới trên .NET 9+: ưu tiên `System.Threading.Lock` thay `object` tùy ý.
- Correlation/request: `AsyncLocal`, không `ThreadLocal`.
- `Parallel` không phải “turbo” cho HTTP/EF.
