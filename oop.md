# Lập trình hướng đối tượng trong C#  

*(Class, OOP, Properties/Indexers, Events)*

---

## Mục lục

- [Lập trình hướng đối tượng trong C#](#lập-trình-hướng-đối-tượng-trong-c)
  - [Mục lục](#mục-lục)
  - [1. Class \& Object cơ bản](#1-class--object-cơ-bản)
    - [1.1 Khai báo class](#11-khai-báo-class)
    - [1.2 Field/const/readonly](#12-fieldconstreadonly)
    - [1.3 Access modifiers](#13-access-modifiers)
    - [1.4 Static vs instance](#14-static-vs-instance)
    - [1.5 Constructor](#15-constructor)
    - [1.6 Finalizer \& IDisposable](#16-finalizer--idisposable)
    - [1.7 Partial/Nested](#17-partialnested)
    - [1.8 Object initializer \& `required`](#18-object-initializer--required)
    - [1.9 Primary constructor (C# 12)](#19-primary-constructor-c-12)
  - [2. Kế thừa \& Đa hình](#2-kế-thừa--đa-hình)
    - [2.1 `virtual`/`override`/`abstract`/`sealed`](#21-virtualoverrideabstractsealed)
    - [2.2 `new` (method hiding)](#22-new-method-hiding)
    - [2.3 `base` \& constructor chaining](#23-base--constructor-chaining)
    - [2.4 Abstract class vs interface](#24-abstract-class-vs-interface)
    - [2.5 Kiểm tra/cast kiểu (`is`/`as`/pattern matching)](#25-kiểm-tracast-kiểu-isaspattern-matching)
  - [3. Interface](#3-interface)
    - [3.1 Khai báo/triển khai](#31-khai-báotriển-khai)
    - [3.2 Default interface methods (C# 8)](#32-default-interface-methods-c-8)
  - [4. Equality \& `ToString`](#4-equality--tostring)
    - [4.1 So sánh theo tham chiếu vs theo giá trị](#41-so-sánh-theo-tham-chiếu-vs-theo-giá-trị)
    - [4.2 `Equals`/`GetHashCode`/`IEquatable<T>`](#42-equalsgethashcodeiequatablet)
    - [4.3 Gợi ý về `record`](#43-gợi-ý-về-record)
  - [5. Properties](#5-properties)
    - [5.1 Auto-property \& backing field](#51-auto-property--backing-field)
    - [5.2 Truy cập backing field (C#14)](#52-truy-cập-backing-field-c14)
    - [5.3 Getter/setter nâng cao](#53-gettersetter-nâng-cao)
    - [5.4 `init`-only (C# 9) \& `required` (C# 11)](#54-init-only-c-9--required-c-11)
    - [5.5 Expression-bodied/Computed property](#55-expression-bodiedcomputed-property)
    - [5.6 Property \& thread-safety](#56-property--thread-safety)
    - [5.7 `INotifyPropertyChanged` tóm tắt](#57-inotifypropertychanged-tóm-tắt)
  - [6. Indexers](#6-indexers)
    - [6.1 Cú pháp \& ví dụ](#61-cú-pháp--ví-dụ)
    - [6.2 Nhiều tham số, quyền truy cập khác nhau](#62-nhiều-tham-số-quyền-truy-cập-khác-nhau)
    - [6.3 Mẫu dùng thường gặp](#63-mẫu-dùng-thường-gặp)
  - [7. Events](#7-events)
    - [7.1 Ôn nhanh delegate](#71-ôn-nhanh-delegate)
    - [7.2 `event` là gì?](#72-event-là-gì)
    - [7.3 Đăng ký/hủy \& phát sự kiện](#73-đăng-kýhủy--phát-sự-kiện)
    - [7.4 Event pattern .NET (`EventHandler<T>`)](#74-event-pattern-net-eventhandlert)
    - [7.5 Custom event accessor \& weak event](#75-custom-event-accessor--weak-event)
    - [7.6 Best practices \& cảnh báo](#76-best-practices--cảnh-báo)
  - [8. Extension members (C\# 14)](#8-extension-members-c-14)
    - [8.1 `extension` block — cú pháp](#81-extension-block--cú-pháp)
    - [8.2 Extension method / property / static / operator](#82-extension-method--property--static--operator)
    - [8.3 So sánh với classic `this` extension methods](#83-so-sánh-với-classic-this-extension-methods)
    - [8.4 Generic extension blocks \& lưu ý](#84-generic-extension-blocks--lưu-ý)
  - [9. Best practices tổng hợp](#9-best-practices-tổng-hợp)

---

## 1. Class & Object cơ bản

### 1.1 Khai báo class

```csharp
public class Person
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}
```

- `class` tạo **kiểu tham chiếu (reference type)**.  
- Một file có thể chứa nhiều class; tên file không nhất thiết trùng tên class.  
- Một class có thể khai báo **thành viên**: field, property, method, event, indexer, nested types.

### 1.2 Field/const/readonly

```csharp
public class Counter
{
    private int _value;                 // field
    public const int Max = 1_000;       // hằng compile-time
    public readonly Guid Id = Guid.NewGuid(); // gán duy nhất tại ctor
}
```

- `const` nhúng giá trị vào call-site → **thay đổi const** cần rebuild nơi dùng.  
- `readonly` cho phép gán ở **field initializer** hoặc **constructor**; không đổi sau đó. Nên dùng bất kỳ khi nào có một **field** bạn không muốn thay đổi giá trị sau khi khởi tạo.

### 1.3 Access modifiers

- `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`.  
- Quy tắc tổng quát: **thu hẹp phạm vi** nhất có thể (*least privilege*).

### 1.4 Static vs instance

```csharp
public class MathUtil
{
    public static double Pi => Math.PI;   // static property
    public static int Add(int a, int b) => a + b; // static method
}
```

- Thành viên `static` gắn với **kiểu**, không gắn instance.  
- Thành viên **instance** hoạt động trên mỗi đối tượng.

### 1.5 Constructor

```csharp
public class HttpClientWrapper
{
    private readonly HttpClient _client;

    public HttpClientWrapper() : this(new HttpClient()) { } // chain

    public HttpClientWrapper(HttpClient client)
    {
        _client = client;
    }

    // static ctor: chạy 1 lần trước khi dùng type (init static state)
    static HttpClientWrapper() { /* ... */ }
}
```

- Có thể overload ctor, **chaining** qua `: this(...)` hoặc gọi base ctor với `: base(...)`.  
- Nếu **không** khai báo ctor nào → compiler sinh **default ctor** (không tham số).  
- **Static constructor** không tham số, không access modifier, chạy 1 lần.

### 1.6 Finalizer & IDisposable

```csharp
public class NativeHandleHolder : IDisposable
{
    private IntPtr _handle;
    public void Dispose()
    {
        // giải phóng tài nguyên quản lý/không quản lý
        ReleaseHandle(_handle);
        GC.SuppressFinalize(this);
    }

    ~NativeHandleHolder() // finalizer
    {
        ReleaseHandle(_handle);
    }
}
```

- **Khuyến nghị dùng `IDisposable` + `using`/`await using`** thay vì trông chờ finalizer.  
- Finalizer tốn kém; chỉ dùng khi giữ tài nguyên unmanaged.

### 1.7 Partial/Nested

```csharp
public partial class HugeType { /* file A */ }
public partial class HugeType { /* file B */ }

public class Outer
{
    public class Inner { } // nested type
}
```

- `partial` chia định nghĩa 1 type ra nhiều file (phương thức, property, nested type…).  
- **C# 14** mở rộng: `partial` **constructor** và `partial` **event** — hữu ích với source generator (một phần khai báo, một phần implement).

#### Partial constructor (C# 14)

Phải có đúng **một** *defining declaration* (chỉ chữ ký) và **một** *implementing declaration* (thân). Chỉ bản implement được có constructor initializer (`: this(...)` / `: base(...)`). Chỉ **một** phần của type được dùng cú pháp **primary constructor**.

```csharp
// File A — defining declaration (có thể do source generator sinh)
public partial class Person
{
    public string Name { get; }
    public int Age { get; }

    public partial Person(string name, int age); // chỉ chữ ký
}

// File B — implementing declaration
public partial class Person
{
    public partial Person(string name, int age) : this(name, age, createdAt: DateTime.UtcNow)
    {
    }

    private Person(string name, int age, DateTime createdAt)
    {
        Name = name;
        Age = age;
        CreatedAt = createdAt;
    }

    public DateTime CreatedAt { get; }
}
```

#### Partial event (C# 14)

Defining declaration giống **field-like event**; implementing declaration **bắt buộc** có `add`/`remove`.

```csharp
// File A — defining (field-like)
public partial class ChatRoom
{
    public partial event EventHandler<string>? MessageReceived;
}

// File B — implementing (custom accessors)
public partial class ChatRoom
{
    private EventHandler<string>? _messageReceived;

    public partial event EventHandler<string>? MessageReceived
    {
        add
        {
            // ví dụ: log / giới hạn số subscriber
            _messageReceived += value;
        }
        remove
        {
            _messageReceived -= value;
        }
    }

    protected virtual void OnMessage(string msg)
        => _messageReceived?.Invoke(this, msg);
}
```

- **Nested type** hữu ích gói gọn logic phụ, chia tách visibility.

### 1.8 Object initializer & `required`

```csharp
public class User
{
    public required string Email { get; init; } // C# 11
    public string Name { get; init; } = "";
}

var u = new User { Email = "a@b.com", Name = "Alice" };
```

- **Object initializer** giúp đọc dễ, tránh nhiều ctor overload.  
- `required` buộc caller khởi tạo trước khi object “hoàn tất”.

### 1.9 Primary constructor (C# 12)

```csharp
public class Rectangle(double width, double height)
{
    public double Width { get; } = width;
    public double Height { get; } = height;
    public double Area => Width * Height;
}
```

- Khai báo tham số **ngay trên tiêu đề class**, được dùng trong body để gán property/field.  
- Tham số primary ctor là **phạm vi toàn class** (có thể dùng trong field/property initializer, method, nested…).  
- Secondary ctor phải gọi `: this(...)` để chạy primary ctor.

#### Pitfalls: capture & mutable

**1. Capture ngầm — tham số trở thành field ẩn**

Nếu dùng tham số primary ctor trong **instance member** (method/property), compiler có thể **capture** thành field private. Không gán vào property/`readonly` field tường minh → state “ẩn”, khó đọc/debug.

```csharp
public class Service(ILogger logger, string name)
{
    // OK rõ ràng: lưu vào field/property
    private readonly ILogger _logger = logger;
    public string Name { get; } = name;

    public void Run() => _logger.LogInformation("run {Name}", Name);
}

// ❌ dễ gây nhầm: `logger` bị capture thành field ẩn
public class Leaky(ILogger logger)
{
    public void Run() => logger.LogInformation("hi"); // capture ngầm
}
```

**2. Tham số primary ctor mặc định mutable**

Không phải `readonly` trừ khi bạn tự lưu vào `readonly` field / `get`-only property. Gán lại tham số trong method **đổi field ẩn**, không tạo biến local mới độc lập.

```csharp
public class Counter(int start)
{
    public void Bump() => start++;          // mutate capture
    public int Value => start;
}

var c = new Counter(0);
c.Bump();
Console.WriteLine(c.Value); // 1
```

**3. Struct + primary ctor**

Với `struct`, cẩn trọng **copy semantics**: mutate qua primary-parameter-capture trên bản copy có thể không phản ánh về biến gốc. Ưu tiên gán tường minh vào property/`readonly` field.

**4. Shadowing & naming**

Tránh trùng tên tham số primary ctor với property cùng kiểu nếu dễ nhầm; convention phổ biến: tham số camelCase, property PascalCase, hoặc gán ngay `Width = width`.

**Gợi ý:** coi primary ctor như “dependency/state đầu vào” — **luôn** đưa vào `readonly` field / `get`-only/`init` property nếu muốn bất biến và API rõ.

---

## 2. Kế thừa & Đa hình

### 2.1 `virtual`/`override`/`abstract`/`sealed`

```csharp
public abstract class Shape
{
    public abstract double Area();       // phải override
    public virtual string Name => "Shape";   // có thể override
}

public sealed class Circle : Shape
{
    public double R { get; }
    public Circle(double r) => R = r;

    public override double Area() => Math.PI * R * R;

    public override string Name => "Circle";
}
```

- `abstract` → **class** không tạo instance, **method** bắt buộc override.  
- `virtual` → method có default behavior, có thể override.  
- `sealed` class → **không** cho kế thừa; `sealed override` → chặn override tiếp.

### 2.2 `new` (method hiding)

```csharp
public class Base { public void Log() => Console.WriteLine("Base"); }
public class Derived : Base { public new void Log() => Console.WriteLine("Derived"); }
```

- `new` **ẩn** member cùng chữ ký ở base; chọn member nào phụ thuộc **tĩnh** loại tham chiếu.

### 2.3 `base` & constructor chaining

```csharp
public class Animal { public Animal(string name) { } }
public class Dog : Animal
{
    public Dog(string name) : base(name) { }
}
```

- Dùng `base` để gọi member/ctor của lớp cha.

### 2.4 Abstract class vs interface

- **Abstract class**: chia sẻ *code + state*, có field, ctor, mức bảo vệ linh hoạt.  
- **Interface**: chỉ hợp đồng thành viên, 1 type có thể implement **nhiều** interface.
- Ta dùng abstract class để tạo ra một bộ khung cho các lớp con, khi chúng ta có nhiều lớp với các thành phần giống nhau, mục đích để **dùng lại** code khai báo trong abstract class. Trong khi đó, interface đóng vai trò là một bản cam kết, một hợp đồng mà trong đó các lớp khác nhau có thể dựa trên đó để làm việc với nhau, một khi ta biết một class implement một interface, ta được đảm bảo rằng class này có chứa các hàm mà interface đó định nghĩa.
- Từ góc độ OOP, abstract class đại diện cho lợi ích do tính **thừa kế** mang lại, trong khi đó interface hỗ trợ tính **trừu tượng**.

### 2.5 Kiểm tra/cast kiểu (`is`/`as`/pattern matching)

```csharp
if (obj is Circle c && c.R > 0) { /* ... */ }

Shape s = GetShape();
switch (s)
{
    case Circle { R: > 0 } circle:
        Console.WriteLine(circle.Area());
        break;
    case null:
        throw new ArgumentNullException();
}
```

- **Pattern matching** (C# hiện đại) giúp code ngắn gọn, an toàn null/type.

---

## 3. Interface

### 3.1 Khai báo/triển khai

```csharp
public interface IRepository<T>
{
    T? FindById(Guid id);
    void Add(T entity);
}

public class MemoryRepository<T> : IRepository<T>
{
    private readonly Dictionary<Guid, T> _store = new();
    public T? FindById(Guid id) => _store.TryGetValue(id, out var v) ? v : default;
    public void Add(T entity) => _store[Guid.NewGuid()] = entity!;
}
```

### 3.2 Default interface methods (C# 8)

```csharp
public interface ILogger
{
    void Write(string message);
    void Info(string message) => Write("[INFO] " + message); // default impl
}
```

- Cho phép **định nghĩa mặc định** trong interface; hữu ích khi mở rộng API mà không phá implement cũ.  
- Dùng tiết chế để tránh “logic trôi dạt” khỏi class cài đặt.

---

## 4. Equality & `ToString`

### 4.1 So sánh theo tham chiếu vs theo giá trị

- `ReferenceEquals(a, b)` kiểm tra **cùng instance**.  
- `Equals` mặc định ở `object` là reference-equality; có thể **override** để so sánh theo **giá trị**.

### 4.2 `Equals`/`GetHashCode`/`IEquatable<T>`

```csharp
public class Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);

    public bool Equals(Point? other) => other is not null && X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Point p && Equals(p);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public override string ToString() => $"({X},{Y})";
}
```

- Nếu override `Equals`, **phải** override `GetHashCode`.  
- Khi dùng trong `Dictionary`/`HashSet` → chức năng này rất quan trọng.

### 4.3 Gợi ý về `record`

- `record`/`record class` mặc định có **value-based equality** và `with`-expression.  
- Nếu mục tiêu là immutable + so sánh theo giá trị → cân nhắc dùng `record`.

---

## 5. Properties

### 5.1 Auto-property & backing field

```csharp
public class User
{
    public string Name { get; set; } = "";
    public int Age { get; private set; }  // setter private
}
```

- Auto-property do compiler sinh backing field.  
- Có thể kiểm soát access: `public string Name { get; private set; }`.

**Full property** (tự quản luật):  
```csharp
private int _age;
public int Age
{
    get => _age;
    set
    {
        if (value < 0) throw new ArgumentOutOfRangeException(nameof(value));
        _age = value;
    }
}
```

### 5.2 Truy cập backing field — từ khóa `field` (C# 14)

Từ khóa contextual **`field`** cho phép viết thân accessor mà **không** khai báo backing field tường minh. Compiler sinh field ẩn và thay mọi `field` trong accessor bằng field đó.

**Trước C# 14** — phải tự khai báo field:

```csharp
private string _msg = "";
public string Message
{
    get => _msg;
    set => _msg = value ?? throw new ArgumentNullException(nameof(value));
}
```

**C# 14** — giữ auto-property, thêm logic qua `field`:

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

Có thể viết thân cho **một hoặc cả hai** accessor:

```csharp
public int Age
{
    get => field;
    set
    {
        if (value < 0) throw new ArgumentOutOfRangeException(nameof(value));
        field = value;
    }
}

public string Name
{
    get => field ??= "(unknown)";
    set;
}
```

Kết hợp `init` / `required`:

```csharp
public required string Email
{
    get;
    init => field = string.IsNullOrWhiteSpace(value)
        ? throw new ArgumentException("Email required", nameof(value))
        : value.Trim();
}
```

**Phạm vi & hạn chế:**

- `field` chỉ hợp lệ **bên trong** accessor của property field-backed đó — không dùng từ method/ctor khác (phải qua property).
- Nếu type đã có thành viên tên `field`, từ khóa **che** identifier. Disambiguate bằng `@field` / `this.field`, hoặc **đổi tên** thành viên cũ.
- Property vẫn là API công khai; field sinh ra là implementation detail (không phải public field).

**Khi nào dùng:** validation nhẹ, chuẩn hóa giá trị, lazy default trong getter — thay vì full property + `_backing`. Khi cần nhiều field phụ / logic phức tạp giữa nhiều property → vẫn dùng backing field tường minh.

### 5.3 Getter/setter nâng cao

- **Access khác nhau** cho get/set: `public int Age { get; internal set; }`.  
- **Static property**: `public static string Version { get; } = "1.0";`  
- **Chỉ-get** (computed): `public double Area => Width * Height;`

### 5.4 `init`-only (C# 9) & `required` (C# 11)

```csharp
public class Order
{
    public required string Id { get; init; }
    public string? Note { get; init; }
}

var o = new Order { Id = "A001", Note = "COD" };
```

- `init` cho phép gán tại init (ctor/object initializer), *không* gán sau đó.  
- `required` buộc caller cung cấp giá trị trước khi object usable.

### 5.5 Expression-bodied/Computed property

```csharp
public string FullName => $"{LastName}, {FirstName}";
```

- Tránh lưu trữ dư thừa; tính toán khi cần.

### 5.6 Property & thread-safety

- Không phải property nào cũng “nhẹ”; nếu có tính toán/chậm → xem xét cache lại (lazy).  
- Nếu set/get thay đổi state dùng chung, cân nhắc **lock** hoặc cấu trúc thread-safe.  

```csharp
private readonly object _sync = new();
private int _count;
public int Count
{
    get { lock (_sync) return _count; }
    private set { lock (_sync) _count = value; }
}
```

### 5.7 `INotifyPropertyChanged` tóm tắt

```csharp
public class Person : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    private string _name = "";
    public string Name
    {
        get => _name;
        set
        {
            if (_name == value) return;
            _name = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
        }
    }
}
```

- Sử dụng khi ta muốn phát ra sự kiện PropertyChanged mỗi khi một thuộc tính nào đó thay đổi giá trị.
- Trong MVVM/WPF người ta bắt PropertyChanged để UI tự cập nhật.

---

## 6. Indexers

### 6.1 Cú pháp & ví dụ

```csharp
public class Matrix
{
    private readonly double[,] _data;
    public Matrix(int m, int n) => _data = new double[m, n];

    public double this[int i, int j]
    {
        get => _data[i, j];
        set => _data[i, j] = value;
    }
}
```

- Indexer giống property nhưng nhận **tham số**.  
- Tên truy cập là `this[...]`.

### 6.2 Nhiều tham số, quyền truy cập khác nhau

```csharp
public class Settings
{
    private readonly Dictionary<string, string> _store = new();

    public string this[string key]
    {
        get => _store.TryGetValue(key, out var v) ? v : "";
        internal set => _store[key] = value;
    }
}
```

- Có thể set access khác nhau cho get/set; có thể overload indexer theo kiểu tham số.

### 6.3 Mẫu dùng thường gặp

- Truy cập bộ sưu tập tuỳ biến (`SparseArray`, `Grid`, `RangeMap`…).  
- Cung cấp API “giống mảng” cho cấu trúc dữ liệu.

---

## 7. Events

### 7.1 Ôn nhanh delegate

```csharp
public delegate void Notifier(string message);

// Hoặc dùng sẵn: Action, Func<T>, EventHandler<T>
Action<string> a = Console.WriteLine;
```

- **Delegate** là kiểu tham chiếu tới **phương thức**; nền tảng của event.

### 7.2 `event` là gì?

```csharp
public class Timer
{
    public event EventHandler? Tick; // khai báo event

    protected virtual void OnTick() => Tick?.Invoke(this, EventArgs.Empty);
}
```

- `event` là **cổng** cho phép **đăng ký/hủy** delegate theo mô hình phát-sự-kiện.  
- Từ ngoài chỉ `+=`/`-=`; chỉ code bên trong type mới được **Invoke**.

### 7.3 Đăng ký/hủy & phát sự kiện

```csharp
var t = new Timer();
t.Tick += (s, e) => Console.WriteLine("tick");
t.Tick -= Handler; // luôn hủy khi không còn dùng để tránh memory leak
```

- Khi phát sự kiện: dùng `?.Invoke` để an toàn null & tránh race condition.

### 7.4 Event pattern .NET (`EventHandler<T>`)

```csharp
public class DownloadProgressChangedEventArgs : EventArgs
{
    public int Percent { get; }
    public DownloadProgressChangedEventArgs(int p) => Percent = p;
}

public class Downloader
{
    public event EventHandler<DownloadProgressChangedEventArgs>? ProgressChanged;
    protected virtual void OnProgress(int p) =>
        ProgressChanged?.Invoke(this, new DownloadProgressChangedEventArgs(p));
}
```

- Chuẩn .NET: `sender` là **this**, `EventArgs` chứa dữ liệu.  
- Tạo **method bảo vệ** `OnXxx` để lớp con có thể override logic phát sự kiện.

### 7.5 Custom event accessor & weak event

```csharp
public class Source
{
    private EventHandler? _changed;
    public event EventHandler Changed
    {
        add    { _changed += value; /* custom logic */ }
        remove { _changed -= value; /* custom logic */ }
    }
}
```

- Custom accessor cho phép kiểm soát đăng ký (giới hạn số lượng, log…).  
- **Weak event** (mẫu nâng cao) giúp tránh giữ mạnh subscriber → giảm rò rỉ bộ nhớ.

### 7.6 Best practices & cảnh báo

- `event` khác **public delegate field**: field có thể bị gọi từ ngoài → **tránh**.  
- Luôn **hủy đăng ký** khi không cần (đặc biệt vòng đời dài).  
- Dùng `protected virtual OnXxx` thay vì `public void RaiseXxx`.  
- Cân nhắc **Async events**? Không có cơ chế chuẩn; thường **không** khuyến khích `async void` trong event handler (khó quản lý lỗi).

---

## 8. Extension members (C# 14)

C# 14 giới thiệu khối **`extension`** trong `static class` (top-level, không generic) để khai báo *extension members*: method, property, operator — theo kiểu **instance** hoặc **static** của receiver. Classic extension method (`this T` trên tham số đầu) vẫn hoạt động và **tương thích IL** với dạng mới.

> Góc nhìn method/API: xem thêm [methods.md — §12](./methods.md#12-this-và-extension-method).  
> **C# 15 Preview:** extension **indexer** trong `extension` block — chưa GA; đánh dấu *preview* nếu dùng.

### 8.1 `extension` block — cú pháp

```csharp
public static class EnumerableExtensions
{
    // Instance extensions: có tên receiver (`source`)
    extension<TSource>(IEnumerable<TSource> source)
    {
        public bool IsEmpty => !source.Any();

        public IEnumerable<TSource> TakeLastN(int n)
            => source.TakeLast(n);
    }

    // Static extensions: chỉ kiểu receiver, không cần tên tham số
    extension<TSource>(IEnumerable<TSource>)
    {
        public static IEnumerable<TSource> Identity
            => Enumerable.Empty<TSource>();

        public static IEnumerable<TSource> Combine(
            IEnumerable<TSource> first,
            IEnumerable<TSource> second)
            => first.Concat(second);

        public static IEnumerable<TSource> operator +(
            IEnumerable<TSource> left,
            IEnumerable<TSource> right)
            => left.Concat(right);
    }
}
```

Gọi:

```csharp
IEnumerable<int> nums = Enumerable.Range(1, 5);
bool empty = nums.IsEmpty;                    // instance extension property
var id = IEnumerable<int>.Identity;           // static extension property
var merged = nums + Enumerable.Range(10, 3);  // extension operator
```

### 8.2 Extension method / property / static / operator

| Loại | Khai báo trong block | Gọi như |
|------|----------------------|---------|
| Instance method | Có tên receiver; method dùng `source` | `seq.Foo(...)` |
| Instance property | `get` (computed; không có storage riêng trên receiver) | `seq.IsEmpty` |
| Static method/property | Block `extension(Type)` không tên receiver; member `static` | `Type.Member` / `Type.Method(...)` |
| Operator | `public static ... operator ...` trong extension | `a + b`, v.v. |

```csharp
public static class StringExtensions
{
    extension(string s)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(s);

        public string Truncate(int max)
            => s.Length <= max ? s : s[..max];
    }
}

"  ".IsBlank;           // true
"hello world".Truncate(5); // "hello"
```

### 8.3 So sánh với classic `this` extension methods

```csharp
// Classic (C# 3+) — chỉ method
public static class Classic
{
    public static bool IsEmpty<T>(this IEnumerable<T> source)
        => !source.Any();
}

// C# 14 extension block — method + property + static + operator
public static class Modern
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
        public int CountOrZero => source.TryGetNonEnumeratedCount(out var c) ? c : source.Count();
    }
}
```

- **Cùng IL / binary compatible:** có thể migrate classic → `extension` block **không** breaking caller.  
- Classic **không** hỗ trợ extension property / static member / operator trên type ngoài.  
- Member **trùng chữ ký** với member thật của type → member của type **thắng**.  
- Mọi member trong cùng static class (kể cả nhiều `extension` block) phải có **signature sinh ra duy nhất** (receiver tham gia signature).

### 8.4 Generic extension blocks & lưu ý

- Type parameter dùng trên **receiver** → khai báo trên `extension<T>(...)`.  
- Type parameter **chỉ** của member → khai báo trên member; **không** lặp cùng `T` ở cả hai chỗ.

```csharp
public static class GenericExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public IEnumerable<T> Spread(int start, int count)
            => source.Skip(start).Take(count);

        public IEnumerable<T> AppendMapped<TArg>(
            IEnumerable<TArg> second,
            Func<TArg, T> map)
        {
            foreach (var x in source) yield return x;
            foreach (var y in second) yield return map(y);
        }
    }
}
```

**Lưu ý thiết kế:**

- Extension property là **computed** — không thêm field vào type gốc.  
- Namespace + `using` vẫn quyết định discoverability (giống extension method).  
- Tránh “API ma” che member quan trọng của BCL; ưu tiên tên rõ miền.

---

## 9. Best practices tổng hợp

- **Encapsulation trước**: ưu tiên property/private field, che giấu chi tiết bên trong.  
- **Immutability khi có thể**: dùng `init`, `readonly`, `record` để giảm bug.  
- **Primary ctor**: gán tường minh vào field/property; tránh mute/capture ẩn.  
- **`field` keyword**: validation nhẹ trên auto-property; đổi tên nếu trùng identifier `field`.  
- **Extension members**: dùng `extension` block khi cần property/operator/static; classic `this` vẫn ổn cho method đơn.  
- **API nhất quán**: tên rõ ràng, overload hợp lý, tránh phương thức “God”.  
- **Không lạm dụng kế thừa**: cân nhắc composition/strategy.  
- **Override đúng cặp**: nếu override `Equals` → override `GetHashCode`.  
- **Sự kiện**: luôn hủy đăng ký, dùng pattern `OnXxx` và `EventHandler<T>`; partial event khi source-gen.  
- **Thread-safety**: lock nơi cần, tránh race condition khi phát event/properties.
